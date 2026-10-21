# Blockchain Factory Demo

A teaching project, rebuilt from scratch with Claude Code by following [docs/PLAYBOOK.md](docs/PLAYBOOK.md). The playbook's prompts describe goals rather than implementations, so each rebuild may differ in names and layout.

The finished project is a Foundry factory that deploys ERC20 tokens with bonding-curve pricing, plus a `viem` + React frontend. It starts with fixed (non-upgradeable) tokens and later adds optional upgradeable tokens built on UUPS proxies.

To reproduce it, open Claude Code in this repo and paste the playbook's phase prompts in order.

## The tools

This project uses a small set of tools. Each has a short intro below and a note on what it does here.

### Foundry

[Foundry](https://getfoundry.sh/) is a toolkit for writing, testing, and deploying Ethereum smart contracts. It is written in Rust. You write both contracts and tests in Solidity, so you don't need a separate JavaScript test harness. It installs several command-line tools:

- **`forge`** compiles contracts, runs tests (including fuzz tests), and installs Solidity libraries.
- **`cast`** is a Swiss-army knife for talking to a chain from the shell. It can send transactions, call view functions, and decode data.
- **`anvil`** is a local blockchain (see below).

**Here:** `forge build` compiles the factory and token contracts, and `forge test` runs the Solidity test suite. `forge` also installs [OpenZeppelin Contracts](https://www.openzeppelin.com/solidity-contracts), the audited library we use for the ERC20 and UUPS proxy base contracts.

### Anvil

[Anvil](https://getfoundry.sh/anvil/overview) is Foundry's local Ethereum node. It starts a private chain on your machine in under a second. That chain comes with ten pre-funded accounts, mines blocks instantly, and costs nothing to use. Everything resets when you stop it.

**Here:** we run `anvil --host 127.0.0.1 --port 8545 --chain-id 31337` in its own terminal. The deploy script and the frontend both talk to it at `http://127.0.0.1:8545`. Its accounts are unlocked, so local deploys need no private key. Restarting Anvil wipes the chain, so redeploy afterwards.

### viem

[viem](https://viem.sh/) is a lightweight, type-safe TypeScript library for interacting with Ethereum. It plays the same role as ethers.js or web3.js. You create a *public client* to read chain state and a *wallet client* to sign and send transactions. If you give it a contract's ABI, it type-checks function names and arguments for you.

**Here:** viem is used in two places. The npm deploy scripts use it to deploy contracts to Anvil. The frontend uses it to read factory and token state (supply, mint cost quotes, proceeds), and to send create-token, mint, and withdraw transactions through the browser wallet.

### React

[React](https://react.dev/) is a JavaScript library for building user interfaces out of components. Each component is a function that returns markup based on its state. When the state changes, React re-renders the affected parts of the page.

**Here:** the frontend in `web/` is a single-page React app written in TypeScript. It lets you connect a wallet, create tokens from the factory, mint tokens priced by each token's bonding curve, and withdraw proceeds and fees.

### Supporting tools

- **[Vite](https://vite.dev/)** is the dev server and bundler for the frontend. `npm run dev` serves the app with hot reload, and `npm run build:web` produces a production bundle.
- **[TypeScript](https://www.typescriptlang.org/)** adds static types to JavaScript. It catches mistakes in contract calls and UI code before runtime.
- **[MetaMask](https://metamask.io/)** is a browser wallet extension that holds your keys and signs transactions. We point it at the local Anvil network (chain ID `31337`) so the frontend can send transactions.
- **[Node.js / npm](https://nodejs.org/)** runs the frontend tooling and the viem deploy scripts.

### How they fit together

```
 Solidity contracts ──forge build/test──▶ compiled ABI + bytecode
                                                │
                     npm run deploy:local (viem)│
                                                ▼
 MetaMask ◀──▶ React + viem frontend ◀──JSON-RPC──▶ Anvil (local chain)
```

The typical loop: edit a contract, run `forge test`, restart Anvil and redeploy, then use the React app in the browser to try it out.

For a deeper look at how `forge`, `cast`, and `anvil` work internally, see [docs/FOUNDRY_INTERNALS.md](docs/FOUNDRY_INTERNALS.md).
