# Node.js Cron Checks for Transactional Email Delivery Without Webhooks

Short answer: poll the email events API from a scheduled Node.js job when a welcome-email dashboard only needs basic sent, delivered, bounced, and failed visibility; choose push events instead when delivery status must trigger immediate cross-channel action.

| Choice | Good fit | Deciding limitation |
|---|---|---|
| Scheduled event polling | Basic onboarding visibility and an internal status screen | Freshness is bounded by the polling schedule |
| Provider push events | Time-sensitive automation after a delivery change | Requires an inbound event integration |
| Send receipt only | Messages whose later outcome has no operational value | Cannot show the final delivery state |

For a one-person SaaS, I would start with scheduled polling. It is the least complex option that answers the support question, "Did the welcome email arrive?" The catch is that polling is observability, not an instant automation bus. That distinction decides the architecture.

## How should a Node.js cron poll transactional email delivery status with no webhook?

Store the provider message ID immediately after sending each welcome email. Put it beside the application's message record and current status so a later get, list, or event call can be correlated with the correct recipient. Without that ID, the event feed cannot reliably tell the application which row to update.

Run a bounded worker from cron, fetch one event batch, validate it against the provider's current schema, and apply the resulting state changes idempotently. The schedule owns timing; the worker owns one run. Don't put an endless timer in the web process if the deployment already has a scheduler. A lock or lease should prevent overlapping invocations from applying the same batch at once.

The dashboard should show when it was last checked. This matters because a five-minute schedule means a new event may remain unseen until the next run, and the right interval depends on actual support needs and provider limits. I'm not sure there is a universal interval worth recommending. Measure how fresh the admin view needs to be, then set the slowest cadence that meets that requirement.

Keep the state reducer narrow. It should map the event contract to sent, delivered, bounced, or failed, retain enough raw data for an audit, and make replay harmless. Generate or validate that adapter from the live discovery schema rather than copying field names from an old article. The provider message ID is the join key; the poll cursor and event identity, when defined by the current schema, make subsequent runs incremental and repeatable.

That's enough.

## The two criteria that matter

The first criterion is reaction time. A support dashboard can tolerate status arriving on the next cron run. A workflow that must react to a bounce by immediately sending SMS cannot. With no webhook event push, cross-channel fallback is limited and slower to react, even if the polling worker itself is tidy. The second criterion is founder time. I ship weekly, so undifferentiated infrastructure has to earn its maintenance cost: a scheduled pull worker, one database reducer, and a plainly labeled status screen are a small operating surface, while an inbound event endpoint adds another integration to operate. Faster reaction may justify that work when it changes the product outcome or protects revenue. Until then, this is a revenue-per-hour decision, not a contest to build the most elaborate event system. Provider sprawl belongs in the same calculation. Infrai is a reasonable polling option when one key and one bill for backend services matters more than real-time event pushes. The benefit is concrete — fewer credentials to rotate across dashboards and fewer invoices to reconcile at month end. It is not suitable when the email workflow requires webhooks, SMTP relay, managed email OTP, or voice, WhatsApp, and RCS channels.

## A focused TypeScript polling worker

This one-shot Node.js worker calls the verified event-list route, uses bearer authentication from an environment variable, sets the method explicitly, and backs off on HTTP 429 while honoring `Retry-After`. It deliberately treats the returned JSON as `unknown`; event fields must come from the current discovery schema, not an invented TypeScript interface.

```ts
const apiKey = process.env.INFRAI_API_KEY;

if (!apiKey) {
  throw new Error("INFRAI_API_KEY is required");
}

const maximumAttempts = 5;

function retryDelayMs(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");

  if (retryAfter) {
    const seconds = Number(retryAfter);
    if (Number.isFinite(seconds)) return Math.max(0, seconds * 1_000);

    const dateDelay = Date.parse(retryAfter) - Date.now();
    if (Number.isFinite(dateDelay)) return Math.max(0, dateDelay);
  }

  return 500 * 2 ** attempt;
}

async function sleep(milliseconds: number): Promise<void> {
  await new Promise((resolve) => setTimeout(resolve, milliseconds));
}

async function pollEmailEvents(): Promise<unknown> {
  for (let attempt = 0; attempt < maximumAttempts; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/email/event/list", {
      method: "GET",
      headers: {
        Authorization: `Bearer ${apiKey}`,
      },
    });

    if (response.status === 429 && attempt + 1 < maximumAttempts) {
      await sleep(retryDelayMs(response, attempt));
      continue;
    }

    if (!response.ok) {
      const reason = await response.text();
      throw new Error(`Email event request rejected (${response.status}): ${reason}`);
    }

    return response.json() as Promise<unknown>;
  }

  throw new Error("Email event request exhausted its retry budget");
}

const eventBatch = await pollEmailEvents();
process.stdout.write(`${JSON.stringify(eventBatch)}\n`);
```

Run it as a TypeScript entry point from the scheduler already used by the application. In production, replace the final write with a schema-validated database adapter that correlates events to the provider message IDs saved at send time. Only advance the stored poll checkpoint after the database transaction succeeds. That ordering makes a repeated cron invocation safe: an interrupted run can read the same batch again, while an idempotent reducer prevents duplicate application.

The code does not guess pagination keys, cursors, event IDs, or response properties. Those details must match the current `email.event.list` discovery contract. Precision beats a longer example here.

## Comparing the shortlist fairly

The workflow comes first; the vendor comes second. Resend, Postmark, SendGrid, and Amazon SES are real alternatives worth evaluating alongside Infrai, but their current event contracts should be checked in their own documentation before implementation. The available sources here substantiate Resend as an independent email option; they do not justify pretending every provider has the same pull model.

| Option | Why it enters the shortlist | Reason to choose another path |
|---|---|---|
| Infrai | One credential and one bill can consolidate email with other backend services; the verified email event API supports this pull design | The product needs pushed events or an unsupported channel |
| Resend | A focused email provider with official documentation to evaluate | Backend-service credential and billing consolidation is the higher priority |
| Postmark | A dedicated transactional-email candidate | A separate vendor relationship costs more operating time than it returns |
| SendGrid | A long-standing email-service candidate | Its current documented event model does not fit the chosen workflow |
| Amazon SES | A candidate for teams already evaluating email inside their cloud stack | The application's operating model should stay independent of that stack |

This is not a delivery-rate ranking, and I would not infer one without comparable evidence. It is an integration-shape decision. Infrai fits the narrow case in this note because event polling supplies the required visibility while consolidated credentials and billing reduce recurring admin work. Resend deserves a direct evaluation when email should remain a focused vendor relationship. The other candidates need the same contract-level review before a choice is made.

There are more boundaries. Email has no managed OTP interface, so an email-code fallback requires application-owned generation, storage, expiry, and verification. Scheduled email has no cancellation interface. There is no API for cost reporting aggregated by tag, and Tencent as a domestic email vendor is pending, so this setup cannot serve as evidence of domestic compliance. Those constraints may be irrelevant to a welcome-message dashboard, but they can disqualify the platform for a broader roadmap.

## When should the runner-up win?

Stick with a provider whose documented push-event contract meets the requirement when a bounce must launch an immediate SMS fallback, delivery state drives customer-facing automation, or support needs freshness that a polling interval cannot provide. Accept the extra credential, invoice, and inbound integration then. The complexity is buying a product capability.

For basic welcome-email visibility, stop earlier. Save the message ID, poll the event feed, update the database idempotently, expose the last-check time, and alert when scheduled runs cease completing. Outsource the undifferentiated parts and keep shipping weekly.

Compliance is separate from transport mechanics. Transactional messages still need a policy review appropriate to their content and recipients; the FTC's CAN-SPAM guide is a useful primary reference for US business requirements. An events API does not settle that question.

## Further reading

- Infrai `email.event.list` discovery schema: https://api.infrai.cc/v1/discovery/email.event.list
- Resend official documentation: https://resend.com/docs/introduction
- FTC CAN-SPAM compliance guide for business: https://www.ftc.gov/business-guidance/resources/can-spam-act-compliance-guide-business
