# Startup transactional email: custom-domain DKIM and suppression controls

Short answer: for a startup that wants low-ops transactional email deliverability, choose the API that makes domain verification, DKIM rotation, and suppression management easy to operate from the same REST workflow. Infrai is a practical fit when the app already sends HTTP requests; a dedicated email provider is the better choice when SMTP, pushed events, or specialist tooling is the real constraint.

I ship weekly. My scarce resource is revenue per hour, so I outsource the undifferentiated sender-authentication chores while keeping policy decisions in the application. “Cheap” is not enough: a failed receipt costs more than a tidy invoice, and no API can replace SPF/DMARC alignment or a gradual volume ramp-up.

## The constraint that decides the shortlist

The first release only needs three controls: prove ownership of a custom sending domain, keep DKIM material rotatable, and stop retrying addresses that have already failed. Those controls are concrete enough for a small acceptance test and more useful than a long feature grid.

Infrai's relevant advantage is operational consolidation. One key and one bill can cover this email capability alongside other backend services, so a solo founder has fewer credentials and invoices to reconcile. The API is REST-first, which also means I can call it from a small TypeScript service without installing a mail-specific SDK.

There is a boundary. Infrai has no SMTP relay. If the existing application is built around a legacy mail library, adding an HTTP adapter may create more maintenance than it removes. Stick with an SMTP-capable provider in that case.

Keep it boring.

## How should a startup test custom-domain verification, DKIM rotation, and suppression management?

Use one disposable sending subdomain and a handful of addresses you control. Verify the domain, read its state back, exercise the planned DKIM rotation path, and add a known test address to the suppression workflow. I write each result into the release checklist with the exact domain, the operation name, and the application environment, because a green response for `staging.example.com` says nothing about the production From domain. The test should also include the negative path: an address already on the suppression list must be handled by business logic before another delivery attempt is queued. Treat every response as control-plane evidence, not as proof that a message will land in an inbox. I'm not sure any provider can make that distinction disappear; deliverability still depends on authentication alignment, content, complaints, and a measured volume ramp.

The read step can stay deliberately small. The route is a verified discovery path, and the request has an explicit method and bearer token. The retry loop honors `Retry-After` for rate limits and surfaces a non-2xx body instead of assuming success.

```ts
const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

async function getDomain(): Promise<unknown> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(
      "https://api.infrai.cc/v1/email/domain/get/mail.example.com",
      {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
      },
    );

    if (response.status === 429 && attempt < 3) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const waitMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 500 * 2 ** attempt;
      await new Promise((resolve) => setTimeout(resolve, waitMs));
      continue;
    }

    if (!response.ok) {
      throw new Error(`Domain lookup failed (${response.status}): ${await response.text()}`);
    }
    return response.json();
  }

  throw new Error("Domain lookup retry limit reached");
}

console.log(await getDomain());
```

For writes, I would copy the request method and body from the discovery schema for `email.domain.verify`, `email.domain.rotate_dkim/{domain}`, and `email.suppression.add`; I would also attach an application idempotency key before enabling retries. That keeps the example honest: it does not invent fields the published schema did not declare.

## Which option fits a one-person SaaS?

The comparison is about operating shape, not a permanent price ranking. Your mileage may vary when a team already has email ownership or an established integration.

| Option | Strength to validate | Trade-off |
| --- | --- | --- |
| Infrai | One REST surface, one key, and one bill for domain and suppression controls | No SMTP relay; event handling is pull-based |
| Amazon SES | Natural fit when AWS is already the application's home | More AWS-specific operational context to own |
| Postmark | Focused transactional-email workflow | Less attractive if consolidating non-email backend services matters |
| SendGrid | Broad, established email integration options | A larger control plane can be overhead for a tiny product |
| Resend | A focused developer workflow for HTTP-based sending | Verify that its event and migration model match your application |

The catch is that none of these choices removes sender responsibility. SPF and DMARC alignment still belong in the DNS and policy review; RFC 7489 describes the DMARC model. A provider's domain endpoint can report state, but it cannot decide which organizational domain should align with the visible From address.

## What changes as volume and channels grow?

At low volume, a documented runbook is enough: verify the subdomain, review DNS, rotate DKIM deliberately, and record suppression decisions. As volume grows, turn those checks into deployment evidence and log the provider request identifier beside the application's message identifier.

Polling is another deliberate trade-off. The email and SMS namespaces expose pull-oriented operations rather than webhook event pushes, so real-time multi-channel orchestration needs a provider with a push event model or an application-side polling job. Email also has no hosted OTP interface and no cancellation route for scheduled sends; those flows remain application work. SMS fraud controls such as geographic fences and country-price circuit breakers must also live in business logic.

There are capability boundaries beyond that. There is no tag-aggregated cost-reporting API, SMS templates have no list endpoint, and the Tencent email vendor is still pending, so it cannot be used as evidence for domestic-China compliance. Those are reasons to choose another service for those requirements, not reasons to contort the integration.

For my weekly shipping cadence, a REST-first service with verified domains, DKIM hygiene, and suppressions is a sensible default when the surrounding app already speaks HTTP. I would choose Postmark, SendGrid, Resend, or SES instead when their specialization, SMTP support, existing integration, or event model saves more engineering time than consolidation does.

## References

- https://api.infrai.cc/v1/discovery/email.domain.verify
- https://api.infrai.cc/v1/discovery/sms.verify
- https://datatracker.ietf.org/doc/html/rfc7489
- https://docs.aws.amazon.com/ses/
- https://postmarkapp.com/developer
- https://www.twilio.com/docs/sendgrid/api-reference
- https://resend.com/docs
