---
title: Native Account Abstraction - BNB NewL1
---

# Native Account Abstraction

BNB NewL1 adds a new native transaction type — envelope `0x76` — that lets any account batch multiple calls atomically, delegate scoped signing authority to secondary keys, sign with a passkey instead of a private key, and have its gas sponsored by another account. All of it is enforced directly by the protocol.

## vs. ERC-4337

Ethereum's ERC-4337 account abstraction relies on a separate mempool, a bundler, an EntryPoint contract, and (optionally) a paymaster contract — all deployed as ordinary smart contracts with their own trust and gas-accounting model. BNB NewL1's account abstraction has none of that: there's no EntryPoint contract to trust, no bundler role, and no userOp-vs-transaction mempool split. Signature verification, key scoping, spend limits, and gas sponsorship are **consensus rules**, so behavior is uniform across every account rather than depending on which wallet contract a user happened to deploy. Standard transactions are completely unaffected, and the AA machinery is only paid for by transactions that actually use it.

## Core concepts

**The `0x76` envelope.** An EIP-2718 typed transaction submitted the normal way, via `eth_sendRawTransaction`. Its body carries: `chain_id`, `nonce_key` (reserved for future parallel-nonce lanes — must be `0` today), `nonce`, standard EIP-1559 fee fields, `gas_limit`, a `calls` array (1–64 ordered `{to, value, input}` entries executed atomically as a batch), an `access_list`, a `valid_after`/`valid_before` validity window, an optional inline `key_authorization`, and an optional `fee_payer_signature`.

**Key hierarchy.**

| Key type | Can do | Can be revoked? |
|---|---|---|
| **Root key** | The account's own implicit key. The only key that can deploy contracts, and can manage/authorize other keys unconditionally. | No |
| **Admin key** | Can authorize or revoke other keys, but cannot deploy contracts. Must be unrestricted — no scope, spend limit, or expiry. | Yes (by root or any other admin) |
| **Access/session key** | An ordinary delegated key, constrained by call scope, spend limit, and expiry. Cannot manage keys. | Yes (by root or any admin) |

**Signature schemes.** secp256k1 (standard ECDSA), P256, and WebAuthn (browser/device passkeys) are all valid for the root key or any keychain key — meaning a BNB NewL1 account can be controlled entirely by a passkey, with no private key ever existing.

**The `AccountKeychain` precompile**, at address `0x0000000000000000000000000000000000004000` (with a companion `SignatureVerifier` precompile at `0x0000000000000000000000000000000000004001`), is where keys are managed and read:

- `authorizeKey(keyId, sigType, expiry, isAdmin, scoped, allowedCalls, limited, limit, period)` — authorize a new key (root/admin only).
- `revokeKey(keyId)` — permanently revoke a key (root/admin only; irreversible — a revoked `keyId` can never be reused).
- `getKey(account, keyId)` — read a key's current state.

**Call scope.** An access key can be restricted to a `{target, selectors}` allow-list. `scoped=false` allows any call; `scoped=true` with an empty list denies everything — scoping is additive and explicit.

**Spend limit.** An access key can be capped by a native-currency spend limit (`limit` per `period`, in wei/seconds). This counts both value transferred *and* any gas the key's account pays for itself. `period=0` creates a lifetime (non-resetting) bucket.

**Validity windows.** `valid_after`/`valid_before` gate whether a transaction can be included based on block timestamp — useful for scheduled or time-boxed transactions.

**Gas sponsorship.** A separate account (the fee payer) can sign to cover another account's gas for a given transaction. Self-sponsorship is rejected; the sponsor's signature must be produced before the account's own signature.

## How to use it

1. **Build the transaction body** with the fields above; leave `fee_payer_signature` empty unless sponsoring.
2. **If sponsored**, the fee payer signs first, over a hash that commits to the transaction body and sender address.
3. **The account signs** — root keys sign directly; keychain (secondary) keys sign an account-bound hash that explicitly names the account, preventing a key's signature from being replayed against a different account.
4. **Submit** the assembled, signed transaction via standard `eth_sendRawTransaction`.
5. **Authorize a secondary key** either inline (via `key_authorization` in an AA transaction, atomic with its first use — up to 32 scopes) or directly via `authorizeKey` on the `AccountKeychain` precompile (up to 256 scopes, 64 selectors per scope) — the latter is callable even from a plain, non-AA transaction.
6. **Revoke a key** with `revokeKey` — permanent, no un-revoke.
7. **Read key state** via the [`newl1_getKey`](../developers/json_rpc/newl1-api-list.md#newl1_getkey) RPC method: `{sigType, expiry, isRevoked, isAdmin, scoped, spendLimit, spendPeriod, spent}`.

### Example flows

- A simple sponsored transfer signed with a passkey (no gas, no private key, on a brand-new device).
- An atomic batch of several calls in one transaction (e.g. `approve` + `swap` in a single atomic unit).
- Delegating a time-boxed, spend-capped session key to a game or trading bot, then revoking it when done.
- An admin key provisioning access keys for a team, without ever exposing the root key.

## Limits

- `nonce_key` must be `0` — parallel-nonce lanes are reserved but not yet enabled.
- No EIP-7702 (set-code) or EIP-4844 (blob) transactions — both are rejected at the RPC layer and at block import.
- Max encoded transaction size: 512 KiB. Max 64 calls per batch.
- Only the **first** call in a batch may be a contract creation (`CREATE`), and only the **root key** can create contracts — a keychain-signed batch containing `CREATE` is rejected. Inline key authorization and `CREATE` cannot appear in the same transaction.
- Admin keys must always be unrestricted — no scope, spend limit, or expiry.
- Spend-limit accounting includes gas the key's account pays for itself, not just value transferred.
- A revoked key can never be re-authorized, and an already-authorized `keyId` (even if expired but not revoked) can't be reused.
- A sponsored or passkey-derived account can start out completely unfunded — sponsorship covers its very first transaction.
- Gas cost includes normal EVM execution cost plus AA-specific surcharges for signature verification (P256, WebAuthn) and keychain checks; underfunding a transaction below its required base cost causes it to be rejected outright at submission, with no receipt, no gas charged, and no nonce change.

## Current status

Implemented today: the `0x76` envelope and batching, the root/admin/access key hierarchy, all three signature schemes with protocol-enforced verification, gas sponsorship, validity windows, scope and spend-limit enforcement, and full RPC/pool/miner/import support — live from genesis, with no feature flag or activation height gating it. Deferred to a future iteration: EIP-7702/EIP-4844 support and parallel-nonce lanes (`nonce_key ≠ 0`), both reserved but currently rejected.
