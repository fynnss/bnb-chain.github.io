---
title: Transaction Types - BNB NewL1
---

# Native Transaction Types

Standard Ethereum transactions (legacy, EIP-2930, EIP-1559) work exactly as you'd expect. On top of them BNB NewL1 adds two native EIP-2718 types of its own:

| Type | Name | Use it for |
|---|---|---|
| `0x76` | [Account abstraction](#0x76-account-abstraction) | Atomic call batches, session/admin keys, passkey signing, gas sponsorship |
| `0x77` | [Shielded transaction](#0x77-shielded-transaction) | Private transfers and withdrawals, submitted without a public sender |

Both are submitted through plain `eth_sendRawTransaction` and produce ordinary receipts (with `type` `0x76` / `0x77`). EIP-4844 (`0x03`) and EIP-7702 (`0x04`) are rejected — the two types above sit in a high, self-assigned range so a future upstream type can't collide with them.

!!! note "No SDK yet"
    Neither type is supported by ethers.js/viem today. You build and sign the RLP payload yourself; the client's own codec is the reference implementation.

## `0x76` account abstraction

One transaction, signed by an account or by a delegated key, carrying a batch of calls. Full concept documentation: [Account Abstraction](../core-concepts/account-abstraction.md).

### Body

| Field | Notes |
|---|---|
| `chain_id` | Replay protection. |
| `nonce_key` | 2D-nonce lane. **Must be `0`** today. |
| `nonce` | Sequential within `nonce_key`. |
| `max_priority_fee_per_gas`, `max_fee_per_gas` | EIP-1559 fee caps. Priority fee must not exceed max fee. |
| `gas_limit` | For the whole batch — and what you're [charged](../get-started/migrate-from-bsc.md#you-pay-for-your-declared-gas-limit). |
| `calls` | 1–64 `{to, value, input}` entries, executed atomically. Only the first may be a `CREATE`, and only a root key may deploy. |
| `access_list` | EIP-2930. |
| `valid_after` / `valid_before` | Optional inclusion window, in block-timestamp seconds. |
| `key_authorization` | Optional inline key provisioning, atomic with its first use. |
| `fee_payer_signature` | Optional sponsor signature. |

### Signing

1. **Sponsor first, if any.** The fee payer signs `fee_payer_signature_hash(sender)` — a hash over the transaction body *and* the sender address, so a sponsorship can't be lifted onto a different sender. Self-sponsorship is rejected.
2. **Then the account.**
    - A **root key** signs the transaction's own signing hash directly.
    - A **keychain key** (admin or access) signs an account-bound hash instead — the signing hash re-hashed together with the account address, so a key's signature can't be replayed against another account. Secp256k1, P256, and WebAuthn signatures are all accepted.
3. **Submit** the EIP-2718 payload via `eth_sendRawTransaction`.

### Gotchas

- **`nonce_key` must be zero.** Parallel nonce lanes are reserved but not enabled.
- **One nonce per transaction.** An inline `key_authorization` is prepended as a call, so combining it with a `CREATE` in the same transaction is rejected — do the authorization first, or deploy from a standalone transaction.
- **Session-key spend limits count gas too**, charged at the declared gas limit like everything else.
- **Key state is readable** at any time via [`newl1_getKey(account, keyId)`](./json-rpc.md#newl1_getkey), and keys can be managed from an ordinary transaction by calling `authorizeKey` / `revokeKey` on the `AccountKeychain` precompile at `0x…4000` — you don't need an AA transaction to bootstrap one.

## `0x77` shielded transaction

A self-authenticating transaction against the [shielded pool](../core-concepts/privacy.md): the two embedded zk-proofs *are* the authorization, so there is **no outer signature and no public sender**. Use it when the sender itself must stay private; use a plain `cast send` to the pool contract when it doesn't.

### Body

| Field | Notes |
|---|---|
| `chain_id` | Checked at import. |
| `gas_limit` | Bounds execution of the synthesized `transact` call. |
| `pool_proof`, `auth_proof` | Two PLONK proof blobs, **exactly 864 bytes each** (gnark BN254 `MarshalSolidity`). |
| `pub_signals` | 29 public signals, ABI-identical to the contract's `uint256[29]`. |
| `output_note_data` | Three output-note delivery ciphertexts. |
| `call_data` | Atomic-call payload; empty for transfer and withdraw. |

There is **no per-gas fee field**. The fee is the public `gasFee` in `pub_signals[28]`, unshielded to the system address by the contract — so a shielded transaction's fee is committed inside the proof, not bid at submission time.

Key public signals: `[0]` Merkle root, `[1..2]` spend nullifiers, `[3..5]` output commitments, `[12]` operation (`0` transfer, `1` withdraw, `2` atomic call), `[13]`/`[14]` public amount and recipient, `[20]` `intentReplayId`, `[23]` `validUntil` deadline, `[24]` chain id, `[28]` `gasFee`.

### The public path (no `0x77` needed)

Entering and leaving the pool is ordinary contract interaction with `ShieldedPool` at `0x…5000`:

```bash
# 1. shield funds — queues a note, doesn't insert it yet
cast send $POOL --value $AMOUNT "privDeposit(uint256,uint8,uint256,bytes)" \
  $OWNER_COMMITMENT $SCHEME 0 0x

# 2. drain the queue into the Merkle tree (permissionless)
cast send $POOL "deferBatchInsert(uint256)" 100

# 3. spend — transfer, withdraw, or atomic call
cast send $POOL "transact(bytes,bytes,uint256[29],bytes[3],bytes)" \
  $POOL_PROOF $AUTH_PROOF $PUB_SIGNALS "[0x,0x,0x]" 0x
```

Useful reads: `merkleRoot()`, `nullifierSpent(uint256)`, `pendingCount()` / `pendingHead()`, `getPrivacyInfo(address)`. Recipients publish their viewing keys with `registerPrivacyInfo(...)` so senders can encrypt note ciphertexts to them.

### Gotchas

- **Deposits are deferred.** `privDeposit` only queues the note; until someone calls `deferBatchInsert`, the note is not in the tree and `merkleRoot()` hasn't moved — so a proof built against the new root will fail.
- **Proofs are pinned to a root, a chain, and a deadline.** A stale `pub_signals[0]` root, a wrong chain id, or an expired `validUntil` is rejected. Rebuild the proof rather than retrying it.
- **Nullifiers are the replay guard** — resubmitting a spent proof fails on `nullifierSpent`.
- **The proof blob length is validated at decode.** Anything other than 864 bytes is rejected before execution, so a truncated proof costs a decode, not a transaction.

### Tooling

Producing proofs and notes requires the circuit tooling, not just an RPC client: `crates/shielded-circuits/scripts/newl1-shield.js` handles note encryption and scanning, and `cargo run -p newl1-engine --example shielded_encode -- <fixture> <chain-id>` encodes a signed `0x77` payload from a fixture using the node's own codec. The `e2e/cases/privacy.sh` and `e2e/cases/shielded-native.sh` scenarios run both paths end-to-end against a live devnet and are the most reliable worked examples.
