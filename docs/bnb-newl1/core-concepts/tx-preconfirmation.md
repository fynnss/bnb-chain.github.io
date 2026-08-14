---
title: Transaction Pre-confirmation - BNB NewL1
---

# Transaction Pre-confirmation

Transaction pre-confirmation lets a client learn — in roughly **20 milliseconds**, compared to waiting a full block interval or more — that the validator currently building a block has committed to including their transaction at a specific position within it. It is always-on network infrastructure (every node relays and serves it), not a per-operator opt-in, though only the current block producer originates a stream at any given moment.

## Guarantees

Pre-confirmation is a **best-effort, validator-signed ordering-inclusion promise**. It says nothing about execution success and nothing about finality:

- It never touches the vote pool, fork choice, or finality logic — it's purely advisory and read-only with respect to consensus.
- A bad or dishonest stream can mislead a subscriber, but it can never affect the chain itself.
- Pre-confirmations **can and do get revoked** (reported as `aborted` or `superseded` on the same subscription) — this is expected, normal behavior, not a fault. Applications must handle revocation rather than treating "pre-confirmed" as final.

## Confirmation ladder

Think of transaction status as progressing through up to four stages, the last two of which are independent of each other:

| Stage | Latency | Meaning |
|---|---|---|
| `preconfirmed` | ~20 ms | The current leader has streamed a signed sub-block including your tx. Revocable. |
| `ordered` | ~1 block interval | Your tx is in an imported block. |
| `finalized` | ~1–few block intervals | Two-thirds of validators voted for the block (see [Consensus](./consensus.md)) — irreversible. |
| `executed` | Shortly after ordering | The EVM has actually run the transaction and produced a result. |

Anything that must be irreversible — releasing funds, settling a trade — should wait for **finalized**, and re-check the block hash, since a reorg can still replace an ordered-but-not-yet-finalized block.

## How to use it

### RPC (WebSocket)

- **`newl1_subscribePreconfirmations`** — subscribe to node-interpreted pre-confirmation events: `{txHash, blockNumber, parentHash, subBlockIndex, subBlockHash, position, leader, timestampMs, status}`. Accepts an optional `{txHash}` filter (no sender-level filter). Also emits `{"status": "lagged", "dropped": n}` flow-control frames if the subscriber falls behind, so a missed revocation is never silent.
- **`newl1_subscribeSubBlocks`** — subscribe to the raw, signed sub-blocks as gossiped, for clients that prefer to independently verify rather than trust the node's interpretation.

### RPC (HTTP)

- **`newl1_getTransactionPreconfirmation(txHash)`** — one-shot cache lookup. Cache retention is short (roughly 80 blocks); a `null` result does not mean the transaction was never pre-confirmed, only that it's outside the retention window or wasn't tracked.
- **`newl1_getValidatorSchedule`** — current validator set, RPC endpoints, current leader index, `epoch_length`/`turn_length`, and next rotation block — used to independently verify who the leader should be.

### Self-verification

A client that doesn't want to trust the node's own interpretation can verify a raw sub-block itself:

1. Subscribe to `newl1_subscribeSubBlocks`.
2. Recover the signer from the sub-block's signature.
3. Confirm the signer is in the current validator set via `newl1_getValidatorSchedule` (this checks set membership only, not exact turn/rank, and only against the live/near-real-time schedule).
4. Re-derive the transaction root and confirm your transaction's position within it.

## Limitations

- Ordering-only: a pre-confirmation never implies execution will succeed.
- Append-only within its ~20 ms window: a later, higher-fee transaction cannot reorder transactions that have already been streamed.
- The promise dies with a leader failover — if the current leader goes offline before finishing its turn, any of its outstanding pre-confirmations may be superseded by the next leader's block.
- Coverage can be incomplete during mixed-version network rollouts.
- A conservative (fail-closed) break rate is expected and is a health metric to monitor, not necessarily a sign of a problem.

!!! warning "Requires a network peer — a fully isolated single node sees nothing"
    Verified directly: sub-blocks are delivered by P2P gossip, and a node does **not** loop its own broadcast back to its own local subscribers. On a completely isolated single-node devnet (0 connected peers), `newl1_subscribePreconfirmations` and `newl1_subscribeSubBlocks` report **zero events** on that node's own WebSocket — even while it is actively producing blocks and real transactions are landing in them (`streamed=true` in the node's logs, but nothing reaches the local subscription). The moment the node has at least one connected peer (e.g. a second, non-validator observer node), pre-confirmation events immediately appear on the peer's subscription. If you're evaluating this feature on a minimal local devnet, boot at least two nodes (they don't both need to be validators) and subscribe from the one that **isn't** producing the block, matching how the project's own test suite validates this feature.
