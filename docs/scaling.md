# Scaling

Scaling Atlas is not "add Kafka". Each service saturates on a
different resource. Independent databases (ADR-002) exist so those
knobs can move independently.

## What is expected to get hot

| Surface | Bottleneck | First lever | What we will not do first |
| --- | --- | --- | --- |
| `GET /v1/portfolio/{address}` | Postgres read + cache miss | Redis, covering index on `(owner, asset)` | Merge with analytics DB |
| Indexer catch-up | RPC rate limits, `eth_getLogs` range size | Batch windows, extra provider, partition by address set | In-process RPC cache as source of truth |
| Analytics 90d windows | Seq scans, memory | Rollup tables, separate replica | Put rollups in portfolio |
| Intent create | Postgres unique insert | already cheap; pool size | Redis-only idempotency |
| Kafka | disk, partition count | more partitions on `atlas.chain.log` keyed by address | A second broker product |

## SLOs (lab targets, not a paid SLA)

- platform-api availability: 99.9% excluding planned compose restarts
- `GET /v1/portfolio/{address}` p95 < 50ms with warm Redis
- indexer head lag < 12s under a single mock RPC
- consumer lag alert: > 10_000 messages for 5 minutes

These numbers are **intent**. PyScale is where we measure Python
itself; Atlas services expose `/metrics` so the same questions can
be asked of the product.

## Horizontal vs vertical

- **platform-api**: stateless. Scale replicas behind the ingress.
  Rate limit counters live in Redis so replicas share a budget.
- **transaction-service**: write-heavy but low QPS. One primary is
  enough; workers scale on submission polling, not on HTTP.
- **indexer**: dangerous to run two *writers* against one chain
  without leader election. Scale by **splitting address sets**, not
  by duplicating the head follower.
- **portfolio / analytics**: N consumers in a Kafka group. Partition
  key = address.

## What we refuse to scale on hope

Connection pools: each replica times `pool_size * replicas` against
Postgres `max_connections`. Default math is in `infrastructure`.
If you double API replicas, you cut pool size or raise Postgres.

## Degrade, don't lie

If Redis is down, portfolio hits Postgres (slower, still correct).
If the RPC provider is brownout, indexer pauses and exposes lag -
it does not invent blocks. If Kafka is down, HTTP still creates
intents (outbox fills); submission and projections stall, API
returns `202` + `X-Atlas-Degraded: bus`.
