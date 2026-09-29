# ADR-002: GenAI observability stack for the Tiltmeter agent system

- Status: Accepted
- Date: 2026-09-27

## Context

`tiltmeter-memory-agent` and other kagent agents produce two kinds of
signals that require different tools to observe:

**Operational signals** — the span tree of a single invocation (AGENT →
LLM → TOOL), latency, token counts per LLM call, MCP tool errors. Used
for live diagnosis: "why did this request take 35s?", "where did
`hybrid_find` fail?".

**Eval signals** — quality comparison across agent configurations,
LLM-judge scoring (faithfulness/correctness/coherence), regression
between runs. Used during development: "did the new prompt improve
answers?", "which model is cheaper at the same quality?".

Three candidates were evaluated on the live cluster (`abox`, kagent 0.10.1):

1. **Standard OTel** — OTLP gRPC export from the kagent ADK to any
   OTLP-compatible backend. kagent is already instrumented out of the box
   via `OTEL_TRACING_ENABLED`.
2. **Arize Phoenix** — LLM-specific observability backend that receives
   the same OTLP spans but adds a GenAI-first UI (token distribution,
   latency heatmap, eval runner). Already deployed in `abox`
   (`releases/phoenix.yaml`).
3. **MLflow Tracing** — experiment tracking and eval platform. Receives
   traces through its Python SDK (`mlflow.openai.autolog()` or
   `@mlflow.trace`). Not deployed in-cluster — run locally or as a
   separate service.

Key observations:
- kagent ADK emits GenAI semantic conventions (OTel 1.27+) without any
  agent code changes: `gen_ai.request.model`,
  `gen_ai.usage.input_tokens/output_tokens`, `gen_ai.operation.name`,
  span kinds AGENT/LLM/TOOL — enabled by `OTEL_TRACING_ENABLED=true` in
  the ConfigMap alone.
- MLflow only sees tokens and prompt text when the OpenAI SDK is called
  inside the same Python process as `mlflow.openai.autolog()`. Calling
  the agent over A2A (HTTP JSON-RPC) gives MLflow only the outer HTTP
  hop — tokens = 0, span tree = 1 level deep.

## Decision

### 1. Phoenix as the primary GenAI observability backend

Phoenix receives OTLP spans directly from each agent pod via
`OTEL_EXPORTER_OTLP_ENDPOINT`. It is the only tool that sees the full
span tree inside an agent invocation — AGENT → LLM → TOOL → LLM —
together with token counts per LLM call, with no changes to agent code.

An alternative backend (Grafana Tempo) remains an option for operational
monitoring of Layer 1 (economy-sim services), using the same OTLP
endpoint with a different `service.namespace` (`economy-sim` vs `abox`).

### 2. MLflow for eval runs, not for live span collection

MLflow is run locally during eval sessions. The eval script (`cmd/eval`
or `mlflow-eval/eval.py`) calls the agent over A2A, wraps each question
in `@mlflow.trace(SpanType.AGENT)`, logs per-question metrics (latency,
faithfulness, correctness), and saves full answers as artifacts.

MLflow **does not replace** Phoenix for live tracing — it supplements it
for experiment comparison: "run A (Claude Haiku) vs run B (Claude Sonnet)".

### 3. Standard OTel — a protocol, not a separate tool

"Standard OTel" is the OTLP protocol and GenAI semantic conventions — the
shared foundation under both Phoenix and any other backend. A standalone
OTel Collector is not added to the cluster: Phoenix accepts OTLP gRPC
directly on :4317, which is sufficient for a single-cluster setup.

### 4. No additional LLM-observability SDKs on top of kagent

`mlflow.openai.autolog()`, OpenLLMetry/Traceloop, and Langfuse SDK are
not added to kagent agent code. Reasons:
- kagent ADK already emits GenAI semconv — wrapping the OpenAI SDK a
  second time produces duplicate spans.
- Double-wrapping the OpenAI SDK breaks streaming (confirmed in
  tiltmeter-plan.html).
- MLflow autolog is only useful in the Python eval harness where we
  control the SDK calls directly.

## Comparison

|                     | Standard OTel    | MLflow            | Phoenix             |
|---------------------|------------------|-------------------|---------------------|
| Span tree           | ✅               | ❌ (A2A boundary) | ✅                  |
| Token counts        | ✅               | ❌ via A2A        | ✅                  |
| Experiment tracking | ❌               | ✅                | ❌                  |
| LLM eval runner     | ❌               | ✅                | ✅                  |
| In-cluster          | needs collector  | locally           | ✅ already in abox  |

## Consequences

**Positive**

- Zero code overhead in agents: `OTEL_TRACING_ENABLED=true` in the
  ConfigMap, nothing else.
- Phoenix provides token distribution and latency heatmap across all
  agent requests without additional instrumentation.
- MLflow produces reproducible eval runs: questions, answers, scores and
  metrics are stored together in one experiment and comparable across runs.
- The two tools do not conflict: Phoenix receives OTLP from agent pods,
  MLflow receives traces from the eval script.

**Negative / accepted as open**

- Phoenix does not show prompt/response text in the span table
  (`input`/`output` = `--`) because kagent ADK records `gen_ai.usage.*`
  but not `llm.input_messages` / `llm.output_messages` in the Arize
  format. Fix requires either a patch to the ADK or a custom Phoenix span
  processor.
- Root spans in Phoenix show `kind: unknown`, `name: POST /` — the HTTP
  layer, not the agent layer. The real span tree is only visible after
  clicking into a trace. The `parent_id is None` filter makes the list
  look empty at first glance.
- MLflow tokens = 0 in A2A mode — accepted: the eval harness measures
  latency and LLM-judge scores, not raw token counts. For tokens, Phoenix
  is the source.
- Flux reconcile overwrites ConfigMap patches on the next helm upgrade.
  For persistence, the `OTEL_*` overrides should be moved into
  `releases/kagent.yaml` as HelmRelease values.
