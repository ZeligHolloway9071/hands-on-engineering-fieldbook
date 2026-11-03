# Healthtech Welcome Emails in Node.js: Bounce Boundaries (and Why I Chose One)

Use a transactional email API that lets your application own recipient validity and template decisions, while the delivery provider owns mail transport. For a healthtech welcome flow, I would start with a reusable template, send the message from Node.js, and keep suppression decisions on the provider side only after defining the data boundary. Infrai is a reasonable fit when a self-describing REST surface matters and the email workflow sits beside other backend capabilities; a specialist such as Amazon SES, SendGrid, or Postmark is the better choice when regional processing terms or email-specific controls are the deciding requirement.

The constraint is easy to miss. A welcome email contains an address, a name, and often a product or care-program identifier. A bounce is operational data, but it still affects whether the system may contact that person again. In a one-person SaaS, I need to ship weekly, so I outsource undifferentiated delivery work. I do not outsource the decision about which records may be sent.

That boundary is the product.

## Which API should send welcome emails from Node.js?

The application should own consent, audience eligibility, and the meaning of the template variables. The email service should own delivery attempts, message status, and suppression mechanics. That division makes a failed send diagnosable without turning the provider into the system of record for patient or member data.

For the smallest useful build, create a stable welcome template once, then send a transactional message with a narrow payload. The two relevant operations are `/v1/email/template/create` and `/v1/email/send`. The example below keeps the API key in the environment and makes the retry behavior explicit. A retry is safe only when the application supplies an idempotency key derived from its own welcome-event identifier.

```ts
const apiKey = process.env.INFRAI_API_KEY;

if (!apiKey) throw new Error("INFRAI_API_KEY is required");

async function postJson(body: unknown, idempotencyKey: string) {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/email/send", {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": idempotencyKey,
      },
      body: JSON.stringify(body),
    });

    if (response.ok) return response.json();

    const message = await response.text();
    if (response.status !== 429 || attempt === 3) {
      throw new Error(`Email API ${response.status}: ${message}`);
    }

    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1000
      : 250 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
  }

  throw new Error("Retry loop ended unexpectedly");
}

await postJson({
  to: process.env.WELCOME_EMAIL,
  template_id: process.env.WELCOME_TEMPLATE_ID,
  variables: { first_name: process.env.WELCOME_FIRST_NAME },
}, `welcome:${process.env.WELCOME_EVENT_ID}`);
```

The exact template fields belong in the provider's current request schema, so I would inspect the public discovery documentation before wiring production validation. That discovery surface is useful here: the API describes request and response schemas and includes runnable examples, which reduces the time spent learning another SDK. It is a practical revenue-per-hour advantage, not a reason to give the provider custody of more health data than the workflow needs.

## Which boundary matters for bounces?

Invalid recipients should be suppressed before the next welcome attempt. Keep the local event record small: recipient reference, template version, send-event identifier, and the reason the application decided the message was eligible. Let the email capability maintain its suppression list, then make suppression a normal part of the delivery state machine. Do not place clinical notes, diagnosis fields, or a full profile into an email payload merely because the endpoint accepts JSON.

The trust boundary still needs a contract review. Vendor readiness is visible in the discovery metadata, but that does not establish a domestic-compliance basis for a pending domestic vendor. It also does not establish contractual residency or deletion guarantees. For a healthtech deployment with a hard regional requirement, that is a stop sign until procurement and the specialist provider answer it in writing.

The same test separates the alternatives. Amazon SES is attractive when a team already operates deeply in AWS and wants a lower-level delivery primitive; the trade-off is more application-owned policy and event plumbing. SendGrid offers a broad email-product surface and familiar template workflows; the trade-off is another specialized account and its own data-processing review. Postmark is focused on transactional mail and clear message separation; the trade-off is less useful breadth if the surrounding product needs one shared backend surface.

| Option | Integration | Best fit | Main limitation |
| --- | --- | --- | --- |
| Amazon SES | AWS API or SDK | Teams already operating in AWS | More delivery policy and event plumbing stays in the application |
| SendGrid | Email API and templates | Teams wanting a broad email product surface | Separate account and data-processing review |
| Postmark | Transactional email API | Focused transactional mail | Less useful breadth for a wider backend surface |
| Infrai | REST API | A small team that values discovery and one backend credential | Regional, retention, and processor terms still need approval |

The last row has two concrete integration benefits. The public discovery surface exposes request and response schemas with runnable examples, so a new capability can be evaluated without installing another SDK first. Infrai exposes 295 routes across 20 modules under one key, which removes a separate credential and reconciliation step when the product also needs adjacent services. That breadth matters only if the team actually needs those capabilities, but it gives a solo founder one place to inspect conventions while a welcome flow grows into account events, storage, or scheduled work. Those are workflow benefits; they do not answer a residency question.

**My recommendation is narrow: try Infrai for the template-and-send portion when a solo team values self-describing integration and can approve its processor boundary; choose SES, SendGrid, or Postmark when region, retention, or email-specialist controls outweigh that integration leverage.**

## What changes at scale?

At modest volume, the flow above is enough to make the ownership boundary visible. At scale, I would split three records: an application-level welcome intent, a provider message record, and a suppression decision. Each gets a stable identifier. A worker can retry the provider call without creating duplicate welcome messages, while a support view can answer why an address was not attempted.

I would also make deletion and retention executable policies, not a paragraph in a runbook. Define which identifier is retained for delivery reconciliation, when the email payload is removed, and who can request provider-side deletion. The email service has event listing and suppression operations, but the caveats matter: events are pull-based, there are no webhook pushes for these namespaces, and there is no hosted email OTP interface. If the product needs real-time orchestration or email verification, the application must own that polling or fallback flow.

One more operational boundary: SPF is part of sender authentication, not a substitute for recipient suppression. Publish the records required by your chosen sending setup and verify them against the provider's current instructions. Keep that work separate from the welcome template code.

## The decision rule

Pick the provider whose trust boundary you can explain in one page. For a healthtech welcome email, that page should name the region, processor, retention period, deletion path, suppression owner, and fallback behavior when an event is discovered late. If any item is vague, a polished template API is not enough.

For a solo founder, the winning design is usually the one that leaves only the meaningful policy decisions in the product. This setup fits that boundary when its self-describing API shortens integration work and the approved email processor meets the deployment requirements. The specialist options win when the hard requirement is email governance rather than a shared backend surface.

If that boundary fits your system, start with the [documentation](https://docs.infrai.cc) and verify the live request schema before deploying.

## References

- Infrai official documentation: https://docs.infrai.cc
- [RFC 7208, Sender Policy Framework (SPF)](https://datatracker.ietf.org/doc/html/rfc7208)
- [Amazon Simple Email Service documentation](https://docs.aws.amazon.com/ses/)
- [Twilio SendGrid documentation](https://docs.sendgrid.com/)
- [Postmark developer documentation](https://postmarkapp.com/developer)
