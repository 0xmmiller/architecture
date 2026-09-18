# Event contracts

Facts on the bus. `schema_version` is additive. Breaking changes get
a new event name, not a silent field reinterpretation.

Envelope (every message):

```json
{
  "event_id": "01J…",
  "event_type": "atlas.tx.intent_created",
  "schema_version": 1,
  "occurred_at": "2026-09-17T16:00:00Z",
  "correlation_id": "01J…",
  "producer": "transaction-service",
  "payload": {}
}
```

## `atlas.tx.lifecycle`

Key: `intent_id`.

Payload variants (`status`): `created | submitted | confirmed | failed`.

```json
{
  "intent_id": "uuid",
  "owner": "0xabc…",
  "status": "submitted",
  "tx_hash": "0xdef…",
  "idempotency_key": "uuid",
  "chain_id": 1,
  "payload_sha256": "…"
}
```

## `atlas.chain.block`

Key: `{chain_id}:{block_number}`

```json
{
  "chain_id": 1,
  "block_number": 19281000,
  "block_hash": "0x…",
  "parent_hash": "0x…",
  "timestamp": 1710000000
}
```

## `atlas.chain.log`

Key: `address` (contract or wallet, depending on subscription).

```json
{
  "chain_id": 1,
  "block_number": 19281000,
  "block_hash": "0x…",
  "tx_hash": "0x…",
  "log_index": 3,
  "address": "0x…",
  "topics": ["0x…"],
  "data": "0x…"
}
```

Natural idempotency key for consumers:
`(chain_id, block_hash, tx_hash, log_index)`.

## `atlas.chain.reorg`

Key: `chain_id`.

```json
{
  "chain_id": 1,
  "common_ancestor": 19280990,
  "from_block": 19280991,
  "to_block": 19281005,
  "detached_hashes": ["0x…"]
}
```

Consumers **must** drop projections in `(from_block, to_block]`
before applying new logs. See ADR-006.
