# ADR-005: OpenTelemetry as the observability backbone

- Status: Accepted
- Date: 2026-09-17
- Deciders: Mark Miller

## Problem

Five processes, a broker, and an RPC provider. "Check the logs" does
not explain a 2.4s p95 on `GET /v1/portfolio/{address}`. We need one
join key across HTTP, Kafka, and SQL.

## Options considered

### 1. Logs only (JSON to stdout)

Necessary, not sufficient. Cannot reconstruct a graph of a request
that touched three services and a consumer.

### 2. Prometheus metrics without traces

Great for "is it on fire" and RED/USE. Bad for "which RPC provider
attempt caused this retry".

### 3. Vendor APM agent per language

Fast to start, expensive to leave, and Atlas is a lab - tying the
design to one SaaS is the wrong lesson.

### 4. OpenTelemetry ? collector ? Prometheus + traces backend

Instrumentation is standard. The backend can be Grafana Tempo, Jaeger,
or a vendor later. Metrics stay Prometheus. Logs stay stdout JSON
with `trace_id` injected.

## Decision

- Every service uses OpenTelemetry SDK (Python).
- W3C `traceparent` on HTTP. Kafka headers carry the same context.
- Collector in `infrastructure` scrapes OTLP.
- Golden signals per service: request rate, error rate, duration,
  plus **queue lag** and **RPC provider error rate**.
- SLOs are written in `docs/scaling.md`. They are targets, not
  decorations.

Logs without `correlation_id` / `trace_id` are a bug.

## Trade-offs

- **Accepted:** collector is another container locally.
- **Accepted:** we do not run a full Grafana Cloud replica. Local
  Prometheus + Grafana is enough to show the shape.
- **Rejected:** a unique tracing format per service.
- **Rejected:** 40 README badges as a substitute for one trace.
