# Project Playbook

This playbook shows how to rebuild this project step by step, in the order it was originally developed. Each phase is a prompt for a coding agent. The prompts describe **goals and constraints**, not implementations, so the agent is free to choose its own contract design, names, and UI layout.

## Project Goals

We are building a small, end-to-end blockchain app for teaching:

1. **A factory contract.** Anyone can pay a small creation fee to have the factory deploy a new token contract for them. The caller becomes the owner of that token. The factory keeps a record of every token it created, and its own owner can collect the fees.
2. **Token contracts with a bonding curve.** Each token is a standard ERC20. New tokens are minted at a price set by a **bonding curve**: a formula that gives the price of the next token based on how many already exist. For example, a flat curve charges the same price forever, and a linear curve makes each token a little more expensive than the last. The token's owner can mint, and other people can mint by paying the price the curve sets. The ETH they pay goes to the token's owner.
3. **A web interface** that makes all of this usable from a browser wallet: create tokens, browse them, see price quotes, mint, and withdraw.
4. **An upgradeable variant** (added last). Creators can pay extra for a token that sits behind a proxy, so its owner can later change the curve or upgrade the code. Keeping both kinds side by side shows exactly what upgradeability adds, and what it costs in trust.

Along the way the project should model good practice: clear access control, input validation, readable errors, events, thorough tests, and documentation that explains *why*.

**Non-goals:** production deployment, selling tokens back to the curve, price oracles, and event indexing. Some of these are listed under [Suggested Next Phases](#suggested-next-phases).

## Conventions

These few things are fixed so the checkpoints, the README, and the docs stay consistent. Everything else is up to the agent.

- **Contracts:** a Foundry project with `src/`, `test/`, and `script/`, using [OpenZeppelin Contracts](https://www.openzeppelin.com/solidity-contracts) v5.7.0.
- **Frontend:** in `web/`, built with React, TypeScript, Vite, and viem.
- **npm scripts:** `dev` (run the frontend), `build:web` (type-check and build it), `deploy:local` (deploy the factory to Anvil), `fund:local` (send test ETH to a wallet).
- **Local chain:** Anvil at `http://127.0.0.1:8545`, chain ID `31337`.

## How to Use This Playbook

Each phase has:

- **Prompt**: paste it into a coding agent as written, or treat it as a spec if you are building by hand.
- **What you should end up with**: the behavior to check for. Names and code will differ between runs.
- **Checkpoint**: commands to run before moving on.
- **Lessons learned**: problems hit during the original build. Every prompt asks the agent to read these first.

To build with Claude Code:

1. Create an empty git repo that contains only this file at `docs/PLAYBOOK.md`.
2. Install the [prerequisites](#prerequisites).
3. Open Claude Code in the repo root.
4. Paste each phase's **Prompt** in order. Wait for the phase to finish, run its **Checkpoint**, then commit and tag before pasting the next prompt:

   ```shell
   git add -A && git commit -m "Phase 1: factory and fixed token" && git tag phase-1
   ```

The order is deliberate. Tokens start out **non-upgradeable**, and upgradeability arrives only in Phase 6. The tags make the history a teaching aid: for example, `git diff phase-5 phase-6` shows exactly what upgradeability adds.

For the reasoning behind the contracts, see `docs/DESIGN.md` (written in Phase 3).

## Prerequisites

Install:

- Git
- Node.js 22+ and npm
- Foundry (`forge`, `cast`, `anvil`)
- MetaMask or another injected browser wallet

Check versions:

```shell
node --version
npm --version
forge --version
anvil --version
```

---

## Phase 1: Factory and Fixed Token

**Prompt**

```text
Read the Project Goals, Conventions, and Phase 1 Lessons learned in
docs/PLAYBOOK.md before starting.

Set up the Foundry project in this repo (keep docs/PLAYBOOK.md) and build the
first version of the two contracts:

1. A token contract: a standard ERC20 with an owner. Its creator chooses the
   name, symbol, an initial supply (minted to the owner), an optional maximum
   supply, and a bonding curve. Support at least two curve shapes: a flat
   price and a linear curve where each new token costs a little more than the
   last. The token should be able to quote the ETH cost of minting a given
   amount from its current supply. For now, only the owner can mint (within
   the cap), and holders can burn their own tokens. The curve is fixed once
   the token is deployed.

2. A factory contract: anyone can pay a creation fee to deploy a new token
   that they own. The factory records every token it created and who created
   it. The factory owner can change the fee and decide which addresses may
   withdraw the collected fees.

Validate inputs, use descriptive custom errors, and emit events for state
changes. Add a Foundry deploy script with a configurable fee and owner, and
tests covering the happy paths, access control, fee handling, the supply cap,
and curve quotes. Make sure forge test passes.
```

**What you should end up with**

- A token contract, a factory contract, a deploy script, and passing tests showing that:
  - creating a token collects the fee and makes the caller its owner
  - the factory remembers every token and its creator
  - underpaying the fee reverts with a clear error
  - only authorized addresses can withdraw fees
  - owner minting works up to the cap and reverts beyond it
  - cost quotes match the curve's formula for both curve shapes

**Checkpoint**

```shell
forge build
forge test
```

**Lessons learned**

- Set `offline = true` in `foundry.toml`. On the original macOS machine, Foundry crashed during `forge test` while trying to look up function selectors online.
- Install OpenZeppelin with `forge install OpenZeppelin/openzeppelin-contracts@v5.7.0`.

---

## Phase 2: Frontend and Local Chain Scripts

**Prompt**

```text
Read the Project Goals, Conventions, and Phase 2 Lessons learned in
docs/PLAYBOOK.md before starting.

Build a lightweight single-page frontend for the factory. A user should be
able to:

- point the app at a factory address and RPC URL (defaulting to local
  Anvil), and have the app remember both between visits
- connect a browser wallet such as MetaMask
- see the factory's key facts: owner, creation fee, ETH balance, and number
  of tokens created
- browse every token the factory created, with its key details, including
  its supply, cap, owner, and curve settings
- create a new token from a form, paying the creation fee
- see whether each transaction is pending, succeeded, or failed

Also add the deploy:local and fund:local npm scripts for working with Anvil:
deploy:local deploys the factory from Anvil's unlocked default account and
prints what the user needs next (address, owner, fee), and fund:local sends
test ETH to a given wallet. Make sure npm run build:web passes.
```

**What you should end up with**

- A frontend in `web/` and the four npm scripts from [Conventions](#conventions).
- From a fresh Anvil chain you can deploy the factory, open the app, connect MetaMask, create a token, and see it listed.

**Checkpoint**

```shell
npm run build:web
anvil --host 127.0.0.1 --port 8545 --chain-id 31337   # separate terminal
npm run deploy:local
npm run dev
```

Paste the printed factory address into the app and create a token. See [Running the Demo](#running-the-demo).

**Lessons learned**

- `forge script` crashed with a Foundry internal error while talking to Anvil in the original environment. Write `deploy:local` with viem instead: it reads the compiled factory from `out/` and, since Anvil's accounts are unlocked, needs no private key.
- The frontend needs the contracts' ABIs. If they are copied by hand, they go stale whenever a contract changes. Either keep them in one place and update them every phase, or generate them from `out/`.

---

## Phase 3: Documentation

**Prompt**

```text
Read the Project Goals in docs/PLAYBOOK.md.

Add docs/DESIGN.md, written for both humans and coding agents. Explain the
factory, the token, the bonding-curve math, how fees flow, and how the
frontend talks to the contracts. Be explicit about what the bonding curve
does not do yet. Update the README with usage, deploy, and frontend
instructions, and link docs/DESIGN.md and docs/PLAYBOOK.md. Do not rewrite
docs/PLAYBOOK.md.
```

**What you should end up with**

- `docs/DESIGN.md` and an updated README. A newcomer can understand the curve math and run the demo from these alone.

---

## Phase 4: Clear Errors

**Prompt**

```text
Read the Phase 4 Lessons learned in docs/PLAYBOOK.md before starting.

When a transaction fails in the UI, the message is often unhelpful. Make
failures explain themselves:

- Catch reverts before the wallet prompt where possible, so users aren't
  asked to sign a transaction that will fail.
- Validate obvious input mistakes in the browser (for example, a cap below
  the initial supply).
- Detect transactions that were mined but reverted.
- Show the contract's custom error by name instead of a raw hex selector.

Use one consistent pattern for every transaction the app sends, so later
features get good errors for free.

In DESIGN.md, add a "General Smart Contract Principles" section covering
checks-effects-interactions, pull vs. push payments, role-based access
control, the factory pattern, and transient storage.
```

**What you should end up with**

- Underpaying the fee shows something like `FeeTooLow` before MetaMask even opens.
- A principles section in `DESIGN.md`.

**Lessons learned**

- viem's `simulateContract` runs a transaction without sending it, which catches reverts before the wallet prompt. After sending, check the receipt's `status`.
- viem can only decode a custom error if that error is in the ABI the frontend uses. Without it, users see "reverted with the following signature: 0x…". In the original build this was only fixed in Phase 6.

---

## Phase 5: Minting From the Curve

**Prompt**

```text
Read the Project Goals and Phase 5 Lessons learned in docs/PLAYBOOK.md
before starting.

Make the bonding curve usable, not just quotable:

- Anyone can mint tokens by paying exactly the curve's price in ETH, within
  the supply cap.
- The token's owner can withdraw the ETH collected from those sales. Choose
  where that ETH is held and explain the choice in DESIGN.md.
- In the UI, a user can enter an amount on any token, see a live price quote,
  and mint. Show how much ETH each token has collected and the connected
  wallet's balance of it. The factory owner (or an authorized withdrawer)
  can withdraw creation fees from the UI.
- Add tests for correct payment, incorrect payment, and withdrawing the
  proceeds.

Update DESIGN.md and the README: tokens can now be bought from the curve,
but there is still no way to sell them back.
```

**What you should end up with**

- A second wallet can buy tokens from the curve, and the creator can withdraw the proceeds.
- Tokens are still fixed contracts: a curve can never change after deployment.

**Checkpoint**

```shell
forge test
npm run build:web
```

**Commit here.** This is the non-upgradeable baseline that Phase 6 is compared against.

**Lessons learned**

- A Foundry gotcha: `vm.prank` applies to the very next call, including one nested inside the arguments. In `vm.prank(buyer); token.mint{value: token.quote(3)}(3);`, the `quote` call uses up the prank, so the mint runs as the test contract. Compute the price on its own line first.

---

## Phase 6: Upgradeable Tokens With UUPS Proxies

**Prompt**

```text
Read the Project Goals and Phase 6 Lessons learned in docs/PLAYBOOK.md
before starting.

Illustrate upgradeable proxy contracts. Keep the fixed token as the default,
and let creators pay an extra fee to get an upgradeable token instead, using
OpenZeppelin's UUPS (Universal Upgradeable Proxy Standard) pattern.

Contracts:
- An upgradeable token behaves like a fixed one, plus its owner can change
  the curve settings and upgrade the token's code. Only the token's owner can
  do either; the factory owner must not be able to touch anyone's token.
- Curve changes must keep the token consistent (for example, the cap can
  never drop below the current supply).
- Follow standard proxy safety: nobody can initialize the implementation
  contract directly or re-initialize a proxy, and each proxy is deployed and
  initialized in a single transaction.
- All upgradeable tokens share one implementation contract. The factory
  owner can change the implementation used for future tokens, but not for
  existing ones.
- Tests: initialization, the extra fee, access control on curve changes and
  upgrades, blocked re-initialization, and an upgrade to a mock V2 that keeps
  balances, ETH, and settings. Make sure forge test passes and check contract
  sizes with forge build --sizes.

Frontend:
- When creating a token, the user chooses fixed or upgradeable and sees the
  total cost.
- Each token shows whether it is fixed or a proxy, and for proxies the code
  version and implementation address.
- The owner of an upgradeable token can edit its curve in the UI. For anyone
  else, and for fixed tokens, the option is visibly disabled with an
  explanation.
- Update the deploy scripts for the new fee.

Docs: in DESIGN.md, explain proxies, delegatecall, ERC-1967 storage slots,
UUPS, storage-layout rules, and the trust trade-off of upgradeable tokens.
Update the README. Do not rewrite docs/PLAYBOOK.md.
```

**What you should end up with**

- Fixed and upgradeable tokens appear side by side in the UI. Only a proxy token's owner can change its curve.
- A test that upgrades a token to a V2 and shows all state survived.

**Checkpoint**

```shell
forge test
npm run build:web
npm run deploy:local   # redeploy: the factory changed
```

Then go through the [Demo Flow](#demo-flow), steps 8 to 13.

**Lessons learned**

- Install the upgradeable package at the **same version** as the core one: `forge install OpenZeppelin/openzeppelin-contracts-upgradeable@v5.7.0`. It compiles against the core package through the `@openzeppelin/contracts/` remapping.
- In OpenZeppelin v5.7, `Initializable` and `UUPSUpgradeable` come from the core package (`@openzeppelin/contracts/proxy/utils/...`). The upgradeable package only re-exports them. Use the upgradeable package just for the upgradeable ERC20 and Ownable contracts.
- A contract behind a proxy can't keep per-token values in `immutable` variables, because those live in the implementation's code, which every proxy shares. Use storage instead.
- Deploying the shared implementation inside the factory's constructor keeps deployment to one step and stays well within contract size limits.

---

## Running the Demo

### Local Chain

```shell
anvil --host 127.0.0.1 --port 8545 --chain-id 31337
```

Anvil's first account is the default deployer:

```text
Address:     0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266
Private key: 0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80
```

This is a public development key. Never use it outside local testing. Restarting Anvil wipes the chain, so you must redeploy afterwards.

### Deploying the Factory

```shell
forge build            # deploy:local reads the compiled contract from out/
npm run deploy:local
```

The script prints the factory address and fees. The deployer becomes the factory owner. The factory address depends on how many transactions the deployer has already sent, so always copy it from the output. The README lists the environment variables for changing the RPC URL, deployer, and fees.

### MetaMask

Add a custom network:

```text
Network name:    Anvil Local
RPC URL:         http://127.0.0.1:8545
Chain ID:        31337
Currency symbol: ETH
Block explorer:  (leave blank)
```

MetaMask may suggest a different currency symbol for chain `31337`. Use `ETH`.

Import the Anvil key above, or send test ETH to an existing wallet with `npm run fund:local` (see the README for its options).

### Demo Flow

1. Start Anvil and run `npm run deploy:local`.
2. Run `npm run dev` and open the URL it prints.
3. Select the Anvil Local network in MetaMask and connect.
4. Paste the factory address into the app.
5. Create a **fixed** token.
6. From a second wallet, mint some of it from the curve, and watch the price quote.
7. Back on the creator wallet, withdraw the token's proceeds. As the factory owner, withdraw the creation fees.
8. Notice that the curve can't be edited on the fixed token.
9. Create an **upgradeable** token. The total cost includes the extra fee.
10. Compare the two tokens. The proxy shows its code version and implementation address, and every upgradeable token shares the same implementation.
11. Edit the proxy token's curve, for example its base price.
12. Check that the price quote changed while the token's address stayed the same.
13. Switch to a wallet that doesn't own the token. Editing is disabled, and calling the contract directly reverts with an ownership error.
14. Walk through the V2 upgrade test to see a full code upgrade that keeps all state.

## Verification Before Handing Off

```shell
forge fmt --check
forge test
npm run build:web
```

Expected:

- All Foundry tests pass.
- The web build passes. Vite may warn about a chunk over 500 kB because `viem` is bundled; that is expected.

## Suggested Next Phases

- A "deploy V2 and upgrade" flow in the UI, so a code upgrade can be demoed in the browser.
- Put upgrade rights behind a timelock or multisig to show how production systems limit owner power.
- A sell-back mechanism: keep a reserve of the ETH paid in, so holders can sell tokens back along the curve.
- EIP-1167 minimal clones as a cheaper, non-upgradeable alternative to compare against UUPS.
- `CREATE2` for deterministic token addresses.
- Generate the frontend ABIs from Foundry artifacts instead of copying them by hand.
- Event-based indexing instead of reading every token one by one.
- A helper in the frontend for switching wallet networks.
