# Prometheus Phase 1: Core Concepts and Architecture

Comprehensive guide to observability fundamentals, Prometheus architecture, and monitoring principles for cloud-native and containerized environments.

## Table of Contents
- [Observability in Modern Cloud Infrastructure](#observability-in-modern-cloud-infrastructure)
- [The Three Pillars of Observability](#the-three-pillars-of-observability)
- [Prometheus Architecture Deep Dive](#prometheus-architecture-deep-dive)
- [Time-Series Database Internals](#time-series-database-internals)
- [Pull vs Push Models](#pull-vs-push-models)
- [Exporters Ecosystem](#exporters-ecosystem)
- [Real-World Monitoring Scenarios](#real-world-monitoring-scenarios)

---

## Observability in Modern Cloud Infrastructure

### The Evolution of Infrastructure

#### Traditional Monolithic Architecture (Pre-2010)

**Characteristics:**
```
Architecture:
┌──────────────────────────────────────┐
│         Single Application           │
│                                      │
│  ┌────────────┐   ┌──────────────┐   │
│  │  Frontend  │ → │   Backend    │   │
│  │   (HTML)   │   │   (Java)     │   │
│  └────────────┘   └──────┬───────┘   │
│                           ↓          │
│                  ┌──────────────┐    │
│                  │   Database   │    │
│                  │  (MySQL)     │    │
│                  └──────────────┘    │
│                                      │
│  Deployed on: 3-5 physical servers   │
└──────────────────────────────────────┘

Monitoring Needs:
✓ Is the server up?
✓ CPU usage < 80%
✓ Memory usage < 90%
✓ Disk space available
✓ Application running

Tools Used:
- Nagios (ping checks, SNMP)
- Cacti (graphs)
- Shell scripts
- Log files on servers
```

**Monitoring Approach:**
```bash
# Simple monitoring script
#!/bin/bash

# Check if server is up
ping -c 1 192.168.1.10 || echo "Server DOWN!"

# Check CPU usage
cpu_usage=$(top -bn1 | grep "Cpu(s)" | awk '{print $2}')
if [ $cpu_usage > 80 ]; then
    echo "High CPU usage: $cpu_usage%"
fi

# Check if application is running
ps aux | grep java | grep -v grep || echo "Application DOWN!"
```

**Problems:**
- Manual configuration for each server
- No auto-discovery
- Difficult to scale
- Host-centric (not service-centric)
- Limited querying capabilities

#### Modern Microservices Architecture (2015+)

**Characteristics:**
```
E-Commerce Platform Architecture:

┌────────────────────────────────────────────────────────────┐
│                 Kubernetes Cluster                         │
│                                                            │
│  User Request Flow:                                        │
│  Browser → Load Balancer → API Gateway → Microservices     │
│                                                            │
│  Microservices:                                            │
│  ┌─────────────────┐  ┌─────────────────┐                  │
│  │  User Service   │  │ Product Service │                  │
│  │  3 pods         │  │  5 pods         │                  │
│  │  Port: 8080     │  │  Port: 8080     │                  │
│  └────────┬────────┘  └────────┬────────┘                  │
│           ↓                     ↓                          │
│  ┌─────────────────┐  ┌─────────────────┐                  │
│  │  Cart Service   │  │ Order Service   │                  │
│  │  2 pods         │  │  4 pods         │                  │
│  └────────┬────────┘  └────────┬────────┘                  │
│           ↓                     ↓                          │
│  ┌─────────────────┐  ┌─────────────────┐                  │
│  │Payment Service  │  │Notification Svc │                  │
│  │  2 pods         │  │  3 pods         │                  │
│  └────────┬────────┘  └────────┬────────┘                  │
│           ↓                     ↓                          │
│  ┌─────────────────┐  ┌─────────────────┐                  │
│  │Inventory Service│  │ Analytics Svc   │                  │
│  │  3 pods         │  │  2 pods         │                  │
│  └─────────────────┘  └─────────────────┘                  │
│                                                            │
│  Infrastructure Components:                                │
│  ┌─────────────────┐  ┌─────────────────┐                  │
│  │  PostgreSQL     │  │   MongoDB       │                  │
│  │  (3 replicas)   │  │  (3 replicas)   │                  │
│  └─────────────────┘  └─────────────────┘                  │
│                                                            │
│  ┌─────────────────┐  ┌─────────────────┐                  │
│  │   Redis Cache   │  │   RabbitMQ      │                  │
│  │  (Cluster)      │  │  (Message Queue)│                  │
│  └─────────────────┘  └─────────────────┘                  │
│                                                            │
│  Total: 24 application pods + 9 infrastructure pods        │
│  Running on: 5 Kubernetes worker nodes                     │
│  Auto-scaling: Pods scale 2-10 based on load               │
└────────────────────────────────────────────────────────────┘
```

**Complexity Factors:**

**1. Dynamic Infrastructure:**
```
Traditional:
server-01 → Always at 192.168.1.10
         → Same hostname forever
         → Monitor this IP/hostname

Kubernetes:
payment-pod-xyz123 → Created at 10.244.1.5 at 10:00 AM
                   → Might die at 10:15 AM (OOMKilled)
                   → Replaced by payment-pod-abc456 at 10.244.2.8
                   → Completely different IP
                   → Must auto-discover continuously
```

**2. Horizontal Scaling:**
```
Traffic Pattern (Typical Day):

Morning (8 AM):
- User Service: 2 pods
- Order Service: 2 pods
- Traffic: 100 req/sec

Lunch (12 PM - Peak):
- User Service: 8 pods (auto-scaled)
- Order Service: 12 pods (auto-scaled)
- Traffic: 2000 req/sec

Night (2 AM):
- User Service: 1 pod (scaled down)
- Order Service: 1 pod (scaled down)
- Traffic: 10 req/sec

Monitoring Challenge:
- Must aggregate metrics across all pods
- Track individual pod performance
- Monitor scaling events
- Detect if scaling is working properly
```

**3. Service Dependencies:**
```
Order Creation Request Flow:

User clicks "Place Order"
    ↓
API Gateway (5ms)
    ↓
Auth Service (10ms) - Validates JWT token
    ↓
Order Service (450ms total)
    ↓
    ├→ User Service (50ms) - Get user details
    │   └→ PostgreSQL (30ms)
    ├→ Cart Service (40ms) - Get cart items
    │   └→ Redis (10ms)
    ├→ Product Service (60ms) - Validate products
    │   └→ MongoDB (40ms)
    ├→ Inventory Service (200ms) - Check stock
    │   └→ PostgreSQL (180ms) ← SLOW!
    ├→ Payment Service (80ms) - Process payment
    │   └→ External Payment Gateway (60ms)
    └→ Notification Service (20ms) - Send email
        └→ RabbitMQ (5ms)

Total: 487ms

Problem: Order creation is slow
Question: Which service/database is the bottleneck?
Traditional Monitoring: Can't tell!
Observability: Shows Inventory Service → PostgreSQL is slow
```

### What is Observability?

**Definition:**
> Observability is the ability to understand the internal state of a system by examining its external outputs (metrics, logs, traces), without needing to deploy new code or add instrumentation for each new question.

**Key Difference from Monitoring:**

```
┌──────────────────────────────────────────────────────────┐
│                   MONITORING                             │
├──────────────────────────────────────────────────────────┤
│ Known Knowns                                             │
│ - Pre-defined checks                                     │
│ - Known failure modes                                    │
│ - "Is the service up?"                                   │
│ - "Is CPU > 80%?"                                        │
│                                                          │
│ Limitations:                                             │
│ - Can only check what you thought to check               │
│ - Requires updating monitoring for new questions         │
│ - Binary: UP or DOWN                                     │
└──────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────┐
│                   OBSERVABILITY                          │
├──────────────────────────────────────────────────────────┤
│ Known Unknowns + Unknown Unknowns                        │
│ - Ad-hoc exploration                                     │
│ - Unknown failure modes                                  │
│ - "Why is checkout slow for iOS users in Germany?"       │
│ - "What changed between yesterday and today?"            │
│                                                          │
│ Capabilities:                                            │
│ - Ask new questions without deploying code               │
│ - Slice and dice data by any dimension                   │
│ - Root cause analysis                                    │
└──────────────────────────────────────────────────────────┘
```

**Real-World Example:**

**Scenario:** Users reporting "slow checkout"

**Traditional Monitoring Approach:**
```
Check predefined alerts:
❌ Server CPU: 45% (normal)
❌ Server Memory: 60% (normal)
❌ Application UP: Yes
❌ Database UP: Yes
❌ Response time: 200ms (normal overall average)

Conclusion: "Everything looks fine" 
But users still experiencing slowness!

Problem: Monitoring only checks what was pre-configured
Can't discover the real issue
```

**Observability Approach:**
```
1. Check metrics (high-level):
   ✓ Overall response time: 200ms average
   ✓ But P95 response time: 3000ms ← Issue!
   ✓ Filter by: iOS users only
   ✓ Filter by: Germany region
   ✓ Payment Service showing high latency

2. Check distributed traces:
   ✓ Trace shows: Payment Gateway timeout
   ✓ Only for European Payment Gateway
   ✓ Timeout happens after 3 seconds

3. Check logs:
   ✓ "Connection timeout to eu.payment-gateway.com"
   ✓ "Falling back to us.payment-gateway.com"
   ✓ Fallback adds 2 seconds

Root Cause: European Payment Gateway having issues
Solution: Route EU users to different payment processor
Time to resolution: 15 minutes (vs hours/days)
```

### Why Cloud-Native Needs Observability

**Challenge 1: Ephemeral Containers**
```
Container Lifecycle:

10:00:00 - Pod payment-xyz created
10:00:05 - Container starts
10:00:10 - Healthy, serving traffic
...
10:45:00 - Memory usage increases
10:45:30 - OOMKilled (out of memory)
10:45:35 - Pod payment-abc created (new IP)
10:45:40 - New container starts

Monitoring Requirements:
✓ Track metrics even for short-lived containers
✓ Aggregate across pod replacements
✓ Identify why pod was killed
✓ Historical data for debugging
```

**Challenge 2: Service Discovery**
```
Without Auto-Discovery:

Day 1: Manually add payment-pod-1 to monitoring
Day 2: payment-pod-1 dies, replaced by payment-pod-2
       → Monitoring broken, manual update needed
Day 3: Auto-scaling creates payment-pod-3, 4, 5
       → 3 pods not monitored!

With Auto-Discovery (Prometheus):

Kubernetes API → List all pods with label: app=payment
Prometheus → Automatically scrapes all matching pods
Pod created → Auto-added to monitoring
Pod deleted → Auto-removed from monitoring
No manual configuration needed!
```

**Challenge 3: Multi-Dimensional Data**
```
Traditional: Monitor "web-server-01"
Metrics: CPU, Memory, Disk

Cloud-Native: Monitor "payment-service"
Need to slice by:
- Pod name: payment-pod-xyz
- Node: worker-node-02
- Version: v2.1.0
- Environment: production
- Region: us-east-1
- Team: payments-team

Example Query:
"Show me error rate for payment-service version v2.1.0 
 in production, deployed in the last hour,
 grouped by pod, filtered to only show errors > 1%"

Prometheus can do this!
Traditional monitoring cannot!
```

---

## The Three Pillars of Observability

### Overview: Why Three Pillars?

```
Each pillar answers different questions:

METRICS          LOGS            TRACES
"What?"          "Why?"          "Where?"

Numbers          Events          Journeys
Aggregated       Detailed        Distributed
Cheap storage    Expensive       Very expensive
Fast queries     Slow searches   Complex analysis
Broad view       Deep dive       Flow visualization
```

### Pillar 1: Metrics

#### What Are Metrics?

**Definition:** Numerical measurements recorded over time

**Characteristics:**
```
✓ Time-stamped
✓ Aggregatable
✓ Low cardinality (limited label combinations)
✓ Efficient storage
✓ Fast queries
✓ Real-time dashboards
```

#### Metric Structure in Prometheus

**Basic Format:**
```
metric_name{label1="value1", label2="value2"} value timestamp

Example:
http_requests_total{method="GET", endpoint="/api/users", status="200"} 1547 1610000000000

Components:
- Metric name: http_requests_total
- Labels (dimensions):
  * method: GET (request type)
  * endpoint: /api/users (which API)
  * status: 200 (response code)
- Value: 1547 (number of requests)
- Timestamp: 1610000000000 (Unix milliseconds)
```

#### Real-World Metrics Example

**Node.js Application - Payment Service:**

```javascript
// payment-service/server.js
const express = require('express');
const promClient = require('prom-client');

const app = express();
const register = new promClient.Registry();

// Metric 1: Counter - Total requests
const httpRequestsTotal = new promClient.Counter({
  name: 'http_requests_total',
  help: 'Total number of HTTP requests',
  labelNames: ['method', 'endpoint', 'status'],
  registers: [register]
});

// Metric 2: Histogram - Response time
const httpRequestDuration = new promClient.Histogram({
  name: 'http_request_duration_seconds',
  help: 'HTTP request duration in seconds',
  labelNames: ['method', 'endpoint'],
  buckets: [0.1, 0.5, 1, 2, 5], // 100ms, 500ms, 1s, 2s, 5s
  registers: [register]
});

// Metric 3: Gauge - Active connections
const activeConnections = new promClient.Gauge({
  name: 'active_connections',
  help: 'Number of active connections',
  registers: [register]
});

// Middleware to track metrics
app.use((req, res, next) => {
  const start = Date.now();
  
  // Increment active connections
  activeConnections.inc();
  
  res.on('finish', () => {
    // Record request
    httpRequestsTotal.inc({
      method: req.method,
      endpoint: req.route?.path || 'unknown',
      status: res.statusCode
    });
    
    // Record duration
    const duration = (Date.now() - start) / 1000;
    httpRequestDuration.observe({
      method: req.method,
      endpoint: req.route?.path || 'unknown'
    }, duration);
    
    // Decrement active connections
    activeConnections.dec();
  });
  
  next();
});

// Business logic
app.post('/api/payment', async (req, res) => {
  try {
    // Process payment
    const result = await processPayment(req.body);
    res.json({ success: true, transaction_id: result.id });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Expose metrics endpoint
app.get('/metrics', async (req, res) => {
  res.set('Content-Type', register.contentType);
  res.end(await register.metrics());
});

app.listen(8080);
```

**Metrics Output (GET /metrics):**
```
# HELP http_requests_total Total number of HTTP requests
# TYPE http_requests_total counter
http_requests_total{method="GET",endpoint="/api/users",status="200"} 1547
http_requests_total{method="POST",endpoint="/api/payment",status="200"} 245
http_requests_total{method="POST",endpoint="/api/payment",status="500"} 12

# HELP http_request_duration_seconds HTTP request duration in seconds
# TYPE http_request_duration_seconds histogram
http_request_duration_seconds_bucket{method="POST",endpoint="/api/payment",le="0.1"} 150
http_request_duration_seconds_bucket{method="POST",endpoint="/api/payment",le="0.5"} 230
http_request_duration_seconds_bucket{method="POST",endpoint="/api/payment",le="1"} 250
http_request_duration_seconds_bucket{method="POST",endpoint="/api/payment",le="2"} 255
http_request_duration_seconds_bucket{method="POST",endpoint="/api/payment",le="5"} 257
http_request_duration_seconds_bucket{method="POST",endpoint="/api/payment",le="+Inf"} 257
http_request_duration_seconds_sum{method="POST",endpoint="/api/payment"} 89.7
http_request_duration_seconds_count{method="POST",endpoint="/api/payment"} 257

# HELP active_connections Number of active connections
# TYPE active_connections gauge
active_connections 42
```

#### When to Use Metrics

**✅ Perfect For:**
```
1. Real-time dashboards
   - Current request rate
   - CPU/Memory usage
   - Error rates

2. Alerting
   - Error rate > 5%
   - Response time > 1s
   - Disk usage > 80%

3. Trend analysis
   - Traffic growth over weeks
   - Seasonal patterns
   - Capacity planning

4. SLO/SLA tracking
   - 99.9% uptime
   - P95 latency < 500ms
```

**❌ Not Good For:**
```
1. Debugging specific errors
   - Why did request xyz fail?
   - What was the exact input?

2. Audit trails
   - Who accessed this resource?
   - When was user data modified?

3. High-cardinality data
   - User IDs (millions of unique values)
   - Transaction IDs
   - IP addresses
```

### Pillar 2: Logs

#### What Are Logs?

**Definition:** Discrete events with rich context, recorded as text

**Characteristics:**
```
✓ Detailed context
✓ Searchable text
✓ High cardinality (unique events)
✓ Timestamp + structured data
✓ Expensive to store
✓ Slow to search (without indexing)
```

#### Log Levels

```
DEBUG   → Verbose information for developers
          "Database query: SELECT * FROM users WHERE id = 12345"

INFO    → General informational messages
          "User 12345 logged in successfully"

WARN    → Warning, potential issues
          "Connection pool at 90% capacity"

ERROR   → Error events, application continues
          "Failed to process payment: timeout"

FATAL   → Severe errors, might crash
          "Out of memory, shutting down"
```

#### Real-World Logs Example

**Structured Logging (JSON):**

```javascript
// payment-service/logger.js
const winston = require('winston');

const logger = winston.createLogger({
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.json()
  ),
  transports: [
    new winston.transports.Console(),
    new winston.transports.File({ filename: 'app.log' })
  ]
});

// Payment processing with structured logs
async function processPayment(paymentData) {
  const start = Date.now();
  
  logger.info('Payment initiated', {
    user_id: paymentData.userId,
    amount: paymentData.amount,
    currency: paymentData.currency,
    payment_method: paymentData.method,
    transaction_id: paymentData.transactionId
  });
  
  try {
    // Check fraud
    logger.debug('Running fraud check', {
      transaction_id: paymentData.transactionId
    });
    
    const fraudCheck = await checkFraud(paymentData);
    if (fraudCheck.flagged) {
      logger.warn('Potential fraud detected', {
        transaction_id: paymentData.transactionId,
        fraud_score: fraudCheck.score,
        reason: fraudCheck.reason
      });
      throw new Error('Transaction flagged for review');
    }
    
    // Process payment
    logger.info('Calling payment gateway', {
      transaction_id: paymentData.transactionId,
      gateway: 'stripe'
    });
    
    const result = await paymentGateway.charge(paymentData);
    
    const duration = Date.now() - start;
    logger.info('Payment successful', {
      transaction_id: paymentData.transactionId,
      gateway_response_id: result.id,
      duration_ms: duration
    });
    
    return result;
    
  } catch (error) {
    const duration = Date.now() - start;
    logger.error('Payment failed', {
      transaction_id: paymentData.transactionId,
      error_message: error.message,
      error_stack: error.stack,
      duration_ms: duration,
      user_id: paymentData.userId
    });
    throw error;
  }
}
```

**Log Output:**
```json
{
  "timestamp": "2024-01-15T14:30:45.123Z",
  "level": "info",
  "message": "Payment initiated",
  "user_id": 12345,
  "amount": 99.99,
  "currency": "USD",
  "payment_method": "credit_card",
  "transaction_id": "tx-789-abc"
}

{
  "timestamp": "2024-01-15T14:30:45.456Z",
  "level": "warn",
  "message": "Potential fraud detected",
  "transaction_id": "tx-789-abc",
  "fraud_score": 0.85,
  "reason": "unusual_location"
}

{
  "timestamp": "2024-01-15T14:30:46.789Z",
  "level": "error",
  "message": "Payment failed",
  "transaction_id": "tx-789-abc",
  "error_message": "Payment gateway timeout",
  "error_stack": "Error: timeout\n at Gateway.charge...",
  "duration_ms": 3200,
  "user_id": 12345
}
```

#### When to Use Logs

**✅ Perfect For:**
```
1. Debugging errors
   - Stack traces
   - Exact inputs that caused failure
   - Sequence of events leading to error

2. Audit trails
   - User actions (login, logout, purchases)
   - Data modifications
   - Security events

3. Business events
   - Order placed
   - Payment processed
   - User registered

4. Compliance
   - GDPR access logs
   - Financial transaction records
```

**❌ Not Good For:**
```
1. Real-time monitoring
   - Too slow to aggregate
   - Expensive to query continuously

2. Trend analysis
   - Better suited for metrics
   - Logs not optimized for aggregation

3. Dashboards
   - Metrics are faster and cheaper
```

### Pillar 3: Traces

#### What Are Traces?

**Definition:** Journey of a request through a distributed system

**Components:**
```
Trace: Complete request journey
  ↓
Spans: Individual operations within trace
  ↓
Tags: Metadata about spans
  ↓
Logs: Events within spans
```

#### Distributed Tracing Concepts

**Trace Structure:**
```
Trace ID: abc-123-def-456
Duration: 487ms
Status: Success
Spans: 8

Parent Span: API Gateway
  Child Span: Auth Service
  Child Span: Order Service
    Child Span: User Service
      Child Span: Database Query
    Child Span: Product Service
    Child Span: Inventory Service
      Child Span: Database Query
  Child Span: Response
```

#### Real-World Trace Example

**Scenario: Order Creation**

```
User Request: POST /api/orders

Trace Visualization (Waterfall):

Timeline (ms):  0    100   200   300   400   500
                │─────│─────│─────│─────│─────│
API Gateway     │██│                             (5ms)
  Auth Service       │████│                      (12ms)
  Order Service           │████████████████████████│ (450ms)
    User Service             │██████│               (50ms)
      DB Query                  │████│              (30ms)
    Cart Service                    │████│          (40ms)
      Redis                           │█│           (10ms)
    Product Service                      │██████│   (60ms)
      DB Query                              │████│  (40ms)
    Inventory Service                          │████████████████│ (200ms) ← SLOW!
      DB Query                                   │██████████████│ (180ms)
    Payment Service                                        │████│ (80ms)
      Gateway API                                            │██│ (60ms)
  Response                                                        │█│ (20ms)

Bottleneck Identified: Inventory Service database query (180ms)
```

**Implementing Tracing (OpenTelemetry):**

```javascript
// order-service/tracing.js
const { trace } = require('@opentelemetry/api');
const { NodeTracerProvider } = require('@opentelemetry/sdk-trace-node');
const { SimpleSpanProcessor } = require('@opentelemetry/sdk-trace-base');
const { JaegerExporter } = require('@opentelemetry/exporter-jaeger');

// Initialize tracing
const provider = new NodeTracerProvider();
const exporter = new JaegerExporter({
  endpoint: 'http://jaeger:14268/api/traces',
});
provider.addSpanProcessor(new SpanProcessor(exporter));
provider.register();

const tracer = trace.getTracer('order-service');

// order-service/controller.js
async function createOrder(req, res) {
  // Start trace for this request
  const span = tracer.startSpan('create_order');
  
  try {
    span.setAttribute('user_id', req.user.id);
    span.setAttribute('items_count', req.body.items.length);
    
    // Get user details
    const userSpan = tracer.startSpan('get_user', { parent: span });
    const user = await userService.getUser(req.user.id);
    userSpan.end();
    
    // Get cart items
    const cartSpan = tracer.startSpan('get_cart', { parent: span });
    const cart = await cartService.getCart(req.user.id);
    cartSpan.end();
    
    // Check inventory
    const inventorySpan = tracer.startSpan('check_inventory', { parent: span });
    const available = await inventoryService.checkStock(cart.items);
    if (!available) {
      inventorySpan.setAttribute('error', true);
      inventorySpan.setAttribute('error.message', 'Out of stock');
      throw new Error('Items out of stock');
    }
    inventorySpan.end();
    
    // Process payment
    const paymentSpan = tracer.startSpan('process_payment', { parent: span });
    const payment = await paymentService.charge({
      amount: cart.total,
      userId: req.user.id
    });
    paymentSpan.setAttribute('payment_id', payment.id);
    paymentSpan.end();
    
    // Create order
    const order = await Order.create({
      userId: req.user.id,
      items: cart.items,
      total: cart.total,
      paymentId: payment.id
    });
    
    span.setAttribute('order_id', order.id);
    span.setStatus({ code: SpanStatusCode.OK });
    
    res.json({ orderId: order.id });
    
  } catch (error) {
    span.setStatus({ 
      code: SpanStatusCode.ERROR,
      message: error.message 
    });
    res.status(500).json({ error: error.message });
  } finally {
    span.end();
  }
}
```

#### When to Use Traces

**✅ Perfect For:**
```
1. Microservices debugging
   - Which service is slow?
   - Where is the bottleneck?
   - Service dependency mapping

2. Performance optimization
   - Identify slow database queries
   - Find N+1 query problems
   - Optimize critical paths

3. Understanding request flow
   - How services interact
   - Request propagation
   - Error propagation

4. SLA validation
   - Did request meet latency requirement?
   - Which component violated SLA?
```

**❌ Not Good For:**
```
1. Every request (too expensive)
   - Usually sample 1-10% of traffic
   - 100% tracing = high overhead

2. Metrics aggregation
   - Use metrics instead
   - Cheaper and faster

3. Long-term storage
   - Very expensive
   - Keep recent traces only
```

### Combining All Three Pillars

**Incident Response Workflow:**

```
STEP 1: Alert fires (METRICS)
────────────────────────────────────────────────────
Alert: High error rate in payment-service
Trigger: error_rate > 5% for 5 minutes
Time: 14:30 PM

STEP 2: Check Dashboard (METRICS)
────────────────────────────────────────────────────
Dashboard shows:
- Error rate: 12% (normal: 0.5%)
- Started: 14:25 PM
- Affected endpoints: POST /api/payment
- Affected pods: All payment-service pods
- Request rate: 50 errors/second

STEP 3: View Distributed Traces (TRACES)
────────────────────────────────────────────────────
Sample trace of failed request:
- Total duration: 30 seconds (timeout)
- Bottleneck: Payment Gateway API call
- Status: HTTP 504 Gateway Timeout
- Pattern: All failures timeout at exactly 30s

STEP 4: Check Application Logs (LOGS)
────────────────────────────────────────────────────
Search for: "payment" AND "error" AND timestamp:14:25-14:30

Logs show:
[ERROR] Payment gateway timeout
  gateway_url: https://api.payment-gateway.com
  timeout_seconds: 30
  error: "Connection timeout"
  transaction_id: tx-12345

Multiple occurrences of same error

STEP 5: Check External Service Status
────────────────────────────────────────────────────
Visit: status.payment-gateway.com
Status: "Degraded performance in EU region"

STEP 6: Root Cause
────────────────────────────────────────────────────
Root Cause: Payment Gateway API having issues
Impact: All payment requests failing
Affected users: All regions (using same gateway)

STEP 7: Mitigation
────────────────────────────────────────────────────
Action: Switch to backup payment processor
- Update configuration
- Deploy change
- Monitor error rate

STEP 8: Verify Fix (METRICS)
────────────────────────────────────────────────────
Dashboard shows:
- Error rate: 0.5% (normal)
- Recovery time: 14:40 PM
- Total incident duration: 15 minutes
```

---

## Prometheus Architecture Deep Dive

### Complete System Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Prometheus Ecosystem                        │
│                                                                     │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │                    Prometheus Server                          │  │
│  │                       (Port 9090)                             │  │
│  │                                                               │  │
│  │  ┌──────────────────────────────────────────────────────┐     │  │
│  │  │              RETRIEVAL COMPONENT                     │     │  │
│  │  │                                                      │     │  │
│  │  │  ┌────────────────────────────────────────────┐      │     │  │
│  │  │  │  Service Discovery                         │      │     │  │
│  │  │  │  - Kubernetes (pods, services, endpoints)  │      │     │  │
│  │  │  │  - Consul                                  │      │     │  │
│  │  │  │  - EC2 (AWS instances)                     │      │     │  │
│  │  │  │  - File-based (static configs)             │      │     │  │
│  │  │  └────────────────────────────────────────────┘      │     │  │
│  │  │                      ↓                               │     │  │
│  │  │  ┌────────────────────────────────────────────┐      │     │  │
│  │  │  │  Target Manager                            │      │     │  │
│  │  │  │  - Maintains list of scrape targets        │      │     │  │
│  │  │  │  - Applies relabeling rules                │      │     │  │
│  │  │  │  - Tracks target health                    │      │     │  │
│  │  │  └────────────────────────────────────────────┘      │     │  │
│  │  │                      ↓                               │     │  │
│  │  │  ┌────────────────────────────────────────────┐      │     │  │
│  │  │  │  Scrape Loop                               │      │     │  │
│  │  │  │  - HTTP GET to /metrics every 15s          │      │     │  │
│  │  │  │  - Parse exposition format                 │      │     │  │
│  │  │  │  - Apply metric relabeling                 │      │     │  │
│  │  │  │  - Send to Storage                         │      │     │  │
│  │  │  └────────────────────────────────────────────┘      │     │  │
│  │  └──────────────────────────────────────────────────────┘     │  │
│  │                              ↓                                │  │
│  │  ┌──────────────────────────────────────────────────┐         │  │
│  │  │         TIME SERIES DATABASE (TSDB)              │         │  │
│  │  │                                                  │         │  │
│  │  │  ┌────────────────────────────────────────┐      │         │  │
│  │  │  │  Head Block (In-Memory)                │      │         │  │
│  │  │  │  - Recent 2-3 hours of data            │      │         │  │
│  │  │  │  - Fast writes                         │      │         │  │
│  │  │  │  - WAL for crash recovery              │      │         │  │
│  │  │  └────────────────────────────────────────┘      │         │  │
│  │  │                      ↓                           │         │  │
│  │  │  ┌────────────────────────────────────────┐      │         │  │
│  │  │  │  Persisted Blocks (Disk)               │      │         │  │
│  │  │  │  - 2-hour immutable blocks             │      │         │  │
│  │  │  │  - Compressed chunks                   │      │         │  │
│  │  │  │  - Indexed for fast queries            │      │         │  │
│  │  │  │  - Retention: 15 days (configurable)   │      │         │  │
│  │  │  └────────────────────────────────────────┘      │         │  │
│  │  └──────────────────────────────────────────────────┘         │  │
│  │                              ↓                                │  │
│  │  ┌──────────────────────────────────────────────────┐         │  │
│  │  │           QUERY ENGINE                           │         │  │
│  │  │  - PromQL parser and evaluator                   │         │  │
│  │  │  - Query optimizer                               │         │  │
│  │  │  - Federation support                            │         │  │
│  │  └──────────────────────────────────────────────────┘         │  │
│  │                              ↓                                │  │
│  │  ┌──────────────────────────────────────────────────┐         │  │
│  │  │           RULE EVALUATION                        │         │  │
│  │  │  - Recording rules (pre-compute queries)         │         │  │
│  │  │  - Alerting rules (check conditions)             │         │  │
│  │  │  - Periodic evaluation (every 15s-1m)            │         │  │
│  │  └──────────────────────────────────────────────────┘         │  │
│  │                              ↓                                │  │
│  │  ┌──────────────────────────────────────────────────┐         │  │
│  │  │           HTTP API SERVER                        │         │  │
│  │  │  - /api/v1/* (REST API)                          │         │  │
│  │  │  - /metrics (Prometheus' own metrics)            │         │  │
│  │  │  - /graph (Expression Browser UI)                │         │  │
│  │  │  - /targets (Scrape targets status)              │         │  │
│  │  └──────────────────────────────────────────────────┘         │  │
│  └───────────────────────────────────────────────────────────────┘  │
│                                                                     │
│  External Components:                                               │
│                                                                     │
│  ┌────────────────────────┐    ┌──────────────────────────┐         │
│  │   TARGET SYSTEMS       │    │    ALERTMANAGER          │         │
│  │   (Scraped by Prom)    │    │    (Port 9093)           │         │
│  │                        │    │                          │         │
│  │ ┌──────────────────┐   │    │ - Receives alerts        │         │
│  │ │ Applications     │   │    │ - Deduplication          │         │
│  │ │ with /metrics    │   │    │ - Grouping               │         │
│  │ └──────────────────┘   │    │ - Routing                │         │
│  │                        │    │ - Silencing              │         │
│  │ ┌──────────────────┐   │    │ - Inhibition             │         │
│  │ │ Exporters        │   │    └────────────┬─────────────┘         │
│  │ │ (Node, MySQL)    │   │                 │                       │
│  │ └──────────────────┘   │                 ↓                       │
│  │                        │    ┌──────────────────────────┐         │
│  │ ┌──────────────────┐   │    │  NOTIFICATION CHANNELS   │         │
│  │ │ Pushgateway      │   │    │  - Slack                 │         │
│  │ │ (for batch jobs) │   │    │  - Email (SMTP)          │         │
│  │ └──────────────────┘   │    │  - PagerDuty             │         │
│  └────────────────────────┘    │  - Webhook               │         │
│                                │  - OpsGenie              │         │
│  ┌────────────────────────┐    └──────────────────────────┘         │
│  │   VISUALIZATION        │                                         │
│  │                        │                                         │
│  │ ┌──────────────────┐   │                                         │
│  │ │ Grafana          │   │                                         │
│  │ │ Dashboards       │←──┼────── PromQL Queries                    │
│  │ └──────────────────┘   │                                         │
│  │                        │                                         │
│  │ ┌──────────────────┐   │                                         │
│  │ │ Expression       │   │                                         │
│  │ │ Browser (UI)     │←──┼────── Built-in                          │
│  │ └──────────────────┘   │                                         │
│  └────────────────────────┘                                         │
└─────────────────────────────────────────────────────────────────────┘
```

### Component Details

#### 1. Service Discovery

**Purpose:** Automatically find targets to monitor

**Kubernetes Service Discovery (Most Common):**

```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
    - role: pod
      namespaces:
        names:
        - default
        - production
    
    # Relabeling: Filter which pods to scrape
    relabel_configs:
    # Only scrape pods with annotation prometheus.io/scrape=true
    - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
      action: keep
      regex: true
    
    # Get port from annotation prometheus.io/port
    - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_port]
      action: replace
      target_label: __address__
      regex: ([^:]+)(?::\d+)?;(\d+)
      replacement: $1:$2
    
    # Add pod name as label
    - source_labels: [__meta_kubernetes_pod_name]
      target_label: pod
    
    # Add namespace as label
    - source_labels: [__meta_kubernetes_namespace]
      target_label: namespace
    
    # Add app label
    - source_labels: [__meta_kubernetes_pod_label_app]
      target_label: app
```

**How it Works:**
```
1. Prometheus queries Kubernetes API every few seconds

2. Gets list of all pods matching criteria:
   apiVersion: v1
   kind: Pod
   metadata:
     annotations:
       prometheus.io/scrape: "true"
       prometheus.io/port: "8080"
       prometheus.io/path: "/metrics"
     labels:
       app: payment-service
       version: v2.0

3. Prometheus extracts metadata:
   - Pod IP: 10.244.1.5
   - Pod Name: payment-pod-abc123
   - Namespace: production
   - App label: payment-service
   - Port: 8080

4. Constructs scrape target:
   http://10.244.1.5:8080/metrics

5. Adds labels to all metrics:
   http_requests_total{
     pod="payment-pod-abc123",
     namespace="production",
     app="payment-service"
   }

6. Automatically updates:
   - Pod created? Added to targets
   - Pod deleted? Removed from targets
   - Pod IP changed? Updated automatically
```

**Other Service Discovery Options:**

```yaml
# EC2 (AWS)
- job_name: 'ec2-instances'
  ec2_sd_configs:
  - region: us-east-1
    port: 9100
    filters:
    - name: tag:Environment
      values: [production]

# Consul
- job_name: 'consul-services'
  consul_sd_configs:
  - server: 'consul.example.com:8500'
    services: ['web', 'api', 'database']

# Static (file-based)
- job_name: 'static-targets'
  static_configs:
  - targets:
    - '192.168.1.10:9100'
    - '192.168.1.11:9100'
    labels:
      environment: production
```

#### 2. Scraping Process

**Detailed Scrape Flow:**

```
STEP 1: Target Discovery
──────────────────────────────────────
Service Discovery finds:
- payment-pod-abc (10.244.1.5:8080)
- payment-pod-def (10.244.1.6:8080)
- payment-pod-ghi (10.244.1.7:8080)

STEP 2: Scrape Schedule
──────────────────────────────────────
Default: Every 15 seconds
Configurable: 5s, 30s, 1m, etc.

Timeline:
00:00:00 - Scrape all targets
00:00:15 - Scrape all targets
00:00:30 - Scrape all targets
...

STEP 3: HTTP GET Request
──────────────────────────────────────
Prometheus makes request:
GET http://10.244.1.5:8080/metrics HTTP/1.1
Host: 10.244.1.5:8080
User-Agent: Prometheus/2.45.0
Accept: application/openmetrics-text

STEP 4: Target Response
──────────────────────────────────────
HTTP/1.1 200 OK
Content-Type: text/plain; version=0.0.4

# HELP http_requests_total Total HTTP requests
# TYPE http_requests_total counter
http_requests_total{method="GET",endpoint="/api/users",status="200"} 1547
http_requests_total{method="POST",endpoint="/api/payment",status="200"} 245

# HELP http_request_duration_seconds Request duration
# TYPE http_request_duration_seconds histogram
http_request_duration_seconds_bucket{le="0.1"} 150
http_request_duration_seconds_bucket{le="0.5"} 230
http_request_duration_seconds_bucket{le="1"} 250
http_request_duration_seconds_bucket{le="+Inf"} 257
http_request_duration_seconds_sum 89.7
http_request_duration_seconds_count 257

STEP 5: Parse Metrics
──────────────────────────────────────
Prometheus parses exposition format:
- Metric name: http_requests_total
- Labels: {method="GET", endpoint="/api/users", status="200"}
- Value: 1547
- Timestamp: current time (assigned by Prometheus)

STEP 6: Add Target Labels
──────────────────────────────────────
From service discovery metadata:
{
  pod="payment-pod-abc",
  namespace="production",
  app="payment-service",
  instance="10.244.1.5:8080",
  job="kubernetes-pods"
}

STEP 7: Store in TSDB
──────────────────────────────────────
Final metric with all labels:
http_requests_total{
  method="GET",
  endpoint="/api/users",
  status="200",
  pod="payment-pod-abc",
  namespace="production",
  app="payment-service",
  instance="10.244.1.5:8080",
  job="kubernetes-pods"
} 1547 @1610000000000
```

#### 3. Storage Layer - TSDB

**Write-Ahead Log (WAL):**
```
Purpose: Durability and crash recovery

Write Flow:
1. Metric arrives from scraper
2. Written to WAL immediately (append-only)
3. WAL synced to disk
4. Metric added to in-memory head block

Crash Recovery:
1. Prometheus restarts
2. Reads WAL from last checkpoint
3. Replays all operations
4. Rebuilds in-memory state
5. Resumes normal operation

WAL Structure:
/data/prometheus/wal/
├── 00000000  ← Oldest WAL segment
├── 00000001
├── 00000002
└── checkpoint.00001  ← Checkpoint marker
```

**Block Structure:**
```
2-Hour Block:
/data/prometheus/01ABCDEF.../
├── chunks/
│   ├── 000001  ← Compressed time series data
│   ├── 000002
│   └── 000003
├── index       ← Fast lookup index
├── meta.json   ← Block metadata
└── tombstones  ← Deleted series markers

meta.json:
{
  "version": 1,
  "ulid": "01ABCDEF...",
  "minTime": 1610000000000,
  "maxTime": 1610007200000,
  "stats": {
    "numSamples": 1500000,
    "numSeries": 10000,
    "numChunks": 50000
  },
  "compaction": {
    "level": 1,
    "sources": ["01ABC...", "01DEF..."]
  }
}
```

**Compression:**
```
Raw data:
14:30:00  1547
14:30:15  1562  → Stored as: +15
14:30:30  1580  → Stored as: +18
14:30:45  1595  → Stored as: +15

Techniques:
1. Delta-of-deltas encoding
2. XOR compression for floats
3. Varbit encoding
4. Run-length encoding

Result: 50-70% space savings
```

**Compaction:**
```
Purpose: Merge small blocks into larger ones

Process:
Multiple 2-hour blocks → Single 12-hour block → Single 7-day block

Before Compaction:
/data/prometheus/
├── 01ABC... (2h, level 1)
├── 01DEF... (2h, level 1)
├── 01GHI... (2h, level 1)
├── 01JKL... (2h, level 1)
├── 01MNO... (2h, level 1)
└── 01PQR... (2h, level 1)

After Compaction:
/data/prometheus/
└── 01XYZ... (12h, level 2)

Benefits:
- Fewer files to manage
- Better compression
- Faster queries
- Automatic by Prometheus
```

#### 4. Query Engine

**PromQL Query Processing:**

```
Example Query:
rate(http_requests_total{app="payment-service"}[5m])

STEP 1: Parse Query
──────────────────────────────────────
AST (Abstract Syntax Tree):
rate(
  selector: http_requests_total{app="payment-service"}
  range: 5m
)

STEP 2: Load Time Series
──────────────────────────────────────
Find all series matching:
http_requests_total{app="payment-service", ...}

Result: 50 time series (different pods, labels)

STEP 3: Load Data Points
──────────────────────────────────────
For each series, load last 5 minutes of data
- From in-memory head block
- From persisted blocks if needed

STEP 4: Apply Function
──────────────────────────────────────
Calculate rate() for each series:
(value_now - value_5min_ago) / 300 seconds

STEP 5: Return Results
──────────────────────────────────────
50 time series with calculated rates
```

**Query Optimization:**
```
Inefficient Query:
sum(rate(http_requests_total[5m]))

Better:
sum(rate(http_requests_total[5m])) without (instance, pod)

Best (if supported):
sum(rate(http_requests_total[5m])) by (app, method)

Why?
- Reduces cardinality
- Faster aggregation
- Less memory usage
```

#### 5. Alerting System

**Alert Rule Evaluation:**

```yaml
# alerting_rules.yml
groups:
- name: payment_service_alerts
  interval: 30s  # Evaluate every 30 seconds
  rules:
  - alert: HighErrorRate
    expr: |
      (
        sum(rate(http_requests_total{app="payment-service",status=~"5.."}[5m]))
        /
        sum(rate(http_requests_total{app="payment-service"}[5m]))
      ) > 0.05
    for: 5m
    labels:
      severity: critical
      service: payment
    annotations:
      summary: "High error rate in payment service"
      description: "Error rate is {{ $value | humanizePercentage }} (threshold: 5%)"
```

**Alert Lifecycle:**

```
STEP 1: Inactive
──────────────────────────────────────
No condition met
State: Inactive

STEP 2: Pending
──────────────────────────────────────
Condition met, waiting for "for" duration
State: Pending
Duration: 0s / 5m

STEP 3: Firing
──────────────────────────────────────
Condition met for 5 minutes
State: Firing
Sent to: Alertmanager

STEP 4: Resolved
──────────────────────────────────────
Condition no longer met
State: Resolved
Notification: Sent to Alertmanager
```

**Alert Flow to Notification:**

```
Prometheus evaluates rule every 30s
         ↓
   Condition TRUE?
         ↓
    Wait 5 minutes (for: 5m)
         ↓
  Still TRUE? → Fire alert
         ↓
   Send to Alertmanager
         ↓
  Alertmanager receives alert
         ↓
   Deduplicates (if multiple Prometheus instances)
         ↓
   Groups related alerts
         ↓
   Routes based on labels
         ↓
   Sends to notification channel(s)
         ↓
  Slack / Email / PagerDuty
```

---

## Time-Series Database Internals

### What is Time-Series Data?

**Definition:**
Data points indexed by time, each consisting of:
- Timestamp
- Metric name
- Labels (dimensions)
- Value

**Example:**
```
Series 1:
cpu_usage{host="server-01", core="0"} 
  @14:30:00 → 45.5
  @14:30:15 → 47.2
  @14:30:30 → 46.8
  @14:30:45 → 48.1

Series 2:
cpu_usage{host="server-01", core="1"}
  @14:30:00 → 52.3
  @14:30:15 → 54.1
  @14:30:30 → 53.7
  @14:30:45 → 55.2
```

### Why Specialized TSDB?

**Traditional Database (PostgreSQL):**
```sql
CREATE TABLE metrics (
  timestamp BIGINT,
  metric_name VARCHAR,
  labels JSONB,
  value DOUBLE PRECISION
);

INSERT INTO metrics VALUES
  (1610000000, 'cpu_usage', '{"host":"server-01"}', 45.5),
  (1610000015, 'cpu_usage', '{"host":"server-01"}', 47.2),
  ...

-- Query: Average CPU over last hour
SELECT AVG(value) 
FROM metrics 
WHERE metric_name = 'cpu_usage' 
  AND timestamp > NOW() - INTERVAL '1 hour';
```

**Problems:**
```
❌ Slow writes (1M+ samples per second)
❌ Slow queries (scanning millions of rows)
❌ Huge storage (no compression)
❌ No time-series optimizations
❌ Difficult to scale
```

**Time-Series Database (Prometheus):**
```
✅ Optimized for writes (append-only)
✅ Fast range queries
✅ Excellent compression (50-70% savings)
✅ Time-based partitioning (2-hour blocks)
✅ Efficient indexing
✅ Built-in downsampling
```

### Prometheus TSDB Structure

**Directory Layout:**
```
/var/lib/prometheus/
├── chunks_head/          ← Current head block (in-memory)
│   └── 000001
├── wal/                  ← Write-Ahead Log
│   ├── checkpoint.000005
│   ├── 000006
│   └── 000007
├── 01FQWE8N.../ ←  Block (2 hours of data)
│   ├── chunks/
│   │   ├── 000001
│   │   └── 000002
│   ├── index
│   ├── meta.json
│   └── tombstones
├── 01FQXH9P.../         ← Another block
├── 01FQZJ2R.../         ← Compacted block (12 hours)
└── queries.active       ← Active queries tracking
```

**Sample Metric Storage Flow:**
```
1. Scrape happens:
   http_requests_total{method="GET"} 1547 @14:30:00

2. Written to WAL (disk):
   /wal/000007
   [timestamp][metric_hash][labels_hash][value]

3. Added to Head Block (memory):
   Time Series ID: 12345
   Latest value: 1547
   Chunk buffer: [1540, 1542, 1545, 1547]

4. After 2 hours:
   Head block persisted to disk
   New block created: 01FQWE8N.../
   
5. After 12 hours:
   Six 2-hour blocks compacted
   New compacted block: 01FQZJ2R.../
```

### Compression Techniques

**Delta-of-Deltas Encoding:**
```
Original timestamps (Unix ms):
1610000000000
1610000015000 → Delta: 15000
1610000030000 → Delta: 15000 → Delta-of-delta: 0
1610000045000 → Delta: 15000 → Delta-of-delta: 0
1610000060000 → Delta: 15000 → Delta-of-delta: 0

Storage:
- Base timestamp: 1610000000000 (8 bytes)
- Delta: 15000 (2 bytes)
- Delta-of-deltas: 0, 0, 0 (1 bit each)

Savings: From 40 bytes to 11 bytes (72% reduction)
```

**XOR Compression for Values:**
```
Original values (float64):
45.5
47.2  → XOR with previous
46.8  → XOR with previous
48.1  → XOR with previous

Float patterns typically have many leading/trailing zeros
XOR compression exploits this pattern
Typical compression: 1.5 bytes per sample
```

### Indexing

**Inverted Index:**
```
Purpose: Fast lookup of time series by labels

Example:
Series 1: http_requests{method="GET", status="200"}
Series 2: http_requests{method="POST", status="200"}
Series 3: http_requests{method="GET", status="500"}

Index Structure:
__name__ = http_requests → [Series 1, 2, 3]
method = GET             → [Series 1, 3]
method = POST            → [Series 2]
status = 200             → [Series 1, 2]
status = 500             → [Series 3]

Query: http_requests{method="GET"}
Lookup: method=GET → [Series 1, 3]
Result: Fast (no scanning needed)
```

### Retention and Deletion

**Retention Configuration:**
```yaml
# prometheus.yml
global:
  scrape_interval: 15s

storage:
  tsdb:
    retention.time: 15d    # Keep data for 15 days
    retention.size: 50GB   # Or until 50GB reached
```

**How Deletion Works:**
```
1. Old blocks identified (> 15 days)
2. Entire block deleted (atomic operation)
3. No partial deletions (blocks are immutable)
4. Deletion happens during compaction

Example:
Day 1-15:  Keep
Day 16:    Delete oldest 2-hour block
Day 17:    Delete next oldest block
...

Storage stays relatively constant
```

### Storage Sizing

**Calculate Storage Needs:**
```
Formula:
storage_bytes = scraped_samples_per_second × retention_seconds × bytes_per_sample

Example Calculation:
- 100 applications × 100 metrics each = 10,000 time series
- Scrape interval: 15 seconds = 4 times per minute
- Samples per second: 10,000 × 4 / 60 = 667 samples/sec
- Bytes per sample: ~1.5 bytes (with compression)
- Retention: 15 days = 1,296,000 seconds

Storage = 667 × 1,296,000 × 1.5
        = 1.3 billion samples
        ≈ 1.95 GB

Plus overhead (index, WAL): ≈ 2.5 GB total
```

**Real-World Example:**
```
Small Setup (Development):
- 50 targets
- 50 metrics per target
- 15-day retention
Storage: ~500 MB

Medium Setup (Production):
- 500 targets
- 200 metrics per target
- 30-day retention
Storage: ~15 GB

Large Setup (Enterprise):
- 5,000 targets
- 500 metrics per target
- 90-day retention
Storage: ~500 GB
```

---

## Pull vs Push Models

### Pull Model (Prometheus Default)

**How It Works:**
```
┌──────────────┐         ┌──────────────┐
│  Prometheus  │         │    Target    │
│    Server    │         │ Application  │
│              │         │              │
│              │─ GET ──→│  /metrics    │
│              │         │   endpoint   │
│              │←metrics─│              │
└──────────────┘         └──────────────┘

Every 15 seconds (configurable)
```

**Detailed Flow:**
```
STEP 1: Prometheus initiates connection
────────────────────────────────────────────
Prometheus: "I want your metrics"
Target: Passive, waiting

STEP 2: HTTP GET Request
────────────────────────────────────────────
GET http://target:8080/metrics HTTP/1.1

STEP 3: Target responds
────────────────────────────────────────────
HTTP/1.1 200 OK
Content-Type: text/plain

# HELP http_requests_total Total requests
# TYPE http_requests_total counter
http_requests_total{method="GET"} 1547

STEP 4: Prometheus stores data
────────────────────────────────────────────
Timestamp: 14:30:00 (Prometheus' clock)
Metric: http_requests_total
Labels: {method="GET", instance="target:8080"}
Value: 1547
```

**Advantages:**

**1. Centralized Control**
```
Prometheus decides:
- When to scrape (interval)
- What to scrape (targets)
- How often (frequency)

Target doesn't need to know:
- Where Prometheus is
- How to connect to it
- When to send data

Example:
# Change scrape interval globally
scrape_configs:
  - job_name: 'apps'
    scrape_interval: 30s  # All targets now 30s
```

**2. Health Detection**
```
Target down? Prometheus knows immediately!

Scrape attempt:
14:30:00 → Success (up=1)
14:30:15 → Timeout (up=0) ← Detected!
14:30:30 → Timeout (up=0)

Automatic metric:
up{job="app", instance="target:8080"} 0

Alert can fire:
alert: TargetDown
expr: up == 0
```

**3. Service Discovery Integration**
```
Kubernetes Example:

New Pod Created:
payment-pod-xyz @ 10.244.1.5

Prometheus automatically:
1. Discovers via K8s API (within seconds)
2. Adds to scrape targets
3. Starts scraping
4. No configuration change needed!

Pod Deleted:
1. Prometheus detects (404/timeout)
2. Removes from targets
3. Stops scraping
4. Marks as down (up=0)
```

**4. Network Security**
```
Firewall Rules:
Target: Allow inbound on :8080 from Prometheus
Prometheus: Allow outbound to targets

Target doesn't need:
- Outbound internet access
- Knowledge of Prometheus location
- Authentication to push
```

**5. Consistency**
```
Same timestamp for all metrics from one scrape:

Single scrape at 14:30:00.000:
http_requests_total 1547 @14:30:00.000
cpu_usage 45.5 @14:30:00.000
memory_usage 524288000 @14:30:00.000

All metrics aligned to same timestamp
Easier to correlate and calculate
```

**Disadvantages:**

**1. Network Requirements**
```
❌ Prometheus must reach targets
   - Requires network connectivity
   - Firewall rules needed
   - NAT traversal issues

❌ Problems with:
   - Targets behind corporate firewall
   - Cloud-to-on-premise monitoring
   - Isolated networks
```

**2. Short-lived Jobs**
```
❌ Batch Job Example:
   Start: 14:30:00
   End: 14:30:05 (5 seconds duration)
   
   Prometheus scrape: 14:30:15
   Result: Job already finished! No metrics collected.
   
❌ Cron job runs for 10 seconds every hour
   - 99.5% of time: Job not running
   - Prometheus scrapes: Nothing to scrape
```

**3. Scaling Challenges**
```
❌ Single Prometheus limits:
   - ~10,000 targets
   - ~1M active time series
   - Beyond this: Need federation/sharding
```

### Push Model (Pushgateway)

**How It Works:**
```
┌──────────────┐      ┌──────────────┐     ┌──────────────┐
│  Batch Job   │      │  Pushgateway │     │  Prometheus  │
│              │      │              │     │              │
│              │─push→│              │←GET─│              │
│              │      │  :9091       │     │              │
└──────────────┘      └──────────────┘     └──────────────┘

Job pushes → Pushgateway stores → Prometheus scrapes
```

**Detailed Flow:**
```
STEP 1: Job completes, pushes metrics
────────────────────────────────────────────
Batch Job (backup script):
Start: 02:00:00
Backup: 500 GB
End: 02:15:00
Duration: 900 seconds

curl -X POST http://pushgateway:9091/metrics/job/backup \
  --data 'backup_duration_seconds 900
backup_size_bytes 536870912000
backup_status{result="success"} 1'

STEP 2: Pushgateway stores metrics
────────────────────────────────────────────
Pushgateway maintains in-memory store:
{job="backup"}: 
  backup_duration_seconds 900
  backup_size_bytes 536870912000
  backup_status{result="success"} 1

Metrics persist until:
- Overwritten by new push
- Manually deleted
- Pushgateway restarts

STEP 3: Prometheus scrapes Pushgateway
────────────────────────────────────────────
Every 15 seconds:
GET http://pushgateway:9091/metrics

Returns all stored metrics from all jobs

STEP 4: Metrics in Prometheus
────────────────────────────────────────────
backup_duration_seconds{job="backup", instance="pushgateway:9091"} 900
backup_size_bytes{job="backup", instance="pushgateway:9091"} 536870912000
backup_status{job="backup",result="success", instance="pushgateway:9091"} 1
```

**Use Cases:**

**1. Batch Jobs**
```bash
#!/bin/bash
# database-backup.sh

# Start timer
start_time=$(date +%s)

# Perform backup
pg_dump mydb > /backups/mydb.sql
backup_result=$?

# Calculate duration
end_time=$(date +%s)
duration=$((end_time - start_time))

# Get backup size
size=$(stat -f%z /backups/mydb.sql)

# Push metrics to Pushgateway
cat <<EOF | curl --data-binary @- http://pushgateway:9091/metrics/job/backup/instance/$(hostname)
# TYPE backup_duration_seconds gauge
backup_duration_seconds $duration
# TYPE backup_size_bytes gauge
backup_size_bytes $size
# TYPE backup_status gauge
backup_status{result="$([ $backup_result -eq 0 ] && echo success || echo failure)"} 1
# TYPE backup_timestamp gauge
backup_timestamp $(date +%s)
EOF
```

**2. Serverless Functions**
```javascript
// AWS Lambda function
const axios = require('axios');

exports.handler = async (event) => {
  const startTime = Date.now();
  
  try {
    // Process event
    const result = await processData(event);
    
    // Calculate metrics
    const duration = (Date.now() - startTime) / 1000;
    
    // Push to Pushgateway
    await pushMetrics({
      job: 'lambda_processor',
      instance: process.env.AWS_LAMBDA_LOG_STREAM_NAME,
      metrics: [
        `function_duration_seconds ${duration}`,
        `function_invocations_total{result="success"} 1`,
        `function_items_processed ${result.count}`
      ]
    });
    
    return result;
    
  } catch (error) {
    await pushMetrics({
      job: 'lambda_processor',
      metrics: [
        `function_invocations_total{result="failure"} 1`
      ]
    });
    throw error;
  }
};

async function pushMetrics({job, instance, metrics}) {
  const url = `http://pushgateway:9091/metrics/job/${job}/instance/${instance}`;
  await axios.post(url, metrics.join('\n'));
}
```

**3. Firewall Scenarios**
```
Corporate Network (strict firewall):

Internet ←─────────[Firewall]─────────→ Internal Network
                   Blocks inbound           ↓
                   Allows outbound    Prometheus + Apps

Problem with Pull:
- Prometheus inside corporate network
- Targets in cloud (AWS/GCP)
- Firewall blocks Prometheus accessing cloud

Solution with Push:
- Cloud targets push to Pushgateway
- Pushgateway inside corporate network
- Prometheus scrapes Pushgateway locally
```

**Disadvantages:**

**1. Metrics Persist After Job Ends**
```
❌ Problem:
Day 1: Backup job runs, pushes metrics
       backup_status{result="success"} 1

Day 2: Backup job FAILS TO RUN (server down)
       Pushgateway still shows: backup_status{result="success"} 1
       
Alert doesn't fire! Stale metrics show success!

Solution: Add timestamp metric
backup_timestamp 1610000000
Alert: If backup_timestamp > 25 hours old, alert
```

**2. Single Point of Failure**
```
❌ Pushgateway down = All batch job metrics lost

Mitigation:
- Run multiple Pushgateway instances
- Use DNS round-robin or load balancer
- Jobs push to multiple Pushgateways
```

**3. No Health Detection**
```
❌ Can't detect if job is "down"
   Job might not be running at all
   Prometheus only knows if Pushgateway is down
   
Pull model: up{instance="target"} 0 ← Clear!
Push model: ??? No indication
```

**4. Timestamp Issues**
```
❌ Job pushes at 14:30:00
   Prometheus scrapes Pushgateway at 14:30:15
   Metrics timestamped: 14:30:15 (Prometheus' scrape time)
   
   Original event time lost!
   
Pull model: All metrics from one scrape = same timestamp
Push model: Metrics from different times = same timestamp (confusing!)
```

### Comparison Table

```
┌────────────────────┬─────────────────────┬─────────────────────┐
│ Aspect             │ Pull Model          │ Push Model          │
├────────────────────┼─────────────────────┼─────────────────────┤
│ Control            │ Prometheus          │ Target              │
│ Network Direction  │ Prom → Target       │ Target → Gateway    │
│ Health Detection   │ Automatic (up)      │ Manual              │
│ Service Discovery  │ Native support      │ Limited             │
│ Short-lived jobs   │ Difficult           │ Easy                │
│ Firewall friendly  │ No                  │ Yes                 │
│ Timestamp accuracy │ Consistent          │ Can be inconsistent │
│ Single point fail  │ No (targets indep.) │ Yes (Pushgateway)   │
│ Stale metrics      │ Cleared on timeout  │ Persist indefinitely│
│ Best for           │ Long-running apps   │ Batch jobs          │
└────────────────────┴─────────────────────┴─────────────────────┘
```

### Best Practices

**When to Use Pull:**
```
✅ Long-running applications (web servers, APIs, databases)
✅ Applications in same network as Prometheus
✅ When health detection is important
✅ Microservices in Kubernetes
✅ Standard monitoring setup
```

**When to Use Push:**
```
✅ Batch jobs (backups, ETL, reports)
✅ Serverless functions (AWS Lambda, Google Cloud Functions)
✅ Behind strict firewalls
✅ Jobs that run < scrape interval
✅ Ephemeral containers (job completion)
```

**Hybrid Approach:**
```
Example Architecture:

┌──────────────────────────────────────────────┐
│ Long-running Apps (Pull)                     │
│ ┌──────────┐  ┌──────────┐  ┌──────────┐     │
│ │ Web API  │  │ Database │  │  Redis   │     │
│ └────┬─────┘  └────┬─────┘  └────┬─────┘     │
│      │             │             │           │
│      └─────────────┴─────────────┘           │
│                    ↓                         │
│              Prometheus ←─────────┐          │
│                                   │          │
│ Batch Jobs (Push)                 │          │
│ ┌──────────┐  ┌──────────┐        │          │
│ │  Backup  │  │   ETL    │        │          │
│ └────┬─────┘  └────┬─────┘        │          │
│      │             │              │          │
│      └─────────────┘              │          │
│             ↓                     │          │
│       Pushgateway ────────────────┘          │
└──────────────────────────────────────────────┘

Best of both worlds!
```

---

## Exporters Ecosystem

### What Are Exporters?

**Definition:**
Exporters are small programs that:
1. Collect metrics from third-party systems
2. Transform them to Prometheus format
3. Expose them on HTTP `/metrics` endpoint
4. Wait for Prometheus to scrape

**Why Needed:**
```
Problem: Many systems don't natively expose Prometheus metrics
- MySQL database
- Nginx web server
- Redis cache
- Hardware (CPU, disk, network)

Solution: Exporter bridges the gap
System → Exporter → Prometheus format → Prometheus scrapes
```

### Official Exporters

#### 1. Node Exporter (Hardware/OS Metrics)

**What It Monitors:**
```
Hardware:
- CPU usage, load average
- Memory (used, free, cached)
- Disk I/O, space, inode usage
- Network traffic (bytes, packets, errors)
- Filesystem stats

Operating System:
- Process counts
- Context switches
- Interrupts
- System load
- Time since boot
```

**Installation:**
```bash
# Download
wget https://github.com/prometheus/node_exporter/releases/download/v1.7.0/node_exporter-1.7.0.linux-amd64.tar.gz

# Extract
tar xvfz node_exporter-1.7.0.linux-amd64.tar.gz

# Run
cd node_exporter-1.7.0.linux-amd64
./node_exporter

# Runs on port 9100
# Metrics at: http://localhost:9100/metrics
```

**Systemd Service:**
```ini
# /etc/systemd/system/node_exporter.service
[Unit]
Description=Node Exporter
After=network.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter \
  --collector.filesystem.mount-points-exclude=^/(dev|proc|sys|run)($|/)

[Install]
WantedBy=multi-user.target
```

```bash
# Enable and start
sudo systemctl daemon-reload
sudo systemctl enable node_exporter
sudo systemctl start node_exporter
```

**Sample Metrics:**
```
# CPU
node_cpu_seconds_total{cpu="0",mode="idle"} 124589.50
node_cpu_seconds_total{cpu="0",mode="user"} 5234.12
node_cpu_seconds_total{cpu="0",mode="system"} 2156.78

# Memory
node_memory_MemTotal_bytes 16777216000
node_memory_MemFree_bytes 2147483648
node_memory_MemAvailable_bytes 8589934592

# Disk
node_disk_read_bytes_total{device="sda"} 536870912000
node_disk_written_bytes_total{device="sda"} 268435456000
node_filesystem_avail_bytes{mountpoint="/"} 107374182400

# Network
node_network_receive_bytes_total{device="eth0"} 1073741824000
node_network_transmit_bytes_total{device="eth0"} 536870912000
```

**Prometheus Configuration:**
```yaml
scrape_configs:
  - job_name: 'node'
    static_configs:
    - targets:
      - 'server1:9100'
      - 'server2:9100'
      - 'server3:9100'
```

#### 2. cAdvisor (Container Metrics)

**What It Monitors:**
```
Per Container:
- CPU usage (user, system)
- Memory usage (working set, RSS)
- Network I/O (bytes, packets)
- Disk I/O (reads, writes)
- Filesystem usage

Kubernetes Specific:
- Pod metrics
- Container restarts
- Resource limits/requests
```

**Docker Run:**
```bash
docker run \
  --volume=/:/rootfs:ro \
  --volume=/var/run:/var/run:ro \
  --volume=/sys:/sys:ro \
  --volume=/var/lib/docker/:/var/lib/docker:ro \
  --volume=/dev/disk/:/dev/disk:ro \
  --publish=8080:8080 \
  --detach=true \
  --name=cadvisor \
  --privileged \
  google/cadvisor:latest

# Metrics at: http://localhost:8080/metrics
```

**Kubernetes DaemonSet:**
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: cadvisor
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: cadvisor
  template:
    metadata:
      labels:
        app: cadvisor
    spec:
      containers:
      - name: cadvisor
        image: gcr.io/cadvisor/cadvisor:latest
        ports:
        - containerPort: 8080
          name: http
        volumeMounts:
        - name: rootfs
          mountPath: /rootfs
          readOnly: true
        - name: var-run
          mountPath: /var/run
          readOnly: true
        - name: sys
          mountPath: /sys
          readOnly: true
        - name: docker
          mountPath: /var/lib/docker
          readOnly: true
      volumes:
      - name: rootfs
        hostPath:
          path: /
      - name: var-run
        hostPath:
          path: /var/run
      - name: sys
        hostPath:
          path: /sys
      - name: docker
        hostPath:
          path: /var/lib/docker
```

**Sample Metrics:**
```
# Container CPU
container_cpu_usage_seconds_total{
  container="payment-service",
  pod="payment-pod-abc",
  namespace="production"
} 1234.56

# Container Memory
container_memory_working_set_bytes{
  container="payment-service",
  pod="payment-pod-abc"
} 524288000

# Network
container_network_receive_bytes_total{
  container="payment-service",
  pod="payment-pod-abc",
  interface="eth0"
} 1073741824
```

#### 3. MySQL Exporter

**Installation:**
```bash
# Download
wget https://github.com/prometheus/mysqld_exporter/releases/download/v0.15.1/mysqld_exporter-0.15.1.linux-amd64.tar.gz
tar xvfz mysqld_exporter-0.15.1.linux-amd64.tar.gz
cd mysqld_exporter-0.15.1.linux-amd64

# Create MySQL user for exporter
mysql -u root -p
CREATE USER 'exporter'@'localhost' IDENTIFIED BY 'password';
GRANT PROCESS, REPLICATION CLIENT, SELECT ON *.* TO 'exporter'@'localhost';
FLUSH PRIVILEGES;

# Create config file
cat > .my.cnf <<EOF
[client]
user=exporter
password=password
EOF

# Run exporter
./mysqld_exporter --config.my-cnf=".my.cnf"

# Port: 9104
```

**Sample Metrics:**
```
# Connections
mysql_global_status_threads_connected 45
mysql_global_status_max_used_connections 123

# Queries
mysql_global_status_queries 1547893
mysql_global_status_slow_queries 245

# InnoDB
mysql_global_status_innodb_buffer_pool_pages_data 12345
mysql_global_status_innodb_row_lock_waits 23
```

#### 4. Blackbox Exporter (Probing)

**What It Does:**
```
External probing:
- HTTP/HTTPS endpoints (availability, response time, status code)
- TCP connections (port availability)
- ICMP ping (host reachability)
- DNS queries (resolution time)

Use case: Monitor external services
Example: "Is example.com up? What's the response time?"
```

**Configuration:**
```yaml
# blackbox.yml
modules:
  http_2xx:
    prober: http
    http:
      valid_status_codes: [200]
      method: GET
      
  http_post_2xx:
    prober: http
    http:
      method: POST
      
  tcp_connect:
    prober: tcp
    
  icmp:
    prober: icmp
```

**Prometheus Config:**
```yaml
scrape_configs:
  - job_name: 'blackbox'
    metrics_path: /probe
    params:
      module: [http_2xx]
    static_configs:
      - targets:
        - https://example.com
        - https://api.example.com
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: blackbox-exporter:9115
```

**Sample Metrics:**
```
probe_success{instance="https://example.com"} 1
probe_duration_seconds{instance="https://example.com"} 0.245
probe_http_status_code{instance="https://example.com"} 200
probe_ssl_earliest_cert_expiry{instance="https://example.com"} 1672531200
```

### Application Instrumentation

#### Node.js Example (prom-client)

**Complete Express Application:**
```javascript
// app.js
const express = require('express');
const promClient = require('prom-client');

const app = express();
const register = new promClient.Registry();

// Enable default metrics (CPU, memory, event loop, GC)
promClient.collectDefaultMetrics({ register });

// Custom Metrics

// 1. Counter: HTTP requests
const httpRequestsTotal = new promClient.Counter({
  name: 'http_requests_total',
  help: 'Total number of HTTP requests',
  labelNames: ['method', 'route', 'status_code'],
  registers: [register]
});

// 2. Histogram: Request duration
const httpRequestDuration = new promClient.Histogram({
  name: 'http_request_duration_seconds',
  help: 'Duration of HTTP requests in seconds',
  labelNames: ['method', 'route', 'status_code'],
  buckets: [0.1, 0.3, 0.5, 0.7, 1, 3, 5, 7, 10],
  registers: [register]
});

// 3. Gauge: Active requests
const activeRequests = new promClient.Gauge({
  name: 'http_requests_active',
  help: 'Number of active HTTP requests',
  registers: [register]
});

// 4. Summary: Database query duration
const dbQueryDuration = new promClient.Summary({
  name: 'database_query_duration_seconds',
  help: 'Database query duration in seconds',
  labelNames: ['query_type'],
  percentiles: [0.5, 0.9, 0.95, 0.99],
  registers: [register]
});

// Middleware: Track all requests
app.use((req, res, next) => {
  const start = Date.now();
  activeRequests.inc();
  
  res.on('finish', () => {
    const duration = (Date.now() - start) / 1000;
    const route = req.route?.path || 'unknown';
    const labels = {
      method: req.method,
      route: route,
      status_code: res.statusCode
    };
    
    httpRequestsTotal.inc(labels);
    httpRequestDuration.observe(labels, duration);
    activeRequests.dec();
  });
  
  next();
});

// Routes
app.get('/api/users', async (req, res) => {
  const start = Date.now();
  
  try {
    const users = await db.query('SELECT * FROM users');
    
    const queryDuration = (Date.now() - start) / 1000;
    dbQueryDuration.observe({ query_type: 'select_users' }, queryDuration);
    
    res.json(users);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

app.post('/api/users', async (req, res) => {
  const start = Date.now();
  
  try {
    const user = await db.query('INSERT INTO users VALUES (?)', [req.body]);
    
    const queryDuration = (Date.now() - start) / 1000;
    dbQueryDuration.observe({ query_type: 'insert_user' }, queryDuration);
    
    res.json(user);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Metrics endpoint
app.get('/metrics', async (req, res) => {
  res.set('Content-Type', register.contentType);
  res.end(await register.metrics());
});

app.listen(3000, () => {
  console.log('Server running on port 3000');
  console.log('Metrics available at http://localhost:3000/metrics');
});
```

**Kubernetes Deployment with Annotations:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: node-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: node-app
  template:
    metadata:
      labels:
        app: node-app
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "3000"
        prometheus.io/path: "/metrics"
    spec:
      containers:
      - name: app
        image: myapp:latest
        ports:
        - containerPort: 3000
          name: http
```

### Exporter Best Practices

**1. Use Official Exporters When Available**
```
✅ MySQL Exporter for MySQL
✅ PostgreSQL Exporter for PostgreSQL
✅ Node Exporter for system metrics

❌ Don't reinvent the wheel
❌ Don't parse logs for metrics
```

**2. Instrument Applications Directly**
```
✅ Add client library to application
✅ Expose /metrics endpoint
✅ Track business metrics (orders, revenue, signups)

Example metrics to track:
- Application-specific: orders_total, revenue_dollars
- Request metrics: http_requests_total, http_request_duration
- Database metrics: db_queries_total, db_connection_pool_size
```

**3. Label Cardinality**
```
✅ Good labels (low cardinality):
- method: GET, POST, PUT, DELETE (4 values)
- status: 200, 400, 500 (few values)
- endpoint: /api/users, /api/orders (limited values)

❌ Bad labels (high cardinality):
- user_id: millions of unique values
- request_id: infinite unique values
- ip_address: thousands of unique values

Why? Each unique label combination = new time series
High cardinality = millions of time series = memory issues
```

**4. Metric Naming Conventions**
```
Format: <namespace>_<name>_<unit>

✅ Good:
http_requests_total (counter)
http_request_duration_seconds (histogram)
database_connections_active (gauge)

❌ Bad:
requests (ambiguous)
latency (no unit)
db (too generic)
```

---

## Real-World Monitoring Scenarios

### Scenario 1: Debugging Slow API Endpoint

**Problem:**
Users report: "Checkout page is slow"

**Step 1: Check Overall Metrics**
```promql
# Request rate
rate(http_requests_total{endpoint="/api/checkout"}[5m])

# Average response time
rate(http_request_duration_seconds_sum{endpoint="/api/checkout"}[5m])
/
rate(http_request_duration_seconds_count{endpoint="/api/checkout"}[5m])

Results:
- Request rate: 50 req/sec (normal)
- Average response time: 2.5 seconds (was 200ms yesterday!)
```

**Step 2: Check Percentiles**
```promql
# P95 latency
histogram_quantile(0.95, 
  rate(http_request_duration_seconds_bucket{endpoint="/api/checkout"}[5m])
)

# P99 latency
histogram_quantile(0.99, 
  rate(http_request_duration_seconds_bucket{endpoint="/api/checkout"}[5m])
)

Results:
- P50: 300ms (normal)
- P95: 5 seconds (slow!)
- P99: 10 seconds (very slow!)

Conclusion: Not all requests slow, but worst cases are very slow
```

**Step 3: Check Dependencies**
```promql
# Database query time
rate(database_query_duration_seconds_sum{query_type="get_inventory"}[5m])
/
rate(database_query_duration_seconds_count{query_type="get_inventory"}[5m])

Result: 4.5 seconds average (was 50ms yesterday!)

# Database connection pool
database_connections_active

Result: 100/100 (pool exhausted!)

Root Cause Found: Database connection pool exhausted
```

**Step 4: Check Database Metrics**
```promql
# MySQL slow queries
rate(mysql_global_status_slow_queries[5m])

Result: 45 slow queries/sec (was 0!)

# InnoDB row locks
mysql_global_status_innodb_row_lock_waits

Result: 230 (was 0)

Deeper Cause: Slow query causing locks
```

**Step 5: Fix**
```sql
-- Check slow query log
SHOW PROCESSLIST;

-- Found: SELECT * FROM inventory WHERE product_id IN (...)
-- Missing index on product_id

-- Solution:
CREATE INDEX idx_product_id ON inventory(product_id);

-- Result:
-- Query time: 4.5s → 50ms
-- Connection pool: 100/100 → 25/100
-- Checkout latency: 2.5s → 200ms
```

### Scenario 2: Auto-Scaling Based on Metrics

**Setup:**
```yaml
# Horizontal Pod Autoscaler
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: payment-service-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: payment-service
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "100"
```

**Custom Metrics:**
```javascript
// Expose requests per second
const requestsPerSecond = new promClient.Gauge({
  name: 'http_requests_per_second',
  help: 'HTTP requests per second (for HPA)',
  registers: [register]
});

// Calculate every 10 seconds
setInterval(async () => {
  const query = 'rate(http_requests_total[1m])';
  const result = await prometheus.query(query);
  requestsPerSecond.set(result.value);
}, 10000);
```

**Scaling Behavior:**
```
Timeline:

09:00 - Low traffic (50 req/sec)
        Pods: 2 (minimum)

12:00 - Lunch rush starts
        Traffic: 150 req/sec
        Per pod: 75 req/sec
        HPA: Scale to 2 pods (150/100 = 1.5, round up to 2)

12:30 - Peak traffic (500 req/sec)
        Per pod: 250 req/sec
        HPA: Scale to 5 pods (500/100 = 5)

13:00 - Traffic dropping (300 req/sec)
        Per pod: 60 req/sec
        HPA: Scale to 3 pods (300/100 = 3)

14:00 - Back to normal (100 req/sec)
        HPA: Scale to 2 pods (minimum)
```

### Scenario 3: Capacity Planning

**Goal:** Predict when to add more infrastructure

**Current State:**
```promql
# Total request rate (all services)
sum(rate(http_requests_total[1h]))
Result: 5000 req/sec

# CPU usage (all nodes)
avg(1 - rate(node_cpu_seconds_total{mode="idle"}[5m]))
Result: 65%

# Memory usage (all nodes)
avg(node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)
Result: 45% free
```

**Growth Analysis:**
```promql
# Request growth (month over month)
sum(rate(http_requests_total[1h])) 
/
sum(rate(http_requests_total[1h] offset 30d))

Result: 1.15 (15% monthly growth)

# Projected in 6 months:
Current: 5000 req/sec
Growth: 1.15^6 = 2.31x
Projected: 11,550 req/sec
```

**Capacity Calculation:**
```
Current Infrastructure:
- 10 nodes
- 5000 req/sec at 65% CPU

At 80% CPU (maximum sustainable):
- Capacity: 5000 / 0.65 * 0.80 = 6154 req/sec

Projected need (6 months):
- 11,550 req/sec

Required nodes:
- 11,550 / (6154/10) = 18.8 nodes
- Round up: 20 nodes

Action: Plan to add 10 nodes over next 6 months
```

### Scenario 4: SLO Tracking

**SLO Definition:**
```
Service Level Objective:
- 99.9% of requests < 500ms response time
- Measured over 30 days

Error Budget:
- 0.1% of requests can be slow
- At 10,000 req/day = 10 requests can be >500ms
```

**PromQL Queries:**
```promql
# Total requests
sum(increase(http_request_duration_seconds_count[30d]))

# Slow requests (>500ms)
sum(increase(http_request_duration_seconds_bucket{le="0.5"}[30d]))

# Calculate SLO
(
  sum(increase(http_request_duration_seconds_bucket{le="0.5"}[30d]))
  /
  sum(increase(http_request_duration_seconds_count[30d]))
) * 100

Result: 99.92% (meeting SLO! ✅)

# Error budget remaining
100 - (
  (
    sum(increase(http_request_duration_seconds_bucket{le="0.5"}[30d]))
    /
    sum(increase(http_request_duration_seconds_count[30d]))
  ) * 100
)

Result: 0.08% of 0.1% budget used (80% remaining)
```

**Alert on SLO Burn:**
```yaml
groups:
- name: slo_alerts
  rules:
  - alert: SLOBurnRateHigh
    expr: |
      (
        sum(rate(http_request_duration_seconds_bucket{le="0.5"}[1h]))
        /
        sum(rate(http_request_duration_seconds_count[1h]))
      ) < 0.999
    for: 5m
    annotations:
      summary: "SLO burn rate too high"
      description: "Current SLO: {{ $value | humanizePercentage }}"
```

---

## Quick Reference

### Core Concepts
```
Observability = Understand system from external outputs
Three Pillars = Metrics + Logs + Traces
Prometheus = Pull-based metrics monitoring
TSDB = Optimized for time-series data
Exporters = Bridge between systems and Prometheus
```

### When to Use What
```
METRICS:  Dashboards, alerts, trends, SLOs
LOGS:     Debugging, audit trails, security events
TRACES:   Microservices debugging, bottlenecks
```

### Architecture Components
```
Prometheus Server:    Scrapes, stores, queries
Service Discovery:    Auto-finds targets
Exporters:           Expose metrics from systems
Alertmanager:        Routes notifications
TSDB:                Stores time-series data
```

### Pull vs Push
```
PULL:  Long-running apps, Kubernetes, standard setup
PUSH:  Batch jobs, serverless, behind firewalls
```

### Common Exporters
```
Node Exporter    :9100  → System metrics
cAdvisor         :8080  → Container metrics
MySQL Exporter   :9104  → MySQL metrics
Blackbox         :9115  → HTTP/TCP probes
```

---

This completes Core Concepts. You now understand observability fundamentals, Prometheus architecture, time-series databases, pull vs push models, exporters, and real-world monitoring scenarios.

