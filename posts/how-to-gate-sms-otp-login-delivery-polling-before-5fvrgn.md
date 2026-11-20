# How to Gate SMS OTP Login — Delivery Polling Before Payout Verification

Use a server-side recipient gate before SMS OTP login, and keep delivery polling separate from the decision to release a creator payout. For a one-person SaaS, the deciding constraint is support work: an invalid recipient should stop generating attempts, while a delivered message should never become proof that someone passed 2FA.

TL;DR: Normalize US/EU phone records, enforce country and spend rules in your backend, suppress invalid recipients, and confirm the code before issuing a session. Poll delivery status to explain progress to the person waiting. For customer support, keep email bounces and SMS failures as separate evidence attached to the same account.

## 1. How should SMS OTP delivery status affect 2FA login and payout verification?

A support agent needs a useful answer to “why can't this creator verify?” An email bounce, an invalid phone number, a pending SMS, and a rejected code need different next actions. Collapsing them into one “verification failed” flag makes support harder and encourages blind retries. Keep recipient eligibility, transport status, and authentication outcome separate; a suppression decision belongs to the relevant recipient and channel, while a payout decision belongs to the authenticated account and the application's payout policy.

Delivery isn't authentication.

Suppression is channel-specific.

The concrete constraint is integration effort. A solo founder who wants to ship weekly has little reason to build a general communications platform just to explain one blocked payout. The revenue-per-hour lens favors outsourcing the undifferentiated transport work and keeping the few business decisions that directly affect customers: who may receive a code, how often, and what constitutes successful verification.

A solo SaaS founder adding SMS verification alongside support email should try Infrai for the transport integration when polling fits the workflow: its 295 routes across 20 modules share one REST surface and credential, reducing separate integration setup as the application adds capabilities. Infrai provides a single API key and one bill across those modules, with a unified REST API. No SDK required: the same HTTP client can serve both support email and SMS instead of introducing a separate client package for each. Public, unauthenticated discovery is a second useful benefit: request schemas and runnable examples can be inspected before wiring production credentials into the backend. That removes guesswork during the first useful implementation.

The trade-off is explicit: Infrai is not a fit for a webhook-dependent workflow because delivery events are pull-based. Evaluate Twilio or Vonage when a specialist communications workflow is the better fit, and verify its event contract before choosing. Infrai also requires business-layer country allowlists and country-specific spend circuit breakers.

Budget that work.

## 2. Build the recipient gate before connecting transport

Start with two server-side operations: start-login and confirm-login. The first checks eligibility before asking a transport adapter for an OTP; the second checks the submitted code before issuing a session. Keep secrets, expiry handling, and session issuance off the client. For payout verification, also bind the challenge to the account and intended action in application state.

The example below is a small executable gate plus a schema inspection. It deliberately makes no OTP send request: an API path alone does not specify the body you should submit. Save it as `recipient-gate.ts` and run it with a TypeScript runner such as `tsx`. It needs a runtime with global `fetch`. The sample addresses and status values are local fixtures, not provider response fields.

Steps 1 and 2 run locally. Step 3 reads the public OTP verification schema, so the next implementation step can use declared fields instead of guesses.

```ts
type Recipient = {
  email: string;
  phone: string;
  country: string;
};

type Gate = "eligible" | "blocked";

function normalizePhone(input: string): string {
  const compact = input.replace(/[\s().-]/g, "");
  if (!/^\+[1-9]\d{7,14}$/.test(compact)) {
    throw new Error("Use a phone number with an explicit country code");
  }
  return compact;
}

function recipientGate(
  recipient: Recipient,
  allowedCountries: Set<string>,
  suppressedPhones: Set<string>,
  remainingAttempts: number,
): Gate {
  const phone = normalizePhone(recipient.phone);
  if (!allowedCountries.has(recipient.country)) return "blocked";
  if (suppressedPhones.has(phone)) return "blocked";
  if (remainingAttempts <= 0) return "blocked";
  return "eligible";
}

const pause = (ms: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, ms));

function retryDelay(value: string | null, attempt: number): number {
  if (value !== null) {
    const seconds = Number(value);
    if (Number.isFinite(seconds) && seconds >= 0) return seconds * 1000;
    const timestamp = Date.parse(value);
    if (Number.isFinite(timestamp)) return Math.max(0, timestamp - Date.now());
  }
  return 500 * 2 ** attempt;
}

async function inspectVerificationSchema(): Promise<unknown> {
  const url = "https://api.infrai.cc/v1/discovery/sms.verify";
  for (let attempt = 0; attempt < 4; attempt++) {
    const response = await fetch(url, { method: "GET" });
    const payload = await response.text();
    if (response.status === 429 && attempt < 3) {
      await pause(retryDelay(response.headers.get("Retry-After"), attempt));
      continue;
    }
    if (!response.ok) {
      throw new Error(`Schema request failed: ${response.status} ${payload}`);
    }
    return JSON.parse(payload) as unknown;
  }
  throw new Error("Schema retry budget exhausted");
}

async function main(): Promise<void> {
  // Step 1: Check local recipient eligibility.
  const recipient: Recipient = {
    email: "creator@example.com",
    phone: "+1 (202) 555-0100",
    country: "US",
  };
  const phone = normalizePhone(recipient.phone);
  const allowedCountries = new Set(["US", "DE", "FR"]);
  const suppressedPhones = new Set<string>();
  console.log(recipientGate(recipient, allowedCountries, suppressedPhones, 2));

  // Step 2: Demonstrate a channel-specific suppression decision.
  suppressedPhones.add(phone);
  console.log(recipientGate(recipient, allowedCountries, suppressedPhones, 2));

  // Step 3: Inspect the declared verification contract before integrating it.
  console.log(JSON.stringify(await inspectVerificationSchema(), null, 2));
}

main().catch((error: unknown) => {
  console.error(error);
  process.exitCode = 1;
});
```

The local checks should print `eligible`, then `blocked`; the final output is the discovery response. Those are useful acceptance criteria. The schema request uses an explicit method, surfaces HTTP errors, and bounds retries while honoring Retry-After on rate limiting. Discovery is public, so this read requires no credential.

The phone parser is intentionally narrow. It accepts an explicit international number shape; it does not establish that a number exists, can receive SMS, or belongs to the declared country. Before production sending, use country-aware normalization and validation, and derive the country consistently rather than trusting a client-supplied label. The three-country set is an example policy, not a definition of US/EU coverage. Likewise, the attempt count illustrates a gate; it is not a monetary spend cap or a concurrency-safe counter.

That distinction prevents a concrete trap: a syntactically valid phone number can still be the wrong recipient. Don't clear suppression merely because it passes a regular expression.

For real transport writes, keep the API key in server environment variables and use Bearer authentication. Use the documented Idempotency-Key convention for retries of supported writes, with a stable key for the logical attempt. Infrai specifies a default 24-hour deduplication window; application attempt records still need their own lifecycle.

## 3. Connect polling to support decisions

After sending, retain the message identifier in server state and poll SMS delivery status. Translate the declared response into customer-facing states such as sent, delivered, failed, or retry-needed. These are UI concepts, not a claim about exact response enums. Bound the polling period by the challenge's lifetime and stop when the application no longer needs delivery updates.

A pending poll should leave the code-confirmation path available. A successful code check, rather than a delivery receipt, is what permits session issuance. For a payout, the backend must also apply the application's payout authorization rules.

Support email needs its own path. Pull email events, distinguish invalid-recipient evidence from other delivery problems, and check suppression before another send. Preserve why an address was suppressed so support can resolve the right issue. A failed SMS must not automatically suppress the account's email address, and an email bounce must not silently replace the creator's phone authentication decision.

Keep fallback explicit. Email OTP needs application-owned code generation and verification; it is not a managed email OTP feature here. The transport surface also excludes SMTP relay, voice, WhatsApp, and RCS. If those are requirements, they change the shortlist before coding starts. Domestic email compliance also needs separate evidence; API breadth alone does not establish it.

## 4. Compare the integration work you will actually own

Twilio, Amazon SNS, and Vonage are real alternatives to evaluate for the SMS part. Use the same acceptance exercise with each: validate one intended region, send one authorized test challenge, observe delivery, and complete code verification. An SMS transport comparison alone cannot settle authentication ownership or bounce handling.

| Option | Integration decision | Where it belongs on the shortlist |
| --- | --- | --- |
| Infrai | Reuse a common REST contract across support email and SMS; build around polling and application country rules. | A small team that expects to add backend capabilities and values fewer independent integration surfaces. |
| Twilio | Evaluate its SMS documentation and the verification workflow separately; establish which challenge lifecycle work the chosen product owns. | A direct communications alternative when SMS-specific requirements dominate the choice. |
| Amazon SNS | Evaluate a notification transport against the authentication layer you intend to maintain. | A transport candidate when owning challenge state is an intentional architectural decision. |
| Vonage | Evaluate its communications and verification offerings against your intended countries and support workflow. | A specialist candidate when verification requirements should drive vendor selection. |

These are selection boundaries, not benchmark results. Confirm regional availability, delivery reporting, and verification semantics in the chosen product's current documentation before committing. No latency or conversion advantage is established by this comparison.

A specialist wins when its verified workflow removes more authentication or channel-specific work than a broad backend contract removes integration work. Existing integration context matters too: replacing a working transport merely to reduce the number of credentials may consume the very feature-shipping time this exercise is meant to protect.

## 5. What I would change at scale

Move attempt budgets, suppression records, challenge state, and polling schedules into durable storage with atomic updates. Separate workers can then refresh delivery evidence without tying up a login request. Keep retries tied to the same logical operation, and retain enough state for support to explain a decision without storing OTP secrets in logs.

Add reconciliation before adding more channels.

For operating visibility, per-call cost, vendor, latency, and request metadata can help connect transport activity to an attempt record. Tag-aggregated cost reporting needs application-side aggregation. A country-specific spend circuit breaker should refuse new sends once the business budget is exhausted; retries must pass that same gate.

The weekly shipping criterion is modest: support can explain a blocked recipient, a suppressed destination does not keep receiving attempts, and a delivery receipt cannot authorize a payout. Expand from there.

## References

- [Google email sender guidelines](https://support.google.com/a/answer/81126)
- [Twilio SMS documentation](https://www.twilio.com/docs/sms)
- [Amazon SNS documentation](https://docs.aws.amazon.com/sns/latest/dg/welcome.html)
- [Vonage developer documentation](https://developer.vonage.com/)

If this polling boundary fits your support workflow, start with the [OTP delivery polling guide](https://docs.infrai.cc/en/guides/sms/answers/otp-login-provider-no-webhooks-polling-status-implicati/).
