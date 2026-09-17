# How to Implement Multi-Source Board Books with Asynchronous Jobs (and Secure Retention)

A media board book can pull an order, advertiser details, issue metadata, and artwork from separate systems. The constraint that changes the design is batch throughput: fetching or rendering every invoice at once can consume the process long before the batch is finished. **Use a durable job per immutable order revision, validate before rendering, cap worker concurrency, and give each attempt a private temporary directory that is removed in `finally`.** Keep the final PDF under a separate retention policy.

Short answer: the request handler should enqueue identifiers, not source data or files; a worker should rebuild a validated snapshot, render idempotently, publish once, and erase its workspace on every exit path. This keeps retries useful without turning temporary files into an unofficial archive.

Ship the boring boundary first. It protects revenue-per-hour because an invoice batch can be replayed without someone sorting duplicate PDFs or hunting abandoned artwork on disk.

## How should a Node.js service validate multi-source board books before asynchronous jobs?

Do not let a long request own the work. The HTTP layer should authenticate the caller, create a batch record, and enqueue one compact job for each order revision. A payload such as `{ orderId, revision, batchId }` is enough. Customer names, addresses, rates, and artwork URLs stay in their systems of record; copying them into queue payloads creates another retention surface and makes a later retry depend on stale values.

The worker then reads all sources and constructs one snapshot. Validation belongs immediately after that read, before a browser, PDF renderer, or temporary file is started. Cross-source checks matter more than checking types in isolation: the order currency must match every line item, artwork must belong to the requested issue, invoice numbers must be present, and totals must be recomputed from integer minor units rather than trusted from display strings. Keep the snapshot ordered so the same order revision produces the same input bytes.

Here is the boundary I use for that step. It relies on injected repositories, so the domain code does not know which database, queue, or renderer is behind it.

```ts
type Job = {
  batchId: string;
  orderId: string;
  revision: number;
};

type Line = {
  description: string;
  quantity: number;
  unitPriceMinor: number;
  currency: string;
};

type Snapshot = {
  invoiceNumber: string;
  issueId: string;
  customerName: string;
  currency: string;
  lines: Line[];
  artwork: Uint8Array[];
};

function validateSnapshot(value: Snapshot): Snapshot {
  if (!value.invoiceNumber || !value.issueId || !value.customerName) {
    throw new Error("INVALID_REQUIRED_FIELD");
  }
  if (value.lines.length === 0) throw new Error("INVALID_EMPTY_INVOICE");

  for (const line of value.lines) {
    if (!Number.isSafeInteger(line.quantity) || line.quantity <= 0) {
      throw new Error("INVALID_QUANTITY");
    }
    if (!Number.isSafeInteger(line.unitPriceMinor) || line.unitPriceMinor < 0) {
      throw new Error("INVALID_UNIT_PRICE");
    }
    if (line.currency !== value.currency) throw new Error("CURRENCY_MISMATCH");
  }

  return {
    ...value,
    lines: [...value.lines].sort((a, b) =>
      a.description.localeCompare(b.description)
    ),
  };
}
```

Validation errors are permanent for that revision. Record them with a small error code and field path, then stop retrying. A timeout while reading a source is different: it may succeed later. This classification prevents a malformed invoice from burning through the same retry budget as a transient dependency failure. Never log the whole snapshot to explain either case.

Retries are not validation.

## The smallest worker that is still safe

A useful job state machine is `queued -> running -> published`, with `retryable` and `rejected` exits. Claim jobs with a lease or visibility timeout so a worker that disappears does not own work forever. The publish operation needs a unique key based on the invoice identity and revision, not the attempt number. That is the idempotency boundary.

Retries should be finite. Exponential backoff with jitter spreads attempts out, while an absolute attempt limit gives operators a real terminal state. Queue products expose these concepts differently: BullMQ documents attempts and backoff for Redis-backed queues; RabbitMQ documents acknowledgements and consumer prefetch; Amazon SQS documents visibility timeouts and dead-letter queues. Those are engineering boundaries, not a ranking. A single-process in-memory array has none of those crash-recovery guarantees and is not suitable when accepted work must survive a restart.

Consider one concrete replay. Order `ord_417` at revision `6` is claimed, its four sources validate, and the renderer writes a valid PDF, but the process loses its lease before publication is confirmed. The next worker receives the same compact job rather than a copied customer payload. It loads revision `6` again, obtains the same canonical line ordering, writes into a new random workspace, and publishes with `invoice/ord_417/6`. If the first publication committed, `publishOnce` recognizes that key and leaves the durable object unchanged; if it did not, the retry completes it. In both branches the active attempt removes its own directory. The batch counter advances only from the durable published state. This is why the order revision, publication key, lease, and workspace lifetime must be separate concepts: collapsing them into one attempt ID makes a normal redelivery look like a new invoice.

This worker core makes cleanup unconditional and keeps output publication separate from rendering:

```ts
import { mkdtemp, open, readFile, rm } from "node:fs/promises";
import { tmpdir } from "node:os";
import { join } from "node:path";
import { randomUUID } from "node:crypto";

type Dependencies = {
  loadSnapshot(job: Job): Promise<Snapshot>;
  renderPdf(snapshot: Snapshot, outputPath: string): Promise<void>;
  publishOnce(key: string, pdf: Uint8Array, deleteAfter: Date): Promise<void>;
};

export async function runInvoiceJob(
  job: Job,
  deps: Dependencies,
  now = new Date()
): Promise<void> {
  const snapshot = validateSnapshot(await deps.loadSnapshot(job));
  const workspace = await mkdtemp(join(tmpdir(), "board-book-"));
  const outputPath = join(workspace, `${randomUUID()}.pdf`);

  try {
    const reserved = await open(outputPath, "wx", 0o600);
    await reserved.close();
    await deps.renderPdf(snapshot, outputPath);

    const pdf = await readFile(outputPath);
    if (pdf.byteLength < 5 || pdf.subarray(0, 5).toString() !== "%PDF-") {
      throw new Error("INVALID_PDF_SIGNATURE");
    }

    const key = `invoice/${job.orderId}/${job.revision}`;
    const deleteAfter = new Date(now.getTime() + 30 * 24 * 60 * 60 * 1000);
    await deps.publishOnce(key, pdf, deleteAfter);
  } finally {
    await rm(workspace, { recursive: true, force: true });
  }
}
```

The `wx` flag requests exclusive creation, and mode `0o600` limits the file to its owner on POSIX systems. Generated names avoid turning an invoice number or uploaded filename into a path. The directory comes from `mkdtemp`, not from a predictable shared name. These controls are defense in depth; production workers should also run as an unprivileged identity with an isolated temporary volume.

Cleanup is correctness.

Thirty days in the example is an application policy, not a universal recommendation. Legal, accounting, and customer-contract requirements decide final invoice retention. The temporary workspace has a much shorter lifetime: one attempt. Do not use the workspace cleanup timer as the control for published documents. Store `deleteAfter` with the durable object, enforce deletion in the storage layer, and audit deletion without putting customer data into the audit event.

Fast failure matters here. If publishing fails after rendering, the retry renders again but `publishOnce` prevents a second durable object for the same order revision. If cleanup itself fails, report the workspace identifier and worker identity to an operational channel, never the invoice contents; an external janitor can delete expired worker directories as a second control.

## Throughput comes from bounds, not a giant Promise.all

A batch of 2,000 invoices is not one job. It is 2,000 independently claimable jobs under one batch ID. That shape allows partial progress, targeted replay, and a batch status derived from counters instead of one enormous promise. It also gives backpressure somewhere concrete: the queue depth grows while the number of active renderers remains fixed.

Start concurrency from the scarce resource. PDF rendering is often CPU- and memory-heavy, while source reads wait on I/O. Run a load test with representative pages and artwork, watch peak resident memory, event-loop delay, render latency, and dependency throttling, then choose the worker count. I'm not sure what the right number is for an unknown template; no honest default can resolve that. A constrained production-like test can.

Keep fetching bounded too. This tiny mapper maintains a fixed number of active tasks and preserves result ordering:

```ts
export async function mapBounded<T, R>(
  values: readonly T[],
  concurrency: number,
  fn: (value: T) => Promise<R>
): Promise<R[]> {
  if (!Number.isSafeInteger(concurrency) || concurrency < 1) {
    throw new Error("INVALID_CONCURRENCY");
  }

  const results = new Array<R>(values.length);
  let next = 0;

  async function consume(): Promise<void> {
    while (true) {
      const index = next++;
      if (index >= values.length) return;
      results[index] = await fn(values[index]);
    }
  }

  const workers = Array.from(
    { length: Math.min(concurrency, values.length) },
    () => consume()
  );
  await Promise.all(workers);
  return results;
}
```

Don't optimize only invoices per minute. Track accepted-to-published latency, retry rate by error class, permanent rejection rate, oldest queued age, and cleanup age. The last metric catches a privacy regression that throughput graphs miss. Logs should carry `batchId`, `orderId`, `revision`, `attempt`, and a stable error code; omit customer names, addresses, source payloads, signed URLs, and document bytes.

Ship weekly, but make the test suite mean something. Unit tests cover total calculation, canonical ordering, error classification, and retention timestamps. Integration tests kill a worker after render but before publish, deliver the same job twice, deny publication, and assert that the workspace disappears. A batch test should use many small invoices plus a few artwork-heavy ones, because averages hide the tail that determines safe concurrency.

## What I would change when the invoice batch grows

At larger volume, separate fetch/validate from render only when measurements show different scaling limits. The catch is that an extra stage creates another durable handoff and another retention surface. If validated snapshots contain personal data, encrypt them, give them their own deletion deadline, and restrict access as tightly as the final document. Staying with a single worker stage is better while it meets the batch window; fewer persisted copies are easier to reason about.

I would also add admission control per tenant, explicit source timeouts, cancellation for superseded order revisions, and a reconciliation task that compares published keys with job states. The reconciliation task must be read-only until it proves a mismatch; automated deletion based on an ambiguous state is too risky. For very large artwork, stream between bounded stages instead of assembling every source in one `Blob`. The Blob API is useful for immutable byte sequences, but calling `arrayBuffer()` necessarily materializes those bytes in memory, so the decision belongs in the throughput model.

This architecture is not suitable for interactive previews that must reflect keystrokes immediately; use a short-lived synchronous preview path with strict size limits there. It is also more machinery than a tiny, disposable internal batch needs. Stick with a local sequential script when losing the process means rerunning a harmless batch and no private files remain afterward. Move to durable jobs when accepted work, retries, and auditable retention become product obligations.

The decision rule is plain: outsource the undifferentiated queue and storage plumbing if operating it steals feature time, but keep snapshot validation, idempotency keys, privacy boundaries, and deletion policy in your own domain code. Those rules survive a provider change.

## References

- https://nodejs.org/api/fs.html#fspromisesmkdtempprefix-options
- https://nodejs.org/api/fs.html#file-system-flags
- https://nodejs.org/api/crypto.html#cryptorandomuuidoptions
- https://developer.mozilla.org/en-US/docs/Web/API/Blob
- https://docs.bullmq.io/guide/retrying-failing-jobs
- https://www.rabbitmq.com/docs/confirms
- https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-visibility-timeout.html
- https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html
