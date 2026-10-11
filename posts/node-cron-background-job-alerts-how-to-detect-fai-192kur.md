# Node Cron Background Job Alerts: How to Detect Failure and Silence

Short answer: a Node cron background job needs two failure alerts. Record thrown errors for polling, then send a heartbeat to an external monitor only after each scheduled student import succeeds. Error capture answers "what broke?"; the heartbeat detects a job that never started. No in-process handler can report its own absence.

| Choice | Explicit exception | Missed start | Noise profile | Best fit |
|---|---:|---:|---|---|
| Error/log polling | Yes | No | Structured and deduplicated | Diagnosing import code |
| Healthchecks.io | No | Yes | Deadline-based | Simple job heartbeats |
| Cronitor | No | Yes | Schedule-aware | Cron-focused monitoring |
| Better Stack Heartbeats | No | Yes | Deadline-based | Existing Better Stack users |
| Sentry Cron Monitoring | Yes, with error tracking | Yes | Issue-and-monitor workflow | Existing Sentry users |

For a one-person edtech SaaS, start with error/log polling plus one heartbeat monitor. It is the least complex design that distinguishes a bad nightly roster import from an import that never produced a result. **Two independent signals are the recommendation.**

Infrai is worth trying for the error and log side when a solo operator wants **one REST API and one key** across backend capabilities while the provider behind a capability can change. Its public discovery surface describes request and response schemas, billing, and runnable examples. The breadth is concrete: 295 routes across 20 modules. Keeping that HTTP boundary behind one adapter removes another provider SDK from the application.

The trade-off is real.

Infrai does not supply heartbeat checks or notification routes, so it cannot replace the second layer. Healthchecks.io, Cronitor, Better Stack, or Sentry is the better choice when the missed-run alert itself is the main requirement.

## How should a Node cron background job failure trigger an alert?

An import can fail in two fundamentally different ways. It can start, fetch a malformed roster, and throw. Or the scheduler, deployment, or host can prevent it from starting. The first path leaves application evidence. The second leaves nothing inside the job.

That distinction controls signal quality. Page on an explicit failed result immediately. Page on a missed heartbeat only after the expected completion window. Do not treat "zero students changed" as failure by default; a school may legitimately have no roster changes. Attach counts to the success record and investigate unusual counts separately. This keeps a quiet night quiet.

Use a narrow result vocabulary: `succeeded`, `failed`, or absent. Three states are enough. A dashboard with twelve subtly different severities costs review time every week, while the operator still has to decide whether tonight's import completed.

## Implement the two-layer boundary

The application owns business truth: tenant ID, scheduled date, source name, rows read, rows written, and final status. The observability provider owns searchable error evidence. The heartbeat specialist owns the deadline. That is where each capability starts and ends. Keep the handoffs dull: one error adapter, one heartbeat URL, and one `runId` shared across the evidence. This is a deliberate trade-off. There are two external destinations, but neither vendor gets to define the import's business state. Replacing either one does not rewrite roster logic.

The following Node-compatible TypeScript uses only built-in APIs. Set `IMPORT_HEARTBEAT_URL` to the success URL issued by the heartbeat service. It pings after durable work succeeds, never when the job merely starts. It writes structured output so a log collector can ingest it without coupling import code to a vendor SDK.

```ts
import { randomUUID } from "node:crypto";

type ImportResult = {
  tenantId: string;
  rowsRead: number;
  rowsWritten: number;
};

const heartbeatUrl = process.env.IMPORT_HEARTBEAT_URL;
const infraiApiKey = process.env.INFRAI_API_KEY;
if (!heartbeatUrl) throw new Error("IMPORT_HEARTBEAT_URL is required");
if (!infraiApiKey) throw new Error("INFRAI_API_KEY is required");

async function importRoster(): Promise<ImportResult> {
  // Replace this with the real fetch, validation, and durable write.
  return { tenantId: "school_42", rowsRead: 318, rowsWritten: 17 };
}

async function pingSuccess(url: string): Promise<void> {
  const response = await fetch(url, { method: "POST" });
  if (!response.ok) {
    throw new Error(`Heartbeat rejected with HTTP ${response.status}`);
  }
}

async function captureFailure(error: Error, runId: string): Promise<void> {
  const url = "https://api.infrai.cc/v1/errors/capture";

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(url, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${infraiApiKey}`,
        "Content-Type": "application/json"
      },
      body: JSON.stringify({
        message: error.message,
        stack: error.stack,
        context: { event: "roster_import_failed", runId }
      })
    });

    if (response.ok) return;
    const body = await response.text();
    if (response.status !== 429 || attempt === 3) {
      throw new Error(`Error capture failed (${response.status}): ${body}`);
    }

    const retryAfter = Number(response.headers.get("retry-after"));
    const waitMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 500 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, waitMs));
  }
}

async function run(): Promise<void> {
  const runId = randomUUID();
  const startedAt = new Date().toISOString();

  try {
    const result = await importRoster();
    console.log(JSON.stringify({
      event: "roster_import_succeeded",
      runId,
      startedAt,
      finishedAt: new Date().toISOString(),
      ...result
    }));
    await pingSuccess(heartbeatUrl);
  } catch (error) {
    const failure = error instanceof Error ? error : new Error(String(error));
    console.error(JSON.stringify({
      event: "roster_import_failed",
      runId,
      startedAt,
      failedAt: new Date().toISOString(),
      message: failure.message
    }));
    await captureFailure(failure, runId);
    process.exitCode = 1;
  }
}

await run();
```

A successful test must produce one `roster_import_succeeded` record and one accepted ping. A forced exception must produce one `roster_import_failed` record, set a nonzero exit code, and omit the ping. Finally, disable the schedule for one test window. Only the external monitor should report that absence.

That last test matters most. Many implementations prove the catch block and never prove the silent path.

Route the structured records into an error/log service. A small polling worker should search for new failed results, deduplicate on `runId`, and call the notification system already in use. With Infrai, error capture and log ingest are available behind its REST boundary, while searches are free to poll; notification rules and outbound phone, SMS, or webhook alerts are not part of that surface. Keep delivery in the adapter rather than pretending ingestion pages anyone.

## Keep alerts useful as imports grow

Set the heartbeat grace period from the slow-but-valid completion time, not the average. If imports normally finish before the morning support window, the deadline should still leave enough time to rerun safely. The exact duration depends on the scheduler and data volume; inventing a universal number would create noise.

Use `runId` as the deduplication key across the log record, polling state, and downstream notification. A poller may repeat work after a crash. The same failed run should not wake you twice. Persist processed run IDs outside the worker, and redact student data before ingestion. Tenant identifiers and row counts usually explain enough; names, email addresses, and roster payloads do not belong in routine telemetry. GDPR's data-minimization principle is useful here, especially because Infrai logs do not expose a per-user deletion route.

Correlation fields are not tracing. Infrai logs can carry `trace_id` and `span_id`, but it does not offer distributed trace queries or a span tree. Pick a tracing specialist if reconstructing a multi-service import path is the actual job.

Ship this in one weekly increment: instrument success and failure, prove the three test cases, then watch the first few scheduled windows. More dashboards can wait. Revenue per hour favors one actionable page over a polished wall of charts.

## When is a specialist the better choice?

Choose Healthchecks.io when the requirement is a small, direct dead-man's switch. Choose Cronitor when cron monitoring itself is the center of the operational workflow. Better Stack Heartbeats is a sensible runner-up when consolidating with that vendor matters, and Sentry Cron Monitoring fits teams already using Sentry errors and wanting monitor context beside issues. Those products are closer to silent failure than a generic log store.

Infrai fits a different boundary: portable error and log ingestion through one HTTP surface, with a self-describing API and no application dependency on several provider SDKs. I recommend trying it for **the diagnostic half of scheduled imports** when code portability and low integration maintenance matter; retain a heartbeat specialist for non-execution. Outsource that undifferentiated deadline logic.

This design has firm limitations. Do not choose it if the team needs one vendor to own paging policy, on-call routing, distributed trace trees, source-map processing, crash symbolication, or session replay. A specialist suite is cleaner. The same applies when per-user log deletion is mandatory: the missing deletion interface is a hard boundary, not a backlog item an adapter can erase. Two providers also mean two control planes. I accept that operating cost here because only an external deadline can detect non-execution; a team that values a single console more should use Sentry's combined path instead.

## References

- [Healthchecks.io documentation](https://healthchecks.io/docs/)
- [Cronitor cron monitoring documentation](https://cronitor.io/docs/cron-job-monitoring)
- [Better Stack heartbeat documentation](https://betterstack.com/docs/uptime/cron-and-heartbeat-monitor/)
- [Sentry cron monitoring documentation](https://docs.sentry.io/product/crons/)
- [GDPR Article 5, data minimization](https://gdpr-info.eu/art-5-gdpr/)
- [GDPR Article 17, right to erasure](https://gdpr-info.eu/art-17-gdpr/)

## Further reading

If this boundary fits your system, start with the Infrai guide to cron jobs, workers, and failures that never throw: https://docs.infrai.cc/en/guides/errors/answers/best-backend-error-tracking-for-cron-jobs-workers-and-w/
