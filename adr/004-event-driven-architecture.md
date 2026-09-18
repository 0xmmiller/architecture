# ADR-004: Event-driven architecture

- Status: Accepted
- Date: 2026-09-17
- Deciders: Mark Miller

## Problem

Transaction submission, chain ingestion, and portfolio projection
happen at different speeds and fail independently. Coupling them
with request/response either drops facts or blocks the caller.

## Options considered

### 1. Orchestrated saga over HTTP

`tx-service` calls indexer, then portfolio, then analytics, with
compensations. Visible control flow. Every new consumer requires
changing the orchestrator. Timeouts become business logic.

### 2. Event-driven (choreography) with a transactional outbox

A service writes its DB row and an `outbox` row in the **same**
Postgres transaction. A publisher drains the outbox to Kafka.
Consumers update their own DB. No 2PC.

### 3. Dual write (DB then Kafka in the handler)

The classic bug: process dies after COMMIT and before produce, or
the other way around. Looks fine until it does not.

## Decision

Event-driven integration with a **transactional outbox** on every
service that produces facts.

Rules:

- Events are **facts in the past tense**. `IntentCreated`, not
  `CreatePortfolioRow`.
- Consumers may not reach into producer databases (ADR-002).
- Commands stay inside a service's HTTP API. The bus is not a
  command bus.
- Schema of events is documented in `docs/events.md` and versioned
  with a `schema_version` field. Additive changes only.

## Why not a command bus

Putting `ApplyFill` on Kafka invites hidden coupling: producers start
caring who consumes. Facts let analytics and portfolio evolve without
transaction-service knowing they exist.

## Trade-offs

- **Accepted:** eventual consistency on read models. The API documents
  that portfolio can lag the chain by seconds.
- **Accepted:** debugging a flow means following a `correlation_id`
  across traces, not reading one stack.
- **Rejected:** dual writes.
- **Rejected:** turning Kafka into RPC (request topics + reply topics).
