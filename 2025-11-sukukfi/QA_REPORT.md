# QA Review Report

## Low-Severity Findings

### L-01: Missing Indexed Parameters in Critical Events

**Files & Lines:**
- `src/WERC7575ShareToken.sol:147` - `RBalanceAdjusted` event
- `src/WERC7575ShareToken.sol:148` - `RBalanceAdjustmentCancelled` event
- `src/WERC7575ShareToken.sol:150` - `KYCStatusChanged` event
- `src/ERC7575VaultUpgradeable.sol:2167-2168` - Cancelation events

**Description:**

Multiple important events lack indexed parameters for amounts and timestamps:

```solidity
event RBalanceAdjusted(address indexed account, uint256 amountInvested, uint256 amountReceived);
event KYCStatusChanged(address indexed user, address indexed kycAdmin, bool indexed isVerified, uint256 timestamp);
event DepositRequestCancelled(address indexed controller, uint256 assets);
```

**Why it matters:**

Off-chain monitoring systems and indexers cannot efficiently filter events by amount or timestamp. This impacts user interfaces, analytics dashboards, and automated monitoring systems that need to track specific value ranges or time periods.

**Suggested fix:**

Add indexed modifier to numeric parameters:
```solidity
event RBalanceAdjusted(address indexed account, uint256 indexed amountInvested, uint256 amountReceived);
event KYCStatusChanged(address indexed user, address indexed kycAdmin, bool indexed isVerified, uint256 indexed timestamp);
```

Note: Solidity allows max 3 indexed parameters per event. Choose the most commonly filtered parameters.

---

### L-02: Magic Numbers Without Named Constants

**File & Line:** `src/ERC7575VaultUpgradeable.sol:189`

**Description:**

Hardcoded value `1000` used for minimum deposit without explanation:

```solidity
$.minimumDepositAmount = 1000;
```

**Why it matters:**

The value `1000` could represent 1000 units in 6-decimal (USDC = 0.001 USDC) or 18-decimal tokens. Without a named constant, the intent is unclear and mistakes could occur during maintenance.

**Suggested fix:**

```solidity
uint16 private constant DEFAULT_MINIMUM_DEPOSIT = 1000; // 1000 units in asset decimals
// In initialize():
$.minimumDepositAmount = DEFAULT_MINIMUM_DEPOSIT;
```

---

### L-03: Misleading Variable Naming in rBalance Storage

**File & Line:** `src/WERC7575ShareToken.sol:127`

**Description:**

rBalance adjustments stored as fixed array with non-descriptive indices:

```solidity
mapping(address => uint256[2]) private _rBalanceAdjustments;
// Used as: _rBalanceAdjustments[account][0] = amounti
//          _rBalanceAdjustments[account][1] = amountr
```

**Why it matters:**

Array indices `[0]` and `[1]` are not self-documenting. Risk of accidentally swapping indices during maintenance or refactoring, leading to incorrect accounting of invested vs received amounts.

**Suggested fix:**

Use a struct for clarity:
```solidity
struct RBalanceAdjustment {
    uint256 investedAmount;
    uint256 receivedAmount;
}
mapping(address => RBalanceAdjustment) private _rBalanceAdjustments;
```

---

### L-04: Inconsistent Use of Custom Errors

**File & Line:** `src/WERC7575ShareToken.sol:270, 278`

**Description:**

String literal reverts used instead of custom errors:

```solidity
revert("ShareToken: cannot verify vault has no outstanding assets");
revert("ShareToken: cannot verify vault asset balance");
```

**Why it matters:**

String reverts cost more gas than custom errors and are inconsistent with the rest of the codebase which uses custom errors. This creates maintenance inconsistency and higher gas costs.

**Suggested fix:**

Define and use custom errors:
```solidity
error CannotVerifyVaultAssets();
error CannotVerifyVaultBalance();

// Usage:
if (...) revert CannotVerifyVaultAssets();
if (...) revert CannotVerifyVaultBalance();
```

---

### L-05: Precision Loss in Small Amount Claims

**File & Lines:**
- `src/ERC7575VaultUpgradeable.sol:570, 646, 897, 939`

**Description:**

Consistent use of `Math.Rounding.Floor` throughout share-asset conversions:

```solidity
shares = Math.mulDiv(assets, supply + VIRTUAL_SHARES, totalAssets + VIRTUAL_ASSETS, Math.Rounding.Floor);
```

**Why it matters:**

Floor rounding systematically favors the vault, causing users to lose dust amounts (<1 wei) on claims. While by design and non-exploitable, users claiming very small amounts might receive 0 assets due to rounding down.

**Suggested fix:**

Document this behavior clearly in user-facing documentation. Consider implementing minimum claim amounts to prevent dust claims that round to zero.

---

### L-06: Commented Out Import Statement

**File & Line:** `src/WERC7575ShareToken.sol:5`

**Description:**

```solidity
// import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
```

**Why it matters:**

Dead code should be removed to avoid confusion. Developers might wonder if this import is needed or was intentionally disabled for a reason.

**Suggested fix:**

Remove the commented line entirely or add a comment explaining why it's commented if there's a specific reason.

---

### L-07: Abbreviated Variable Names Reduce Readability

**File & Lines:**
- `src/WERC7575ShareToken.sol:1436` - `ts` parameter
- `src/WERC7575ShareToken.sol:1439` - `amounti`, `amountr` variables

**Description:**

```solidity
function adjustrBalance(address account, uint256 ts, uint256 amounti, uint256 amountr) external onlyRevenueAdmin {
```

**Why it matters:**

Abbreviated names like `ts`, `amounti`, `amountr` make code harder to read and maintain.

**Suggested fix:**

Use descriptive names:
```solidity
function adjustrBalance(
    address account,
    uint256 timestamp,
    uint256 investedAmount,
    uint256 receivedAmount
) external onlyRevenueAdmin {
```

---

## Governance & Centralization Risks

### G-01: Centralized KYC Admin Can Censor Users

**Description:**

KYC admin (`setKycVerified()`) has unilateral power to revoke KYC for any address. All transfers require recipient KYC, so revoked users are immediately locked out of their funds.

**Risk:**

Single point of failure with no appeal mechanism or timelock. Users can be censored instantly.

**Recommendation:**

Multi-sig for KYC revocation and time-delayed changes.

---

### G-02: Validator Controls All Batch Transfers

**Description:**

Validator can execute `batchTransfers()` and `rBatchTransfers()` for any users, controlling when transfers execute and selectively updating rBalances.

**Risk:**

Compromised validator key can manipulate balances. No rate limiting or multi-sig required.

**Recommendation:**

Require multi-sig for validator role and add rate limits.

---

### G-03: Immediate Contract Upgrades Without Timelock

**Description:**

UUPS pattern allows owner to upgrade contracts instantly via `upgradeTo()` and `upgradeToAndCall()`.

**Risk:**

No review period for users. Owner can upgrade to malicious implementation affecting all users immediately.

**Recommendation:**

Add mandatory timelock before upgrades activate.

---

### G-04: Investment Manager Has Unlimited Power Over Requests

**Description:**

Investment manager controls fulfillment timing for all deposit/redeem requests with no timeout mechanisms or forced deadlines.

**Risk:**

Can delay indefinitely, process selectively, or manipulate timing. Users have no recourse.

**Recommendation:**

Add maximum fulfillment timeout with automatic processing.

---

### G-05: No Multi-Sig Requirements for Critical Operations

**Description:**

All privileged roles (owner, validator, KYC admin, revenue admin, investment manager) require only single signature.

**Risk:**

Single point of failure. Compromised key gives total system control.

**Recommendation:**

Implement multi-sig for critical operations like upgrades, pause, and large batch transfers.

---

### G-06: Revenue Admin Controls Profit Distribution Without Verification

**Description:**

Revenue admin can adjust rBalance for any account via `adjustrBalance()` with no on-chain proof of actual returns.

**Risk:**

Can show false profits or hide losses. Users must trust admin's accounting.

**Recommendation:**

Add on-chain verification of returns or time-delayed adjustments with public review.

---

### G-07: Vault Registration Can Impact All Users

**Description:**

Owner can register/unregister vaults affecting how shares are valued for all users without individual consent.

**Risk:**

Changes affect entire system. Removing vault can strand funds.

**Recommendation:**

Add timelock before vault changes take effect.

---

## Code Quality Notes

### Documentation Issues

- Incomplete NatSpec in `DecimalConstants.sol` - no explanation of design choices
- Missing `@param` and `@return` tags in `SafeTokenTransfers.sol`
- Extremely long comment blocks (65-208 lines) in `WERC7575ShareToken.sol` hinder navigation
- Single character variable `$` throughout upgradeable contracts (OpenZeppelin pattern)

### Complex Functions

- `_computeRBalanceFlagsInternal` (160 lines, `WERC7575ShareToken.sol:802-963`) - nested loops with bit manipulation
- `consolidateTransfers` (`WERC7575ShareToken.sol:1006-1062`) - duplicates logic increasing maintenance risk
- `rBatchTransfers` (83 lines) - multiple responsibilities in one function

### Redundant Code

- Duplicate documentation in `ShareTokenUpgradeable.sol`
- Repeated comment patterns
- Functions reading storage twice (e.g., `setKycVerified`)

### Gas Inefficiencies

- String reverts instead of custom errors in vault unregistration
- Duplicate logic in `deposit()` and `mint()` functions
- Uncached storage reads in loops

### Style Inconsistencies

- Inconsistent `unchecked` block usage
- Variable event emission locations
- Mixed comment styles
- Trailing whitespace in NatSpec

### Bounded Gas DOS

- `MAX_VAULTS_PER_SHARE_TOKEN = 10` limits loops, but gas increases linearly with vault count
- Functions like `getCirculatingSupplyAndAssets` iterate all vaults

### Other

- Virtual shares/assets (`1e6`) choice not fully justified
- Dead imports commented out instead of removed
- Complex bit manipulation needs more inline comments
