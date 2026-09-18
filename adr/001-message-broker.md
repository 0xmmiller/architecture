# ADR-001: Kafka/Redpanda as the inter-service bus

- Status: Accepted
- Date: 2026-09-17
- Deciders: Mark Miller

## Problem

Indexer, transaction-service, portfolio-service, and analytics-service
need to share facts. HTTP chaining (indexer ? portfolio ? analytics)
creates temporal coupling: a slow consumer blocks a producer, retries
duplicate side effects, and makes replay a folklore script.

## Options considered

### 1. Synchronous HTTP between services

Simple to demo. Fails the moment analytics is down or the indexer is
replaying a day of logs. Portfolio latency becomes a function of the
slowest downstream. No replay.

### 2. NATS (core or JetStream)

Excellent latency, operationally small. Core NATS is fire-and-forget -
wrong for ledger-like facts. JetStream can persist, but retention,
consumer groups, and replay tooling are weaker than Kafka's for
"rebuild a read model from offset 0".

### 3. Kafka / Redpanda

Partitioned log. Multiple independent consumers. Replay is a first-class
operation. Ordering can be guaranteed *per key* (address, tx hash).
Heavier than NATS. Locally we run **Redpanda** (Kafka protocol, one
binary, no JVM). Production can be Kafka or Redpanda without changing
clients.

### 4. Postgres LISTEN/NOTIFY or outbox-only

Good inside one service. Cross-service fan-out and independent replay
are awkward. We still use an **outbox inside** transaction-service
(ADR-004). The bus is for inter-service facts.

## Decision

Use Kafka protocol (Redpanda locally) as the integration bus.

Topics are facts, partitioned by entity id:

| Topic | Key | Why |
| --- | --- | --- |
| `atlas.chain.block` | `chain_id:block_number` | Canonicality stream |
| `atlas.chain.log` | `address` | Portfolio can shard by wallet contract |
| `atlas.chain.reorg` | `chain_id` | All consumers must see the same detach |
| `atlas.tx.lifecycle` | `intent_id` | Exactly one partition per intent |

## Why not NATS

Atlas has a rebuild problem: portfolio and analytics must be
reconstructable after a schema change. That is a log, not a mailbox.
If this system were 2 services and a websocket, NATS would win.

## Trade-offs

- **Accepted:** more moving parts locally (mitigated by Redpanda).
- **Accepted:** at-least-once delivery. Consumers must be idempotent
  (ADR-003).
- **Rejected:** "smart" broker routing. The broker stores bytes.
  Schema lives in `docs/events.md`.
