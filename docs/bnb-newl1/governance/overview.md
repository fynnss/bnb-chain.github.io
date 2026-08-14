---
title: Governance - BNB NewL1
---

# Governance

BNB NewL1 uses fully on-chain, token-weighted governance — the same Compound/OpenZeppelin `Governor` + `Timelock` engine that powers BNB Smart Chain governance — to control protocol parameters. There is no admin key or externally-owned account that can bypass this process: every governed contract only accepts changes routed through the governance flow.

## What's governable

Protocol and system parameters — for example staking and slashing constants, and BNB NewL1–specific configuration such as the [multi-lane blockspace](../core-concepts/multi-lane.md) quotas in `LaneRegistry`. Validator-set **membership** itself is not a governance vote; it's decided automatically by stake-weighted election each epoch (see below), while the parameters that shape staking and elections are governable.

## Voting power

1. Stake bridged BNB with `StakeHub`.
2. Staking mints **govBNB** 1:1 against staked value — a non-transferable `ERC20Votes` token that exists purely as a voting ledger.
3. govBNB only counts as active voting power once **delegated** (to yourself or to a validator). Holding undelegated govBNB carries zero voting weight.

## Lifecycle

1. **Propose** — a proposer with enough delegated govBNB to clear the proposal threshold submits a proposal targeting `GovHub.updateParam(key, value, target)`. Multiple parameter changes can be bundled atomically into a single proposal.
2. **Vote** — govBNB holders cast votes during the voting period.
3. **Quorum** — the proposal succeeds once quorum is reached and the voting period ends.
4. **Queue** — a succeeded proposal is queued on the `Timelock` contract.
5. **Timelock delay** — a mandatory minimum delay must elapse before the change can execute, giving the community time to react to a passed but not-yet-active change.
6. **Execute** — the `Timelock` calls `GovHub.updateParam`, which forwards `updateParam(key, value)` to the target system contract.

Two access-control gates make this the only path to change anything governed: every governed target only accepts calls from `GovHub`, and `GovHub` only accepts calls from the `Timelock`.

!!! note
    Some parameters (for example, [multi-lane](../core-concepts/multi-lane.md) quota changes) take effect only after an additional short activation delay on top of the timelock delay, since node clients read that configuration from a slightly-lagged executed-state checkpoint rather than the very latest block.

## How to participate

- **Delegate/stake**: bridge BNB to BNB NewL1, then call `StakeHub.delegate(validatorOperator, delegateVotePower=true)` — this also mints govBNB automatically.
- **Become a validator**: register via `StakeHub.createValidator`, which verifies your BLS vote key.
- **Vote**: delegate your govBNB (to yourself or a validator), then cast votes on active proposals through the `Governor` contract.
- **Propose**: hold or have delegated enough govBNB to clear the proposal threshold, then submit a proposal via `Governor.propose`.

`Governor` and `Timelock` are compiled from the same OpenZeppelin/Compound contracts BSC uses (with BNB NewL1's own timing constants), so tooling built against a standard OZ Governor interface — e.g. Tally — should work without changes; this manual hasn't independently verified any specific third-party tool against a live deployment.

## Validator-set rotation

Validator-set membership updates automatically once per epoch (see [Consensus](../core-concepts/consensus.md)): at the first block of each epoch, the protocol reads the top validators by voting power from `StakeHub` and updates the active validator set on-chain — no separate governance proposal is required for this routine rotation.

See [System Contracts](./system-contracts.md) for the full list of contracts involved and their addresses.
