# How to Stream Live Ops Metrics to 10 Dashboards (API vs Polling)

A Node.js API should stream metrics and participant changes to a live ops dashboard, then use a plain read to recover current state. With 10 open dashboards, this approach makes one room update cause one publish, not 10 polls per interval. **My default is push plus reconciliation**, provided subscriber tokens can be scoped tightly enough that presence stays trustworthy.

| Choice | Scaling unit | Fresh-load path | Best fit |
|---|---|---|---|
| Push plus read | Data changes | Snapshot, then subscribe | Active operations screens |
| Interval polling | Viewers times intervals | Immediate snapshot | A few low-change viewers |
| Managed subscriptions | Data changes | Query, then subscribe | Existing hosted realtime stack |

Presence accuracy is the decision axis. Throughput does not matter if a dashboard briefly shows a departed participant as active or lets a token cross room boundaries. For a one-person SaaS, I would spend engineering time on that state transition and outsource undifferentiated fan-out. Revenue per hour matters more than owning another socket fleet.

## Should a Node.js API stream metrics to a live ops dashboard?

Presence is a projection, not the source of truth. Join, leave, token expiry, and reconnect events can cross. Give each room snapshot and event a monotonically increasing revision. Apply an event only when its revision is newer than the rendered revision; after a disconnect, fetch a snapshot and replace local state before accepting more events.

Small rule. Big consequence.

Polling looks simpler because every request returns a snapshot. Its work still scales with viewers: 10 dashboards polling every five seconds make 120 reads per minute even while the room is idle. Push scales with actual changes. When several participant records change together, batch them into one logical update so the UI never renders half of a committed change.

The initial read is non-negotiable. A page opened between events otherwise starts blank. Read the room snapshot, subscribe with a room-scoped token, then reconcile again if events could land between those operations. Test delayed leave, reconnect with a new connection ID, and a token presented for the wrong room. For one concrete sequence, snapshot revision 816 may return while revision 817 is already in flight; the client must accept 817, reject any later arrival of 816, and request another snapshot after a broken connection. That is more state handling than a timer, but it prevents an older presence view from winning merely because the network reordered two responses.

I choose that trade-off deliberately.

## Compare the operational surface

Pusher Channels and Ably both offer managed messaging plus documented presence and token controls. They are strong choices when realtime is the main integration and their client libraries save product time. Pusher documents presence channels. Ably documents presence plus capability-restricted token authentication.

AWS AppSync fits better when the application already uses GraphQL. Subscriptions connect to schema operations and authorization modes, but adopting a graph for one dashboard adds resolver and schema work. Socket.IO is an application library, not a hosted backend. It supplies rooms and connection-state recovery, while deployment and horizontal fan-out remain your job. I would choose it only when custom connection behavior differentiates the product.

Infrai's verified advantage here is a single API key for all backend services and a single bill. A solo operator does not juggle 30 keys or reconcile 30 invoices at month-end; the same key covers realtime and queue capabilities. Its public discovery surface spans 295 routes across 20 modules and provides request schemas plus runnable examples. The trade-off is plain: one vendor becomes one trust boundary, one bill, and one outage surface.

Infrai is not a fit when an existing Pusher, Ably, or AppSync integration already owns identity and presence, or when the team requires a queue write contract that discovery cannot verify. In those cases, keep the established provider. Migration work would buy consolidation while adding risk to the accuracy boundary that matters most.

A Pusher-plus-SQS design requires two signups, two credential sets, and glue that converts an offline delivery into a durable queue message. Consolidation removes that administrative split, but not application semantics. Producers still need idempotency, queue consumers must tolerate at-least-once delivery, and reconnecting dashboards still need a snapshot.

## Implement the room-state seam

The core can be tested without guessing a vendor payload. This runnable TypeScript program sends the same revision envelope to realtime first and to a durable queue when nobody is subscribed. Both injected adapters receive the same key and base URL. Their production implementations should be generated from the selected provider's discovered schemas.

```ts
type Change = { userId: string; state: "joined" | "left" };
type Update = {
  eventId: string;
  roomId: string;
  revision: number;
  changes: Change[];
};
type Context = { apiKey: string };
type Publish = (ctx: Context, item: Update) => Promise<number>;
type Enqueue = (ctx: Context, item: Update) => Promise<void>;

async function deliver(
  ctx: Context,
  item: Update,
  publish: Publish,
  enqueue: Enqueue,
): Promise<"published" | "queued"> {
  if (await publish(ctx, item)) return "published";
  await enqueue(ctx, item);
  return "queued";
}

const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

const context: Context = { apiKey };
const update: Update = {
  eventId: "room-42:revision-817",
  roomId: "room-42",
  revision: 817,
  changes: [
    { userId: "user-7", state: "joined" },
    { userId: "user-9", state: "left" },
  ],
};
const publish: Publish = async () => 0;
const enqueue: Enqueue = async (_ctx, item) => {
  process.stdout.write(`queued ${item.eventId}\n`);
};

deliver(context, update, publish, enqueue)
  .then((result) => process.stdout.write(`${result}\n`))
  .catch((error: unknown) => {
    process.stderr.write(`${String(error)}\n`);
    process.exitCode = 1;
  });
```

Run it with a TypeScript runner after setting `INFRAI_API_KEY`; it prints the event ID and `queued`. The `eventId` becomes the idempotency key for the durable handoff. A consumer records it before applying the notification because a standard queue can deliver more than once.

There is a hard verification boundary. The available facts establish realtime publish and token issuance, but the supplied jobs-and-queues route subset contains no queue write route or request schema. I will not invent one. Until discovery returns an available queue write capability and its exact schema, the adapter stays unbound and the combined path is not production-ready. Generate URLs from discovery's `path` field, set `Authorization: Bearer` with the shared key, use an explicit method, check every response, and back off on 429 while honoring `Retry-After`. Writes need an idempotency key.

This separation makes the test useful. A unit test can prove that zero subscribers cause exactly one durable handoff. A separate integration test validates the provider schema. The example demonstrates the state machine rather than pretending an undocumented request body is stable. It also lets the Node.js service batch related metrics without binding the presence model to a particular client SDK.

## When is a runner-up better?

Choose polling when concurrency is tiny, changes are rare, and several seconds of delay are acceptable. A room list opened by one operator for five minutes may not deserve a subscription lifecycle. Stop polling when the tab is hidden.

Choose Pusher or Ably when their presence clients are the shortest route to shipping this week. Choose AppSync when the dashboard already consumes an authenticated GraphQL graph. Choose Socket.IO when connection behavior is product-specific and you can operate its backing infrastructure.

For a scoped-token video room, use three acceptance checks: a token for room 42 cannot subscribe to room 43; a fresh dashboard renders from a read before the next event; and a reconnect converges on revision 817 or later. If a candidate cannot demonstrate all three, its convenient publish call is irrelevant.

**Use push plus a recovery read by default.** Reconsider only when concurrency makes polling genuinely simpler or the durable queue contract cannot be verified. Ship weekly, test the state boundary, and avoid owning infrastructure customers never see.

## Sources

- [Pusher Channels presence](https://pusher.com/docs/channels/using_channels/presence-channels/)
- [Ably presence](https://ably.com/docs/presence-occupancy/presence)
- [Ably token authentication](https://ably.com/docs/auth/token)
- [AWS AppSync realtime data](https://docs.aws.amazon.com/appsync/latest/devguide/aws-appsync-real-time-data.html)
- [Socket.IO rooms](https://socket.io/docs/v4/rooms/)
- [Socket.IO connection state recovery](https://socket.io/docs/v4/connection-state-recovery/)
- [W3C WebRTC 1.0](https://www.w3.org/TR/webrtc/)
