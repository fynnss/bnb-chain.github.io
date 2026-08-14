---
title: Run a Local Devnet - BNB NewL1
---

# Run a Local Devnet

BNB NewL1 has no public endpoint yet, so developers run a local devnet from the client source. Network shape and endpoints: [Network Information](./network-info.md).

## Prerequisites

- Rust toolchain (pinned version fetched automatically via `rustup show` from the repository's `rust-toolchain.toml`)
- The `newl1` client repository, built from source

## Build the client

```bash
make check       # cargo check --workspace --all-targets
make test        # cargo test --workspace
make release     # cargo build --release
make maxperf     # cargo build --profile maxperf  (devnet-grade performance build)
```

## Smoke-test the binary

```bash
cargo run -- --devnet --http.addr none --ws.addr none --p2p.addr 127.0.0.1:0
```

This is the fastest way to confirm the binary runs at all: it persists genesis and starts producing and finalizing real sealed blocks. Disabling HTTP/WS/P2P binding (as above) skips exposing any external surface; drop those flags (or point them at real addresses) to expose JSON-RPC and accept peers.

## Run a local devnet (recommended)

`scripts/devnet_up.sh` boots a configurable-size devnet (1–255 validators, plus any number of non-validator observers) with real JSON-RPC, WebSocket, and P2P endpoints — this is what this manual's examples were verified against, down to a single-validator devnet:

```bash
scripts/devnet_up.sh start -v 1 -p 1   # 1 validator + 1 observer
scripts/devnet_up.sh status            # health snapshot (head/finalized/peers per node)
scripts/devnet_up.sh stop              # tear down
```

Node 0 (the validator) is reachable at `http://127.0.0.1:8545` / `ws://127.0.0.1:8546`; each subsequent node adds 10 to both ports (node 1 → `8555`/`8556`, etc).

!!! tip "Add at least one observer, even for a 1-validator devnet"
    Some features are inherently peer-to-peer — most notably [transaction pre-confirmation](../core-concepts/tx-preconfirmation.md), whose sub-block feed is delivered by P2P gossip and is **not** looped back to the producing node's own subscribers. A single fully-isolated node (`-v 1 -p 0`) still produces and finalizes blocks correctly, but its own `newl1_subscribePreconfirmations`/`newl1_subscribeSubBlocks` will show nothing. Run `-p 1` (or more) and subscribe from the non-validator observer to see the feature working.

!!! tip "Set `NEWL1_PERSIST_SYNC=safe-no-sync` when driving real transaction load"
    The default `durable` persistence mode fsyncs every commit, which can't keep pace with the network's 200 ms block interval under sustained load and will make the node abort. Set `export NEWL1_PERSIST_SYNC=safe-no-sync` before `devnet_up.sh start` for load-testing — it's crash-consistent and recovers via re-execution on restart.

To run the full scenario suite (governance, multi-lane, pre-confirmation, and privacy, each exercised end-to-end against a real running devnet) use the repository's end-to-end test harness:

```bash
e2e/run.sh              # run every scenario
e2e/run.sh --list       # list available scenarios
e2e/run.sh governance   # run a single named scenario
```

The harness requires [Foundry](https://getfoundry.sh/) (`cast`/`forge`), `jq`, `curl`, and `python3` on `PATH`, and builds the release `newl1` binary itself.

## Deploy your first contract

This walkthrough is verified end-to-end against a running local devnet, using [Foundry](https://getfoundry.sh/)'s `cast` (the `forge`/`cast` CLI pair — `forge` compiles, `cast` sends transactions and reads state).

1. **Get a funded account.** `scripts/genesis.conf` predefines a couple of pre-funded test accounts (100,000 ether each) for local use:

    ```bash
    OPERATOR_SK=$(jq -r '.test_eoas[0].sk' scripts/genesis.conf)
    OPERATOR=$(jq -r '.test_eoas[0].addr' scripts/genesis.conf)
    cast balance "$OPERATOR" --rpc-url http://127.0.0.1:8545
    ```

2. **Scaffold and build a contract.** `forge init` drops in a minimal example (`Counter.sol`, with `number()`, `setNumber(uint256)`, `increment()`):

    ```bash
    forge init my-contract && cd my-contract
    forge build
    ```

3. **Deploy it.** Supply gas price and limit explicitly rather than letting `cast`/`forge` auto-estimate them:

    ```bash
    BYTECODE=$(jq -r '.bytecode.object' out/Counter.sol/Counter.json)
    cast send --private-key "$OPERATOR_SK" --legacy \
      --gas-price 1000000000 --gas-limit 300000 \
      --rpc-url http://127.0.0.1:8545 --create "$BYTECODE"
    ```

    Note the `contractAddress` in the output.

    !!! note "`forge create` used to fail here — the RPC-side cause is fixed"
        Earlier client builds rejected the explicit `"to": null` that Foundry's `forge create` (and SDK helpers like ethers.js `ContractFactory.deploy()`) send when pre-flighting an `eth_estimateGas` for a deployment, failing with `invalid address in call object: invalid type: null, expected 20 bytes`. The current client parses `"to": null` as a contract creation, so that gap is closed. The explicit-gas `cast send --create` form above remains the walkthrough's verified path; if you hit an estimation problem on a specific client build, passing `--gas-price`/`--gas-limit` still bypasses it.

4. **Read and write it** like any EVM contract:

    ```bash
    CONTRACT=0x...   # the contractAddress from step 3

    cast call "$CONTRACT" "number()(uint256)" --rpc-url http://127.0.0.1:8545
    # → 0

    cast send --private-key "$OPERATOR_SK" --legacy \
      --gas-price 1000000000 --gas-limit 60000 \
      --rpc-url http://127.0.0.1:8545 "$CONTRACT" "setNumber(uint256)" 42

    cast call "$CONTRACT" "number()(uint256)" --rpc-url http://127.0.0.1:8545
    # → 42
    ```

This exact sequence — deploy, read, write, read — was run against a live single-validator devnet while writing this manual.

## Connecting an SDK

ethers.js, viem, and web3.py connect to the node's HTTP/WS endpoint with no custom transport. Reading balances, calling view functions, subscribing to logs, and waiting for receipts all behave like any other EVM endpoint.

Two things to set correctly before you write anything real: a real `maxPriorityFeePerGas` (the base fee is zero, so the tip is the whole price) and a tight gas limit (you're charged the limit you declare). Both, plus nonce handling, are covered in [Migrating from BSC](./migrate-from-bsc.md). For BNB NewL1–specific methods, see the [JSON-RPC Reference](../developers/json-rpc.md).
