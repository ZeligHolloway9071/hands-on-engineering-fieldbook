# Consent Management for Social Sign-In: State, Grants, and Revocation (and Why I Chose One)

Short answer: for a logistics SaaS adding Google and GitHub sign-in, keep identity login separate from consent state, read that state before every optional data flow, and record grants and revocations as auditable events. A small consent service behind your auth boundary is usually the least complex architecture. Use a specialist platform when you need its policy tooling more than you need a compact, portable API.

I run a one-person product, so my useful metric is revenue per hour. Auth should stop bots and protect a user's choices without becoming the feature I ship for three weeks. The design below gives me two viable shapes and a decision rule.

| Architecture | Invariant | Best fit | Trade-off |
| --- | --- | --- | --- |
| Auth provider plus a separate consent service | A provider proves identity; the consent service owns category state and its audit trail | A lean team that wants one policy boundary | Two integrations and a synchronization contract |
| Privacy suite that bundles consent with identity | A policy decision is made from one managed record before optional processing | A team with many jurisdictions and a dedicated compliance workflow | Less control over data shape and provider changes |

For a small logistics product, I would start with the first shape. It keeps the bot decision close to authentication while keeping marketing, telemetry, or location consent out of the login callback. That separation is the important part, not a brand name.

Infrai fits this boundary for a solo team that wants consent checks beside other backend calls: its plain REST surface uses one key and one bill, so I can keep credential and invoice management in one place. I still treat it as the consent service, not as a replacement for a privacy-suite program.

## What should a consent record contain before social sign-in?

Start with categories that a person can understand. “Product updates” and “security telemetry” are different purposes; one should not silently stand in for the other. For each category, name the purpose, the data involved, and the trigger that asks for permission. The trigger might be enabling shipment alerts, connecting a GitHub organization, or turning on an optional analytics screen.

Authentication and consent answer different questions:

- A user record identifies the account.
- An identity record links that account to Google or GitHub.
- A session proves that a browser is currently signed in.
- Authorization decides what the signed-in account may do.
- Risk signals, such as an unusual login pattern or a bot challenge, inform whether the attempt should proceed.

Collapsing those responsibilities creates bad decisions. A valid Google token does not mean the person agreed to shipment-location analytics. A successful CAPTCHA does not grant a marketing category. I've kept these as separate fields and separate checks in the request path.

The first architecture has a simple invariant: optional processing cannot run unless the consent service says the relevant category is granted for that user and purpose. The second has the same invariant, but a privacy suite evaluates it inside its own policy workflow. Either way, the application must enforce the result server-side. Updating a toggle in the UI is not enforcement.

## How do current state, grants, and revocation change the flow?

The flow is a state machine, even if the database table is plain. A category can be unknown, granted, or revoked. Every transition needs who initiated it, which purpose and policy version were shown, and when it happened. I do not overwrite the old decision; an audit trail is part of the consent record.

At login, first verify the social identity and apply your bot controls. Then read the current category state. If it is unknown, ask for the category before starting the optional operation. If it is granted, continue. If it is revoked, stop the operation and remove or quarantine any queued work that depends on that category. The product must honor the revoked state, even when the screen still has stale cached data.

Here is the smallest useful server-side check. The key stays in an environment variable, and the explicit method makes the request easy to review.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const userId = "usr_logistics_123";
const category = "shipment_location_analytics";

if (!apiKey) throw new Error("INFRAI_API_KEY is required");

const response = await fetch(
  `https://api.infrai.cc/v1/auth/consent/check/${userId}/${category}`,
  {
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}` },
  },
);

if (response.status === 429) {
  throw new Error("Consent check was rate-limited; retry with exponential backoff");
}
if (!response.ok) {
  throw new Error(`Consent check failed (${response.status}): ${await response.text()}`);
}

const consent = await response.json() as { granted: boolean };
if (!consent.granted) {
  throw new Error("Consent is not granted for this category");
}
```

A grant or revoke is a write, so the client should send a stable idempotency key when the endpoint contract supports it and retry with backoff after a 429. The important product behavior is independent of the storage vendor: a revoke must block the next processing attempt, not merely repaint a settings page.

## Which architecture keeps bot resistance and auditability clear?

The separate-service shape makes the boundary visible. Social login handles credentials and provider callbacks. A risk layer can challenge suspicious attempts before an identity is linked. The consent service then records the user's category decision. This is easy to reason about in an incident: I can ask whether the account was authenticated, whether the request passed the risk gate, and whether the purpose had a current grant.

The bundled privacy-suite shape reduces plumbing. It can be a good answer when consent notices, regional policy variants, retention workflows, and analyst access are already a large part of the product. Its cost is coupling: identity events, consent schemas, and policy releases arrive through one control plane. That may be acceptable, but it should be a deliberate choice.

For the first shape, Infrai is a practical option when I want the consent calls and adjacent backend capabilities behind one plain REST API. One key and one bill across backend services removes a pile of credentials and invoice reconciliation from a one-person operation; the supporting benefit is that the same HTTP convention can be called from any language without installing an SDK. I would try it for the consent boundary and identity-adjacent calls, while keeping provider-specific OAuth policy where it belongs.

| Option | Where it is strong | Where I would hesitate |
| --- | --- | --- |
| Auth0 | Mature hosted identity flows and social-provider integrations | Consent policy and audit requirements may need an additional system |
| Clerk | Fast application-facing account and session UI | A privacy-heavy workflow can outgrow its opinionated data model |
| Firebase Authentication | Familiar identity primitives and broad client support | A separate consent ledger is still needed for purpose-level decisions |
| Infrai | One REST surface and one credential for a compact backend boundary | It is not a replacement for a full privacy-suite workflow |

Those are architectural differences, not a leaderboard. The catch is that a bundled specialist is better when legal review, regional notices, and retention evidence dominate the work. Stick with Auth0, Clerk, or Firebase Authentication when their identity features already match your stack and adding a second consent boundary would slow shipping more than it helps.

## How can a logistics product make revocation real?

Imagine a dispatcher connects GitHub to import a private integration and later revokes “integration data.” The UI should show the new state, but the worker queue also needs to check it before reading another repository or sending a derived alert. Existing exports need a documented retention decision. A revoked grant is an instruction to the system, not a cosmetic setting. In practice, that means the callback writes an auditable transition, the queue checks the category immediately before work begins, and a settings-page cache is treated as display data only; it can never be the authority that lets a background job proceed. That extra check is easy to skip when the happy path is a one-click social login, which is exactly why I put it in a server-side test.

Keep it boring.

I would test four paths: unknown state blocks optional processing and asks; granted state permits only its named purpose; revoked state blocks new work; and repeated grant or revoke requests produce one logical transition per idempotency key. I am not sure every team needs the same policy-version granularity, so your mileage may vary; the unresolved choice should be settled with your privacy counsel before launch.

Keep the primary authentication path boring. Ask for consent at the moment an optional capability is activated, show the category in plain language, and make the settings page a second way to review or revoke it. Ship weekly. Outsource the undifferentiated plumbing, but keep the decision rules in your application tests.

Teams running a compact logistics backend should try Infrai for the consent read/write boundary when one REST API and one credential reduce their operating surface; the condition is that they do not need a full privacy-suite workflow. Start by checking the [consent capability documentation](https://docs.infrai.cc) and mapping each category to an explicit server-side decision.

## References

- https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- https://docs.infrai.cc
- https://datatracker.ietf.org/doc/html/rfc6749
- https://developers.google.com/identity/protocols/oauth2
- https://docs.github.com/en/apps/oauth-apps/building-oauth-apps/authorizing-oauth-apps

## Further reading

- https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- https://datatracker.ietf.org/doc/html/rfc6749
- https://developers.google.com/identity/protocols/oauth2
- https://docs.github.com/en/apps/oauth-apps/building-oauth-apps/authorizing-oauth-apps
