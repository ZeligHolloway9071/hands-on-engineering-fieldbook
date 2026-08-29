# Node.js App Alerts Explained: Template Custody for Email, SMS, Webhook, and Polling APIs

Short answer: for a fintech compliance notice, keep the approved email and SMS template version inside the application, send through a replaceable API adapter, accept delivery events by webhook, and use polling only to reconcile unresolved attempts. The deciding factor is custody of the notice, not the size of a provider's feature list.

| Operating model | Who releases the words? | Delivery evidence path | Choose it when |
| --- | --- | --- | --- |
| App-owned template, webhook plus polling | Application release process | Fast callback with bounded reconciliation | A notice must be reproduced from retained application records |
| Provider-owned template, webhook | Provider-side publishing process | Fast callback | Copy changes must be independent of application deploys and the provider's export model meets the evidence policy |
| App-owned template, polling | Application release process | Scheduled status reads | Inbound callbacks are prohibited and delayed status is acceptable |

**Decision note:** the first model is the practical default for a one-person fintech SaaS. It outsources transport while preserving the artifact that defines the obligation. The catch is real: the application now owns rendering tests, version retention, and a small reconciliation worker. Ship weekly, but don't trade away the ability to show exactly what shipped.

This note does not treat a delivery receipt as proof that a person read a notice. Product, compliance, and counsel must define the required evidence and retention period. I'm not sure one retention rule can cover every jurisdiction or notice type; a written policy and legal review must settle that boundary.

## How should an event notifications provider comparison evaluate webhook and polling?

Template ownership is release governance in disguise. If the application stores only `template_id: notice-current`, a later audit depends on what that name resolves to then. A stronger attempt record freezes the template version, normalized inputs, rendered subject and body, destination reference, creation time, transport message identifier, and receipts observed after the send.

The distinction gets sharp during an ordinary edit. Suppose release `margin-policy-v7` is approved at 08:40 and account `acct_731` is selected at 09:00. The application renders the notice and submits it at 09:01. A callback arrives at 09:02, the same event is delivered again at 09:04, and a scheduled status read observes a later state at 09:10. At noon, an editor publishes v8 and the customer changes an address. Reconstructing the send from the current profile and current template would create a different artifact. The durable record is the v7 rendered content and its hash, the send-time destination reference, the provider message identifier, and both unique observations. A mutable delivery summary may advance, but it must not rewrite the original attempt.

Keep the snapshot.

This also makes channel parity testable. One fixture can render email and SMS from the same approved notice version, then assert that required policy identifiers and effective dates appear in both. SMS deserves a separate rendered-text check: GSM-7 and UCS-2 have different character limits, and concatenated messages have smaller per-segment limits. A smart quote can change the encoding and therefore segmentation. Test the final string rather than estimating from the source template.

Provider-owned templates are a valid runner-up when non-engineers need to release copy without an application deployment. Before choosing them, verify version retention, rendered previews, access control, approval history, and export behavior against the evidence policy. The operational trade is simple: editorial speed moves out of the code repository, while another configuration surface enters the controlled release process.

## A receipt-processing data model

Webhooks and polling should feed one receipt ledger. They are transports for observations, not competing definitions of delivery. Normalize both into the same internal event shape, deduplicate on a documented stable event identifier, retain late observations, and apply an explicit transition policy to the current projection.

Webhooks reduce status delay, but they add an inbound security boundary. Verify authenticity exactly as the transport documents, retain the raw request body when its signature scheme requires that body, record the event durably, and acknowledge it quickly. Don't make a callback perform unrelated customer-facing work before the receipt is stored.

Polling is the repair loop — intentionally narrow and boring. Select unresolved attempts, query them on a bounded schedule, pass the responses through the same normalizer, obey documented rate limits, and stop after a documented terminal state or retention boundary. This protects the audit trail during a callback deployment window without creating a second state machine.

Pure polling remains suitable when the network policy forbids inbound endpoints, volume is modest, and delayed status is acceptable. Webhook-only ingestion can fit low-consequence alerts where an occasional manual status check is allowed. Neither shortcut should silently inherit the requirements of a compliance notice just because all three jobs use an email API.

Acceptance is not delivery. Delivery is not reading.

## Governance questions before signing a contract

The first test is **evidence portability**. Can the adapter preserve a provider-independent attempt record, export the raw receipts required by policy, and map documented statuses without losing meaning? A dashboard is useful for operations, but it cannot be the only place where the relationship between an approved notice and a send attempt exists.

The second is **change control**. Identify who can edit a template, how an edit becomes active, whether an old version remains retrievable, and how a release is tied to approval. This criterion tends to eliminate more options than a long checkbox comparison because it follows the actual compliance job. Channel breadth, visual editors, and campaign features may matter elsewhere; they don't answer who owns the notice artifact.

Then run adapter contract tests. Submit one fixed template version, duplicate a receipt, reverse the order of two observations, replay an old event, and render a Unicode SMS fixture. Confirm that exactly one immutable attempt remains while the receipt ledger preserves distinct observations. Also test authentication failure at the adapter boundary without placing rejected input in the trusted processing path. These checks are small enough to run for every transport change, which matters when one person is balancing infrastructure work against feature revenue.

Your mileage may vary on status names, event identifiers, authentication rules, query windows, and retention because those are provider contracts rather than shared email or SMS semantics. Read the contract before writing the mapping. No guesses.

## A focused evaluation harness in TypeScript

The application boundary below keeps template custody and receipt ingestion independent from any commercial SDK. The adapter may use an email or SMS API internally, but business code sees stable types.

```ts
import { createHash } from "node:crypto";

type Channel = "email" | "sms";
type DeliveryState =
  | "queued"
  | "accepted"
  | "delivered"
  | "undeliverable"
  | "unknown";

type NoticeAttempt = {
  attemptId: string;
  accountId: string;
  channel: Channel;
  destinationRef: string;
  templateVersion: string;
  renderedContent: string;
  contentSha256: string;
  createdAt: string;
  providerMessageId?: string;
  state: DeliveryState;
};

type Receipt = {
  receiptId: string;
  providerMessageId: string;
  observedAt: string;
  state: DeliveryState;
  source: "webhook" | "poll";
  raw: unknown;
};

interface EvidenceStore {
  insertAttempt(attempt: NoticeAttempt): Promise<void>;
  attachProviderId(attemptId: string, providerMessageId: string): Promise<void>;
  insertReceiptOnce(receipt: Receipt): Promise<boolean>;
  advanceState(providerMessageId: string, state: DeliveryState): Promise<void>;
}

interface DeliveryAdapter {
  send(input: {
    channel: Channel;
    destination: string;
    content: string;
  }): Promise<{ messageId: string }>;
  normalize(raw: unknown, source: Receipt["source"]): Receipt;
}

const digest = (content: string): string =>
  createHash("sha256").update(content, "utf8").digest("hex");

async function submitNotice(
  store: EvidenceStore,
  adapter: DeliveryAdapter,
  input: {
    attemptId: string;
    accountId: string;
    channel: Channel;
    destination: string;
    destinationRef: string;
    templateVersion: string;
    renderedContent: string;
    createdAt: string;
  },
): Promise<void> {
  await store.insertAttempt({
    attemptId: input.attemptId,
    accountId: input.accountId,
    channel: input.channel,
    destinationRef: input.destinationRef,
    templateVersion: input.templateVersion,
    renderedContent: input.renderedContent,
    contentSha256: digest(input.renderedContent),
    createdAt: input.createdAt,
    state: "queued",
  });

  const result = await adapter.send({
    channel: input.channel,
    destination: input.destination,
    content: input.renderedContent,
  });

  await store.attachProviderId(input.attemptId, result.messageId);
}

async function recordObservation(
  store: EvidenceStore,
  adapter: DeliveryAdapter,
  raw: unknown,
  source: Receipt["source"],
): Promise<void> {
  const receipt = adapter.normalize(raw, source);
  const inserted = await store.insertReceiptOnce(receipt);

  if (inserted) {
    await store.advanceState(receipt.providerMessageId, receipt.state);
  }
}
```

Enforce uniqueness for `attemptId` and `receiptId` in storage, not only in process memory. Implement `advanceState` with a transition table rather than enum ordering; external status vocabularies do not form one universal linear sequence. The authenticated webhook handler and scheduled poller should both call `recordObservation`, which keeps deduplication and projection logic in one place.

The raw receipt supports later review. Normalized fields keep queries stable. Meanwhile, the destination reference should follow the system's access-control and data-minimization policy; an audit requirement is not permission to scatter sensitive addresses through logs.

## Migration and rollout boundaries

Choose provider-owned templates when copy must change independently of weekly application releases and the provider's version, approval, and export behavior satisfies the written policy. It is not suitable when the application must reproduce a notice without access to provider configuration. In that case, stick with app-owned rendering.

Choose app-owned templates with polling alone when an inbound endpoint is prohibited. Accept the status lag, and first confirm that scheduled reads expose every observation the policy requires for long enough to collect it. Choose webhook-only for ordinary app alerts when fast status matters but reconciliation does not justify another worker.

For the default design, the operational budget is rendering tests, immutable storage, one callback adapter, and a bounded reconciler. That's more plumbing than a dashboard template and a send call. It is also a clean division of labor: the application owns the regulated words and their evidence; the transport owns delivery. Outsource the undifferentiated part.

## Sources

- https://resend.com/docs/introduction
- https://www.twilio.com/docs/glossary/what-sms-character-limit
