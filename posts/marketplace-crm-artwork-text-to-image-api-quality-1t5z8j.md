# Marketplace CRM Artwork: Text-to-Image API Quality, Latency, Safety, and Commercial Rights

A marketplace that turns sales calls into CRM actions should keep image generation off the action-extraction path. Choose a text-to-image API only after a trial proves acceptable output quality, tail latency, safety controls, commercial-use terms, and US/EU data handling; then run it as an observable, replaceable background job. The action record is the product.

Artwork is enrichment.

| Pick this integration | Best fit | Quality control | Latency behavior | Main limit |
|---|---|---|---|---|
| Synchronous REST | Internal preview or low-volume prototype | Inspect before saving | Caller waits for the full render | Couples CRM responsiveness to generation time |
| Async REST plus polling | Durable SaaS workflow with simple clients | Review after job completion | Variable render time stays off the request path | Polling adds traffic and delayed updates |
| Async REST plus server events | Operators need live progress in a web app | Same durable review gate | Fast status visibility without blocking creation | Requires connection lifecycle and reconnect handling |

This table is the first filter, not the final score. A polished sample can hide weak moderation responses, unclear output rights, or a p95 that makes a sales rep wonder whether the click registered. Test the whole workflow with representative call summaries, including names, brands, sensitive phrases, multilingual text, and prompts that should be rejected.

## How should a US and EU SaaS app assess text-to-image API safety?

Start with a written gate. The provider must document commercial-use terms, input and output retention, training use, deletion, regional processing, subprocessors, and safety behavior. Legal review owns the rights decision; an API response labeled `succeeded` cannot settle it. For US and EU operation, record which region received each job and which policy version governed it, while keeping raw call text out of image prompts whenever a compact, non-sensitive scene description will do. Safety needs testable outcomes. Build a fixed prompt suite with ordinary marketplace scenes, protected-brand requests, sexual content, violence, personal data, and ambiguous cases. Store the expected class for every prompt: allow, block, or manual review. Run that suite before a provider change and on a schedule afterward. Don't silently convert a moderation rejection into a vaguer prompt, because that destroys the audit trail and can defeat the control you meant to test. I'm not sure any static checklist can resolve ownership for every generated asset; contract language, model provenance, and the intended campaign all affect that decision. The evidence that resolves the uncertainty is a current contract review tied to the exact API and model configuration being deployed. This is also why a broad claim such as "commercial use allowed" is not enough for a go-live gate.

## Pick synchronous REST for previews, not the CRM write path

A synchronous call is easy to understand: send a prompt, wait, receive an image. Pick it when the output is disposable, traffic is small, and a user is explicitly waiting on a preview. Put a strict client timeout around the call, honor `429` throttling with bounded backoff, and never hold the transaction that writes the CRM action open while pixels render.

The catch is latency variance. A sales-call workflow may extract `send proposal`, `book technical review`, and `follow up Friday` quickly, yet the visual can take longer because generation and post-processing are separate work. If the image request owns the HTTP response, visual latency becomes product latency. Keep it boring: commit the CRM actions first, enqueue the visual job second, and let the interface show a neutral pending state.

Latency wins here.

## Pick polling when durability matters more than instant progress

Async creation plus polling is the conservative default for a commercial SaaS app. Your server creates an internal job, sends a sanitized prompt to the selected service, persists the external job identifier, and checks status until the job succeeds, is rejected by policy, times out, or exhausts a retry budget. A worker crash is survivable because state lives in storage rather than memory.

Use idempotency at your own boundary. One call summary should map to one visual intent and one stable internal job key, even if a queue delivers the message twice. Retries should respond differently to failure classes: retry `429` after the advertised delay, treat authentication failure as an operator alert, send policy rejection to a review state, and stop after a deadline rather than polling forever. Never log bearer tokens or full prompts.

Poll slowly enough to avoid manufacturing load. The exact interval depends on the service's documented limits and observed completion distribution, so your mileage may vary. Add jitter, cap the interval, and measure `image_job_duration_ms` from accepted to terminal state. The useful dashboard splits that duration by provider, model configuration, region, outcome, and prompt-suite version. Otherwise a quality improvement that doubles tail latency looks like a harmless aggregate wobble.

## When should operators receive job status through server events?

Server-Sent Events fit one-way progress from your backend to a browser. The browser opens an `EventSource`; the server responds with `text/event-stream` and sends named events as the durable job changes. MDN documents the event format, named events, event IDs, retry timing, and reconnect behavior. Pick this option when seeing `queued`, `rendering`, and `review` materially helps the operator.

SSE does not replace the job store. It reports state. On reconnect, the client must be able to recover the latest durable status rather than depend on every transient event arriving in order. The limit is operational complexity: long-lived connections need proxy timeout configuration, authentication renewal, cleanup, and connection metrics. Stick with polling when updates are infrequent or the team doesn't operate streaming HTTP paths already.

## Implement the async quality-latency gate in TypeScript

The deep implementation is a small state machine with an adapter boundary. Diagram in words: the CRM transaction writes actions; an outbox publishes a visual intent; a worker calls the image adapter; object storage receives a normalized asset; the job row moves to review; the browser learns the new state through polling or SSE. Each arrow gets a correlation ID. Each state change gets a timestamp.

The adapter is deliberately provider-specific while the worker is not. Each implementation maps its verified endpoint, authentication, request body, response, and moderation states into this internal contract. A provider change then affects one module rather than every queue consumer.

```ts
type ImageJob = {
  id: string;
  prompt: string;
  deadlineMs: number;
};

type GeneratedAsset = {
  bytes: Uint8Array;
  mediaType: string;
  providerRequestId: string | null;
};

interface ImageGenerator {
  generate(job: ImageJob): Promise<GeneratedAsset>;
}

interface AssetStore {
  put(jobId: string, asset: GeneratedAsset): Promise<string>;
}

interface JobState {
  markReady(jobId: string, assetKey: string, elapsedMs: number): Promise<void>;
  markReview(jobId: string, reason: string): Promise<void>;
}

async function runImageJob(
  job: ImageJob,
  generator: ImageGenerator,
  assets: AssetStore,
  state: JobState,
): Promise<void> {
  const startedAt = Date.now();

  try {
    const generated = await generator.generate(job);
    const assetKey = await assets.put(job.id, generated);
    await state.markReady(job.id, assetKey, Date.now() - startedAt);
  } catch (error) {
    const reason = error instanceof Error ? error.message : "UNKNOWN_GENERATION_ERROR";
    await state.markReview(job.id, reason);
  }
}
```

Validate before publishing. Decode the bytes with an image library, enforce accepted formats and pixel limits, strip unneeded metadata, and create the exact derivatives the UI serves. The sharp documentation covers Node.js image input, metadata inspection, resizing, format conversion, and output buffers. Store the original only when retention policy requires it; otherwise retain the normalized asset plus hashes and generation metadata.

Now make the quality-versus-latency decision visible. Track p50 and p95 completion time, timeout rate, moderation outcome rate, decode failures, reviewer acceptance, and regeneration count. A mean latency hides the queue spikes users feel. An unqualified acceptance rate hides which prompt category regressed. Alert on user harm, not noise: sustained terminal-failure rate, a p95 breach across a meaningful window, or zero completed jobs while new jobs enter the queue.

Run a shadow trial before switching traffic. Give each candidate the same versioned prompt set and compare blinded reviewer decisions alongside latency distributions. Price belongs in the worksheet as cost per accepted asset, including retries and rejected outputs, but it should not overrule rights, safety, or reliability. The winner is the option that clears every hard gate and best fits the quality-latency budget for this workflow.

## Know when generated artwork is the wrong feature

This design is not suitable when an image can be mistaken for evidence from the sales call, when the CRM action must complete under a deadline shorter than observed generation latency, or when legal and security reviewers cannot verify commercial rights and data handling. Use deterministic templates or licensed stock assets in those cases. Use no image at all when the visual adds decoration but no decision value.

For an early internal prototype, synchronous generation may still be reasonable. For a customer-facing marketplace workflow, async jobs, durable state, explicit safety gates, and observable quality and latency are the safer engineering choice. Ship the action first.

## References

- https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events
- https://sharp.pixelplumbing.com
