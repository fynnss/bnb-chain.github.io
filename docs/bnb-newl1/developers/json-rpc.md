---
title: JSON-RPC Reference - BNB NewL1
---

# JSON-RPC Reference

BNB NewL1 exposes a standard Ethereum-compatible JSON-RPC surface (`eth_*`, `net_*`, `web3_*`) alongside a `newl1_*` namespace for the chain's own features, plus the `parlia_*` compatibility methods BSC tooling expects. Existing `eth_*`-compatible tooling (ethers.js, viem, web3.py, wallets, block explorers) works without modification, subject to the differences noted below.

HTTP and WebSocket serve the same method set, with one exception: **subscriptions are WebSocket-only**. Enable or disable each transport with `--http.addr` / `--ws.addr` (pass `none` to disable).

## Block tags and the two tips

Because [ordering and execution are decoupled](../core-concepts/async-execution.md), the node tracks two heights:

- **ordered tip** — the freshest block accepted by consensus.
- **executed tip** — the highest block whose post-state and receipts are materialized. It trails the ordered tip by the proposer-chosen delay.

There are **no custom tag strings on the wire** — only the five standard Ethereum tags plus an explicit number:

| Tag | Resolves to |
|---|---|
| `latest` / `pending` | The **executed** tip. `pending` is an alias for `latest`; there is no mempool-simulated pending state. |
| `safe` / `finalized` | The height of the nearest published execution commitment at or before the justified / finalized block. Always at or below the executed tip. |
| `earliest` | Block 0. |
| explicit number | That block, subject to the gating below. |

State-dependent reads (`eth_call`, `eth_getBalance`, `eth_getCode`, `eth_getStorageAt`, `eth_estimateGas`, `eth_getBlockReceipts`, `eth_getTransactionCount`, `eth_getLogs`, `newl1_getKey`) also accept an EIP-1898 `{"blockHash": "0x…"}` object in place of the tag, the same shape BSC and geth accept.

Two error codes distinguish the failure modes an explicit number can hit:

- **`-38004`** — the block is ordered but not yet executed. Retry once execution catches up.
- **`-38001`** — the block number is beyond the ordered head, or an EIP-1898 block hash doesn't resolve to a canonical block. No such block exists; retrying is pointless.

Block *header* lookups (`eth_getBlockByNumber` / `eth_getBlockByHash`) are not execution-gated: an ordered-but-unexecuted block is readable by its explicit number, and its four execution-derived fields (`stateRoot`, `receiptsRoot`, `logsBloom`, `gasUsed`) are `null` until a later block publishes the matching commitment.

!!! warning "`eth_getTransactionCount("latest")` is the deliberate exception"
    It returns the **ordered-tip** nonce, not the executed-tip one, so a sender accounts for its own already-ordered-but-unexecuted transactions instead of reusing a nonce. It does not look ahead into the not-yet-ordered pool — a sender pipelining transactions back-to-back must poll `latest` and wait for each to be ordered before using the next nonce (the same limitation Monad documents for its own `pending`).

## Differences from `eth_*`

BNB NewL1's design (see [Overview](../overview.md) and [Consensus](../core-concepts/consensus.md)) changes a few standard assumptions:

- **`eth_getProof` is not supported** and rejects with `-32004`. State commitment uses a cumulative hash accumulator (LtHash), not a Merkle-Patricia trie, so there is no trie-based inclusion proof to serve. An accumulator-native proof scheme is planned follow-up work, not available today.
- **No public mempool introspection.** BNB NewL1 has no global mempool — transactions route directly to the current and next block producer (see [`newl1_getValidatorSchedule`](#newl1_getvalidatorschedule)) — so there is no `txpool_content`-style endpoint. `eth_subscribe("newPendingTransactions")` exists, but reports only the transactions admitted to the node's *own* local pool.
- **Replace-by-fee works, but only locally.** A same-nonce resubmission must beat the incumbent's effective tip by at least 10% (`replacement transaction underpriced` otherwise), matching geth's threshold. Two caveats follow from having no global mempool: the replacement only displaces the incumbent in the pools that hold it, and a transaction forwarded from another node never displaces one this node received directly from a client.
- **Transactions are charged for their declared gas limit, not their gas used.** Unused gas is not reimbursed and EIP-3529 refunds are void, because the producer sells block space by declared gas before executing it. Consequently **`gasUsed` in a receipt equals the transaction's `gas` limit**, and `header.gasUsed` is the sum of declared limits — neither reports actual consumption. `eth_estimateGas` and `eth_call` deliberately simulate with this model *disabled*, so an estimate is still a true measurement of usage. A transaction's gas limit is capped at 16,777,216 (BEP-652 / EIP-7825). See [Migrating from BSC](../get-started/migrate-from-bsc.md#you-pay-for-your-declared-gas-limit).
- **Fees are all tip.** The base fee is consensus-pinned to zero (BEP-222), so the effective gas price is the lower of `maxFeePerGas` and `maxPriorityFeePerGas` — a transaction that caps high but tips zero pays nothing and is rejected by the pool's minimum-price gate (default 1 wei, matching geth's `PriceLimit`). Set a real `maxPriorityFeePerGas`; don't leave it at 0.
- **Pooled transactions expire.** A transaction that isn't included within the pool's TTL (60 s by default) is evicted, and nonces more than 256 ahead of the account's current nonce are rejected at submission.
- **No EIP-4844 (blob) or EIP-7702 (set-code) transactions.** Both types are rejected by the pool's transaction-type gate.
- **`eth_syncing` always returns `false`** — there is no staged-sync driver to report progress from.
- **Uncle methods are compatibility stubs** — counts are always `0x0`, bodies always `null`.
- **Blocks carry extra fields.** Alongside the standard header fields, `eth_getBlockBy*` returns `systemTransactionsRoot` and a `commitments` object (the published execution commitment for an earlier block). Standard deserializers ignore unknown fields.

## `newl1_*` methods

### `newl1_getValidatorSchedule`

Returns the current validator set and leader schedule. Used for discovering where to send transactions, and for independently verifying pre-confirmation signers (see [Transaction Pre-confirmation](../core-concepts/tx-preconfirmation.md)).

```jsonc
{
  "head_block": "0x961",
  "epoch_length": "0x3e8",      // configured blocks per epoch
  "turn_length": 16,
  "validators": [
    { "address": "0x…", "tx_rpc_endpoint": "http://10.0.0.1:8545" }
  ],
  "current_leader_index": 2,
  "next_rotation_block": "0x962"
}
```

`tx_rpc_endpoint` is the peer's VDN-advertised endpoint, and is an empty string for a validator that hasn't completed a VDN handshake with this node. Clients only need to re-fetch the full schedule at epoch boundaries; within an epoch, the current leader can be derived locally from the head block number and `turn_length`.

### `newl1_getKey`

Reads the on-chain state of an [account abstraction](../core-concepts/account-abstraction.md) delegated key. Parameters: `(account, keyId, blockTag?)`.

```jsonc
{
  "sigType": "0x0",      // 0x0 = secp256k1, 0x1 = P256, 0x2 = WebAuthn (raw hex, not a name)
  "expiry": "0xffffffffffffffff",   // max uint64 = no expiry; 0x0 = key was never authorized
  "isRevoked": false,
  "isAdmin": false,
  "scoped": true,
  "spendLimit": "0x...",
  "spendPeriod": "0x0",
  "spent": "0x..."
}
```

Under the hood this simulates `AccountKeychain.getKey` against executed state, so it accepts the same block tags and EIP-1898 hash objects as `eth_call`.

### `newl1_getTransactionStatus`

Pool-plane status of a transaction — where it sits in the node's local `TxStream`, which is a different question from "is it in a block" (`eth_getTransactionReceipt`).

```jsonc
{
  "state": "ready",     // pending | ready | parked | ordered | evicted | removed | rejected | unknown
  "reason": null,       // set only for a terminal state (e.g. why it was evicted)
  "sender": "0x…",
  "nonce": 0,
  "ageMs": 12
}
```

`unknown` is ambiguous by design: it covers both "never seen" and "aged out of the bounded status ring". Treat it as "no information", not as a negative outcome. Note this is a **local** view — it answers for the node you asked, not the network.

### `newl1_getTransactionPreconfirmation`

One-shot lookup of a transaction's last known [pre-confirmation](../core-concepts/tx-preconfirmation.md) status from a short-lived cache. A `null` result does not mean the transaction was never pre-confirmed — only that it fell outside the retention window or wasn't tracked.

### `newl1_laneConfig`

Returns the live, governance-resolved [multi-lane blockspace](../core-concepts/multi-lane.md) configuration for the current canonical head — the resolver's view (parent context, anchor depth, branch-aware), not a raw registry dump.

```jsonc
{
  "parentNumber": "0x13",
  "parentHash": "0x...",
  "anchorNumber": "0xb",        // the lagged executed-state checkpoint this config was resolved from
  "anchorHash": "0x...",
  "configAnchorDepth": "0x8",   // activation-delay depth, in blocks
  "version": "0x0",             // null until governance has ever set a version
  "inert": true,                // true until governance has ever set a lane config
  "laneDefs": [],               // [{ laneId, quotaBps }, ...] once configured
  "laneRoutes": []              // [{ to, laneId }, ...] address -> lane assignments
}
```

`inert: true` with empty `laneDefs`/`laneRoutes` is the genesis default, until a governance proposal activates a lane.

### `newl1_getSystemReceiptsByBlock`

Receipts for the block's **system transactions** (slashing, deposits, finality rewards, validator-set updates, shielded-pool drains). These run in the block executor after ordinary transactions and have no place in `transactions_root`, so they don't appear in `eth_getBlockReceipts`. Parameters: `(blockTag?)`.

Returns `null` — distinct from an empty array — when the block's system transactions haven't been committed by a descendant block yet. Because the system-tx hash index is only populated at finalized-persist time, a hash returned here can still be `null` on `eth_getTransactionReceipt` until finality catches up.

### `newl1_debugSenderSnapshot`

Debug surface: one sender's live pool lane — every entry nonce-ascending with its state and age, plus the watermarks the block packer works from (`confirmedBase`, `orderedNonce`). Answers "why isn't this sender's transaction being packed" without reading node logs. Parameters: `(address)`.

## Subscriptions (WebSocket only)

| Subscribe | Notification | Unsubscribe | Streams |
|---|---|---|---|
| `eth_subscribe("newHeads")` | `eth_subscription` | `eth_unsubscribe` | Complete Ethereum-compatible header on every canonical-chain advance. |
| `eth_subscribe("newPendingTransactions", fullTx?)` | `eth_subscription` | `eth_unsubscribe` | Transactions newly admitted to this node's local pool — hashes, or full objects with `fullTx=true`. The node closes the subscription if the receiver falls behind. |
| `newl1_subscribeNewHeads` | `newl1_newHeads` | `newl1_unsubscribeNewHeads` | `{hash, number, justifiedHash, justifiedNumber, finalizedHash, finalizedNumber}` on every canonical advance — the ordered tip plus both consensus pointers in one frame. |
| `newl1_subscribePreconfirmations` | `newl1_preconfirmation` | `newl1_unsubscribePreconfirmations` | Node-interpreted [pre-confirmation](../core-concepts/tx-preconfirmation.md) events. Accepts an optional `{txHash}` filter. |
| `newl1_subscribeSubBlocks` | `newl1_subBlock` | `newl1_unsubscribeSubBlocks` | Raw signed sub-blocks straight from the gossip mesh, for clients that want to verify signatures themselves. |

A pre-confirmation notification carries `{txHash, blockNumber, parentHash, subBlockIndex, subBlockHash, position, leader, timestampMs, status}`, with `status` one of `preconfirmed` / `aborted` / `superseded`. If a subscriber falls behind, the node sends `{"status": "lagged", "dropped": n}` rather than silently dropping a revocation — re-query `newl1_getTransactionPreconfirmation` when you see it.

!!! warning "The pre-confirmation surface is unauthenticated"
    `newl1_subscribePreconfirmations`, `newl1_subscribeSubBlocks`, and `newl1_getTransactionPreconfirmation` are unauthenticated by design, and each subscription holds server-side memory. Firewall the RPC bind address, or put this surface behind a separate admin port, on any non-loopback deployment.

## BSC-compatible extras

Beyond the standard Ethereum set, the node serves the methods BSC tooling expects:

| Method | Behavior |
|---|---|
| `eth_getFinalizedHeader(verifiedValidatorNum)` | Explicit finality query. Accepts `[2,21]`, returns `max(fastFinalized, latest - verifiedValidatorNum)`. |
| `eth_getFinalizedBlock(verifiedValidatorNum, fullTx)` | Block form of the above. |
| `eth_getTransactionReceiptsByBlockNumber(tag)` | All receipts for the selected block. |
| `eth_getTransactionsByBlockNumber(tag)` | Full transaction objects for the selected block. |
| `eth_health` | Lightweight liveness probe; `true` while the RPC service is running. |
| `parlia_getSnapshot(tag?)` / `parlia_getSnapshotAtHash(hash)` | Parlia snapshot at a block. |
| `parlia_getValidators(tag?)` / `parlia_getValidatorsAtHash(hash)` | Validator addresses from the selected snapshot. |
| `parlia_getJustifiedNumber(tag?)` / `parlia_getFinalizedNumber(tag?)` | Snapshot vote-data target / source number. |
| `parlia_getTurnLength(tag?)` | Validator turn length from the selected snapshot. |

## Error codes

| Code | Meaning |
|---:|---|
| `-32602` | Malformed params, or a log-filter range exceeding the configured maximum. |
| `-32601` | Method not found — also used for "registered but not yet meaningfully implemented". |
| `-38004` | The requested block or transaction is ordered but not yet executed. Retry. |
| `-38001` | The requested block doesn't exist (beyond the ordered head, or a non-canonical hash). Don't retry. |
| `-32004` | Protocol-shape mismatch — today only `eth_getProof`. |
| `-32000` | A simulated call reverted (`data` carries the ABI-encoded revert reason) or halted. |
| `-32603` | Internal error. |
