# Security

Atlas is a lab. Treat it like a system that *could* sit in front of
money, then deliberately refuse the parts that would make it a
custodian.

## Threat model (STRIDE-lite)

| Threat | Where | Mitigation |
| --- | --- | --- |
| Spoofed API client | platform-api | Bearer API keys hashed at rest (scrypt/argon2id), never logged |
| Replay of POST /intents | transaction-service | Idempotency-Key + payload hash (ADR-003) |
| Tampered Kafka message | bus | Private network. Lab does not run Internet-facing Kafka. Prod would add mTLS + ACLs |
| Poison log payload | indexer ? consumers | Strict pydantic schema; malformed messages ? DLQ, not crash loop |
| SSRF via RPC URL | indexer | RPC endpoints from config, not from user input |
| Secret in traces | all | Attribute allow-list. No raw Authorization, no hex keys |
| Confused deputy (API as open proxy) | platform-api | Allow-listed downstream paths, no user-supplied URLs |

## Explicit non-features

- **No hot private keys in Atlas.** Signing is out of process. The
  lab may mock `eth_sendRawTransaction` with a fixture key *in tests
  only*.
- **No user passwords.** API keys are enough for a portfolio API.
- **No admin UI** until there is an audit trail.

## Data classes

| Data | Store | Notes |
| --- | --- | --- |
| API key hash | `atlas_api` | Salted hash, prefix for lookup |
| Intent payload | `atlas_tx` | May contain calldata; treat as sensitive |
| Chain logs | `atlas_indexer` | Public data |
| Positions | `atlas_portfolio` | Derived public data, still PII if mapped to a person |

## Supply chain

Pinned Python deps in each service. Multi-stage images, non-root
user, no `latest` tags in Helm values for prod profiles. GitHub
Actions with OIDC once this is deployed anywhere real.

## Disclosure

This is a personal lab. There is no bug bounty. Issues on the
architecture repo are the right place to argue with a decision.
