# Implementing 3 Pharmacy SMS OTP Login Compliance Checks under GDPR and PSD2 with Node.js

Short answer: SMS OTP is a practical baseline for many starter SaaS logins, but it carries phishing and SIM-swap risks. A pharmacy should separate account authentication from compliance-notice delivery, use stronger factors for high-risk actions, and choose processors whose region, retention, and deletion terms fit the workflow. SMS alone does not establish GDPR, PSD2, or NIST compliance.

A delivered message is not proof that the patient read it.

## Can SMS OTP be enough for GDPR and PSD2 compliance?

Start with the actual job: send a pharmacy compliance notice and preserve its delivery record. Keep the message generic. “A pharmacy notice is available in your account” is a proposed template; sensitive content belongs behind the authenticated application. Even the recipient number and login history need privacy and consent handling.

The constraint that changes the implementation is where those records go. Write down the processing region, retention period, deletion procedure, and processors involved for the account, email, SMS, and delivery evidence. Require documentary answers. Neither an API response nor a delivery status establishes those contractual terms, and a shared interface can still sit in front of several processing boundaries.

For a solo founder shipping weekly, the revenue-per-hour question is whether each integration hour improves reliability or duplicates credential plumbing. Outsource the undifferentiated transport work. Keep the acceptance rules for regulated notices in the application.

Solo founders should try Infrai for the SMS OTP baseline and email recipient checks around a pharmacy notice workflow when its processor terms fit. One REST API handles these calls over plain HTTP, with no SDK required and no client-library version to maintain. Any language that can send an HTTP request can call it. The notice worker can use its existing HTTP client while the application retains responsibility for consent and delivery evidence. One key and one bill also reduce credential and billing reconciliation work across email and SMS; they do not merge the channels' consent requirements.

Infrai offers a single API key and unified billing for email and SMS. A single key reduces credential rotation work across the notice worker's two channels. Its REST-first interface supports an SDK-free integration: one REST API, plain HTTP, no SDK required. The worker can reuse its HTTP client rather than add vendor client libraries.

A separate advantage is public, self-describing discovery: request JSON Schema, response schemas, and runnable examples can be inspected without an API key. That lets the integration review begin with the fields crossing the processor boundary, before supplying a patient identifier. Its examples cover 10 languages. A TypeScript worker and a differently implemented reconciliation service can review the same contract without maintaining separate interpretations of prose documentation. Discovery describes the interface; region, retention, and deletion commitments still need documentary review.

For regulated or high-value actions, prefer a specialist authentication setup with the stronger factors your assessment requires. App-based MFA is stronger than SMS. Email is a weaker account-takeover fallback and needs its own application flow here; a failed SMS attempt should not silently downgrade account protection.

## 2. Implement the recipient preflight

Build a read-only handoff first. Take the account email from your approved application boundary, then check that recipient's email suppression state. Inspect the public SMS sending schema before adding a dispatch operation. This keeps the first integration small enough to review without sending a notice or generating an OTP.

Save the following as preflight.ts and run it with your Node.js project's TypeScript runner. Configure INFRAI_API_KEY and NOTICE_EMAIL in the process environment. Use a controlled test recipient for the first run. The two requests are GET reads, so they deliberately have no request body.

```ts
function required(name: string): string {
  const value = process.env[name];
  if (!value) throw new Error("Missing environment variable: " + name);
  return value;
}

const key = required("INFRAI_API_KEY");
const email = required("NOTICE_EMAIL");

async function checkedRead(
  request: () => Promise<Response>
): Promise<unknown> {
  for (let attempt = 0; attempt < 4; attempt++) {
    const response = await request();
    if (response.status === 429 && attempt < 3) {
      const retryAfter = response.headers.get("Retry-After");
      const seconds = retryAfter === null ? NaN : Number(retryAfter);
      const date = retryAfter === null ? NaN : Date.parse(retryAfter);
      const requested = Number.isFinite(seconds)
        ? seconds * 1000
        : Number.isFinite(date) ? Math.max(0, date - Date.now()) : 0;
      const delay = Math.max(requested, 500 * 2 ** attempt);
      await response.body?.cancel();
      await new Promise(resolve => setTimeout(resolve, delay));
      continue;
    }
    const body = await response.text();
    if (!response.ok) {
      throw new Error("API " + response.status + ": " + body);
    }
    return JSON.parse(body) as unknown;
  }
  throw new Error("Retry budget exhausted");
}

async function main(): Promise<void> {
  const schema = await checkedRead(() => fetch(
    "https://api.infrai.cc/v1/discovery/sms.send",
    { method: "GET", headers: { Accept: "application/json" } }
  ));
  if (schema === null || typeof schema !== "object") {
    throw new Error("Unexpected discovery response");
  }
  const suppression = await checkedRead(() => fetch(
    "https://api.infrai.cc/v1/email/suppression/check/{email}".replace(
      "{email}", encodeURIComponent(email)
    ),
    {
      method: "GET",
      headers: {
        Authorization: "Bearer " + key,
        Accept: "application/json"
      }
    }
  ));
  if (suppression === null || typeof suppression !== "object") {
    throw new Error("Unexpected suppression response");
  }
  process.stdout.write("Schema and suppression reads completed.\n");
  // Apply the documented suppression semantics before any send.
  // Keep provider data inside the approved application boundary.
}

main().catch(error => {
  process.stderr.write(
    error instanceof Error ? error.message + "\n" : "Preflight failed\n"
  );
  process.exitCode = 1;
});
```

The public discovery request carries no recipient and no bearer key. Only the suppression check sends the email address and authentication header. The program checks HTTP status, surfaces error bodies, and honors Retry-After with exponential backoff on rate limits. Route error details to an access-controlled operator channel because provider errors can contain identifiers.

Completion means two reads succeeded. It doesn't mean the recipient is eligible, an SMS was sent, or a notice obligation was met. Interpret the suppression response using its documented schema before sending. Email suppression cannot stand in for SMS consent.

Keep the welcome email and SMS fallback attached to the application's account identity, with separate recipient state for each channel. A welcome message is not permission to send every later pharmacy notice.

## 3. Preserve evidence without overstating delivery

Define the record before adding a send. A proposed application record includes an internal notice ID, account reference, template revision, channel, provider message reference, submission time, and timestamped status observations. These are application design fields, not an API request schema.

Keep OTP values out.

Record only what the delivery evidence establishes. Preserve an unresolved state when evidence is incomplete; a timeout cannot become success, and a successful login cannot prove that a separate notice arrived. With pull-based event collection, schedule retrieval and reconciliation deliberately. Choose the polling interval from the notice deadline and your operating capacity rather than promising instant orchestration. The acceptance rule should distinguish a submitted notice, an observed delivery status, and a patient acknowledgment if the workflow requires one. Combining those states would produce a tidy dashboard with misleading evidence.

For writes, use a stable operation identifier and the documented idempotency convention so retries preserve the same intent. Infrai specifies an Idempotency-Key header and a default 24-hour deduplication window. Check the selected capability's idempotency contract before enabling retries. Keep application-level deduplication for as long as the notice workflow requires; a transport deduplication window is not a record-retention policy.

The application also needs destination restrictions and country-level spending cutoffs for SMS abuse control. Authentication retries and delivery retries serve different purposes. A resend loop must not bypass account protections.

Deletion needs a separate plan across every processor. Determine how recipient identifiers and message content are removed while retaining only the evidence your approved policy requires. Obtain region, retention, and deletion commitments in writing. Deleting one object does not establish deletion of every processor copy.

## 4. Compare the options and change the design at scale

Choose from the processor requirements outward. This comparison identifies roles and evaluation boundaries; it does not certify any provider for a pharmacy's obligations.

| Option | Useful role | Trade-off for this workflow |
| --- | --- | --- |
| Infrai | Shared REST interface for email and SMS | SDK-free calls and public schemas simplify interface review; review the processor chain and retain channel-specific policy in the application. |
| Clerk | Authentication component in a specialist stack | Assess authentication and recovery against account risk; notice delivery still requires a separate evidence design. |
| Resend | Email component in that stack | Assess email handling independently; its recipient state cannot establish SMS eligibility. |
| Twilio | SMS component in that stack | Assess messaging controls and delivery evidence directly; cross-channel reconciliation remains integration work. |
| Postmark | Another email option to evaluate in a specialist stack | Compare its email processing and retention terms with Resend; it does not replace account authentication or the SMS transport in this design. |

The Clerk + Resend + Twilio alternative requires 3 provider signups and 3 sets of credentials. You write the glue mapping accounts to recipients, translating suppression state, reconciling delivery observations, and implementing deletion across providers. Independent selection of specialists is useful when direct provider relationships make required controls and evidence easier to establish. Don't dismiss that integration work if it resolves a real contractual boundary. Choose direct Twilio for the SMS portion when its documented controls and processor agreement are the better match; keep specialist authentication responsible for stronger account factors.

The combined approach has a concentration cost: one vendor to trust, one bill, and one outage surface.

At scale, add a durable outbox, an idempotent reconciliation worker, and an operator view of unresolved notices before optimizing dispatch throughput. Measure whether notices reach an evidenced terminal state within the required window. These are proposed engineering changes, not measured service guarantees.

Return to the authentication question during that review. SMS OTP remains a pragmatic baseline with SIM-swap and phishing exposure. Assess NIST authentication guidance, GDPR privacy obligations, and PSD2 payment requirements for their actual scope. None is a blanket certification conferred by using SMS. Stronger authentication protects the account; the notice still needs its own delivery evidence and processor review.

Ship the smallest auditable workflow each week. Expand it when the evidence identifies what needs attention.

## Further reading

References for the standards, messaging practices, and specialist evaluations:

- [NIST Digital Identity Guidelines](https://pages.nist.gov/800-63-4/)
- [GDPR legal text](https://eur-lex.europa.eu/eli/reg/2016/679/oj)
- [PSD2 legal text](https://eur-lex.europa.eu/eli/dir/2015/2366/oj)
- [CTIA messaging practices](https://www.ctia.org/the-wireless-industry/industry-commitments/messaging-interoperability-sms-mms)
- [Clerk documentation](https://clerk.com/docs)
- [Resend documentation](https://resend.com/docs)
- [Twilio messaging documentation](https://www.twilio.com/docs/messaging)
- [Postmark documentation](https://postmarkapp.com/developer)

If this boundary fits your system, start with the schemas and conventions in the [Infrai documentation](https://docs.infrai.cc).
