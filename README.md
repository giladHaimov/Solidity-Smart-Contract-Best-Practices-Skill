# Solidity Best Practices Skill

Handbook + AI skill for writing Solidity that you'd actually want to maintain six months from now. Focus is **engineering quality**: naming, layout, invariants, tests, reviews, deploys — not a full audit methodology.

Built from Solidity docs, OpenZeppelin, SCSFG, Trail of Bits, Secureum, Solcurity, the SWC Registry, and the Foundry Book. Every item below cites where it comes from.

---

## What's in the repo

```
Solidity-Smart-Contract-Best-Practices-Skill/   ← GitHub repo (this folder)
├── README.md         ← this file: categorized list, cited, human-readable
├── SKILL.md           ← what the agent loads: same list, engine-oriented (detection cues, no prose)
├── reference.md      ← naming tables, invariant templates, deployment checklists
└── examples.md       ← good vs. sloppy code, side by side
```

---

## Install it

Works as a skill in **Claude Code or Cursor** — copy the three files into that tool's skills folder.

```bash
# Claude Code
mkdir -p .claude/skills/solidity-best-practices
cp SKILL.md reference.md examples.md .claude/skills/solidity-best-practices/

# or Cursor
mkdir -p .cursor/skills/solidity-best-practices
cp SKILL.md reference.md examples.md .cursor/skills/solidity-best-practices/
```

Global install: same layout under `~/.claude/skills/` or `~/.cursor/skills/`. Not installing as a skill? The markdown works fine as a team doc on its own.

---

## Code-checkable vs. test-checkable

Every item below is written so an AI (or a human) can actually apply it to real code — not just read it as a slogan. Two different mechanisms, depending on the item:

- **Code-checkable** — the violation (or its absence) is a pattern in the `.sol` file itself: a missing `_` prefix, `tx.origin` in a condition, an external call before a state write. The engine reads the contract and reports pass/fail directly. Most items below are this kind.
- **Test-checkable** — the item isn't about the contract's code, it's about whether a specific *test* exists for it (a fuzz test, a negative-access-control test). The engine can't verify a test's existence by reading the contract, so instead it recognizes the **code pattern that creates the need** (e.g. "this function sends ETH to an address parameter") and outputs a fixed recommendation — "adversarial-receiver test needed, here's why, here's roughly what it should look like" — rather than trying to confirm the test is already there.

A third category exists but isn't included below: pure **process** facts with no trace in any code or config file at all — who holds a multisig key, whether someone clicked "approve" on a PR, whether Etherscan verification actually ran. Those aren't checklist items an AI can evaluate from a repo, so they're left out of this list entirely (`SKILL.md` documents why, if you're curious).

---

## The Checklist

74 items, organized by category, each cited to a primary source. **Detect** = code-checkable. **Trigger → Flag** = test-checkable (see above).

### Visibility & Encapsulation

1. Prefix `private`/`internal` members with `_`. — [OZ GUIDELINES](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/GUIDELINES.md)
2. Default to minimal visibility: `private` > `internal` > `external` > `public`. — [Solidity style guide](https://docs.soliditylang.org/en/latest/style-guide.html)
3. State variables: always `private`/`internal`; add a `view` getter only if needed. — [OZ GUIDELINES](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/GUIDELINES.md)
4. Store dependencies as their interface type (`IERC20`), not `address`. — [SCSFG defensive programming](https://scsfg.io/developers/defensive-programming/)
5. Named imports only, no wildcards. — [Solidity style guide](https://docs.soliditylang.org/en/latest/style-guide.html)
6. Interfaces get their own file, prefixed `I`. — [Solidity style guide](https://docs.soliditylang.org/en/latest/style-guide.html)
7. Mark functions `virtual` by default in library/base contracts, except trivial alias functions. — [OZ GUIDELINES](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/GUIDELINES.md)
8. Mark `override` explicitly on every overriding function. — [Solidity docs — inheritance](https://docs.soliditylang.org/en/latest/contracts.html)

### State, Storage & Data Structures

9. Use `mapping` for key-based lookup; `array` only for iteration. — [Solidity docs — types](https://docs.soliditylang.org/en/latest/types.html)
10. Bound or cap any collection a user can grow. — [SWC-128: DoS with Block Gas Limit](http://swcregistry.io/docs/SWC-128/)
11. Pack struct fields to fit 32-byte slots. — [Solidity docs — storage layout](https://docs.soliditylang.org/en/latest/internals/layout_in_storage.html)
12. Anything fixed after the constructor: `immutable`/`constant`. In upgradeable contracts `immutable` is still safe (bytecode, not storage) but only for a value identical across every proxy sharing that implementation. — [OZ Upgrades Plugins](https://docs.openzeppelin.com/upgrades-plugins/writing-upgradeable)
13. Nothing on-chain is private data. — [Solidity docs — security considerations](https://docs.soliditylang.org/en/latest/security-considerations.html)
14. `delete` doesn't clear mappings nested inside array/struct elements. — [Solidity docs — types](https://docs.soliditylang.org/en/latest/types.html)
15. Downcasting integers truncates silently — use `SafeCast`. — [OZ Utils API](https://docs.openzeppelin.com/contracts/5.x/api/utils)
16. Never compare balance with `==` for accounting; balances can be force-fed. — [Trail of Bits — Building Secure Contracts](https://github.com/crytic/building-secure-contracts)
17. Never hardcode chain-specific addresses in contract code. — [SCSFG deployment](https://scsfg.io/developers/deployment/)

### Access Control & Privilege

18. Every state-mutating function needs an access modifier or an explicit "intentionally open" comment. — [SCSFG defensive programming](https://scsfg.io/developers/defensive-programming/)
19. If a user-triggered function mutates data outside that user's own record, require an explicit check. — *heuristic, in the spirit of [SCSFG's least-privilege guidance](https://scsfg.io/developers/defensive-programming/); no single doc states this exact rule*
20. `AccessControl` for multi-role systems; `Ownable2Step` is enough for single-admin contracts. — [OZ Access Control docs](https://docs.openzeppelin.com/contracts/5.x/api/access)
21. Two-step ownership/role transfer, never one-shot. — [OZ Access Control docs](https://docs.openzeppelin.com/contracts/5.x/api/access)
22. Timelock irreversible or high-impact admin changes. — [OZ Governance docs](https://docs.openzeppelin.com/contracts/5.x/api/governance)

### Checks-Effects-Interactions & Money

23. CEI ordering: checks → effects → interactions. — [Solidity common patterns](https://docs.soliditylang.org/en/latest/common-patterns.html)
24. Reentrancy guard on every externally-callable function touching external calls + shared state. — [OZ Utils API](https://docs.openzeppelin.com/contracts/5.x/api/utils)
25. Read-only reentrancy: `view` functions can return manipulated/stale state mid-callback. — [SWC-107: Reentrancy](http://swcregistry.io/docs/SWC-107/)
26. Don't accept ETH unless ETH handling is core to the contract. — [Solidity common patterns](https://docs.soliditylang.org/en/latest/common-patterns.html)
27. Never use `address(this).balance` as internal accounting. — [Trail of Bits — Building Secure Contracts](https://github.com/crytic/building-secure-contracts)
28. Avoid `.transfer()`/`.send()`; use `.call{value: x}("")`. — [Solidity docs — address members](https://docs.soliditylang.org/en/latest/units-and-global-variables.html)
29. Prefer pull-payments over push loops. — [Solidity common patterns](https://docs.soliditylang.org/en/latest/common-patterns.html)
30. Always check the return value of calls that return one. — [SWC-104: Unchecked Call Return Value](http://swcregistry.io/docs/SWC-104/)
31. Handle fee-on-transfer/rebasing tokens via balance delta, not the trusted parameter. — [weird-erc20](https://github.com/d-xo/weird-erc20)
32. Never allow `delegatecall` to a user-suppliable address. — [SWC-112: Delegatecall to Untrusted Callee](http://swcregistry.io/docs/SWC-112/)
33. Cap/rate-limit gas forwarded to unbounded external calls inside loops. — [SWC-128: DoS with Block Gas Limit](http://swcregistry.io/docs/SWC-128/)
34. Add slippage + deadline params to price/state-dependent functions. — [DASP Top 10 — Front-Running](https://www.dasp.co/)
35. Use `SafeERC20.forceApprove` for tokens requiring zero-first approval (USDT-style). — [OZ Token API](https://docs.openzeppelin.com/contracts/5.x/api/token/erc20)
36. ERC-4626/share-based vaults need a first-depositor inflation guard. — [EIP-4626](https://eips.ethereum.org/EIPS/eip-4626)

### Errors, Events & Assertions

37. Every state-changing function — including admin/privilege changes — emits an event. — [OZ GUIDELINES](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/GUIDELINES.md)
38. Custom errors over `require(cond, "string")`. — [EIP-6093](https://eips.ethereum.org/EIPS/eip-6093)
39. Factor a repeated check used at the start of 2+ functions into a modifier. — [Solidity style guide](https://docs.soliditylang.org/en/latest/style-guide.html)
40. Validate inside `internal`/`private` helpers too, not just the external entry point. — [SCSFG defensive programming](https://scsfg.io/developers/defensive-programming/)
41. Reserve `assert()` for conditions that should be mathematically impossible. — [Solidity docs — control structures](https://docs.soliditylang.org/en/latest/control-structures.html)

### Inheritance & Upgradeability

42. Pausable — and deliberately, possibly upgradeable — for non-trivial/high-value contracts. — [OZ Utils API](https://docs.openzeppelin.com/contracts/5.x/api/utils)
43. Storage layout is append-only across upgrades. — [OZ Upgrades Plugins](https://docs.openzeppelin.com/upgrades-plugins/writing-upgradeable)
44. Never add a new base contract with its own state during an upgrade. — [OZ Upgrades Plugins](https://docs.openzeppelin.com/upgrades-plugins/writing-upgradeable)
45. Keep inheritance shallow. When 2+ parents implement the same function, resolution follows the contract's own inheritance declaration order (C3 linearization), not the order written in `override(B, C)` — and `super.foo()` walks the full chain, not just the nearest parent. — [Solidity docs — inheritance](https://docs.soliditylang.org/en/latest/contracts.html)
46. `_disableInitializers()` in the implementation constructor. — [OZ Proxy API](https://docs.openzeppelin.com/contracts/5.x/api/proxy)
47. `__gap` arrays in every upgradeable base contract. — [OZ Upgrades Plugins](https://docs.openzeppelin.com/upgrades-plugins/writing-upgradeable)
48. UUPS bricking risk: verify `_authorizeUpgrade` can't be accidentally removed. — [OZ Proxy API](https://docs.openzeppelin.com/contracts/5.x/api/proxy)

### Oracles & External Data

49. Never use a DEX spot price as an oracle; use a TWAP or dedicated oracle. — [Chainlink Data Feeds docs](https://docs.chain.link/data-feeds)
50. Check oracle staleness (`updatedAt`/heartbeat) on every read. — [Chainlink Data Feeds docs](https://docs.chain.link/data-feeds)
51. Sanity-bound oracle values before use. — [Chainlink Data Feeds docs](https://docs.chain.link/data-feeds)
52. Have a fallback or pause path for a stale/unavailable oracle. — [Chainlink Data Feeds docs](https://docs.chain.link/data-feeds)

### Compiler, Types, Math & Signatures

53. Pin an exact compiler version, never a floating range. — [Solidity style guide](https://docs.soliditylang.org/en/latest/style-guide.html)
54. `unchecked` blocks require a comment proving why overflow/underflow can't happen. — [Solidity docs — control structures](https://docs.soliditylang.org/en/latest/control-structures.html)
55. Multiply before you divide; document rounding direction. — [SCSFG defensive programming](https://scsfg.io/developers/defensive-programming/)
56. Never use `tx.origin` for authorization. — [SWC-115: Authorization through tx.origin](http://swcregistry.io/docs/SWC-115/)
57. Check `ecrecover` isn't returning `address(0)`. — [OZ Cryptography API](https://docs.openzeppelin.com/contracts/5.x/api/utils/cryptography)
58. Use OZ `ECDSA` + nonce/chainid for signatures. — [EIP-712](https://eips.ethereum.org/EIPS/eip-712)
59. Never use `blockhash`/`block.timestamp`/`block.difficulty` as randomness. — [SWC-120: Weak Sources of Randomness](http://swcregistry.io/docs/SWC-120/)

### Testing & Tooling (Trigger → Flag)

60. **Trigger:** a revert condition exists. **Flag:** unit test needed for the revert path. — [Foundry Book — tests](https://book.getfoundry.sh/)
61. **Trigger:** a parameter is used in arithmetic or as an array index. **Flag:** fuzz test needed with `bound()`. — [Foundry Book — fuzz testing](https://book.getfoundry.sh/forge/fuzz-testing)
62. **Trigger:** the contract calls an external token/oracle interface. **Flag:** fork test needed at a pinned block. — [Foundry Book](https://book.getfoundry.sh/)
63. **Trigger:** a solvency/conservation property spans multiple functions. **Flag:** invariant test needed. — [Foundry Book — invariant testing](https://book.getfoundry.sh/guides/invariant-testing)
64. **Trigger:** a function sends ETH/tokens to an address parameter. **Flag:** adversarial-receiver test needed. — [Trail of Bits — Building Secure Contracts](https://github.com/crytic/building-secure-contracts)
65. **Trigger:** a function has an access-control check. **Flag:** negative-path (unauthorized-caller) test needed. — [Foundry Book](https://book.getfoundry.sh/)
66. **Trigger:** repo has no CI static analysis. **Flag:** add Slither as a required check. — [Slither](https://github.com/crytic/slither)
67. **Trigger:** repo has no gas-regression tracking. **Flag:** add `forge snapshot`. — [Foundry Book](https://book.getfoundry.sh/)
68. **Trigger:** repo runs only one static analyzer. **Flag:** add a second one with a different detector set. — [Aderyn](https://github.com/Cyfrin/aderyn)
69. **Trigger:** contract manages significant value with no formal-verification artifacts. **Flag:** consider symbolic testing. — [Halmos](https://github.com/a16z/halmos)

### Advanced / Situational

70. Snapshot voting power for governance instead of live balances. — [OZ Governance API](https://docs.openzeppelin.com/contracts/5.x/api/governance)
71. EIP-7702 lets an EOA delegate to contract code mid-transaction — don't treat "no code" as a permanent guarantee. — [EIP-7702](https://eips.ethereum.org/EIPS/eip-7702)
72. Internal function pointers stored in state can become invalid after an upgrade. — [OZ Upgrades Plugins](https://docs.openzeppelin.com/upgrades-plugins/writing-upgradeable)
73. Negating `type(int256).min` overflows. — [Solidity docs — types](https://docs.soliditylang.org/en/latest/types.html)
74. Precompute and verify CREATE2 addresses before relying on deterministic deployment. — [Solidity docs — control structures](https://docs.soliditylang.org/en/latest/control-structures.html)

---

## Where to go next

| If you… | Read… |
|---------|--------|
| Want the agent to follow this | `SKILL.md` — same 74 items, written as detection cues instead of prose |
| Need templates (invariant catalog, PR audit log) or the full resource library | `reference.md` |
| Want "show me good code vs. bad code" | `examples.md` |

Questions, gaps, wrong citations — open an issue or fix the doc. This is a living skill, not a whitepaper.
