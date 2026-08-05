# Solidity Best Practices — Reference

Extended conventions, checklists, invariant templates, and authoritative resources. Read when implementing or reviewing specific areas.

---

## Resource Library (by topic)

### Official Solidity

| Resource | URL | Best for |
|----------|-----|----------|
| Solidity Style Guide | https://docs.soliditylang.org/en/latest/style-guide.html | Layout, naming, NatSpec, element order |
| Common Patterns | https://docs.soliditylang.org/en/latest/common-patterns.html | Withdrawal, access restriction, state machines |
| Security Considerations | https://docs.soliditylang.org/en/latest/security-considerations.html | Modularity, CEI, fail-safe, peer review |
| NatSpec Format | https://docs.soliditylang.org/en/latest/natspec-format.html | `@notice`, `@param`, `@dev` tags |
| Breaking Changes (0.8.x) | https://docs.soliditylang.org/en/latest/080-breaking-changes.html | Compiler migration |
| Supported compiler versions | https://github.com/argotorg/solidity/security/policy#supported-versions | Which versions receive patches |

### Smart Contract Security Field Guide (SCSFG)

| Section | URL | Best for |
|---------|-----|----------|
| Developers hub | https://scsfg.io/developers/ | Roadmap index |
| System Design | https://scsfg.io/developers/system-design/ | Composability, diagrams, data separation |
| Defensive Programming | https://scsfg.io/developers/defensive-programming/ | PoLP, proactive checks, concrete types, delays |
| Dependencies | https://scsfg.io/developers/dependencies/ | Supply chain, pinning, hygiene |
| Documentation | https://scsfg.io/developers/documentation/ | Spec levels, NatSpec, auto-doc in CI |
| Testing | https://scsfg.io/developers/testing/ | Test types, TDD, structure, forking |
| Deployment | https://scsfg.io/developers/deployment/ | Staged rollout, init security, beta sunset |
| Upgradeability | https://scsfg.io/developers/upgradeability/ | Proxy patterns, storage collisions, when to avoid |
| Monitoring | https://scsfg.io/developers/monitoring/ | Events, alerting, incident response |
| Audit Preparation | https://scsfg.io/developers/audit-preparation/ | Pre-audit documentation checklist |

### OpenZeppelin

| Resource | URL | Best for |
|----------|-----|----------|
| GUIDELINES.md | https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/GUIDELINES.md | State encapsulation, events, errors, testing culture |
| solidity-style skill | https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/.claude/skills/solidity-style/SKILL.md | Imports, assembly, inheritance, errors |
| Contracts docs | https://docs.openzeppelin.com/contracts/ | ERC implementations, AccessControl, proxies |
| Security Center | https://contracts.openzeppelin.com/security | Release process, supported versions |
| Upgrades plugin | https://docs.openzeppelin.com/upgrades-plugins/ | Storage layout validation |

### ethereum.org

| Resource | URL | Best for |
|----------|-----|----------|
| Smart contract security | https://docs.ethereum.org/developers/docs/smart-contracts/security/ | Lifecycle, complexity reduction, peer review |
| Security guidelines tutorial | https://ethereum.org/developers/tutorials/smart-contract-security-guidelines/ | Documentation, implementation, compiler choice |

### ConsenSys / Diligence

| Resource | URL | Best for |
|----------|-----|----------|
| Development Recommendations | https://consensysdiligence.github.io/smart-contract-best-practices/development-recommendations/ | General principles, Solidity quirks |
| Field Guide (successor) | https://scsfg.io/ | Replaces deprecated best-practices repo |

### Trail of Bits

| Resource | URL | Best for |
|----------|-----|----------|
| Invariant-driven development | https://blog.trailofbits.com/2025/02/12/the-call-for-invariant-driven-development/ | Invariants across SDLC |
| Invariant development service | https://blog.trailofbits.com/2023/10/05/introducing-invariant-development-as-a-service/ | Handler patterns, CI integration |
| Building Secure Contracts | https://github.com/crytic/building-secure-contracts | Dev guidelines, program analysis |
| secure-workflow-guide skill | https://github.com/trailofbits/skills/tree/main/plugins/building-secure-contracts | 5-step dev workflow, Slither integration |
| Blockchain services | https://trailofbits.com/services/blockchain/ | Design assessment, code maturity |

### Foundry & testing

| Resource | URL | Best for |
|----------|-----|----------|
| Foundry Book | https://book.getfoundry.sh/ | forge, cast, anvil, cheatcodes |
| Invariant testing guide | https://book.getfoundry.sh/guides/invariant-testing | Handlers, ghost vars, config |
| Moloch Testing Guide | https://github.com/MolochVentures/moloch/tree/master/test#readme | Test quality philosophy (referenced by OZ) |

### Community references

| Resource | URL | Best for |
|----------|-----|----------|
| Guy Lando knowledge list | https://github.com/guylando/KnowledgeLists/blob/master/EthereumSmartContracts.md | Curated Ethereum dev links |
| Solidity Patterns | https://github.com/fravoll/solidity-patterns | Reusable design patterns |
| Rari Capital style guide | https://github.com/Rari-Capital/solcurity | Style + review checklist |

### Tools (defaults)

| Need | Default |
|------|---------|
| Standard contracts | OpenZeppelin Contracts / Contracts Upgradeable |
| Gas-optimized primitives | Solmate |
| Fixed-point math | PRBMath |
| Multisig | Gnosis Safe |
| Test framework | Foundry (preferred) or Hardhat |
| Static analysis | Slither (CI), Aderyn |
| Linting | Solhint |
| Fuzz / invariant | Foundry + Echidna/Medusa for extended campaigns |
| Upgrade safety | OpenZeppelin Upgrades plugin |
| Verification | Etherscan, Sourcify |
| Doc generation | solidity-docgen |
| Diagrams | Slither printers, Surya, Mermaid |

---

## Invariant Catalog Template

Copy into your project spec. Each invariant should map to at least one test.

```markdown
# Protocol Invariants

## Global (protocol-level)
| ID | Invariant | Test | Priority |
|----|-----------|------|----------|
| G-01 | totalAssets >= totalLiabilities | invariant_solvency | P0 |
| G-02 | totalSupply == sum(balances) | invariant_supplyConservation | P0 |
| G-03 | feeBps <= MAX_FEE_BPS | invariant_feeBounds | P1 |

## Per-function (FREI-PI)
### deposit(assets, receiver)
- **Requirement:** assets > 0, !paused, receiver != address(0)
- **Effect:** _totalAssets += assets; mint shares to receiver
- **Interaction:** safeTransferFrom(msg.sender, address(this), assets)
- **Invariant preserved:** G-01, G-02
- **Event:** Deposited(msg.sender, receiver, assets, shares)

### withdraw(shares, receiver, owner)
- ...
```

### Invariant categories reference

| Category | Description | Example assertion |
|----------|-------------|-------------------|
| Solvency | Contract can cover all liabilities | `totalAssets >= totalLiabilities` |
| Conservation | Inputs equal outputs | `totalSupply == sum(balances)` |
| Monotonicity | Value only moves one direction | `epochNumber` only increases |
| Bounds | Values stay in range | `feeBps <= MAX_FEE` |
| Access control | Only authorized actors act | `onlyRole(MINTER)` enforced |
| State consistency | Related vars stay in sync | `locked + available == total` |

---

## FREI-PI Framework

For each state-changing function, document and test:

| Step | Question | Output |
|------|----------|--------|
| **F**unction | What does it do? | One-line summary |
| **R**equirement | What must be true before? | Preconditions / modifiers |
| **E**ffect | What state changes? | Variables written, events emitted |
| **I**nteraction | What external calls? | Token transfers, ETH sends (after effects) |
| **P**rotocol invariant | What must still hold? | Link to invariant catalog ID |

Assert protocol invariants in unit tests after each state-changing call. Use invariant tests for cross-function properties.

---

## Composable Architecture (SCSFG)

Three principles:

1. **Modularity** — separate by business-logic concern, not arbitrary file splits
2. **Autonomy** — each contract operates independently unless explicitly integrated
3. **Discoverability** — verified source on Etherscan; NatSpec on all public interfaces

### Module design rules

- Each module gets only the permissions it needs (PoLP)
- Minimize cross-module dependencies
- Prefer composition over deep inheritance chains
- Keep state localized — avoid Diamond/Eternal Storage unless no alternative
- Extract to libraries for pure/view logic shared across contracts

### When refactoring monolith → modules

1. Map existing logic and data flows
2. Detach one module at a time
3. Test before and after each extraction
4. Do not rush — incremental is safer

---

## Architecture Diagrams (SCSFG)

### Types to maintain

| Diagram | Purpose | Tool |
|---------|---------|------|
| System architecture (SAD) | Contracts, proxies, libraries, external deps | draw.io, Mermaid |
| Inheritance tree | Class hierarchy | Slither/Surya auto-gen |
| Sequence (SSD) | User flows, cross-contract calls | Mermaid, WebSequenceDiagrams |
| Authorization map | Who can call what | Slither printers |
| Deployment | Chain, addresses, proxy→impl | Manual + deploy scripts |

### Diagram hygiene

- Balance count: too few = gaps; too many = unmaintainable
- Structural consistency: same shapes/colors across diagrams
- Semantic consistency: diagrams sync with code on every release
- Legends when not using UML
- Version control diagrams alongside code
- Blend auto-generated (inheritance) + manual (sequence, trust boundaries)

---

## Naming Conventions

| Element | Convention | Example |
|---------|------------|---------|
| Contract | CapWords | `TokenVault` |
| Interface | `I` + CapWords | `IERC20`, `IVault` |
| Library | CapWords | `SafeCast` |
| Struct / Enum | CapWords | `DepositInfo`, `OrderStatus` |
| Constant | `ALL_CAPS` | `MAX_FEE_BPS` |
| Immutable | `_camelCase` | `_asset` |
| State variable | `_camelCase` | `_totalSupply` |
| Custom error | CapWords, domain-prefixed | `VaultInsufficientBalance` |
| Event | CapWords, past tense | `Deposited`, `RoleGranted` |
| Modifier | camelCase | `onlyOwner`, `whenNotPaused` |
| Test contract | `ContractNameTest` | `VaultTest` |
| Invariant test | `ContractInvariantTest` | `VaultInvariantTest` |
| Handler | `ContractHandler` | `VaultHandler` |

---

## Visibility Decision Tree

```
Is the function called from outside the contract?
├─ No  → internal or private
│         └─ Only this contract? → private
│         └─ Subclasses too?     → internal
└─ Yes → Also called internally?
          ├─ No  → external (calldata params)
          └─ Yes → public
```

State variables: default `private`. Expose via explicit `view` getters when logic is needed.

---

## Upgradeability Decision Framework (SCSFG)

### When to stay immutable

- Logic is simple and unlikely to need fixes
- Migration to new contract is acceptable
- Maximum trust guarantees required
- You can afford thorough pre-deploy testing + external audit

### When upgradeability is justified

- Complex evolving protocol with genuine maintenance need
- Migration would be prohibitively expensive for users
- Strong governance (multisig + timelock + community oversight) in place

### Pattern selection

| Pattern | When | Notes |
|---------|------|-------|
| Transparent proxy | Admin-heavy; separate admin/user call paths | OZ default for many projects |
| UUPS | Gas-sensitive deploys; upgrade logic in implementation | Risk: bricking if upgrade fn removed |
| Beacon | Many proxies sharing one implementation | Upgrade all proxies via beacon |
| Migration | New contract + user opt-in | Preferred over proxy when feasible |
| Diamond (EIP-2535) | Avoid unless no alternative | OZ excludes; high complexity |

### Storage rules (all proxy patterns)

- Append-only between versions
- `__gap` in every base contract
- Never reorder or change types of existing variables
- Never insert variables between existing ones
- Struct growth: append only; use `_gap` inside structs for large layouts
- Run OZ Upgrades storage diff in CI

---

## Dependency Hygiene Checklist (SCSFG)

- [ ] Each dependency reviewed at least once
- [ ] Exact versions pinned (no `^` or `~` in production lockfiles)
- [ ] Audit reports checked for mission-critical deps
- [ ] Unused dependencies removed
- [ ] No huge lib for one function — copy or use minimal import
- [ ] Overlap between deps resolved (pick one)
- [ ] Security bulletins monitored (GitHub advisories, Discord, Twitter)
- [ ] Lock files committed; CI validates integrity
- [ ] Repository precedence configured to prevent dependency confusion

---

## Testing Checklist (extended, SCSFG)

### Preparation

- [ ] Testing objectives defined (correctness, accounting, gas, user flows)
- [ ] Use cases documented
- [ ] Test plan written (scope, approach, schedule)
- [ ] Time budgeted for fixing test-discovered bugs

### Unit tests

- [ ] Happy path per public/external function
- [ ] Revert path per custom error / require
- [ ] Edge cases: zero, max, empty, unauthorized
- [ ] Access control: authorized + unauthorized callers
- [ ] Internal function validation (not just public shell)

### Integration tests

- [ ] Cross-contract interactions within system
- [ ] Third-party contract behavior (reverts, bad返回值, upgrades)
- [ ] Fork tests with real mainnet addresses

### E2E / system tests

- [ ] Complete user journeys (deposit → earn → withdraw)
- [ ] Invalid journey paths handled correctly
- [ ] Serve as integration examples for external devs

### Fuzz tests

- [ ] `bound()` on realistic ranges
- [ ] Math properties (non-negative, bounded, monotonic)
- [ ] Decimal normalization (6 vs 18 decimals)

### Invariant tests

- [ ] Handler constrains inputs
- [ ] Ghost variables for cumulative accounting
- [ ] Multiple actors
- [ ] All spec invariants covered
- [ ] Failing sequences → regression tests
- [ ] `depth` tuned for multi-step properties

### CI

- [ ] Compile with exact pragma
- [ ] Solhint + Slither
- [ ] Coverage threshold on changed files
- [ ] No merge on warnings
- [ ] Optional: extended Echidna/Medusa campaigns pre-release

---

## Staged Deployment Checklist (SCSFG)

### Private testnet / fork

- [ ] Deploy in test suite on mainnet fork
- [ ] Simulate failure conditions with cheatcodes
- [ ] Begin audit prep and threat modeling

### Public testnet

- [ ] Community can interact
- [ ] Monitoring tools tested
- [ ] Feedback channel open (Discord/Telegram)
- [ ] Bug bounty can launch

### Mainnet beta

- [ ] Deposit caps enforced
- [ ] Users informed of beta risks
- [ ] Bug bounty + monitoring operational
- [ ] Incident response plan ready
- [ ] Sunset block or auto-deprecation planned (e.g., 6 months)

### Mainnet release

- [ ] Caps removed gradually
- [ ] Full monitoring watch window
- [ ] Post-deploy smoke tests
- [ ] Changelog published

### Secure initialization (proxies)

- [ ] `initialize()` called in same tx as deploy
- [ ] Parent initializers not called twice
- [ ] `_disableInitializers()` in implementation constructor
- [ ] Implementation contract not usable directly

---

## Code Review Audit Log Template

```markdown
## Review Log

**Purpose:** [what this change does]
**Contracts:** [list of .sol files]
**Reviewer(s):** [names]
**Date:** [YYYY-MM-DD]

### Invariant coverage
| Invariant ID | Covered by test? | Notes |
|--------------|------------------|-------|
| G-01         | yes/no           |       |

### Findings
| # | Severity | Location | Issue | Resolution |
|---|----------|----------|-------|------------|

### Checklist
- [ ] Invariants verified
- [ ] Visibility sweep
- [ ] CEI order
- [ ] Defensive checks in internal fns
- [ ] Events on state changes
- [ ] NatSpec current
- [ ] Tests cover changed code
- [ ] Slither triaged
```

---

## Documentation Levels (SCSFG + ethereum.org)

| Level | Audience | Content | Format |
|-------|----------|---------|--------|
| Plain English | Stakeholders | Problem, approach, trust assumptions | README, Notion |
| Architecture | Developers, auditors | Contract diagram, deps, upgrade path | Mermaid, draw.io |
| NatSpec | Integrators | Per-function API, `@dev` assumptions | In-code `///` |
| Auto-generated | Integrators | Full API reference | solidity-docgen → GitHub Pages |
| Changelog | Users | Breaking changes, migrations | CHANGELOG.md |
| Runbook | Operators | Deploy, upgrade, incident steps | docs/ops.md |

---

## Trail of Bits — 5-Step Dev Workflow

For comprehensive pre-audit preparation:

1. **Known issues** — Slither on full codebase; triage all findings
2. **Special features** — `slither-check-upgradeability`, `slither-check-erc`, token integration review
3. **Visual inspection** — inheritance graphs, function summaries, authorization maps
4. **Security properties** — document invariants; set up Echidna/Manticore
5. **Manual review** — privacy, ordering, cryptography, DeFi assumptions

---

## Code Maturity Categories (Trail of Bits)

Evaluate project maturity across:

1. Arithmetic safety
2. Testing quality
3. Access controls
4. Complexity management
5. Code quality and readability
6. Protocol documentation
7. Transaction ordering risks
8. Declared vs actual behaviour
9. General engineering excellence

Use as a self-assessment before external audit.

---

## Version Notes

- Solidity 0.8.x: overflow checks on by default. Pin exact patch version.
- `require(condition, CustomError())` via IR in `^0.8.26`, stable `^0.8.27` — don't bump pragma solely for this.
- ConsenSys best-practices repo deprecated → use [scsfg.io](https://scsfg.io/).
- OpenZeppelin: all state variables should be `private` per GUIDELINES.md.
- Functions should be `virtual` by default in library code (OZ), with documented exceptions.
