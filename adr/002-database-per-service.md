# ADR-002: Database per service

- Status: Accepted
- Date: 2026-09-17
- Deciders: Mark Miller

## Problem

Five services share a domain (wallets, transactions, positions) but
not a lifecycle. A shared PostgreSQL schema looks cheaper on day one
and becomes a distributed monolith on day thirty: migrations couple
release trains, and "just join portfolio to indexer tables" leaks
across boundaries.

## Options considered

### 1. Shared database, shared schema

Fastest to prototype. One backup, one set of credentials. Any service
can `SELECT` another service's tables, so the boundary is a comment.

### 2. Shared database, separate schemas

Slightly better. Still one blast radius for migrations, connection
saturation, and backup restore. Temptation to join across schemas
remains.

### 3. Database per service

Each service owns its PostgreSQL database (separate logical DB on
one server locally; separate instances when a service's load or
compliance diverges). Integration is via events, not foreign keys.

### 4. One service, one database (modular monolith)

Valid, and if Atlas were a product with two engineers shipping weekly
I would start here. The *point of this lab* is to show service
boundaries, independent scaling, and the cost of that choice. A
modular monolith would hide the interesting problems.

## Decision

Database per service. Locally: one Postgres server, five databases
(`atlas_tx`, `atlas_indexer`, `atlas_portfolio`, `atlas_analytics`,
`atlas_api`). Redis is shared infrastructure but **namespaced by
service prefix**. No service reads another service's tables.

Read models that need chain data subscribe to `atlas.chain.*`.
They do not query the indexer DB.

## Trade-offs

- **Accepted:** no cross-service joins. Portfolio stores the columns
  it needs, even if they duplicate indexer fields.
- **Accepted:** more migration discipline.
- **Rejected:** distributed transactions. We use outbox + idempotent
  consumers instead of 2PC.
- **Local convenience:** one Postgres container is not a contradiction.
  Isolation is logical. Physical split is a scaling lever
  (see `docs/scaling.md`), not a day-one requirement.
