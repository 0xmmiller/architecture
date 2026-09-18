# ADR-006: Indexer owns chain canonicality

- Status: Accepted
- Date: 2026-09-17
- Deciders: Mark Miller

## Problem

EVM chains reorg. A log that was "final" at block N can disappear at
N+1. If portfolio applies logs as immortal facts, balances drift from
chain reality and never converge.

## Options considered

### 1. Ignore reorgs; wait for N confirmations

Simple. Slow. Still wrong during a deep reorg past N. Fine for a
price ticker, not for a position ledger.

### 2. Every service talks to RPC and decides canonicality

Duplicated reorg logic, inconsistent views, RPC cost multiplied.

### 3. Indexer emits canonical facts and explicit reorg events

Indexer tracks a block hash chain. On mismatch, it walks back to the
common ancestor, emits `atlas.chain.reorg` with the detached range,
and re-emits logs for the new fork. Consumers detach by `block_number`
and replay.

## Decision

**blockchain-indexer is the only component allowed to interpret
RPC canonicality.** Downstream services are fork-agnostic: they apply
and detach by block number / log identity.

Confirmations are a *display* concern (API can mark a position
`unconfirmed` until depth ? K). They are not a substitute for detach.

## Consumer contract

On `atlas.chain.reorg`:

1. Delete or mark invalid all projections with `block_number > common_ancestor`.
2. Wait for replayed `atlas.chain.log` events.
3. Do not try to "fix" balances with compensating arithmetic.

## Trade-offs

- **Accepted:** indexer is a single point of chain-truth. High
  availability for this service matters more than for analytics.
- **Accepted:** consumers must store `block_number` on every row they
  might detach.
- **Rejected:** "eventual overwrite" without explicit detach. Silent
  overwrite hides bugs.
