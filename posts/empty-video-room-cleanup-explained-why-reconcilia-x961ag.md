# Empty Video Room Cleanup Explained (Why Reconciliation Beats Leave Hooks)

| Choice | Presence accuracy | Failure behavior | Operational cost |
|---|---|---|---|
| Delete on the last `leave` callback | Weak when callbacks are lost, duplicated, or reordered | A missed callback retains an empty room; a late callback can race a new join | Low until cleanup becomes manual |
| Mark activity, then reconcile expired rooms | Strong when the store supports conditional updates | A failed pass can be retried without treating one event as truth | One small scheduled worker plus metrics |

**TL;DR:** For a B2B SaaS that creates video rooms and scoped tokens, I would treat leave callbacks as hints and let a periodic reconciler make the deletion decision. The useful invariant is not “I saw the final leave.” It is “the room has no active leases, its inactivity deadline has passed, and its version has not changed since I checked.” This costs more code than a callback, but it buys the presence accuracy that matters when empty rooms otherwise turn into recurring debug work.

Ship the boring control loop. Spend product time elsewhere.

## Why does the last participant leave fail as a cleanup trigger?

A browser closing a peer connection and an application declaring a room empty are different events. WebRTC defines `RTCPeerConnection` state, including aggregate connection state, but signaling is outside the specification. The server still has to decide how client state, signaling sessions, reconnects, and room membership become one durable presence model. A client-side `connectionState` change cannot make that decision by itself.

The tempting implementation keeps an in-memory member count and deletes the room when it reaches zero. It works in a tidy test: two people join, both leave, the callback fires twice, and the count lands on zero. Real execution adds the cases that matter. A process can stop after recording the leave but before deleting. A callback can arrive twice. Two callbacks can read the same count. A disconnected browser can reconnect while an older cleanup task is still queued. A deployment can erase process memory while the durable room record remains.

Those cases follow from putting durable state behind an ephemeral event handler. The event says that one observer saw something at one time. It does not prove that nobody is entitled to rejoin now.

Events aren't truth.

For scoped room tokens, the boundary gets sharper. Issuing a token establishes an authorization decision for a room and participant scope; it does not establish live presence. Likewise, a token that remains cryptographically valid is not evidence that its holder is connected. Mixing authorization, connection state, and cleanup into one counter creates a model that is easy to code and hard to reason about.

I use three separate concepts in the design: token scope controls what a participant may do, a renewable presence lease records recent activity, and the room lifecycle records whether new activity may be accepted. That separation is the main choice.

## Two criteria decide the design

The first criterion is **whether absence can be proven from durable, current data**. Each participant session gets a lease with an expiry. Heartbeats extend that lease using the server's clock. A graceful leave may expire it early, which improves cleanup latency, but correctness does not depend on receiving that leave. The room becomes a cleanup candidate only after every lease is expired and an inactivity deadline has elapsed.

Why use the server's clock? A browser clock can be wrong, and accepting its timestamps lets one client stretch or shorten the cleanup window. The server can record receipt time consistently for all sessions. The exact lease duration is an operating choice, not a universal constant: it must exceed normal heartbeat jitter and remain short enough to meet the product's empty-room retention objective. Measure both before choosing it.

The second criterion is **whether a stale decision can delete a newly active room**. Reading “zero active leases” and deleting in two unrelated operations leaves a race. A participant may join between them. The reconciler therefore needs a compare-and-delete condition: delete only if the room version and inactivity deadline still match the snapshot it inspected. A join increments the version and changes the deadline, so an old cleanup attempt loses.

This is the part I would not trade away for a shorter implementation. Presence accuracy is the primary decision axis, and a conditional write turns a timing assumption into an enforceable rule.

Consider one concrete interleave. At 10:00:00, the worker reads room `sales-demo-42` with zero active leases and version 18. At 10:00:01, a participant reconnects; the join transaction creates a fresh lease and advances the room to version 19. A second later, the worker tries to claim the version-18 snapshot. The conditional update affects no record, so resource release never starts. Without that version check, both the reconnect and deletion can succeed from their own handler's point of view, leaving the user with a valid scoped token for a room being dismantled. The sequence is small enough for a deterministic test and important enough to keep out of an optimistic read-then-delete path.

Keep the lifecycle small: `open`, `closing`, and `deleted` are enough for many systems. A join may update an `open` room. The reconciler conditionally moves an eligible room to `closing`, releases external resources, then marks it `deleted`. If resource release must be retried, persist an idempotency key derived from the room ID and lifecycle version. Do not make a request handler wait for every downstream cleanup operation.

## A minimal TypeScript control loop

The interfaces below leave storage and token signing outside the room lifecycle. That is deliberate. The store must implement the conditional transitions atomically; the worker merely expresses the policy.

```ts
type RoomSnapshot = {
  id: string;
  version: number;
  state: "open" | "closing" | "deleted";
  inactiveAfterMs: number;
  activeLeaseCount: number;
};

type ScopedTokenRequest = {
  roomId: string;
  participantId: string;
  permissions: readonly ("publish" | "subscribe")[];
};

interface RoomStore {
  recordJoin(input: {
    roomId: string;
    sessionId: string;
    leaseExpiresAtMs: number;
  }): Promise<void>;
  findCleanupCandidates(nowMs: number, limit: number): Promise<RoomSnapshot[]>;
  beginCleanupIfUnchanged(input: {
    roomId: string;
    expectedVersion: number;
    inactiveAfterMs: number;
    nowMs: number;
  }): Promise<boolean>;
  finishCleanup(roomId: string): Promise<void>;
}

interface TokenIssuer {
  issue(input: ScopedTokenRequest): Promise<string>;
}

interface RoomResources {
  release(roomId: string, idempotencyKey: string): Promise<void>;
}

const leaseMs = 45_000;

async function joinRoom(
  store: RoomStore,
  tokens: TokenIssuer,
  input: ScopedTokenRequest & { sessionId: string },
): Promise<{ token: string; leaseExpiresAtMs: number }> {
  const leaseExpiresAtMs = Date.now() + leaseMs;
  await store.recordJoin({
    roomId: input.roomId,
    sessionId: input.sessionId,
    leaseExpiresAtMs,
  });
  const token = await tokens.issue(input);
  return { token, leaseExpiresAtMs };
}

async function reconcileEmptyRooms(
  store: RoomStore,
  resources: RoomResources,
  nowMs = Date.now(),
): Promise<void> {
  const candidates = await store.findCleanupCandidates(nowMs, 100);

  for (const room of candidates) {
    if (room.state !== "open" || room.activeLeaseCount !== 0) continue;

    const claimed = await store.beginCleanupIfUnchanged({
      roomId: room.id,
      expectedVersion: room.version,
      inactiveAfterMs: room.inactiveAfterMs,
      nowMs,
    });
    if (!claimed) continue;

    await resources.release(room.id, `${room.id}:${room.version}`);
    await store.finishCleanup(room.id);
  }
}
```

The illustrative `45_000` milliseconds is a configuration example, not a recommended universal timeout. The `100`-room batch is also a capacity knob. Set both from observed heartbeat delay, reconnect behavior, worker duration, and the maximum cleanup lag the product can tolerate.

Numbers need owners.

One subtle failure remains: token issuance can fail after `recordJoin` succeeds. The lease then expires naturally, so the room is not retained forever. Reversing those two operations is worse because a usable token might escape before the server records the session. If the token issuer supports a stable request identifier, retries can avoid creating logically distinct grants; the concrete mechanism belongs to that issuer's contract.

## Test the races, then watch the invariant

A unit test for “count reaches zero” is insufficient. The valuable tests control time and interleave operations. Start cleanup, pause after the candidate read, join a participant, then resume cleanup; the conditional transition must fail. Deliver the same graceful leave twice; the lease state must remain valid. Stop the worker after it claims a room, run it again, and verify that resource release is safe to repeat. Advance the clock past lease expiry without sending a leave, and verify eventual deletion.

Also test the inverse. A heartbeat received before expiry must prevent the room from becoming a candidate. A heartbeat received after the room has entered `closing` must be rejected or routed through an explicit room-recreation path. Silently reopening a half-cleaned room makes the lifecycle ambiguous.

I would expose four operational signals: open rooms with zero active leases, age of the oldest eligible room, conditional cleanup conflicts, and cleanup retries. The first two show backlog. Conflicts show the protection doing real work; a sudden increase can indicate reconnect churn or an overly aggressive inactivity window. Retries separate lifecycle accuracy from trouble releasing downstream resources.

Alert on elapsed time against the product's retention objective, not on a raw room count. Ten empty rooms for three seconds may be healthy. One empty room stuck for a day is not. This is where a small SaaS keeps the revenue-per-hour lens honest: automate the invariant, keep the dashboard narrow, and ship weekly instead of repeatedly inspecting abandoned room records.

## What are the limitations, and when is the runner-up better?

The reconciliation design has real limitations. It adds a scheduled worker, durable lease writes, clock-based policy, conditional storage operations, and at least one more path to observe. Cleanup is deliberately delayed until lease expiry, so it cannot provide immediate reclamation after every normal departure. A last-leave callback is a reasonable runner-up when rooms are process-local, have no durable or billable resources, disappear automatically with the process, and a false empty decision cannot affect reconnecting users. It is also useful as a fast-path signal alongside reconciliation. In that role, it shortens ordinary cleanup without owning correctness.

The extra worker is harder to justify for a disposable prototype with no persistent room record. Once a room owns scoped authorization, retained metadata, recording state, or external resources, the trade changes. Outsource undifferentiated media transport if that helps delivery, but keep the room-state invariant explicit in your own system boundary.

I chose reconciliation for this case because presence accuracy outranks immediate cleanup, but that trade-off isn't free. Use callbacks for speed and a durable reconciler for truth. That gives reconnects a fair chance, makes cleanup retryable, and keeps empty rooms from becoming a recurring manual queue.

## References

- W3C, WebRTC 1.0: Real-Time Communication Between Browsers: https://www.w3.org/TR/webrtc/
