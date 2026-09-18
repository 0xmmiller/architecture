# Atlas

> Personal engineering lab project. Not a company, not a custodian.

Atlas is a small distributed system for **digital asset operations**:
intent tracking, EVM indexing with reorgs, portfolio read models, and
analytics. It exists to make architecture decisions reviewable.

**Start here:** [architecture](https://github.com/0xmmiller/architecture)

| Repo | Role |
| --- | --- |
| architecture | ADRs, C4, failure modes |
| platform-api | External HTTP |
| transaction-service | Idempotent intents |
| blockchain-indexer | Chain facts + reorgs |
| portfolio-service | Positions |
| analytics-service | History |
| infrastructure | Compose / Helm / telemetry |
| sdk-python | Typed client |
