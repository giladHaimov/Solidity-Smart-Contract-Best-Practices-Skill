---
name: solidity-best-practices
description: >-
  Guides writing, reviewing, and shipping high-quality Solidity smart contracts
  using established engineering standards. Covers invariant-driven design,
  composable architecture, coding style, visibility, events, NatSpec, gas
  discipline, dependency hygiene, upgradeability decisions, testing pipelines,
  staged deployment, and AI-assisted development. Use when writing or reviewing
  .sol files, designing contract architecture, setting up Foundry/Hardhat
  projects, or when the user asks for Solidity best practices, style, or
  production-quality contract development.
---

# Solidity Best Practices

Engineering guide for production-quality Solidity. Focus: **how to build well**, not vulnerability hunting. For security audits, use a dedicated security skill.

**Synthesized from:** [Solidity docs](https://docs.soliditylang.org/) · [OpenZeppelin GUIDELINES](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/GUIDELINES.md) · [SCSFG](https://scsfg.io/developers/) · [ethereum.org](https://ethereum.org/developers/docs/smart-contracts/security/) · [ConsenSys dev recommendations](https://consensysdiligence.github.io/smart-contract-best-practices/development-recommendations/) · [Trail of Bits invariant-driven development](https://blog.trailofbits.com/2025/02/12/the-call-for-invariant-driven-development/) · [Foundry Book](https://book.getfoundry.sh/)

## When to Apply

- Writing or refactoring `.sol` files
- Designing contract architecture before implementation
- Reviewing PRs for quality and maintainability
- Setting up testing, linting, or CI
- Generating Solidity with AI assistance

---

## Core Philosophy

1. **Simplicity over cleverness** — small, single-responsibility contracts; prefer audited libraries (OpenZeppelin, Solmate, PRBMath) over custom primitives ([SCSFG defensive programming](https://scsfg.io/developers/defensive-programming/)).
2. **Invariants drive everything** — identify properties that must always hold *before* writing code; embed them in design, tests, docs, and monitoring ([Trail of Bits](https://blog.trailofbits.com/2025/02/12/the-call-for-invariant-driven-development/)).
3. **Explicit over implicit** — visibility, access control, rounding direction, and trust assumptions visible in code and NatSpec.
4. **Least privilege at every layer** — contract, function, role, and data access minimized ([SCSFG PoLP](https://scsfg.io/developers/defensive-programming/)).
5. **Composable, localized state** — modular design with encapsulated state; avoid fancy data-separation patterns (Diamond, Eternal Storage) unless indispensable ([SCSFG system design](https://scsfg.io/developers/system-design/)).
6. **Observable state changes** — private state, controlled setters, indexed events after every mutation.
7. **Test what you specify** — unit → integration → E2E → fuzz → invariant; structure tests like production code ([SCSFG testing](https://scsfg.io/developers/testing/)).

Per [Solidity Security Considerations](https://docs.soliditylang.org/en/latest/security-considerations.html): keep contracts small and modular, use CEI, include fail-safe modes, and always seek peer review.

---

## Phase 0 — Invariant-Driven Design

Start here. Invariants are the contract between design, implementation, testing, and operations.

### Write invariants in Hoare-triple form

For each critical operation document: **precondition → command → postcondition**.

```
INVARIANT: totalAssets >= totalLiabilities
  pre:  assets > 0, receiver != address(0)
  cmd:  deposit(assets, receiver)
  post: shares minted ≤ assets deposited; totalAssets increased by assets
```

### Invariant categories (prioritize top-down)

| Category | Example |
|----------|---------|
| Solvency | `totalAssets >= sum(userBalances)` |
| Conservation | `totalSupply == sum(balances)` |
| Access control | only `MINTER_ROLE` can increase `totalSupply` |
| Monotonicity | `totalStaked` only increases between epochs |
| Bounds | `feeBps ≤ MAX_FEE_BPS` |
| State consistency | `locked + available == total` |

### Embed invariants across the lifecycle

| Stage | Action |
|-------|--------|
| Design | List invariants in spec; use to guide architecture |
| Implementation | Assert invariants after state changes (FREI-PI) |
| Testing | Foundry `invariant_*` + handler; Echidna for extended campaigns |
| Review | Reviewer verifies each invariant has test coverage |
| Deployment | Smoke-test invariants on fork before mainnet |
| Operations | Monitor invariant violations via events/Tenderly/Forta |

---

## Phase 1 — Architecture & Design

Complete before implementation ([SCSFG system design](https://scsfg.io/developers/system-design/)):

| Step | Practice |
|------|----------|
| Spec | State transitions, invariants, events, trust assumptions |
| Composability | Break into modules by separation of concerns; each module autonomous with minimal deps |
| On-chain scope | Keep on-chain only what must be trustless; document off-chain attack surface |
| Dependencies | Registry of every external contract/token/oracle; **pin exact versions** ([SCSFG dependencies](https://scsfg.io/developers/dependencies/)) |
| Privilege map | Every privileged function listed; OZ `AccessControl` over bare `onlyOwner` |
| Admin ops | Multisig + timelock + purposeful delays on critical changes |
| Upgrade decision | Default **immutable**; upgrade only when maintenance need outweighs trust cost ([SCSFG upgradeability](https://scsfg.io/developers/upgradeability/)) |
| Emergency | Pause, rate limits, speed bumps for irreversible actions |
| Diagrams | Architecture + sequence diagrams; auto-generate inheritance (Slither/Surya) + manual overlays |

**Upgradeability decision tree:**
```
Need post-deploy logic fixes?
├─ No  → immutable; plan migration path if ever needed
└─ Yes → Can you migrate users to new contract instead?
          ├─ Yes → prefer migration over proxy
          └─ No  → OZ UUPS/Transparent/Beacon; never hand-rolled proxy
```

Avoid Diamond (EIP-2535) unless no alternative — OZ deliberately excludes it. Prefer localized, well-encapsulated state over data-separation patterns.

---

## Phase 2 — Coding Standards

### File header

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.28;  // exact pin — never a floating range
```

Pin the latest **patched** compiler for your minor version. CI must compile with the same pin.

### Visibility & naming ([Solidity Style Guide](https://docs.soliditylang.org/en/latest/style-guide.html), [OZ GUIDELINES](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/GUIDELINES.md))

| Rule | Practice |
|------|----------|
| Lowest visibility | `private` > `internal` > `external` > `public` |
| State variables | `private`/`internal` only; explicit getters when needed |
| Prefix | `_` on private/internal (`_balances`, `_calculateFee`) |
| `external` vs `public` | `external` when not called internally (cheaper calldata) |
| Access control | Every mutating function: modifier **or** comment why intentionally open |
| Concrete types | Store `IERC20` not `address`; cast only when truly needed ([SCSFG](https://scsfg.io/developers/defensive-programming/)) |

### Contract layout

```
Pragma → Imports → Errors → Interfaces → Libraries → Contracts
  → Types → State → Events → Modifiers → Constructor
  → Receive/Fallback → External → Public → Internal → Private
```

- One function, one responsibility; split at ~40 lines
- Interface per ERC behavior, own file, `I` prefix
- Named imports only; no wildcards ([OZ style](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/.claude/skills/solidity-style/SKILL.md))

### Defensive programming ([SCSFG](https://scsfg.io/developers/defensive-programming/))

- Validate at the **earliest point** — including `internal` functions, not just public entry points ("hard shell, weak core" anti-pattern)
- Check state before acting (`whenNotPaused`, `whenUnlocked`)
- Proactive checks in libraries and internal helpers

### Types, storage & math

- `mapping` over `array` for key lookups; bound any required arrays
- No `uint8`/`uint128` outside structs without packing justification
- Pack struct fields into 32-byte slots
- `immutable` / `constant` instead of storage for fixed values
- Multiply before divide; document rounding direction in `@dev`
- `unchecked` blocks need proof comments

### Gas discipline

- Custom errors over string reverts ([EIP-6093](https://eips.ethereum.org/EIPS/eip-6093))
- `calldata` for read-only dynamic external params
- Cache storage reads used more than once
- `ALL_CAPS` named constants — no magic numbers
- Cheapest revert condition first (short-circuit)
- Assembly: always `assembly ("memory-safe")` with inline comments

### ETH & payments ([Solidity Common Patterns](https://docs.soliditylang.org/en/latest/common-patterns.html))

- No `receive()`/`fallback()` unless ETH is core logic
- Track internal balances — never `address(this).balance` as accounting
- **Pull-over-push**: `mapping(address => uint256) pending` + `claim()`
- `call{value: x}("")` with return check

### Time & delays

- Second-level `block.timestamp` tolerance OK; exact-second logic is not
- **Purposeful delays** (speed bumps) on admin actions — give users time to react ([SCSFG](https://scsfg.io/developers/defensive-programming/))

### Events & errors

- Emit **after** state change; past-tense names (`Deposited`, `RoleGranted`)
- ERC-mandated tense follows the standard (`Transfer` = present)
- Index key identifiers (max 3 per event)
- Errors: domain-prefixed, placed in interface/library when context fits

### NatSpec ([NatSpec format](https://docs.soliditylang.org/en/latest/natspec-format.html), [SCSFG documentation](https://scsfg.io/developers/documentation/))

Annotate all public/external interfaces. Generate docs in CI with `solidity-docgen`.

```solidity
/// @notice Deposits tokens and mints shares
/// @param assets Amount of underlying tokens to deposit
/// @param receiver Address receiving minted shares
/// @return shares Amount of shares minted
/// @dev Rounds down in protocol's favor. Reverts on zero amount.
function deposit(uint256 assets, address receiver) external returns (uint256 shares);
```

Document: trust assumptions, rounding, preconditions, side effects.

### Upgradeables ([SCSFG upgradeability](https://scsfg.io/developers/upgradeability/), OZ Upgrades plugin)

- OZ UUPS/Transparent/Beacon only
- Append-only storage; `__gap` in base contracts
- `_disableInitializers()` in implementation constructor
- Initialize in **same tx** as deployment (frontrun protection)
- Storage-layout diff before every upgrade
- Dry-run upgrades on fork

### Multi-chain

CREATE2 for deterministic addresses; document and verify salts.

---

## Phase 3 — Structural Patterns

From [Solidity Common Patterns](https://docs.soliditylang.org/en/latest/common-patterns.html):

| Pattern | Use |
|---------|-----|
| **CEI** | Checks → Effects (state + events) → Interactions (external calls) |
| **Withdrawal** | Users claim owed funds; contract never pushes in loops |
| **State machine** | Modifiers (`atStage`, `whenNotPaused`) guard lifecycle transitions |
| **Fail-safe** | Pause/circuit-breaker for emergency stop ([Solidity recommendations](https://docs.soliditylang.org/en/latest/security-considerations.html)) |
| **Access restriction** | Role-based modifiers; two-step ownership transfer |
| **SafeERC20** | All token transfers; balance-delta for fee-on-transfer tokens |

---

## Phase 4 — Dependencies ([SCSFG](https://scsfg.io/developers/dependencies/))

| Rule | Practice |
|------|----------|
| Known-good libs | OpenZeppelin, Solmate, PRBMath, Gnosis Safe — prefer audited, widely used |
| Pin versions | Exact versions in `package.json`/`foundry.toml`; never floating |
| Review once | Read each dependency at least once; don't trust blindly |
| Minimize surface | Remove unused deps; don't import a whole lib for one function |
| Stay current | Subscribe to security advisories; update deliberately with re-review |
| Supply chain | Lock files committed; CI checks dependency integrity |

---

## Phase 5 — Testing & Tooling ([SCSFG testing](https://scsfg.io/developers/testing/))

### Test pyramid

| Layer | Tool | Purpose |
|-------|------|---------|
| Unit | Foundry/Hardhat | Every function, valid + invalid paths |
| Integration | Foundry fork | Cross-contract + third-party interactions |
| E2E / system | Foundry | Full user journeys, success + failure paths |
| Fuzz | `testFuzz_*` | Input-range edge cases with `bound()` |
| Invariant | `invariant_*` + handler | Protocol properties across call sequences |
| Static | Slither + Solhint | CI gates on every PR |

### Test structure

```
test/
├── unit/           # one file per contract/module
├── integration/    # one file per component relationship
├── e2e/            # user journey scenarios
└── utils/          # shared fixtures, helpers
```

Tests are as readable as production code. Externalize shared setup into fixtures. No duplication between test files.

### TDD workflow (recommended for core logic)

1. Write failing test from spec/invariant
2. Implement minimal code to pass
3. Refactor; re-run full suite
4. Repeat

### Invariant testing (Foundry)

```toml
[invariant]
runs = 256
depth = 100
fail_on_revert = false  # true once handler is tight
```

- **Handler contract** bounds inputs; target handler not bare protocol
- **Ghost variables** track cumulative deposits/withdrawals
- **`bound()`** over `vm.assume()`
- Multiple actors, not just `address(this)`
- Start solvency: `assets >= liabilities`
- Increase `depth` before `runs` for multi-step properties
- Save failing sequences as permanent regression tests

### Mainnet forking

Fork at pinned block; test against real token/oracle addresses. Reset state between tests. Manipulate state with Foundry cheatcodes for edge scenarios.

### CI gates

`compile → lint → slither → test → coverage threshold`. No merge on warnings. Coverage near 100% on core logic ([OZ GUIDELINES](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/GUIDELINES.md)).

---

## Phase 6 — Code Review Protocol

Every significant change: author + ≥1 reviewer. Treat review like a lightweight audit ([ethereum.org](https://ethereum.org/developers/docs/smart-contracts/security/)).

1. **Context** — architecture and change intent
2. **Invariants** — each spec invariant has test coverage?
3. **Diff** — module dependencies, what changed
4. **Walk-through** — assumptions for every value read/written
5. **Visibility** — hierarchy, `_` prefix, external-vs-public
6. **CEI** — verified function by function
7. **Defensive depth** — validation in internal functions too
8. **Events** — state change → event
9. **Docs** — NatSpec matches implementation
10. **Static analysis** — Slither triaged

Optional tooling during review:
- Slither printers for inheritance/function/authorization diagrams
- `slither-check-upgradeability` before proxy changes
- `slither-check-erc` for standard conformance

Record in PR audit log (template in [reference.md](reference.md)).

---

## Phase 7 — Staged Deployment ([SCSFG deployment](https://scsfg.io/developers/deployment/))

```
Private testnet (fork) → Public testnet → Mainnet beta (capped) → Mainnet release
```

| Stage | Practice |
|-------|----------|
| Local/fork | Deploy in test suite; simulate failures with cheatcodes |
| Public testnet | Community QA; test monitoring; open feedback channel |
| Mainnet beta | Cap deposits; bug bounty live; incident plan ready; set sunset block |
| Mainnet release | Remove caps gradually; monitoring watch window |

| Always | Detail |
|--------|--------|
| Compiler | Exact pin; verify on Etherscan/Sourcify |
| Scripts | Fork-tested; constructor args triple-checked |
| Keys | Multisig (≥2-of-3) + hardware wallets + timelock |
| Init | Proxy: initialize in same tx as deploy |
| Monitoring | Admin events, large movements, invariant deviations |
| Upgrades | Storage diff + fork dry-run + watch window |

---

## Phase 8 — AI-Assisted Development

AI output is **untrusted** until compiled, tested, and reviewed ([SolidityBench patterns](https://arxiv.org/abs/2606.19988)).

1. Never zero-shot to production
2. Prompt: interface → control flow (checks → state → events) → invariants → code
3. RAG against OpenZeppelin; cap at ~2 examples
4. Compile immediately; verify imports are real
5. Review first: access modifiers, state relationships, CEI
6. Audit retrieved snippets
7. Same review + test gauntlet as human code

---

## Pre-Submit Checklist

```
Design
- [ ] Invariants documented (Hoare-triple form)
- [ ] Architecture diagram + privilege map
- [ ] Upgrade/emergency story decided
- [ ] Dependencies pinned and reviewed

Code
- [ ] SPDX + exact pragma
- [ ] Lowest visibility; _ prefix; concrete types
- [ ] Custom errors; calldata; CEI; events after state
- [ ] Validation in internal functions too
- [ ] Full NatSpec on external/public

Tests
- [ ] Unit (valid + invalid paths)
- [ ] Integration (fork with real deps)
- [ ] Fuzz on math/inputs
- [ ] Invariant tests for every spec invariant
- [ ] Slither + Solhint pass in CI

Review
- [ ] ≥1 reviewer walk-through
- [ ] Audit log on PR
```

---

## Additional Resources

- Extended conventions, invariant templates, resource library: [reference.md](reference.md)
- Good vs. poor patterns with code: [examples.md](examples.md)
