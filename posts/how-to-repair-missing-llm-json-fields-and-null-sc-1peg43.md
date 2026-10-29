# How to Repair Missing LLM JSON Fields and Null Schema Values: Node.js Enum Extraction

Short answer: treat a missing field, a JSON `null`, and an invalid enum as three different events. Normalize them behind one small adapter, validate the normalized object, and keep the rest of the sales-call pipeline unaware of which model produced it. That is the practical portability boundary for a one-person SaaS.

| Option | Strength | Cost or risk |
| --- | --- | --- |
| Provider-native structured output | Strong schema enforcement when supported | Ties request behavior to one API contract |
| Prompt-only JSON | Easy to start | Missing keys and enum drift reach production |
| Adapter plus JSON Schema validation | Portable, testable, explicit failures | Requires a little local code |

My choice is the third row. The adapter owns extraction quirks. JSON Schema owns the business contract. The CRM writer receives only a canonical object. That separation keeps shipping weekly possible and puts scarce founder time on revenue-producing work instead of provider-specific repair code.

## Which LLM JSON schema should handle missing fields and null values?

Start with meaning, not syntax. A media sales call may contain a publication, an advertiser, a campaign window, a budget, a next action, and an outcome. The transcript may omit the budget entirely. “Not discussed” is different from “the caller said there is no budget,” and both differ from an extraction failure.

I use explicit nullable fields for facts that can be absent, and a closed enum for workflow state. I do not let the model invent a default status merely to satisfy a required-key check. The schema should make that choice visible:

```ts
export const salesCallSchema = {
  type: "object",
  additionalProperties: false,
  required: ["account", "campaign", "budgetUsd", "nextAction", "outcome"],
  properties: {
    account: { type: ["string", "null"] },
    campaign: { type: ["string", "null"] },
    budgetUsd: { type: ["number", "null"] },
    nextAction: { type: ["string", "null"] },
    outcome: { type: "string", enum: ["qualified", "nurture", "closed_lost", "unknown"] }
  }
} as const;
```

The `null` union is intentional. “Missing key” is a transport or model-shape problem; `null` is a domain value. Conflating them makes downstream analytics lie. Use JSON Schema 2020-12 or another published draft consistently, and pin the validator version in the application lockfile. The draft controls vocabulary; your adapter still controls how an absent fact is represented. The three-state rule is the useful one: missing means inspect the shape, `null` means the fact is absent, and an enum mismatch means the mapping needs review.

## What happens after LLM JSON extraction reaches the CRM?

There are two boundaries. First, the extraction response is parsed. Second, the parsed value is mapped into the CRM command. A key can survive the first boundary and disappear at the second through a truthiness check such as `budgetUsd || undefined`. A zero budget is a real value, even if it is unusual.

That is the failure.

The other common failure is enum translation. Sales language is messy: “maybe next quarter,” “not ready,” and “no fit” are not enum literals. Put that translation in a deterministic function. It should return `unknown` when evidence is weak, not force the nearest label.

```ts
type RawRecord = {
  account?: unknown;
  campaign?: unknown;
  budgetUsd?: unknown;
  nextAction?: unknown;
  outcome?: unknown;
};

type SalesCall = {
  account: string | null;
  campaign: string | null;
  budgetUsd: number | null;
  nextAction: string | null;
  outcome: "qualified" | "nurture" | "closed_lost" | "unknown";
};

function textOrNull(value: unknown): string | null {
  return typeof value === "string" && value.trim() ? value.trim() : null;
}

function numberOrNull(value: unknown): number | null {
  return typeof value === "number" && Number.isFinite(value) ? value : null;
}

function normalizeOutcome(value: unknown): SalesCall["outcome"] {
  if (value === "qualified" || value === "nurture" || value === "closed_lost") return value;
  return "unknown";
}

export function normalize(raw: RawRecord): SalesCall {
  return {
    account: textOrNull(raw.account),
    campaign: textOrNull(raw.campaign),
    budgetUsd: numberOrNull(raw.budgetUsd),
    nextAction: textOrNull(raw.nextAction),
    outcome: normalizeOutcome(raw.outcome)
  };
}
```

This is deliberately boring. Boring code is cheap to replace when a provider changes its response format.

Define one internal interface and make each provider adapter return it. The adapter can parse a native schema response, plain JSON, or a repair attempt. None of those details should leak into CRM code.

During a migration, I keep the old adapter and the new adapter behind the same interface until both pass the same fixtures. A fixture with a missing budget must produce the same `budgetUsd: null` result in both paths. A fixture with an unrecognized outcome must produce `unknown`, not whichever label happens to be closest. This is where portability becomes observable: the comparison is over canonical business data, not over two providers' raw response shapes. It takes more test work up front, but it prevents a provider switch from quietly changing CRM stages.

The decision rule is simple: if a provider cannot return enough evidence for a field, return `null` or `unknown` and record the validation reason. Do not silently retry forever. One repair attempt is usually enough for an interactive workflow; after that, queue the transcript for review or leave the CRM action unapplied. The exact retry count is an operational policy, so keep it in configuration and test it.

For tests, keep fixtures small and ugly. Include a transcript with no budget, one with `budgetUsd: null`, one with the string `"unknown"`, and one with an extra property. Assert the canonical result and the validation error separately. This catches prompt regressions without depending on a live model.

I also log a schema version, adapter name, validation result, and request correlation ID. I do not log the full transcript by default; sales calls can contain personal and commercial information. Portability includes the ability to change a provider without changing retention policy or CRM semantics.

The limitation is clear: the adapter cannot recover a fact that was never said, and it cannot make incompatible provider schema subsets identical. A portability layer also adds maintenance and a second test surface. For a low-risk prototype with human review, that overhead may be the wrong trade-off.

## What does the portable contract leave unresolved?

Provider-native structured output is the better fit when the team already accepts that provider's request and response contract, its supported schema subset matches the record, and the workflow can tolerate its operational constraints. It can remove local parsing work. That is a valid trade.

Prompt-only JSON is reasonable for a prototype that never writes to a CRM and has a human checking every result. It stops being reasonable when an absent field triggers a task, changes a pipeline stage, or enters a report. At that point, a prompt is guidance, not a contract.

The portable adapter is extra code, but it buys a clean replacement point. For a solo builder, that matters more than shaving a few lines from the first integration. The test suite tells me whether a change preserves business meaning. The schema tells me what the CRM is allowed to receive.

## References

- https://json-schema.org/understanding-json-schema/reference/type
- https://json-schema.org/understanding-json-schema/reference/enum
- https://ajv.js.org/guide/getting-started.html
- https://nodejs.org/api/assert.html
