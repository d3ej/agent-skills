---
name: observability-and-monitoring
description: Designs and implements the three pillars of observability — metrics, logs, and traces — alongside alerting strategies and SLO-based monitoring. Use when adding instrumentation to a new service, auditing an existing system's observability, designing an alerting strategy, or building runbooks tied to observable signals.
---

# Observability and Monitoring

## Overview

Observability is the degree to which you can understand a system's internal state from its external outputs. A fully observable system lets you ask and answer novel questions about its behavior in production — without deploying new code.

The three pillars are complementary: metrics tell you *something* is wrong, logs tell you *what* happened, and traces tell you *where* and *why*.

## When to Use

- Instrumenting a new service for production
- Auditing an existing service with blind spots (alerts that never fire, dashboards nobody trusts)
- Designing or reviewing an SLO (Service Level Objective)
- Building runbooks that reference observable signals
- Diagnosing production incidents with insufficient signal

## Process

### 1. Define SLIs and SLOs Before Instrumenting

**Service Level Indicator (SLI):** A quantitative measure of service behavior.
**Service Level Objective (SLO):** A target value or range for an SLI.
**Error Budget:** 100% − SLO; the acceptable amount of unreliability per window.

Common SLIs by service type:
| Service type | SLI candidates |
|---|---|
| Request-serving | Availability (success rate), latency (p95/p99), error rate |
| Data pipeline | Freshness (time since last successful run), completeness (% records processed) |
| Storage | Durability, read/write latency, error rate |
| Scheduled job | Success rate, run duration, missed schedule |

Write the SLO before writing the code. Example:
> "99.9% of API requests complete successfully (non-5xx) over a rolling 30-day window."

### 2. Metrics

**Instrument with the RED method for services:**
- **R**ate — requests per second
- **E**rrors — error rate (absolute count + percentage)
- **D**uration — p50, p95, p99 latency

**Instrument with the USE method for resources:**
- **U**tilization — % of capacity in use
- **S**aturation — queue depth, pending work
- **E**rrors — error events

**Metric naming conventions (Prometheus):**
```
# Pattern: <service>_<subsystem>_<name>_<unit>
http_server_requests_total           # counter
http_server_request_duration_seconds # histogram
db_connection_pool_active            # gauge
queue_messages_pending               # gauge
```

- Counters: monotonically increasing, use `_total` suffix
- Histograms: for latency and size distributions; always include buckets that cover your SLO boundary
- Gauges: for values that go up and down (queue depth, active connections, cache size)

**Cardinality discipline:**
- Labels should have bounded cardinality — never use user IDs, request IDs, or unbounded strings as labels
- High cardinality → use traces, not metrics

### 3. Logs

**Structured logging format (JSON):**
```json
{
  "timestamp": "2026-04-20T12:00:00.000Z",
  "level": "info",
  "service": "order-service",
  "version": "1.4.2",
  "trace_id": "abc123",
  "span_id": "def456",
  "message": "Order placed successfully",
  "order_id": "ord_789",
  "user_id": "usr_012",
  "duration_ms": 43
}
```

**Log level discipline:**
| Level | When to use |
|-------|-------------|
| DEBUG | Detailed diagnostic data; disabled in production by default |
| INFO | Significant operational events (service started, request processed, job completed) |
| WARN | Unexpected but handled state; service is degraded but still working |
| ERROR | Unhandled error; requires human investigation |
| FATAL | Service cannot continue; immediate alert warranted |

**What not to log:**
- Passwords, tokens, secrets, or credentials (even hashed)
- Full request/response bodies (log schema only; add sampling for debugging)
- PII without explicit legal basis and masking
- High-frequency low-signal events at INFO level (degrades signal-to-noise ratio)

**Correlation:**
- Propagate `trace_id` from the incoming request into every log line for that request
- Include `correlation_id` for async flows where a single user action spans multiple services

### 4. Traces

**Distributed tracing concepts:**
- **Trace:** The end-to-end journey of a request across services
- **Span:** A single operation within a trace (HTTP call, DB query, cache lookup)
- **Context propagation:** Passing trace context via HTTP headers (`traceparent`), message metadata, or gRPC metadata

**OpenTelemetry instrumentation (recommended):**
```python
from opentelemetry import trace

tracer = trace.get_tracer("order-service")

with tracer.start_as_current_span("process_order") as span:
    span.set_attribute("order.id", order_id)
    span.set_attribute("order.total", total)
    result = process(order)
    span.set_attribute("order.status", result.status)
```

**What to instrument:**
- Every inbound request (auto-instrumented by most frameworks)
- Every outbound HTTP call
- Every database query
- Every cache operation (hit/miss/set)
- Every message queue publish/consume

**Sampling strategy:**
- 100% sampling for error traces (never drop errors)
- Probabilistic sampling (1–10%) for normal traffic in high-volume systems
- Head-based sampling: decide at trace start
- Tail-based sampling: decide at trace completion (more accurate, more complex)

### 5. Alerting

**Alert on symptoms, not causes:**
- Alert: "Error rate > 1% for 5 minutes" (user-facing symptom)
- Don't alert: "CPU > 80%" (cause — investigate, don't page)

**SLO-based alerting (preferred over threshold-based):**
Use burn rate alerts to detect SLO violations before the error budget is exhausted:
```
Burn rate = (error rate now) / (1 - SLO)
```
- Fast burn alert (1-hour window, 14× burn rate): catches outages quickly
- Slow burn alert (6-hour window, 5× burn rate): catches sustained degradation

**Alert anatomy — every alert must have:**
```yaml
alert: HighErrorRate
expr: rate(http_requests_total{status=~"5.."}[5m]) / rate(http_requests_total[5m]) > 0.01
for: 5m
labels:
  severity: page
annotations:
  summary: "{{ $labels.service }} error rate above 1%"
  runbook: "https://wiki.example.com/runbooks/high-error-rate"
  dashboard: "https://grafana.example.com/d/abc/service-dashboard"
```

**Alert routing:**
- Page (PagerDuty/OpsGenie): SLO breach, data pipeline down, security event
- Ticket (Jira/Linear): trending degradation, non-critical capacity warning
- Slack notification: deployment completed, circuit breaker opened (informational)

### 6. Dashboards

**Golden signals dashboard (per service):**
1. Request rate (RPS)
2. Error rate (% and absolute count)
3. Latency (p50, p95, p99 with SLO line overlaid)
4. Saturation (CPU, memory, connection pool, queue depth)

**Dashboard discipline:**
- Every graph should have a threshold line for the SLO target
- Red = SLO breach, yellow = approaching threshold, green = healthy
- Include a link to the relevant runbook in the dashboard header
- Review dashboards in post-mortems; add panels for blind spots discovered during incidents

## Common Rationalizations

- **"We'll add observability later"** — Observability added after the fact is always incomplete. Add it before going to production.
- **"Logs are enough"** — Logs alone make cross-service debugging nearly impossible at scale. Traces are essential for distributed systems.
- **"More metrics is better"** — High cardinality and metric sprawl degrade query performance and increase cost. Instrument purposefully.
- **"We alert on everything"** — Alert fatigue causes real pages to be ignored. Alert only on user-facing symptoms; the rest goes to dashboards.

## Red Flags

- Services with no `/metrics` endpoint or equivalent
- Log lines without structured fields (plain text logs)
- No `trace_id` propagation across service calls
- Alerts with no runbook link
- Dashboards that nobody looks at (last viewed > 30 days ago)
- Error budget tracking absent while an SLO exists on paper
- `SELECT *` in slow query logs with no index recommendation

## Verification

- [ ] SLI and SLO defined and documented for the service
- [ ] RED metrics (rate, errors, duration) instrumented and visible in dashboard
- [ ] Structured JSON logging with `trace_id` in every log line
- [ ] Distributed tracing enabled with context propagation across service boundaries
- [ ] At least one alerting rule wired to a runbook
- [ ] Error budget burn rate alerts configured for the SLO
- [ ] Dashboard reviewed and trusted (not just created)
