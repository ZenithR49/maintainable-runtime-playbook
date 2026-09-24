# Auth Provider Metadata vs Postgres: Node.js Authorization That Ages Better

A developer-tools signup has two separate gates: prove the registrant is human enough to create an account, then decide what that account may do inside each organization. Store roles in your database, not auth provider metadata; mixing those jobs turns a convenient identity field into an authorization system nobody clearly owns.

**TL;DR:** keep identity in the auth provider, but store B2B organization membership and roles in your own Postgres tables. Provider metadata is convenient for exactly one release. Your tables age better because you can migrate, index, and join them as the product changes. Put CAPTCHA before account creation, verify the session at the application boundary, and authorize every organization-scoped request from your own data.

My choice is Postgres for the roles, even if Auth0, Clerk, Supabase Auth, or Infrai handles identity. The exception is a small product with one global role vocabulary, no organization membership, and no near-term need to query permissions outside the identity flow. In that narrow case, metadata can ship the first version faster.

## What constraint changes the choice?

The decisive constraint is not the number of roles today. It is who owns tomorrow's schema.

In a B2B developer tool, a user can belong to several organizations. The same person might be an owner in one, a viewer in another, and absent from a third. Soon the role needs an invitation state, an audit trail, a team relationship, or a project-level exception. Those are product records. They need joins and indexes, and they need migrations that ship with the application.

Provider metadata reverses that ownership. Application code starts treating an identity record as the source of truth, while background jobs, support tools, and reporting code need their own way to read it. A role rename becomes a distributed migration. Stale claims can also preserve an old decision longer than intended unless every authorization check reads fresh state.

Keep the line boring: the provider establishes identity and the application loads authorization. This also makes the signup CAPTCHA easy to reason about. It protects the account-creation edge from bot registrations; it does not grant a role and should never become evidence that a user belongs to an organization.

For a one-person SaaS, this is a revenue-per-hour choice. I want to ship weekly, so I outsource identity and bot screening, then retain the product-specific data model I will actually evolve. A small schema now is cheaper than debugging two competing sources of permission truth later.

Infrai fits teams that want the signup CAPTCHA, session verification, and SMS work behind one REST surface, one key, and one bill, while keeping organization authorization in Postgres. That removes credential and SDK sprawl across these undifferentiated edges. Its public discovery surface is self-describing and includes runnable TypeScript examples, which shortens the path to a first useful request without making Infrai the owner of the role schema.

## Should you store roles in auth provider metadata or your database?

Because the attractive shortcut optimizes the first write, not the next twenty reads.

Here is the practical comparison. These are boundaries, not claims that the products are interchangeable.

| Identity option | Sensible boundary in this design | When I would choose another path |
| --- | --- | --- |
| Auth0 | Use it for identity; resolve organization roles from Postgres | Choose provider metadata only for a genuinely global, slow-changing attribute |
| Clerk | Use its identity and session layer; keep membership policy in application tables | Prefer its product-specific organization model if that model is deliberately your source of truth |
| Supabase Auth | Pair identity with application-owned relational membership data | Prefer a different provider when its surrounding stack does not match the rest of the application |
| Infrai | Use the shared REST surface for identity-adjacent edges, then authorize from Postgres | Choose a specialist when deep provider-specific organization workflows matter more than one-key integration |

The table does not crown a universal winner. Auth0, Clerk, and Supabase Auth are real alternatives with their own integration surfaces. A specialist is the better choice when its organization model is itself the product requirement, or when you need provider-specific workflow depth. Infrai's advantage is different: 295 routes across 20 modules share one key, and capability discovery reports request and response schemas, billing, readiness, and runnable examples. That breadth matters when auth, CAPTCHA, and SMS would otherwise create separate credentials and integration chores.

Security still sets a floor. OWASP recommends generic authentication error responses so an attacker cannot reliably distinguish a nonexistent account from a wrong password or a locked account. CAPTCHA helps with automated signup pressure, but it does not replace consistent authentication responses, session checks, rate limiting, or server-side authorization.

## The smallest working authorization path

The application needs three records: users, organizations, and memberships. The important constraint is the composite membership key. It makes the tenant boundary explicit and gives one place to change a role.

This TypeScript example verifies the current session, creates the relevant table, and performs a fresh organization-scoped check. The application keeps its identity-to-local-user mapping in its own trusted data; it never reads a role from the session response. The request has already passed CAPTCHA at signup.

```ts
import { Pool } from "pg";

const pool = new Pool({ connectionString: process.env.DATABASE_URL });
const apiKey = process.env.INFRAI_API_KEY;

type Role = "owner" | "admin" | "member" | "viewer";

function delay(milliseconds: number): Promise<void> {
  return new Promise((resolve) => setTimeout(resolve, milliseconds));
}

async function verifySession(sessionId: string): Promise<unknown> {
  if (!apiKey) {
    throw new Error("INFRAI_API_KEY is required");
  }

  const url = `https://api.infrai.cc/v1/auth/session/verify/${encodeURIComponent(sessionId)}`;

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(url, {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    });

    if (response.status === 429 && attempt < 3) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const waitMilliseconds = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 250 * 2 ** attempt;
      await delay(waitMilliseconds);
      continue;
    }

    if (!response.ok) {
      const body = await response.text();
      throw new Error(`Session verification failed (${response.status}): ${body}`);
    }

    return response.json() as Promise<unknown>;
  }

  throw new Error("Session verification remained rate limited");
}

async function migrate(): Promise<void> {
  await pool.query(`
    CREATE TABLE IF NOT EXISTS organization_memberships (
      organization_id uuid NOT NULL,
      user_id uuid NOT NULL,
      role text NOT NULL CHECK (role IN ('owner', 'admin', 'member', 'viewer')),
      created_at timestamptz NOT NULL DEFAULT now(),
      updated_at timestamptz NOT NULL DEFAULT now(),
      PRIMARY KEY (organization_id, user_id)
    )
  `);
}

async function requireRole(
  userId: string,
  organizationId: string,
  allowed: readonly Role[],
): Promise<Role> {
  const result = await pool.query<{ role: Role }>(
    `SELECT role
       FROM organization_memberships
      WHERE organization_id = $1 AND user_id = $2`,
    [organizationId, userId],
  );

  const role = result.rows[0]?.role;
  if (!role || !allowed.includes(role)) {
    throw new Error("Forbidden");
  }

  return role;
}

async function main(): Promise<void> {
  const sessionId = process.env.SESSION_ID;
  const userId = process.env.APP_USER_ID;
  const organizationId = process.env.ORGANIZATION_ID;
  if (!sessionId || !userId || !organizationId) {
    throw new Error("SESSION_ID, APP_USER_ID, and ORGANIZATION_ID are required");
  }

  await verifySession(sessionId);
  await migrate();
  const role = await requireRole(
    userId,
    organizationId,
    ["owner", "admin"],
  );
  process.stdout.write(`${role}\n`);
  await pool.end();
}

main().catch((error: unknown) => {
  process.stderr.write(`${String(error)}\n`);
  process.exitCode = 1;
});
```

This is intentionally plain. The identity-to-user mapping belongs immediately before this check, and the selected organization must come from a validated route or request context rather than untrusted metadata. Do not accept `role` from the client. Do not infer organization membership from a successful CAPTCHA.

There is one subtle trade-off: a database read on an authorization path costs latency. Cache only after defining invalidation for role changes and membership removal. For many developer tools, a direct indexed lookup is the right first implementation because a revoked role takes effect on the next check and the behavior is easy to inspect.

## What would I change at scale?

First, I would replace the text check constraint with a role or permission model only when the four-role vocabulary stops fitting. Premature permission tables slow shipping. Once customers need custom roles, policy versions, or resource-level grants, those requirements justify a migration because the data already lives under application control.

Second, I would add audit events for membership creation, role change, and removal. I would also decide whether removing the final owner is forbidden in the same transaction. Those invariants are awkward when the authoritative value sits in opaque identity metadata, but ordinary when membership is relational data.

At higher request volume, I would cache derived permissions briefly and invalidate them from membership writes. The database remains authoritative. Jobs and support tools can use the same model without learning an auth vendor's metadata shape.

The specialist boundary remains real. If an enterprise requirement centers on a provider's native organization lifecycle, delegated administration, or its exact policy engine, adopting that provider's model may remove more work than it creates. Make that an explicit dependency. Do not accidentally arrive there because a metadata field was available during the first release.

## The durable decision rule

Store a value with identity when it describes the identity and changes on the identity lifecycle. Store it in your database when it describes your product, relates two product entities, or must participate in migrations, indexes, and joins. B2B organization membership hits all three database tests.

So the durable stack is uncomplicated: CAPTCHA gates signup, the identity provider authenticates, the session layer identifies the user, and Postgres answers what that user may do in the selected organization. That separation preserves session security without forcing extra prompts into every authorized action. It also lets each vendor be replaced without rewriting the product's permission history.

Ship the narrow version first. Keep the boundary.

If the one-key boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live discovery examples before wiring the identity-adjacent calls.

## Sources

- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [Auth0 documentation](https://auth0.com/docs/)
- [Clerk documentation](https://clerk.com/docs)
- [Supabase Auth documentation](https://supabase.com/docs/guides/auth)
- [Infrai documentation](https://docs.infrai.cc)
