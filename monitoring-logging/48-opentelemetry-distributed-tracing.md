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
