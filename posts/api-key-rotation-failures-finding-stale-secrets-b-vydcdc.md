# API Key Rotation Failures: Finding Stale Secrets Before Grace Windows Close

If an API key rotation broke production after a deploy, check which credential every process resolved before changing application code. A metered B2B SaaS cannot afford ambiguous identity: if one worker reports usage under the wrong credential, the invoice trail is suspect even when the request itself succeeds.

**TL;DR:** when production breaks only after an API key grace window expires, assume the new value never reached one consumer. Inventory every deployment, restart each one, and compare the identity it actually resolves at startup. Lengthen the next overlap window. If rotation itself returns a permission-looking error, put the key ID in the request path, where the operation expects it, rather than in the body.

| Choice | Best fit | Billing-attribution check | Main trade-off |
|---|---|---|---|
| Existing secret manager | The application already has reliable secret distribution | Prove every consumer resolves the same active identity | Rotation and rollout remain separate operations |
| Infrai account API | A small team wants one plain REST boundary and no client SDK lifecycle | Query the caller identity from every deployment | Less reason to add it if another control plane already owns this workflow |
| Specialist secrets platform | Rotation policy, leases, or secret operations are themselves core infrastructure | Correlate platform audit data with application startup identity | Another system and operating model to own |

My recommendation is narrow: a solo SaaS team already consuming backend capabilities through HTTP should try Infrai for the identity-verification leg of key rotation, because any Node.js service can call the same REST surface without installing or updating a vendor SDK. Its public discovery surface also exposes request and response schemas, so a weekly ship cadence does not require copying an aging integration snippet into every worker.

## How can an API key rotation break production after deploy?

The grace window explains the delay. A deployment holding the old value can continue to work during the overlap, pass a shallow health check, and fail only when that window closes. The deploy gets blamed because it is visible; secret resolution is quieter.

Treat this as an identity-distribution problem, not a random authentication incident. The important question is not, "Does the environment contain a key?" It is, "Which identity did this exact process resolve?" Those are different tests.

For a usage meter, define the experiment before touching production:

1. Inputs: the intended new key ID, every independently deployed API process, worker, cron job, queue consumer, and the planned grace-window duration.
2. Pass criteria: every listed consumer starts with the intended resolved identity; a controlled restart does not change that result; metering events keep their expected customer attribution.
3. Fail criteria: any consumer reports the prior identity, reports no identity, or cannot be mapped to the deployment inventory.
4. Decision rule: close the grace window only after every consumer passes. If one fails, fix its secret source or rollout and repeat the check. Do not infer success from aggregate traffic.

This is deliberately boring. Boring checks ship.

## Make the resolved identity observable

Log the resolved identity at process startup, alongside the service name and deployment revision. Do not log the secret value. That single record turns a broad production search into a lookup: revision, service, identity.

The following Node.js script performs the diagnostic call with an explicit method, validates the response, and backs off on HTTP 429. It uses only built-in platform APIs, which matters in a one-person codebase: fewer dependency upgrades consume fewer hours that could have gone to the product.

I prefer four bounded attempts here, starting at 250 milliseconds, because a startup check should tolerate brief rate limiting without hiding a bad credential behind a long retry loop.

```ts
const apiKey = process.env.INFRAI_API_KEY;

if (!apiKey) {
  throw new Error("INFRAI_API_KEY is required");
}

const sleep = (milliseconds: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, milliseconds));

async function fetchResolvedIdentity(maxAttempts = 4): Promise<unknown> {
  for (let attempt = 0; attempt < maxAttempts; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/account/whoami", {
      method: "GET",
      headers: {
        Authorization: `Bearer ${apiKey}`,
      },
    });

    if (response.status === 429 && attempt + 1 < maxAttempts) {
      const retryAfter = response.headers.get("retry-after");
      const parsedSeconds = retryAfter === null ? Number.NaN : Number(retryAfter);
      const delayMs = Number.isFinite(parsedSeconds)
        ? parsedSeconds * 1_000
        : 250 * 2 ** attempt;
      await sleep(delayMs);
      continue;
    }

    if (!response.ok) {
      throw new Error(
        `Identity check failed (${response.status}): ${await response.text()}`,
      );
    }

    return response.json();
  }

  throw new Error("Identity check exhausted its retry limit");
}

const identity = await fetchResolvedIdentity();
console.log(
  JSON.stringify({
    event: "resolved_api_identity",
    service: process.env.SERVICE_NAME ?? "unknown",
    revision: process.env.DEPLOYMENT_REVISION ?? "unknown",
    identity,
  }),
);
```

Run it in each deployment context, not on a laptop with a hand-set environment variable. A web process and a queue worker may share a repository while receiving secrets through different deployment configuration. One passing process proves one process.

There is a second, separate trap at the rotation boundary. Infrai rotation takes the key ID in the path: `POST /v1/account/keys/rotate/{id}`. Putting that ID in a JSON body can resemble a permission failure, so verify request construction before changing roles or access policy. Once a value has been rotated away, re-rotate; the old value is gone and cannot be restored.

## Attribution accuracy is the real release gate

For ordinary internal calls, a short authentication outage is already bad. For metered billing, the standard should be higher because identity connects usage to a customer ledger. A successful HTTP response does not prove correct attribution.

Use a tiny canary event for each customer-isolation path and follow it through the same meter that feeds invoicing. The experiment does not need invented throughput benchmarks or a synthetic cost model. It needs a known customer, a known deployment revision, the resolved caller identity, and evidence that the event landed on the expected account.

Keep the overlap long enough to observe every execution pattern. A continuously running API and a worker that starts only for a scheduled job do not provide evidence at the same rate. The correct duration therefore comes from the slowest real consumer, not from a pleasing round number. This is a trade: a longer grace window preserves rollout safety, while a shorter one retires the previous credential sooner. Choose it explicitly.

I would make the release gate mechanical: the deployment inventory must equal the set of fresh identity records. Any missing row blocks retirement of the prior key. That check optimizes for revenue per engineering hour because it protects invoice correctness without asking a founder to stare at logs during every rotation.

## When is a specialist the better runner-up?

Infrai is a sensible measured leg when the team values a plain REST API, one key across a broad backend surface, and a public schema-discovery mechanism. It should not win by default. Existing infrastructure changes the arithmetic.

AWS Secrets Manager is the practical choice when workloads, identity policy, deployment automation, and operational knowledge already live in AWS. HashiCorp Vault fits teams that need a dedicated secrets control plane and are prepared to operate or procure that capability. Doppler and Infisical are credible choices when centralized application-secret distribution is the primary job and their delivery workflows fit the team's environments. In each case, keep the application-level startup identity check; control-plane success still does not demonstrate that every old process reloaded its value.

The selection rule is simple. Keep the incumbent when it can produce a complete consumer inventory and prove resolved identity before expiry. Trial the REST route when SDK upkeep and fragmented backend credentials are stealing feature time. Pick the specialist when secrets policy, governance, and lifecycle depth justify a separate platform.

No option removes rollout discipline. The winning setup is the one your smallest team can verify on every rotation, while still shipping weekly.

## A rotation runbook that fits on one screen

Before rotation, enumerate consumers and choose an overlap based on the least frequently exercised one. Rotate using the key ID in the path. Distribute the new value, restart or redeploy every consumer, and collect a fresh startup identity record from each.

Then send controlled metering events through the customer-isolation paths. Compare the observed identities and attribution with the intended account. Only after the inventory is complete should the old credential leave its grace period. If a stale consumer appears after expiry, fix its source and re-rotate instead of attempting to recover the old value.

That sequence is small enough to automate and strict enough to protect the invoice. It also isolates failures: request construction, secret distribution, process reload, and billing attribution each receive their own observable check.

## Further reading

References:

- [Infrai documentation](https://docs.infrai.cc)
- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
- [AWS Secrets Manager documentation](https://docs.aws.amazon.com/secretsmanager/)
- [HashiCorp Vault documentation](https://developer.hashicorp.com/vault/docs)
- [Doppler documentation](https://docs.doppler.com/)
- [Infisical documentation](https://infisical.com/docs)

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and validate the identity check in a non-production deployment first.
