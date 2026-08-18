---
title: Native Account Abstraction - BNB NewL1
---

# Native Account Abstraction

BNB NewL1 introduces envelope `0x76`, a native transaction type that provides batched atomic calls, delegated signing authority through secondary keys, passkey-based signing, and gas sponsorship — all enforced by consensus rules rather than by smart contracts.

There is no bundler, no EntryPoint contract, no paymaster contract, and no smart-contract wallet to deploy. An ordinary account gains every AA capability the moment it signs a `0x76` transaction, and it keeps its address while doing so.

AA is live **from genesis**. There is no hardfork gate and no feature flag: `0x76` is a valid transaction type at block 0 on every ingress path — RPC, the validator forward transport, the node's TxStream, and peer block bodies. Standard transactions are completely unaffected; they decode and execute exactly as before, and you pay for the AA machinery only when you use it.

## Why native account abstraction

Ethereum's account model binds three things together that users experience as separate concerns:

1. **Who may authorize a transaction** — exactly one secp256k1 key, for the lifetime of the account.
2. **What a transaction may do** — exactly one call.
3. **Who pays for it** — the sender, always, in the native coin.

The friction users take for granted in today's wallets traces back to those three bindings. Seed phrases exist because there is one irreplaceable key. Approving a token and then swapping it needs two transactions and two confirmations because there is one call per transaction. New users must acquire the native coin before their first transaction because there is no way for anyone else to pay.

### The ERC-4337 approach, and its cost

ERC-4337 solves this *above* the protocol. A user operation is not a transaction; it is a data structure that an off-chain **bundler** collects and submits inside a normal transaction to a singleton **EntryPoint** contract, which loops over operations and calls each user's **smart-contract wallet**, with an optional **paymaster** contract handling sponsorship. It works without changing consensus, and that is precisely why it is expensive:

- **Every account is a contract.** Users deploy a wallet before their first transaction, and pay contract-call gas for validation and execution thereafter. Signature checks run in the EVM rather than natively.
- **A parallel propagation layer.** User operations never travel the node's own transaction path, so bundlers reimplement gas accounting, replacement rules, and DoS protection, and the network's canonical fee market does not apply.
- **New trust assumptions.** Users depend on bundler liveness and honesty, and on the correctness of the EntryPoint and paymaster contracts, none of which are consensus-enforced.
- **Address discontinuity.** A smart-contract wallet has a different address from the EOA it replaces, and cannot be a transaction origin.
- **Signature schemes are contract code.** Passkey support means a Solidity P256 verifier, at Solidity prices.

### BNB NewL1's choice

BNB NewL1 makes AA a transaction type. A `0x76` transaction is a real transaction: it travels the same path as every other transaction, competes in the same fee market, produces a standard receipt, gets a standard transaction hash, and is the transaction origin for the calls it makes. Signature verification, including P256 and WebAuthn, is native code priced as intrinsic gas. Delegated-key state lives in an enshrined precompile rather than in per-user contract bytecode. Every rule described on this page is enforced by every node, so there is no bundler or contract whose good behaviour a user has to assume.

!!! note "A note on propagation"
    BNB NewL1 has **no global mempool**. A transaction is held by the node that received it — its *custodian* — in that node's **TxStream**, which forwards copies directly to the validators about to produce blocks. AA transactions use that same path, with the same admission checks and the same fee-ordered packing as standard transactions. Nothing AA-specific is bolted on beside it.

An existing account does not migrate, deploy, or opt in. It simply starts sending `0x76` transactions when it wants a batch, a passkey, a session key, or a sponsor.

## Lifecycle of an AA transaction

A `0x76` transaction passes through the same stages as any other transaction. The stages that behave differently for AA are noted.

1. **Submission.** The raw EIP-2718 bytes arrive at `eth_sendRawTransaction`. The node decodes the envelope and dispatches the `0x76` arm out of every single-transaction executor entry, before any conversion into EVM transaction parameters.
2. **Admission.** TxStream runs the statically checkable rules (Class A1) and the state-dependent ones (Class A2), including keychain permission as projected against committed state. Signature recovery happens once here and is memoized for later stages.
3. **Forwarding.** The custodian node forwards copies to the validators about to produce blocks, over the authenticated validator transport.
4. **Packing.** A leader re-checks the state-dependent rules against the latest state and packs by *declared* gas, before execution.
5. **Execution.** The AA handler injects the per-transaction authorization context, charges intrinsic gas plus the AA surcharge, deducts gas from the account or its sponsor, and runs the whole call batch inside a single journal checkpoint. Transaction-level settlement — nonce, gas, fee credit — then happens once.
6. **Import.** Peers re-run the static rules and the gas floor over the block body, so a block carrying an invalid AA is rejected wholesale.
7. **Result.** A standard receipt of type `0x76`, and any keychain writes landing in the block's state like ordinary contract storage.

Three properties of that pipeline are worth stating explicitly:

- **The batch is one transaction.** Nonce, fee deduction, gas settlement, and fee credit happen once per `0x76` transaction, whatever the batch does.
- **Keychain state is ordinary state.** The precompile reads and writes through the EVM journal, so its storage participates in the block's state root exactly like contract storage.
- **Verification is deterministic and state-free.** Signature recovery depends only on the transaction's bytes, never on chain state, which is what makes it usable as a block-import check on blocks produced by other validators.

## The `0x76` envelope

An AA transaction is EIP-2718 typed with leading byte `0x76` and submitted through the ordinary `eth_sendRawTransaction`. Receipts reuse the standard receipt type with a BNB NewL1 transaction-type tag, so `0x76` round-trips through RPC and storage without a forked receipt format.

### Fields

| Field | Meaning |
|---|---|
| `chain_id` | Replay protection. Checked against the chain spec at import. |
| `nonce_key` | Second nonce dimension. **Must be `0`** in the current version. |
| `nonce` | Sequential account nonce, same semantics as any account nonce. |
| `max_priority_fee_per_gas` | EIP-1559 priority fee. Must not exceed `max_fee_per_gas`. |
| `max_fee_per_gas` | EIP-1559 fee cap. |
| `gas_limit` | Gas limit for the **whole batch**, including AA surcharges. |
| `calls` | 1–64 ordered entries, each a target, a native value, and calldata. Executed atomically in order. |
| `access_list` | EIP-2930 access list. |
| `valid_after` | Earliest block timestamp, inclusive. Optional. |
| `valid_before` | Latest block timestamp, exclusive. Optional. |
| `key_authorization` | Inline key grant, applied within this transaction. Optional. |
| `fee_payer_signature` | Sponsor's signature. Optional. |

Each call names either a target address or a contract creation, with calldata or init code accordingly.

The account's outer signature travels alongside the body. It is not a body field, and the body's own signing hash does not commit to it.

### Encoding

The body is a single flat list in the field order of the table above, prefixed with the `0x76` type byte. Two conventions keep decoding positional: absent validity bounds encode as zero, and absent optional signatures and key authorizations encode as an empty string.

A single AA transaction may not exceed **128 KiB** encoded. Per-call calldata and access-list sizes are not capped separately; the whole transaction is bounded by that one limit, and an oversized transaction is rejected both at submission and at block import.

### The three hashes

| Hash | Signed by | What it covers |
|---|---|---|
| **Signing hash** | The account, via its root key or a keychain key | The type byte, every body field, whether a fee payer is present, and the inline key authorization |
| **Fee-payer hash** | The sponsor | The same body, but with the account's address in the fee-payer position instead of the presence marker |
| **Transaction hash** | — | The full signed encoding |

The presence marker in the signing hash is exactly that: a marker recording *whether* the transaction is sponsored, not the sponsor's identity or signature bytes. The account therefore commits to being sponsored without committing to who sponsors it.

The two signing hashes cannot collide, and no extra domain tag is needed to keep them apart: a 20-byte address in the fee-payer position can never be confused with a one-byte marker.

## Authorization model

Every AA transaction acts **for exactly one account** — one address holding the balance, the nonce, and the keychain state. Root, admin, and access are *authorization levels* for that account, not separate accounts.

| Authority | What it is | Manage keys | Deploy contracts | Revocable | Restrictions |
|---|---|---|---|---|---|
| **Root key** | The account's own key. Implicit, exactly one, and its address *is* the account. | Yes, unconditionally | **Yes** | No — it is the account | None; bypasses scope, limit, and expiry |
| **Admin key** | A delegated key authorized with the admin flag; a privileged access key. | Yes: authorize and revoke other keys | No | Yes | Must be unrestricted: no scope, no limit, no expiry |
| **Access key** | An ordinary delegated key, often used as a session key. | No | No | Yes | Bound by its call scope, spend limit, and expiry |

The model is deliberately **flat**: any admin key can revoke any sibling admin key and any access key, but **no delegated key can revoke root**. Two tiers — the account itself, and everything it delegates — is all the precompile has to encode and enforce.

!!! warning "A compromised admin key can lock you out of your session keys"
    Because an admin key can revoke every *other* delegated key, a compromised admin key can lock an account out of its own session keys. It can never revoke root, deploy contracts, or remove the account's own authority, so root always retains control. Issue admin keys accordingly.

## Signature schemes

Any of three schemes may serve as a root key or a keychain key. Root is *not* restricted to secp256k1.

| Scheme | Use | How the key is identified |
|---|---|---|
| **secp256k1** | Ordinary account key | The usual address, **recovered** from the signature |
| **P256** (secp256r1) | Passkey or secure-enclave key | **Derived** from the public key: the last 20 bytes of the hash of its two coordinates |
| **WebAuthn** | Browser or device passkey assertion | Same derivation as P256; the signature is a WebAuthn assertion over a P256 key |

### Why P256 matters, and what it changes

P256, also called secp256r1, is the curve consumer security hardware actually implements: Apple's Secure Enclave, Android Keystore, Windows Hello, TPMs, and security keys. The WebAuthn standard is built on it. Signing with a fingerprint or face scan means having that hardware sign with a P256 key that never leaves the chip, and the user never handles a seed phrase.

The practical difference from secp256k1 is **recoverability**. Ethereum's signatures allow the public key to be recovered from the signature itself, which is why a transaction never carries its sender's address. P256 and WebAuthn signatures are not recoverable, so the signature must carry the public key alongside the two signature scalars, and the key identifier is derived from that public key. The derivation is Ethereum's own address formula, applied to a P256 public key, so the result is an unremarkable 20-byte address.

A passkey may be an account's **root** key, in which case that derived identifier *is* the account's address. On-chain such an account is ordinary — it holds a balance and a nonce like any other. Two things set it apart:

- **No secp256k1 private key controls it**, so it can act *only* through `0x76`. It cannot sign a legacy or EIP-1559 transaction, ever. In particular it cannot reach the keychain precompile with a plain transaction the way a conventional account can.
- **It exists from the moment the passkey is created**, with a zero balance and no means of funding its own first transaction. The passkey experience has no step where the user acquires the native coin.

For a conventional account, sponsorship is a convenience. For a passkey-rooted account it is effectively the only way to get started.

### Consensus-enforced verification rules

Every node re-verifies these deterministically. A client that deviates produces a transaction that never lands.

For **P256**, and for the inner signature of a WebAuthn assertion:

- the public key must be a valid point on the curve;
- the second signature scalar must be in its low range, so normalize before submitting; malleable high values are rejected;
- the signature must verify against the message hash.

For **WebAuthn**, additionally:

- the authenticator data must be exactly 37 bytes, so no extensions: the extension-data and attested-credential-data flags must be clear;
- at least one of the user-present or user-verified flags must be set;
- the client data type must be the WebAuthn "get" assertion type;
- the client data challenge must equal the transaction's signing hash, encoded as unpadded base64url;
- the verified message is the hash of the authenticator data concatenated with the hash of the client data.

A standard browser passkey assertion whose challenge is the transaction's signing hash satisfies all of the above.

## How a transaction is signed

The outer signature is self-describing. The chain reads the scheme and the authorization form out of the signature bytes, not out of body fields, and a bare 65-byte signature is unambiguously secp256k1 — which keeps the common case byte-compatible with ordinary tooling.

**Root signature.** The account signs the signing hash directly. The account is recovered or derived from the signature itself, so it appears nowhere else in the transaction and the signature is inherently bound to it.

**Keychain signature.** An access key signs on behalf of an account. Because the same access key may be authorized on several accounts, the signature **carries the account address** as a field, and the inner signature is therefore taken over an account-bound digest: a domain-separator byte, the signing hash, and the account address, hashed together.

That domain separation is the point. If an access key signed the plain signing hash, a signature made for one account could be replayed against another simply by rewriting the declared account field — a cross-account replay the key holder never intended. Mixing the account into the digest pins the signature to the one account it was made for. Root signatures need no such step, since their account comes from the signature itself.

!!! note "Only the account-bound wrapping is accepted"
    An earlier form in which the inner key signs the bare signing hash is rejected outright. BNB NewL1 runs one configuration from genesis, so there is no legacy era to remain compatible with.

## The AccountKeychain precompile

Delegated-key state lives in two enshrined precompiles:

| Precompile | Address |
|---|---|
| `AccountKeychain` | `0x0000000000000000000000000000000000004000` |
| `SignatureVerifier` | `0x0000000000000000000000000000000000004001` |

These are **stateful** precompiles: they read and write through the EVM journal, so their storage participates in the block's state root exactly like contract storage. The `0x…4000` band is reserved for AA and is disjoint from the BSC system-contract addresses.

### Call surface

| Call | Purpose |
|---|---|
| `authorizeKey(keyId, sigType, expiry, isAdmin, scoped, allowedCalls, limited, limit, period)` | Authorize a key for the calling account |
| `revokeKey(keyId)` | Permanently revoke a key for the calling account |
| `getKey(account, keyId)` | Read a key's stored metadata |
| `isActiveKey(account, keyId)` | Whether a key is registered, unrevoked, and unexpired |
| `transactionKey()` | The key identifier that signed the current AA transaction; zero means root |
| `transactionAccount()` | The account the current AA transaction acts for |
| `transactionOrigin()` | The origin of the current AA transaction |

`keyId` is always the 20-byte identifier derived from the signing key: the hash-derived value for a passkey, the recovered address for secp256k1.

### Authorization parameters

| Parameter | Meaning |
|---|---|
| `keyId` | The key's identifier. |
| `sigType` | Signature scheme of the key: secp256k1, P256, or WebAuthn. |
| `expiry` | Expiry in unix seconds. Passing zero requests a key that never expires. |
| `isAdmin` | Whether this is an admin key, which must then be unrestricted. |
| `scoped` | Whether the key is restricted to `allowedCalls`. |
| `allowedCalls` | Target-and-selector scopes. At most 256 for a direct call, 32 inline, each with at most 64 selectors. |
| `limited` | Whether a native spend cap applies. |
| `limit` | The native spend cap, in wei. |
| `period` | The cap's window in seconds. Zero means a single lifetime bucket. |

The metadata `getKey` returns covers the signature scheme, expiry, whether the key is revoked, whether it is an admin key, whether it is scoped, the spend limit, the spend period, and the amount consumed in the current period. A never-authorized or revoked key reports blank metadata.

### Two ways to authorize a key

| Path | How | Max scopes |
|---|---|---|
| **Inline** | Attach a key authorization to an AA transaction. It is realized as an `authorizeKey` call **prepended** to the batch, so it is atomic with the calls that follow. | **32** |
| **Direct call** | Call `authorizeKey` as root or an admin key — including **from a plain, non-AA transaction** sent by the account itself, since the account being the origin makes it root. | **256** |

Either path allows at most **64 selectors per scope**. The inline path is bounded more tightly because its limits are signed into the transaction and checked statically before submission.

The direct-call path is worth emphasizing: a conventional account **does not need to send an AA transaction just to provision a key.** An ordinary transaction to the keychain address provisions the account's first passkey or session key. The exception is a passkey-rooted account, which has no way to sign an ordinary transaction and must be bootstrapped through a sponsored AA.

### Call scope

| Configuration | Effect |
|---|---|
| Unscoped | Any target, any calldata |
| Scoped with an empty list | Deny all |
| A target with no selectors | That target only, any calldata |
| A target with selectors | That target, and only those 4-byte function selectors |

Scope is enforced per call during execution, against the key that signed the transaction. A root signature bypasses scope entirely.

Scope is not free: each target and each selector is a storage slot the authorizing call pays for. See [Gas](#gas).

### Spend limit

A limited key counts its **native outflow** against the cap: both the value its calls move and the gas it pays when self-paying. Under the chain's prepaid fee model that gas figure is the full declared `gas_limit`. A sponsor's gas never touches the key's limit. A zero period is a lifetime bucket; a non-zero period is a rolling window anchored at the key's authorization time.

Counting self-paid gas against the limit is deliberate: a session key with a small daily cap should not be able to burn unbounded gas. Size limits to cover gas as well as the value you intend to move.

On a reverted batch the accounting splits. The gas still counts against the limit, because it was really charged. The rolled-back value does not. The spend is recorded after the batch rolls back, so it survives the rollback.

### Revocation

Revoking a key permanently **tombstones** it. It stops working immediately, and the same identifier can never be authorized again on that account.

Note also that authorization refuses an identifier whose slot is already occupied, **including an expired-but-not-revoked key**: expiry alone does not free the slot. Rotation means authorizing a fresh identifier.

### Per-transaction authorization context

A precompile ordinarily sees only its caller and the transaction origin. It cannot tell which of an account's authorized keys signed the transaction it is serving. The executor therefore **injects** that context — the signing key identifier, the acting account, and the origin — immediately before the transaction runs, and the three `transaction*` getters expose it.

Transient storage is the right vehicle: the EVM clears it at every transaction boundary, so the context cannot leak across transactions and no module-global state is introduced.

This gives applications something no ERC-4337 wallet can offer the contracts it calls: a contract can read *which session key* is acting, and require root authorization for a sensitive action while accepting a session key for routine ones.

### SignatureVerifier

The companion precompile exposes the chain's signature machinery to contracts directly, with four operations: recover the signer of a hash; verify that a signature over a hash matches a given identity, in any of the three schemes; verify that a signature is a keychain wrapping whose inner key is currently active on an account; and the same check additionally requiring that inner key to be an admin key.

The practical value is that a contract can validate a passkey signature natively instead of shipping its own Solidity verifier, and can accept "any active key of this account" as authorization.

## Atomic batch execution

The AA handler's batch lives entirely in the execution phase. Everything transaction-level — nonce bump, fee deduction, gas settlement, fee credit — happens in the standard phases and settles exactly once, whatever the batch does. Two of those phases are adjusted for the chain rather than for AA: the fee is credited to the system address, and the prepaid model charges the declared gas limit instead of refunding unused gas.

The batch itself runs inside **one journal checkpoint**:

1. Take a checkpoint.
2. For each call in order, point the transaction environment at it, run one message frame, and thread the remaining gas forward.
3. If any call fails, **revert the checkpoint**, so every call rolls back, and return the failed frame.
4. If all succeed, **commit the checkpoint**.

So a batch is all-or-nothing. Approving a token and then swapping it either both take effect or neither does, with one signature, one nonce, and one receipt.

### Ordering rules

- **Contract creation may only be the first call** in the batch, and only a **root**-signed AA may create. A keychain-signed creation is rejected statically.
- **An inline key authorization cannot be combined with a creation call.** The authorization is prepended as a call, which advances the transaction nonce, and a creation frame advances it again — two bumps for one AA, which TxStream accounts as one. Authorize and create in separate transactions.

### Authorize-and-use

An inline authorization runs as the batch's first call, under the *authorizing* key, after which the signing key is restored so the user calls are attributed to, and enforced against, the **newly authorized** key. One transaction can therefore mint a session key and take its first action with it, atomically. If any call reverts, the grant reverts with it.

This is the recommended shape. Splitting authorize-and-use across two transactions works, but the second was admitted against a *projection* of the first, and a reordering or revert in between can invalidate it. See [Failure model](#failure-model).

## Gas

!!! warning "BNB NewL1 charges the declared `gas_limit`, not the gas used"
    Under the chain's prepaid fee model a user transaction pays its declared limit times the effective price regardless of what it actually consumes: unused gas is **not** reimbursed and post-execution storage refunds are void. Receipts report gas used equal to the gas limit.

    This is a property of the chain, not of AA — standard transactions are charged the same way. The reason is ordering: blocks are packed by *declared* gas before being executed, so declared gas is the block space actually sold. Charging gas-used would hand out free block space and let an ordered-but-reverting transaction underpay for it.

    **Consequence: over-estimating is not insurance, it is spend.** Size the gas limit tightly, and prefer `newl1_estimateGas` over hand arithmetic — it prices the exact batch, signature scheme, and keychain state your transaction will meet. Read-only simulation through `eth_call` and `eth_estimateGas` runs with the prepaid model off, so those still report actual usage.

An AA pays normal EVM gas for its calls, plus an **intrinsic surcharge** for the AA machinery it actually uses, layered on stock intrinsic gas.

| Component | Surcharge |
|---|---|
| secp256k1 signature | 0 — stock recovery |
| P256 signature verification | 5,000 |
| WebAuthn assertion verification | 5,000 plus 16 per byte of authenticator data |
| Keychain access-key check | 3,000 |
| Inline key authorization, base | 27,000 |
| …carrying a spend limit | additional 22,000 |
| Calldata of every call after the first | 16 per non-zero byte, 4 per zero byte |

Two subtleties in that last row. Stock intrinsic gas only charges the **first** batch call's calldata, because that call seeds the transaction's data field. Later calls run as frames that pay no transaction-level calldata cost, so the surcharge charges them at the standard rate — never free, but never subject to the higher calldata *floor* rate. And with an inline authorization the prepended authorizing call *is* the first batch call, so **all** user calls are charged through the surcharge.

### Call scope is priced per storage slot

Scope is **not** an intrinsic surcharge. The authorizing call is metered inside the precompile per storage slot written, at roughly 20,000 each, on top of a base of about 5,000.

| Grant | Approximate cost |
|---|---|
| Unrestricted key | 25,000 |
| Key record, plus each scope target, plus each selector | 5,000 base, then 20,000 per slot |
| Adding a spend limit | Three more slots, so about 60,000 more |

This matters because it is enforced *at the precompile*, not at admission. A scoped authorization that clears the pre-submission floor but cannot pay for its scope slots is **admitted and then fails on-chain**. Size the gas limit for the scope you grant.

### The calldata floor and the declarable range

On top of intrinsic gas, calldata pays the EIP-7623 floor: roughly 40 gas per non-zero byte and 10 per zero byte of the transaction's **leading** call. For calldata-heavy transactions this floor, not the surcharges, is the dominant term.

The minimum declarable gas limit is therefore the larger of two quantities: stock intrinsic gas plus the AA surcharge, or the calldata floor. A gas limit below that minimum is **rejected at submission**; it is not admitted and then failed on-chain.

There is a hard ceiling at the other end. A user transaction may declare at most **16,777,216** gas, mirroring BSC's per-transaction cap. TxStream admission rejects an over-cap transaction, and a block containing one is consensus-invalid. The cap is fixed protocol configuration, not a command-line option and not a governed parameter, and it is independent of the block gas limit.

??? example "Worked example — sizing the gas limit"
    A keychain-signed, self-paid AA with one call carrying 4,000 bytes of calldata (3,600 non-zero and 400 zero) and an access list of 3 addresses and 8 storage keys:

    | Term | Value |
    |---|---|
    | Base transaction | 21,000 |
    | Calldata, at 16 and 4 per byte | 59,200 |
    | Access list, at 2,400 per address and 1,900 per key | 22,400 |
    | **Stock intrinsic subtotal** | **102,600** |
    | Keychain key-check surcharge | 3,000 |
    | **Intrinsic plus surcharge** | **105,600** |
    | Calldata floor, at 40 and 10 per byte | 169,000 |
    | **Minimum declarable limit** — the larger of the two | **169,000** |

    The calldata is large enough that the floor binds, so a gas limit under 169,000 is rejected. Add the call's own execution gas on top: if the call needs about 40,000, the floor still dominates, and 169,000 is the figure to declare. If the call needs about 300,000, then intrinsic plus surcharge plus execution — about 405,600 — overtakes the floor and becomes the figure to declare instead.

    Note what the prepaid model does to the usual advice about safety margins. You pay every gas unit you declare, so a margin is a real cost, not free insurance. Declare the estimate, not the estimate plus half again.

    Two adjustments for other shapes. For batches, stock intrinsic gas and the calldata floor are computed on the first executed call's calldata only; every other call's calldata is charged at the standard rate as a surcharge, never at the floor rate. For an inline authorization, the authorizing call is the first executed call, so every user call's calldata goes through the standard-rate surcharge.

### Fee routing

Whoever pays, the account or its sponsor, an AA's fee is credited to the system address rather than to the block's beneficiary, through the same path standard transactions use. At the end of the block the accumulated balance is forwarded into the validator set contract, so the BEP-95 burn and the system-reward split cover AA fees exactly as they cover standard ones. The base fee is pinned at zero, so the whole execution fee reaches the system address.

Routing is a property of the chain's economics, not of a transaction type. Paying the proposer directly would hand it the whole of every AA fee instead of the post-split share, make the burn rate a function of AA adoption, and give validators an ordering incentive to prefer `0x76`.

## Fee sponsorship

Any AA can be paid by a **fee payer** instead of the account itself. The sponsor signs the fee-payer hash for that specific account, and the account includes the result in the transaction.

- Gas is charged to the **sponsor**.
- The account only advances its nonce and pays any native value its calls move.
- **Self-sponsorship is rejected.** The fee payer must be a different account.
- A sponsored AA is admitted only if the sponsor can cover the gas reservation *and* the account can cover the total value its calls send.

### Signing order

**The sponsor signs first; the account signs last.** Reversing the order breaks verification.

1. **Fill in the body** — chain id, nonce, calls, fees, gas limit, any validity window, and an inline key authorization if you are granting a key. Leave the fee-payer signature empty for now.
2. **Sponsor signs** the **fee-payer hash**, for sponsored transactions only. This is a *distinct* digest from the one in the next step; do not sign the same hash twice.
3. **Account signs** the **signing hash**, which now commits to the presence of a fee payer. A root key signs it directly. A keychain key signs the account-bound digest instead, and the result is wrapped together with the account address.
4. **Assemble and submit** — encode with type `0x76` and send it through `eth_sendRawTransaction`.

The account's signing hash commits to *whether* the transaction is sponsored, so attaching a sponsor after the account signed would change the recomputed hash and fail verification. The sponsor's own hash is built from the account address and the body only, never from the account's signature, which is why the sponsor can sign first.

For a self-paid AA the order is simply fill, account signs, assemble. Either way, **the account is always the last signer.**

## Validity windows

The two validity bounds time-box an AA against the **block timestamp**, in unix seconds. The lower bound makes the transaction includable only in a block at or after that timestamp; the upper bound only in a block strictly before it.

Both are consensus rules, enforced by the proposer and by importing nodes. On top of them sits a **local acceptance policy** so a node does not hold transactions indefinitely:

| Bound | Policy |
|---|---|
| Lower bound | Must be no more than **120 seconds** in the future, or the node rejects the transaction at submission |
| Upper bound | Must be at least **3 seconds** in the future, or the node rejects or evicts it |

A not-yet-valid transaction is held and promoted automatically when its window opens. An expired one is dropped.

## Failure model

Where a problem surfaces determines what it costs. Every failure is one of three classes.

| Class | Meaning | Receipt | Gas charged | Nonce advances |
|---|---|---|---|---|
| **A1** | **Statically** invalid: wrong on its face, independent of chain state. Caught at submission *and* re-checked at block import. | no | **no** | no |
| **A2** | **State-dependent** invalid: only wrong given chain state. Rejected at submission and re-checked before packing; one that still reaches execution lands as a failed receipt. | yes, failed status | full gas limit | **yes** |
| **B** | Valid and **executes**, but a call reverts or runs out of gas. | yes, failed status | full gas limit | yes |

The rule of thumb is that **anything that lands costs you** — and under the prepaid fee model it costs the whole declared gas limit, which is also what a fully successful AA pays. So the gas column really only separates A1, which never lands and is never charged, from the two that land. A2 and B differ in *why* they failed and in whether a key spend is recorded, not in price.

**Class A1** covers a bad signature, a wrong chain id, a non-zero second nonce dimension, an empty call list, more than 64 calls, a misplaced creation, a keychain-signed creation, an inline authorization combined with a creation, an inline authorization over the 32-scope or 64-selector cap, a priority fee above the fee cap, an oversized transaction, a gas limit below the required floor or above the per-transaction cap, a validity window outside the acceptance policy, self-sponsorship, and set-code (EIP-7702) or blob (EIP-4844) transaction types.

A block that carries a statically invalid AA is **rejected wholesale** at import. AA safety is enforced by validation on every path, which is precisely why no off-switch is needed.

**Class A2** covers an access key that is revoked, expired, out of scope, or over its spend limit, and a fee payer that cannot cover gas. The atomic checkpoint reverts every call and no key spend is recorded, but the transaction pays its full declared gas limit.

**Class B** is the normal "transaction failed, you paid for the attempt" outcome, applied to the whole atomic batch. All user-call state rolls back, the receipt reports failure, the declared gas limit is charged, and the nonce advances. For a limited access key, that gas counts against the limit; the rolled-back value does not.

A note on underpricing: a fee cap below the block base fee is *not* a static rejection, because the base fee is block context and changes over time. Such a transaction simply is not packed while it is underpriced, and a block that nonetheless carries one is rejected at import, so it can never land underpriced.

### Why keychain permission is an execution-time check

TxStream's admission checks are **best-effort**; execution is authoritative.

To let a dependent transaction be admitted optimistically, TxStream *projects* exactly two kinds of pending keychain change against committed state: an inline key authorization, and a revocation — whether the revocation arrives inside an AA batch or as a plain transaction to the precompile. At most one projected change per key is allowed at a time.

That projection is what makes optimistic admission possible, and it is also why an admitted transaction can still fail. The key may be revoked, or its limit consumed, by a later-ordered change, or a projected authorization may never commit. The execution-time check is the backstop that settles those stragglers, as Class A2: a failed receipt charging the full gas limit.

A **direct** authorization call is not part of the projected set. A later use of a key authorized that way simply waits until the call has executed and committed, so there is nothing to project and nothing to race.

One narrow shape is worth knowing. If a key is authorized **inline** in one transaction and a separate transaction uses it, and the first is later reordered out or reverts before executing, the second was admitted against a projection that never materialized. Because the projection lives at the admission and import boundary rather than inside the first transaction's execution, the second can surface as a block-level import error rather than a clean per-transaction receipt. It is recoverable — resubmit it — but it is the reason to prefer combining authorize-and-use into one atomic AA, or to gate the dependent transaction on the authorizing one having already executed.

!!! tip "Passing submission does not guarantee success"
    For an access-key transaction, size the gas limit for the full batch, and do not assume a keychain permission problem always fails for free at submission.

## How to use it

The end-to-end sequence, pulling together the pieces above.

1. **Build the transaction body** with the fields you need: chain id, nonce, the ordered call list, the EIP-1559 fee fields, and a gas limit. Add a validity window and an inline key authorization if you want them. Leave the fee-payer signature empty unless the transaction is sponsored.
2. **Size the gas limit** with `newl1_estimateGas` rather than by hand, and remember you pay what you declare.
3. **If sponsored, the fee payer signs first**, over the fee-payer hash, which commits to the transaction body and the account it is sponsoring.
4. **The account signs last.** A root key signs the signing hash directly; a keychain key signs the account-bound digest that prevents cross-account replay.
5. **Submit** the encoded transaction through the standard `eth_sendRawTransaction`.
6. **Provision secondary keys** either inline, with up to 32 scopes, or through a direct authorization call, with up to 256 scopes and 64 selectors per scope. A conventional account can make that direct call from an ordinary transaction; a passkey-rooted account must use a sponsored AA.
7. **Revoke keys** when you are done with them. Revocation is immediate and permanent.
8. **Read key state** with [`newl1_getKey`](../developers/json-rpc.md#newl1_getkey), and read receipts with `eth_getTransactionReceipt`, where the type is `0x76` and the status distinguishes a successful batch from a reverted one.

## Example flows

**Simple transfer, root-signed and self-paid.** Build a single call moving native value, sign the signing hash with the account's key, submit, and poll for the receipt.

**Atomic batch.** Put several calls in one transaction, such as an approval followed by a swap, or a set of transfers. Either all of their effects persist, or none do.

**Sponsored passkey transaction.** Derive the account address from a passkey public key. A funded relayer signs the fee-payer hash for that account; the passkey then signs the transaction. Gas is paid by the relayer, and the passkey account advances its nonce. This is how a brand-new device with no private key and no balance makes its first transaction.

**Time-boxed, spend-capped session key.** Root submits an AA carrying an inline authorization for a key scoped to one contract with a native spend cap and an expiry — the shape a game or a trading bot wants. The key then signs its own AAs within that grant. Out-of-scope or over-limit uses are rejected at submission. Root or an admin key can revoke it at any time, permanently.

**Admin key provisioning for a team.** Root authorizes an unrestricted admin key. That admin key then provisions scoped access keys for individual team members or services, without root having to sign each grant, and can revoke any of them later.

**Scheduled or expiring transaction.** Set a near-future lower validity bound, within the acceptance horizon, and an upper bound as an expiry. The node holds the transaction until its window opens, then it becomes eligible for inclusion. An expired one is dropped.

**Authorize and use in one transaction.** Combine an inline authorization with the new key's first call. The authorization runs first under root, the call runs under the new key, and both roll back together if anything fails.

## FAQ

**Do I need to deploy anything to use AA?**
No. Any existing account can send a `0x76` transaction immediately. Nothing is deployed, nothing is delegated, and the address does not change.

**Is my address different when I use AA?**
No. The account is the same address, and it is the transaction origin for the calls in the batch, so contracts that inspect the origin or the caller behave as they would for an ordinary transaction.

**Do I need a bundler?**
No. AA transactions travel the same path as every other transaction, and they compete in the same fee market.

**Can a contract tell that it was called through an AA transaction?**
Yes, and usefully so. The keychain precompile reports the key identifier that signed the transaction, with zero meaning root, and the account it acts for. An application can require root authorization for a sensitive action while accepting a session key for routine ones.

**What happens if one call in a batch reverts?**
The whole batch reverts; every call's state changes roll back. The transaction still lands with a failed status, the declared gas limit is charged, and the nonce advances.

**Do I get unused gas back?**
No. The chain uses a prepaid fee model: every user transaction, AA or standard, pays its declared gas limit, with no reimbursement for unused gas and no post-execution storage refund. Blocks are packed by declared gas before execution, so declared gas is the block space you are actually buying. Size the limit tightly and use `newl1_estimateGas`.

**Can I authorize a key and use it in the same transaction?**
Yes, and it is the recommended pattern. The inline authorization runs first under root, then the user calls run under the newly authorized key, all inside one atomic checkpoint.

**Can an access key be revoked out from under a transaction I already sent?**
Yes. Admission-time keychain checks are a best-effort projection; the authoritative check happens at execution. A revoked key surfaces as a Class A2 failed receipt charging the full gas limit.

**Can an admin key take over my account?**
An admin key can revoke and authorize any *other* delegated key, including other admin keys, but it can never revoke root, deploy contracts, or remove the account's own authority. Root always retains control. Issue admin keys with the lockout risk in mind.

**Can I sponsor my own transaction?**
No. Self-sponsorship is rejected; the fee payer must be a different account.

**Why is my sponsored transaction failing verification?**
Almost always signing order. The sponsor must sign **before** the account, because the account's signing hash commits to whether a fee payer is present. The two also sign different digests.

**How does a passkey-only account provision more keys?**
Through a sponsored AA. A passkey-rooted account cannot sign an ordinary transaction, so the shortcut of calling the keychain precompile directly from a plain transaction is not available to it.

**Is there a parallel-nonce mode?**
Not in the current version. The second nonce dimension exists in the encoding and must be zero; parallel lanes are reserved for a future version.
