# ADR-003: Transaction idempotency

- Status: Accepted
- Date: 2026-09-17
- Deciders: Mark Miller

## Problem

Clients retry. Kafka redelivers. Load balancers replay POST bodies.
If `CreateIntent` is not idempotent, a user can get two chain
submissions for one click. In a real asset platform that is a
loss event. Here it is the load-bearing correctness property.

## Options considered

### 1. Redis distributed lock

```
SET intent:{key} NX EX 30
```

Fast. Fails closed if Redis is empty after a flush, fails open if
the lock expires while the handler is still in Postgres. Locks are
about mutual exclusion, not durable "this already happened".

### 2. Idempotency key in Redis only

Same durability problem. Redis is the cache and the lock service,
not the ledger.

### 3. Database unique constraint on idempotency key

`UNIQUE (api_key_id, idempotency_key)` on `intents`. The first insert
wins. Concurrent retries collide on the unique index and the loser
reads the winner's row. Survives process crashes. Cheap. Well
understood.

### 4. Idempotency key + Redis lock + DB unique

Lock reduces stampede on the unique index under extreme retry storms.
Still, the **source of truth is the unique constraint**. Redis is an
optimization, not a correctness dependency.

## Decision

**Database unique constraint is the source of truth.**

- Clients send `Idempotency-Key` (UUID or ULID).
- `platform-api` forwards it unchanged.
- `transaction-service` inserts `intents (owner, idempotency_key, ...)`.
- `IntegrityError` ? fetch existing row, return it (same body if
  payload hash matches, `409` if the key is reused with a different
  payload).
- Kafka consumers use `(topic, partition, offset)` *or* a natural
  key (`tx_hash`, `intent_id`) with `INSERT ... ON CONFLICT DO NOTHING`.

Redis may hold a short-lived lock around the insert to keep retry
storms off the unique index. If Redis is down, we still insert.
Correctness does not depend on Redis.

## Payload hash

Storing `payload_sha256` next to the key prevents "same key, different
body" from silently returning the original intent. That class of bug
looks like success in load tests and like theft in production.

## Trade-offs

- **Accepted:** Postgres is on the write path. Intent creation is not
  a million-QPS problem; chain RPCs are the bottleneck.
- **Accepted:** keys are retained (not 24h TTL like Stripe's typical
  window). Intents are durable records. A later cleanup job can
  archive, not invent a second source of truth.
- **Rejected:** "just UUID the intent in the client" without server
  enforcement. Clients lie, proxies retry, users double-click.
