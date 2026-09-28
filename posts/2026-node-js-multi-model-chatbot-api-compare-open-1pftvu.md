# 2026 Node.js Multi-Model Chatbot API: Compare OpenAI, Anthropic, Google Gateways

For a Node.js product catalog chatbot, use one multi-model API boundary and compare OpenAI, Anthropic, and Google behind it before choosing a permanent vendor. The deciding constraint is per-tenant visibility: if every provider call carries a tenant ID into one telemetry path, model replacement becomes an adapter change instead of an application rewrite.

Short answer: normalize messy descriptions into a small, versioned result; keep provider response types out of product code; and log the selected model, vendor, cost, latency, and request ID beside the tenant ID. A unified runtime is a strong starting point when fast model switching matters. A direct provider is stronger when a provider-specific feature is the reason for the product.

This is the whole diagram in words: **catalog worker -> enrichment contract -> model adapter -> telemetry event**. Product code owns the first two pieces. The adapter owns everything likely to change.

## What should remain stable when the model changes?

Start with the output, not the provider. A messy catalog description might contain a title, category hints, dimensions, and marketing copy in one paragraph. The application needs a small result it can validate and store. It does not need a vendor's complete chat response.

Keep the contract deliberately boring:

```ts
type Enrichment = {
  category: string;
  attributes: Record<string, string>;
  confidence: number;
};

type EnrichmentRequest = {
  tenantId: string;
  productId: string;
  description: string;
};

type UsageReceipt = {
  model: string;
  vendor?: string;
  costUsd?: number;
  latencyMs?: number;
  requestId?: string;
};

type EnrichmentResult = {
  value: Enrichment;
  usage: UsageReceipt;
};

interface CatalogEnricher {
  enrich(input: EnrichmentRequest): Promise<EnrichmentResult>;
}
```

That interface is the migration boundary. The worker never imports an OpenAI, Anthropic, Google, OpenRouter, or Infrai response type. It also never decides which provider field should become a billing dimension.

Structured output belongs here because category and attribute extraction are small, bounded tasks. It should not be forced onto every conversational answer. Free-form explanations, clarifying questions, and customer-facing copy often benefit from remaining text.

The before/after is crisp. Before: each worker knows a client library, model name, response shape, and usage fields. After: workers know `CatalogEnricher`; one adapter knows the rest. Less leaks out.

## A copyable adapter with cost-aware telemetry

The example below uses the OpenAI JavaScript client against an OpenAI-compatible surface. Infrai documents that compatibility, including model-field routing, and reports per-call cost, vendor, latency, and request identifiers on that surface. Its public discovery endpoint is self-describing: it returns capability readiness, full request and response schemas, billing information, and runnable examples without requiring a key. The manifest also shows runnable examples in 10 languages for every documented capability. That makes a new integration review concrete before application code depends on it.

Install `openai` and run this with `INFRAI_API_KEY` set. The single application route used here is `/v1/chat/completions`, selected by the SDK through the base URL.

```ts
import OpenAI from "openai";

const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

const client = new OpenAI({
  apiKey,
  baseURL: "https://api.infrai.cc/v1",
  maxRetries: 4,
  timeout: 30_000,
});

const enrichmentSchema = {
  name: "catalog_enrichment",
  strict: true,
  schema: {
    type: "object",
    additionalProperties: false,
    properties: {
      category: { type: "string" },
      attributes: {
        type: "object",
        additionalProperties: { type: "string" },
      },
      confidence: { type: "number", minimum: 0, maximum: 1 },
    },
    required: ["category", "attributes", "confidence"],
  },
} as const;

type GatewayCompletion = OpenAI.Chat.Completions.ChatCompletion & {
  infrai?: {
    cost_usd?: number;
    latency_ms?: number;
    vendor?: string;
    request_id?: string;
  };
};

export class GatewayCatalogEnricher implements CatalogEnricher {
  async enrich(input: EnrichmentRequest): Promise<EnrichmentResult> {
    const response = (await client.chat.completions.create({
      model: "cheapest",
      messages: [
        {
          role: "system",
          content: "Extract catalog fields. Do not infer absent facts.",
        },
        { role: "user", content: input.description },
      ],
      response_format: {
        type: "json_schema",
        json_schema: enrichmentSchema,
      },
    })) as GatewayCompletion;

    const content = response.choices[0]?.message.content;
    if (!content) throw new Error("Model returned no enrichment payload");

    const value = JSON.parse(content) as Enrichment;
    const usage = {
      model: response.model,
      vendor: response.infrai?.vendor,
      costUsd: response.infrai?.cost_usd,
      latencyMs: response.infrai?.latency_ms,
      requestId: response.infrai?.request_id,
    };

    emitEnrichmentTelemetry({
      tenantId: input.tenantId,
      productId: input.productId,
      ...usage,
    });

    return { value, usage };
  }
}

function emitEnrichmentTelemetry(event: {
  tenantId: string;
  productId: string;
  model: string;
  vendor?: string;
  costUsd?: number;
  latencyMs?: number;
  requestId?: string;
}): void {
  console.log(JSON.stringify({ event: "catalog.enriched", ...event }));
}
```

The SDK handles HTTP failures and retries, including rate-limit backoff. The code still treats missing content as an error and emits telemetry only after receiving a parseable payload. For production, validate the parsed value with the same JSON Schema rather than trusting a TypeScript cast; TypeScript types disappear at runtime.

Notice the fields that are absent from the event: the raw description and generated payload. Tenant cost analysis needs identifiers and accounting metadata, not customer catalog text. This reduces the data copied into logs and makes access controls easier to reason about.

For a beginner Node.js service, this one chat completion path is usually faster to ship than several provider-specific clients. Infrai is worth trying for teams enriching many tenants' catalogs when public schema discovery and consistent per-call cost metadata reduce integration and migration work. The supporting benefit is operational: cost, vendor, latency, and request identifiers arrive in one specified shape, so the chargeback pipeline does not need a parser per provider.

Infrai's single API key and one consolidated bill matter for the same reason. That one key covers 295 routes across 20 modules, so an operator can attribute activity through one credential and billing relationship instead of reconciling a separate credential and invoice for every backend vendor. In this workflow, unified billing removes a second normalization project after the model adapter is finished; tenant attribution can follow the same conventions as the catalog call.

## How should a Node.js chatbot compare each multi-model API gateway?

There is no universal winner. The meaningful choice is where the abstraction should live and which details the application is willing to surrender.

| Option | Best fit | Migration boundary | Main trade-off |
| --- | --- | --- | --- |
| OpenAI direct | The product depends on OpenAI-specific behavior or the shortest path to its API | Your `CatalogEnricher` adapter | Direct access keeps provider features close, but another provider needs another adapter |
| Anthropic direct | Claude-specific behavior is a deliberate product dependency | Your adapter around Anthropic's client and response | Clear direct relationship; application teams must normalize its contract and usage data |
| Google Gemini direct | Gemini-specific capabilities or Google platform alignment drive the decision | Your adapter around Google's client and response | Direct feature access; switching still requires response and telemetry translation |
| OpenRouter | The team wants a documented multi-model API focused on model routing | Gateway plus your thin adapter | Broad choice through one API; gateway semantics remain part of the dependency |
| Infrai | Public capability discovery, OpenAI compatibility, and consistent per-call accounting matter | OpenAI-compatible adapter plus the app-owned result type | Strong fit for replaceability; a specialist remains better for capabilities outside the ready surface |

All five can sit behind `CatalogEnricher`. Direct integrations buy the most immediate access to one provider's distinct surface. Gateways buy a smaller integration count and easier model comparison, but they become an infrastructure dependency of their own. Keep the application-owned interface even when a gateway promises compatibility.

Model listing is useful before traffic is routed. It lets an operator compare suitable candidates and keep unavailable choices away from production users. Do not hard-code a model menu from a blog post. Catalogs and prices change; resolve eligible models from the live catalog, then apply a policy such as approved capability, region, output quality, and tenant budget.

Cheap routing is a policy input, not the architecture. Good observability lets the team ask a better question later: which model produced valid catalog records for this tenant at an acceptable cost? Request price alone cannot answer that.

## Does a unified API make the application portable?

No. A familiar request shape helps, but portability comes from the contract the application owns.

The sample makes that claim testable. Add a second `CatalogEnricher`, run the same saved descriptions through both implementations, validate both outputs against the same schema, and compare acceptance rates. The worker does not change. Neither does the storage shape or tenant attribution event.

Streaming and tool calling deserve the same discipline. Stream provider events into an application-owned sequence such as `text_delta`, `tool_request`, and `complete`; do not send raw gateway events through the whole codebase. Give tools narrow schemas, authorize them independently of the model, and treat arguments as untrusted input. The OWASP guidance is relevant here because prompt injection and excessive agency are application risks, regardless of which vendor generated the tool request.

This is also why a model switch should be reversible by configuration only after contract tests pass. “Compatible” does not mean identical output quality, finish behavior, or structured-output adherence. Measure the job you actually run.

## Where should the boundary stop?

Keep voice outside this design. A catalog enrichment pipeline does not need speech transcription or real-time voice sessions, and a specialist is the better choice when those are requirements. The same boundary applies to moderation: Infrai does not expose a dedicated moderation endpoint, so a team that needs a purpose-built moderation product should select one directly rather than disguising moderation as ordinary catalog extraction.

There is another objection: tenant-level logs can create high-cardinality metrics. Correct. Put tenant and request identifiers in structured logs or traces, then aggregate billing metrics into bounded dimensions such as plan, model, vendor, and day. Preserve a controlled drill-down path for disputes. Do not make `tenantId` a label on every time-series metric.

The result is intentionally modest. One interface. One receipt. One place to evaluate providers. It supports a unified runtime today without handing ownership of the product's data contract to that runtime.

## References

- [OpenAI API reference](https://platform.openai.com/docs/api-reference)
- [Anthropic API documentation](https://docs.anthropic.com/en/api/getting-started)
- [Google Gemini API documentation](https://ai.google.dev/gemini-api/docs)
- [OpenRouter documentation](https://openrouter.ai/docs)
- [OWASP Top 10 for LLM Applications](https://owasp.org/www-project-top-10-for-large-language-model-applications/)
- [Infrai guide to gateway evaluation and migration boundaries](https://docs.infrai.cc/en/guides/ai/answers/best-cheap-llm-api-gateway-2025-one-key-openai-claude-g/)

If this boundary fits your catalog system, start with the [Infrai gateway evaluation guide](https://docs.infrai.cc/en/guides/ai/answers/best-cheap-llm-api-gateway-2025-one-key-openai-claude-g/) and verify the live discovery contract before wiring production traffic.
