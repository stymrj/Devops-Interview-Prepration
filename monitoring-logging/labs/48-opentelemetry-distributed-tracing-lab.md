# OpenTelemetry & Distributed Tracing Hands-On Lab

*Instrument a real two-service app with OpenTelemetry, ship traces through the Collector, and query them — 60-90 minutes.*

**Prereqs:** Docker and Docker Compose. Everything runs locally; no cloud account needed.

---

## Exercise 1: Stand up the tracing stack

**Goal:** Get the OpenTelemetry Collector and Jaeger running with one command.

**Commands:**
```bash
mkdir otel-lab && cd otel-lab
cat > docker-compose.yml <<'EOF'
services:
  collector:
    image: otel/opentelemetry-collector-contrib:0.110.0
    ports: ["4317:4317", "4318:4318"]   # OTLP gRPC + HTTP
    volumes: ["./otel-collector.yaml:/etc/otelcol/config.yaml"]
    command: ["--config=/etc/otelcol/config.yaml"]
  jaeger:
    image: jaegertracing/all-in-one:1.62
    ports: ["16686:16686", "4317:4317"]
    environment:
      - COLLECTOR_OTLP_ENABLED=true
EOF
cat > otel-collector.yaml <<'EOF'
receivers:  { otlp: { protocols: { grpc: {}, http: {} } } }
processors: [ memory_limiter, batch ]
exporters:  { otlp/jaeger: { endpoint: jaeger:4317, tls: { insecure: true } } }
service:
  pipelines:
    traces: { receivers: [otlp], processors: [memory_limiter, batch], exporters: [otlp/jaeger] }
EOF
docker compose up -d && docker compose ps
```

**Expected output:** Both containers show `Up` status.

**Why it matters:** This mirrors the production pattern — apps export OTLP, the Collector batches and routes, Jaeger stores and queries.

---

## Exercise 2: Instrument a Flask app with auto-instrumentation

**Goal:** Get traces from a real HTTP app with zero code changes.

**Commands:**
```bash
pip install opentelemetry-distro flask requests
opentelemetry-bootstrap -a install
cat > app.py <<'EOF'
from flask import Flask, request
import requests, time, random
app = Flask(__name__)

@app.get("/checkout")
def checkout():
    time.sleep(random.uniform(0.1, 0.4))            # fake inventory check
    r = requests.get("http://localhost:5001/price") # downstream call
    return {"total": r.json()["price"] + 5.0}

app.run(port=5000)
EOF
```

**Expected output:** `pip` installs succeed; `opentelemetry-bootstrap` reports installed instrumentations.

**Why it matters:** Auto-instrumentation is how most teams start — framework plumbing traced for free.

---

## Exercise 3: Add a second service to trace across

**Goal:** Create a downstream service so traces span two services.

**Commands:**
```bash
cat > price.py <<'EOF'
from flask import Flask, jsonify
import time, random
app = Flask(__name__)

@app.get("/price")
def price():
    time.sleep(random.uniform(0.2, 0.8))  # fake DB lookup
    return jsonify(price=round(random.uniform(10, 100), 2))

app.run(port=5001)
EOF
```

**Expected output:** File created, no errors.

**Why it matters:** A single-service trace is a toy. Real value starts when one request crosses service boundaries.

---

## Exercise 4: Run both apps with OTel and hit the endpoint

**Goal:** Generate end-to-end traces through both services.

**Commands:**
```bash
export OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4317
export OTEL_SERVICE_NAME=checkout-api
python price.py &                                    # terminal 1 (downstream)
OTEL_SERVICE_NAME=checkout-api python -m opentelemetry.instrumentation.auto_instrumentation \
  --traces_exporter otlp --metrics_exporter none app.py &   # terminal 2
sleep 3
curl -s localhost:5000/checkout && echo
```

**Expected output:** `{"total": 42.37}`-style JSON response.

**Why it matters:** This is the moment telemetry becomes real — every request now emits spans to the Collector.

---

## Exercise 5: Find your trace in Jaeger

**Goal:** Verify the full trace arrived and inspect the waterfall.

**Commands:**
```bash
curl -s 'http://localhost:16686' -o /dev/null -w '%{http_code}\n'   # Jaeger UI up?
# open http://localhost:16686 in a browser -> select service -> Find Traces
```

**Expected output:** HTTP 200; the Jaeger UI shows traces for `checkout-api` with 2+ spans (`GET /checkout` parent, `GET /price` child).

**Why it matters:** The waterfall is the core debugging surface — slowest child span = first suspect.

---

## Exercise 6: Add a manual span with attributes

**Goal:** Trace business logic, not just framework plumbing.

**Commands:**
```bash
cat > app_manual.py <<'EOF'
from flask import Flask
from opentelemetry import trace
import requests, time, random
app = Flask(__name__)
tracer = trace.get_tracer(__name__)

@app.get("/checkout")
def checkout():
    with tracer.start_as_current_span("inventory.reserve") as span:
        span.set_attribute("warehouse", "bom-2")     # bounded attribute
        span.add_event("stock.confirmed")
        time.sleep(random.uniform(0.1, 0.3))
    return {"total": requests.get("http://localhost:5001/price").json()["price"]}
app.run(port=5000)
EOF
```

**Expected output:** New trace in Jaeger shows a custom `inventory.reserve` span nested under `GET /checkout`.

**Why it matters:** Manual spans make traces explain business behavior, not just HTTP timings.

---

## Exercise 7: Watch context propagation break (then fix it)

**Goal:** See what happens when context isn't propagated through a thread.

**Commands:**
```bash
# in app_manual.py, wrap the requests.get call in a ThreadPoolExecutor
# without passing context -> the /price span becomes a NEW trace (orphan)
# then fix it by propagating contextvars:
from opentelemetry.context import Context
```

**Expected output:** Broken version: Jaeger shows two separate traces for one request. Fixed version: one trace again.

**Why it matters:** Broken propagation is the #1 real-world tracing bug — recognizing orphan traces saves hours.

---

## Exercise 8: Turn on tail-based sampling in the Collector

**Goal:** Keep all errors and slow traces, drop the boring ones.

**Commands:**
```bash
# add to otel-collector.yaml processors:
#   tail_sampling:
#     policies:
#       - { name: errors,  type: status_code, status_code: { status_codes: [ERROR] } }
#       - { name: slow,    type: latency,     latency: { threshold_ms: 500 } }
# then add tail_sampling to the traces pipeline processors list
docker compose restart collector
```

**Expected output:** Collector restarts cleanly; Jaeger keeps error/slow traces while normal traffic is sampled down.

**Why it matters:** This is how production tracing stays affordable — 100% sampling is the classic cost mistake.

---

## Exercise 9: Correlate logs with traces

**Goal:** Make every log line carry its trace ID.

**Commands:**
```bash
pip install opentelemetry-instrumentation-logging
# in app.py:
import logging
from opentelemetry.instrumentation.logging import LoggingInstrumentor
LoggingInstrumentor().enable(set_logging_format=True)
logging.getLogger().warning("checkout started")
# check stdout: log line now includes otelTraceID=... otelSpanID=...
```

**Expected output:** Log lines print with `otelTraceID=<32 hex chars>` attached.

**Why it matters:** Trace IDs in logs are the cheapest, highest-ROI tracing practice — one lookup jumps from log to trace.

---

## Exercise 10: Clean up

**Goal:** Tear down the lab.

**Commands:**
```bash
pkill -f "python.*price.py"; pkill -f "opentelemetry.instrumentation"
docker compose down -v
cd .. && rm -rf otel-lab
```

**Expected output:** No containers running, lab directory removed.

**Why it matters:** Leave the machine clean — and you now know the full instrument → collect → query loop.
