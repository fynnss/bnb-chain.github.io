---
title: Shielded Pool (Privacy) - BNB NewL1
---

# Shielded Pool: Native Transaction Privacy

BNB NewL1 includes a native **shielded pool** — a system contract that lets users deposit ("shield") funds, transact inside the pool with sender, recipient, and amount all hidden, and later withdraw ("unshield") back to a normal, transparent address. It sits alongside standard, fully transparent EVM transactions as an opt-in privacy layer, using the same cryptographic building blocks as established shielded-pool designs (notes, commitments, nullifiers, zk-SNARKs).

!!! warning "Not yet audited or production-ready"
    The shielded pool is an early-phase feature. As detailed below, its zk-proof trusted setup is currently a single-contributor development ceremony — **proofs can be forged** until a real multi-party ceremony replaces it. Do not treat it as safe for real value until this manual (and the project's own security disclosures) say otherwise.

## Why it matters

In a normal transparent EVM transaction, `from`, `to`, `amount`, and asset are always public on-chain. A shielded-pool transaction instead proves — via a zero-knowledge proof — that a transaction is valid (no funds created from nothing, no double-spend) without revealing which of the pool's notes were spent or where the value went. An observer only learns that *some* valid shielded-pool transaction occurred.

## Core concepts

- **Note** — a private record of value: an owner commitment, amount, and asset, plus whether it's "open" (see below). Notes are the pool's unit of value, analogous to a UTXO.
- **Commitment** — a hash of a note's contents, inserted as a leaf into an append-only Merkle tree. Spending a note requires proving Merkle-tree membership without revealing which leaf it is.
- **Nullifier** — a value published when a note is spent, derived from the note and the owner's spend key, used to prevent double-spending. Real nullifiers are indistinguishable from "phantom" (dummy) ones, so an observer can't even tell how many real inputs a given transfer had.
- **zk-SNARK ("pool proof")** — proves value conservation (inputs equal outputs plus any public amount leaving the pool), correct Merkle membership, correct nullifier derivation, and knowledge of the spend key authorizing the spend.
- **Keys.** A single seed derives: a nullifier key (spend authority — knowledge of it is what the proof establishes), a note-secret seed (per-note blinding, so notes to the same recipient are unlinkable to each other), and a view key (used only to detect and decrypt incoming notes — it never touches the proof itself).
- **Shielded address.** Registering on-chain (`registerPrivacyInfo`) publishes an account's view key so senders can encrypt outgoing note data to it. This links an EOA to its privacy public key, but individual notes sent to it remain unlinkable to one another.
- **Open notes.** A note can optionally have its amount and asset made public (`isOpen=1`) while its owner stays private — used for composing shielded value with ordinary DeFi calls (see "atomic call" below). Standard notes (`isOpen=0`) keep amount and asset hidden as well.

## Flows

- **Shield (deposit)** — a normal, public transaction from your EOA that locks funds into the pool and creates a note. No proof required.
- **Shielded transfer** — a private transfer inside the pool: two nullifiers (one may be a phantom) are spent to produce three outputs (recipient note, change note, and a private fee note), all encrypted to their respective recipients' view keys. Requires a zk proof generated off-chain.
This flow is submitted through a **relayer** rather than directly, so the sender's IP/identity isn't tied to the on-chain submission.
- **Unshield (withdraw)** — the reverse of shielding: a proof reveals the withdrawal amount, asset, and destination (these necessarily become public, since real funds are moving on-chain), and funds land at a normal address.
- **Atomic call** — unshield into a public EVM call (e.g. a DEX swap) and reshield the result into an open note, all within a single, indivisible transaction.

## How to use it

1. Generate a privacy keypair with the project's `newchain-shield` CLI (`keygen`).
2. Register your view key on-chain once (`register` / `registerPrivacyInfo`), so others can find it.
3. **Shield funds**: call the pool's deposit function (`deposit`) directly from your EOA.
4. **Send privately**: look up the recipient's registered view key, encrypt the outgoing note off-chain, generate a Groth16 proof (via a WASM prover, or an optional native prover for significantly faster proving), and submit through a relayer (`transfer`).
5. **Discover incoming funds**: scan on-chain note-commitment events and trial-decrypt them with your view key (`scan` / `wallet`) — discovery is near-immediate, though spendability can lag slightly (see below).
6. **Unshield**: generate a withdraw proof and submit (`withdraw`), optionally via a relayer to avoid linking the withdrawal request to your IP.
7. **Combine with DeFi**: use `atomic-call` to unshield, execute a public call, and reshield in one transaction.

The pool's on-chain address is `0x0000000000000000000000000000000000005000`; a companion auth-proof verifier lives at `0x0000000000000000000000000000000000005003`.

## Phase 1 scope

Shipped today:

- Native BNB only — no ERC-20/BEP-20 assets yet.
- Deposit, withdraw, and private transfer, plus open notes and atomic EVM calls.
- Fees are private by default (paid from a hidden fee note, spent by the relayer).
- An on-chain privacy registry for discovering recipients' view keys.
- Submission only via relayer — there is no dedicated privacy transaction type or direct/builder submission path yet.
- A deferred, batched note-insertion optimization: new note commitments are inserted into the Merkle tree in periodic batches (roughly every 50 blocks / ~10 seconds) rather than individually per transaction, to keep per-transaction cost down. **Practical consequence:** an incoming note is *discoverable* (via scanning) immediately, but only *spendable* once the next batch insertion runs — this lag can grow under network congestion.
- A minimal pause mechanism that can only halt new deposits; withdrawals, transfers, and atomic calls can never be paused or frozen.

Deferred to later phases: ERC-20/BEP-20 asset support, a dedicated shielded transaction type with direct (non-relayer) submission, full governance controls, per-asset compliance hooks, and additional authentication schemes (multisig, post-quantum).

## Security assumptions

- **Trusted setup.** The Groth16 proving system requires a per-circuit trusted-setup ceremony. The current deployment uses a single-contributor development ceremony, meaning proofs can currently be forged — a real multi-party ceremony is required before this is safe for real value.
- **What stays public even in "private" flows.** Deposits reveal sender, amount, and asset (funds must visibly enter the pool). Withdrawals reveal recipient, amount, and asset. Atomic calls reveal the unshielded amount and the public call's target/calldata. Only a pure shielded-to-shielded transfer hides everything.
- **The `isOpen` flag itself is public** per note — mixing open and standard notes without care can create identifiable patterns.
- **No network-level anonymity.** The shielded pool hides on-chain data, not network metadata — submitting without a relayer can still deanonymize you via IP or timing regardless of what's hidden on-chain.
- **Deposits can be paused** by a governance-controlled emergency switch; **withdrawals can never be paused**, so funds already in the pool can never be frozen.
