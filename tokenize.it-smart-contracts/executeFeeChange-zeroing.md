# `executeFeeChange()` Can Be Called Without Prior `planFeeChange()`, Zeroing All Protocol Fees

**Target:** https://github.com/corpus-io/tokenize.it-smart-contracts/tree/11fbe80
**Type:** Smart Contract
**Severity:** Low / Informational

---

## Vulnerability Details

### Root Cause

`executeFeeChange()` has no guard requiring that `planFeeChange()` was called first.
Its only precondition is:

```solidity
/**
 * @notice Executes a fee change that has been planned before
 */
function executeFeeChange() external onlyOwner {
    require(
        block.timestamp >= proposedDefaultFees.validityDate,
        "Fee change must be executed after the change time"
    );
    // block.timestamp (e.g. 1_700_000_000) >= proposedDefaultFees.validityDate (0) → PASSES
    fees[address(0)] = proposedDefaultFees; // copies {0, 0, 0, 0} into live fees
    emit SetFee(
        proposedDefaultFees.tokenFeeNumerator,
        proposedDefaultFees.crowdinvestingFeeNumerator,
        proposedDefaultFees.privateOfferFeeNumerator
    );
    delete proposedDefaultFees;
}
```

`proposedDefaultFees` is a storage-slot `Fees` struct. It holds all-zero values in
two situations:

1. **On fresh deployment** — before any `planFeeChange()` is ever called, Solidity
   zero-initialises every storage slot, so `proposedDefaultFees.validityDate = 0`.
2. **After any prior `executeFeeChange()`** — `delete proposedDefaultFees` on the
   last line resets the struct back to all zeros.

Because `block.timestamp` is always `> 0`, the `require` passes unconditionally in
both cases, writing `{0, 0, 0, 0}` into `fees[address(0)]`.

---

### Impact

| | |
|---|---|
| **Who can trigger it** | `owner` of the `FeeSettings` clone only |
| **Effect** | All three fee types (token mint, crowdinvesting, private offer) set to 0% |
| **Duration** | Minimum 12 weeks to restore — any fee increase from 0 triggers the timelock |
| **User funds** | Unaffected |
| **Protocol revenue** | Lost for all trades executed during the zeroed period |

---

## Validation Steps

Add the following test to `test/FeeSettings.t.sol` and run with:

```
forge test --match-test testExecuteFeeChangeWithoutPlan -vvvv
```

```solidity
function testExecuteFeeChangeWithoutPlan() public {
    // 1. Verify initial fees are non-zero (set in setUp)
    (uint32 liveToken, uint32 liveCrowdinvesting, uint32 livePrivate, ) = feeSettings.fees(address(0));
    assertGt(liveToken,          0, "token fee should be non-zero");
    assertGt(liveCrowdinvesting, 0, "crowdinvesting fee should be non-zero");
    assertGt(livePrivate,        0, "private offer fee should be non-zero");

    // 2. Verify no planFeeChange() was called — proposedDefaultFees is zero
    (uint32 propToken, uint32 propCrowdinvesting, uint32 propPrivate, uint64 propValidity) =
        feeSettings.proposedDefaultFees();
    assertEq(propToken,          0, "no plan should exist");
    assertEq(propCrowdinvesting, 0, "no plan should exist");
    assertEq(propPrivate,        0, "no plan should exist");
    assertEq(propValidity,       0, "no plan should exist");

    // 3. Owner calls executeFeeChange() with NO prior planFeeChange()
    vm.prank(admin);
    feeSettings.executeFeeChange(); // must NOT revert

    // 4. All fees are now zero
    (uint32 afterToken, uint32 afterCrowdinvesting, uint32 afterPrivate, ) = feeSettings.fees(address(0));
    assertEq(afterToken,          0, "token fee wiped");
    assertEq(afterCrowdinvesting, 0, "crowdinvesting fee wiped");
    assertEq(afterPrivate,        0, "private offer fee wiped");

    // 5. Restoration requires 12-week timelock
    Fees memory restoredFees = Fees(100, 100, 100, uint64(block.timestamp + 12 weeks - 1));

    vm.prank(admin);
    vm.expectRevert("Fee change must be at least 12 weeks in the future");
    feeSettings.planFeeChange(restoredFees); // reverts — cannot restore before 12 weeks

    // Correctly scheduled restoration (just over 12 weeks)
    restoredFees.validityDate = uint64(block.timestamp + 12 weeks + 1);
    vm.prank(admin);
    feeSettings.planFeeChange(restoredFees); // succeeds

    // 6. Fees remain zero until the timelock elapses
    (, uint32 finalCrowdinvesting, , ) = feeSettings.fees(address(0));
    assertEq(finalCrowdinvesting, 0, "fees still zero until timelock elapses");
}
```
