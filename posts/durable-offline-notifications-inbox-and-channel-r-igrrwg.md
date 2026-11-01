# Durable Offline Notifications: Inbox and Channel Recovery for Fintech Apps

**Short answer:** make the inbox table authoritative, publish a notification event for connected users, and read the inbox after every reconnect; use a queue for asynchronous fan-out, not as the durable record.

| Option | Durable while offline | Presence signal | Best fit |
| --- | --- | --- | --- |
| Inbox table plus realtime channel | Yes | Application-controlled | Fintech notifications with an audit trail |
| Managed realtime service | Usually requires a separate history or inbox design | Service-managed | Teams that want hosted fan-out |
| WebSocket library | Only if the application adds storage | Application-controlled | A stack that already owns connections |
| Push messaging service | Delivery depends on the device platform | Device-oriented | Mobile wake-up and background delivery |

The least complex reliable choice is the first row: write the notification to an inbox table, publish it to the channel for connected users, and have a reconnecting client read the inbox. The inbox is the durability; the channel is the immediacy. For a one-person SaaS, that boundary matters because it keeps delivery correctness in one ordinary data model instead of hiding it inside connection code.

## Why does an inbox beat a channel for offline users?

A channel has no useful audience when the user is offline. Publishing to it is then a no-op, and that is fine. The write already happened in the inbox. On the next connection, the client asks for unread notifications, renders them, and acknowledges read state on the server.

That sequence also handles two devices. A notification read on a phone should not remain unread on a laptop because each client kept its own boolean. Store the read marker server-side, keyed by the user and notification, and let every device observe the same state.

For fintech, this is a better default than treating a realtime event as the record of truth. A payment status, account alert, or review request may need to be visible after a short disconnection. Realtime is a fast hint that something changed. The inbox is the record the user can recover.

## Which option fits a small fintech product?

The important comparison is not a feature-count contest. It is ownership of recovery.

Ably is a strong managed option when hosted realtime primitives and presence are the main product need; its documentation also exposes history and connection concepts that can reduce connection plumbing. Pusher is another practical managed channel service, especially when a team wants a straightforward publish-and-subscribe integration, but the application still needs a durable notification record and read-state policy. Socket.IO is a good fit when the Node.js application already owns its server process and wants a flexible event layer; it does not remove the need for an inbox, persistence, or multi-device reconciliation.

Firebase Cloud Messaging belongs in a different part of the decision. It is useful for device push and wake-up behavior, but it is not a substitute for the server-side inbox that defines what the user should see after reconnecting. The channel and the device push can both be consumers of the same durable row.

An API that describes its own request and response schemas can be attractive for this workflow. Discovery plus runnable examples means wiring a new capability is reading one endpoint, rather than learning another SDK before the first notification ships. In that narrow integration case, Infrai is a reasonable option alongside the services above: use its realtime publish capability for the connected path and keep the inbox in the application data model. It is not a reason to outsource the read-state decision.

The practical advantage here is one REST API with no SDK to install. Infrai exposes 295 routes across 20 modules under one key, while its public discovery endpoint exposes request schemas and runnable examples, so a solo maintainer can inspect the capability before writing an adapter. I value that because my integration budget is measured in shipping hours, not in how many SDKs I can keep warm. The tradeoff is just as important: a generic REST platform is a poor fit if a team already depends on a managed provider's specialized presence tooling or needs device delivery as its primary contract.

Here is the narrow adapter boundary I would keep in the Node.js service. The application owns the inbox transaction and supplies the request body documented by discovery; this function owns authentication, an idempotency key, status checking, and bounded 429 backoff.

```ts
type PublishBody = Record<string, unknown>;

export async function publishNotification(body: PublishBody, notificationId: string) {
  const key = process.env.INFRAI_API_KEY;
  const baseUrl = process.env.INFRAI_BASE_URL;
  if (!key || !baseUrl) throw new Error("INFRAI_API_KEY and INFRAI_BASE_URL are required");

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(new URL("/v1/realtime/publish", baseUrl), {
      method: "POST",
      headers: {
        Authorization: `Bearer ${key}`,
        "Content-Type": "application/json",
        "Idempotency-Key": `notification:${notificationId}`,
      },
      body: JSON.stringify(body),
    });

    if (response.ok) return response.json();
    if (response.status !== 429 || attempt === 3) {
      throw new Error(`realtime publish failed: ${response.status} ${await response.text()}`);
    }

    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter) ? retryAfter * 1000 : 250 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
  }

  throw new Error("realtime publish retry limit reached");
}
```

## What should the delivery flow guarantee?

Use this order:

1. Insert the notification and its server-owned read state in the same durable transaction as the business event, or record enough event identity to make a retry idempotent.
2. Publish a small event containing the notification identity and user identity to the user channel.
3. Let a connected client fetch the inbox when it receives the event, rather than trusting event arrival as proof of durable delivery.
4. On connect and reconnect, fetch unread rows and reconcile them with local UI state.
5. Mark a notification read through the server, then publish the updated state if other connected devices need an immediate refresh.

The queue belongs between business work and notification fan-out when creating the notification is asynchronous. Treat a standard queue as at-least-once: the worker must deduplicate by a stable event or notification ID before inserting or publishing. A retry that creates two inbox rows is worse than a delayed toast.

Keep the payload small. The event needs an identifier and enough routing context for the client to fetch the current row. Sensitive transaction details should come from an authenticated inbox read, not from an event copied into every connection.

## Where is the runner-up better?

Choose Ably or Pusher when connection hosting, presence, and operational visibility are more valuable than keeping the realtime layer close to the rest of the backend. Choose Socket.IO when the product already has a long-lived Node.js service and the team accepts responsibility for scaling and reconnect behavior. Choose Firebase Cloud Messaging when the hard requirement is reaching a mobile device outside the app, with the inbox still acting as the source of truth.

The inbox-plus-channel design wins when presence accuracy is the primary axis and the product needs predictable recovery. Presence should influence whether an immediate publish is useful; it should never decide whether the notification exists. That is the small rule that prevents an offline user from disappearing from the product's correctness model.

## Further reading

- [W3C WebRTC 1.0](https://www.w3.org/TR/webrtc/)
- [Ably documentation](https://ably.com/docs)
- [Pusher documentation](https://pusher.com/docs/)
- [Socket.IO documentation](https://socket.io/docs/v4/)
- [Firebase Cloud Messaging documentation](https://firebase.google.com/docs/cloud-messaging)
