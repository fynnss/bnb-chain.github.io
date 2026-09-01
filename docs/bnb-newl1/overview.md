---
title: BNB NewL1 Overview - BNB NewL1
---

# BNB NewL1 — High-Performance EVM L1

BNB NewL1 is a high-performance, EVM-compatible Layer 1 and the newest member of the BNB Chain family, built for workloads that need sub-second finality and predictable inclusion: high-frequency trading, real-time payments, confidential finance, and agent-driven activity. **BNB is the native asset — no new token is issued**, and a native BSC ↔ BNB NewL1 bridge (in development) moves it between chains.

Existing Solidity contracts and standard wallet flows run unchanged (execution builds on [reth](https://github.com/paradigmxyz/reth) and [revm](https://github.com/bluealloy/revm)). What's rebuilt is everything around the EVM — consensus, node architecture, mempool, state storage — which is where 200 ms blocks, sub-second finality, ~20 ms pre-confirmation, and reserved capacity per traffic class come from. That rebuild also leaves a short list of RPC-surface differences (no `eth_getProof`, no global mempool, no EIP-4844/EIP-7702) that tooling built against those behaviors must account for — see [JSON-RPC Endpoint](./developers/json_rpc/json-rpc-endpoint.md).

## Where BNB NewL1 fits

| | | |
|---|---|---|
| 2017 | ERC-20 BNB | utility token on Ethereum |
| 2019 | Beacon Chain | governance and staking |
| 2020 | BSC | general-purpose EVM L1 |
| 2023 | opBNB + Greenfield | scaling and decentralized storage |
| Next | **BNB NewL1** | vertically optimized, latency-first EVM L1 |

BSC remains the broad-base chain for the existing ecosystem. BNB NewL1 is a clean-slate design for the workloads that outgrow it, keeping what defines the family: full EVM compatibility, BNB as the economic center, and shared tooling with BSC, opBNB, and Greenfield.

## System architecture

![BNB NewL1 system architecture](../assets/newl1-architecture.png)

| Layer | What's in it |
|---|---|
| Application | DeFi, payments, privacy-preserving finance, AI agents |
| [Account & UX](./core-concepts/account-abstraction.md) | Gas sponsorship, passkey signing, batched calls, scoped access keys — as protocol rules, not contracts, so a transaction needn't come from a BNB-holding EOA |
| [Execution](./core-concepts/async-execution.md) | Parallel EVM with ordering and execution decoupled, so consensus never waits on the EVM |
| [Consensus](./core-concepts/consensus.md) | Enhanced Parlia: 200 ms blocks, BLS fast finality, 20 ms [sub-blocks](./core-concepts/tx-preconfirmation.md) |
| Network | P2P block and sub-block gossip, direct-to-proposer transaction routing in place of a global mempool |
| [Storage](./core-concepts/async-execution.md#lthash-state-commitment) | Flat key-value store under a cumulative lattice-hash (LtHash) commitment instead of a Merkle-Patricia trie |

## What's new

Protocol-level features new to the BNB Chain ecosystem:

| Feature | What it gives you | Read more |
|---|---|---|
| **BLS fast finality** | Blocks become irreversible in roughly one block interval once two-thirds of validators vote, instead of relying on confirmation depth | [Consensus](./core-concepts/consensus.md) |
| **Async execution** | Block ordering and finality no longer wait on EVM execution to keep up — execution runs on its own decoupled track, so consensus liveness doesn't depend on execution throughput | [Async Execution](./core-concepts/async-execution.md) |
| **Transaction pre-confirmation** | A sub-second (~20ms), best-effort signal that the current block producer has committed to including your transaction, well before the block itself lands | [Pre-confirmation](./core-concepts/tx-preconfirmation.md) |
| **Multi-Lane** | Reserved gas capacity per transaction class (e.g. AMM, liquidation) so latency-sensitive traffic can't be crowded out by unrelated load | [Multi-Lane](./core-concepts/multi-lane.md) |
| **Native account abstraction** | A protocol-level transaction type (`0x76`) with batched calls, session/admin keys, passkey (WebAuthn/P256) signing, and gas sponsorship — no bundler or entry-point contract required | [Account Abstraction](./core-concepts/account-abstraction.md) |
| **Shielded pool (privacy)** | An opt-in, native system contract for zk-shielded transfers — sender, recipient, and amount hidden — alongside fully transparent transactions | [Privacy](./core-concepts/privacy.md) |
| **On-chain governance** | Token-weighted (govBNB) proposal/vote/timelock governance controlling protocol parameters, including the Multi-Lane configuration | [Governance](./governance/overview.md) |

## Design principles

- **EVM compatibility is non-negotiable.** Contracts, opcodes, and per-operation gas costs behave exactly as on any other EVM chain. What differs is billing: a transaction pays for its declared gas limit, not its gas used, because block space is sold by declared gas before execution runs ([details](./get-started/migrate-from-bsc.md#you-pay-for-your-declared-gas-limit)).
- **No global mempool.** Transactions go directly to the current and next block producer rather than a shared pending pool — part of what makes pre-confirmation and Multi-Lane packing possible.
- **New features are additive.** Account abstraction, pre-confirmation, Multi-Lane, and the shielded pool run alongside standard transactions; none of them change how an existing contract behaves.

## Network parameters

| Parameter | Value |
|---|---|
| Block interval | 200 ms |
| Turn length (blocks per validator, per rotation) | 16 |
| Epoch length | 2,250 blocks (7.5 minutes) |
| Consensus | Parlia PoSA (turn rotation + BLS fast finality), up to 64 validators |
| Gas asset | BNB (bridged from BSC) |

## Current status

BNB NewL1 runs today as a multi-validator devnet with real block production, P2P gossip, and end-to-end finality, and is under active development ahead of a public testnet/mainnet launch. Each feature page calls out what's shipped versus still in progress.

Every interface in this manual — the `eth_*`/`newl1_*` RPC surface, the full governance lifecycle, account-abstraction key management and the `0x76` envelope, and the complete shielded-pool flow — has been exercised against a devnet built from source, with no discrepancies beyond minor response-shape corrections already reflected here. The client has moved on since: the [JSON-RPC Endpoint](./developers/json_rpc/json-rpc-endpoint.md) page is kept current against client source, so newer methods are documented from the implementation rather than from a live call.
