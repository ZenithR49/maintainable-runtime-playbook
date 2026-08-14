# Marketplace Password Recovery Email: A Deliverability Test for SPF, DKIM, and Suppression

Short answer: pick the email provider that passes a real password-reset test for your custom domain, DKIM, SPF, suppression list, and bounce handling with the fewest new systems to own. For a one-person marketplace, integration effort is the decision axis. A provider that needs a second dashboard, an unfamiliar event model, and a bespoke retry worker can cost more revenue-per-hour than its feature list suggests.

| Test | Accept when | Pause when |
| --- | --- | --- |
| Custom-domain identity | DNS records can be published and independently checked | The provider's dashboard is the only evidence of setup |
| SPF and DKIM | The sender identity and rotating DKIM keys are documented | The application must guess which records or selectors matter |
| Suppression | A pre-send check and a durable local decision are possible | The same bad address can be retried blindly |
| Bounce handling | Events reach the state machine within the product's time budget | Bounce state is too stale for account recovery |
| Integration | One small adapter covers send, event retrieval, and failure paths | Mail logic leaks across signup and recovery code |

That is the rule I would use before comparing plans. Ship weekly. Outsource the undifferentiated. Keep the security boundary and the state machine in your own application.

## What should a marketplace password reset email provider prove first?

Start with the identity of the sender, not the provider's sending volume claims. A marketplace account-recovery message needs to arrive promptly, but it also needs to be recognizable as legitimate mail from the domain your users already trust. The acceptance test should cover a custom sending domain, SPF publication, DKIM signing, and a key-rotation procedure that does not require rewriting the recovery flow.

The DNS work is part of the integration. Ask what record names and selectors are required, publish them in a test domain, then query the public DNS records independently. A green check in a control panel is useful feedback. It is not the final test. Keep the records, selector names, and ownership of the DNS change in the deployment runbook so the next person can reproduce the setup.

DKIM proves that the message was signed by an authorized sender and that its signed content was not changed in transit. SPF authorizes sending systems for the domain. They solve different parts of the trust problem. Neither one is a substitute for measuring the actual recovery flow.

There is a security constraint before deliverability: the reset request must not reveal whether an email belongs to an account. OWASP recommends a consistent response for existing and nonexistent accounts, consistent timing where practical, single-use tokens, and an expiry window. That means the public endpoint can say the same thing for both cases while the internal mail job decides whether there is a message to send.

Short. Specific. Testable.

## How do suppression lists and bounce handling change the recovery flow?

Treat suppression as a control loop, not as a report you read after a failed campaign. A user who clicks “send again” may create several requests while waiting. Before each delivery attempt, consult the current suppression state. If the address is suppressed, stop and show a recovery path that does not disclose private account state. Do not keep sending simply because the user asked twice.

Bounce handling closes the loop. A hard bounce or a provider-level block should become a durable event in your system; later recovery requests must consult that event. A transient failure needs a different policy from a permanent rejection. Do not let a generic `send failed` flag collapse those cases, because it gives support no useful answer and makes retries unsafe.

The event path has a practical choice. A push event can update a local record as soon as the provider delivers it. A pull-based event API needs a scheduled worker, a cursor or time window, deduplication, and a declared freshness target. Polling is fine for a small product when the recovery requirement tolerates its latency. It is a poor fit when a bounce must change user-visible behavior in seconds.

I would record three timestamps: reset request, provider event observed, and suppression state updated. Add the event identifier to a uniqueness constraint. Then test duplicate delivery, an out-of-order event, a retry after a permanent bounce, and a temporary failure that later clears. Those cases are more valuable than a claimed inbox percentage you cannot reproduce.

Test it twice.

On the first pass, create a reset request for a controlled address, capture the request ID, and verify that the public response is unchanged for a known and an unknown account. On the second pass, replay the same event and then deliver a permanent-bounce event out of order. The expected result is one suppression record, no duplicate side effect, and no later send to that address. Then let a temporary failure follow the documented retry policy and confirm that the cursor only advances after the event and its state transition are durable. This sequence is deliberately boring: it exercises the boundary between the scheduler, the provider event feed, and the database, which is where a small team's "just send an email" feature usually grows hidden ownership. Record the timestamps, but do not turn this one controlled run into a universal deliverability claim.

One hard boundary: never use the mail event as proof that a password-reset token was consumed. Token consumption belongs to the application, where it can be single-use and bound to an expiry policy. Mail delivery only tells you what happened to an attempt to deliver a message.

## How can a TypeScript adapter keep password-reset email provider choice reversible?

The adapter should expose the behavior the recovery system needs, not every option in a provider dashboard. Keep provider-specific payloads at the boundary. The rest of the application can work with a stable result and an event cursor. That boundary pays off during a real migration: the signup handler still creates its single-use token, the event worker still advances its cursor, and the suppression table still answers the same question even when the mail transport changes. You can test those three contracts without pulling an external dashboard into every integration test, which is a small but meaningful win when one person owns the release and the support queue.

```ts
type SendResetInput = {
  to: string;
  resetUrl: string;
  requestId: string;
};

type SendResult = {
  messageId: string;
};

type MailAdapter = {
  sendPasswordReset(input: SendResetInput): Promise<SendResult>;
  listEvents(cursor: string | null): Promise<{
    events: Array<{ id: string; kind: string; address: string }>;
    nextCursor: string | null;
  }>;
};

export async function requestReset(
  email: string,
  requestId: string,
  mail: MailAdapter,
): Promise<void> {
  const normalized = email.trim().toLowerCase();

  // Keep this lookup internal. The HTTP response stays identical either way.
  const account = await findAccountByEmail(normalized);
  if (!account || (await isSuppressed(normalized))) return;

  const token = await createSingleUseToken(account.id);
  await mail.sendPasswordReset({
    to: normalized,
    resetUrl: `https://market.example/reset?token=${encodeURIComponent(token)}`,
    requestId,
  });
}

async function syncMailEvents(mail: MailAdapter): Promise<string | null> {
  const cursor = await readEventCursor();
  const result = await mail.listEvents(cursor);

  for (const event of result.events) {
    if (event.kind === "permanent-bounce" || event.kind === "blocked") {
      await markSuppressed(event.address, event.id);
    }
  }

  await saveEventCursor(result.nextCursor);
  return result.nextCursor;
}

declare function findAccountByEmail(email: string): Promise<{ id: string } | null>;
declare function isSuppressed(email: string): Promise<boolean>;
declare function createSingleUseToken(accountId: string): Promise<string>;
declare function readEventCursor(): Promise<string | null>;
declare function markSuppressed(email: string, eventId: string): Promise<void>;
declare function saveEventCursor(cursor: string | null): Promise<void>;
```

The example leaves response fields behind the interface on purpose. Your provider's event schema must be checked before mapping its statuses to `permanent-bounce` or `blocked`; those names are application states, not universal protocol values. Also make the event consumer idempotent. A worker restart should not create a second suppression record or advance a cursor before the event is persisted.

In production, add bounded retry behavior for rate limits and transport failures, with jitter and an observable dead-letter path. Do not retry authentication failures or permanent recipient failures. Redact reset URLs from logs. A URL containing a live token is a credential, not harmless diagnostic text.

## When is a simpler or more controlled option better?

The recommendation is not universal. The catch is that a hosted transactional provider is not suitable when your compliance boundary requires mail infrastructure you operate and audit yourself. Choose a self-managed relay or a platform already approved by your security team when that ownership is a hard requirement, even if integration takes longer.

Choose a provider with push events when recovery behavior must react to bounce state within seconds. Choose a provider with a mature SMTP path when an existing mail system cannot be changed to HTTP without a wider migration. Conversely, for a solo SaaS with one reset-link flow, a narrowly scoped HTTP adapter is often easier to test and replace than a general mail subsystem. Stick with the existing approved mail system when its event freshness, domain controls, and audit trail already meet the recovery requirement; changing it only for a nicer API is integration work without a user-facing payoff.

The runner-up is better when it reduces operational ownership in the specific failure path you care about. This is where your mileage may vary. I am not sure what “prompt delivery” means for your marketplace until you set a measurable freshness target and identify the regions, mailbox types, and support hours that target covers.

Price can be part of the final comparison, but it should come after the control loop works. A low invoice does not compensate for a reset flow that cannot explain a bounce, suppress a known bad address, or rotate its signing key without a late-night DNS scramble.

For this flow, the durable decision is the one that passes the same test in staging and production: authenticated domain, verifiable SPF and DKIM, safe suppression, classified bounce events, and a small adapter your application owns. The provider is an interchangeable boundary. The recovery policy is not.

## References

- [OWASP Forgot Password Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html)
- [FTC CAN-SPAM Act compliance guide for business](https://www.ftc.gov/business-guidance/resources/can-spam-act-compliance-guide-business)
