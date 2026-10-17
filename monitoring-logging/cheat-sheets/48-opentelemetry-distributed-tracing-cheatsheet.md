# OpenTelemetry & Distributed Tracing Cheat Sheet

*One-page dense reference — concepts, OTLP, Collector, sampling, debugging.*

## Core vocabulary

```
Trace    = one request's full journey (tree of spans)
Span     = one unit of work: name, start/end, attributes, events, status
Trace ID = 128-bit ID shared by all spans in a trace
Span ID  = 64-bit ID of one span; child spans store parent span ID
Context  = trace_id + span_id + flags, propagated via W3C traceparent header
Exemplar = link from a metric data point back to a representative trace
```

## OTLP endpoints (default)

```
OTLP/gRPC: http://collector:4317     # preferred, binary protobuf
OTLP/HTTP: http://collector:4318     # JSON or protobuf
```

## SDK env vars (zero-config instrumentation)

```bash
OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317
OTEL_SERVICE_NAME=checkout-api            # always set this
OTEL_TRACES_SAMPLER=parentbased_traceidratio
OTEL_TRACES_SAMPLER_ARG=0.05              # keep 5%
OTEL_RESOURCE_ATTRIBUTES=deployment.environment=prod,team=payments
OTEL_PROPAGATORS=tracecontext,baggage     # W3C standard
```

## Python quickstart

```bash
pip install opentelemetry-distro
opentelemetry-bootstrap -a install
python -m opentelemetry.instrumentation.auto_instrumentation \
  --traces_exporter otlp app.py
```

```python
# manual span
from opentelemetry import trace
tracer = trace.get_tracer(__name__)
with tracer.start_as_current_span("db.query") as span:
    span.set_attribute("db.table", "orders")   # bounded only
    span.add_event("query.slow", {"rows": 12000})
```

## Collector pipeline skeleton

```yaml
receivers:  { otlp: { protocols: { grpc: {}, http: {} } } }
processors: [ memory_limiter, tail_sampling, batch ]
exporters:  { otlp/tempo: { endpoint: tempo:4317, tls: { insecure: true } } }
service:
  pipelines:
    traces: { receivers: [otlp], processors: [memory_limiter, tail_sampling, batch], exporters: [otlp/tempo] }
```

**Processor order matters:** memory_limiter → tail_sampling → batch (sample before you batch).

## Sampling cheat table

| Type | Decides | Cost | Catches errors? |
|------|---------|------|-----------------|
| Head-based (ratio 5%) | at request start | cheap, spans never sent | no — blind |
| Tail-based (Collector) | after full trace | buffers traces | yes — keep errors + slow |

```yaml
# tail_sampling policies worth copying
processors:
  tail_sampling:
    policies:
      - { name: errors, type: status_code, status_code: { status_codes: [ERROR] } }
      - { name: slow,   type: latency,     latency: { threshold_ms: 500 } }
```

## Span attributes: do / don't

```
DO:    service.version, http.route, error.type, warehouse, retry.count
DON'T: user_id, email, request_id, raw SQL, request/response bodies
```
**Rule:** bounded + filterable = good; unbounded or PII = cardinality explosion and a privacy incident.

## Jaeger UI debugging flow

```
1. alert/metric -> grab trace ID (exemplar or log line)
2. Jaeger -> Find Traces -> paste trace ID
3. waterfall: longest child span = prime suspect
4. span attributes: query, table, endpoint, error
5. cross-check logs at that timestamp + service metrics
```

## Correlate logs fast

```python
from opentelemetry.instrumentation.logging import LoggingInstrumentor
LoggingInstrumentor().enable(set_logging_format=True)
# every log line now carries otelTraceID / otelSpanID
```

## Common failure modes

```
Orphan traces (one request -> many traces) = context not propagated through threads/queues
Missing spans = head sampler dropped them, or exporter failing silently
Collector OOM = no memory_limiter processor, or tail_sampling buffer too big
Trace ID not in logs = LoggingInstrumentor not enabled on that service
```
