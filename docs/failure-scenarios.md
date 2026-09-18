# Failure scenarios

If a design only works on the happy path it is a demo. These are the
cases Atlas is built to survive, and the ones it will **not** pretend
to survive.

## 1. RPC provider brownout

**Symptom:** `eth_getLogs` times out or returns 429.

**Behaviour:** indexer stops advancing the checkpoint. Head lag
metric rises. No synthetic blocks. transaction-service submission
worker backs off with jitter. API reads still work from last
projected state, marked `as_of_block`.

**Not done:** silently switching to a second provider with a
different head without recording a provider failover event. Failover
is allowed; lying about block hash is not.

## 2. Reorg deeper than confirmation display depth

**Symptom:** parent hash mismatch (ADR-006).

**Behaviour:** indexer emits `atlas.chain.reorg`. Portfolio deletes
projections above the ancestor and waits for replay. API may return
stale-then-correct balances; `as_of_block` moves backwards, which is
honest.

**Not done:** "smoothing" balances so the UI never jumps. Jumps are
the feature.

## 3. Kafka / Redpanda unavailable

**Symptom:** produce fails.

**Behaviour:** HTTP intent create still commits (outbox row stays
`pending`). Publisher retries. Portfolio freezes at last offset.
API sets `X-Atlas-Degraded: bus` on read paths that depend on
freshness.

**Not done:** dual-write directly to Kafka from the request handler.

## 4. Duplicate Kafka delivery

**Symptom:** same `event_id` or same log identity twice.

**Behaviour:** consumers insert with `ON CONFLICT DO NOTHING` on the
natural key. Side effects do not run twice.

**Test:** `transaction-service` and `portfolio-service` unit tests
replay the same event.

## 5. Poison message

**Symptom:** payload fails schema validation.

**Behaviour:** skip + write to `{topic}.dlq` + increment
`poison_messages`. Do not block the partition with a crash loop.

**Not done:** auto-drop without metric. Silent drop is data loss.

## 6. Postgres primary down for one service

**Symptom:** that service 5xx / fail ready probe.

**Behaviour:** others continue. platform-api returns 503 for the
affected routes only (`/v1/intents` vs `/v1/portfolio`).

**Not done:** a global kill switch that takes the whole demo down.

## 7. Redis down

**Symptom:** cache and optional lock missing.

**Behaviour:** idempotency still correct (Postgres). Portfolio
serves from DB. Rate limiter fails **closed** (reject) rather than
open, because an open limiter turns an outage into an accidental
load test of Postgres.

## 8. Process killed mid-outbox drain

**Symptom:** row committed, message not produced.

**Behaviour:** publisher loop selects `pending` outbox rows.
At-least-once produce; consumers are idempotent.

## 9. What we do not survive (yet)

- Byzantine RPC (node returns plausible wrong receipts). Would need
  multi-provider quorum.
- Total disk loss of a service DB without backups.
- A second indexer writer without a leader lock.

Those are listed so a reviewer does not have to hunt for humility.
