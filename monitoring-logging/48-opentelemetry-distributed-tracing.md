# OpenTelemetry & Distributed Tracing Interview Preparation Guide

*How to Answer Tracing Questions Confidently*

**Note for Students:** This guide is written exactly how you should answer in interviews. Practice reading these answers out loud to make them natural when speaking.

---

## Table of Contents

1. [Tracing Basics](#tracing-basics)
2. [OpenTelemetry Architecture](#opentelemetry-architecture)
3. [Instrumentation](#instrumentation)
4. [Sampling and Production Ops](#sampling-and-production-ops)

---

## Tracing Basics

### Q1: Why do we need distributed tracing — aren't logs and metrics enough?

**How to Answer:**

"Logs tell you what happened inside one service, metrics tell you something's wrong, but neither tells you how one slow user request flowed across ten services. A trace stitches the whole journey into one timeline, so I can see the exact hop that added the 800ms."

"In a monolith I could tail one log file. In microservices the same request jumps across pods, queues, and databases — tracing is the only way to see the full path. That's the gap it fills."

**Key Point:** "Tracing answers 'where did this request go and which hop was slow' — logs and metrics can't do that across services."

---

### Q2: What's the difference between a trace, a span, and a trace ID?

**How to Answer:**

"A trace is the full journey of one request end to end. A span is one unit of work inside it — an HTTP call, a DB query, a queue publish — each with a start time, duration, and attributes. The trace ID is the shared identifier glued to every span so the backend can reassemble them into one tree."

"Think of it like this: trace is the trip, spans are the legs of the trip, and the trace ID is the ticket number that ties them together."

**Key Point:** "Trace = whole request; span = one operation; trace ID = the glue that ties spans together."

---

### Q3: How does context propagation actually work across services?

**How to Answer:**

"The trace ID and span ID ride in HTTP headers — W3C Trace Context is the standard now, `traceparent` and `tracestate`. Service A creates a root span, injects the context into the outgoing request headers, and service B extracts it and creates a child span under the same trace ID."

"The tricky part is it has to be explicit. Async workers, Kafka consumers, thread pools — anywhere a new thread or message picks up work, you have to propagate the context manually or the trace breaks. Broken context is the number one reason traces look incomplete."

**Key Point:** "Context travels in headers via W3C Trace Context — and you have to propagate it explicitly wherever work hops threads or queues."

---

## OpenTelemetry Architecture

### Q4: What is OpenTelemetry, and what are its main parts?

**How to Answer:**

"OpenTelemetry is the CNCF standard for generating and collecting telemetry — traces, metrics, and logs — vendor-neutral, so I'm not locked into one backend. The main pieces are the API and SDK that instrument my code, the Collector that receives and processes telemetry, and exporters that ship it to backends like Jaeger, Tempo, or Datadog."

"The big win is I instrument once with OTel and can switch backends by changing Collector config, not rewriting code. That's the pitch I give teams deciding between vendor agents."

**Key Point:** "OTel = one vendor-neutral standard: instrument once with the SDK, route anywhere through the Collector."

---

### Q5: Where does the OpenTelemetry Collector fit — why not export directly from the app?

**How to Answer:**

"The Collector sits between my apps and the backend as a telemetry pipeline. My services export OTLP to it over gRPC, and it handles retries, batching, tail-based sampling, and routing — things I never want to rebuild in every service."

"I'd skip it for a tiny side project, but in production it gives me one choke point for config: change the backend or sampling policy in the Collector and every service follows. It also shields the backend from spikes with its queue and retry logic."

```yaml
# minimal collector pipeline
receivers:  { otlp: { protocols: { grpc: {} } } }
processors: [ batch, memory_limiter ]
exporters:  [ tempo ]
service:
  pipelines:
    traces: { receivers: [otlp], processors: [batch], exporters: [tempo] }
```

**Key Point:** "The Collector centralizes retries, batching, and sampling — one config change reaches every service."

---

### Q6: How do traces, metrics, and logs relate in OpenTelemetry?

**How to Answer:**

"They're three views of the same system. Traces show the request path, metrics show the aggregate health, logs show the detail. OTel ties them together with correlation — the trace ID and span ID get injected into logs, and exemplars link metrics back to traces."

"My real debugging flow: an alert fires on a metric, I find the slow trace behind it, and the trace points me to the exact log lines. Without correlation I'm jumping between three tools guessing. That's why I always wire trace IDs into structured logs first."

**Key Point:** "Metrics find the problem, traces locate it, logs explain it — correlation through trace IDs ties the three together."


---

## Instrumentation

### Q7: Auto-instrumentation versus manual instrumentation — when do you use each?

**How to Answer:**

"Auto-instrumentation is my starting point — it hooks into frameworks and libraries with zero code changes and captures the boring stuff: incoming HTTP requests, DB driver calls, outgoing RPCs. For Python or Java I just add the OTel agent and get traces immediately."

"But auto-instrumentation can't know what my code actually does. For business logic — a payment step, a cache fallback, a batch job stage — I add manual spans with meaningful names and attributes. My rule is auto for framework plumbing, manual for anything I'd want to see when debugging."

**Key Point:** "Auto-instrumentation covers framework plumbing for free; manual spans cover your business logic — use both."

---

### Q8: How do you add useful attributes to spans without making them noisy?

**How to Answer:**

"I add low-cardinality attributes that help me filter: service version, endpoint, tenant tier, error type. What I never add is high-cardinality stuff like user IDs, raw SQL with parameters, or request bodies — those explode storage and can leak PII."

"A span event is for a moment in time — 'cache miss, fell back to DB', 'retry attempt 2'. And I keep span names generic with attributes for the variables, so `GET /orders/:id` with `order_id` as an attribute, not a unique span name per order."

```python
# python - manual span, bounded attributes
with tracer.start_as_current_span("checkout") as span:
    span.set_attribute("payment.method", "upi")   # bounded: fine
    span.add_event("inventory.reserved")
    # never: span.set_attribute("user.email", email)  # PII, high cardinality
```

**Key Point:** "Attributes are for bounded, filterable facts — never IDs, PII, or unbounded values."

---

### Q9: How do you connect traces with your metrics and logs in practice?

**How to Answer:**

"Two moves. First, my logging library injects the current trace and span ID into every structured log line, so from a log I can jump straight to its trace. Second, in Grafana I set up the Tempo/Jaeger datasource so a span links to the Loki logs for that exact time window and trace ID."

"This is the part teams skip and then wonder why tracing feels useless. Correlation is what turns three disconnected tools into one debugging flow. I wire it in the Collector and logging config, not by hand per service."

**Key Point:** "Trace IDs in every log line plus linked datasources in Grafana turn three tools into one debugging flow."

---

## Sampling and Production Ops

### Q10: Head-based versus tail-based sampling — which do you pick?

**How to Answer:**

"Head-based sampling decides at the start of the request — keep 5% of all traces. It's cheap because dropped spans never leave the app, but it's blind: it can drop the one trace with the error. Tail-based sampling waits until the full trace arrives at the Collector and keeps everything interesting — errors, slow traces, high latency."

"I run tail-based in production with the Collector, keeping all errors and slow traces while dropping the boring successful ones. The cost is the Collector has to buffer and decide per trace, but that's exactly what it's built for. Head-based is fine for dev and staging."

**Key Point:** "Head-based is cheap but blind; tail-based at the Collector keeps every error and slow trace — that's the production pick."

---

### Q11: Walk me through debugging a slow request using a trace in production.

**How to Answer:**

"I start from the alert or the complaint and grab a trace ID — from the log line or the exemplars on the latency metric. In Jaeger or Tempo I open the trace and look at the waterfall: the longest child span is usually the culprit."

"Say the trace shows 2.1 seconds and the DB span is 1.9 of it. I check the span attributes — the query, the table, the rows scanned. Then I cross-check with the logs at that exact time and the DB's own metrics. The trace narrows a 'the app is slow' complaint to 'this specific query on this specific call path' in a couple of minutes."

**Key Point:** "Alert gives the trace ID, the waterfall finds the longest span, its attributes point at the exact query or hop."

---

### Q12: What are the common mistakes teams make when adopting OpenTelemetry?

**How to Answer:**

"First, instrumenting everything at 100% sampling and drowning in cardinality and cost — tracing bills are real. Second, treating OTel like a vendor agent and skipping the Collector, then hardcoding backends into every service. Third, spans with no attributes — a waterfall of anonymous spans tells you nothing."

"My rollout playbook: start with auto-instrumentation plus trace IDs in logs, put the Collector in the path, use tail-based sampling, and only then add manual spans around business logic. Keep the number of instrumented services small until the team actually uses the traces."

**Key Point:** "Biggest mistakes: 100% sampling, skipping the Collector, and spans with no attributes — fix all three before scaling up."
