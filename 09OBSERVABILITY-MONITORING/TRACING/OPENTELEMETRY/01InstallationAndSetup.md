# OpenTelemetry - Installation & Setup

Getting the OpenTelemetry SDK into your application and the Collector running, ending with your first exported trace.

## Table of Contents
- [The Two Things You Need](#the-two-things-you-need)
- [Installing the OpenTelemetry Collector](#installing-the-opentelemetry-collector)
- [Basic Collector Configuration](#basic-collector-configuration)
- [Instrumenting a Python Application](#instrumenting-a-python-application)
- [Instrumenting a Node.js Application](#instrumenting-a-nodejs-application)
- [Running Everything Together](#running-everything-together)
- [Verifying Your First Trace](#verifying-your-first-trace)
- [Real-Life DevOps Use Case](#real-life-devops-use-case)

---

## The Two Things You Need

**Visual:**
```
┌───────────────────────────────────────────┐
│  1. SDK (in your application)                     │
│     Generates the telemetry (spans, metrics, logs)   │
│                                                  │
│  2. Collector (a separate process)                    │
│     Receives, processes, and routes telemetry             │
│     to your chosen backend(s)                                │
└───────────────────────────────────────────┘

Both are needed for a complete setup, though the
SDK CAN technically export directly to a backend
without a Collector for very simple setups —
the Collector becomes valuable once you need
processing, multiple backends, or centralized control.
```

---

## Installing the OpenTelemetry Collector

### Docker

```bash
docker run -d --name otel-collector \
  -p 4317:4317 -p 4318:4318 \
  -v $(pwd)/otel-collector-config.yaml:/etc/otelcol/config.yaml \
  otel/opentelemetry-collector:latest
```

**Visual:**
```
Port Purposes:
┌──────────────────────────────────┐
│  4317  →  OTLP gRPC receiver            │
│  4318  →  OTLP HTTP receiver              │
└──────────────────────────────────┘

Most SDKs default to gRPC (4317) for efficiency,
but HTTP (4318) is available for environments
where gRPC is blocked/problematic (some corporate
proxies, browsers).
```

---

## Basic Collector Configuration

```yaml
# otel-collector-config.yaml
receivers:
  otlp:
    protocols:
      grpc:
      http:

processors:
  batch:

exporters:
  logging:
    loglevel: debug
  otlp/jaeger:
    endpoint: jaeger:4317
    tls:
      insecure: true

service:
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [logging, otlp/jaeger]
```

**Visual:**
```
Config Structure Maps Directly to the Architecture:
receivers:   → HOW data gets IN (otlp = accept OTLP data)
processors:  → WHAT happens to it in between (batch = group
              spans together before exporting, more efficient)
exporters:   → WHERE it goes OUT (logging = print to console
              for debugging; otlp/jaeger = send to Jaeger)
service.pipelines: → WIRES receivers → processors → exporters
              together into an actual data flow
```

---

## Instrumenting a Python Application

```bash
pip install opentelemetry-distro opentelemetry-exporter-otlp
opentelemetry-bootstrap -a install
```

```python
# app.py
from opentelemetry import trace
from flask import Flask

app = Flask(__name__)
tracer = trace.get_tracer(__name__)

@app.route("/")
def hello():
    with tracer.start_as_current_span("hello-handler"):
        return "Hello, World!"

if __name__ == "__main__":
    app.run(port=5000)
```

**Running with auto-instrumentation:**
```bash
opentelemetry-instrument \
  --traces_exporter otlp \
  --exporter_otlp_endpoint http://localhost:4317 \
  --service_name my-python-app \
  python app.py
```

**Visual:**
```
What "opentelemetry-instrument" does:
Wraps your application's startup, automatically
instrumenting KNOWN libraries (Flask, requests,
psycopg2, etc.) WITHOUT you manually adding spans
for every single library call — you only add manual
spans (like "hello-handler" above) for YOUR OWN
custom business logic that auto-instrumentation
can't know about.
```

---

## Instrumenting a Node.js Application

```bash
npm install @opentelemetry/sdk-node @opentelemetry/auto-instrumentations-node @opentelemetry/exporter-trace-otlp-grpc
```

```javascript
// tracing.js (must be required BEFORE your app code)
const { NodeSDK } = require('@opentelemetry/sdk-node');
const { getNodeAutoInstrumentations } = require('@opentelemetry/auto-instrumentations-node');
const { OTLPTraceExporter } = require('@opentelemetry/exporter-trace-otlp-grpc');

const sdk = new NodeSDK({
  traceExporter: new OTLPTraceExporter({ url: 'http://localhost:4317' }),
  instrumentations: [getNodeAutoInstrumentations()],
  serviceName: 'my-node-app',
});

sdk.start();
```

```bash
node -r ./tracing.js app.js
```

**Visual:**
```
Why "-r ./tracing.js" (require FIRST) matters:
OpenTelemetry's auto-instrumentation works by
"patching" library code (e.g. wrapping the http
module) at load time. If your application code
requires 'http' or 'express' BEFORE the SDK has
initialized, those modules are already loaded
unpatched, and won't be automatically traced.
Tracing setup must ALWAYS load first.
```

---

## Running Everything Together

```yaml
# docker-compose.yml
version: "3"
services:
  otel-collector:
    image: otel/opentelemetry-collector:latest
    volumes:
      - ./otel-collector-config.yaml:/etc/otelcol/config.yaml
    ports:
      - "4317:4317"
      - "4318:4318"

  jaeger:
    image: jaegertracing/all-in-one:latest
    ports:
      - "16686:16686"  # Jaeger UI
      - "4317"          # Jaeger's own OTLP receiver (alternative)
```

```bash
docker-compose up -d
opentelemetry-instrument --service_name my-python-app python app.py
```

**Visual:**
```
Full Local Stack:
┌──────────┐  OTLP  ┌──────────────┐  OTLP  ┌──────────┐
│  Your App      │ ──────→│  OTel Collector    │ ──────→│  Jaeger        │
│  (instrumented)   │           │  (batches, routes)   │           │  (stores,        │
└──────────┘                └──────────────┘                │   visualizes)    │
                                                              └──────────┘
```

---

## Verifying Your First Trace

```bash
curl http://localhost:5000/
```

Then open the Jaeger UI: `http://localhost:16686`

**Visual:**
```
Jaeger UI Search:
┌─────────────────────────────────────┐
│  Service: my-python-app                    │
│  [Find Traces]                                │
├─────────────────────────────────────┤
│  Trace: a1b2c3d4  (12ms)                       │
│    hello-handler        ▓▓ 8ms                    │
│      GET /                ▓ 3ms                        │
└─────────────────────────────────────┘

Seeing your custom "hello-handler" span AND an
automatically-generated "GET /" span (from Flask's
auto-instrumentation) confirms BOTH manual and
automatic instrumentation are working correctly
together.
```

---

## Real-Life DevOps Use Case

**Scenario:** A DevOps engineer is setting up OpenTelemetry for a new Python microservice for the first time and wants to confirm the pipeline end-to-end before rolling it out to the rest of the team.

**What they do:**
1. Starts with the **`logging` exporter** in the Collector config (printing telemetry straight to console output) BEFORE wiring up Jaeger — this lets them immediately confirm the SDK is generating and sending spans at all, without needing to also debug whether Jaeger itself is receiving/displaying correctly at the same time.
2. Once console output confirms spans are flowing, adds the **Jaeger exporter** alongside logging (not replacing it) — keeping the debug visibility available while adding the real backend.
3. Deliberately tests the **require-order/import-order pitfall** by intentionally getting it wrong first (importing Flask before initializing tracing) — confirming they understand what "broken" looks like (missing auto-instrumented spans) before rolling this out as a checklist item for other teams to avoid.
4. Documents a **standard bootstrap snippet** for the team's Python services, ensuring every new service correctly initializes tracing before any other imports, avoiding the most common early OpenTelemetry mistake.

**Why this matters:** Verifying each piece (SDK → Collector → backend) incrementally, rather than wiring everything at once, makes it dramatically faster to isolate WHERE a problem is when something doesn't show up in the final dashboard — a debugging discipline worth establishing before rolling instrumentation out broadly.

---

Next: **02core_concepts_signals.md** — the deep theory behind spans, traces, context propagation, and resources.