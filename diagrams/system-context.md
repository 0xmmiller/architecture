# System context

Atlas sits between a user (or a developer with the SDK) and public
EVM networks. 0xMiller Labs does not operate a custodian. Signing, if
any, is out of process - the platform tracks intents and on-chain
facts.

```mermaid
C4Context
  title Atlas - system context

  Person(user, "Operator / developer", "Tracks positions and intents via API or SDK")
  System(atlas, "Atlas", "Digital asset operations platform. Indexes chain data, tracks transaction intents, serves portfolio and analytics APIs.")
  System_Ext(rpc, "EVM RPC providers", "JSON-RPC / WebSocket. eth_getLogs, eth_getBlockByNumber, eth_getTransactionReceipt")
  System_Ext(chain, "EVM networks", "Canonical ledger. Reorgs happen.")

  Rel(user, atlas, "HTTPS / JSON")
  Rel(atlas, rpc, "JSON-RPC")
  Rel(rpc, chain, "Node protocol")
```

Trust boundary: Atlas **believes the indexer**, not a client's idea
of their balance. Clients may lie; the chain (after reorg handling)
does not.
