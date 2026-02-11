# Missing Authorization Check in WERC7575Vault Allows Unauthorized Fund Withdrawal

## Severity
**CRITICAL/HIGH**

## Finding description and impact

The `_withdraw()` internal function in `WERC7575Vault.sol` fails to verify that `msg.sender` has authorization to withdraw funds on behalf of the `owner`. The function only checks if the owner has self-allowance set via `spendSelfAllowance(owner, shares)`, but never validates whether `msg.sender` is authorized to act on behalf of the owner.

### Root Cause

In `WERC7575Vault.sol` at lines 397-411, the `_withdraw()` function performs the following check:

```solidity
function _withdraw(uint256 assets, uint256 shares, address receiver, address owner) internal {
    if (receiver == address(0)) {
        revert IERC20Errors.ERC20InvalidReceiver(address(0));
    }
    if (owner == address(0)) {
        revert IERC20Errors.ERC20InvalidSender(address(0));
    }
    if (assets == 0) revert ZeroAssets();
    if (shares == 0) revert ZeroShares();

    _shareToken.spendSelfAllowance(owner, shares);  // Only checks allowance(owner, owner)
    _shareToken.burn(owner, shares);
    SafeTokenTransfers.safeTransfer(_asset, receiver, assets);
    emit Withdraw(msg.sender, receiver, owner, assets, shares);
}
```

**The Critical Missing Check:**

The function **never verifies that `msg.sender` has authorization** to withdraw on behalf of `owner`. It only calls `spendSelfAllowance(owner, shares)` which checks if the owner has self-allowance set, but doesn't validate if `msg.sender` (the actual caller) has any permission.

According to the ERC-4626 standard, when `msg.sender != owner`, the function must verify that `allowance(owner, msg.sender) >= shares` and spend from that allowance. This check is completely missing.

**Why the V12 Fix Doesn't Prevent This Attack:**

A related issue was found in V12 audit where `spendSelfAllowance()` incorrectly checks `allowance(owner, owner)` instead of `allowance(owner, vault)`:

```solidity
// Current implementation in WERC7575ShareToken.sol:660-662
function spendSelfAllowance(address owner, uint256 shares) external onlyVaults {
    _spendAllowance(owner, owner, shares);  // Checks owner→owner instead of owner→vault
}
```

**However, even if the V12 bug is fixed to properly check `allowance(owner, vault)`, this vulnerability would still exist** because:

1. The vault would check if the **vault itself** has permission from owner
2. But it still wouldn't check if **msg.sender** (the attacker) has permission to call the vault on behalf of owner
3. Any random attacker can call `withdraw(amount, attacker, victim)` and the vault would spend the vault's own allowance, not checking the attacker's authorization

The attack flow would be:
- Victim approves vault: `allowance(victim, vault) = MAX`
- Attacker (who has NO approval) calls: `vault.withdraw(amount, attacker, victim)`
- Vault checks its own allowance from victim ✓ (passes)
- Vault never checks if attacker has permission ✗ (missing)
- Attacker steals funds

**This is a separate, distinct vulnerability** that requires checking `msg.sender`'s authorization before allowing the withdrawal.

### Affected Functions

Both public functions are vulnerable as they call the same flawed internal function:
1. `withdraw(uint256 assets, address receiver, address owner)` - Line 434
2. `redeem(uint256 shares, address receiver, address owner)` - Line 464

### Impact

**Complete loss of user funds.** Any attacker can:
1. Identify victims who have shares or assets in the protocol.
2. Call `withdraw()` or `redeem()` with `owner` set to the victim's address and `receiver` set to his address
3. Drain the victim's entire balance while the victim's shares are burned

The vulnerability affects all users who have set self-allowance via the `permit()` function, which is a normal requirement for using the vault's withdrawal functionality.

### Proof of Concept

paste `testMaliciousWithdraw_StealFunds()` , `testMaliciousRedeem_StealFunds()` and `import {console} from "forge-std/console.sol";` in `test/AuditReproduction.t.sol` :

Run the tests:
```bash
forge test --match-test testMaliciousWithdraw_StealFunds -vvv
forge test --match-test testMaliciousRedeem_StealFunds -vvv
```

## Recommended mitigation steps

Add an authorization check in `_withdraw()` to verify that `msg.sender` has permission to withdraw on behalf of `owner`:

```diff
function _withdraw(uint256 assets, uint256 shares, address receiver, address owner) internal {
    if (receiver == address(0)) {
        revert IERC20Errors.ERC20InvalidReceiver(address(0));
    }
    if (owner == address(0)) {
        revert IERC20Errors.ERC20InvalidSender(address(0));
    }
    if (assets == 0) revert ZeroAssets();
    if (shares == 0) revert ZeroShares();

+   // Check authorization: msg.sender must be owner or have allowance from owner
+   if (msg.sender != owner) {
+       uint256 allowed = _shareToken.allowance(owner, msg.sender);
+       if (allowed < shares) revert InsufficientAllowance();
+   }

    _shareToken.spendSelfAllowance(owner, shares);
    _shareToken.burn(owner, shares);
    SafeTokenTransfers.safeTransfer(_asset, receiver, assets);
    emit Withdraw(msg.sender, receiver, owner, assets, shares);
}
```