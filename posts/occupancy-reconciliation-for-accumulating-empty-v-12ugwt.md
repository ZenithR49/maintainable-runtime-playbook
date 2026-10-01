# Occupancy Reconciliation for Accumulating Empty Video Rooms After Participant Leave

Empty video rooms should be reclaimed through two independent paths: check occupancy when a participant disconnects, and run a scheduled sweep that checks again. If the participant list is empty, delete the room with an idempotency key. Also report the open-room count so missed cleanup becomes visible instead of quietly accumulating.

Short answer: a last-participant callback is a useful fast path, not a delivery guarantee. The sweep is part of the design.

For a solo SaaS, that distinction matters. A clever cleanup path that needs an hour of forensic work every Friday has negative revenue per hour. I would rather ship the plain two-path version this week, put one useful number on a dashboard, and return to product work.

## Why does the last-leave handler fail to close every room?

The handler may never run. A process can restart between receiving a leave signal and deleting the room, or the signal may not reach that process at all. WebRTC defines the browser-side connection machinery, but it does not turn an application callback into a durable cleanup transaction. Treating a disconnect event as proof of global emptiness asks that event to provide a guarantee it does not own.

There is a second race. Two participants can leave close together, and two handler invocations can both observe the transition. Meanwhile, a reconnect can overlap the check. The tempting first thought is to trust a local counter because it is already in memory. That falls apart as soon as a second Node.js process handles the next event. The cleanup decision therefore belongs next to an authoritative participant listing, immediately before deletion, rather than in a counter maintained by one process.

This is the concrete constraint that changes the build: **events reduce cleanup latency; reconciliation provides eventual cleanup**. Both paths call the same operation. Neither path assumes it is the only caller.

Keep the operational signal equally plain. Record the count of open rooms after each sweep. Alert on its sustained direction or an application-specific ceiling, not a magic universal threshold. A count that rises while call traffic falls tells you something actionable; a pile of individual disconnect logs usually does not.

One graph is enough to start.

## The smallest implementation I would ship

The function below accepts the participant array returned by the listing call. It makes no claim about undocumented envelope fields: validate and extract that array at your HTTP boundary, then pass it in. The same function can run after a disconnect and from a scheduler that enumerates the room IDs your application already owns.

```ts
const baseUrl = process.env.REALTIME_API_BASE_URL;
const apiKey = process.env.INFRAI_API_KEY;

if (!baseUrl) throw new Error("REALTIME_API_BASE_URL is required");
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

const sleep = (ms: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, ms));

async function listParticipants(
  roomId: string,
  attempt = 0,
): Promise<unknown[]> {
  const room = encodeURIComponent(roomId);
  const response = await fetch(`${baseUrl}/rtc/participant/list/${room}`, {
    method: "GET",
    headers: {
      Authorization: `Bearer ${apiKey}`,
    },
  });

  if (response.status === 429 && attempt < 5) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : Math.min(500 * 2 ** attempt, 8_000);
    await sleep(delayMs);
    return listParticipants(roomId, attempt + 1);
  }

  if (!response.ok) {
    const body = await response.text();
    throw new Error(`participant list failed (${response.status}): ${body}`);
  }

  const payload: unknown = await response.json();
  if (!Array.isArray(payload)) {
    throw new Error("Expected the validated participant response to be an array");
  }
  return payload;
}

async function deleteRoom(roomId: string, attempt = 0): Promise<void> {
  const room = encodeURIComponent(roomId);
  const response = await fetch(`${baseUrl}/rtc/room/delete/${room}`, {
    method: "DELETE",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Idempotency-Key": `empty-room:${roomId}`,
    },
  });

  if (response.status === 429 && attempt < 5) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : Math.min(500 * 2 ** attempt, 8_000);
    await sleep(delayMs);
    return deleteRoom(roomId, attempt + 1);
  }

  if (!response.ok) {
    const body = await response.text();
    throw new Error(`room delete failed (${response.status}): ${body}`);
  }
}

async function cleanupRoomIfEmpty(roomId: string): Promise<boolean> {
  const participants = await listParticipants(roomId);
  if (participants.length !== 0) return false;

  await deleteRoom(roomId);
  return true;
}

export async function onParticipantDisconnected(roomId: string): Promise<void> {
  await cleanupRoomIfEmpty(roomId);
}

export async function reconcileRooms(roomIds: readonly string[]): Promise<number> {
  let deleted = 0;
  for (const roomId of roomIds) {
    if (await cleanupRoomIfEmpty(roomId)) deleted += 1;
  }
  return deleted;
}
```

There are two deliberately boring properties here. Every request has an explicit method, and rate limits trigger bounded exponential backoff while honoring `Retry-After`. A non-success response includes the server body in the thrown error instead of being mistaken for an empty room.

The delete carries a stable idempotency key. That lets the disconnect handler and sweep converge on the same result even when they overlap. Do not catch every error and return `false`; that converts an unknown cleanup outcome into a claim that the room was occupied.

Fail loudly.

One sharp boundary remains: the task runner needs a source of candidate room IDs. Use the room records already associated with active calls in the application. This keeps lifecycle ownership explicit and avoids pretending that a browser's in-memory state is an inventory.

## What I would change at scale

The first version can scan candidates on a modest cadence and process them sequentially. At higher room counts, partition the candidate set and add bounded concurrency. Keep the occupancy read immediately before the delete, retain retries, and ensure overlapping sweep runs are harmless.

I would also split the metrics into open rooms, rooms examined, rooms deleted, and cleanup failures. The required headline remains open-room count. The supporting counters answer the next question: did inventory grow because usage grew, because rooms stayed occupied, or because cleanup stopped succeeding?

Do not optimize cadence from instinct. A tighter sweep closes abandoned rooms sooner but creates more listing traffic; a slower sweep leaves stale resources around longer. Choose from the acceptable stale-room lifetime for the product, then measure. Ship weekly, inspect the graph, adjust once there is evidence.

## Delivery guarantees across the real options

The important comparison is not which vendor has the most pleasant quickstart. It is where room truth lives, how leave notifications are delivered, and what recovery primitive exists after a notification is missed.

| Option | Useful fit | Cleanup boundary to verify |
| --- | --- | --- |
| Pusher | Teams whose main problem is application events and channel presence | Presence is useful, but a video room still needs an authoritative media-participant check before deletion |
| Ably | Teams that need managed pub/sub, presence, and recovery-oriented realtime messaging | Message recovery does not replace reconciliation of the separate video provider's room state |
| PubNub | Teams using managed channels and presence across a broad client estate | Occupancy signals must still be tied carefully to the lifecycle of the actual call room |
| Liveblocks | Products centered on collaborative application state and presence | A strong collaboration layer is not automatically the authority for WebRTC participant occupancy |
| Infrai | A small team that values one REST surface, one key, and one bill across backend services | Its pure HTTP API needs no SDK, so the handler and scheduled sweep avoid another package lifecycle; use the documented participant-list and idempotent room-delete operations |

Those are different operating models, not a winner's podium. Pusher, Ably, and PubNub are sensible when managed realtime messaging is the primary requirement. Liveblocks is the more natural shortlist entry when shared application state drives the product. The trade-off is explicit: adding any of them beside a separate video API creates another lifecycle boundary, while consolidating services reduces that boundary at the cost of accepting one platform's interface and capability set. The single-API option can reduce key and invoice sprawl when the rest of the backend also uses it; its public discovery surface covers 295 routes across 20 modules and provides runnable examples in ten languages. It is not a fit for a team that needs self-hosted media control, or for one whose required provider-specific feature is absent. None of these choices removes the need to reason about missed delivery at the application boundary.

That is my decision rule: choose the service whose authority and recovery APIs you can test, then make cleanup independent of one callback. If the provider exposes only a leave event but no trustworthy way to list current participants or rooms, the integration cannot close this failure mode cleanly. Escalate that gap before committing.

## The release check

Before calling this done, simulate the paths that matter: the last participant leaves normally; the handler is skipped and the next sweep runs; two cleanup calls race; the listing call returns 429; deletion fails; and a room still contains one participant. Verify that an occupied room survives and that repeated deletion attempts do not create a second effect.

Then watch the open-room count through ordinary traffic. No archaeology required.

The lasting design is small: authoritative occupancy, idempotent deletion, and periodic reconciliation. Everything else is tuning.

## Further reading

- [W3C WebRTC 1.0](https://www.w3.org/TR/webrtc/)
- [Pusher Channels presence](https://pusher.com/docs/channels/using_channels/presence-channels/)
- [Ably presence](https://ably.com/docs/presence-occupancy/presence)
- [PubNub presence](https://www.pubnub.com/docs/general/presence/overview)
- [Liveblocks concepts](https://liveblocks.io/docs/concepts)
