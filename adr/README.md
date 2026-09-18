# Architecture Decision Records

ADRs are the interesting part of this repository.

Each record captures a decision that is expensive to reverse: the
problem, the options that were actually considered, the choice, and
what we accepted as a cost.

Format is Michael Nygard's, kept short enough to read in a hiring loop.

| ID | Title | Status |
| --- | --- | --- |
| [000](000-record-architecture-decisions.md) | Record architecture decisions | Accepted |
| [001](001-message-broker.md) | Kafka/Redpanda as the inter-service bus | Accepted |
| [002](002-database-per-service.md) | Database per service | Accepted |
| [003](003-idempotency.md) | Transaction idempotency | Accepted |
| [004](004-event-driven-architecture.md) | Event-driven integration | Accepted |
| [005](005-observability.md) | OpenTelemetry as the observability backbone | Accepted |
| [006](006-reorg-handling.md) | Indexer is the source of chain canonicality | Accepted |

| [007](007-go-notification-plane.md) | Go for the notification plane | Accepted |

When a decision is superseded, the old ADR stays. We add a new one and
link back. Rewriting history of *why* is worse than being wrong.
