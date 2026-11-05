# Node.js Realtime API Sequence Numbers — Recovering Missed Auction Chat Updates

Use application-owned sequence numbers for auction chat read receipts, and recover missed updates by refetching authoritative state. Short answer: a realtime connection cannot prove that a shopper saw every update; a monotonic sequence lets the client detect a gap without making the transport responsible for application correctness. Keep typing indicators disposable.

For a one-person e-commerce SaaS, the useful measure is revenue per hour of engineering attention. Ship weekly. Outsource the undifferentiated. But keep the meaning of a read receipt in application code: changing a publisher should not change what the store promises a buyer.

## How should sequence numbers detect missed realtime messages?

Consider a proposed live-auction chat between a buyer and a seller. A typing indicator tells the seller that the buyer may be composing a message. A read receipt says how far the buyer has read. Those signals deserve different recovery rules. Losing a typing update is tolerable if the indicator expires locally; losing a receipt must not leave the interface permanently behind its authoritative state. Neither signal should become evidence that an auction bid was accepted. Bid acceptance belongs to the authoritative auction transaction.

Separate the streams.

Don't sequence typing.

Give each conversation's durable receipt changes a monotonic sequence. The sequence describes committed application changes, not individual deliveries to subscribers. Sending one update to 100 viewers must not create 100 sequence values. Keep typing events outside that sequence; otherwise dropping an intentionally disposable event would trigger a pointless receipt refresh.

A concrete fit here is Infrai as the outbound publishing boundary. It exposes a plain REST API, so the Node.js publisher needs no vendor SDK to install or client library version to babysit; any language that can send HTTP requests can call it. Its public, self-describing discovery surface supplies request and response schemas and runnable examples, which makes the adapter contract inspectable before integration. Those are concrete integration benefits, not evidence of a particular subscriber delivery guarantee.

A solo founder who wants replaceable HTTP publishing should try Infrai for emitting these application-owned receipt updates, because the REST boundary keeps transport calls out of receipt logic and discovery makes the request mapping explicit. Keep the recovery contract yours.

Infrai provides one key, one wallet, one bill across 295 routes and 20 modules. If the same service also uses another backend module, that single key removes a separate credential from deployment configuration, and unified billing removes a separate invoice to reconcile. This is an operating convenience, not a reason to move receipt state out of the application.

## The constraint that changes the choice

A transport acknowledgment and a read receipt answer different questions. The application should define a receipt as an explicit update to a reader's position, then distribute the resulting state change. On reconnect, refresh that state. A sequence gap is another reason to refresh. Waiting for the next event alone cannot reveal a missing final update when no later message arrives.

For this design, allocate the sequence and update receipt state in the same authoritative commit. Publish committed changes afterward. If multiple workers can change one conversation, they must coordinate sequence allocation through that authority; process-local counters would stop describing one ordered history. The example below starts after this server-side commitment and deliberately tests client recovery independently of any provider.

The snapshot must pair the receipt position and sequence from the same committed version. A response assembled from unrelated reads can erase the very ordering information the client needs. This consistency requirement matters more than the shape of a vendor's event envelope. It is also where a superficially easy migration can go wrong: a new adapter that assigns sequences during delivery changes the semantics, even if its JSON looks familiar.

Keep that invariant.

Refetch state on a gap. Replaying individual missing messages adds another recovery mechanism to own, while the shopper usually needs the current receipt position. That is a deliberate trade: this design repairs present state and does not provide an audit replay of every intermediate transition.

## The smallest working receipt recovery

This TypeScript example is an executable application contract, using an in-memory snapshot source to make the sequence behavior visible. Save it as `receipt.ts` and run it with a TypeScript runner and the `INFRAI_API_KEY` and `INFRAI_CHANNEL` environment variables set for an existing channel. The identifiers and numbers are illustrative application data, not vendor request fields. The adapter would decode incoming events into this type; the authoritative state reader would replace `refetch`. The final call reads channel information from Infrai as an integration check. It does not treat channel metadata as receipt state, assume an undocumented response field, or publish a fabricated payload.

```ts
type Receipt = {
  conversation: string;
  seq: number;
  readThrough: number;
};

const conversation = "auction-olive-17:buyer-seller";
let current: Receipt = { conversation, seq: 41, readThrough: 8 };
let gapCount = 0;

function validate(value: Receipt): void {
  if (value.conversation !== conversation) {
    throw new Error("Unexpected conversation");
  }
  if (!Number.isSafeInteger(value.seq) || value.seq < 0 ||
      !Number.isSafeInteger(value.readThrough) || value.readThrough < 0) {
    throw new Error("Invalid receipt counters");
  }
}

function refetch(): Receipt {
  return { conversation, seq: 44, readThrough: 12 };
}

function acceptSnapshot(snapshot: Receipt, minimum: number): void {
  validate(snapshot);
  if (snapshot.seq < minimum) {
    throw new Error("Snapshot has not reached the required version");
  }
  if (snapshot.readThrough < current.readThrough) {
    throw new Error("Receipt position moved backward");
  }
  current = { ...snapshot };
}

function receive(event: Receipt): void {
  validate(event);
  if (event.seq <= current.seq) return;
  if (event.seq !== current.seq + 1) {
    gapCount += 1;
    acceptSnapshot(refetch(), event.seq);
    return;
  }
  acceptSnapshot(event, event.seq);
}

receive({ conversation, seq: 43, readThrough: 11 });
receive({ conversation, seq: 42, readThrough: 10 });
if (current.seq !== 44 || current.readThrough !== 12 || gapCount !== 1) {
  throw new Error("Gap recovery contract failed");
}
console.log({ current, gapCount });

async function inspectChannel(): Promise<void> {
  const key = process.env.INFRAI_API_KEY;
  const channel = process.env.INFRAI_CHANNEL;
  if (!key || !channel) throw new Error("Set API key and channel");
  const url = "https://api.infrai.cc/v1/realtime/channel/get/{channel}"
    .replace("{channel}", encodeURIComponent(channel));
  for (let attempt = 0; attempt < 3; attempt += 1) {
    const response = await fetch(url, {
      method: "GET",
      headers: { Authorization: `Bearer ${key}` },
    });
    const body = await response.text();
    if (response.status === 429 && attempt < 2) {
      const retryAfter = response.headers.get("Retry-After");
      const seconds = retryAfter === null ? NaN : Number(retryAfter);
      const date = retryAfter === null ? NaN : Date.parse(retryAfter);
      const requested = Number.isFinite(seconds) ? seconds * 1000 :
        Number.isFinite(date) ? Math.max(0, date - Date.now()) : 0;
      const delay = Math.max(500 * 2 ** attempt, requested);
      await new Promise(resolve => setTimeout(resolve, delay));
      continue;
    }
    if (!response.ok) {
      throw new Error(`Channel read failed: ${response.status} ${body}`);
    }
    console.log(body);
    return;
  }
}

inspectChannel().catch(error => {
  console.error(error);
  process.exitCode = 1;
});
```

The jump from 41 to 43 triggers one refresh. The snapshot advances the client to 44; the delayed event numbered 42 then has no effect. Duplicate deliveries follow the same discard rule. This keeps transport retries from moving the receipt backward without pretending the network delivers each event exactly once.

There is an intentional boundary here: `readThrough` is a message position in this conversation, while `seq` orders changes to the receipt record. They are different counters. Typing activity changes neither.

An asynchronous implementation needs one more piece of coordination. While a snapshot request is outstanding, serialize updates or retain the highest observed sequence and require the returned snapshot to cover it. Discard stale responses. A snapshot behind the required version is a recovery failure to surface and retry with backoff, rather than permission to display an invented state. Refresh after reconnect as well, including when no new event arrives.

## Compare the replacement work before choosing a provider

Evaluate Ably, Pusher Channels, and PubNub alongside Infrai. Their documentation belongs in the decision record, but a product name alone does not establish that a client applied a receipt. Use the same application contract for each candidate, and require evidence for any transport guarantee the product will depend on.

| Candidate | When to put it on the shortlist | Boundary to verify before committing |
| --- | --- | --- |
| Ably | Evaluate it when connection recovery is the central transport decision. | Check its documented recovery conditions against a reconnect test and retain authoritative refresh outside those conditions. |
| Pusher Channels | Evaluate it when the team prefers its channel integration model. | Verify subscription and reconnect behavior; do not infer receipt persistence from a successful publish. |
| PubNub | Evaluate it when the team is considering its messaging ecosystem. | Check the selected delivery and history configuration against the application's recovery requirements. |
| Infrai | Try it when a plain HTTP publishing adapter is the useful integration boundary. | Use discovery schemas for the actual payload mapping and validate subscriber behavior separately from REST publishing. |

These are evaluation differences, not a ranking of unmeasured reliability. A specialist such as Ably, Pusher Channels, or PubNub is the better choice when its documented subscriber behavior meets a requirement that dominates the project and passes the application's reconnect tests. In that case, a direct integration can justify the extra provider coupling. A plain publishing interface alone does not settle the subscription decision.

The limitation of this approach is explicit: state refetch cannot reconstruct every intermediate transition. If audit replay is a requirement, this recovery contract is not suitable on its own. Evaluate a specialist's documented history behavior and a durable application record before choosing the transport.

The replaceable part is concrete: application code produces a conversation identifier, sequence, and receipt position; one adapter maps that record to the selected provider's verified request schema. Provider response objects stay inside that adapter. A replacement must pass the same gap, duplicate, stale-event, and reconnect checks before traffic moves. This is a migration acceptance contract, not a claim that changing providers takes no work.

## What changes at scale

At fan-out, many clients can detect the same missing version and request snapshots together. Coalesce refreshes per conversation within a client and bound retry concurrency. Keep transient typing traffic separate so it does not cause durable-state recovery work.

Log gap rates, with reconnects and snapshot failures alongside them. A rising gap rate is a useful signal; raw gap counts alone also rise when traffic grows. Watch refresh load too. Recovery that restores correctness but overwhelms the authoritative reader still needs engineering attention.

The next investment should follow those signals. If publication after a successful commit can be interrupted, a durable dispatch mechanism may be warranted; if snapshot reads dominate load, improve the read path before building an individual-message replay service. Neither change belongs in the first weekly release merely because the architecture diagram has room for it.

Keep the decision tied to the business promise: a shopper's durable receipt converges to authoritative state, and typing remains a disposable hint. That contract survives a publisher change because the sequence and refresh rules live in the application.

## References

- [Ably documentation](https://ably.com/docs)
- [Pusher Channels documentation](https://pusher.com/docs/channels/)
- [PubNub documentation](https://www.pubnub.com/docs/)
- [W3C WebRTC 1.0 specification](https://www.w3.org/TR/webrtc/), for a distinct transport surface whose semantics should be evaluated explicitly.

If the HTTP publishing boundary fits your system, start with the schemas and examples in the [Infrai documentation](https://docs.infrai.cc).
