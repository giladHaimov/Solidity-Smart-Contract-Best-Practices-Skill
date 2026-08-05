# Solidity Best Practices — Examples

Concrete patterns from [SKILL.md](SKILL.md), aligned with [SCSFG](https://scsfg.io/developers/), [OpenZeppelin GUIDELINES](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/GUIDELINES.md), and [Solidity Common Patterns](https://docs.soliditylang.org/en/latest/common-patterns.html).

---

## Visibility & Encapsulation

### Good — private state, controlled setter, event

```solidity
contract Vault {
    uint256 private _totalAssets;
    mapping(address => uint256) private _shares;

    event TotalAssetsUpdated(uint256 oldAssets, uint256 newAssets);

    function totalAssets() external view returns (uint256) {
        return _totalAssets;
    }

    function _setTotalAssets(uint256 newTotal) internal {
        uint256 old = _totalAssets;
        _totalAssets = newTotal;
        emit TotalAssetsUpdated(old, newTotal);
    }
}
```

### Poor — public state, no event

```solidity
contract Vault {
    uint256 public totalAssets;
    mapping(address => uint256) public shares;
}
```

---

## Concrete Types (SCSFG)

### Good — store interface, not address

```solidity
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";

contract Staking {
    IERC20 private immutable _stakingToken;

    constructor(IERC20 token_) {
        _stakingToken = token_;
    }

    function _stake(uint256 amount) internal {
        _stakingToken.transferFrom(msg.sender, address(this), amount);
    }
}
```

### Poor — repeated casting

```solidity
contract Staking {
    address private _token;

    function _stake(uint256 amount) internal {
        IERC20(_token).transferFrom(msg.sender, address(this), amount);
        // every call repeats the cast
    }
}
```

---

## Defensive Depth — Internal Validation (SCSFG)

### Good — validate in internal function too

```solidity
function deposit(uint256 assets, address receiver) external whenNotPaused returns (uint256) {
    return _deposit(assets, receiver);
}

function _deposit(uint256 assets, address receiver) internal returns (uint256 shares) {
    if (assets == 0) revert ZeroAmount();
    if (receiver == address(0)) revert ZeroAddress();
    shares = _convertToShares(assets);
    _mint(receiver, shares);
    _totalAssets += assets;
    emit Deposited(msg.sender, receiver, assets, shares);
}
```

### Poor — "hard shell, weak core"

```solidity
function deposit(uint256 assets, address receiver) external {
    if (assets == 0) revert ZeroAmount();
    _deposit(assets, receiver);  // no validation inside
}

function _deposit(uint256 assets, address receiver) internal {
    // if another path calls _deposit, zero-amount deposits are possible
    _mint(receiver, _convertToShares(assets));
}
```

---

## Custom Errors

### Good

```solidity
error ZeroAmount();
error Unauthorized(address caller);

function withdraw(uint256 amount) external {
    if (amount == 0) revert ZeroAmount();
    if (!hasRole(WITHDRAWER_ROLE, msg.sender)) revert Unauthorized(msg.sender);
}
```

### Poor

```solidity
require(amount > 0, "amount is zero");
require(hasRole(WITHDRAWER_ROLE, msg.sender), "not authorized");
```

---

## CEI (Checks-Effects-Interactions)

### Good

```solidity
function withdraw(uint256 amount) external {
    if (amount == 0) revert ZeroAmount();
    if (_balances[msg.sender] < amount) revert InsufficientBalance();

    _balances[msg.sender] -= amount;
    _totalDeposits -= amount;
    emit Withdrawn(msg.sender, amount);

    (bool ok, ) = msg.sender.call{value: amount}("");
    if (!ok) revert TransferFailed();
}
```

### Poor — interaction before effect

```solidity
function withdraw(uint256 amount) external {
    (bool ok, ) = msg.sender.call{value: amount}("");
    if (!ok) revert TransferFailed();
    _balances[msg.sender] -= amount;
}
```

---

## Pull-Over-Push (Solidity Common Patterns)

### Good

```solidity
mapping(address => uint256) private _pendingWithdrawals;

function requestWithdrawal(uint256 amount) external {
    _balances[msg.sender] -= amount;
    _pendingWithdrawals[msg.sender] += amount;
    emit WithdrawalRequested(msg.sender, amount);
}

function claimWithdrawal() external {
    uint256 amount = _pendingWithdrawals[msg.sender];
    if (amount == 0) revert NothingToClaim();
    _pendingWithdrawals[msg.sender] = 0;
    emit WithdrawalClaimed(msg.sender, amount);
    (bool ok, ) = msg.sender.call{value: amount}("");
    if (!ok) revert TransferFailed();
}
```

### Poor — push loop

```solidity
function distribute(address[] calldata recipients, uint256[] calldata amounts) external {
    for (uint256 i = 0; i < recipients.length; i++) {
        (bool ok, ) = recipients[i].call{value: amounts[i]}("");
        require(ok);
    }
}
```

---

## Purposeful Delays (SCSFG)

### Good — speed bump on admin action

```solidity
uint256 public constant DELAY = 2 days;
bytes32 public pendingAction;
uint256 public actionReadyAt;

event ActionScheduled(bytes32 indexed action, uint256 readyAt);

function scheduleAction(bytes32 action) external onlyRole(ADMIN_ROLE) {
    pendingAction = action;
    actionReadyAt = block.timestamp + DELAY;
    emit ActionScheduled(action, actionReadyAt);
}

function executeAction() external onlyRole(ADMIN_ROLE) {
    if (block.timestamp < actionReadyAt) revert ActionNotReady();
    // apply pendingAction...
    delete pendingAction;
    delete actionReadyAt;
}
```

---

## State Machine (Solidity Common Patterns)

### Good — lifecycle guarded by modifier

```solidity
enum Stage { Setup, Active, WindDown, Closed }

Stage private _stage;

modifier atStage(Stage required) {
    if (_stage != required) revert WrongStage();
    _;
}

function activate() external onlyRole(ADMIN_ROLE) atStage(Stage.Setup) {
    _stage = Stage.Active;
    emit StageChanged(Stage.Setup, Stage.Active);
}

function close() external onlyRole(ADMIN_ROLE) atStage(Stage.WindDown) {
    _stage = Stage.Closed;
    emit StageChanged(Stage.WindDown, Stage.Closed);
}
```

---

## NatSpec

### Good

```solidity
/// @notice Converts assets to shares at the current exchange rate
/// @param assets Amount of underlying tokens
/// @return shares Shares minted, rounded down in favor of the protocol
/// @dev Returns 0 when assets is 0. Reverts when vault is paused.
function convertToShares(uint256 assets) public view returns (uint256 shares) {
    return assets.mulDiv(_totalSupply, _totalAssets, Math.Rounding.Floor);
}
```

### Poor

```solidity
// converts assets to shares
function convertToShares(uint256 assets) public view returns (uint256) {
    return assets * _totalSupply / _totalAssets;
}
```

---

## Struct Packing

### Good

```solidity
struct UserInfo {
    uint128 balance;      // slot 0 (lower)
    uint128 rewardDebt;   // slot 0 (upper)
    uint64 lastUpdate;    // slot 1 (lower)
    uint64 nonce;         // slot 1 (upper)
}
```

### Poor

```solidity
struct UserInfo {
    uint256 balance;     // slot 0
    uint256 rewardDebt;  // slot 1 — packable into one slot
}
```

---

## Upgradeable Storage

### Good — append-only with gap

```solidity
abstract contract VaultStorage {
    uint256 internal _totalAssets;
    uint256[49] private __gap;
}

contract VaultV2 is VaultStorage {
    uint256 internal _feeBps;  // appended in new version
}
```

### Poor — insert breaks layout

```solidity
contract VaultV2 is VaultStorage {
    uint256 internal _feeBps;  // inserted before _totalAssets → corrupts storage
}
```

---

## Proxy Initialization (SCSFG deployment)

### Good — init in deploy script, disable on implementation

```solidity
// Implementation.sol
contract VaultImpl is Initializable, UUPSUpgradeable {
    /// @custom:oz-upgrades-unsafe-allow constructor
    constructor() {
        _disableInitializers();
    }

    function initialize(address admin, IERC20 asset_) external initializer {
        __UUPSUpgradeable_init();
        _grantRole(DEFAULT_ADMIN_ROLE, admin);
        _asset = asset_;
    }
}
```

```solidity
// Deploy.s.sol — same transaction
function run() external {
    address impl = address(new VaultImpl());
    ERC1967Proxy proxy = new ERC1967Proxy(impl, abi.encodeCall(VaultImpl.initialize, (admin, asset)));
}
```

### Poor — uninitialized implementation

```solidity
contract VaultImpl is Initializable {
    // no constructor with _disableInitializers()
    // attacker can initialize implementation directly
}
```

---

## Invariant Test with Handler (Foundry)

```solidity
// test/invariant/VaultHandler.sol
contract VaultHandler {
    Vault public vault;
    IERC20 public asset;
    uint256 public ghost_depositSum;
    uint256 public ghost_withdrawSum;

    constructor(Vault vault_, IERC20 asset_) {
        vault = vault_;
        asset = asset_;
    }

    function deposit(uint256 assets, uint256 actorSeed) external {
        address actor = address(uint160(bound(actorSeed, 1, type(uint160).max)));
        assets = bound(assets, 1, 1e24);
        deal(address(asset), actor, assets);
        vm.startPrank(actor);
        asset.approve(address(vault), assets);
        vault.deposit(assets, actor);
        vm.stopPrank();
        ghost_depositSum += assets;
    }

    function withdraw(uint256 shares, uint256 actorSeed) external {
        address actor = address(uint160(bound(actorSeed, 1, type(uint160).max)));
        uint256 balance = vault.balanceOf(actor);
        shares = bound(shares, 0, balance);
        if (shares == 0) return;
        uint256 assetsBefore = vault.totalAssets();
        vm.prank(actor);
        vault.redeem(shares, actor, actor);
        ghost_withdrawSum += assetsBefore - vault.totalAssets();
    }
}

// test/invariant/VaultInvariant.t.sol
contract VaultInvariantTest is Test {
    Vault public vault;
    VaultHandler public handler;

    function setUp() public {
        IERC20 asset = IERC20(address(new MockERC20()));
        vault = new Vault(asset);
        handler = new VaultHandler(vault, asset);
        targetContract(address(handler));
    }

    function invariant_solvency() public view {
        assertGe(vault.totalAssets(), vault.totalLiabilities());
    }

    function invariant_ghostAccounting() public view {
        assertEq(
            vault.totalAssets(),
            handler.ghost_depositSum() - handler.ghost_withdrawSum()
        );
    }
}
```

---

## FREI-PI Test Pattern

```solidity
function test_deposit_preservesSolvencyInvariant() public {
    // F: deposit
    // R: precondition
  uint256 assets = 100e18;
    deal(address(token), alice, assets);

    // E + I: effect then interaction (inside deposit)
    vm.startPrank(alice);
    token.approve(address(vault), assets);
    vault.deposit(assets, alice);
    vm.stopPrank();

    // P: protocol invariant
    assertGe(vault.totalAssets(), vault.totalLiabilities());
    assertEq(vault.totalAssets(), assets);
}
```

---

## TDD Workflow Example

```solidity
// Step 1: write failing test first
function test_mint_revertsOnZeroAmount() public {
    vm.expectRevert(ZeroAmount.selector);
    vault.deposit(0, alice);
}

// Step 2: implement minimal code
function deposit(uint256 assets, address receiver) external returns (uint256) {
    if (assets == 0) revert ZeroAmount();
    // ...
}

// Step 3: refactor, add next test
function test_mint_revertsOnZeroAddress() public {
    vm.expectRevert(ZeroAddress.selector);
    vault.deposit(100e18, address(0));
}
```

---

## Test Directory Structure (SCSFG)

```
test/
├── unit/
│   ├── Vault.t.sol
│   └── VaultMath.t.sol
├── integration/
│   ├── VaultWithOracle.t.sol
│   └── VaultWithGovernance.t.sol
├── e2e/
│   └── UserJourney.t.sol
├── invariant/
│   ├── VaultHandler.sol
│   └── VaultInvariant.t.sol
└── utils/
    ├── BaseTest.sol
    └── MockTokens.sol
```

---

## Mainnet Beta Sunset (SCSFG deployment)

```solidity
uint256 public immutable sunsetBlock;

constructor(uint256 blocksUntilSunset) {
    sunsetBlock = block.number + blocksUntilSunset;
}

modifier beforeSunset() {
    if (block.number > sunsetBlock) revert BetaEnded();
    _;
}

function deposit(uint256 assets, address receiver) external beforeSunset returns (uint256) {
    // new deposits blocked after sunset
}

function withdraw(uint256 shares, address receiver, address owner) external returns (uint256) {
    // withdrawals always allowed — users can exit
}
```

---

## AI-Generated Code — First-Pass Review Checklist

When reviewing AI output, verify in this order:

```
1. [ ] Compiles with project's exact pragma pin
2. [ ] Imports resolve to real packages (not hallucinated paths)
3. [ ] Visibility explicit on every function and state variable
4. [ ] Constructor or initializer correct for proxy pattern
5. [ ] CEI order in state-changing functions
6. [ ] payable only where ETH is intentionally accepted
7. [ ] Access control on every mutating function
8. [ ] Events on state changes
9. [ ] Internal functions validate inputs (not just public shell)
10. [ ] NatSpec present on external/public API
```

---

## Dependency Pinning

### Good — foundry.toml

```toml
[profile.default]
solc_version = "0.8.28"
evm_version = "cancun"

[dependencies]
"@openzeppelin-contracts" = "5.0.2"
"forge-std" = "1.9.4"
```

### Poor — floating versions

```toml
solc_version = "0.8"  # which patch?
# package.json: "@openzeppelin/contracts": "^5.0.0"  # breaking changes possible
```
