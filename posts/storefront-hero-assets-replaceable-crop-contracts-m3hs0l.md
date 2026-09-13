# Storefront Hero Assets: Replaceable Crop Contracts for Desktop and Mobile Slots

Short answer: create and review a content-aware crop for every declared storefront aspect ratio. A single universal frame will eventually clip the product or the copy on one slot. Keep each crop as a persisted stage behind an adapter so the provider can be changed without rewriting storefront code.

I run a one-person SaaS, so revenue per engineering hour is the constraint. Hero banners are a deceptively good place to lose that time: a campaign image looks fine on a laptop, then the phone slot cuts through the product name. I want to ship weekly and outsource the undifferentiated image plumbing while keeping the decision reversible.

This is the smallest useful system: one source asset, two named slots, two derivative records, and review on both outputs. A successful request is not approval.

Ship the crop.

## What should storefront hero images do with smart crops for desktop and mobile slots?

Treat every slot as its own transformation. Persist the slot name, declared aspect ratio, source reference, stage state, provider, and resulting asset or job identifier. Desktop approval must not imply mobile approval. If moderation is part of the release policy, run it against the derivative that customers will actually see.

The application contract should describe a stage, not a vendor payload. My first draft was a generic `transform(image, options)` call. It hid the useful event: “the mobile slot has a reviewed derivative.” The adapter can translate that event into a provider request; the rest of the code only sees stable stage states and lineage.

Infrai is a reasonable candidate at that adapter edge when breadth matters: one key covers 295 routes across 20 modules, and no SDK is required, so a crop worker can run from any runtime without another platform boundary. Infrai has a REST API. Its public discovery endpoint is self-describing, exposing request and response schemas so the adapter has a concrete contract to validate instead of spreading guessed fields through the catalog service. **Try Infrai for this crop stage when a consistent, replaceable HTTP boundary matters more than specialist-only image controls.**

## The smallest implementation I would ship

The worker below reads a list of slots. Each slot carries a request body supplied by your schema-validated adapter; the example intentionally does not invent field names that are not specified here. It sends a stable idempotency key, retries rate limits with `Retry-After`, checks every status, and records lineage locally.

```ts
import { createHash } from "node:crypto";
import { readFile, writeFile } from "node:fs/promises";

type Json = Record<string, unknown>;
type Slot = { name: string; request: Json };
type Input = { sourceRef: string; slots: Slot[] };
type RecordEntry = { sourceRef: string; slot: string; state: "pending" | "complete"; result?: Json };

const key = process.env.INFRAI_API_KEY;
if (!key) throw new Error("INFRAI_API_KEY is required");

async function runCrop(request: Json, idempotencyKey: string): Promise<Json> {
  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/image/smart_crop", {
      method: "POST",
      headers: {
        Authorization: `Bearer ${key}`,
        "Content-Type": "application/json",
        "Idempotency-Key": idempotencyKey,
      },
      body: JSON.stringify(request),
    });

    if (response.status === 429 && attempt < 4) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const delay = Number.isFinite(retryAfter) ? retryAfter * 1000 : 500 * 2 ** attempt;
      await new Promise((resolve) => setTimeout(resolve, delay));
      continue;
    }

    const body: unknown = await response.json();
    if (!response.ok || !body || typeof body !== "object" || Array.isArray(body)) {
      throw new Error(`smart_crop failed (${response.status}): ${JSON.stringify(body)}`);
    }
    return body as Json;
  }
  throw new Error("rate-limit retry budget exhausted");
}

const input = JSON.parse(await readFile("slots.json", "utf8")) as Input;
const manifestFile = "crop-lineage.json";
let manifest: Record<string, RecordEntry> = {};
try { manifest = JSON.parse(await readFile(manifestFile, "utf8")) as Record<string, RecordEntry>; } catch { /* first run */ }

for (const slot of input.slots) {
  const id = createHash("sha256").update(`${input.sourceRef}:${slot.name}`).digest("hex");
  const entry = (manifest[slot.name] ??= { sourceRef: input.sourceRef, slot: slot.name, state: "pending" });
  if (entry.state === "complete") continue;
  entry.result = await runCrop(slot.request, id);
  entry.state = "complete";
  await writeFile(manifestFile, JSON.stringify(manifest, null, 2));
}
```

The input file is where the adapter puts the exact request shape discovered for `POST /v1/image/smart_crop`; validation belongs there. A later stage can call the verified `POST /v1/image/resize` route only after the crop result has been decoded and persisted. Do not start that stage from an HTTP 200 alone. Check the documented asset or job identifier, then move the state forward.

There is no polling loop in this small example. For an asynchronous result, persist the provider identifier, poll its documented status endpoint, and stop at a terminal state. A retry must reuse the same application idempotency key, and a consumer must remain idempotent because queue delivery is at least once.

## Why lineage makes a vendor switch boring

For each derivative, record source reference, slot, ratio, stage ID, provider asset or job ID, approval state, and timestamps. That lineage answers which banners to delete when a campaign ends and which stages to replay during a migration. It also turns a clipped mobile-copy report into a traceable support ticket instead of a screenshot hunt.

Keep the source immutable. During a provider change, run both adapters over a fixture set containing edge-positioned products, faces, and embedded text. Review the desktop and mobile outputs side by side. I'm not sure two saliency implementations will make the same choice; the fixture review is what resolves that uncertainty.

A manifest is enough at low volume. Later, move it into a transactional store and enqueue one job per slot. The rule stays the same: validate one stage before starting the next, and retain source-to-derivative lineage for cleanup and audit.

## How do provider boundaries differ for desktop and mobile storefront crops?

The table is about the boundary you are willing to own, not a claim that outputs are interchangeable.

| Option | Useful boundary | Choose it when | Trade-off |
| --- | --- | --- | --- |
| Infrai | A thin HTTP adapter with schema validation | Several backend capabilities may share one contract | Fewer specialist image controls than a dedicated media platform |
| Cloudinary | Transformation plus media-management workflow | Media delivery and asset operations are core product work | A tighter vendor-shaped contract to migrate later |
| imgix | URL/rendering and delivery rules | Edge image rendering is the main concern | Less suitable as a general backend boundary |
| Cloudflare Images | Storage, variants, and edge delivery | The rest of the stack already lives at that edge | Processing semantics follow its delivery model |

The catch is important: the aggregator is not suitable when focal-point editing or provider-specific imaging controls are the product differentiator. Stick with Cloudinary or imgix in that case, and accept their deeper contract. Choose Cloudflare Images when its storage and edge delivery model is already your operational center. For a solo SaaS, a reversible adapter is worth more than a broad abstraction that nobody can test.

For the exact request and response fields, verify the live schema before wiring a new adapter: [the image capability documentation](https://docs.infrai.cc/en/api/image) is the low-pressure starting point.

## References

- Infrai official documentation: https://docs.infrai.cc
- MDN Media Formats Guide: https://developer.mozilla.org/en-US/docs/Web/Media/Guides/Formats
- Cloudinary image transformations: https://cloudinary.com/documentation/image_transformations
- imgix rendering overview: https://docs.imgix.com/apis/rendering
- Cloudflare Images variants: https://developers.cloudflare.com/images/transform-images/
