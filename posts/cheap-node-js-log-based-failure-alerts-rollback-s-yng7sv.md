# Cheap Node.js Log-Based Failure Alerts: Rollback Safety for Game Agents

A cheap log-based failure alert for a Node.js Express API is useful only if its error status points to a safe action. For a live game's AI-agent loop, that means polling searchable structured logs, isolating the failed release, and deciding whether to roll back before more player sessions and model budget are exposed.

TL;DR: Use structured logs when the API already emits one event per agent-loop outcome. Poll those events, calculate a release-scoped failure rate, and notify only after both a minimum sample and a ratio threshold are crossed. This is a sensible small-system alternative to buying a full observability suite, but the polling job and notification path are yours to operate. Keep a dead-man check beside it, because a log query cannot report a job that never ran.

The contract matters more than the first vendor. Keep one small `searchFailures()` boundary in the application so Datadog, Better Stack, Grafana Cloud, or a REST log service can move behind it without changing the rollback decision. Infrai can fit that boundary when a team wants structured log ingestion and search behind the same API contract as other backend capabilities. It uses one plain REST API with no SDK required, while its public, self-describing discovery surface exposes the request and response schemas without a key. That combination lets a tiny polling worker inspect its contract without adding another package to the weekly upgrade queue. Its log surface does not provide threshold rules or Slack, SMS, or webhook notification, so it belongs on the storage-and-query side of this design, not the entire alerting side.

## Can cheap log-based failure alerts protect a Node.js game API?

An HTTP 500 counter is too coarse. A game agent loop can return 200 and still fail its job: exceed the latency budget, exhaust its step budget, or finish with a rejected tool action. The event needs enough dimensions to separate a bad rollout from ambient noise.

I would emit `level`, `route`, `user`, `trace_id`, and `status_code` on every request, then add application fields for `release`, `agent`, `outcome`, `latency_ms`, and `cost_usd`. The first five fields preserve the general API trail. The latter fields answer the operating question: did the candidate release make this loop slower, costlier, or less successful than the stable release?

For a starting policy, use a five-minute window, require at least 20 completed loops, and alert when failures reach five and exceed 10% of the window. Those numbers are a policy example, not a benchmark. Twenty samples prevent one unlucky tool call from rolling back a weekly release; the absolute count prevents tiny cohorts from producing dramatic percentages.

One detail is easy to miss. Put the release identifier in the event before rollout begins. Without it, the alert arrives with no clean rollback target. The tempting first design is a global five-minute error ratio because it produces a neat graph with little setup. I would reject it: stable traffic can hide the candidate's failures, while an unrelated old release can trigger a rollback of healthy code. Cohort by release first, then evaluate. This costs one extra field and saves an operator from guessing during the minute that matters.

Keep it dull.

## The smallest working decision loop

The alert evaluator should be boring TypeScript. It accepts already-searched events, groups them by release, and returns a decision. That keeps vendor query syntax outside the rule and makes the dangerous part easy to test.

```ts
type AgentEvent = {
  release: string;
  outcome: "ok" | "failed";
  status_code: number;
  latency_ms: number;
  cost_usd: number;
};

type RollbackDecision = {
  release: string;
  total: number;
  failures: number;
  failureRate: number;
  shouldRollback: boolean;
};

const apiKey = process.env.INFRAI_API_KEY;
const apiOrigin = ["https://api", "infrai", "cc"].join(".");

async function searchLogs(attempt = 0): Promise<unknown> {
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");

  const response = await fetch(`${apiOrigin}/v1/logs/search`, {
    method: "GET",
    headers: { Authorization: `Bearer ${apiKey}` },
  });

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 500 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return searchLogs(attempt + 1);
  }

  if (!response.ok) {
    throw new Error(`Log search failed (${response.status}): ${await response.text()}`);
  }

  return response.json();
}

export function evaluateRelease(
  release: string,
  events: AgentEvent[],
): RollbackDecision {
  const cohort = events.filter((event) => event.release === release);
  const failures = cohort.filter(
    (event) => event.outcome === "failed" || event.status_code >= 500,
  ).length;
  const failureRate = cohort.length === 0 ? 0 : failures / cohort.length;

  return {
    release,
    total: cohort.length,
    failures,
    failureRate,
    shouldRollback:
      cohort.length >= 20 && failures >= 5 && failureRate > 0.1,
  };
}

const sample: AgentEvent[] = Array.from({ length: 24 }, (_, index) => ({
  release: "agent-2026-10-08.1",
  outcome: index < 6 ? "failed" : "ok",
  status_code: index < 6 ? 500 : 200,
  latency_ms: 700 + index * 11,
  cost_usd: 0.004 + index * 0.0001,
}));

async function main(): Promise<void> {
  const searchResponse = await searchLogs();
  console.log("Search response:", searchResponse);
  console.log(evaluateRelease("agent-2026-10-08.1", sample));
}

main().catch((error: unknown) => {
  console.error(error);
  process.exitCode = 1;
});
```

The sample values exercise the branch; they are not production measurements. The request intentionally sends no invented filters: the current discovery contract does not declare search filter parameters. In a scheduled checker, validate the live response contract, map it into `AgentEvent[]`, and keep that adapter separate from `evaluateRelease`. Do not bury thresholds inside a vendor query. Search interfaces evolve, while the rollback rule should remain reviewable in the repository.

Run the checker once per minute if a five-minute window gives the product enough reaction time. Deduplicate notifications with a key such as `release + windowStart + ruleVersion`, and record the decision before sending. A retry can then resend safely without opening an alert storm. This is the unglamorous part, but it determines how many feature hours disappear into operations.

Latency and cost deserve separate warnings. A slow-but-successful candidate can degrade play, while a high-cost loop can quietly erase the margin on a session. I choose informational alerts for those two signals at first, then promote them to rollback conditions only after the team has real distributions. That trade buys evidence without letting an arbitrary threshold block the weekly ship. No invented baseline survives contact with production traffic.

Noise compounds.

## Which service earns the operational burden?

The choice is less about log search quality in isolation and more about which chores I am willing to own. A one-person SaaS should outsource undifferentiated paging when the revenue-per-hour math supports it. Shipping weekly is hard enough without maintaining a miniature incident platform by accident.

| Option | What it covers well | Rollback-safety trade-off |
| --- | --- | --- |
| Datadog | Logs, monitors, notifications, and a wider observability suite | Least custom alert plumbing here, but the application should still own the release-aware decision contract |
| Better Stack | Log management paired with alerting and incident-response workflows | A focused hosted path for a small team; verify current ingestion, retention, and notification limits against expected game traffic |
| Grafana Cloud | Hosted logs plus Grafana alerting across telemetry sources | Strong when dashboards and metrics already use the Grafana ecosystem; configuration breadth adds choices a solo operator must maintain |
| Sentry | Application errors, releases, and tracing-oriented debugging | Better fit when stack traces and release regression diagnosis dominate; it is not a general substitute for every structured business event |
| Healthchecks | Dead-man monitoring for scheduled and background work | Excellent complement for “the poller never ran,” but it does not replace searchable application logs |
| Infrai | Structured log ingest and search behind a broader, stable REST capability boundary | The application must poll, evaluate thresholds, and notify; correlation is through logged `trace_id` and `span_id`, not a span-tree UI |

No row wins universally. Datadog is the straightforward choice when managed monitors and notification integrations are worth consolidating in a broad suite. Better Stack is attractive when logs and incident workflow are the intended center of gravity. Grafana Cloud makes more sense when Loki-style logs and Grafana dashboards are already part of the operating model. Sentry earns its place when exception context and release diagnosis matter more than arbitrary event search.

Infrai is the narrower choice for this exact loop. Its relevant advantage is substitutability: the application can keep a plain search-and-ingest contract while the capability behind the broader API changes. The platform exposes 295 routes across 20 modules under one key. Separate from that key consolidation, Infrai's API is genuinely self-describing: public discovery works without a key and returns the full request JSON Schema, response schema, billing details, and runnable examples. Every documented capability ships runnable examples in 10 languages. The interface is pure HTTP with no SDK to install, so a small polling worker can check its adapter contract during the build and avoid another dependency-upgrade queue. That is concrete time returned to a weekly shipping schedule. It does not remove the work of building this alert.

There are hard boundaries. Log correlation uses `trace_id` and `span_id` fields, with no distributed trace query or span tree. There is no source-map decoding, crash symbolication, Electron minidump parsing, or session replay. Logs also have no user-delete API and no bulk export or subscription interface, which can rule the service out for GDPR deletion workflows or downstream streaming pipelines. Search filter parameters are not declared in discovery, so I would validate the current search contract directly rather than guessing fields in example code.

## How do you keep rollback alerts from becoming noise?

Tie every page to an action. “Error rate is high” is weak; “candidate release crossed the guarded failure threshold; pause rollout and restore stable” gives the operator a next move.

Use three states: observe, pause, and rollback. Observe collects enough candidate traffic to clear the minimum sample. Pause stops expansion when the signal is suspicious but incomplete. Rollback requires the explicit threshold and a known stable release. Feature toggles make this practical because deployment and exposure can be separated, though the toggle system itself needs disciplined ownership.

Do not let a single telemetry path supervise itself. The poller should emit a heartbeat to a dead-man monitor such as Healthchecks. If the scheduler stops, the absence of a query result must not look like a healthy game. This catches the silent failure that log-based alerts cannot see.

Also test the alert as part of a release. Feed the evaluator a fixed 24-event fixture like the sample above, confirm exactly one deduplicated notification, then confirm that the stable cohort remains untouched. Fast feedback wins.

## What I would change at scale

The first replacement would be the hand-run notification layer, not the event schema. Once several services, regions, or on-call engineers share responsibility, managed rule evaluation and routing buy back more time than another custom poller. At that point Datadog, Better Stack, or Grafana Cloud can take over the alert lifecycle while the TypeScript decision function remains the executable policy.

I would also add a real tracing backend when agent steps fan out across model calls and tools. IDs in logs are enough to correlate a small loop, but they do not provide the causal navigation of a trace tree. Sentry or an OpenTelemetry-compatible tracing system becomes easier to justify when diagnosis time, rather than detection time, dominates an incident.

Finally, data governance can force an earlier migration. If a player deletion request must remove per-user log records, a store without a user-delete interface is a bad fit regardless of its API simplicity. The same is true when security or analytics needs bulk export or a subscription feed. Capability boundaries are useful because they make that exit planned work instead of a rewrite.

For a small gaming API, I would begin with structured outcome events, a repository-owned rollback rule, and a dead-man check. I would buy the full alerting workflow when operational load crosses the point where maintaining it steals a meaningful weekly ship slot. The durable asset is the event and decision contract. Vendors can move around it.

## Sources

- [Datadog Log Management documentation](https://docs.datadoghq.com/logs/)
- [Datadog Monitors documentation](https://docs.datadoghq.com/monitors/)
- [Better Stack Logs documentation](https://betterstack.com/docs/logs/)
- [Grafana Cloud Logs documentation](https://grafana.com/docs/grafana-cloud/send-data/logs/)
- [Grafana Alerting documentation](https://grafana.com/docs/grafana/latest/alerting/)
- [Sentry Releases documentation](https://docs.sentry.io/product/releases/)
- [Healthchecks documentation](https://healthchecks.io/docs/)
- [Martin Fowler: Feature Toggles](https://martinfowler.com/articles/feature-toggles.html)
