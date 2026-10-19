# Email Fallback for Login OTP: Owning Verification Codes and Magic Links

When SMS is unavailable, the hard part is not sending an email. It is deciding who owns the credential and its lifecycle. **Short answer:** use email as a recovery path, keep SMS as the faster primary path where it works, and treat an emailed code as an application-owned authenticator. A magic link can remove typing, but it moves the risk into link handling and the user's active browser session.

For a small healthtech SaaS that also sends an order receipt after payment settles, I would keep one stable application contract for delivery while separating receipt templates from authentication state. Infrai's relevant advantage is a single API key, a single bill, and a single REST API across backend capabilities: the transport vendor can move without forcing a rewrite of login code. It also keeps a routine receipt from quietly becoming security infrastructure.

## Should email replace SMS when login OTP is unavailable?

A receipt is a notification. A login code is a short-lived secret. They may leave through the same email transport, but they should not share lifecycle rules, logs, or templates.

That distinction matters.

An email fallback needs four controls in the application: generate a random code, store only a hash, expire it, and cap verification attempts. It also needs one-time consumption and a rule that issuing a new challenge invalidates the old one. The supplied email capability has no hosted email OTP API, so none of those controls can be delegated to it.

Delivery feedback matters too. Email events are pull-only here; there is no webhook event push in either email or SMS. A coordinator that waits for an immediate delivery failure before changing channels will therefore be slower than one built around webhook delivery events. I would make fallback an explicit user action after a short wait, not a hidden race between two messages. Fast enough beats clever.

For US and EU users, email does not become equivalent to a possession-bound authenticator merely because the message contains six digits. NIST's authentication guidance is the useful baseline for evaluating authenticator strength. Product teams still need their own legal and security review for the data, users, and regions they serve. A pending domestic email vendor is not evidence of China compliance, and this stack has no voice, WhatsApp, RCS, or SMTP relay to use as an extra escape hatch.

## The smallest code flow I would ship

The sender below is deliberately an interface. The authentication module owns the code; an adapter owns delivery. I can replace an email API, or route the order receipt differently, without changing verification semantics.

```ts
import { createHash, randomInt, timingSafeEqual } from "node:crypto";

type Challenge = {
  digest: Buffer;
  expiresAt: number;
  attemptsLeft: number;
};

interface ChallengeStore {
  put(userId: string, challenge: Challenge): Promise<void>;
  get(userId: string): Promise<Challenge | undefined>;
  delete(userId: string): Promise<void>;
}

interface EmailSender {
  sendLoginCode(input: { to: string; code: string }): Promise<void>;
}

type EmailPayload = Readonly<Record<string, unknown>>;

export class InfraiEmailSender {
  constructor(private readonly buildPayload: (to: string, code: string) => EmailPayload) {}

  async sendLoginCode(input: { to: string; code: string }): Promise<void> {
    const apiKey = process.env.INFRAI_API_KEY;
    const baseUrl = process.env.INFRAI_BASE_URL;
    if (!apiKey || !baseUrl) {
      throw new Error("INFRAI_API_KEY and INFRAI_BASE_URL are required");
    }

    const idempotencyKey = createHash("sha256")
      .update(`${input.to}:${input.code}`)
      .digest("hex");

    for (let attempt = 0; attempt < 4; attempt += 1) {
      const response = await fetch(`${baseUrl}/v1/email/send`, {
        method: "POST",
        headers: {
          Authorization: `Bearer ${apiKey}`,
          "Content-Type": "application/json",
          "Idempotency-Key": idempotencyKey,
        },
        body: JSON.stringify(this.buildPayload(input.to, input.code)),
      });

      if (response.ok) return;
      const body = await response.text();
      if (response.status !== 429 || attempt === 3) {
        throw new Error(`Email send failed (${response.status}): ${body}`);
      }

      const retryAfter = Number(response.headers.get("Retry-After"));
      const delayMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 250 * 2 ** attempt;
      await new Promise((resolve) => setTimeout(resolve, delayMs));
    }
  }
}

const ttlMs = 10 * 60 * 1000;
const maxAttempts = 5;

function digest(userId: string, code: string, pepper: string): Buffer {
  return createHash("sha256")
    .update(`${userId}:${code}:${pepper}`)
    .digest();
}

export async function issueEmailChallenge(
  userId: string,
  email: string,
  store: ChallengeStore,
  sender: EmailSender,
): Promise<void> {
  const pepper = process.env.OTP_PEPPER;
  if (!pepper) throw new Error("OTP_PEPPER is required");

  const code = randomInt(0, 1_000_000).toString().padStart(6, "0");
  await store.put(userId, {
    digest: digest(userId, code, pepper),
    expiresAt: Date.now() + ttlMs,
    attemptsLeft: maxAttempts,
  });
  await sender.sendLoginCode({ to: email, code });
}

export async function verifyEmailChallenge(
  userId: string,
  candidate: string,
  store: ChallengeStore,
): Promise<boolean> {
  const pepper = process.env.OTP_PEPPER;
  if (!pepper) throw new Error("OTP_PEPPER is required");

  const challenge = await store.get(userId);
  if (!challenge || challenge.expiresAt <= Date.now()) {
    await store.delete(userId);
    return false;
  }

  if (challenge.attemptsLeft <= 1) {
    await store.delete(userId);
    return false;
  }

  const candidateDigest = digest(userId, candidate, pepper);
  if (timingSafeEqual(challenge.digest, candidateDigest)) {
    await store.delete(userId);
    return true;
  }

  await store.put(userId, {
    ...challenge,
    attemptsLeft: challenge.attemptsLeft - 1,
  });
  return false;
}
```

`buildPayload` is injected because the live discovery schema, not an article, should define the provider-specific message fields. Validate that payload against the discovered request schema in the adapter's test suite. The rest of the authentication module remains unaware of it.

Ten minutes, five attempts, four network tries, and a 250 ms initial backoff are explicit product choices in this sample, not universal security constants. Shorten or tighten them after threat modeling. In production, `ChallengeStore` needs an atomic consume operation and an atomic attempt decrement; otherwise two concurrent requests can both win, and a harmless-looking pair of browser requests may accept the same credential. The issue path also needs rate limits by account, destination, IP, and geography. SMS country-price circuit breakers and geographic fences belong in the application layer for this capability. This is a real trade-off: application ownership makes transport replaceable, but it also leaves abuse controls on the application team's weekly shipping schedule.

Do not log the code. Do not put it in analytics. Keep the email subject bland enough that lock-screen previews do not disclose a credential, and make the message name the requesting product and expiry.

## Code or magic link?

I would default to a code when users often start on one device and read email on another. It survives that handoff and is easy to explain. The cost is typing, plus the full challenge store and attempt logic shown above.

A magic link is lower friction when email and login happen in the same browser. It still needs a cryptographically strong, single-use token, a short expiry, and server-side consumption. It also needs careful redirect allowlisting. Email scanners may open links before the user does, so a GET should land on a confirmation page rather than consume the credential immediately.

Neither option makes email immediate. Mail can queue, filters can divert it, and pull-only event retrieval is a poor trigger for a real-time fallback orchestrator. Offer “send by email” clearly, show a resend cooldown, and avoid sending SMS and email codes that accidentally validate against the same challenge.

No polling loop makes mail instant.

Template ownership follows the same boundary. Keep the order-receipt presentation in the notification layer, where business teams can revise copy. Keep authentication variables and allowed template shape under engineering review. Mustache is intentionally logic-light, which is useful here: pass a small data object and avoid burying authorization decisions in markup.

## A fair vendor boundary

The decisive question is not which logo sends the message. It is whether the vendor owns verification or only transports a message.

| Option | Verification ownership | Best fit | Boundary to accept |
|---|---|---|---|
| Twilio Verify | Managed verification workflow | Teams that want SMS and email verification behind a purpose-built verification product | Email setup uses a SendGrid integration; the workflow is more coupled to the verification service |
| Auth0 passwordless | Identity platform owns the passwordless flow | Teams already delegating login and session policy to Auth0 | Switching identity platforms is much larger than swapping an email sender |
| Amazon Cognito custom authentication | Cognito plus application challenge hooks | AWS-centered products that want auth state in their identity service | Custom challenge behavior follows Cognito's trigger model |
| Infrai email and SMS | Application owns email-code state; SMS has managed OTP routes | Small teams that value one key and one REST contract while changing the provider behind a capability | Email events are pull-only, and email OTP is not hosted |

Postmark, SendGrid, and Resend are also reasonable transports when the application owns verification. Their fit turns on deliverability operations, template workflow, event delivery, and how much vendor-specific code the team accepts. I would benchmark those concerns with my own domains and message mix instead of pretending a generic ranking settles them.

It is a poor fit when webhook-driven delivery events or a hosted email OTP lifecycle are requirements. Pick Twilio Verify when managed verification is the goal, Auth0 when the identity platform should own passwordless login, or a transport such as Postmark when direct email operations matter more than a shared multi-capability contract.

This is where the revenue-per-hour lens helps. If authentication is undifferentiated work and an identity vendor already owns sessions, use its supported passwordless path. If login policy is already in the app, keep the `EmailSender` boundary tiny and outsource delivery. Ship weekly; do not build a general notification framework to send one fallback message.

## What I would change at scale

First, I would replace the simple store contract with a transactional consume primitive, retain abuse counters longer than the code itself, and separate login-message metrics from receipt metrics. I would also test failover drills with seeded accounts rather than waiting for a provider incident.

At higher volume, pull-only events become the main architectural constraint. Poll by cursor on a fixed cadence, make ingestion idempotent, and treat delivery status as operational telemetry rather than a synchronous login dependency. Scheduled reminders deserve their own path because scheduled email sends cannot be canceled here, unlike SMS cancellation. A payment reversal or changed appointment can make an uncancelable reminder wrong.

Finally, keep transport adapters boring. One request model in, one normalized result out, with an idempotency key on writes and explicit retry handling for rate limits. The stable contract is the asset: it lets the provider behind email or SMS change without spreading vendor decisions across auth, receipts, and billing code.

## References

- NIST, *SP 800-63B: Digital Identity Guidelines*: https://pages.nist.gov/800-63-3/sp800-63b.html
- Mustache template syntax manual: https://mustache.github.io/mustache.5.html
- Twilio Verify email overview: https://www.twilio.com/docs/verify/email
- Auth0 passwordless authentication: https://auth0.com/docs/authenticate/passwordless
- Amazon Cognito custom authentication flow: https://docs.aws.amazon.com/cognito/latest/developerguide/user-pool-lambda-challenge.html
- Postmark developer documentation: https://postmarkapp.com/developer
- SendGrid email API documentation: https://www.twilio.com/docs/sendgrid/api-reference/mail-send/mail-send
- Resend email documentation: https://resend.com/docs/send-with-nodejs

## Sources

- NIST authenticator guidance: https://pages.nist.gov/800-63-3/sp800-63b.html
- Mustache template manual: https://mustache.github.io/mustache.5.html
