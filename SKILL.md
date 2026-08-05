---
name: solidity-best-practices
description: >-
  Guides writing, reviewing, and shipping high-quality Solidity smart contracts
  using established engineering standards. Covers invariant-driven design,
  composable architecture, and a 70+ item detection checklist — every item
  names the exact pattern to find in the code, so it can be applied
  mechanically during code generation or a pre-push review, not just read as
  prose. Use when writing or reviewing .sol files, designing contract
  architecture, setting up Foundry/Hardhat projects, or when the user asks for
  Solidity best practices, style, or production-quality contract development.
---

# Solidity Best Practices

Engineering guide for production-quality Solidity. Focus: **how to build well**, not vulnerability hunting. For security audits, use a dedicated security skill.

**Synthesized from:** [Solidity docs](https://docs.soliditylang.org/) · [OpenZeppelin GUIDELINES](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/GUIDELINES.md) · [SCSFG](https://scsfg.io/developers/) · [ethereum.org](https://ethereum.org/developers/docs/smart-contracts/security/) · [ConsenSys dev recommendations](https://consensysdiligence.github.io/smart-contract-best-practices/development-recommendations/) · [Trail of Bits](https://github.com/crytic/building-secure-contracts) · [Secureum](https://secureum.substack.com/) · [Solcurity](https://github.com/transmissions11/solcurity) · [SWC Registry](https://swcregistry.io/) · [Foundry Book](https://book.getfoundry.sh/)

## When to Apply

- Writing or refactoring `.sol` files
- Designing contract architecture before implementation
- Reviewing PRs for quality and maintainability
- Setting up testing, linting, or CI
- Generating Solidity with AI assistance

---

## Core Philosophy

1. **Simplicity over cleverness** — small, single-responsibility contracts; prefer audited libraries (OpenZeppelin, Solmate, PRBMath) over custom primitives.
2. **Invariants drive everything** — identify properties that must always hold *before* writing code; embed them in design, tests, docs, and monitoring.
3. **Explicit over implicit** — visibility, access control, rounding direction, and trust assumptions visible in code and NatSpec.
4. **Least privilege at every layer** — contract, function, role, and data access minimized.
5. **Composable, localized state** — modular design with encapsulated state; avoid fancy data-separation patterns (Diamond, Eternal Storage) unless indispensable.
6. **Observable state changes** — private state, controlled setters, indexed events after every mutation.
7. **Test what you specify** — unit → integration → fuzz → invariant; structure tests like production code.

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

Complete before implementation:

| Step | Practice |
|------|----------|
| Spec | State transitions, invariants, events, trust assumptions |
| Composability | Break into modules by separation of concerns; each module autonomous with minimal deps |
| On-chain scope | Keep on-chain only what must be trustless; document off-chain attack surface |
| Dependencies | Registry of every external contract/token/oracle; **pin exact versions** |
| Privilege map | Every privileged function listed; OZ `AccessControl` over bare `onlyOwner` |
| Admin ops | Multisig + timelock + purposeful delays on critical changes |
| Upgrade decision | Default **immutable**; upgrade only when maintenance need outweighs trust cost |
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

## The Checklist — Code Review & Generation

**How to use this:** every item names the exact pattern to look for in the `.sol` code being written or reviewed — no external context needed. Items marked **Trigger / Flag** don't check whether a test already exists (that would require reading test files); instead they tell you when the code you're looking at warrants recommending a specific test, and give you the message to surface. Run this list top to bottom on every diff.

### Visibility & Encapsulation

1. Prefix `private`/`internal` members with `_`. **Detect:** a `private`/`internal` variable or function whose name doesn't start with `_`.
2. Default to minimal visibility (`private` > `internal` > `external` > `public`). **Detect:** a `public` function with no internal call site — should be `external`.
3. State variables: always `private`/`internal`; add a `view` getter only if needed. **Detect:** any state variable declared `public` or with no visibility keyword.
4. Store dependencies as their interface type (`IERC20`), not `address`. **Detect:** a state variable typed `address` that gets wrapped in an interface cast at each use site.
5. Named imports only. **Detect:** `import "X.sol";` without `{...} from`.
6. Interfaces get their own file, prefixed `I`. **Detect:** an `interface` declaration not named `I...`.
7. Mark functions `virtual` by default in library/base contracts, except trivial alias functions. **Detect:** a base/abstract contract function lacking `virtual` that isn't a one-line delegate.
8. Mark `override` explicitly on every overriding function. **Detect:** a function signature matching a parent's without the `override` keyword.

### State, Storage & Data Structures

9. Use `mapping` for key-based lookup; `array` only for iteration. **Detect:** an array searched via a linear loop comparing to a key.
10. Bound or cap any collection a user can grow. **Detect:** a `push()` into a storage array reachable from an external function, with no cap, later iterated in a loop.
11. Pack struct fields to fit 32-byte slots. **Detect:** adjacent struct fields whose combined width doesn't fill a 256-bit slot when it could.
12. Anything fixed after the constructor: `immutable`/`constant`. In upgradeable contracts `immutable` is still safe — it lives in bytecode, not storage — but only for a value identical across every proxy sharing that implementation; mark it with `@custom:oz-upgrades-unsafe-allow state-variable-immutable`. **Detect:** a state variable only ever assigned in the constructor, not declared `immutable`.
13. Nothing on-chain is private. **Detect:** a variable/comment naming "secret," "password," "apiKey," or "privateKey" stored in contract state.
14. `delete` doesn't clear nested mappings. **Detect:** `delete` applied to an array/struct element whose type contains a `mapping` field.
15. Downcasting integers truncates silently — use `SafeCast`. **Detect:** an explicit cast to a smaller int/uint type not wrapped in `SafeCast`.
16. Never compare balance with `==` for accounting. **Detect:** `== address(this).balance` or `balance == amount` in a conditional.
17. Never hardcode chain-specific addresses (tokens, oracles) in contract code. **Detect:** a literal `address(0x...)` for a known external contract, instead of a constructor/config parameter.

### Access Control & Privilege

18. Every state-mutating function needs an access modifier or an explicit "intentionally open" comment. **Detect:** a function that writes storage, has no modifier, and no such comment.
19. If a user-triggered function mutates data outside that user's own record, require an explicit access check. **Detect:** a function keyed on `msg.sender` that also writes a variable not indexed by `msg.sender`, with no role/owner check.
20. `AccessControl` for multi-role systems; `Ownable2Step` is enough for single-admin contracts. **Detect:** 3+ distinct privilege levels implemented via separate boolean mappings instead of `AccessControl`.
21. Two-step ownership/role transfer, never one-shot. **Detect:** `owner = newOwner` assigned directly in one call rather than a propose/accept pair.
22. Timelock irreversible or high-impact admin changes (fees, oracle address, upgrade target). **Detect:** a critical-parameter setter that takes effect in the same call, with no `pendingX`/`readyAt` delay.

### Checks-Effects-Interactions & Money

23. CEI ordering: checks → effects → interactions. **Detect:** an external call (`.call`, `.transfer`, external interface invocation) appearing before a storage write or event in the same function.
24. Reentrancy guard on every externally-callable function touching external calls + shared state. **Detect:** a function containing an external call, without `nonReentrant` (or equivalent), that isn't read-only.
25. Read-only reentrancy risk. **Detect:** a `view` function returning a ratio/price derived from state that another function updates non-atomically (e.g. deposit/withdraw pairs) — flag for manual review.
26. Don't accept ETH unless ETH handling is core to the contract. **Detect:** a `receive()`/payable `fallback()` present while ETH isn't used anywhere else in the contract.
27. Never use `address(this).balance` as internal accounting. **Detect:** `address(this).balance` used in a state-changing calculation, not just a diagnostic view.
28. Avoid `.transfer()`/`.send()`; use `.call{value: x}("")`. **Detect:** literal `.transfer(` or `.send(` on an address.
29. Prefer pull-payments over push loops. **Detect:** a loop sending ETH/tokens to multiple addresses within one external function.
30. Always check return values of calls that return one. **Detect:** an unchecked `.call(...)` result, or an ERC20 `transfer`/`transferFrom` return value ignored and not wrapped in `SafeERC20`.
31. Handle fee-on-transfer/rebasing tokens via balance delta, not the trusted parameter. **Detect:** a deposit-style function that credits the caller with the `amount` parameter directly instead of `balanceAfter - balanceBefore`.
32. Never allow `delegatecall` to a user-suppliable address. **Detect:** `.delegatecall(` where the target isn't hardcoded or allowlisted.
33. Cap/rate-limit gas forwarded to unbounded external calls inside loops. **Detect:** a loop making an external call per iteration with no gas stipend or iteration cap.
34. Add slippage + deadline params to price/state-dependent functions. **Detect:** a swap/trade-style function missing a `deadline` and/or `minOut`-style parameter.
35. Use `SafeERC20.forceApprove` for tokens requiring zero-first approval (USDT-style). **Detect:** raw `.approve(` call on an arbitrary `IERC20`.
36. ERC-4626/share-based vaults need a first-depositor inflation guard. **Detect:** an `ERC4626`/share-based vault constructor with no dead-shares mint or virtual-shares offset — the standard first finding on any vault review ([EIP-4626](https://eips.ethereum.org/EIPS/eip-4626)).

### Errors, Events & Assertions

37. Every state-changing function — including admin/privilege changes — emits an event. **Detect:** a function that writes storage with no corresponding `emit`.
38. Custom errors over `require(cond, "string")`. **Detect:** `require(` with a string-literal second argument.
39. Factor a repeated check into a modifier. **Detect:** the same revert condition duplicated verbatim in 2+ functions.
40. Validate inside `internal`/`private` helpers too. **Detect:** an `internal` state-changing function with no precondition checks, called from more than one external entry point.
41. Reserve `assert()` for impossible conditions. **Detect:** `assert(` guarding a condition derived from a function parameter or external input.

### Inheritance & Upgradeability

42. Pausable — and deliberately, possibly upgradeable — for non-trivial/high-value contracts. **Detect:** a contract handling transfers of value with no `Pausable`/circuit-breaker.
43. Storage layout is append-only across upgrades. **Detect (compare old vs. new version):** an existing state variable's slot position or type changed.
44. Never add a new base contract with its own state during an upgrade. **Detect (compare old vs. new version):** a new parent contract declaring state variables added in a version bump.
45. Keep inheritance shallow. When 2+ parents implement the same function, `override(B, C)` is required — but that list's order doesn't control resolution; the contract's own inheritance declaration order does (C3 linearization), and `super.foo()` walks the full linearized chain, not just the nearest parent. **Detect:** a function implemented/overridden by 2+ parents in the same contract's inheritance list.
46. `_disableInitializers()` in the implementation constructor. **Detect:** an `Initializable` contract's constructor missing this call.
47. `__gap` arrays in every upgradeable base contract. **Detect:** an upgradeable base contract with state and no trailing `uint256[N] private __gap;`.
48. UUPS bricking risk: verify `_authorizeUpgrade` can't be accidentally removed. **Detect:** `_authorizeUpgrade` gated by a role with no recovery path if that role is lost.

### Oracles & External Data

49. Never use a DEX spot price as an oracle. **Detect:** a price calculation reading `getReserves()`/`slot0` directly, with no TWAP wrapper.
50. Check oracle staleness on every read. **Detect:** a Chainlink `latestRoundData()` call whose `updatedAt` return value is discarded.
51. Sanity-bound oracle values before use. **Detect:** an oracle-returned price used directly with no min/max bounds check.
52. Fallback/pause path for a stale or unavailable oracle. **Detect:** a staleness check present with no corresponding revert/pause branch.

### Compiler, Types, Math & Signatures

53. Pin an exact compiler version. **Detect:** `pragma solidity` using `^`, `~`, or a range instead of an exact version.
54. `unchecked` blocks need a proof comment. **Detect:** an `unchecked { ... }` block with no preceding comment explaining why overflow/underflow is impossible.
55. Multiply before you divide; document rounding. **Detect:** division applied before multiplication in the same value chain.
56. Never use `tx.origin` for authorization. **Detect:** literal `tx.origin` in any conditional ([SWC-115](http://swcregistry.io/docs/SWC-115/)).
57. Check `ecrecover` isn't returning `address(0)`. **Detect:** `ecrecover(` result used without a zero-address check and not using OZ `ECDSA.recover`.
58. Use OZ `ECDSA` + nonce/chainid (EIP-712) for signatures. **Detect:** raw `ecrecover` on a custom hash with no nonce or `block.chainid` bound into the signed data.
59. Never use `blockhash`/`block.timestamp`/`block.difficulty` as randomness. **Detect:** any of these combined with `%` to produce a "random" value ([SWC-120](http://swcregistry.io/docs/SWC-120/)).

### Testing & Tooling — Trigger / Flag (recognize the code pattern, don't check for existing tests)

60. **Trigger:** a revert condition exists (require/custom error). **Flag:** "Unit test needed — assert this reverts under [condition] and succeeds otherwise."
61. **Trigger:** a parameter is used in arithmetic or as an array index. **Flag:** "Fuzz test needed — fuzz this parameter with `bound()` across its valid range and assert [invariant] holds."
62. **Trigger:** the contract calls an external token/oracle interface. **Flag:** "Fork test needed — test against the real mainnet contract at a pinned block; mocks won't catch non-standard behavior."
63. **Trigger:** the contract has a solvency/conservation-style property spanning multiple functions (vault, pool, ledger). **Flag:** "Invariant test needed — assert [property] holds via a Foundry handler across randomized call sequences."
64. **Trigger:** a function sends ETH or tokens to an address parameter. **Flag:** "Unit test needed — test with a receiver that reverts, burns gas, or reenters on receipt."
65. **Trigger:** a function has an access-control modifier/check. **Flag:** "Unit test needed — assert an unauthorized caller reverts."
66. **Trigger:** reviewing a repo with no CI config found. **Flag:** "CI setup needed — add Slither (and ideally Aderyn) as a required check."
67. **Trigger:** repo has no gas-snapshot mechanism. **Flag:** "CI setup needed — add `forge snapshot` to catch unintended gas regressions."
68. **Trigger:** repo runs only one static analyzer. **Flag:** "Tooling gap — add a second analyzer (e.g. Aderyn) with a different detector set."
69. **Trigger:** contract manages significant value (vault, lending, bridge-style logic) with no spec/symbolic test files present. **Flag:** "Consider Halmos or Certora coverage for [invariant] beyond fuzzing."

### Advanced / Situational

70. Snapshot voting power for governance instead of live balances. **Detect:** a governance contract reading `balanceOf` directly for vote weight instead of a checkpoint/snapshot.
71. EIP-7702 changes a long-standing assumption: an address with no code today can delegate to contract code mid-transaction. **Detect:** `extcodesize(addr) == 0` used as a definitive "this is an EOA" security check.
72. Internal function pointers stored in state can become invalid after an upgrade. **Detect:** a function-type variable declared in upgradeable contract storage.
73. Negating `type(int256).min` overflows. **Detect:** signed integer negation inside an `unchecked` block.
74. Precompute and verify CREATE2 addresses before relying on deterministic deployment across chains. **Detect:** `create2`/`CREATE2` usage with an undocumented or dynamically-derived salt.

---

## Phase 4 — Dependencies

| Rule | Practice |
|------|----------|
| Known-good libs | OpenZeppelin, Solmate, PRBMath, Gnosis Safe — prefer audited, widely used |
| Pin versions | Exact versions in `package.json`/`foundry.toml`; never floating |
| Review once | Read each dependency at least once; don't trust blindly |
| Minimize surface | Remove unused deps; don't import a whole lib for one function |
| Stay current | Subscribe to security advisories; update deliberately with re-review |
| Supply chain | Lock files committed; CI checks dependency integrity |

---

## Phase 5 — Testing Setup

### Test structure

```
test/
├── unit/           # one file per contract/module
├── integration/    # one file per component relationship
├── invariant/      # handler + invariant_*.t.sol
└── utils/          # shared fixtures, helpers
```

Which tests to write for a given piece of code is driven by **The Checklist**'s Trigger/Flag items above — this section is about how to set the test suite up, not what to write.

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
- Multiple actors, not just `address(this)`
- Save failing sequences as permanent regression tests

---

## Phase 6 — Code Review Protocol

Every significant change: author + ≥1 reviewer. Treat review like a lightweight audit.

1. **Context** — architecture and change intent (author talks, reviewer listens)
2. **Diff walk-through** — run **The Checklist** against every changed function
3. **Invariants** — each spec invariant (Phase 0) has test coverage?
4. **Static analysis** — Slither/Aderyn diff triaged, even informational

Record findings in a short PR audit log (template in [reference.md](reference.md)).

---

## Phase 7 — Staged Deployment

```
Private testnet (fork) → Public testnet → Mainnet beta (capped) → Mainnet release
```

| Stage | Practice |
|-------|----------|
| Local/fork | Deploy in test suite; simulate failures with cheatcodes |
| Public testnet | Community QA; test monitoring; open feedback channel |
| Mainnet beta | Cap deposits; bug bounty live; incident plan ready; set sunset block |
| Mainnet release | Remove caps gradually; monitoring watch window |

This phase is organizational — it can't be verified by reading the contract code, only followed as process. Keep it as a human checklist for the launch itself, separate from the code-level checklist above.

---

## Phase 8 — AI-Assisted Development

AI output is **untrusted** until compiled, tested, and reviewed.

1. Never zero-shot to production
2. Prompt: interface → control flow (checks → state → events) → invariants → code
3. RAG against OpenZeppelin; cap at ~2 examples
4. Compile immediately; verify imports are real
5. Run **The Checklist** against the generated code before treating it as done
6. Same review + test gauntlet as human code

---

## Additional Resources

- Extended conventions, invariant templates, resource library, PR audit-log template: [reference.md](reference.md)
- Good vs. poor patterns with code: [examples.md](examples.md)
