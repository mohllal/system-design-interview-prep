# Observability

Observability is the ability to understand what is happening inside a system by examining its outputs.

In practice, observability helps us answer unknown questions during incidents, debugging, and performance tuning.

## Why Observability Matters

- Detect issues early before users report them
- Reduce time to diagnose and recover (MTTR)
- Understand behavior under load and failure
- Support safer releases with measurable signals

## Monitoring vs Observability

- **Monitoring**: Tracks known failure conditions (predefined dashboards and alerts)
- **Observability**: Explores unknown issues by correlating telemetry across services

You need both: monitoring for fast detection, observability for deep diagnosis.

## Core Telemetry Signals

### Metrics

Numerical time-series data (for example, request rate, error rate, latency, CPU, queue depth).

- Best for alerting and trend analysis
- Low-cost at high scale

### Logs

Structured event records with context (request ID, user ID, service, error details).

- Best for detailed debugging and audit trails
- Should be queryable and correlated with traces/metrics

### Traces

Request-level end-to-end timing across services, with spans per hop.

- Best for finding latency bottlenecks in distributed systems
- Great for dependency and fan-out analysis

## Distributed Tracing

Distributed tracing follows one logical request across multiple services, queues, and databases.

Core concepts:

- Trace: Full request journey
- Span: One operation within that journey
- Parent/child spans: Shows call hierarchy and critical path
- Trace ID: Shared across all spans for the same request
- Span ID: Unique ID per span

### Common Standards

- W3C Trace Context: Standard header format (`traceparent`, `tracestate`) for cross-service context propagation
- W3C Baggage: Standard for lightweight key-value context propagation
- OpenTelemetry: Vendor-neutral APIs/SDKs and OTLP protocol for collecting/exporting traces
- Zipkin B3: Older but still common propagation format in legacy systems

### Typical Implementation Pattern

1. Ingress service starts a root span for each request
2. Trace context is propagated on every outbound call (HTTP/gRPC/messaging)
3. Downstream services create child spans using received context
4. Spans are exported to a collector (often OpenTelemetry Collector)
5. Collector forwards data to a tracing backend (Jaeger, Zipkin, Tempo, vendor APM)
6. Query traces by `trace_id` and correlate with logs/metrics

### Sampling

Sampling controls trace volume and cost while trying to preserve high-value diagnostic traces.

- Head sampling: Decide at request start (usually on the root span). Low overhead, but can miss rare failures if they were not sampled.
- Tail sampling: Decide after the entire trace is complete with all spans collected. Better for keeping error/slow traces, but needs more compute/memory in the collector path to buffer the entire trace.

How?

- Head sampling: Often uses probabilistic logic (e.g., sample 20% of all traces at random).
- Tail sampling: Often uses a threshold-based or count-based logic (e.g., sample all traces with latency > 100ms).

Practical starter policy:

1. Keep 100% of error traces
2. Keep 100% of high-latency traces (for example, p95/p99 threshold breaches)
3. Keep a small baseline sample of normal traces (for example, 1-10%) for trend visibility
4. Increase sampling temporarily during incidents or new releases

What to watch:

- Too little sampling hides useful context
- Too much sampling increases storage cost and query noise
- Inconsistent service-level sampling can break end-to-end trace visibility

## Other Useful Telemetry Data

- Events: Deployments, feature flag changes, schema migrations
- Profiles: CPU/memory profile samples for hot path analysis
- Crash reports: Stack traces and exception aggregates  
- Health checks: Readiness/liveness status
- Synthetics: Automated user-journey probes from outside the system

## Common Pitfalls

- High-cardinality labels causing metric cost explosions
- Logging too much noise and missing useful context
- Missing correlation IDs across services
- Alert fatigue from low-signal thresholds

## Interview Talking Points

- Observability is about answering "why" and "where" when systems fail.
- Metrics detect, traces localize, and logs explain.

## Reference Materials

- [Google SRE - Monitoring Distributed Systems](https://sre.google/sre-book/monitoring-distributed-systems/)
- [OpenTelemetry Documentation](https://opentelemetry.io/docs/)
- [The USE Method](https://www.brendangregg.com/usemethod.html)
