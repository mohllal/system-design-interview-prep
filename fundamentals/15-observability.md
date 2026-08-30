---
title: "Observability"
concepts:
  - monitoring-vs-observability
  - metrics
  - logs
  - distributed-tracing
  - trace-sampling
  - opentelemetry
  - red-and-use-methods
  - slo-burn-rate-alerting
related:
  - fundamentals/03-latency-and-throughput.md
  - fundamentals/08-availability.md
  - fundamentals/09-reliability.md
  - fundamentals/14-resilience.md
---

# Observability

Observability is the ability to understand what is happening inside a system by examining its outputs.

The practical test is whether you can answer a question you did not anticipate, using data you are already collecting, without shipping new code. That is what makes it different from having a dashboard.

## Why observability matters

- Detect issues early, before users report them
- Reduce time to diagnose and recover (MTTR), which usually dominates incident length more than time to fix
- Understand behavior under load and failure, not just at steady state
- Support safer releases by comparing measurable signals before and after a deploy

## Monitoring vs observability

- **Monitoring**: Tracks known failure conditions through predefined dashboards and alerts. Answers "is the thing I expected to break, broken?"
- **Observability**: Explores unknown issues by correlating telemetry across services. Answers "why is this request slow, for these users, only since Tuesday?"

You need both: monitoring for fast detection, observability for deep diagnosis.

## Core telemetry signals

### Metrics

Numerical time-series data (for example, request rate, error rate, latency, CPU, queue depth).

- Best for alerting and trend analysis
- Cheap to store and query at high scale, because they aggregate
- Weak at explaining a specific request, since the detail is aggregated away

### Logs

Structured event records with context (request ID, user ID, service, error details).

- Best for detailed debugging and audit trails
- Should be structured (key-value or JSON), queryable, and carry the trace ID
- Expensive at volume, which is why sampling and log levels matter

### Traces

Request-level end-to-end timing across services, with a span per hop.

- Best for finding latency bottlenecks and unexpected fan-out in distributed systems
- Show the critical path and which dependency actually owns the latency
- Expensive to keep in full, which is why sampling policy matters

The three work as a sequence: **metrics detect a problem, traces localize it to a service or dependency, and logs explain what happened there.** They are only useful together if they share correlation identifiers, so propagate a trace ID and attach it to every log line.

## Choosing what to measure

Two well-known checklists cover most cases, and answer different questions:

| Method              | Applies to                            | Measures                             |
| ------------------- | ------------------------------------- | ------------------------------------ |
| RED                 | Request-driven services and endpoints | Rate, Errors, Duration               |
| USE                 | Resources (CPU, disk, pool, queue)    | Utilization, Saturation, Errors      |
| Four golden signals | Any user-facing service               | Latency, traffic, errors, saturation |

RED tells you what users are experiencing; USE tells you which resource is the constraint. A service with a healthy RED profile and a saturated thread pool is about to have an incident, and only USE sees it coming.

For latency, always record distributions rather than averages. An average hides the tail that users actually complain about, so alert and report on p95 and p99. See [Latency and throughput](./03-latency-and-throughput.md).

## Distributed tracing

Distributed tracing follows one logical request across multiple services, queues, and databases.

Core concepts:

- **Trace**: Full request journey
- **Span**: One operation within that journey
- **Parent/child spans**: Show call hierarchy and critical path
- **Trace ID**: Shared across all spans for the same request
- **Span ID**: Unique ID per span

### Common standards

- **W3C Trace Context**: Standard header format (`traceparent`, `tracestate`) for cross-service context propagation
- **W3C Baggage**: Standard for lightweight key-value context propagation
- **OpenTelemetry**: Vendor-neutral APIs/SDKs and OTLP protocol for collecting/exporting traces
- **Zipkin B3**: Older but still common propagation format in legacy systems

### Typical implementation pattern

1. Ingress service starts a root span for each request
2. Trace context is propagated on every outbound call (HTTP/gRPC/messaging)
3. Downstream services create child spans using the received context
4. Spans are exported to a collector (often OpenTelemetry Collector)
5. Collector forwards data to a tracing backend (Jaeger, Zipkin, Tempo, vendor APM)
6. Query traces by `trace_id` and correlate with logs/metrics

Asynchronous hops are the step teams usually miss: trace context has to travel in message headers through queues and topics, or the trace ends at the producer and the consumer's work looks unrelated.

### Sampling

Sampling controls trace volume and cost while trying to preserve high-value diagnostic traces.

| Sampling type     | When it's decided              | Trade-offs                                                                           | Typical method                          |
| ----------------- | ------------------------------ | ------------------------------------------------------------------------------------ | --------------------------------------- |
| **Head sampling** | At request start (root span)   | Low overhead, but can miss rare failures                                             | Probabilistic (e.g., 20% of traces)     |
| **Tail sampling** | After the full trace completes | Better error/slow-trace capture, but needs more collector memory to buffer the trace | Threshold-based (e.g., latency > 100ms) |

Practical starter policy:

1. Keep 100% of error traces
2. Keep 100% of high-latency traces (for example, p95/p99 threshold breaches)
3. Keep a small baseline sample of normal traces (for example, 1-10%) for trend visibility
4. Increase sampling temporarily during incidents or new releases

What to watch:

- Too little sampling hides useful context
- Too much sampling increases storage cost and query noise
- Inconsistent service-level sampling can break end-to-end trace visibility, since one service dropping a span leaves a hole in the middle of every trace

## Other useful telemetry data

- **Events**: Deployments, feature flag changes, schema migrations. Overlaying these on a metrics graph answers "what changed?" faster than any other signal
- **Profiles**: CPU/memory profile samples for hot path analysis
- **Crash reports**: Stack traces and exception aggregates
- **Health checks**: Readiness/liveness status
- **Synthetics**: Automated user-journey probes from outside the system, which catch failures that internal metrics cannot see (DNS, TLS, CDN, or a totally down service emitting nothing)

## Alerting

Telemetry is only useful if the right signal reaches a human at the right time.

- **Alert on symptoms, not causes**: Page on "checkout error rate is above the SLO", not on "CPU is at 80%". High CPU with happy users is not an incident
- **Alert on SLO burn rate**: Compare how fast the [error budget](./08-availability.md) is being consumed against the rate that would exhaust it for the period. Use a fast window to catch sharp outages and a slow window to catch slow degradation
- **Page only on what is both urgent and actionable**: Everything else becomes a ticket or a dashboard
- **Attach context to the alert**: Link the runbook, the dashboard, and the relevant trace query, so the responder starts diagnosing instead of hunting

Alert fatigue is a reliability problem, not just an annoyance. A team that has learned to ignore pages will also ignore the real one.

## How telemetry ties back to reliability and resilience

Observability is not a separate concern from the previous documents; it is how their goals are measured and enforced.

- **[Availability](./08-availability.md)**: SLIs are computed from metrics, and the error budget only exists because something is counting successful versus failed requests
- **[Reliability](./09-reliability.md)**: MTTR is mostly detection plus diagnosis time, which is exactly what good telemetry shortens. Correctness failures are also invisible by default, so measure them deliberately: reconciliation drift, duplicate-processing counts, and validation rejection rates
- **[Resilience](./14-resilience.md)**: Every resilience mechanism needs its own telemetry, or you cannot tell whether it is working or hiding a problem. Instrument circuit breaker state transitions, retry counts and retry budget consumption, fallback and degraded-response rates, load-shedding counts, queue depth, and pool saturation

A circuit breaker that has been open for two hours is a silent outage unless something is watching its state. The same is true of a fallback that quietly serves stale data to every user.

## Common pitfalls

- High-cardinality labels (user ID, request ID, full URL) causing metric cost explosions
- Logging high volume with low signal, so the useful line is unfindable
- Missing correlation IDs, which makes cross-service debugging manual guesswork
- Alert fatigue from low-signal, cause-based thresholds
- Dashboards that only make sense to the person who built them, with no runbook attached
- Telemetry that shares infrastructure with the system it monitors, so it dies in the same outage

## Interview talking points

- Observability is about answering "why" and "where" when systems fail, not about having dashboards.
- Metrics detect, traces localize, logs explain. Correlation IDs are what make the three usable together.
- Name RED for services and USE for resources rather than listing signals arbitrarily.
- Alert on user-visible symptoms and SLO burn rate, not on resource thresholds.
- Mention cost explicitly: cardinality limits, log volume, and trace sampling policy are real design decisions.
- Say what you would instrument for the resilience patterns in the design, not just for the happy path.

## Reference materials

- [Google SRE - Monitoring Distributed Systems](https://sre.google/sre-book/monitoring-distributed-systems/)
- [OpenTelemetry Documentation](https://opentelemetry.io/docs/)
- [The USE Method](https://www.brendangregg.com/usemethod.html)
