---
title: Migrating from BSC - BNB NewL1
---

# Migrating from BNB Smart Chain

Same gas asset, same Parlia consensus family, same staking and governance contracts at the same addresses, unmodified EVM. Contracts deploy unchanged and standard tooling connects unchanged. The differences are almost all **above** the EVM — fees, mempool, block cadence, and how state is committed.

!!! note "No public network yet"
    There is no public testnet or mainnet endpoint, so migrating today means porting against a [local devnet](./local-devnet.md). A native BNB NewL1 ↔ BSC bridge is designed and under development but not live; the `CrossChain` address currently holds a placeholder (see [System Contracts](../governance/system-contracts.md)).

## You pay for your declared gas limit

**A transaction pays its full declared `gas_limit`, regardless of gas used.** No unused-gas refund, and EIP-3529 refunds are void. Blocks are packed by *declared* gas before execution, so declared gas is the block space actually sold.

- **Padding costs money.** `gasLimit = estimate * 1.5`, ethers' default padding, and hardcoded `500000` are all pure waste here.
- **`eth_estimateGas` still measures real usage** — simulation runs with charge-by-limit disabled. Declare its result plus only the headroom your call path genuinely needs.
- **`gasUsed` in receipts equals the declared limit**, and so does `header.gasUsed`. You were charged `effectiveGasPrice × gasUsed`; actual consumption needs a trace.
- **Limit is capped at 16,777,216** (2²⁴, BEP-652 / EIP-7825).
- **Same model for [`0x76` AA transactions](../developers/transaction-types.md)**, including sponsor charges and session-key spend limits.

System transactions and read-only simulation keep stock refund behavior.

## Carries over unchanged

| Area | Notes |
|---|---|
| EVM semantics | Full **Prague** from genesis — no fork ramp. |
| Gas asset | BNB. |
| Staking & governance contracts | `ValidatorSet`, `SlashIndicator`, `SystemReward`, `StakeHub`, `StakeCredit`, `Governor`, `GovToken`, `Timelock` at their BSC addresses ([details](../governance/system-contracts.md)). |
| Consensus | Parlia + BLS fast finality, with `parlia_getSnapshot` / `parlia_getValidators` / `parlia_getJustifiedNumber`. |
| BSC-flavored RPC | `eth_getFinalizedHeader`, `eth_getFinalizedBlock`, `eth_getTransactionReceiptsByBlockNumber`, `eth_getTransactionsByBlockNumber`. |
| Millisecond timestamps | As on BSC: seconds in `header.timestamp`, milliseconds in the low 8 bytes of `mixHash`. |

## Contracts

Bytecode runs as-is. What changes meaning is anything reasoning about **time or block cadence**.

- **`block.number` advances ~5×/second.** Every block-count constant — vesting, `blocksPerDay`, TWAP windows, auction durations — is off by ~3.75× versus BSC. Recompute them, or switch to timestamp math.
- **`block.timestamp` repeats** across roughly five consecutive blocks. `require(block.timestamp > lastUpdate)`, per-block rate limiting, and timestamp-as-unique-key all break.
- **`block.prevrandao` is not randomness** — it's the millisecond part of the timestamp, `0–999`. `block.difficulty` is Parlia's `2`/`1`.
- **The fee payer may not be the sender**, and one transaction can carry up to 64 atomic calls (`0x76`).
- **EIP-7702 and EIP-4844 transactions are rejected.** For smart accounts, use the native [`0x76` envelope](../developers/transaction-types.md) and the `AccountKeychain` precompile instead.

## Submitting transactions

- **Base fee is pinned to zero (BEP-222), so the tip is the price** — `min(maxFeePerGas, maxPriorityFeePerGas)`. Tipping `0` pays nothing and is rejected. Fees route through the system address to `ValidatorSet`, with BEP-95 burning its share.
- **No global mempool.** Transactions go straight to the current and next producer. No `txpool_*`, and `eth_subscribe("newPendingTransactions")` shows only this node's admissions.
- **Replace-by-fee needs a 10% bump and is local only** — it displaces the incumbent just in the pools holding it.
- **Pooled transactions expire** after 60 s, and a nonce more than 256 ahead is rejected at submission.
- **`eth_getTransactionCount(addr, "latest")` returns the ordered-tip nonce** and does not look ahead. Track nonces client-side when pipelining.

## Reading state and receipts

Where [async execution](../core-concepts/async-execution.md) shows up — the biggest behavioral gap versus BSC.

- **`latest` means executed, not ordered.** State reads trail the head you see on a `newHeads` subscription. Expected, not a lagging node.
- **`-38004` is retryable, `-38001` is not.** Ordered-but-unexecuted versus no-such-block. `eth_getTransactionReceipt` uses `-38004` for "block not executed yet" and `null` for "unknown transaction" — keep polling on the former.
- **`eth_getProof` rejects with `-32004`.** State is committed with an LtHash accumulator, not an MPT. Light clients, proof-based bridges, and similar need a different design; an accumulator-native proof scheme is planned.
- **`eth_syncing` is always `false`** — use `eth_health` plus block-number progress. Snap sync doesn't exist.

## Indexers and explorers

- **System transactions are invisible in the block body.** Slashing, deposits, finality rewards, validator-set updates, and shielded drains are not in the transaction list and not counted in `header.gasUsed`. Read them via [`newl1_getSystemReceiptsByBlock`](../developers/json-rpc.md#newl1_getsystemreceiptsbyblock) — they never appear in `eth_getBlockReceipts`.
- **Blocks carry `systemTransactionsRoot` and `commitments`**, and `stateRoot` / `receiptsRoot` / `logsBloom` / `gasUsed` are `null` while a block is ordered but unexecuted. Strict deserializers fail at the tip.
- **`newl1_subscribeNewHeads`** delivers head, justified, and finalized numbers and hashes in one frame per canonical advance.

Once you're ported, the [Quick Guide](../developers/quick-guide.md) covers what BNB NewL1 adds on top: pre-confirmation, multi-lane blockspace, native account abstraction, and the shielded pool.
