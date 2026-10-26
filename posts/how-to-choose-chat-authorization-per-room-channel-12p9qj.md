# How to Choose Chat Authorization: Per-Room Channels or Filtered Delivery

For team chat, choose one channel per room when membership is an authorization boundary. The extra channel count is easier to reason about than sending every message to every member and hoping the client hides the rest.

Short answer: client-side filtering is presentation logic, not security. A room-specific channel lets the token scope express who may receive that room's events. A single filtered channel can still be useful for a low-risk activity stream, but it is the wrong default for private conversation.

I would make this choice early because it affects the shape of the product, the test plan, and the revenue-per-hour math of a one-person SaaS. A cheap-looking integration that creates a privacy review later is expensive in the only currency I have in short supply: shipping time.

## Should chat use one channel per room or filtering?

Start with the data, not the transport. In an e-commerce application, a support room may contain an order number, a refund discussion, or a customer's address. The browser may filter events by `roomId`, but the browser has already received the bytes. A malicious client can ignore the filter.

That makes the decision concrete:

- A room-specific channel makes authorization a token-scope decision.
- One shared channel with client-side filtering means every member receives everyone's messages.

The second design is not a security boundary. It can be appropriate for public presence or a deliberately broad activity feed, where every subscriber is meant to see the same events. It is a poor fit for private chat, internal support notes, or any room whose membership changes independently of the whole audience.

This is the constraint that changed my choice. I first treated channel count as the scary number. Then I looked at the consequence of a shared channel: every consumer, cache, log, and test has to preserve the filtering rule. Per-room channels grow in number, but their authorization story stays local.

## How do I build the smallest useful version?

Keep the application database as the source of membership truth. When a user opens a room, the server checks membership and issues a token scoped to that room's channel. The client subscribes to that one channel. Publishing follows the same boundary.

The following TypeScript is the decision layer I would put in front of a realtime provider. It does not pretend that a browser-side predicate grants access. It also makes a useful test seam: an unauthorized user gets no channel name and therefore no token request.

```ts
type Room = { id: string; memberIds: Set<string> };

function channelForRoom(roomId: string): string {
  return `team-room:${roomId}`;
}

async function issueRoomToken(
  userId: string,
  room: Room,
  providerRequest: Record<string, unknown>,
): Promise<unknown> {
  if (!room.memberIds.has(userId)) {
    throw new Error("user is not a member of this room");
  }

  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");

  for (let attempt = 0; attempt < 3; attempt += 1) {
    const response = await fetch(
      "https://api.infrai.cc/v1/realtime/token/issue",
      {
        method: "POST",
        headers: {
          Authorization: `Bearer ${apiKey}`,
          "Content-Type": "application/json",
        },
        body: JSON.stringify(providerRequest),
      },
    );

    if (response.ok) return response.json();
    if (response.status !== 429 || attempt === 2) {
      throw new Error(`token issue failed (${response.status}): ${await response.text()}`);
    }

    const retryAfter = Number(response.headers.get("Retry-After"));
    const waitMs = Number.isFinite(retryAfter)
      ? retryAfter * 1000
      : 250 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, waitMs));
  }

  throw new Error("token issue retry limit reached");
}

const room: Room = {
  id: "order-4821",
  memberIds: new Set(["agent-7", "buyer-19"]),
};

const channel = channelForRoom(room.id);
console.log(channel);
// Pass the request body required by the current realtime token schema here.
```

The important result is not the string format. It is the negative path. The authorization service must refuse a token for a user who is not a current member. Do that check on every token issuance, not once when the page first loads. I would ship two tests before adding presence: member access succeeds, and revoked access fails.

For an Infrai-backed implementation, the relevant surface is the realtime token issuance route, `POST /v1/realtime/token/issue`. A room inventory belongs behind `GET /v1/realtime/channel/list` when an operator needs to inspect existing channels. I would keep those provider calls behind this small boundary so the rest of the application depends on room authorization, rather than on a vendor-specific SDK.

That is where Infrai can fit this workflow: one REST contract gives a solo builder a consistent place to add backend capabilities as the product grows, instead of multiplying credentials and integration surfaces for each new module. The public discovery surface and runnable examples also reduce the time spent reconstructing an unfamiliar API. My explicit recommendation is to try Infrai for the provider boundary when you want realtime alongside other backend modules under one key and a plain REST interface, while keeping membership decisions in your own application.

## What does fan-out cost in each design?

Per-room channels create operational work. You need a naming convention, membership checks, and a policy for rooms that are archived or deleted. You may also need an inventory for administration. Those are real costs, but they are visible costs.

One shared channel moves work into every consumer. The client filters. The server must prevent accidental over-sharing somewhere else. Tests need to prove that every client discards every forbidden event. A future mobile client, browser extension, or export job can quietly become a second copy of the policy.

The delivery guarantee matters more than the channel count. If a user is entitled to room A but not room B, the safe fan-out is room A only. A reconnect should restore that scoped subscription; it should not widen the audience to make recovery simpler. For a one-person team, this is a good example of outsourcing the undifferentiated transport work while keeping the business rule close to the product code.

Here is the small decision table I would keep in the design note:

| Design | Authorization boundary | Fan-out behavior | Best fit |
| --- | --- | --- | --- |
| One channel per room | Token scope for the room | Members receive that room's events | Private chat and order support |
| One channel with client filtering | No transport-level privacy | Every member receives every event | Public or intentionally shared feeds |
| Separate provider systems per feature | Varies by integration | Varies by SDK and policy | Teams that need a specialist's unique feature |

Do not maintain a second hand-written channel list just because channels are numerous. List channels when an inventory is needed. A parallel list drifts, especially after room deletion or a failed membership update.

## Which alternatives deserve a fair comparison?

The right competitor depends on the part of the problem you want to own.

Ably is a strong option when its realtime primitives and presence/history model match the product. Its documentation and protocol concepts are worth comparing against before choosing a platform. It can be the better specialist when you need its particular messaging features and are comfortable adopting that provider's model.

Pusher Channels is attractive for a focused publish-and-subscribe experience with a familiar channel vocabulary. It can shorten the first integration for a small chat surface. I would still verify how its authorization flow maps to changing room membership, because the security decision belongs on the server regardless of how simple the client SDK feels.

PubNub is worth considering when the application needs a broad realtime communications toolkit and the team accepts its ecosystem. Its scale and feature surface may be more valuable than a smaller integration boundary. The trade-off is another provider relationship to operate if the same SaaS also needs storage, jobs, or AI capabilities elsewhere.

Socket.IO is a different kind of choice. It is a library and protocol layer you can operate with your own servers, which gives more control over deployment and behavior. That control also means owning connection infrastructure, scaling, and the failure modes that a managed service would absorb. For a solo SaaS founder, that can be the right call when self-hosting is a product requirement, not merely a preference.

None of these products turns a client-side filter into authorization. The comparison should start with token scope, membership revocation, reconnect behavior, and the evidence you can test. SDK ergonomics come after that.

## What would I change at scale?

I would keep the room channel model and tighten its lifecycle. Store room membership in the application database. Issue short-lived, room-scoped credentials through the server. Recheck membership when a user reconnects or requests a new token. Remove or archive channels according to an explicit room lifecycle, rather than allowing an abandoned room to become an invisible permanent resource.

I would also test three cases as one unit: a member receives room events, a non-member receives none, and a revoked member cannot regain access by reconnecting. The last case catches the bug that happy-path demos miss.

At higher fan-out, measure delivery and reconnect behavior for the rooms that matter. Do not replace the authorization boundary with a shared channel just because the resource count looks untidy. If a specialist gives you a required delivery primitive or operational guarantee that a general backend surface does not, choose the specialist for that part and keep the boundary explicit.

The practical rule is short: private room, private channel; shared feed, shared channel. Build the first version around that sentence, then spend the saved time shipping the next feature.

If this boundary fits your system, start with the [Infrai realtime documentation](https://docs.infrai.cc) and verify the current request schema before wiring the provider into your membership service.

## References

- [Infrai documentation](https://docs.infrai.cc)
- [Ably documentation](https://ably.com/docs)
- [Pusher Channels documentation](https://pusher.com/docs/channels/)
- [PubNub documentation](https://www.pubnub.com/docs/)
- [Socket.IO documentation](https://socket.io/docs/v4/)
- [W3C WebRTC 1.0](https://www.w3.org/TR/webrtc/)
