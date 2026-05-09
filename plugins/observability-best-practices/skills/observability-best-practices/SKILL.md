---
name: observability-best-practices
description: Python observability best practices for async-first services using OpenTelemetry — covers SDK bootstrap with `setup_telemetry(component)` and an `OTEL_EXPORTER_OTLP_ENDPOINT` off-switch contract, single-shot idempotency guard, per-component `service.name` resource attribute, OTLP HTTP/protobuf exporters with `BatchSpanProcessor` / `BatchLogRecordProcessor` / `PeriodicExportingMetricReader`, auto-instrumentation (`GrpcAioInstrumentorServer/Client`, `SQLAlchemyInstrumentor`, `FastAPIInstrumentor`, `LoggingInstrumentor`), the critical `set_logging_format=False` + custom `_inject_trace_context` log hook + `_CleanLoggingHandler` pattern that bridges OTel SDK and structlog, two-tier log field taxonomy (Tier-1 structured `key=value` indexed in Loki + Tier-2 raw payload after `|` separator), `<service>.<business_key>` span attribute prefix for cross-service Tempo search, sensitive-field redaction via copy-and-replace at the proto/JSON serialization boundary, status taxonomy mapping, metrics catalogue conventions with strict cardinality control (no `transaction_id` / `user_id` labels), and tail-based sampling design. Use this skill whenever the user is writing, reviewing, debugging, or designing OpenTelemetry / observability code in a Python backend — including any work involving imports from `opentelemetry` / `opentelemetry.instrumentation.*` / `opentelemetry.sdk.*`, `setup_telemetry()` calls, `tracer.start_as_current_span()`, `meter.create_counter()` / `create_histogram()`, `LoggingInstrumentor`, structlog with trace_id binding, span attribute setting, OTLP exporter configuration, Loki / Tempo / Grafana / Mimir integration, distributed tracing across gRPC services, log correlation via `trace_id` / `span_id`, sensitive-field redaction in logs, metric cardinality decisions, alert rule design, or Sentry-with-OTel integration. Trigger on any review of `telemetry.py`, `*_logger.py`, log-decorator modules, or Grafana dashboard / Prometheus alert YAML in a Python project. Do NOT use for proprietary observability SDKs without OTel (raw New Relic / Datadog / Honeycomb agents), browser/frontend tracing, log analysis questions unrelated to instrumentation, or generic "what's a span" intro questions.
---

# Observability Best Practices (Python OTel-First, Async-Native)

This skill captures production-tested patterns for OpenTelemetry instrumentation in async-first Python services deployed to Kubernetes. The patterns are battle-tested across multi-service domains where traces span gRPC hops between services and log correlation via `trace_id` is non-negotiable.

User-emphasized depth: **OTel SDK bootstrap, log correlation (the tricky part), span attributes, metrics cardinality, sensitive-field redaction**. Custom metrics catalogue and dashboard provisioning are typically project-specific — this skill establishes naming/conventions but defers detail to the project's own observability doc.

## Stack

- `opentelemetry-api` + `opentelemetry-sdk` (the core)
- `opentelemetry-exporter-otlp-proto-http` — OTLP HTTP/protobuf transport (preferred over gRPC transport for simpler firewall traversal in cloud-managed observability)
- `opentelemetry-instrumentation-grpc` — async server + client interceptors (`GrpcAioInstrumentorServer`, `GrpcAioInstrumentorClient`)
- `opentelemetry-instrumentation-sqlalchemy` — DB query spans
- `opentelemetry-instrumentation-fastapi` — HTTP server spans (apply only where applicable)
- `opentelemetry-instrumentation-logging` — `trace_id` / `span_id` injection on every `LogRecord`
- `structlog` — structured logging that coexists with OTel
- `sentry-sdk` with `instrumenter="otel"` — exception capture with trace context

## Three pillars + correlation

The model is the standard OTel three-pillar architecture, with **`trace_id` as the cross-pillar correlation key**:

- **Metrics** (Prometheus / Mimir) — counters, histograms, gauges; bounded cardinality; aggregable.
- **Traces** (Tempo) — span trees across service boundaries via W3C `traceparent` propagation; high cardinality, sampled.
- **Logs** (Loki) — structured key=value fields; every log line carries `trace_id` so a metric exemplar or trace deep-link round-trips into log search.

The investigation flow: dashboard alert → exemplar trace → logs filtered by `trace_id`. Anything that breaks that chain (logs without `trace_id`, spans without business attributes, metrics with unbounded labels) is a footgun.

## OTel SDK bootstrap

The canonical setup is a single `setup_telemetry(component)` function called as **the very first action** in every entrypoint script:

```python
import logging
import os

from opentelemetry import metrics, trace
from opentelemetry._logs import set_logger_provider
from opentelemetry.exporter.otlp.proto.http._log_exporter import OTLPLogExporter
from opentelemetry.exporter.otlp.proto.http.metric_exporter import OTLPMetricExporter
from opentelemetry.exporter.otlp.proto.http.trace_exporter import OTLPSpanExporter
from opentelemetry.instrumentation.grpc import GrpcAioInstrumentorServer, GrpcAioInstrumentorClient
from opentelemetry.instrumentation.logging import LoggingInstrumentor
from opentelemetry.instrumentation.sqlalchemy import SQLAlchemyInstrumentor
from opentelemetry.sdk._logs import LoggerProvider, LoggingHandler
from opentelemetry.sdk._logs.export import BatchLogRecordProcessor
from opentelemetry.sdk.metrics import MeterProvider
from opentelemetry.sdk.metrics.export import PeriodicExportingMetricReader
from opentelemetry.sdk.resources import Resource
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor


_METRIC_EXPORT_INTERVAL_MS = 30_000
_initialized: bool = False  # idempotency sentinel


def setup_telemetry(component: str) -> None:
    """Initialise OTel tracing, metrics, and logs for one process.

    Off-switch: if OTEL_EXPORTER_OTLP_ENDPOINT is unset, return immediately
    with zero side effects. Idempotent: second call is a no-op.
    """
    global _initialized
    if not os.getenv("OTEL_EXPORTER_OTLP_ENDPOINT"):
        return
    if _initialized:
        return
    _initialized = True

    resource = Resource.create({"service.name": f"myservice-{component}"})

    # Traces
    tracer_provider = TracerProvider(resource=resource)
    tracer_provider.add_span_processor(BatchSpanProcessor(OTLPSpanExporter()))
    trace.set_tracer_provider(tracer_provider)

    # Metrics
    metric_reader = PeriodicExportingMetricReader(
        OTLPMetricExporter(),
        export_interval_millis=_METRIC_EXPORT_INTERVAL_MS,
    )
    metrics.set_meter_provider(MeterProvider(resource=resource, metric_readers=[metric_reader]))

    # Logs (OTel Logging Bridge)
    logger_provider = LoggerProvider(resource=resource)
    logger_provider.add_log_record_processor(BatchLogRecordProcessor(OTLPLogExporter()))
    set_logger_provider(logger_provider)
    logging.getLogger().addHandler(_CleanLoggingHandler(level=logging.INFO, logger_provider=logger_provider))

    LoggingInstrumentor().instrument(set_logging_format=False, log_hook=_inject_trace_context)

    GrpcAioInstrumentorServer().instrument()
    GrpcAioInstrumentorClient().instrument()
    SQLAlchemyInstrumentor().instrument()
    if component == "http":
        from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor
        FastAPIInstrumentor().instrument()
```

### Off-switch contract — non-negotiable

1. The function reads `OTEL_EXPORTER_OTLP_ENDPOINT` from the environment; if unset, returns immediately with **no side effects** — no instrumentor patches, no background threads, no log records emitted.
   - **Why:** tests need determinism. CI doesn't have an OTLP endpoint, and tests should not produce telemetry noise or spawn exporter threads. The off-switch makes test runs identical regardless of whether telemetry is wired in.

### Idempotency guard

2. The `_initialized` module-level sentinel guards against double-initialisation (uvicorn `--reload`, accidental double-call from compound entrypoints, dynamic backoffice reloads).
   - **Why:** `logging.getLogger().addHandler` *stacks* — every reload would ship every log record N+1 times to OTLP. The OTel SDK warns-but-overwrites on duplicate `set_*_provider`, but the logging handler chain is the silent failure path.

### `service.name` per component, `service.name` family for log correlation

3. Resource attribute `service.name` includes the component (`myservice-grpc`, `myservice-worker`, `myservice-http`) so each Grafana Cloud target / Mimir series can be filtered independently.
4. The log-injected `otelServiceName` field stays at the **family** name (`myservice`) — Loki correlates by service family, not per-component, because cross-component log searches are common.

## Log correlation — the tricky part

The interaction between OTel `LoggingInstrumentor` and `structlog` has three footguns that all need to be solved together. Each one alone produces broken output.

### `set_logging_format=False`

5. Always pass `set_logging_format=False` to `LoggingInstrumentor().instrument()`.
   - **Why:** without it, the instrumentor rewrites the log format string globally, breaking any pre-existing structured logging setup (logs_lib / structlog stdout formatting). The visible symptom is double-formatted JSON-in-JSON or trace_id appearing as plain text smashed into the message body.

### Custom `log_hook` to inject trace context

6. With `set_logging_format=False`, the instrumentor **does NOT** auto-inject `otelTraceID` / `otelSpanID` on `LogRecord`. Bridge with a manual hook:

   ```python
   def _inject_trace_context(span: trace.Span, record: logging.LogRecord) -> None:
       ctx = span.get_span_context()
       if ctx and ctx.trace_id != 0:
           record.otelTraceID = format(ctx.trace_id, "032x")
           record.otelSpanID = format(ctx.span_id, "016x")
           record.otelTraceSampled = ctx.trace_flags.sampled
           record.otelServiceName = "myservice"
       else:
           record.otelTraceID = "0"
           record.otelSpanID = "0"
           record.otelTraceSampled = False
           record.otelServiceName = ""
   ```
   - **Why:** structlog records that fire outside an active span need the same field shape as in-span records (Loki LogQL queries expect a stable schema). Setting empty / zero values on out-of-span records prevents `__error__=field_missing` parsing failures.

### `_CleanLoggingHandler` for structlog non-serializable attrs

7. structlog injects `_logger` and `_name` attributes on `LogRecord` (bound-logger references). These are not OTLP-serializable. Strip them before export with a custom handler subclass:

   ```python
   _SKIP_LOG_ATTRIBUTES = frozenset({"_logger", "_name"})

   class _CleanLoggingHandler(LoggingHandler):
       def emit(self, record: logging.LogRecord) -> None:
           for attr in _SKIP_LOG_ATTRIBUTES:
               if hasattr(record, attr):
                   delattr(record, attr)
           super().emit(record)
   ```
   - **Why:** the OTel `LoggingHandler` walks all record attributes for the OTLP body. Without stripping, you get serialization warnings every log line and bound-logger references leak into Loki as garbage.

## Two-tier log field taxonomy

The structured log shape that works well in Loki:

- **Tier-1**: `key='value'` pairs, parsed by `| logfmt`, indexed for filtering and aggregation. Approximately **15-20 fields per log line**.
- **Tier-2**: raw request/response payload after a `|` separator in `msg`. Not separately indexed; available for grep when investigating a specific case.

### Tier-1 universal fields (every log line)

| Field | Source |
|---|---|
| `ts` (ISO 8601) | structlog `TimeStamper` |
| `level` | structlog `add_log_level` |
| `msg` | the logger call |
| `trace_id` / `span_id` | OTel hook (#6) |
| `component` | logs setup (e.g. `myservice-grpc`) |
| `env` | logs setup (`prod` / `staging` / `dev`) |

### Tier-1 per-RPC fields (extracted via context extractor — see decorator pattern)

Per-RPC, ~5-10 business-context fields: `method`, primary entity IDs (`request_id`, `user_id`, `order_id`, etc.), `direction` (if applicable), `current_state`, `prev_state`, `idempotency_key` (redacted), `correlation_id`, `duration_ms` on response lines.

### Tier-1 outcome / classification fields

`status` (canonical enum across all logs), `reason_code` (namespaced like `validation.amount_zero`, `psp.insufficient_funds`), `actor_type` / `actor_id` (when applicable).

### Status taxonomy

A canonical `status` field with mapped log levels:

| `status` | Log level | Semantic |
|---|---|---|
| `ok`, `created`, `approved`, `settled` | `info` | Successful path |
| `rejected`, `failed`, `transient` | `warning` | Expected business failure |
| `internal_error` | `error` | Unhandled exception — Sentry alert |
| `replayed` | `info` | Idempotent return for same request |

Alerts fire on **rates**, never on single occurrences. Expected business failures (`rejected`, `failed`) at warning level shouldn't page; only **anomalous rates** of them should.

## Span attribute conventions

Span attributes carry business IDs for **cross-service Tempo search**:

```python
from opentelemetry import trace

span = trace.get_current_span()
if span.is_recording():
    span.set_attribute("myservice.request_id", str(request_id))
    span.set_attribute("myservice.user_id", str(user_id))
    span.set_attribute("myservice.direction", direction)
    span.set_attribute("myservice.current_state", current_state)
```

8. **Use a `<service>.<business_key>` prefix.** All services in the same domain use the same business-ID names (`myservice.request_id`, `risk.request_id`, `wallet.request_id` all reference the same business UUID), so a Tempo search with `myservice.request_id="..."` returns spans across the entire request lifecycle.
   - **Why:** without a stable prefix, attribute names collide (a `request_id` on one service vs `tx_id` on another), and cross-service searches become a manual join.
9. **Always guard with `span.is_recording()`** before setting attributes when telemetry might be off (no-op span). Otherwise you allocate strings that no one reads.
10. **Don't bake high-cardinality values into attributes that get used as metric dimensions.** Span attributes are fine at any cardinality (Tempo is search-not-aggregate); metric labels are not (see metrics section).

## Decorator pattern for cross-cutting concerns

The canonical pattern (covered in detail in the `grpc-python-best-practices` skill, see rule 30 there) is **decorators on servicer methods** rather than `ServerInterceptor`:

```python
@grpc_logger(extract_context=create_request_context)
@grpc_error_handler(ERROR_MAP)
async def DoSomething(self, request, context): ...
```

The logger decorator binds Tier-1 fields from a per-method `extract_context(request)` callable, emits two log lines (request on entry, response on exit), and sets the same fields as `<service>.<key>` span attributes. The error decorator runs innermost so business exceptions are mapped to gRPC status before the logger sees the response.

For HTTP services, the equivalent is FastAPI middleware that does the same field-extraction + bind pattern. Defer to the `fastapi-best-practices` skill for FastAPI-specific patterns.

## Sensitive-field redaction

Redaction happens at the **serialization boundary** (just before logging the proto/JSON), not in business code. Pattern: copy the message, replace sensitive fields, then format.

11. **Redact at the boundary.** Business logic operates on the original message; only the *log/trace* representation is redacted. This keeps the audit trail truthful while keeping logs PII-clean.
12. Common redaction modes:

    | Mode | Input | Output |
    |---|---|---|
    | `redact_full` | `"DE89370400440532013000"` | `"***"` |
    | `redact_keep_last_4` | `"DE89370400440532013000"` | `"***3000"` |
    | `redact_email` | `"alice@example.com"` | `"a***@example.com"` |

    `redact_keep_last_4` preserves correlation potential (operators can match against PSP-side logs that show last 4) without leaking the full value.

13. **What's NOT sensitive** (and should NOT be redacted):
    - Internal UUIDs (`request_id`, `user_id`, `order_id`) — these are the audit data; redacting them defeats the purpose of structured logs.
    - Domain values (amounts, currency codes, status enums) — domain data, not secrets.
    - Provider-side identifiers from upstream systems — needed for cross-system reconciliation.

    The litmus test: "if I redact this, can I still answer 'why did this request fail?' from logs alone?" If no, don't redact.

## Where logging happens — RPC boundary, not business logic

14. Logs go in **RPC handlers and entrypoint code** — `apps/<server>/handlers`, FastAPI route handlers, message-bus consumer entrypoints. **Not** in business services (`services/`, `domain/`, `repository/`, `workflows/`).
    - **Why:** logging in business code creates duplicate lines (handler logs the call + service logs internally), couples domain code to log format, and pollutes refactoring (every business change touches logging).
    - The cross-cutting decorator at the RPC boundary captures `request → response` automatically; the business code stays clean.

## Metrics — naming and cardinality

15. **Naming convention**: `<service>_<verb>_<noun>` snake_case, histograms suffixed `_seconds`. Counters end in `_total`. Gauges have no suffix. Examples: `myservice_requests_created_total`, `myservice_validation_duration_seconds`, `myservice_active_subscriptions`.
16. **Two collection points**:
    - **gRPC interceptor / FastAPI middleware** — infrastructure metrics (call latency, status codes, request rate). Auto-instrumentation handles most of this.
    - **Business code** — domain metrics (state transition counters, gate timings, recon match rate). Explicit `meter.create_counter()` / `create_histogram()` calls.
17. **Cardinality control — strict.** Never use unbounded values as metric labels:

    | Forbidden as metric label | Reason |
    |---|---|
    | `request_id`, `transaction_id`, etc. | One series per request — explodes Prometheus |
    | `user_id`, `account_id` | Series per user — millions+ of series |
    | `idempotency_key`, `correlation_id` | Per-call uniqueness destroys aggregation |
    | Free-form error messages | Stack traces, varying texts |

    These dimensions belong in **logs and traces**, not metrics. Use bounded enums for label values: `direction` (~2), `state` (~10-15), `actor_type` (~5), `outcome` (~5-10).

18. **Estimate cardinality before adding metrics.** A counter with 5 labels of cardinality 5/15/2/10/30 = 22.5k unique series — manageable. A counter with one wrong label of cardinality 1M = pages-the-on-call. Document the cardinality math in the project's observability doc when introducing new metric labels.

## Sampling

19. **Tail-based sampling** in production: 100% errors + 100% slow + N% normal (typically 1-10%). Implemented in the OTel Collector, not in the SDK — head-based sampling at the SDK loses errors that happen in low-sampled normal traffic.
20. In dev / PoC phase before a Collector is deployed, head-based 100% is fine; volume is low. Plan the Collector deployment as part of "production observability" milestone, not as a footnote.

## Sentry integration

21. When using both OTel and Sentry, configure Sentry to use OTel-managed spans rather than its own tracer:

    ```python
    import sentry_sdk
    sentry_sdk.init(
        dsn=settings.sentry_dsn,
        environment=settings.environment,
        traces_sample_rate=0.1,
        instrumenter="otel",  # critical: use OTel-managed spans
    )
    ```
    - **Why:** without `instrumenter="otel"`, Sentry creates a parallel span tree, breaking trace continuity across services. The trace_id Sentry exposes won't match the trace_id in Loki / Tempo.
22. **Tag Sentry events with business context** — same fields as Tier-1 log fields (`direction`, `current_state`, `reason_code`). This makes Sentry searchable by the same dimensions ops uses elsewhere.

## What this skill defers to project-specific docs

- **Custom metrics catalogue** — the project should maintain a per-service observability doc listing every counter/histogram/gauge with labels, semantics, and cardinality math. The conventions above (#15-18) are the *grammar*; the catalogue is the *vocabulary*.
- **Alert rule definitions** — specific thresholds (`>0.1 failure rate for 5m`) are deployment-specific.
- **Dashboard JSON** — Grafana dashboards live as provisioned YAML/JSON in the project, not in this skill. The conventions above (Tier-1 field shape, span attribute prefix) make dashboards portable; the dashboards themselves are project artifacts.
- **Sensitive-field set** — the *modes* (`redact_keep_last_4`, etc.) are reusable; the field list is project-specific.

## When applying these rules

- **Be opinionated about footguns** (#1 off-switch contract, #2 idempotency guard, #5 `set_logging_format=False`, #6 manual log hook for trace context, #7 `_CleanLoggingHandler`, #11 redact at boundary not in business code, #14 logs at RPC boundary, #17 metric label cardinality) — these cause silent observability failures, double-shipped log records, broken trace correlation, or runaway Prometheus cardinality bills.
- **Be flexible about conventions** (specific field names, status enum values, redaction sets) — match the project's established taxonomy. The conventions in this skill are the *shape*; the project owns the *content*.
- **Read existing telemetry first.** If the project has a `telemetry.py`, conform to its `setup_telemetry` shape rather than introducing parallel patterns. If it has a `_logger.py` decorator, use that. New services adopt the existing conventions; don't reinvent.
- **Cross-skill awareness:**
  - `grpc-python-best-practices` — for the `@grpc_logger` + `@grpc_error_handler` decorator chain that this skill's logging conventions slot into.
  - `fastapi-best-practices` — for FastAPI middleware and `FastAPIInstrumentor` placement.
  - `sqlalchemy-best-practices` — for `SQLAlchemyInstrumentor` and DB query span behaviour.
- **The hard part is correlation, not signal volume.** Anyone can emit metrics + traces + logs. The skill is in making them link via `trace_id` so an alert at the top of the funnel resolves to specific log lines at the bottom. Most of these rules exist to keep that chain unbroken.
