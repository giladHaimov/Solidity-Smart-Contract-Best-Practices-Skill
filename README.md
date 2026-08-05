# Solidity Best Practices (Cursor Skill)

Handbook + Cursor agent skill for writing Solidity that you'd actually want to maintain six months from now.

This is **not** an audit checklist. If you need reentrancy trees and oracle manipulation playbooks, use your security skill. This repo is about the boring stuff that saves you later: naming, layout, invariants, tests, reviews, deploys.

Built from stuff I keep re-reading anyway — Solidity docs, OpenZeppelin's `GUIDELINES.md`, [scsfg.io](https://scsfg.io/developers/), Trail of Bits on invariants, Foundry's testing guides. Opinions are mine where sources disagree.

---

## What's in the repo

```
Solidity-Smart-Contract-Best-Practices-Skill/   ← GitHub repo (this folder)
├── README.md
├── SKILL.md          ← main skill file
├── reference.md      ← checklists, links, templates
└── examples.md       ← good vs. sloppy patterns
```

- **SKILL.md** — what the agent loads. Phases 0–8, workflow, pre-submit checklist.
- **reference.md** — naming tables, invariant templates, deployment checklists, link dump.
- **examples.md** — side-by-side code for reviews.

### Naming (three different things — all intentional)

| Layer | Name | Purpose |
|-------|------|---------|
| **GitHub repo** | `Solidity-Smart-Contract-Best-Practices-Skill` | Human-readable project name on disk / GitHub |
| **Skill id** | `solidity-best-practices` | YAML `name:` in `SKILL.md` — how Cursor identifies the skill (lowercase, hyphens, max 64 chars) |
| **Install folder** | `.cursor/skills/solidity-best-practices/` | Where files live when installed in a project or `~/.cursor/skills/` |

The repo name does **not** need to match the skill id. Compare OpenZeppelin: repo `openzeppelin-contracts`, skill `solidity-style`. Trail of Bits: plugin `building-secure-contracts`, skill `secure-workflow-guide`. Short skill ids discover better.

Do **not** rename the skill to `solidity-smart-contract-best-practices-skill` — longer, no benefit.

---

## Install it

**In a single project** (good for teams):

```bash
mkdir -p .cursor/skills/solidity-best-practices
cp SKILL.md reference.md examples.md .cursor/skills/solidity-best-practices/
```

**Globally:**

```bash
mkdir -p ~/.cursor/skills/solidity-best-practices
cp SKILL.md reference.md examples.md ~/.cursor/skills/solidity-best-practices/
```

The file inside must be named `SKILL.md` (already is in this repo). Supporting files: `reference.md`, `examples.md` — same folder, one level deep (Cursor convention).

**No Cursor?** Just read the markdown. Works fine as a team doc.

Trigger it explicitly if the agent drifts: *"follow the solidity best practices skill on this contract."*

---

## How I think about this

A few rules I won't shut up about:

**Start with invariants, not interfaces.** Before you sketch `deposit()` and `withdraw()`, write down what must stay true no matter what order people call things. `totalAssets >= what we owe users` is a good first one. Trail of Bits calls this invariant-driven development; I call it "stopping yourself from painting into a corner." Put them in the spec, test them with Foundry handlers, mention them in review.

**Default to immutable.** Upgradeable proxies are a product decision, not a convenience. If you can migrate users to a new contract, do that. If you need a proxy, use OpenZeppelin's — don't hand-roll delegatecall because you read a blog post once.

**Private state, explicit getters.** OpenZeppelin's been yelling about this for years and people still `public` their mappings. Don't. Events on every real state change. Past tense. Index the stuff you'll filter on.

**Validate inside `internal` functions too.** Putting all your checks on the external wrapper and leaving `_mint` wide open is the "hard shell, weak core" mistake. One forgotten code path and you're explaining it on Discord.

**CEI isn't optional.** Checks → update storage + emit events → then token transfers / ETH sends. You learned this in 2017. Still skip it sometimes. Don't.

**Pin your compiler.** `pragma solidity 0.8.28;` — full stop. Not `^0.8.0`. Your CI and your laptop compile the same bytecode or you're not really testing what you ship.

**AI output is a draft.** Compile it, read access control first, run Slither, don't `@`-mention me when it reentrancy-guards the wrong function.

---

## The workflow (short version)

The skill file has the full version. Roughly:

1. **Design** — spec, invariants, privilege map, upgrade story (probably "we're immutable"), dependency list with pinned versions.
2. **Code** — style guide + OZ conventions: visibility, errors, NatSpec, layout, gas habits that don't make the code unreadable.
3. **Patterns** — CEI, pull payments, state machines, SafeERC20. The [Solidity common patterns](https://docs.soliditylang.org/en/latest/common-patterns.html) page is still good.
4. **Dependencies** — fewer is better. Read what you import once. Lock versions.
5. **Tests** — unit for logic, fork for real USDC weirdness, fuzz for math, invariants for "does this property hold after 10,000 random call sequences."
6. **Review** — someone else reads it aloud. Check invariants have tests. Slither on the diff.
7. **Deploy** — fork → testnet → capped mainnet beta → full launch. Initialize proxies in the same tx. Verify on Etherscan.
8. **AI** — same bar as human code. No exceptions.

---

## A vault sketch that follows the rules

Not production-ready — trimmed for teaching. Shows the habits, not every ERC-4626 edge case.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.28;

import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {SafeERC20} from "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import {AccessControl} from "@openzeppelin/contracts/access/AccessControl.sol";
import {Pausable} from "@openzeppelin/contracts/utils/Pausable.sol";
import {Math} from "@openzeppelin/contracts/utils/math/Math.sol";

/// @title SimpleVault — illustrative, not audited
contract SimpleVault is AccessControl, Pausable {
    using SafeERC20 for IERC20;

    bytes32 public constant PAUSER_ROLE = keccak256("PAUSER_ROLE");

    IERC20 private immutable _asset;
    uint256 private _totalAssets;
    uint256 private _totalShares;
    mapping(address => uint256) private _shares;

    error ZeroAmount();
    error ZeroAddress();

    event Deposited(address indexed caller, address indexed receiver, uint256 assets, uint256 shares);
    event Withdrawn(address indexed owner, address indexed receiver, uint256 assets, uint256 shares);

    constructor(IERC20 asset_, address admin) {
        if (address(asset_) == address(0)) revert ZeroAddress();
        _asset = asset_;
        _grantRole(DEFAULT_ADMIN_ROLE, admin);
        _grantRole(PAUSER_ROLE, admin);
    }

    function totalAssets() external view returns (uint256) {
        return _totalAssets;
    }

    function sharesOf(address account) external view returns (uint256) {
        return _shares[account];
    }

    /// @notice Deposit `assets`, mint shares to `receiver`. Rounds down — favors the vault.
    function deposit(uint256 assets, address receiver) external whenNotPaused returns (uint256 shares) {
        return _deposit(assets, receiver);
    }

    function redeem(uint256 shares, address receiver, address owner) external whenNotPaused returns (uint256 assets) {
        if (shares == 0) revert ZeroAmount();
        if (receiver == address(0) || owner == address(0)) revert ZeroAddress();
        if (_shares[owner] < shares) revert ZeroAmount();

        assets = _convertToAssets(shares);

        _shares[owner] -= shares;
        _totalShares -= shares;
        _totalAssets -= assets;
        emit Withdrawn(owner, receiver, assets, shares);

        _asset.safeTransfer(receiver, assets);
    }

    function _deposit(uint256 assets, address receiver) internal returns (uint256 shares) {
        if (assets == 0) revert ZeroAmount();
        if (receiver == address(0)) revert ZeroAddress();

        shares = _convertToShares(assets);
        if (shares == 0) revert ZeroAmount();

        _totalAssets += assets;
        _totalShares += shares;
        _shares[receiver] += shares;
        emit Deposited(msg.sender, receiver, assets, shares);

        _asset.safeTransferFrom(msg.sender, address(this), assets);
    }

    function _convertToShares(uint256 assets) internal view returns (uint256) {
        uint256 supply = _totalShares;
        return supply == 0 ? assets : assets.mulDiv(supply, _totalAssets, Math.Rounding.Floor);
    }

    function _convertToAssets(uint256 shares) internal view returns (uint256) {
        uint256 supply = _totalShares;
        return supply == 0 ? shares : shares.mulDiv(_totalAssets, supply, Math.Rounding.Floor);
    }
}
```

Notice: `_asset` is `IERC20` not `address`. Checks live in `_deposit` too. State updates and events before `safeTransferFrom`. Custom errors. `@dev` would mention rounding if this were real.

More patterns (pull payments, proxy init, handlers): **examples.md**.

---

## FREI-PI — one function at a time

When I write tests or review PRs I ask five questions per mutating function:

- **F** — what does it do?
- **R** — what has to be true before it runs?
- **E** — what storage changes, what events fire?
- **I** — what external calls happen (and are they *after* effects)?
- **P** — which protocol invariant still holds when we're done?

For `deposit`: requirement is `assets > 0`, effect is `_totalAssets += assets`, interaction is `transferFrom`, invariant is solvency. Write a unit test that asserts P at the end. If you can't name P, you don't understand the function yet.

Template for a full invariant catalog: **reference.md**.

---

## Testing — what actually matters

Directory layout I like:

```
test/
├── unit/
├── integration/
├── invariant/    # handler + invariant_*.t.sol
└── utils/
```

**Unit** — happy path + every revert you care about. Boring. Necessary.

**Fork** — point at mainnet USDC/WETH at a pinned block. Catches the "oh right, that token doesn't return a bool" stuff mocks hide.

**Fuzz** — `bound()` your inputs. Don't `vm.assume()` yourself into zero coverage.

**Invariants** — the fun part. Handler contract wraps your protocol, bounds inputs so the fuzzer isn't 90% reverting, ghost vars track deposits/withdrawals off-chain style. Target the handler, not the vault directly.

```toml
# foundry.toml — starting point, tune for CI time budget
[invariant]
runs = 256
depth = 100
fail_on_revert = false   # flip true once handler is tight
```

First invariant I'd write: `totalAssets >= 0` and whatever solvency property your spec claims. Turn a failing sequence into a regression test and keep it forever.

CI: `forge build`, Solhint, Slither, tests. Don't merge on Slither medium+ without a comment explaining why you're ignoring it.

Full handler example: **examples.md**.

---

## Review — before you ask for audit

Someone who didn't write the code reads it. Non-negotiable for anything touching money.

I usually walk through:

1. What changed and why (author talks, reviewer listens)
2. Do the spec invariants have tests?
3. Visibility pass — anything `public` that shouldn't be?
4. CEI order, function by function
5. Events — did we forget any state change?
6. NatSpec — does it match behavior, especially rounding and trust assumptions?
7. Slither diff — triage everything, even informational

Drop a short audit log on the PR. Template in **reference.md**. Doesn't need to be fancy — purpose, files, findings, done.

---

## Deploy — don't yeet to mainnet

Boring pipeline that works:

**Fork in CI** → **public testnet** (real monitoring, real feedback channel) → **mainnet with caps** → **remove caps slowly**.

Proxy gotchas worth repeating: `_disableInitializers()` in the impl constructor, `initialize()` in the **same transaction** as deploy. I've seen someone lose an hour to a frontrun initialize so you don't have to.

Beta contracts can use a `sunsetBlock` — no new deposits after date X, withdrawals always open. Forces you to ship the real version.

---

## AI-generated Solidity

Use it. Don't trust it.

Workflow that hasn't burned me yet:

1. Describe interfaces + invariants first, code second
2. Point it at OpenZeppelin, not random GitHub gists
3. Compile immediately — hallucinated import paths are real
4. Read modifiers and storage layout before anything else
5. Same tests and review as if a junior dev wrote it

Ten-item sanity checklist for AI output lives in **examples.md**.

---

## Before you open the PR

```
[ ] Invariants written down (even bullet points beat nothing)
[ ] Exact pragma, SPDX, private state, custom errors
[ ] CEI + events on state changes
[ ] Internal functions validate too
[ ] Tests: unit + at least one fuzz or invariant if there's math
[ ] Slither clean-ish on the diff
[ ] Someone else looked at it
```

---

## Stuff I bookmark

**Solidity**

- Style guide: https://docs.soliditylang.org/en/latest/style-guide.html
- Common patterns (withdrawal, access, state machine): https://docs.soliditylang.org/en/latest/common-patterns.html
- Security considerations (modularity, CEI, peer review): https://docs.soliditylang.org/en/latest/security-considerations.html

**SCSFG** — best single site for "how to run a serious project": https://scsfg.io/developers/

Standouts: [system design](https://scsfg.io/developers/system-design/), [defensive programming](https://scsfg.io/developers/defensive-programming/), [testing](https://scsfg.io/developers/testing/), [deployment](https://scsfg.io/developers/deployment/), [upgradeability](https://scsfg.io/developers/upgradeability/)

**OpenZeppelin**

- Engineering rules: https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/GUIDELINES.md
- Docs: https://docs.openzeppelin.com/contracts/
- Upgrades plugin: https://docs.openzeppelin.com/upgrades-plugins/

**Trail of Bits**

- Invariant-driven dev: https://blog.trailofbits.com/2025/02/12/the-call-for-invariant-driven-development/
- Building Secure Contracts: https://github.com/crytic/building-secure-contracts

**Foundry**

- Book: https://book.getfoundry.sh/
- Invariant guide: https://book.getfoundry.sh/guides/invariant-testing

**Other**

- ethereum.org lifecycle stuff: https://ethereum.org/developers/docs/smart-contracts/security/
- Guy Lando's link list: https://github.com/guylando/KnowledgeLists/blob/master/EthereumSmartContracts.md
- Solidity patterns repo: https://github.com/fravoll/solidity-patterns

Full table with tools defaults: **reference.md**.

---

## Where to go next

| If you… | Read… |
|---------|--------|
| Want the agent to follow this | `SKILL.md` |
| Need a checklist or link | `reference.md` |
| Want "show me good code" | `examples.md` |

Questions, gaps, wrong opinions — open an issue or fix the doc. This is a living skill, not a whitepaper.
