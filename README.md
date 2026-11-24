# Open-Source LLM and Agent Observability Stack


> **Decision:** Use this guide to choose capture boundaries and trace model calls, retrieval, tools, and agent decisions without making one UI the data contract; skip it if you expect observability to replace privacy policy or evaluation.
>
> **For:** LLM platform engineers, agent operators, and incident investigators  ·  **Not for:** teams seeking a product leaderboard or permission to retain raw sensitive payloads

Instrument model calls, retrieval, tool execution, and agent decisions without making one UI your data contract. Start by choosing the capture boundary and privacy policy; then choose instrumentation and storage.

“Open source” does not by itself prove that every hosted feature is self-hostable, that schemas are interchangeable, or that payloads are safely redacted.

## Two reference architectures

### A. OpenTelemetry-first

```text
application / agent
  → explicit or auto instrumentation
  → OpenTelemetry/OpenInference-shaped spans
  → local OTel Collector (redact/filter/sample)
  → chosen backend (for example SigNoz or an AI-focused backend)
```

Use when portability and control before export matter. Verify the exact semantic-convention version and each backend’s OTLP/schema behavior; project identity alone does not prove compatibility.

### B. Proxy-first

```text
application
  → LLM proxy/gateway (for example Helicone-style path)
  → provider
  ↘ request/cost/latency records + optional export/backend
```

Use when model traffic centralization is acceptable. The proxy becomes a sensitive routing/trust boundary and may see full payloads. Confirm outage behavior, TLS/auth, logging defaults, retention, regional/privacy requirements, bypass path, and self-hosted feature limits before adoption.

## Minimum span model

[`assets/event-schema.yaml`](assets/event-schema.yaml) defines a vendor-neutral starting record. Recommended fields:

- trace/span IDs, parent, timestamps, status;
- operation type (`model`, `tool`, `retrieval`, `agent-step`);
- provider/model identifier as observed, not guessed;
- token/cost/latency only when supplied or measured with provenance;
- tool/retrieval target identifiers with sensitive arguments excluded by default;
- prompt/input/output represented as **off by default**, redacted preview, or approved secure reference;
- error class and retry relation;
- schema/semantic-convention version and instrumentation version.

Do not log API keys, authorization headers, session cookies, raw credentials, or production secrets. Hashing low-entropy personal/sensitive values may still leak them.

## Redaction and retention checklist

Before enabling capture:

1. inventory data classes at every span attribute/event;
2. default-deny prompt/output/tool arguments; allowlist necessary fields;
3. redact at the earliest controlled boundary, ideally before network export;
4. separate tenant/user access and verify authorization with synthetic records;
5. encrypt transport/storage and define key/access owners;
6. set retention by data class, deletion process, backup behavior, and legal policy;
7. sample intentionally; never assume “debug mode” is safe in production;
8. test failure mode: collector/backend unavailable must not expose payloads or block critical work unexpectedly;
9. verify exported data with fake canaries; then delete the test trace;
10. record self-hosted versus hosted components and license/version explicitly.

## Project placement and evidence limits

| Layer | Project/source | Narrow role | Verify before use |
|---|---|---|---|
| Telemetry contract | [OpenTelemetry specification](https://github.com/open-telemetry/opentelemetry-specification) | General traces/metrics/logs/context/OTLP concepts | exact spec/SDK/collector versions |
| GenAI semantics | [OpenTelemetry GenAI conventions](https://opentelemetry.io/docs/specs/semconv/gen-ai/) and [OpenInference](https://github.com/Arize-ai/openinference) | Attribute/span conventions for AI operations | stability/version and backend mapping |
| Instrumentation | [OpenLLMetry](https://github.com/traceloop/openllmetry) | OTel-oriented LLM framework/provider instrumentation | integrations and default captured data |
| AI backend | [Langfuse](https://github.com/langfuse/langfuse), [Phoenix](https://github.com/Arize-ai/phoenix), [OpenLIT](https://github.com/openlit/openlit) | Tracing/evaluation/analysis platforms with different scopes | deployment/license, ingestion, retention, feature parity |
| Proxy boundary | [Helicone](https://github.com/Helicone/helicone) and [gateway docs](https://docs.helicone.ai/getting-started/integration-method/gateway) | Gateway/request observability path | payload routing, outage/privacy, self-hosted limits |
| General OTel backend | [SigNoz](https://github.com/SigNoz/signoz) | OTel-native traces/metrics/logs backend | AI dashboards/schema work required |

The linked pages support these broad roles. OTLP interoperability, capture defaults, hosted/self-hosted parity, licenses, and retention must be checked against specific documentation/version before claiming them. The two architectures, minimum schema, and privacy checklist are **author recommendations in this guide**, not vendor guarantees.

## Selection procedure

1. Classify data and choose OTel-first or proxy-first boundary.
2. Implement the minimum schema with synthetic values.
3. Put redaction/filtering before the first untrusted or external hop.
4. Send traces to two candidate backends only if policy allows; verify field loss/mapping rather than comparing screenshots.
5. Run four failure queries: failed tool call, retry chain, slow retrieval, and unauthorized-data canary.
6. Test collector/backend outage and deletion/retention.
7. Choose based on required query, privacy, operational cost, and recoverability—not feature count.

## Useful failure queries

- Which tool error caused the final failed run, and was it retried?
- Which retrieval span contributed context to a wrong answer?
- Did a model/provider alias change between baseline and regression?
- Did any canary/sensitive attribute cross the redaction boundary?
- Are missing spans instrumentation loss, sampling, or an actual skipped step?

Missing telemetry is unknown, not proof the action did not happen.

## Maintenance

Version the schema and semantic-convention mapping. Re-run canary, outage, access, and deletion tests after instrumentation/backend upgrades. Change retention only through reviewed policy, never as a silent UI default.

Sources: [`sources/sources.json`](sources/sources.json).

## Adjacent guides

- [agent-execution-control-sandbox](https://github.com/MargaritaOfficial/agent-execution-control-sandbox) — Use it when traces reveal that the execution boundary itself needs stronger isolation and lifecycle control.
- [llm-test-determinism-guide](https://github.com/Simmondsun/llm-test-determinism-guide) — Use it when observed behavior must be converted into stable CI regression checks.
- [code-rag-chunking-benchmark](https://github.com/Mustafa1801/code-rag-chunking-benchmark) — Use it when retrieval spans need a controlled benchmark rather than ad hoc production anecdotes.

## License

Original architecture, checklist, and event schema are available under CC BY 4.0. Linked projects/specifications retain their terms.

If this saved you a decision, star the repo — it is how the next reader finds it.
