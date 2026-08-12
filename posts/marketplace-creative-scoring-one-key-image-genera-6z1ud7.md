# Marketplace Creative Scoring — One-Key Image Generation API Across Competing Models

Short answer: put a small, app-owned scoring contract in front of a unified image generation API, send the same marketplace brief to two models, and promote the faster model only when its rubric score clears a fixed quality floor.

| Choice | Weekly maintenance | Quality control | Latency control | Best fit |
| --- | --- | --- | --- | --- |
| One unified gateway, one model | Low | Low | Medium | Early validation |
| One unified gateway, two-model evaluation | Medium | High | High | Marketplace production |
| Separate provider integrations | High | Highest | Highest | Native features are central |

The middle option is my recommendation for a solo SaaS. It outsources credential handling and model access, while the product still owns the part that affects revenue: deciding whether a generated marketplace creative is good enough to show. Keep the initial pool to two models. More choice creates more evaluation work, and evaluation work competes directly with shipping the next feature.

## Why should one key route text-to-image generation across multiple AI models?

A single credential is useful because it reduces operational surface area. It isn't the decision system. A marketplace still needs to determine which image candidate follows the job brief, preserves required text, avoids prohibited content, and arrives before the request becomes stale. Those judgments belong in application code because they are product policy, not transport details.

The clean boundary has three parts: a normalized generation request, an immutable record of what was requested, and a scoring result that can explain why a candidate passed. The gateway can translate the request for different models. The marketplace owns the rubric and the release rule.

This distinction matters for a one-person operation. Swapping credentials or parsing another response shape is undifferentiated work. Deciding that a seller's hero image must show the product clearly, leave space for a price badge, and avoid unreadable embedded copy is product work. I would outsource the first and keep the second close.

Don't confuse generation with retrieval. Text embeddings represent text as vectors and can support semantic matching; a vector extension for Postgres can store and compare those vectors. That can help retrieve an appropriate rubric or find similar briefs, but it doesn't prove that the pixels satisfy a visual requirement. The final image still needs a visual evaluator or human review for the criteria that matter.

One key also creates a concentration trade-off. If every model is reached through one gateway, its quota policy and normalization choices sit on the critical path. The boundary should therefore remain narrow enough that direct integrations are possible later. No grand abstraction. A request type, a result type, and evidence for the score are enough.

## Quality is a release rule, not a model leaderboard

Start with a job-shaped rubric. For a marketplace listing, the prompt might ask for a square hero image of a red desk lamp on a white background with empty space in the upper-right corner. The rubric should score observable requirements rather than taste: product identity, requested color, background, composition, and policy compliance.

Use a weighted score only after defining hard failures. A candidate that depicts the wrong product should fail even if its lighting and composition are excellent. Likewise, a candidate that violates a marketplace policy shouldn't be rescued by a high average. Weighted totals are useful for ordering candidates that already satisfy the non-negotiable constraints.

For example, set an initial quality floor from a reviewed evaluation set, then revisit it when the job mix changes. I'm not sure any universal threshold would survive across fashion, furniture, and digital goods; the evidence needed to settle that is a labeled set from the marketplace's own traffic. A threshold is a release policy, not an industry fact.

Keep the comparison paired. Each model receives the same normalized brief, dimensions, and seed when the underlying capability supports it. Store the model identifier, normalized inputs, rubric version, evaluator version, duration, and decision. Without that record, a model change and a rubric change can land in the same week and leave no clean explanation for a quality shift.

The evaluator deserves skepticism too. If an automated evaluator consistently favors a certain visual style, the routing policy will optimize toward that preference. Sample disagreements for human review, especially near the quality floor. The point isn't to build a perfect judge before launch. It is to make the uncertainty visible and cheap to inspect.

Ship weekly. A small reviewed set added every week is more valuable than a large evaluation project that never enters the release path.

## Put latency into the scoring contract

Quality versus latency should be an explicit policy. Don't ask for the best image in the abstract; ask for the best eligible image before the marketplace's deadline.

Split the deadline into queue time, generation time, evaluation time, and storage time. Measure each segment separately. An end-to-end number can tell you a request was slow, but it can't tell you whether parallel generation, a cheaper evaluator, or a shorter queue would help. Use percentiles over comparable jobs rather than one average across every image size and job type.

The practical routing rule is simple: call the primary model first for interactive work, accept it when it clears the quality floor, and use the second model for a bounded retry or for jobs that can complete asynchronously. Consider a seller waiting for a listing preview versus a nightly catalog refresh. The preview has a person on the other side of the request, so the first eligible candidate should win; generating two candidates in parallel can consume capacity without changing the release decision. The catalog refresh has no person waiting, so both candidates can run together and the higher-scoring eligible result can win. Those jobs need separate latency budgets even when their prompts look identical. Handle overload as a product state as well: a rate-limited request should enter a bounded retry policy with jitter, while a request that has exceeded its deadline should stop consuming generation capacity. Preserve an idempotency key at the application boundary so a client retry refers to the same job. The exact remote behavior varies, so verify it against the service contract rather than assuming retries are free or automatically deduplicated. Your mileage may vary — log the pass rate, deadline misses, and winning model by job class before making parallel generation the default. For a solo founder, the useful metric is successful marketplace jobs per hour of engineering attention. A model that wins a lab comparison but requires constant exception handling can lose on that metric; choosing only the lowest-latency model can create manual moderation work that wipes out the saved seconds.

Fast is contextual.

Count the review queue.

## A thin TypeScript contract keeps the rubric portable

The application doesn't need a universal schema for every image feature. It needs a stable schema for this job. Preserve an escape hatch for optional model controls, but don't let those controls leak into the scoring rule.

```ts
type ModelId = string;

type CreativeJob = {
  jobId: string;
  prompt: string;
  width: number;
  height: number;
  deadlineMs: number;
  rubricVersion: string;
};

type Candidate = {
  model: ModelId;
  assetUrl: string;
  durationMs: number;
};

type Criterion = {
  name: "product" | "color" | "background" | "composition" | "policy";
  score: number;
  required: boolean;
  evidence: string;
};

type Evaluation = {
  candidate: Candidate;
  criteria: Criterion[];
  weightedScore: number;
};

interface ImageRuntime {
  generate(job: CreativeJob, model: ModelId): Promise<Candidate>;
}

interface CreativeEvaluator {
  evaluate(job: CreativeJob, candidate: Candidate): Promise<Evaluation>;
}

const isEligible = (result: Evaluation, qualityFloor: number): boolean => {
  const requiredCriteriaPass = result.criteria
    .filter((criterion) => criterion.required)
    .every((criterion) => criterion.score === 1);

  return requiredCriteriaPass && result.weightedScore >= qualityFloor;
};

async function chooseCreative(
  job: CreativeJob,
  models: readonly [ModelId, ModelId],
  runtime: ImageRuntime,
  evaluator: CreativeEvaluator,
  qualityFloor: number,
): Promise<Evaluation> {
  const primary = await runtime.generate(job, models[0]);
  const primaryResult = await evaluator.evaluate(job, primary);

  if (isEligible(primaryResult, qualityFloor)) return primaryResult;

  const runnerUp = await runtime.generate(job, models[1]);
  const runnerUpResult = await evaluator.evaluate(job, runnerUp);

  if (isEligible(runnerUpResult, qualityFloor)) return runnerUpResult;
  throw new Error("NO_ELIGIBLE_CREATIVE");
}
```

This example is intentionally boring. It doesn't claim that every runtime accepts the same dimensions, supports a seed, or returns a durable URL. The adapter must validate the capability before dispatch and copy accepted output into storage controlled by the application. The decision function sees only the fields it actually needs.

`NO_ELIGIBLE_CREATIVE` is a business outcome, not proof of an infrastructure failure. The marketplace can send that job to review, request a revised brief, or leave the previous approved asset in place. Keep that path separate from authentication, quota, and transport errors so dashboards don't mix creative rejection with runtime health.

Before changing a model or evaluator, replay a fixed, versioned job set and compare eligibility decisions. Inspect changed decisions, not just the average score. A small average movement can hide a serious regression on one required criterion. Release the routing change behind a flag, watch deadline misses and review volume, then widen it. This is enough ceremony to catch expensive mistakes without building an internal platform.

## When is the runner-up architecture better?

Separate provider integrations are the better choice when native image controls are part of the product promise. If sellers depend on a provider-specific editing workflow, mask format, provenance field, or deterministic control, flattening that feature into a common gateway contract may erase the reason customers use the product. Stick with direct integrations when those native capabilities drive conversion or when contractual control over data handling requires a direct relationship.

The catch is maintenance. Every direct integration adds authentication, capability checks, error mapping, observability, and release testing. That can still be correct, but it should earn its place through a measurable product requirement. A generic desire to keep every option open isn't enough.

At the other end, one model behind one gateway is suitable during initial validation. There is no reason to operate a model tournament before the marketplace has a stable rubric and reviewed examples. Add the second model when misses have recognizable categories and the alternative can be tested against them. Otherwise, complexity arrives before evidence.

A unified gateway with two-model evaluation is not suitable when the gateway cannot expose the fields required for audit, cannot identify the actual model version, or cannot preserve the input and output artifacts long enough for the marketplace's review process. In that case, use direct integrations or a self-managed adapter. The selection criterion is traceability, not the length of the provider list.

The durable choice is the smallest boundary that protects the marketplace's decision rule. One credential and multiple models can reduce integration work, but the value comes from a repeatable answer to a harder question: did this image satisfy this job, at acceptable latency, with evidence someone can review?

## Sources

- https://platform.openai.com/docs/guides/embeddings
- https://github.com/pgvector/pgvector
