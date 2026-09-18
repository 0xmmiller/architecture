# Containers

Five runtime services, one broker, Postgres (logical DBs), Redis,
and an OTel collector. Notification-service is intentionally absent:
nothing currently needs a second consumer enough to justify a sixth
process. When it does, it subscribes to existing topics.

```mermaid
flowchart TB
  subgraph Edge["edge"]
    API["platform-api\nFastAPI\nauth, RPS limit, BFF"]
  end

  subgraph Domain["domain services"]
    TX["transaction-service\nintents, outbox, lifecycle"]
    IDX["blockchain-indexer\nRPC, checkpoints, reorgs"]
    PORT["portfolio-service\npositions read model"]
    AN["analytics-service\nhistorical aggregates"]
  end

  subgraph Data["data plane"]
    K[(Redpanda)]
    PG[(PostgreSQL)]
    RD[(Redis)]
  end

  subgraph Telemetry["telemetry"]
    OTEL[OTel collector]
    PROM[Prometheus]
  end

  API --> TX
  API --> PORT
  API --> AN
  TX --> PG
  TX --> RD
  TX --> K
  IDX --> K
  IDX --> PG
  K --> PORT
  K --> AN
  PORT --> PG
  PORT --> RD
  AN --> PG
  API --> OTEL
  TX --> OTEL
  IDX --> OTEL
  PORT --> OTEL
  AN --> OTEL
  OTEL --> PROM
```

## Why these cuts

- **platform-api vs transaction-service:** public contract vs write-side
  invariants. The API can be rewritten (GraphQL, later) without
  touching idempotency.
- **indexer vs portfolio:** ingestion is CPU/RPC bound and must
  rewind. Portfolio is query-bound and cached. Different saturation
  profiles (see `docs/scaling.md`).
- **analytics vs portfolio:** a 90-day timeseries query must not
  share a connection pool with `GET /positions`.
