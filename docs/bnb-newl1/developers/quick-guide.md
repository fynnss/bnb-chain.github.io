---
title: Quick Guide - BNB NewL1
---

# Developer Quick Guide

If you build on any EVM chain, you can build on BNB NewL1. Solidity, the toolchain (Foundry, Hardhat, Remix), and the SDKs (ethers.js, viem, web3.py) all work unmodified against a node's standard JSON-RPC endpoint.

This page is the orientation: how to get a chain running, what behaves differently from what you're used to, and where the reference material is.

!!! note "Devnet stage"
    There is no public testnet or mainnet endpoint, no faucet, and no hosted explorer yet. Today you develop against a local devnet built from the client source. Network parameters on these pages reflect the current devnet client and may change before a public launch.

## Getting started

1. **Run a chain.** Build the `newl1` client and boot a local devnet — one validator plus one observer is enough for everything except load testing. → [Run a Local Devnet](../get-started/local-devnet.md)
2. **Point your tools at it.** Node 0 serves JSON-RPC at `http://127.0.0.1:8545` and WebSocket at `ws://127.0.0.1:8546`. Pre-funded test accounts come from `scripts/genesis.conf`.
3. **Deploy something.** The [contract walkthrough](../get-started/local-devnet.md#deploy-your-first-contract) goes deploy → read → write with `cast`, verified end-to-end.
4. **Read the differences below** before you port anything real.

## What's different from a typical EVM chain

Five things account for nearly every surprise. Each links to the page that covers it properly.

| What | Why it matters | Details |
|---|---|---|
| **You pay for your declared gas limit** | Unused gas is never refunded, so SDK gas padding is money spent, and receipts report `gasUsed` equal to the limit. | [Migrating from BSC](../get-started/migrate-from-bsc.md#you-pay-for-your-declared-gas-limit) |
| **`latest` means executed, not ordered** | Ordering and execution are decoupled; state reads resolve to the executed tip, and an explicit block number that hasn't executed yet returns `-38004` rather than stale data. | [Async Execution](../core-concepts/async-execution.md) |
| **No global mempool** | Transactions route straight to the current and next block producer. No `txpool_*`, no public pending-transaction feed, and replace-by-fee only affects the pools holding the transaction. | [JSON-RPC Reference](./json-rpc.md) |
| **No `eth_getProof`** | State is committed with an LtHash accumulator, not a Merkle-Patricia trie, so there is no trie proof to serve. Anything proof-based needs a different design. | [Async Execution](../core-concepts/async-execution.md) |
| **200 ms blocks** | `block.number` advances about five times a second, and `block.timestamp` (still whole seconds) repeats across consecutive blocks. Block-count and timestamp math both need review. | [Migrating from BSC](../get-started/migrate-from-bsc.md#contracts) |

Coming from BSC specifically? [Migrating from BSC](../get-started/migrate-from-bsc.md) is the full checklist, including what carries over unchanged — the gas asset, the Parlia staking and governance contracts at their original addresses, and the `parlia_*` RPC methods.

## What you get that you don't have elsewhere

None of these are required to run a standard EVM app — they're additive, and they're the reason to build here.

| Capability | What it gives you |
|---|---|
| [Transaction pre-confirmation](../core-concepts/tx-preconfirmation.md) | A validator-signed inclusion signal in ~20 ms, long before the block lands. Advisory and revocable — never treat it as finality. |
| [Native account abstraction](../core-concepts/account-abstraction.md) | The `0x76` transaction envelope: atomic call batches, session and admin keys, passkey (WebAuthn/P256) signing, and gas sponsorship — no bundler, no EntryPoint contract. |
| [Multi-lane blockspace](../core-concepts/multi-lane.md) | Governance-configured reserved gas quota per traffic class, so latency-sensitive flows aren't crowded out by unrelated load. |
| [Shielded pool](../core-concepts/privacy.md) | An opt-in native system contract for private transfers, alongside ordinary transparent ones. |
| [BLS fast finality](../core-concepts/consensus.md) | Irreversibility in roughly one block interval, instead of waiting on confirmation depth. |

## Reference

- **[Examples](./examples.md)** — runnable demos: watch a pre-confirmation land, find the current leader, drive load, run the end-to-end scenario suite.
- **[Transaction Types](./transaction-types.md)** — the two native EIP-2718 types (`0x76` account abstraction, `0x77` shielded), their fields, how to sign them, and what trips people up.
- **[JSON-RPC Reference](./json-rpc.md)** — every method the node serves: the standard `eth_*` surface and its differences, the `newl1_*` namespace, WebSocket subscriptions, the `parlia_*` compatibility methods, and the error codes.
- **[Network Information](../get-started/network-info.md)** — chain id, protocol parameters, endpoints.
- **[Run a Local Devnet](../get-started/local-devnet.md)** — building the client, booting a devnet, deploying a contract, connecting an SDK.
- **[System Contracts](../governance/system-contracts.md)** — every system contract and precompile address, including the BSC carry-overs and the BNB NewL1–original ones.
- **[Governance](../governance/overview.md)** — the propose → vote → queue → execute lifecycle that controls protocol parameters.
- **[FAQ](../faq.md)**

## Tooling

Standard EVM tooling connects with no chain-specific client:

- **Contracts and testing** — [Foundry](https://book.getfoundry.sh/), [Hardhat](https://hardhat.org), [Remix](https://remix.ethereum.org)
- **SDKs** — [ethers.js](https://docs.ethers.org), [viem](https://viem.sh), [web3.py](https://web3py.readthedocs.io)

Two caveats when you wire them up: set a real `maxPriorityFeePerGas` (the base fee is pinned to zero, so the tip is the entire price), and don't pad gas limits (you pay for what you declare). Hosted explorers, indexers, and faucets are not available at this stage.
