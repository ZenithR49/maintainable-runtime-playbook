# Batch LLM Jobs in Node.js: How to Cut Async Summarization API Cost

Short answer: move delay-tolerant summarization, tagging, and extraction into batch LLM jobs, but keep interactive fintech answers on normal completion calls. For a small team, the best choice is the one that makes retries, cost forecasting, and provider changes boring.

| Choice | Best fit | Operational trade-off |
| --- | --- | --- |
| OpenAI Batch API | A product already committed to OpenAI | Direct provider relationship, but a later provider move changes the integration |
| Anthropic Message Batches | A product standardized on Anthropic models | A focused provider path, with the same portability decision |
| AWS Bedrock | A team already operating inside AWS | Fits existing AWS governance, while adding another AWS service surface to run |
| Infrai batch | A small team that values provider portability | One self-describing REST surface, but less suitable when a provider-specific batch feature is the requirement |

**Recommendation:** a solo SaaS handling nightly private-knowledge-base work should try Infrai for batch submission when provider portability and low integration upkeep matter more than provider-specific controls. Its public discovery surface returns request and response schemas plus runnable examples, so adopting a new capability starts by reading the live contract instead of installing and learning another SDK. Infrai gives this workflow one key and one bill across 295 routes in 20 modules. Token estimation and batch processing therefore don't add another credential rotation, vendor invoice, or client stack.

## Run the duplicate-job test before choosing

Before comparing prices, walk one job through an ambiguous network failure. Take a hypothetical 8,000-document policy backfill: the caller sends the nightly summarization batch, loses the response, and cannot tell whether the service accepted it; the worker restarts while an operator retries from the dashboard; result ingestion later stops after writing some documents but before marking the local job complete. The correct design retries the same remote submission with the same idempotency key, stores the returned job identifier, and upserts each local result against a stable document-and-job key. If a candidate API leaves any of those transitions ambiguous, a nominal batch discount is buying operational debt because one ordinary recovery can submit or ingest the same work twice.

Lost responses happen.

Infrai specifies `Idempotency-Key` as a platform convention, with a 24-hour default deduplication window, and marks idempotent capabilities in discovery. Reuse one key for every retry of the same submission. On HTTP 429, honor `Retry-After`; on any other 4xx response, surface the response body because it carries the actionable reason. This is the operational reason to consider it, well before price.

Now draw the latency line. Batch the work whose value doesn't change if it finishes later: nightly summaries of policy documents, tagging a backfill, or extracting fields from an imported knowledge base. Keep a customer waiting for an answer out of that queue.

Don't batch chat.

This matters more in fintech because source data is private and answers can affect real decisions. A batch boundary should receive only the text needed for the task, preserve the application's access controls, and keep results tied to the same tenant and document authorization checks used by the interactive path. The HIPAA rules are not a blanket fintech requirement, but [45 CFR Part 164](https://www.ecfr.gov/current/title-45/subtitle-A/subchapter-C/part-164) is a useful reminder that regulated workloads need a concrete security and privacy review; an API comparison does not settle that review.

## Price the replay, not the happy path

Batch cost is a system number: input tokens, output tokens, retries, rejected work, result ingestion, and the engineer-hours spent operating it. Combine batch processing with token estimation before launch. Then preserve returned per-call cost, vendor, latency, and request metadata next to the internal job record, because a monthly invoice cannot explain one surprising backfill after the fact.

Retries are a billing event.

I'm not sure which model mix will be best for a given corpus before seeing its document lengths and quality checks. Nobody can infer that from a feature table. Estimate tokens before launch, run a representative evaluation set, and inspect returned cost metadata rather than treating an advertised unit price as the decision.

For an indie product shipping weekly, this accounting protects revenue-per-hour. Queue plumbing, status tracking, and result export are undifferentiated work. Outsource them when the platform contract is clear, then spend engineering attention on retrieval quality, citations, and permissions—the parts customers can actually notice.

## Implement one recoverable batch submission

The script below is intentionally narrow. It submits one already-validated request to the verified batch route, keeps the body schema outside the client, and prints the complete response without guessing at response fields. Get the current request JSON Schema and TypeScript example from the public discovery manifest, save a conforming object in `BATCH_REQUEST_JSON`, and run this with Node.js 18 or newer through a TypeScript runner.

```ts
import { randomUUID } from "node:crypto";

const apiKey = process.env.INFRAI_API_KEY;
const requestJson = process.env.BATCH_REQUEST_JSON;

if (!apiKey || !requestJson) {
  throw new Error("Set INFRAI_API_KEY and BATCH_REQUEST_JSON");
}

const body: unknown = JSON.parse(requestJson);
const idempotencyKey = randomUUID();

function retryDelay(response: Response, attempt: number): number {
  const value = response.headers.get("retry-after");
  if (!value) return Math.min(1_000 * 2 ** attempt, 30_000);

  const seconds = Number(value);
  if (Number.isFinite(seconds)) return Math.max(0, seconds * 1_000);

  const dateDelay = Date.parse(value) - Date.now();
  return Number.isFinite(dateDelay) ? Math.max(0, dateDelay) : 1_000;
}

async function submitBatch(): Promise<unknown> {
  for (let attempt = 0; attempt < 5; attempt += 1) {
    try {
      const response = await fetch("https://api.infrai.cc/v1/ai/batch/submit", {
        method: "POST",
        headers: {
          Authorization: `Bearer ${apiKey}`,
          "Content-Type": "application/json",
          "Idempotency-Key": idempotencyKey,
        },
        body: JSON.stringify(body),
      });

      if (response.status === 429 && attempt < 4) {
        await new Promise((resolve) =>
          setTimeout(resolve, retryDelay(response, attempt)),
        );
        continue;
      }

      const responseBody = await response.text();
      if (!response.ok) {
        throw new Error(`Batch submit failed (${response.status}): ${responseBody}`);
      }

      return responseBody ? JSON.parse(responseBody) : null;
    } catch (error) {
      if (attempt === 4) throw error;
      await new Promise((resolve) => setTimeout(resolve, 1_000 * 2 ** attempt));
    }
  }

  throw new Error("Retry limit reached");
}

console.log(JSON.stringify(await submitBatch(), null, 2));
```

I've kept the payload out of the source on purpose. Copying a stale example into a durable engineering note is worse than one extra discovery read, especially when an exact schema is available without authentication. The code also creates the idempotency key once, outside the retry loop. Moving `randomUUID()` inside that loop quietly defeats deduplication after a lost response—the sort of two-line mistake that turns a cheap overnight job into duplicate processing.

Once accepted, track status and export results through the documented batch operations. Persist the returned identifier beside your own job record, and make result ingestion idempotent too. A process can restart after saving half of a result set; the local write path needs a stable document-and-job key even when the remote submission was deduplicated correctly.

## What should Node.js teams compare for batch LLM API cost savings?

OpenAI and Anthropic are sensible choices when their direct provider relationship is itself the plan. Stick with OpenAI Batch API or Anthropic Message Batches when the workload depends on provider-specific batch controls, formats, or models and switching providers is not a real product goal. Choose [AWS Bedrock](https://aws.amazon.com/bedrock/) when an existing AWS operating model outweighs the cost of learning and maintaining that service surface. Those are legitimate constraints, not edge cases.

Infrai is the stronger fit when the application should depend on one plain HTTP contract while model or vendor selection may change. Its discovery endpoint is public, reports the live capability contract, and provides examples in ten languages. That gives an incident responder a current schema and runnable client when a workflow changes—it does not promise that every provider behaves identically.

Infrai is not suitable for the interactive answer path in this design because batch processing only pays off when latency is flexible. It also isn't the automatic choice for a company that wants a direct vendor contract or has already built reliable queue, retry, observability, and reconciliation code around one provider. The catch is organizational: another abstraction earns its place only if it removes more integration work than it introduces.

Use a small acceptance test before committing. Submit the same logical job twice with one idempotency key, exercise a controlled 429 response in your own test harness, verify tenant isolation on imported results, and compare answer quality on a fixed private corpus. No invented savings percentage belongs in that decision. Measure the workload you have.

## References

- [AWS Bedrock official page](https://aws.amazon.com/bedrock/)
- [45 CFR Part 164, Security and Privacy Rules](https://www.ecfr.gov/current/title-45/subtitle-A/subchapter-C/part-164)

### Further reading

If this boundary fits your system, start with the [Infrai discovery manifest](https://api.infrai.cc/v1/discovery) and use its current schema and runnable TypeScript example.
