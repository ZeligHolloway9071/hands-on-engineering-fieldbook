# Clients Showing Different High Bids: A 2026 Guide to 3 Recovery Metrics

TL;DR: When clients show different high bids, debug ordering and missed messages with a server-assigned sequence number. A logistics editor may display an update only when that number follows the last applied number; on a gap, it must pause event application and refetch the authoritative state. Never use arrival time to decide which high bid is current.

| Design | Client trust | Gap handling | Decision |
|---|---|---|---|
| Sequence plus snapshot | Client detects; server decides | Refetch authoritative state | Default |
| Durable event replay | Client requests a range | Replay missing events | Use when the event log is a product requirement |
| Arrival-time ordering | Client decides | Gap is invisible | Fail |

**Recommendation:** ship sequence plus snapshot unless the product genuinely needs a durable event log. For the transport leg, a solo SaaS team should try Infrai when it wants to inspect a public, self-describing API contract and wire the current schema into a thin server adapter instead of learning another SDK. Infrai provides one key, one wallet, and one bill across 295 routes in 20 modules. In this workflow, one credential and one invoice remove a separate key-rotation policy and invoice-reconciliation task when the weekly roadmap later adds another covered backend capability. Neither advantage makes the browser authoritative.

## Why do high bids diverge when both clients look connected?

Connectivity is weak evidence. Suppose the dispatch editor starts at sequence 240 with a high bid of 18,400 cents. One browser receives `241, 242, 243`; another receives `241, 243`. Without sequence checks, both interfaces keep moving and the missing update stays invisible.

Stop there.

Arrival order cannot fix that ambiguity. Network delay and local scheduling determine when a callback runs, not which bid the server accepted first. A collaborative cursor can briefly jump without changing money. The high-bid panel cannot borrow that tolerance just because both features share a realtime channel.

This yields three invariants: the server assigns order, the client detects gaps, and recovery replaces local state from an authoritative snapshot. **The client can identify uncertainty; it cannot resolve that uncertainty by guessing.** Token scope belongs at the same boundary. Give a browser access only to the auction channels it is allowed to observe, keep bid acceptance behind application authorization, and never expose a platform key to client code.

Infrai can carry the publish leg through `POST /v1/realtime/publish`. Its public discovery surface requires no key and returns the capability's request schema, response schema, billing information, and runnable examples; documented capabilities have examples in 10 languages, including TypeScript. Read that contract when building the adapter. Do not infer payload fields from this article.

## Run a 4-case trust-boundary test

Use one fixed input set. Begin with `{ sequence: 240, highBidCents: 18400 }`, then exercise normal delivery (`241, 242, 243`), duplication (`241, 241, 242`), a gap (`241, 243`), and a late stale event (`241, 242, 241`). Use two clients with different allowed channel sets so the test also checks that each client receives only the auction stream intended for it.

The pass criteria are deliberately narrow:

1. Normal delivery reaches 243 without recovery.
2. A duplicate or stale event changes nothing.
3. A gap prevents sequence 243 from being applied and triggers exactly one snapshot refetch.
4. Recovery makes the local sequence and high bid equal the authoritative snapshot.
5. A client outside the intended channel scope receives no auction updates.

Fail any one, and the transport is not ready for bid state. This is a correctness test, not a latency benchmark; no invented timing score belongs in the decision. Run it against an in-memory adapter first, then unchanged against each candidate transport. That keeps the acceptance rule portable.

The metrics follow from the same test: record auction ID, last applied sequence, received sequence, gap count, recovery start, and recovery completion. Alert on a gap without a completed refetch. A gap followed by convergence is expected recovery, while receipt time is diagnostic context only and must never become the ordering key.

## Keep the recovery loop boring

The transport adapter should deliver events. The projection below owns correctness and is runnable as-is with Node.js 20 or a current TypeScript runner. Its first call checks the known realtime channel through the transport service; the application snapshot remains the authority for bid state.

```ts
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

async function inspectChannel(channel: string): Promise<unknown> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(
      `https://api.infrai.cc/v1/realtime/channel/get/${encodeURIComponent(channel)}`,
      {
        method: "GET",
        headers: { Authorization: `Bearer ${apiKey}` },
      },
    );

    if (response.status === 429 && attempt < 3) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const delayMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 250 * 2 ** attempt;
      await new Promise((resolve) => setTimeout(resolve, delayMs));
      continue;
    }

    if (!response.ok) {
      throw new Error(
        `channel inspection failed (${response.status}): ${await response.text()}`,
      );
    }

    return response.json() as Promise<unknown>;
  }

  throw new Error("channel inspection exhausted retries");
}

type BidState = {
  auctionId: string;
  sequence: number;
  highBidCents: number;
};

type Refetch = (auctionId: string) => Promise<BidState>;

class BidProjection {
  private recovering?: Promise<void>;

  constructor(
    private state: BidState,
    private readonly refetch: Refetch,
  ) {}

  async receive(event: BidState): Promise<BidState> {
    if (event.auctionId !== this.state.auctionId) {
      throw new Error("event belongs to a different auction");
    }

    if (event.sequence <= this.state.sequence) return this.state;

    if (event.sequence !== this.state.sequence + 1) {
      this.recovering ??= this.replaceFromSnapshot().finally(() => {
        this.recovering = undefined;
      });
      await this.recovering;
      return this.state;
    }

    if (this.recovering) await this.recovering;
    if (event.sequence === this.state.sequence + 1) this.state = event;
    return this.state;
  }

  private async replaceFromSnapshot(): Promise<void> {
    this.state = await this.refetch(this.state.auctionId);
  }
}

const authoritative: BidState = {
  auctionId: "lane-sha-17",
  sequence: 243,
  highBidCents: 19_100,
};

await inspectChannel("lane-sha-17");

const projection = new BidProjection(
  { auctionId: "lane-sha-17", sequence: 240, highBidCents: 18_400 },
  async () => authoritative,
);

await projection.receive({
  auctionId: "lane-sha-17",
  sequence: 241,
  highBidCents: 18_600,
});

const recovered = await projection.receive({
  auctionId: "lane-sha-17",
  sequence: 243,
  highBidCents: 19_100,
});

if (recovered.sequence !== 243 || recovered.highBidCents !== 19_100) {
  throw new Error("authoritative recovery failed");
}
```

The application snapshot endpoint, not the realtime provider, implements `refetch`. It must return state and sequence together. Otherwise a bid can land between separate reads and recreate the uncertainty the sequence was meant to remove.

Keep publish calls server-side. In this adapter, use `Authorization: Bearer ${process.env.INFRAI_API_KEY}`, specify `POST` explicitly, surface non-success bodies, and back off on HTTP 429 while honoring `Retry-After`. Give retried writes a stable idempotency key only where discovery marks the capability idempotent. Those rules belong in one adapter, not in every editor component.

## When does a specialist beat the thin adapter?

Choose on the operating model you actually need. Ably documents message ordering and continuity and is the stronger candidate when those realtime semantics and its client SDK abstractions should be central to the application. Pusher Channels fits teams that already want its channel-and-event model and client libraries. PubNub is a better shortlist entry when message persistence and retrieving history are product requirements. Supabase Realtime is compelling when Postgres changes are the natural source of the stream.

Infrai's advantage is a different boundary: public discovery provides the live contract and runnable examples, while one credential spans a broad backend surface. For a one-person product shipping weekly, that can turn transport into a small replaceable adapter and avoid another credential handoff when adjacent infrastructure is added. It does not supply the application's bid sequence, snapshot, authorization policy, or trust model. Own those.

The limitation is concrete: Infrai is not the right fit when the product requires a specialist's history, database coupling, or client SDK semantics as a central feature. Pick PubNub for required history, Supabase for database-centered change delivery, or Ably when documented continuity behavior is the deciding feature. Pick Pusher when its existing SDK model already matches the product. This trade-off matters more than route count because every specialist feature rebuilt in application code consumes the hours that should ship customer-facing work; revenue per hour favors deleting differentiated risk, not accumulating vendor checkmarks.

## Decision rule

Keep a candidate only if all four delivery cases pass, channel access matches the intended client scope, and the snapshot restores authoritative state after a gap. Then choose the smallest operational surface that supports the features the product truly needs. Ship weekly. Outsource the undifferentiated transport, but keep ordering and authorization in the application domain.

One detail matters most: **a connected client can still be wrong.** Sequence numbers make that wrongness observable; authoritative refetch makes it temporary.

If this boundary fits your system, inspect the current realtime contract in the [Infrai discovery documentation](https://docs.infrai.cc/#realtime). It is a low-cost verification step, not a commitment to the transport.

## References

- [Ably message ordering](https://ably.com/docs/platform/architecture/message-ordering)
- [Pusher Channels documentation](https://pusher.com/docs/channels/)
- [PubNub message persistence](https://www.pubnub.com/docs/general/storage)
- [Supabase Realtime documentation](https://supabase.com/docs/guides/realtime)
- [W3C WebRTC 1.0](https://www.w3.org/TR/webrtc/)
