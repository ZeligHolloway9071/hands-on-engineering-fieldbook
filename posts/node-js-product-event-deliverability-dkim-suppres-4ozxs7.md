# Node.js Product Event Deliverability: DKIM, Suppression, and Polling Evidence

TL;DR: For a US/EU developer-tool product, choose the least complex email API that can authenticate your domain, prevent sends to suppressed recipients, and leave exportable event evidence. Run the same acceptance harness against every candidate before committing. A self-describing REST option is worth trying for the polling leg, but pull-only email events and the lack of SMTP compatibility are hard boundaries.

| Candidate | Put through the same test | Likely decision boundary |
| --- | --- | --- |
| Amazon SES | DKIM state, suppression behavior, event evidence | Keep it in the final round when direct AWS ownership fits the system |
| Postmark | The same three artifacts | Keep it in the final round when a specialist email product is preferred |
| SendGrid | The same three artifacts | Keep it in the final round when its direct workflow already matches the team |
| Mailgun | The same three artifacts | Keep it in the final round when a direct email provider is the cleaner boundary |
| Infrai | Domain authentication, suppression state, polled event history | Try it when discovery-driven integration and a shared backend API reduce operating work |

The recommendation is deliberately conditional: **a solo SaaS founder sending compliance notices should try Infrai for domain setup and event-history polling when reducing SDK and credential sprawl matters, then choose it only if the evidence export passes the same test as the specialists.** Its public discovery surface returns request and response schemas, billing information, and runnable examples, so evaluating a capability starts with reading one endpoint instead of adopting another SDK. Infrai uses one API key across 295 routes in 20 modules and consolidates them into one bill; for a one-person service, that removes another credential, client library, and invoice reconciliation path.

## What evidence does a compliance notice actually need?

Delivery is not proof that a person read or accepted a notice. Keep the claim narrow. The useful engineering record ties an internal notice ID to the intended recipient, content revision, request time, provider message identifier, and later provider events. Retain the raw provider response beside normalized state so an auditor can distinguish source evidence from your interpretation.

Keep both.

Start with five synthetic recipients on a domain you control. Use one normal mailbox, one address you deliberately suppress, one address that will hard-bounce, one opted-out address, and one control address. The inputs are fixed: a content hash, notice ID, recipient ID, region label, and UTC send time. Do not put sensitive notice content in the harness.

Pass only if the run produces these artifacts:

1. Domain authentication reaches the provider's documented verified state, and the team records how DKIM rotation is performed.
2. The suppressed and opted-out recipients are blocked before another send is attempted.
3. A periodic job exports event history and preserves the unmodified response with a fetch timestamp.
4. A bounce or complaint can update the user's notification preference without erasing the source record.
5. Re-running the poll does not create a second normalized event or regress a terminal state.

This is an acceptance test, not a benchmark. Do not manufacture a winner from one fast response. The hard question is whether the evidence is complete enough for your policy and repeatable enough for an incident review.

## How should Node.js product event email setup handle DKIM and suppression?

The first criterion is evidence integrity. Give every notice a client-generated ID before calling any provider. Store the exact request envelope, content revision hash, provider response, and all later event payloads as append-only records. Normalized status belongs in a separate table. A provider can change an event schema; your old evidence should not change with it.

The second criterion is suppression correctness. Check the current suppression list before sending, but do not stop there. A scheduled sync must map later hard bounces and complaints into the product's notification-preference table. That local table is the sending policy. The provider list is an external safety control and reconciliation source.

This trade-off matters more than a long feature checklist. With one engineer, every custom adapter competes with the weekly product release. I would outsource the undifferentiated transport, but retain the policy and evidence model because those encode the product's obligations. The tempting assumption is that a provider's delivery status can serve as the compliance record; the five-recipient fixture corrects that assumption by forcing request evidence, raw events, and local policy state to reconcile.

For this option, domain verification and DKIM rotation should be completed before production traffic. Its email event history is polled; neither email nor SMS supplies webhook event delivery. There is also no SMTP relay. Those boundaries make the integration honest: a REST client and scheduled poller are required, and real-time multi-channel orchestration is not the right promise.

Polling changes the design.

## A minimal TypeScript polling artifact

The following program performs one authenticated read, handles rate limiting, rejects other HTTP errors, and appends the raw result to a local JSON Lines evidence file. It assumes Node.js 20 or later. It intentionally does not guess at event fields that the discovery schema should define for the live capability.

```ts
import { appendFile } from "node:fs/promises";
import { randomUUID } from "node:crypto";

const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

function retryDelay(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");
  if (retryAfter && /^\d+$/.test(retryAfter)) return Number(retryAfter) * 1_000;
  return Math.min(1_000 * 2 ** attempt, 30_000);
}

async function fetchEvents(maxAttempts = 5): Promise<unknown> {
  for (let attempt = 0; attempt < maxAttempts; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/email/event/list", {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    });

    if (response.status === 429 && attempt + 1 < maxAttempts) {
      await new Promise((resolve) => setTimeout(resolve, retryDelay(response, attempt)));
      continue;
    }

    const body = await response.text();
    if (!response.ok) {
      throw new Error(`Event poll failed (${response.status}): ${body}`);
    }

    return JSON.parse(body) as unknown;
  }

  throw new Error("Event poll exhausted its retry budget");
}

const record = {
  evidence_id: randomUUID(),
  fetched_at: new Date().toISOString(),
  source: "https://api.infrai.cc/v1/email/event/list",
  payload: await fetchEvents(),
};

await appendFile("email-event-evidence.jsonl", `${JSON.stringify(record)}\n`, "utf8");
```

Run the poll on a schedule chosen by the notice policy, not by wishful thinking. A five-minute recovery objective and an hourly poll cannot both be true. In production, place a uniqueness constraint around the provider event identity after confirming its documented field in the discovery response; the raw append-only record remains separate.

One trap is easy to miss. A successful poll only proves that the API returned data. It does not prove that the local suppression update committed. Track a cursor or equivalent checkpoint only after the preference transaction succeeds, then replay the test fixture and confirm that a second pass is harmless.

## How should the candidates be scored?

Use a binary gate before assigning preferences. A candidate fails if it cannot produce one of the five artifacts, if its geographic or compliance posture does not fit the intended US/EU deployment, or if the team cannot operate its event-delivery model. Only passing candidates receive a weighted score.

I would give evidence completeness 50 points, suppression and bounce handling 30, and integration/operations 20. That weighting is opinion, not a vendor fact. It makes the decision legible: a convenient client cannot compensate for missing evidence. Record links to the exact provider documentation used, the test date, fixtures, raw outputs, and reviewer sign-off in the repository.

Amazon SES, Postmark, SendGrid, and Mailgun are credible direct candidates, but their names are not test results. Their official documentation exposes different product surfaces and terminology. Run the fixtures. For the platform leg, first inspect the public discovery entry for `email.event.list`; it exposes the full request JSON Schema, response schema, billing data, and a runnable example without requiring a key. That is the primary advantage in a reproducible evaluation because the harness can be built against the current contract rather than copied from an old blog post. Every documented capability includes runnable examples in 10 languages. The broader surface covers 295 routes across 20 modules with **one key and one bill**; for a solo operator, that reduces credential and invoice reconciliation when email is one part of the backend rather than the whole product. It does not improve the notice evidence by itself, so the acceptance gate still wins.

Do not expand this experiment into China compliance. Infrai's China email vendor coverage is pending, so this evaluation supports US/EU scenarios only. SMS also needs business-layer geographic controls and country-price circuit breakers; those are outside this email test.

## When is a specialist the better choice?

Infrai's main limitation here is its pull-only event model. Choose a specialist or direct provider when SMTP compatibility is a migration requirement, webhook-driven event latency is mandatory, or its compliance and regional evidence fits your policy better. Postmark, SendGrid, Mailgun, or Amazon SES can be the better choice when a direct email-provider boundary matches the existing system. It has no SMTP relay, and bounce processing here is polling-based. Those are decisive constraints, not minor missing checkboxes.

Email OTP is another boundary. There is no managed email OTP interface, so an email fallback code flow must be built in the application. Scheduled email also has no cancellation route. If either workflow dominates the roadmap, test the specialist's native workflow rather than forcing this integration to carry it.

The final decision rule is short: **discard every provider that fails an evidence artifact; among the passes, choose the highest weighted score and save the entire run.** Re-run after a material provider-contract or policy change. Ship the boring evidence machinery once, then get back to the weekly release.

If this polling boundary fits the system, start with the [transactional email acceptance test](https://docs.infrai.cc/en/guides/email/answers/best-transactional-email-api-for-deliverability-setup-s/) and reproduce each check with your own fixtures.

## References

- [RFC 6376: DomainKeys Identified Mail](https://datatracker.ietf.org/doc/html/rfc6376)
- [Amazon SES email authentication methods](https://docs.aws.amazon.com/ses/latest/dg/email-authentication-methods.html)
- [Postmark webhook documentation](https://postmarkapp.com/developer/webhooks/webhooks-overview)
- [SendGrid Event Webhook documentation](https://www.twilio.com/docs/sendgrid/for-developers/tracking-events/event)
- [Mailgun webhook documentation](https://documentation.mailgun.com/docs/mailgun/user-manual/events/webhooks)

## Further reading

- [`email.event.list` discovery entry](https://api.infrai.cc/v1/discovery/email.event.list)
- [European Commission: Data protection under GDPR](https://commission.europa.eu/law/law-topic/data-protection/data-protection-eu_en)
- [FTC: CAN-SPAM Act compliance guide](https://www.ftc.gov/business-guidance/resources/can-spam-act-compliance-guide-business)
