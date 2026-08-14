---
title: BNB NewL1 Overview - BNB NewL1
---

# BNB NewL1 — High-Performance EVM L1

BNB NewL1 is a new, high-performance Layer 1 blockchain in the BNB Chain ecosystem. The EVM execution layer itself is unchanged — existing Solidity contracts run without modification, and standard wallet flows (sign a legacy/EIP-1559 transaction, submit it, read back the receipt) work as on any EVM chain. Everything above the EVM — consensus, node architecture, mempool, blockspace policy — has been rebuilt for higher throughput and faster finality, and that rebuild does introduce a handful of concrete RPC-surface differences (no `eth_getProof`, no global mempool, no EIP-4844/EIP-7702 transactions — see [JSON-RPC Reference](./developers/json-rpc.md) for the full list) that some wallets or infra tooling built against those specific behaviors would need to account for.

It's built for latency-sensitive on-chain activity — DEX/AMM trading, liquidations, and other flows where waiting a full block for any signal is too slow. 200ms blocks and BLS fast finality mean transactions finalize in roughly one block interval instead of accumulating confirmations, and two features unique to BNB NewL1 build on top of that: sub-20ms transaction pre-confirmation and reserved blockspace lanes, so latency-sensitive trades don't have to compete with unrelated traffic for inclusion. (Under the hood it builds on [reth](https://github.com/paradigmxyz/reth) and [revm](https://github.com/bluealloy/revm) for the execution layer.)

## What's new

BNB NewL1 keeps Parlia — the same proof-of-authority-with-BLS-fast-finality consensus family that secures BNB Smart Chain — but adds several protocol-level features that are new to the BNB Chain ecosystem:

| Feature | What it gives you | Read more |
|---|---|---|
| **BLS fast finality** | Blocks become irreversible in roughly one block interval once two-thirds of validators vote, instead of relying on confirmation depth | [Consensus](./core-concepts/consensus.md) |
| **Async execution** | Block ordering and finality no longer wait on EVM execution to keep up — execution runs on its own decoupled track, so consensus liveness doesn't depend on execution throughput | [Async Execution](./core-concepts/async-execution.md) |
| **Transaction pre-confirmation** | A sub-second (~20ms), best-effort signal that the current block producer has committed to including your transaction, well before the block itself lands | [Pre-confirmation](./core-concepts/tx-preconfirmation.md) |
| **Multi-lane blockspace** | Reserved gas capacity per transaction class (e.g. AMM, liquidation) so latency-sensitive traffic can't be crowded out by unrelated load | [Multi-Lane Blockspace](./core-concepts/multi-lane.md) |
| **Native account abstraction** | A protocol-level transaction type (`0x76`) with batched calls, session/admin keys, passkey (WebAuthn/P256) signing, and gas sponsorship — no bundler or entry-point contract required | [Account Abstraction](./core-concepts/account-abstraction.md) |
| **Shielded pool (privacy)** | An opt-in, native system contract for zk-shielded transfers — sender, recipient, and amount hidden — alongside fully transparent transactions | [Privacy](./core-concepts/privacy.md) |
| **On-chain governance** | Token-weighted (govBNB) proposal/vote/timelock governance controlling protocol parameters, including the multi-lane configuration | [Governance](./governance/overview.md) |

## Design principles

- **EVM compatibility is non-negotiable.** revm's execution semantics are never forked at the behavioral level — contracts, opcodes, and per-operation gas costs behave exactly as on any other EVM chain. What *is* different is how the resulting gas is billed: a transaction pays for its declared gas limit rather than its gas used, because block space is sold by declared gas before execution runs (see [Migrating from BSC](./get-started/migrate-from-bsc.md#you-pay-for-your-declared-gas-limit)).
- **No global mempool.** Transactions are routed directly to the current and next block producer (see [JSON-RPC Reference](./developers/json-rpc.md)) rather than gossiped through a shared pending-transaction pool, which is part of what makes pre-confirmation and multi-lane packing possible.
- **Ordering and execution are decoupled.** Blocks are ordered (and can be voted on for fast finality) before execution fully catches up, which keeps consensus liveness independent of EVM execution speed — see [Async Execution & State Commitment](./core-concepts/async-execution.md) for how this works and what it changes about `eth_getProof` and reading state.
- **New features are additive.** Account abstraction, pre-confirmation, multi-lane, and the shielded pool all run alongside standard transactions and standard EVM execution — none of them change how an existing contract or a plain transaction behaves.

## Network parameters

| Parameter | Value |
|---|---|
| Block interval | 200 ms |
| Turn length (blocks per validator, per rotation) | 16 |
| Epoch length (validator-set rotation) | 1,000 blocks |
| Consensus | Parlia (PoA turn rotation + BLS fast finality) |
| Gas asset | BNB (bridged from BSC) |

## Current status

BNB NewL1 runs today as a multi-validator devnet with real block production, P2P gossip, and end-to-end finality. Consult each feature page in this manual for what's shipped versus still in progress — this manual calls that out explicitly per feature, since BNB NewL1 is under active development ahead of a public testnet/mainnet launch.

Every interface documented in this manual — the core `eth_*`/`newl1_*` RPC surface, the full governance lifecycle (stake → propose → vote → queue → execute) activating a multi-lane quota, account-abstraction key management and the `0x76` envelope itself (secp256k1, P256, and WebAuthn signers, sponsorship, validity windows), and the complete shielded-pool flow (deposit, deferred batch insertion, withdraw with a real Groth16 proof, nullifier burn) — has been exercised directly against a locally-run single-validator devnet built from source, with zero discrepancies found beyond minor RPC response-shape corrections already reflected on this manual's pages. The client has moved on since that pass: the [JSON-RPC Reference](./developers/json-rpc.md) is kept current against the client source, so methods added after that devnet run are documented from the implementation rather than from a live call.
