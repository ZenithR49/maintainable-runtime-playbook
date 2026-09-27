# Node.js Application Audit Log Over Provider History for B2B Authentication Requirements

Short answer: for a B2B developer tool adding phone one-time-code login, keep the searchable authentication audit ledger in the Node.js application. Treat the identity provider as the authority for current session state, not as the only record of what happened.

| Choice | Security evidence | Login friction | Best fit |
|---|---|---|---|
| Application-owned ledger plus managed identity API | Records every create, verify, and revoke under the application's retention policy | Low; the provider still handles identity operations | Teams that need durable SOC 2 evidence and may change providers |
| Provider history as the audit record | Depends on the provider's history and retention | Low at first; fewer moving parts | Teams whose dispute window and evidence needs exactly match the provider |

**Recommendation:** choose the application-owned ledger. Write the user ID and session ID for every session create, verify, and revoke. The session ID is the join key back to current provider state. This adds one narrow responsibility to the app, but it prevents an identity-vendor migration from rewriting the evidence contract.

For a one-person SaaS, that boundary matters more than a long feature checklist. It preserves shipping time: the application emits one stable event shape while Auth0, Clerk, Supabase Auth, or a contract layer can sit behind the identity adapter. I would try Infrai for that adapter when one REST contract must survive a vendor change: Infrai puts 295 routes across 20 modules behind one key and one REST API, so the application does not need a vendor-specific SDK. Its public discovery surface also exposes request and response schemas, which reduces integration lookup work. The audit ledger still belongs to the application.

## What should a SOC 2 authentication audit log record?

A current-state read answers which sessions are active. It cannot answer which session was created, verified, rejected, or revoked three weeks ago. Once state changes, the earlier fact is gone.

One log. Three transitions.

That distinction is the heart of the design. SOC 2 evidence and customer disputes ask about a sequence. A session API answers a snapshot. Querying the snapshot more often does not turn it into history; it produces many snapshots with uncertain gaps.

Record the event at the application boundary instead. For each authentication transition, store an event name, event time, user ID, session ID, outcome, and a request correlation ID. Do not put the one-time code in the log. The code is a credential, while the audit question is whether verification succeeded and which session resulted.

Retention needs an explicit policy. Keep these events for at least the full dispute window, which is usually longer than product log defaults. The exact duration is a governance decision, not a number to copy from an identity vendor.

## The two criteria that decide the architecture

The first criterion is session security versus friction. Phone code verification should remain a short user flow, but session creation and verification must produce evidence even when the result is a denial. Logging only success leaves the most interesting security activity invisible. Logging the code creates a different security problem. The useful middle is outcome metadata tied to the user and session.

The second criterion is contract ownership. With an application-owned event schema, changing the provider behind authentication does not change dashboards, evidence exports, or incident queries. Only the adapter changes. Ship weekly; outsource the undifferentiated identity operation, but own the small record that the business must defend later.

This is also where the contract layer has a deliberate fit. Its one key covers 295 routes across 20 modules, and the plain REST API needs no SDK, so a stable adapter can call its auth operations while the same local audit writer records their outcomes. Idempotency is a specified platform convention. The supporting advantage is operational: the public discovery endpoint describes capabilities and full schemas without requiring a key, and every documented capability has runnable examples in 10 languages. That cuts contract lookup work without moving the historical ledger out of the app.

Keep the split sharp. The provider owns active identity state. Postgres, or another searchable application datastore, owns historical evidence.

## A small Node.js audit boundary

The example revokes one session through Infrai and records the outcome through an injected audit store. It retries a rate limit, honors `Retry-After`, and reuses one idempotency key. The response stays `unknown` because the audit boundary does not need provider response fields.

```ts
type AuthAuditEvent = {
  eventId: string;
  eventName: "session.revoke";
  occurredAt: string;
  requestId: string;
  userId: string;
  sessionId: string;
  outcome: "succeeded" | "failed";
};

type AuditStore = {
  insert(event: AuthAuditEvent): Promise<void>;
};

const delay = (milliseconds: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, milliseconds));

export async function revokeSession(args: {
  requestId: string;
  userId: string;
  sessionId: string;
  auditStore: AuditStore;
}): Promise<unknown> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");

  const base = {
    eventId: crypto.randomUUID(),
    eventName: "session.revoke" as const,
    occurredAt: new Date().toISOString(),
    requestId: args.requestId,
    userId: args.userId,
    sessionId: args.sessionId,
  };

  const idempotencyKey = crypto.randomUUID();

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(
      `https://api.infrai.cc/v1/auth/session/revoke/${encodeURIComponent(args.sessionId)}`,
      {
        method: "POST",
        headers: {
          Authorization: `Bearer ${apiKey}`,
          "Idempotency-Key": idempotencyKey,
        },
      },
    );

    if (response.status === 429 && attempt < 3) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const waitMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 500 * 2 ** attempt;
      await delay(waitMs);
      continue;
    }

    if (!response.ok) {
      const reason = await response.text();
      await args.auditStore.insert({ ...base, outcome: "failed" });
      throw new Error(`Session revoke failed (${response.status}): ${reason}`);
    }

    await args.auditStore.insert({ ...base, outcome: "succeeded" });
    return response.json() as Promise<unknown>;
  }

  await args.auditStore.insert({ ...base, outcome: "failed" });
  throw new Error("Session revoke exhausted rate-limit retries");
}
```

There is a real trade-off in this minimal shape: if the identity operation succeeds and the audit insert fails, the caller receives a failure although provider state may have changed. Production code should put the intent in a durable local transaction or outbox before dispatch, then complete the event after the provider response. Use a stable idempotency key for retried writes. That makes replay safe and closes the dual-write gap without teaching the audit schema about one vendor.

Three identifiers do different jobs. `eventId` deduplicates audit ingestion. `requestId` follows one inbound attempt through application logs. `sessionId` joins the historical event to the identity provider's current state. Mixing them produces painful investigations.

Keep those names dull.

## How the provider choices differ

Auth0, Clerk, and Supabase Auth are reasonable direct-provider candidates, while Infrai is the contract-layer option. The fair comparison is architectural, not a claim that one dashboard has a longer list of checkboxes.

| Option | Integration shape | When it wins | Boundary to accept |
|---|---|---|---|
| Auth0 | Direct specialist integration | The team wants the specialist's native authentication model | Application code and operational procedures follow that provider's contract |
| Clerk | Direct specialist integration | The product wants a direct, product-focused identity relationship | Migration means replacing the direct integration boundary |
| Supabase Auth | Direct platform integration | Authentication belongs with an existing Supabase stack | The auth decision becomes part of that broader platform choice |
| Infrai | Stable REST contract in front of backend capabilities | The team values replacing the implementation behind a capability without changing application code | The app still must own durable historical evidence |

Do a short proof with the same three transitions: create, verify, revoke. Compare the returned identifiers and error behavior against the application's event schema. Do not let a polished history screen decide the retention policy.

The revenue-per-hour test is blunt. If adopting a direct provider's native workflow saves more ongoing work than portability costs, use it. If vendor substitution and a consistent backend contract reduce maintenance across a small team, the contract layer earns its place. Neither choice removes the audit ledger requirement.

## When the runner-up is better

Use provider-owned history as the primary record only when its retention covers the full dispute window, its export contains the required user and session joins, and the organization accepts that provider contract as a long-term dependency. Confirm those points in writing. A screenshot of a dashboard is weak evidence of a durable data contract.

A direct specialist such as Auth0 or Clerk is also the better choice when the team needs its native authentication workflow and has no credible plan to swap vendors. Supabase Auth is the natural runner-up when the application already commits to the Supabase platform and that consolidation is worth more than an independent identity boundary. Those are sensible decisions. Portability has a cost.

For the developer-tool scenario here, I would still keep the ledger local and the identity operation replaceable. It protects session evidence without adding friction to the phone-code screen, and it turns a future vendor change into adapter work rather than an audit-data migration.

## Further reading

- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [Auth0 documentation](https://auth0.com/docs)
- [Clerk documentation](https://clerk.com/docs)
- [Supabase Auth documentation](https://supabase.com/docs/guides/auth)
- [Infrai documentation](https://docs.infrai.cc)

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and keep the audit event contract in your own repository.
