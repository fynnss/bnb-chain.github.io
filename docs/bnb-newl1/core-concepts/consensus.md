---
title: Consensus - BNB NewL1
---

# Consensus: Parlia PoA + BLS Fast Finality

BNB NewL1 uses **Parlia**, the same consensus family that secures BNB Smart Chain: a bounded, on-chain-elected set of validators takes turns proposing blocks (proof-of-authority), with BLS-signature-based fast finality layered on top so blocks become irreversible almost immediately rather than through accumulated confirmations.

## Block production

- A fixed **validator set** is elected on-chain (see [Governance](../governance/overview.md)) and rotates at **epoch boundaries every 1,000 blocks** (roughly 3.3 minutes at the current block interval).
- Within an epoch, validators take turns proposing blocks in a fixed order. Each validator seals a run of **16 consecutive blocks** (the "turn length") before rotation passes to the next validator.
- New blocks are produced every **200 ms** (5 blocks/second).
- If the in-turn validator misses its slot, out-of-turn validators can step in as backups after a deterministic back-off delay, so the chain keeps producing blocks even if a proposer is temporarily offline.

## Finality

Unlike a plain longest-chain / probabilistic-finality design — where a block is only ever "probably final" — Parlia adds **BLS fast finality**: validators sign BLS votes on the blocks they consider canonical, and once two-thirds or more of the validator set has voted for a block, it's **finalized** — cryptographically irreversible, typically within a small number of block intervals rather than a long confirmation wait.

Finality here is a statement about **ordering** — which block comes next — not about the result of executing it. Block execution itself runs on a separate, decoupled track; see [Async Execution](./async-execution.md) for how that works and what it means for reading state, receipts, and `eth_getProof`.

## For developers

- Validator admission and rotation are on-chain and queryable (see [`newl1_getValidatorSchedule`](../developers/json_rpc/newl1-api-list.md#newl1_getvalidatorschedule)).
- Block interval, turn length, and epoch length are chain parameters, not assumptions to hardcode into client code.
- Applications that need a hard guarantee before acting irreversibly (e.g. a bridge release, a large settlement) should wait for **finalized**, not just **ordered/imported** — see [Transaction Pre-confirmation](./tx-preconfirmation.md) for the faster, best-effort signal available before finality.
