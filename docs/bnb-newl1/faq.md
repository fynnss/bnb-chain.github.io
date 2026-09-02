---
title: FAQ - BNB NewL1
---

# FAQ

**Is BNB NewL1 EVM-compatible?**
Yes at the execution layer — revm semantics are a hard design invariant, so existing Solidity contracts and standard transaction flows work unmodified. The RPC surface above that has a few explicit differences (no `eth_getProof`, no global mempool, no EIP-4844/EIP-7702 transactions) that some wallets or infra tooling may need to account for. See [Overview](./overview.md) and [JSON-RPC Endpoint](./developers/json_rpc/json-rpc-endpoint.md).

**How is it different from BNB Smart Chain?**
Same Parlia PoSA + BLS fast-finality consensus family, but with a faster block interval, decoupled ordering/execution, and several new protocol features not present on BSC: [native account abstraction](./core-concepts/account-abstraction.md), [transaction pre-confirmation](./core-concepts/tx-preconfirmation.md), [Multi-Lane](./core-concepts/multi-lane.md), and a native [shielded pool](./core-concepts/privacy.md). See [Overview](./overview.md) for the full comparison.

**Is BNB NewL1 live on mainnet?**
Not yet — it currently runs as an internal multi-validator devnet. Endpoints will be published on [Network Information](./get-started/network-info.md) once a network is available.

**Do I need to use ERC-4337 tooling (bundlers, paymasters) for account abstraction?**
No. BNB NewL1's account abstraction is native — enforced by consensus, not by a deployed EntryPoint contract — so there's no bundler or paymaster contract involved. See [Account Abstraction](./core-concepts/account-abstraction.md).

**Can I treat a pre-confirmed transaction as final?**
No. Pre-confirmation is a fast (~20ms), best-effort signal that can still be revoked before the block finalizes. Wait for **finalized** status for anything irreversible. See [Transaction Pre-confirmation](./core-concepts/tx-preconfirmation.md).

**Is the shielded pool safe to use with real funds today?**
No — the circuits are unaudited, and the deployment runs on an imported trusted setup rather than a BNB NewL1 ceremony. Both are open items before mainnet. See the warning in [Privacy](./core-concepts/privacy.md).

**Why doesn't `eth_getProof` work?**
BNB NewL1 commits state with a cumulative hash accumulator instead of a Merkle-Patricia trie, so there's no trie-based proof to serve. See [State DB](./core-concepts/state-db.md).

**How do I get my contract reserved capacity?**
Register your contract's address in the `LaneRegistry` system contract with a basis-point gas reservation, via [governance](./governance/overview.md). See [Multi-Lane](./core-concepts/multi-lane.md).

**What asset is used for gas and staking?**
BNB, bridged in from BNB Smart Chain — the same gas asset as BSC and opBNB.
