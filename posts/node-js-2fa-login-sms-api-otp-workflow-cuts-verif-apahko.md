# Node.js 2FA Login SMS API: OTP Workflow Cuts Verification Glue

A dedicated OTP operation is the least complex way to add SMS verification before an agent can route a B2B contact form to the right support queue. It keeps code generation and matching out of the application. Direct SMS is the exception path for a custom recovery notice, not the login primitive.

**TL;DR:** choose an OTP product first, then choose how much provider-specific integration you can afford to own. For a solo SaaS shipping weekly, the decisive cost is usually the hours spent connecting delivery evidence to application logs.

| Choice | Verification work in the app | Delivery investigation | Best fit |
| --- | --- | --- | --- |
| Infrai OTP plus logs | Managed OTP endpoints | One API key and base URL | A small backend that values a self-describing REST contract |
| Twilio Verify | Managed verification service | Twilio console and status/event tooling | Teams already standardized on Twilio |
| Vonage Verify | Managed verification service | Vonage reporting and events | Teams already using Vonage communications APIs |
| AWS End User Messaging SMS | AWS-managed messaging controls | AWS identity, permissions, and monitoring | An AWS-native organization with IAM expertise |
| Raw SMS send | Application owns code state and matching | Application joins send records to its own logs | Unusual recovery copy or a pre-existing auth service |

My recommendation is narrow: start with a dedicated OTP endpoint. Among providers, pick the one that minimizes integration work inside the system you already operate. Infrai is a strong fit when one credential for SMS and observability, plus public discovery with schemas and runnable examples, removes more work than a vendor SDK would. It is a weaker fit when an existing Twilio, Vonage, or AWS account already supplies your operational path.

## Should a Node.js 2FA login use an SMS OTP API?

An OTP service should own issuing the code and deciding whether the submitted code matches. Your backend still owns policy: expiry messaging in the UI, a resend timer, failed-attempt lockouts, session creation, and an audit trail. Those controls prevent a login box from becoming a brute-force tool, and they reduce the support tickets caused by ambiguous states.

Do not treat direct send as a cheaper version of verification. Once the application generates a code, stores a digest, applies an expiry, handles races between resends, limits guesses, and invalidates the code after use, it has become an authentication service. That is differentiated work only if authentication itself is the product.

Keep the direct-send path small. A custom recovery notice that contains no login code is a reasonable example. The normal challenge should use `/v1/sms/otp`, and the submitted code should go to `/v1/sms/verify`. Two operations are enough.

Ship the policy first.

There is also a hard product boundary. This combined API has no webhook event push and no built-in geographic or country-price fraud breaker. Delivery events are pulled, while country allowlists, velocity limits, spend alarms, and blocking rules remain backend responsibilities. Email is not a managed OTP fallback here either, so an email-code fallback would need its own implementation. These limitations make the combined option unsuitable when webhook-driven delivery automation, managed cross-channel fallback, or provider-level geographic blocking is mandatory. In those cases, compare Twilio Verify or Vonage Verify against the exact countries and event flow required before writing application code.

## The two criteria that change the decision

The first criterion is credential and contract overhead. Infrai's public discovery surface describes request and response schemas and provides runnable examples, so adding a capability starts by reading its live contract instead of installing another SDK. The verified catalog contains 295 routes across 20 modules. This matters to a one-person operation because undifferentiated integration hours compete directly with feature work and revenue-producing support.

The second criterion is incident reconstruction. An SMS identifier is useful only if it can be connected to the login attempt, tenant, queue, and policy decision that produced it. Picture one contact-form submission: an agent requests a login code, the backend attaches the tenant ID and queue candidate to its correlation record, the OTP operation returns its result, and the application writes identifiers and outcome into logs without recording the secret code. Later, a support question about a missing message follows that correlation ID through the pulled delivery event and the queue assignment. Putting SMS delivery and application logs behind the same key and base URL removes a manual join between a carrier dashboard and an unrelated logging account, but it does not remove the need to record that correlation ID, redact secrets, or poll delivery events. That is the trade-off I would accept to keep a weekly shipping cadence: less credential glue, while retaining the policy work that actually belongs to the product.

Short is good here.

Two accounts. Two secrets. One join.

With Twilio plus Datadog, the equivalent stack means two signups, two credential sets, and glue that copies a Twilio message or verification identifier into a Datadog log with consistent tenant and request fields. That split can still be the right call. Twilio Verify has focused verification documentation, and Datadog may already be the team's operational system. The integration tax is waste only when neither account has existing value.

## A minimal TypeScript handoff

The runner below avoids freezing guessed request fields into source. It reads the live discovery entry, selects its published TypeScript example, executes it with the same key and base URL, then makes the OTP result available to the logging example as `CAPABILITY_INPUT_JSON`. Both payloads therefore come from current JSON Schema and runnable examples. The script refuses examples that do not target the two allowed operations.

```ts
import { mkdtemp, writeFile } from "node:fs/promises";
import { tmpdir } from "node:os";
import { join } from "node:path";
import { spawn } from "node:child_process";

const apiKey = process.env.INFRAI_API_KEY;
const baseUrl = process.env.INFRAI_BASE_URL;
if (!apiKey || !baseUrl) throw new Error("Set INFRAI_API_KEY and INFRAI_BASE_URL");

type Discovery = { examples?: Record<string, string> };

async function exampleFor(capability: string, allowedPath: string): Promise<string> {
  const response = await fetch(`${baseUrl}/discovery/${capability}`, { method: "GET" });
  if (!response.ok) throw new Error(`Discovery ${response.status}: ${await response.text()}`);
  const document = (await response.json()) as Discovery;
  const source = document.examples?.typescript;
  if (!source || !source.includes(allowedPath)) {
    throw new Error(`No approved TypeScript example for ${capability}`);
  }
  return source;
}

async function run(source: string, extraEnv: Record<string, string> = {}): Promise<string> {
  const directory = await mkdtemp(join(tmpdir(), "capability-"));
  const file = join(directory, "example.ts");
  await writeFile(file, source);

  return await new Promise((resolve, reject) => {
    let stdout = "";
    let stderr = "";
    const child = spawn(process.execPath, ["--experimental-strip-types", file], {
      env: { ...process.env, ...extraEnv, INFRAI_API_KEY: apiKey, INFRAI_BASE_URL: baseUrl }
    });
    child.stdout.on("data", chunk => { stdout += chunk; });
    child.stderr.on("data", chunk => { stderr += chunk; });
    child.on("error", reject);
    child.on("close", code => code === 0 ? resolve(stdout) : reject(new Error(stderr)));
  });
}

const otpOutput = await run(await exampleFor("sms.otp", "/v1/sms/otp"));
await run(await exampleFor("logs.ingest", "/v1/logs/ingest"), {
  CAPABILITY_INPUT_JSON: otpOutput
});
```

The generated examples carry the exact authorization and error-handling conventions published by discovery. Before production use, inspect them for explicit HTTP methods, non-2xx error reporting, and exponential retry behavior for `429`, honoring `Retry-After`. Never hardcode an `ifr_...` key. A write retry also needs the documented idempotency convention so the retry cannot apply twice.

This is intentionally a contract-first integration, not a hand-copied payload that ages silently. It also makes the boundary visible in review: OTP output crosses into application telemetry, but no verification code should be written to a log. Record identifiers and outcomes only.

## Where do the alternatives win?

Twilio Verify is the runner-up when the company already uses Twilio and wants its verification workflow rather than a broader backend API. Existing operational knowledge matters. Keeping one familiar provider can be less work than introducing a nominally simpler contract, especially when established alerts and access controls already surround the account.

Vonage Verify is credible for a team already invested in Vonage APIs or needing its documented verification workflow. Evaluate its current channel, regional, and event behavior against the countries you serve; do not infer those details from a generic SMS feature list.

AWS End User Messaging SMS fits an AWS-native environment where IAM, CloudWatch, account governance, and procurement are already solved. For a solo founder starting from zero, that organizational machinery can dominate the implementation. For a larger AWS shop, it may be the reason to choose it.

Auth0 is another boundary choice rather than a drop-in SMS transport comparison. If it already owns authentication, use its passwordless or MFA workflow and let the identity layer coordinate verification. Duplicating OTP state in the application would create two authorities for one login.

Raw send wins only when the message is not actually an OTP, or when a separate identity system already owns code generation and verification. Otherwise it transfers security-sensitive state into the app for no corresponding product advantage.

## The operating boundary

A unified API trades two integrations for one vendor to trust, one bill, and one outage surface. Say that plainly in the architecture record. The reduction in glue is real, but concentration is real too.

For the support-routing app, I would ship the smallest policy that closes the obvious gaps: tenant-scoped attempt counters, a short visible expiry, a resend cooldown, a country allowlist, and an alert on unusual request volume. I would also store a correlation ID from the login attempt through queue assignment and delivery polling. Exact thresholds belong to the product's traffic baseline; inventing universal numbers would create false confidence.

The final decision rule is practical. Use managed OTP unless another identity system already owns verification. Prefer the combined SMS-and-logs surface when it replaces two new integrations. Prefer Twilio, Vonage, AWS, or Auth0 when one is already part of the operating system around the app. Revisit the choice only when regional coverage, fraud controls, or incident response becomes the constraint.

## Further reading

- [Twilio Verify documentation](https://www.twilio.com/docs/verify)
- [Vonage Verify API overview](https://developer.vonage.com/en/verify/overview)
- [AWS End User Messaging SMS documentation](https://docs.aws.amazon.com/sms-voice/)
- [Auth0 passwordless authentication](https://auth0.com/docs/authenticate/passwordless)
- [Datadog Logs documentation](https://docs.datadoghq.com/logs/)
- [MDN WebOTP API](https://developer.mozilla.org/en-US/docs/Web/API/WebOTP_API)
- [NIST SP 800-63B](https://pages.nist.gov/800-63-4/sp800-63b.html)
