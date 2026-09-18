# ADR-007: Go for the notification plane

- Status: Accepted
- Date: 2026-09-18
- Deciders: Mark Miller

## Problem

Atlas already has a Python write path, read models, and HTTP edge.
Notifications are a different saturation profile: many long-lived
subscribers, fan-out from the same Kafka facts, and a requirement
that a slow webhook must not stall portfolio queries.

Rewriting platform-api in Go would be language tourism. Adding a
sixth Python service that holds thousands of SSE connections would
couple the GIL-adjacent runtime to a problem Go is boringly good at.

## Options considered

### 1. Python notification-service (FastAPI + asyncio)

Same repo conventions. Fine for tens of connections. Awkward for
fan-out + blocking webhook IO unless we add a worker fleet anyway.

### 2. Put notify into platform-api

Keeps service count at five. Mixes a public REST contract with
connection lifecycle. A notify deploy would bounce the API.

### 3. Go notification-service on the existing bus

Consumes `atlas.tx.lifecycle` and `atlas.chain.reorg`. Does not own
intents or positions. Idempotent on `event_id`. SSE + webhook sinks.
Python services stay the source of facts.

## Decision

**notification-service is Go.** Domain HTTP and read models stay
Python. The bus is the contract (ADR-001, ADR-004). Language is a
saturation choice, not a brand.

## Trade-offs

- **Accepted:** two languages in one lab. CI and Dockerfiles differ.
- **Accepted:** no shared in-process types. Event JSON is the schema.
- **Rejected:** rewriting working Python services "so the GitHub is Go".
