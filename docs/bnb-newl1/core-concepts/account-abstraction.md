---
title: Native Account Abstraction - BNB NewL1
---

# Native Account Abstraction

BNB NewL1 has a native account-abstraction transaction: EIP-2718 envelope type `0x76`. One transaction can carry a batch of calls, be authorized by a passkey or a delegated key instead of the account's own key, and have its gas paid by someone else.

There is nothing to deploy or opt into. There is no bundler, no EntryPoint contract, and no smart-contract wallet — the rules below are enforced by every node. An ordinary account keeps its address and simply starts sending `0x76` transactions. Type `0x76` is valid from genesis, and standard transactions are unaffected.

## What it gives you

- **Atomic batches.** 1–64 calls in one transaction, one signature, one nonce, one receipt. Either all of their effects persist or none do.
- **Passkey signing.** secp256k1, P256, and WebAuthn are all valid, so an account can be controlled by a device passkey with no private key ever existing.
- **Delegated keys.** Authorize additional keys with a call scope, a spend limit, and an expiry — session keys for a game, a bot, or a team member.
- **Gas sponsorship.** Another account can pay, so a brand-new unfunded account can make its first transaction.

## Sending an AA transaction

### Fields you set

| Field | Meaning |
|---|---|
| `chain_id` | Replay protection. |
| `nonce_key` | Second nonce dimension. **Must be `0`** today. |
| `nonce` | The account's ordinary nonce. |
| `max_fee_per_gas`, `max_priority_fee_per_gas` | Standard EIP-1559 fees. The priority fee must not exceed the cap. |
| `gas_limit` | Covers the **whole batch**. See [Gas](#gas). |
| `calls` | 1–64 ordered entries, each a target, a native value, and calldata. A contract creation may only be the first entry, and only a root-signed transaction may create. |
| `access_list` | Standard EIP-2930 access list. |
| `valid_after`, `valid_before` | Optional validity window. See [Validity windows](#validity-windows). |
| `key_authorization` | Optional inline key grant, applied within this transaction. |
| `fee_payer_signature` | Optional sponsor signature. |

A single transaction may not exceed **128 KiB** encoded.

### Signing

The account is always the **last** signer. If the transaction is sponsored, the sponsor signs first — reversing the order breaks verification, because the account's own hash commits to whether a fee payer is present.

1. Fill in the body. Leave the fee-payer signature empty for now.
2. **Sponsored only:** the fee payer signs the **fee-payer hash**, which commits to the body and to the account being sponsored. This is a *different* digest from the one in the next step — do not sign the same hash twice.
3. The account signs the **signing hash**:
    - a **root** key signs it directly, and the account is recovered or derived from the signature itself;
    - a **delegated** key signs an **account-bound** digest instead — the signing hash mixed with the account address — and the result is wrapped together with that address. This is what stops a delegated key's signature from being replayed against a different account it is also authorized on.
4. Encode with type `0x76` and submit.

### Submitting and reading the result

Submit with the ordinary `eth_sendRawTransaction`. Read the outcome with `eth_getTransactionReceipt`: the receipt is a standard receipt with type `0x76`, and its status distinguishes a successful batch from a reverted one. `eth_getTransactionCount` is unchanged — an AA account has one ordinary nonce.

## Signature schemes

| Scheme | Use | How the key is identified |
|---|---|---|
| **secp256k1** | Ordinary account key | The usual address, recovered from the signature |
| **P256** (secp256r1) | Passkey or secure-enclave key | Derived from the public key, so the signature carries it |
| **WebAuthn** | Browser or device passkey | Same derivation as P256; the signature is a WebAuthn assertion |

Any of the three can be a root key or a delegated key. P256 and WebAuthn signatures are not recoverable, so the public key travels inside the signature.

!!! warning "Client-side rules that are consensus-enforced"
    Every node re-verifies these, so a client that deviates produces a transaction that never lands.

    For **P256**, and for the inner signature of a WebAuthn assertion: the public key must be a valid curve point, and the `s` scalar must be low-s — **normalize before submitting**, as high-s signatures are rejected.

    For **WebAuthn**: the authenticator data must be exactly 37 bytes, so no extensions; at least one of the user-present or user-verified flags must be set; the client data type must be the "get" assertion type; and the client data **challenge must be the transaction's signing hash**, unpadded base64url. A standard browser passkey assertion whose challenge is that hash satisfies all of it.

An account whose **root** key is a passkey cannot sign a legacy or EIP-1559 transaction at all — it can only act through `0x76`. It also starts with no balance, so its first transaction has to be sponsored.

## Delegated keys

Every AA transaction acts for exactly one account. Root, admin, and access are authorization levels for that account, not separate accounts.

| Authority | What it can do | Restrictions |
|---|---|---|
| **Root key** | The account's own key; its address *is* the account. Deploys contracts, manages keys unconditionally. Cannot be revoked. | None |
| **Admin key** | Authorize and revoke other keys, including other admin keys. Cannot deploy contracts. Revocable. | Must be unrestricted: no scope, limit, or expiry |
| **Access key** | Ordinary delegated key, typically a session key. Cannot manage keys. Revocable. | Bound by its scope, spend limit, and expiry |

No delegated key can ever revoke root, so root always retains control. But an admin key can revoke every *other* delegated key, so a compromised one can lock an account out of its own session keys — issue them accordingly.

Keys are managed through the `AccountKeychain` precompile at `0x0000000000000000000000000000000000004000`, with a companion `SignatureVerifier` at `0x0000000000000000000000000000000000004001`.

### Authorizing a key

Two paths, and the choice matters mostly for how many scopes you need:

| Path | How | Max scopes |
|---|---|---|
| **Inline** | Attach a key authorization to an AA transaction. It runs as the batch's first call, so you can authorize a key and take its first action **atomically** — if anything reverts, the grant reverts too. | 32 |
| **Direct call** | Call `authorizeKey` as root or an admin key. A conventional account can do this **from a plain, non-AA transaction**, so no AA transaction is needed just to provision a key. A passkey-rooted account cannot, and must use a sponsored AA. | 256 |

Either path allows at most 64 selectors per scope. Prefer the inline authorize-and-use shape when a key's first action is known: splitting authorize and use across two transactions can fail if the first is reordered or reverts in between.

`authorizeKey` takes the key identifier, its signature scheme, an expiry in unix seconds (zero means never), the admin flag, the scope, and the spend limit.

### Scope, limit, expiry

**Scope.** Unscoped means any target and any calldata. Scoped restricts the key to a list of targets, each optionally narrowed to specific 4-byte selectors; a scoped key with an empty list can do nothing. Root bypasses scope entirely.

**Spend limit.** A cap on native outflow, counting both the value the key's calls move *and* the gas it pays when self-paying — which under this chain's fee model is the full declared `gas_limit`. Size a limit to cover gas, not just the transfer. A zero period is a lifetime bucket; a non-zero period is a rolling window. A sponsor's gas never counts against the key.

**Revocation is permanent.** A revoked identifier can never be authorized again on that account. Note also that an identifier cannot be re-authorized while its slot is occupied, *including* one that has expired but not been revoked — rotate to a fresh identifier.

### Reading key state

Call [`newl1_getKey`](../developers/json-rpc.md#newl1_getkey) for a key's signature scheme, expiry, revoked and admin flags, scope flag, spend limit, period, and the amount consumed in the current period. A never-authorized or revoked key reports blank metadata.

## Gas

!!! warning "You are charged the `gas_limit` you declare, not the gas you use"
    Unused gas is **not** refunded and post-execution storage refunds are void, for AA and standard transactions alike. Blocks are packed by declared gas before being executed, so declared gas is the block space you are actually buying, and receipts report gas used equal to the gas limit.

    **Over-estimating is not insurance, it is spend.** Size the limit tightly with `newl1_estimateGas`, which prices the exact batch, signature scheme, and keychain state your transaction will meet. `eth_call` and `eth_estimateGas` simulate with the prepaid model off, so they still report actual usage.

An AA pays normal EVM gas for its calls plus a small surcharge for the machinery it uses: about 5,000 for a P256 signature, 5,000 plus a per-byte cost for a WebAuthn assertion, 3,000 for a delegated-key check, and about 27,000 for an inline key authorization — more if it carries a spend limit or a large scope, since each scope target and selector is a storage write.

Two bounds constrain the limit you declare. Below, it must clear the intrinsic cost plus that surcharge and the EIP-7623 calldata floor, or the transaction is **rejected at submission**. Above, no user transaction may declare more than **16,777,216** gas.

## Fee sponsorship

A fee payer signs to cover another account's gas. Gas is charged to the sponsor; the account only advances its nonce and pays any native value its calls move.

- **Self-sponsorship is rejected** — the fee payer must be a different account.
- The transaction is accepted only if the sponsor can cover the gas *and* the account can cover the value its calls send.
- The sponsor signs **before** the account. See [Signing](#signing).

## Validity windows

The two optional bounds gate inclusion by **block timestamp**: `valid_after` is inclusive, `valid_before` is exclusive. A not-yet-valid transaction is held and promoted automatically when its window opens; an expired one is dropped.

Nodes also apply a local acceptance policy so they do not hold transactions forever: `valid_after` may be at most **120 seconds** in the future, and `valid_before` must be at least **3 seconds** out, or the transaction is refused at submission.

## When something goes wrong

There are only two outcomes worth planning for.

**Rejected at submission.** You get an error, nothing lands, no gas is charged, and the nonce does not move. This covers a bad signature, a wrong chain id, a malformed batch, a `gas_limit` outside the allowed range, a validity window outside the acceptance policy, self-sponsorship, a delegated key that is revoked or out of scope, and blob or set-code transaction types.

**Lands and fails.** If a call reverts or runs out of gas, the whole batch rolls back but the transaction still lands: the receipt reports failure, the declared `gas_limit` is charged, and the nonce advances. For a limited key that gas counts against the limit; the rolled-back value does not.

!!! tip "Passing submission does not guarantee success"
    Delegated-key permission is checked again at execution, against state that may have shifted since you submitted — the key could be revoked, or its limit consumed, in between. Such a transaction lands as a failed receipt charging the full `gas_limit`, so size the limit for the full batch and do not assume a permission problem always fails for free.

## For contracts and indexers receiving AA calls

Nothing breaks by default, and one new capability is available.

- **The account is the transaction origin**, and it keeps its ordinary address, so contracts inspecting the origin or the caller behave exactly as they would for a standard transaction.
- **Receipts and blocks are standard**, tagged with type `0x76`. Indexers need no special handling beyond recognizing the type.
- **A contract can see which key is acting.** The keychain precompile reports the key identifier that signed the current transaction — zero meaning root — and the account it acts for. An application can require root authorization for a sensitive action while accepting a session key for routine ones. The `SignatureVerifier` precompile also lets a contract verify a passkey signature natively, or accept "any active key of this account", without shipping its own verifier.

## Limits

| Item | Value |
|---|---|
| Transaction type | `0x76` |
| Calls per batch | 1–64 |
| Max encoded size | 128 KiB |
| Max declared `gas_limit` | 16,777,216 |
| Scopes per authorization | 32 inline, 256 direct |
| Selectors per scope | 64 |
| `nonce_key` | must be `0` |
| `valid_after` / `valid_before` at submission | at most 120 s ahead / at least 3 s out |
| Contract creation | first call only, root-signed only |
| Unsupported types | EIP-4844 blobs, EIP-7702 set-code |
