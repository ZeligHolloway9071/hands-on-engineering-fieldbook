# Realtime Notification API Alternatives — Node.js Chat Without Socket Servers

| Choice | Token boundary to verify | Reconnect proof to demand | Best fit |
| --- | --- | --- | --- |
| Managed channel API | Server-minted, room-scoped capability | Catch-up from a stored cursor | Chat transport without a socket fleet |
| Notification workflow API | Server-controlled recipient identity | Durable inbox state plus live delivery | User feeds and workflow fan-out |
| Self-hosted gateway | Application-defined authorization | Application-owned replay log | Protocol or deployment control |

**Short answer:** for property-management chat rooms, choose the least complex managed transport that passes two tests: the browser can receive a token for exactly one authorized room, and a reconnect can recover messages after a known cursor. Do not choose from a headline price or feature checklist. Put Pusher, Ably, and Knock through the same adversarial test harness, alongside a self-hosted gateway if operational control matters.

My recommendation is a server-authorized, room-scoped capability with a short lifetime, backed by an application-owned message log. The realtime API carries hints that new data exists; the log defines truth. This keeps a one-person SaaS out of the socket-server business without handing authorization to an untrusted client. It also protects the scarce resource that matters: hours available to ship the next weekly release.

## What should a realtime notification API alternative prove without a socket server?

Almost none. A browser may request access to `property:prop_42:room:maintenance`, but it must not decide that it belongs there. The application server derives the signed-in user from its own session, loads that user's membership, and mints a capability only after matching the requested property and room. A channel name is an address, not permission.

The happy path can hide this distinction. The tenant clicks a room, JavaScript subscribes, and messages appear. The failure shows up when someone edits a room ID in developer tools, reuses a token after leaving a property, or opens two tabs while a membership change is propagating. The useful question is: what independently verified statement allows this connection to subscribe?

Use the narrowest useful claim set. A chat token needs a subject, property, room, allowed operations, issued-at time, expiration time, and a unique identifier. Do not put every property room into one long-lived browser token. Revocation becomes coarse, leakage has a larger blast radius, and auditing cannot distinguish why a connection was allowed.

Reject four mismatches before contacting any realtime service:

1. The authenticated session subject differs from the requested subject.
2. The room does not belong to the requested property.
3. The subject has no current membership in that property and room.
4. The requested operation is broader than server policy permits.

A provider can sign or distribute the resulting capability. It cannot repair an authorization decision the application already got wrong. That is the first evaluation criterion for Pusher, Ably, Knock, or any alternative: can the integration preserve this narrow, server-owned decision without treating client input as authority?

Tokens expire.

## Reconnect is a data problem, not a socket feature

A reconnected socket proves only that a new connection exists. It does not prove that the client saw every message sent during the gap. Property chat makes that gap consequential: a tenant may close a laptop after reporting a leak, while a manager posts access instructions minutes later. A green connection badge is not delivery evidence.

Keep an append-only application record for each room. Every accepted message gets a stable message ID and ordering cursor before live fan-out. The client stores the last applied cursor. On startup and reconnect, it asks the application for records after that cursor, applies them idempotently, and then consumes live events. If a live event arrives before catch-up finishes, buffer it or merge it by cursor.

Design for duplicates. A message ID makes repeated delivery harmless, while a client-generated idempotency key prevents a retried send from creating two records. Ordering needs an explicit scope too: room-local order is useful; a global order across every managed property adds coordination without improving a resident's conversation.

Keep it boring.

History does not.

WebRTC does not remove these requirements. The W3C specification defines browser APIs for real-time peer connections and data channels, but peer connectivity is not an authorization policy or durable room history. It may suit a call or direct data exchange. It does not, by itself, explain how an offline manager recovers missed chat messages.

## A narrow Node.js boundary

The transport adapter should receive claims that have already been authorized. It should never receive a raw user ID from the request body and assume it is true.

```ts
type RoomGrant = {
  subject: string;
  propertyId: string;
  roomId: string;
  operations: readonly ["subscribe", "publish"];
  issuedAt: number;
  expiresAt: number;
  tokenId: string;
};

type ChatMessage = {
  id: string;
  roomId: string;
  cursor: number;
  authorId: string;
  body: string;
};

interface Memberships {
  canEnter(subject: string, propertyId: string, roomId: string): Promise<boolean>;
}

interface MessageLog {
  after(roomId: string, cursor: number, limit: number): Promise<ChatMessage[]>;
}

interface RealtimeTransport {
  mint(grant: RoomGrant): Promise<string>;
}

async function openRoom(
  sessionSubject: string,
  propertyId: string,
  roomId: string,
  afterCursor: number,
  memberships: Memberships,
  messages: MessageLog,
  transport: RealtimeTransport,
): Promise<{ token: string; missed: ChatMessage[] }> {
  const allowed = await memberships.canEnter(sessionSubject, propertyId, roomId);
  if (!allowed) throw new Error("room access denied");

  const now = Math.floor(Date.now() / 1000);
  const grant: RoomGrant = {
    subject: sessionSubject,
    propertyId,
    roomId,
    operations: ["subscribe", "publish"],
    issuedAt: now,
    expiresAt: now + 300,
    tokenId: crypto.randomUUID(),
  };

  const [token, missed] = await Promise.all([
    transport.mint(grant),
    messages.after(roomId, afterCursor, 100),
  ]);
  return { token, missed };
}
```

Five minutes is an example policy choice, not a universal optimum. Set the lifetime against membership-change urgency, refresh behavior, mobile sleep, and signer load. Likewise, the catch-up limit of 100 needs pagination or a bounded snapshot path when a room can accumulate a larger gap. Those numbers belong in configuration and tests.

The adapter is deliberately small. Replacing a transport should change `mint` and the live publish path, not membership policy or message history. I would outsource the undifferentiated connection work while keeping the product's security decisions in code I control.

## Test the boundary before comparing dashboards

A sales page cannot demonstrate isolation. A staging harness can. Create two properties, two rooms in each property, a manager with access to both, a tenant with access to one, and a former tenant whose membership has been removed. Run the same suite against every candidate.

Attempt a cross-property subscription by changing only the channel name. Replay an expired token, reuse a valid token from another account, request publish permission from a subscribe-only role, and reconnect after messages were written during a forced offline interval. Record authorization results, recovered message IDs, duplicate counts, cursor order, and time to revoke access. A pass means unauthorized events never arrive and the authorized client converges on stored room history.

Rotate signing keys with old and new clients connected. Restart the application during token refresh. Delay live events so catch-up and fan-out arrive out of order. Drop acknowledgments after persistence to force a send retry. These tests expose where a candidate's API ends and application responsibility begins.

Observability should use identifiers, not message bodies. Log the token ID, subject, property ID, room ID, decision, policy version, connection ID, and last acknowledged cursor. Never log the bearer token. Track denied cross-room attempts, refresh failures, and growing replay gaps. Measure reconnect convergence, but set targets from the product's user expectations rather than borrowing a generic threshold.

This is the revenue-per-hour trade. I would spend founder time on automated isolation and recovery tests because a failure can expose tenant data or lose a maintenance instruction. I would not spend the week rebuilding connection fan-out merely to own it. Ship weekly, and outsource the undifferentiated layer only after its boundary is measurable.

## When is the runner-up the better choice?

There is no universal runner-up. Choose the notification-workflow shape when the primary object is a durable per-user inbox with read state, preferences, templates, and delivery across several destinations. Chat can trigger those notifications, but a notification feed and an ordered room transcript are different records. Evaluate Pusher, Ably, and Knock against that distinction rather than assuming one integration should own both.

The recommended split has a real limitation: it requires operating a durable message store and a catch-up endpoint even though the socket layer is managed. It is not suitable when messages are intentionally ephemeral and losing an event during sleep has no product consequence; in that case, the replay machinery is needless work. It is also a poor fit when the team cannot keep authorization policy consistent between the token signer and the history reader. A workflow-oriented API is the better shape for inbox state and delivery preferences, while a self-hosted gateway is the better trade-off when control of protocol, network placement, or event retention outweighs the on-call cost.

Choose a self-hosted gateway when protocol control, deployment location, or integration with an existing event platform justifies on-call ownership. The cost is engineering attention: capacity planning, upgrades, connection draining, abuse controls, and incident response now compete with product work. For a solo SaaS, that trade must buy a capability customers can feel.

A managed channel API is the straightforward shape when room-scoped publish and subscribe are central, the application owns durable history, and the provider passes the hostile-client and reconnect suite. Compare contract fit, security controls, operational evidence, and exit effort. Price can break a tie after modeling actual connections and messages, but it should not decide the trust model.

**The browser requests access, the application grants it, and the message log settles what happened.** A socket is only the fast path. With that boundary intact, changing transport is contained work instead of a rewrite of property authorization or chat history.

## Further reading

- W3C, WebRTC 1.0: Real-Time Communication Between Browsers: https://www.w3.org/TR/webrtc/
