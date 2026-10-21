# Foundry Under the Hood: Forge, Cast, and Anvil

The [README](../README.md) explains *what* each tool is for. This document explains *how* each one works, so that when something breaks you have a mental model to debug with.

All three tools are written in Rust and share the same core pieces:

- **[revm](https://github.com/bluealloy/revm)** is a Rust implementation of the Ethereum Virtual Machine (EVM). Forge runs your tests on it, and Anvil runs your local chain on it.
- **[Alloy](https://alloy.rs/)** is a Rust library for Ethereum types, ABI encoding, signing, and JSON-RPC. Cast and Forge use it to talk to nodes.

Because they share an EVM, a contract behaves the same way in `forge test` as it does on Anvil.

---

## Forge

Forge has four main jobs: installing dependencies, compiling, testing, and running scripts.

### `forge install`: dependencies are git submodules

Forge has no package registry. `forge install OpenZeppelin/openzeppelin-contracts` runs `git submodule add` to clone the repo into `lib/openzeppelin-contracts`. It records the pinned commit in `.gitmodules` and in the parent repo.

Import paths like `@openzeppelin/contracts/token/ERC20/ERC20.sol` are resolved through **remappings**: prefix rewrites set in `foundry.toml` or `remappings.txt`, for example:

```
@openzeppelin/contracts/=lib/openzeppelin-contracts/contracts/
```

Forge passes these remappings to the Solidity compiler. If an import fails to resolve, check the remappings first (`forge remappings` prints them).

### `forge build`: driving the Solidity compiler

Forge does not compile Solidity itself. It orchestrates `solc`, the official compiler:

1. **Pick a compiler.** Forge reads each file's `pragma solidity` line and picks a matching `solc` version, unless `foundry.toml` pins one. Missing versions are downloaded automatically and kept in `~/.svm/`.
2. **Skip unchanged files.** `cache/solidity-files-cache.json` stores a content hash for each source file and its imports. Only files that changed (or whose dependencies changed) are recompiled.
3. **Call solc.** Forge sends solc a JSON request containing the sources, remappings, and optimizer settings. solc returns JSON describing every contract.
4. **Write artifacts.** For every contract, Forge writes `out/<File>.sol/<Contract>.json`. It contains:
   - `abi`: the contract's interface (functions, events, errors). Frontends and Cast use this to encode calls.
   - `bytecode`: the *creation* code. A deploy transaction sends this, and it runs the constructor once.
   - `deployedBytecode`: the *runtime* code the constructor returns, which is what actually lives on-chain.
   - `metadata`, `methodIdentifiers`, and source maps for debugging.

In this project, the frontend uses those `abi` fields (copied or generated) to talk to the contracts, and `npm run deploy:local` reads `bytecode` from `out/` to deploy the factory.

### `forge test`: tests are contracts

A Foundry test file is a contract that inherits from `forge-std/Test.sol`. When you run `forge test`:

1. Forge compiles everything, then starts an **in-process EVM** (revm). No node or RPC is involved, which is why tests are fast.
2. For each test contract, it deploys the contract into that EVM and calls `setUp()`.
3. It snapshots the state after `setUp()`. Each function whose name starts with `test` then runs against its own copy of that snapshot, so tests cannot leak state into each other.
4. A test **fails if it reverts** (for example on a failed `assertEq`, which reverts) and passes otherwise.
5. Test contracts run in parallel across CPU cores.

**Cheatcodes** are how tests bend the rules of the chain. `forge-std` defines `vm` as an interface at the fixed address `0x7109709ECfa91a80626fF3989D68f67F5b1DD12D`. No contract actually lives there. Forge hooks into the EVM (an "inspector") and intercepts every call to that address, then performs the effect directly:

| Cheatcode | What Forge does |
|---|---|
| `vm.prank(alice)` | Makes the next call's `msg.sender` equal `alice` |
| `vm.deal(alice, 1 ether)` | Sets `alice`'s balance directly in state |
| `vm.warp(t)` | Sets `block.timestamp` |
| `vm.expectRevert(...)` | Marks the next call as expected to revert, and fails the test if it doesn't |
| `vm.expectEmit(...)` | Compares the next emitted event against an expected one |

Note that `vm.prank` applies to the *next call*, wherever it appears. In `vm.prank(buyer); token.mint{value: token.quote(3)}(3);`, the nested `quote` call runs first and uses up the prank, so the mint runs as the test contract. Compute the cost on its own line before the prank.

**Fuzz tests.** If a test function takes parameters, for example `testMint(uint256 amount)`, Forge treats it as a property test. It generates random inputs (256 runs by default), mixing in "interesting" values such as 0, `type(uint256).max`, and constants found in your bytecode. When a run fails, Forge **shrinks** the input to a smaller one that still fails and prints it. Use `vm.assume` or `bound` to restrict inputs to valid ranges.

**Traces.** With `-vvvv`, Forge prints a call tree for each test: every call, its arguments, return values, emitted events, and where it reverted. The decoded names come from the ABIs in `out/`. This is the fastest way to find out *why* a test failed.

### `forge script`: simulate, then broadcast

A Foundry script is a Solidity contract with a `run()` function. Forge:

1. Runs `run()` in a local EVM (optionally forked from a real RPC).
2. Records each call made between `vm.startBroadcast()` and `vm.stopBroadcast()` as a transaction to send.
3. If you passed `--broadcast`, it signs those transactions and sends them to `--rpc-url`, then saves the results in `broadcast/`.

This project deploys with a viem script (`npm run deploy:local`) instead, because `forge script` crashed with an internal error against Anvil while the project was being built. See the playbook for details. Both approaches send the same deploy transactions.

---

## Cast

Cast is a command-line **JSON-RPC client** plus a set of encoding utilities. Every Ethereum node (Anvil, Geth, Infura, and so on) exposes the same JSON-RPC API over HTTP. Cast turns friendly commands into those requests.

### Encoding a call

To call a contract, you must turn a function call into raw bytes (**calldata**):

1. Take the function signature, for example `balanceOf(address)`.
2. Hash it with keccak256 and keep the first 4 bytes. This is the **function selector**: `0x70a08231`.
3. ABI-encode the arguments: each `address` or `uint256` is padded to 32 bytes.
4. Concatenate the selector and the encoded arguments.

You can see each step with Cast:

```bash
cast sig "balanceOf(address)"                    # 0x70a08231
cast calldata "balanceOf(address)" 0xf39F...2266 # selector + padded address
cast keccak "Transfer(address,address,uint256)"  # an event topic
```

### `cast call`: reading state

```bash
cast call $TOKEN "balanceOf(address)(uint256)" $ACCOUNT --rpc-url http://127.0.0.1:8545
```

1. Cast encodes the calldata as above.
2. It sends an `eth_call` request. The node runs the call against current state and **throws away** any changes. Nothing is mined, and no gas is paid.
3. The node returns raw bytes. Cast decodes them using the return types you wrote in the second set of parentheses: `(uint256)`.

### `cast send`: writing state

```bash
cast send $TOKEN "transfer(address,uint256)" $TO 1000 \
  --private-key $KEY --rpc-url http://127.0.0.1:8545
```

A transaction needs more than calldata, so Cast fills in the rest by asking the node:

| Field | How Cast gets it |
|---|---|
| `nonce` | `eth_getTransactionCount` for the sender |
| `chainId` | `eth_chainId` (31337 on Anvil) |
| `gas` limit | `eth_estimateGas`, which runs the transaction in a simulation |
| `maxFeePerGas`, `maxPriorityFeePerGas` | Fee history and current base fee (EIP-1559) |

Then it:

1. **Signs** the transaction locally with the private key and RLP-encodes it.
2. Sends it with `eth_sendRawTransaction`. The node replies immediately with the transaction hash.
3. Polls `eth_getTransactionReceipt` until the transaction is mined, then prints the receipt (status, gas used, logs).

If you pass `--unlocked --from <address>` instead of a key, Cast skips signing and calls `eth_sendTransaction`, asking the node to sign. Anvil allows this for its own accounts. Real networks don't.

viem's `readContract` and `writeContract` follow exactly the same steps. Cast is useful for checking what the frontend *should* see.

### Other handy commands

```bash
cast balance $ACCOUNT --ether          # ETH balance
cast receipt $TX_HASH                  # receipt and logs for a transaction
cast run $TX_HASH                      # replay a mined transaction with a full trace
cast 4byte-decode $CALLDATA            # guess a function from calldata
cast to-wei 1.5 ether                  # unit conversion
cast rpc anvil_mine                    # call any raw RPC method
```

---

## Anvil

Anvil is a complete Ethereum node that runs in memory on your machine. Real nodes connect to peers, agree on blocks through consensus, and store state on disk. Anvil skips all of that: it is the only node, and it decides what goes into each block.

### Startup

When you run `anvil --host 127.0.0.1 --port 8545 --chain-id 31337`, it:

1. Creates an empty in-memory state (or loads one with `--load-state`, or lazily copies one with `--fork-url`).
2. Derives **10 accounts** from the well-known test mnemonic `test test test test test test test test test test test junk`, using the standard wallet path `m/44'/60'/0'/0/<i>`. Each gets 10,000 ETH. Account 0 is `0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266`.
3. Mines a genesis block and starts a JSON-RPC server over HTTP and WebSocket on port 8545.

The keys are public knowledge, so **never send real funds to these addresses** on any real network.

### Handling a transaction

1. A transaction arrives through `eth_sendRawTransaction` (already signed), or through `eth_sendTransaction` from one of the 10 **unlocked** accounts, in which case Anvil signs it itself. That is why this project's deploy script needs no private key.
2. Anvil checks the signature, nonce, balance, and gas, then puts the transaction in its mempool.
3. With **automine** (the default), Anvil immediately builds a new block containing just that transaction. It runs the transaction on revm, applies the resulting state changes, and computes the receipt and logs. With `--block-time 2`, it instead mines a block every 2 seconds containing whatever is pending, which is closer to a real chain.
4. Read requests (`eth_call`, `eth_getBalance`, `eth_getLogs`, and so on) are answered straight from the in-memory state.

All of this lives in RAM, so stopping Anvil throws it away. Next time, the same deploy will produce the same contract addresses (addresses come from the deployer address and its nonce), but those contracts won't exist until you redeploy.

### Test-only RPC methods

Beyond the standard `eth_*` methods, Anvil exposes methods that let you manipulate the chain. Cast calls them with `cast rpc`, and viem has a "test client" for them.

| Method | Effect |
|---|---|
| `anvil_setBalance` | Set any account's ETH balance |
| `anvil_impersonateAccount` | Send transactions as any address without its key |
| `evm_increaseTime`, `evm_mine` | Move time forward, mine a block on demand |
| `evm_snapshot`, `evm_revert` | Save the whole chain state and roll back to it later |
| `anvil_dumpState`, `anvil_loadState` | Export or import state, to survive restarts |

These are the same powers that cheatcodes give you in `forge test`, offered over RPC instead.

### Forking

`anvil --fork-url <mainnet RPC>` starts a local chain that *appears* to contain all of mainnet. Anvil doesn't download everything. When a transaction reads an account or storage slot it hasn't seen, it fetches that value from the remote node and caches it. Your transactions only change the local copy. This lets you test against real deployed contracts for free.

### A MetaMask gotcha

MetaMask remembers each account's nonce per network. After you restart Anvil, the chain is fresh (nonce 0), but MetaMask may still use the old nonce, and transactions fail with a "nonce too high" error. Fix it in MetaMask under **Settings → Advanced → Clear activity tab data**.

---

## Putting it together: one click in the frontend

When you mint tokens from a token's curve in the React app:

1. viem calls `eth_estimateGas` on Anvil, which runs the mint in a throwaway simulation. If it would revert, you see the error before signing.
2. viem asks MetaMask to sign. MetaMask shows the popup, signs with your key, and sends the transaction to Anvil with `eth_sendRawTransaction`.
3. Anvil automines a block: revm runs the token's mint function, checks the ETH sent against the curve's price, updates balances and supply, and records the events in the receipt.
4. viem polls `eth_getTransactionReceipt`, sees the transaction succeeded, then uses `eth_call` to read the new supply, proceeds, and cost quote, and React re-renders.

You could do every one of those steps by hand with `cast estimate`, `cast send`, `cast receipt`, and `cast call`. Doing so once is a good way to see exactly what the frontend is doing.
