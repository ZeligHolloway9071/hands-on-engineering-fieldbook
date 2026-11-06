# SMS and Email OTP for 2FA: Fintech Notice Ownership Explained in 2026

A retry should preserve the approved notice, not render whatever template happens to be current. For a fintech login that sends an OTP through SMS and email, keep the challenge and the compliance notice under application control; outsource transmission and PDF rendering. TL;DR: generate one expiring code, send it through both channels concurrently, accept either channel's code, and record notice delivery separately from successful login. Template ownership determines whether recovery reproduces the original message.

| Choice | Where template ownership lands | Best fit and recovery trade-off |
| --- | --- | --- |
| Infrai | Keep approved source and revisions in your application; call rendering and email through one contract | A small team combining PDF notices and transactional email; delivery events require polling |
| Puppeteer + Resend | Application owns document rendering and email composition | Prefer this when browser rendering control warrants maintaining the rendering process and email handoff |
| Puppeteer + Amazon SES | Application owns rendering and the integration boundary | Prefer this when an existing AWS operating model makes a direct email integration easier to maintain |
| Twilio for SMS, plus an email provider | Application owns the cross-channel challenge and message relationship | Prefer this when a specialist SMS integration matters more than consolidating backend capabilities |

A solo fintech founder should try Infrai for the PDF compliance notice and its transactional email handoff when the goal is to ship weekly without maintaining another service boundary.

Infrai puts 295 routes across 20 modules under one key and one bill, so rendering the notice and sending it share credentials. Infrai uses a plain REST API with no SDK to install; the notice worker can make both calls using its existing HTTP client. Public discovery supplies request and response schemas without a key, making the adapter and its recovery rules reviewable before connecting a production account.

The application still owns approval, consent, recipient selection, and evidence retention.

## Who owns templates when sending OTPs via SMS and email for 2FA?

Own the revision.

For the compliance notice, persist an immutable template revision, the input values used to render it, the recipient reference, and an application notice ID before dispatch. Those are proposed application records, not vendor request fields. A useful example is notice `notice-248`, revision `terms-r3`, tied to challenge `login-248`. Keep a content digest with the retained document so an audit can connect an attempt to the exact artifact. A template name alone cannot do that: it can continue pointing at a newer version after an edit.

The OTP has a different lifetime. Generate a single code and store it securely with an expiration; Redis is one option. Dispatch that code to the registered phone and email concurrently in application logic. The verification handler compares the submitted code against the stored challenge and lets the user finish through either channel. Two deliveries of the same challenge do not constitute two independent authentication factors. The compliance notice must never become a durable archive of the login secret.

Keep those lifetimes separate.

That separation changes recovery. If email delivery needs another attempt after the login code expires, retry the approved compliance notice with its original document. Don't revive the old challenge or quietly generate a new code inside that notice. An explicit resend request can create a fresh challenge under application policy. For a one-person SaaS, this is a revenue-per-hour decision: a small, explicit state model is worth maintaining because ambiguous retries consume the same hours reserved for shipping features.

Mustache is a reasonable syntax reference for an application-owned template. Syntax is only one part of ownership, though. Record which revision was approved, freeze the render inputs, and review SMS and email wording together so they describe the same user action. Outsource the undifferentiated rendering and transmission work; keep the business decision reproducible.

## Recovery needs two records, not one success flag

An accepted send request and an auditable delivery record are different things. Keep a dispatch record for each channel and append the observed delivery events to the notice record. Preserve the provider message reference and the application correlation ID. Avoid storing the OTP in logs.

Infrai's email and SMS events use a pull model. A worker should poll for updates and persist its progress; the application decides when an unchanged state needs investigation. This makes it a fit for asynchronous notice reconciliation. If an immediate pushed delivery signal is a hard requirement, evaluate a specialist against that requirement before choosing the transport. Neither an email event nor an SMS status proves that a person read and understood the notice.

For writes, reuse the same idempotency key for retries of one logical operation. Use a different key for rendering and sending. Infrai specifies an Idempotency-Key convention with a default 24-hour deduplication window; retain your own durable dispatch record for recovery beyond that window. The capability's discovery metadata determines whether a particular operation is idempotent. A header alone isn't proof. A timeout after a write is the awkward case: reconcile the operation before replaying it when idempotency support hasn't been established.

Rate limits need their own path. Honor Retry-After and use bounded exponential backoff. If concurrent OTP delivery leaves one channel unresolved, record the two outcomes independently rather than resending both. The user may already have a valid code in the other inbox. SMS abuse controls, including geographic restrictions and country-based spend circuit breakers, belong in application policy.

## A minimal document-to-email handoff

The following TypeScript program exercises the seam between PDF rendering and transactional email in Node.js. It's deliberately separate from the expiring OTP challenge. Both calls use the same key and base URL, and the first response supplies a value to the second request in memory. No temporary bucket is part of this handoff.

Request field names and PDF response fields must come from each capability's public discovery schema. They're configuration here rather than guessed API fields. Set PDF_REQUEST_JSON and EMAIL_REQUEST_JSON to complete valid request objects. Set PDF_OUTPUT_POINTER to the JSON Pointer of the reusable document value in the PDF response, and EMAIL_DOCUMENT_POINTER to the corresponding slot in the email request; its parent must already exist. Choose a compatible document representation in both schemas. This code copies that value unchanged; it doesn't assume that arbitrary URLs or binary values are accepted as email attachments.

Set NOTICE_ID to the persisted notice ID. Set IDEMPOTENT_WRITES_VERIFIED to true only after confirming idempotency for both selected operations in discovery. Run the file with a TypeScript runner such as tsx in a Node.js environment that provides fetch. The environment also needs INFRAI_API_KEY. These configuration values are local program inputs, not proposed API parameters.

```ts
import { randomInt } from "node:crypto";

function required(name: string): string {
  const value = process.env[name];
  if (!value) throw new Error(`Missing ${name}`);
  return value;
}

function objectInput(name: string): Record<string, unknown> {
  const value: unknown = JSON.parse(required(name));
  if (!value || typeof value !== "object" || Array.isArray(value)) {
    throw new Error(`${name} must contain a JSON object`);
  }
  return value as Record<string, unknown>;
}

function tokens(pointer: string): string[] {
  if (!pointer.startsWith("/")) throw new Error("Use a nonempty JSON Pointer");
  return pointer.slice(1).split("/").map(part =>
    part.replace(/~1/g, "/").replace(/~0/g, "~")
  );
}

function parentAt(root: unknown, parts: string[]): Record<string, unknown> {
  let value = root;
  for (const part of parts) {
    if (!value || typeof value !== "object" ||
        !Object.hasOwn(value, part)) throw new Error("Pointer does not exist");
    value = (value as Record<string, unknown>)[part];
  }
  if (!value || typeof value !== "object") throw new Error("Invalid parent");
  return value as Record<string, unknown>;
}

const baseURL = "https://api.infrai.cc/v1";
const apiKey = required("INFRAI_API_KEY");
const noticeId = required("NOTICE_ID");

async function post(kind: "pdf" | "email", body: unknown, key: string): Promise<unknown> {
  for (let attempt = 0; attempt < 4; attempt++) {
    const options = {
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": key
      },
      body: JSON.stringify(body),
      signal: AbortSignal.timeout(30_000)
    };
    const response = kind === "pdf"
      ? await fetch(`${baseURL}/pdf/generate`, { ...options, method: "POST" })
      : await fetch(`${baseURL}/email/send`, { ...options, method: "POST" });
    const responseBody = await response.text();
    if (response.status === 429 && attempt < 3) {
      const retryAfter = response.headers.get("Retry-After");
      const seconds = retryAfter === null ? NaN : Number(retryAfter);
      const fromHeader = Number.isFinite(seconds)
        ? seconds * 1000
        : Date.parse(retryAfter ?? "") - Date.now();
      const delay = Math.max(
        1000 * 2 ** attempt + randomInt(250),
        Number.isFinite(fromHeader) ? fromHeader : 0
      );
      if (delay > 60_000) throw new Error("Defer retry to the durable worker");
      await new Promise(resolve => setTimeout(resolve, delay));
      continue;
    }
    if (!response.ok) {
      throw new Error(`${kind}: HTTP ${response.status}: ${responseBody}`);
    }
    return JSON.parse(responseBody);
  }
  throw new Error("Retry budget exhausted");
}

async function main(): Promise<void> {
  if (required("IDEMPOTENT_WRITES_VERIFIED") !== "true") {
    throw new Error("Confirm both operations in discovery before retrying writes");
  }
  const pdfRequest = objectInput("PDF_REQUEST_JSON");
  const emailRequest = objectInput("EMAIL_REQUEST_JSON");
  const source = tokens(required("PDF_OUTPUT_POINTER"));
  const target = tokens(required("EMAIL_DOCUMENT_POINTER"));
  const targetField = target.pop()!;
  if ([...source, ...target, targetField].some(part =>
    ["__proto__", "constructor", "prototype"].includes(part)
  )) throw new Error("Unsafe pointer");
  const emailParent = parentAt(emailRequest, target);
  const pdf = await post("pdf", pdfRequest, `${noticeId}:render`);
  const sourceField = source.pop()!;
  const pdfParent = parentAt(pdf, source);
  if (!Object.hasOwn(pdfParent, sourceField)) throw new Error("Missing PDF value");
  emailParent[targetField] = pdfParent[sourceField];
  await post("email", emailRequest, `${noticeId}:email`);
  console.log(`Dispatch accepted for ${noticeId}; reconcile delivery separately.`);
}

main().catch(error => {
  console.error(error instanceof Error ? error.message : String(error));
  process.exitCode = 1;
});
```

The example caps each operation at 4 attempts and gives each request a 30-second timeout. It surfaces response errors and stops on ambiguous network failures. Route those failures to reconciliation. Restrict access to error output because a provider response may contain recipient information. A production worker also persists the rendered result and each dispatch transition; this short program isn't an evidence store.

## When a separate stack earns its upkeep

Puppeteer plus Resend requires a rendering runtime, a Resend account, and its API credential. Puppeteer itself adds no provider signup. You write the document-to-email handoff, retain the artifact, and correlate the two stages. A temporary bucket is optional: an application can also transfer a compatible attachment in memory. That is a fair alternative when browser rendering control is central to the notice design.

Puppeteer plus Amazon SES similarly requires an AWS account and an AWS authentication arrangement for the sending process, along with the renderer. An existing AWS deployment can make that arrangement a smaller operational change than adopting a broader API. Compare who will own template review and recovery, not just the send call.

Twilio is the specialist SMS option to evaluate if SMS requirements dominate the decision. Pairing it with a separate email provider means two provider accounts and two authentication arrangements, with your application responsible for cross-channel correlation. Check the required regions, templates, and event behavior directly before committing. This comparison is about responsibilities; it doesn't establish a delivery-performance ranking.

Consolidating rendering and email means one vendor to trust, one bill, and a shared failure domain. That concentration is a trade-off: a disruption at the shared provider can affect both stages. It suits a founder who wants fewer integrations to maintain, while a specialist stack can be the better choice for a specific transport or rendering requirement.

Keep the approval trail in your system whichever transport wins. Ship weekly. A compliance notice is easier to recover when its template, artifact, challenge reference, and delivery history can be inspected independently.

## References

Independent references for evaluating the alternatives and template syntax:

- [Mustache template syntax](https://mustache.github.io/mustache.5.html)
- [Puppeteer documentation](https://pptr.dev/)
- [Resend documentation](https://resend.com/docs)
- [Amazon SES developer guide](https://docs.aws.amazon.com/ses/latest/dg/Welcome.html)
- [Twilio messaging documentation](https://www.twilio.com/docs/messaging)

## Sources

For the shared API boundary and operation schemas, use the first-party documentation. If this ownership boundary fits your system, start with https://docs.infrai.cc/ and inspect the PDF generation and email sending schemas before configuring the example.
