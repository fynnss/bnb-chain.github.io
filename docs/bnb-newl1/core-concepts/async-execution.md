---
title: Async Execution & State Commitment - BNB NewL1
---

# Async Execution & State Commitment

BNB NewL1 separates **ordering** (deciding which transactions go in which block, and in what order) from **execution** (running those transactions through the EVM and computing the resulting state). A block can be proposed, gossiped, and even voted on for [fast finality](./consensus.md) before its execution result is fully computed — execution runs asynchronously, on its own track, and catches up shortly after.

## Why it matters

In a conventional EVM chain, block production is gated by execution: a proposer can't seal a block until every transaction in it has actually run. At sub-second block times, that coupling caps how fast blocks can be produced at how fast the EVM can execute them. Decoupling the two means consensus liveness — producing and finalizing blocks on schedule — no longer depends on execution keeping pace in real time. Execution is free to lag by a small, bounded amount and catch up asynchronously, without slowing down block production or finality itself.

## How it works

A `NewL1Header` carries the fields known at proposal time — including the transaction list — but omits the fields that can only be computed by actually executing the block: `state_root`, `receipts_root`, `logs_bloom`, `gas_used`, `blob_gas_used`, and `requests_hash`. Those values are published later, in **execution commitments** attached to a subsequent block.

- A block at height `N` carries a commitment list covering earlier blocks up to `N − D`, where `D` (the execution lag, in blocks) is chosen by the proposer and adjusts dynamically — nudged by at most one slot per block within a bounded range — based on how far behind the local execution scheduler actually is.
- Validators don't re-derive or second-guess the proposer's exact lag. They only enforce the **structure** of the commitment list: entries must be contiguous from the previous commitment, strictly increasing, and advance by a bounded amount per block.
- Execution itself runs off the hot path: an execution scheduler picks up imported blocks, runs the EVM work in the background, and produces the real post-execution state once it's done — independent of the block-production and voting loop.

## LtHash state commitment

BNB NewL1 commits state with a cumulative hash accumulator (`BLAKE3` over a lattice hash, "LtHash") rather than Ethereum's Merkle-Patricia trie. This is what makes async execution practical: updating a cumulative hash as execution completes is cheap, whereas maintaining an MPT is exactly the kind of synchronous-with-execution bookkeeping this design avoids.

Two concrete consequences for anyone integrating with the chain:

- **`eth_getProof` is not supported** (rejected with `-32004`) — there's no trie to produce a Merkle inclusion proof from. An LtHash-native inclusion proof scheme is a planned follow-up, not available today.
- **Snap sync is not supported.** A new node backfills over the chain's own P2P protocol rather than replicating a trie.

See [JSON-RPC Endpoint](../developers/json_rpc/json-rpc-endpoint.md) for the full list of RPC differences this causes.

## Correctness enforcement

Because ordering and execution are decoupled, a block's fast-finality vote is fundamentally a vote on ordering, not on someone else's claimed execution result. The correctness check happens somewhere else: a validator abstains from voting for a head whose published execution commitments it has locally computed to a **different** result. Since the state commitment is cumulative, any divergence at one block propagates into every descendant commitment, so this check reliably catches a bad result within a bounded number of blocks. The practical effect is that an incorrect execution result cannot gather the two-thirds vote quorum needed to finalize — even though block *import* itself doesn't reject it outright (rejecting at import would break the ability for later blocks to resolve their parent while execution is still catching up).

## For developers

Three block-height concepts exist side by side, and they're all reachable through **standard Ethereum block tags** — there are no custom tag strings or custom height methods on the wire:

| Concept | How to read it | Meaning |
|---|---|---|
| Executed | `eth_blockNumber` (no params, or `latest`/`pending`) | Latest block whose EVM execution has actually completed locally |
| Finalized | `eth_blockNumber("finalized")` (or `"safe"` for justified) | Latest published execution commitment at or before the finalized block — a two-thirds BLS vote quorum (see [Consensus](./consensus.md)) |
| Ordered | The `number` field on `newl1_subscribeNewHeads` / `eth_subscribe("newHeads")` | Latest block whose transaction order is settled |

The executed tip trails the ordered tip by the proposer-chosen delay, normally a small number of blocks.

Practical guidance:

- A transaction's ordering is settled quickly, but execution-derived data (state, receipts) for the *very latest* blocks lags behind it. You rarely have to think about this: `latest` and `pending` both resolve to the **executed** tip, so an ordinary read never returns pre-execution state and never fails with "not yet executed". Asking for an explicit block number that is ordered but not yet executed is what returns `-38004` — retry, don't treat it as an error.
- Only canonical **and** finalized blocks are persisted to disk — the unfinalized window lives in memory. This is why finalized, not just ordered, is the right bar for anything that shouldn't ever be revisited.

## Current status

The async execution pipeline itself is live and running today on the devnet. The LtHash state-commitment migration is functional today; follow-up work (a checkpoint-based state sync to replace snap sync, and a native LtHash inclusion-proof scheme for `eth_getProof`) is still in progress.
