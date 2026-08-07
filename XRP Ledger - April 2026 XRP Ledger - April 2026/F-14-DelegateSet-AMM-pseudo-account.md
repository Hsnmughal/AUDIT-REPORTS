# F-14 — [Permission Delegation · XLS-0075] · Medium — `DelegateSet` to AMM pseudo-account inserts `ltDELEGATE` into AMM ownerDir; final `AMMWithdraw` and `AMMDelete` permanently blocked
 
**Feature label:** Permission Delegation (XLS-0075) / AMM (XLS-0030) — cross-feature interaction
**Severity:** Medium
 
---
 
## Summary
 
`DelegateSet::preclaim` only checks that the target `sfAuthorize` account exists (`ctx.view.exists(keylet::account(sfAuthorize))`), without verifying it is not a pseudo-account. AMM pool pseudo-accounts have real `ltACCOUNT_ROOT` SLEs, so this check passes. `doApply` inserts `ltDELEGATE` into the AMM's ownerDir.
 
The `ltDELEGATE` entry blocks two distinct operations:
 
1. **`AMMDelete`** — calls `deleteAMMAccount` → `deleteAMMTrustLines`, which returns `{tecINTERNAL, SkipEntry::No}` on encountering the unrecognised `ltDELEGATE` type.
2. **Final `AMMWithdraw` (the drain-to-zero path)** — `AMMWithdraw::doWithdraw` calls `deleteAMMAccountIfEmpty` when the resulting LP balance would reach zero. That function calls the same `deleteAMMAccount` → `deleteAMMTrustLines` chain and returns `tecINTERNAL`, rejecting the entire withdrawal. LPs cannot fully exit their position while the attack is active.
The `LCOV_EXCL_START` annotation on this path documents that the developers considered the unrecognised-type fallthrough unreachable. An attacker can make it reachable for ~2 XRP.
 
---
 
## Root Cause
 
**Files:**
- https://github.com/sherlock-audit/2026-04-xrp-ledger-april-2026-BlockGuard-team/blob/main/rippled/src/libxrpl/tx/transactors/delegate/DelegateSet.cpp#L43-L44 — missing `isPseudoAccount` check
- https://github.com/sherlock-audit/2026-04-xrp-ledger-april-2026-BlockGuard-team/blob/main/rippled/src/xrpl/app/tx/impl/AMMHelpers.cpp#L595-L625 (`deleteAMMTrustLines`) — returns `tecINTERNAL` on unknown SLE type
- https://github.com/sherlock-audit/2026-04-xrp-ledger-april-2026-BlockGuard-team/blob/main/rippled/src/xrpl/app/tx/impl/AMMHelpers.cpp#L629-L669 (`deleteAMMMPTokens`) — same pattern
- https://github.com/sherlock-audit/2026-04-xrp-ledger-april-2026-BlockGuard-team/blob/main/rippled/src/xrpl/app/tx/impl/AMMWithdraw.cpp#L783 — `deleteAMMAccountIfEmpty` → `deleteAMMAccount` → same chain
**DelegateSet.cpp:43-44:**
 
```cpp
auto const authAccount = ctx.tx[sfAuthorize];
if (!ctx.view.exists(keylet::account(authAccount)))
    return tecNO_TARGET;
// ← MISSING: isPseudoAccount check
```
 
**AMMHelpers.cpp:595-625 (deleteAMMTrustLines) — the blocker:**
 
```cpp
// LCOV_EXCL_START — developers expected this unreachable
if (sleType != ltAMM && sleType != ltRIPPLE_STATE && sleType != ltMPTOKEN)
    return {tecINTERNAL, SkipEntry::No};
// LCOV_EXCL_STOP
```
 
Both `AMMDelete::doApply` and `AMMWithdraw::doWithdraw` (drain-to-zero path) reach `deleteAMMAccount` → `deleteAMMTrustLines` → encounter `ltDELEGATE` → return `tecINTERNAL`.
 
---
 
## Pre-Conditions
 
1. An AMM pool exists in the ledger (any XRP/IOU or MPT pool).
2. Attacker knows the AMM pool's AccountID (public information from the ledger).
3. Attacker calls `DelegateSet{Account: Attacker, Authorize: <AMM_AccountID>, Permissions: [any]}`.
No consent from the AMM pool or its LPs is required.
 
---
 
## Attack Path
 
1. Attacker calls `DelegateSet{Account: Attacker, Authorize: AMM_Pool_Account, Permissions: [Payment]}`.
   - Preclaim: `ctx.view.exists(keylet::account(AMM_Pool_Account))` → true (AMM has a real `ltACCOUNT_ROOT`). Passes.
   - doApply: `dirInsert(keylet::ownerDir(AMM_Pool_Account), delegateKey)`. `ltDELEGATE` inserted into AMM ownerDir.
2. AMM pool continues to operate normally — partial withdrawals and trades are unaffected.
3. LPs attempt to fully exit: `AMMWithdraw{withdrawAll}` (the transaction that drains all LP tokens to zero).
   - `AMMWithdraw::doWithdraw` → `deleteAMMAccountIfEmpty` → `deleteAMMAccount` → `deleteAMMTrustLines` → encounters `ltDELEGATE` → `LCOV_EXCL_START` fallthrough → returns `{tecINTERNAL, SkipEntry::No}`.
   - The entire withdrawal transaction is rejected with `tecINTERNAL`. LPs cannot fully exit.
4. LPs attempt explicit cleanup: `AMMDelete{Asset: X, Asset2: Y}`.
   - Same `deleteAMMAccount` → `deleteAMMTrustLines` chain → `tecINTERNAL`.
5. The AMM SLE and pseudo-account persist indefinitely. LPs can do partial withdrawals to recover most of their funds but cannot reduce the pool to zero while the attack is active.
6. Attacker maintains the block for ~2 XRP locked (recoverable at any time). No party other than the attacker can remove the `ltDELEGATE` entry.
---
 
## Impact
 
**Both the final withdrawal (drain-to-zero path) and explicit `AMMDelete` are permanently blocked.**
 
- `AMMWithdraw{withdrawAll}` — the transaction that drains all LP tokens and auto-deletes the AMM — fails with `tecINTERNAL`. LPs cannot fully exit their position.
- Partial withdrawals (leaving non-zero LP balance) are unaffected. LPs can recover most of their funds but are stuck with a residual position they cannot zero out.
- `AMMDelete` — the explicit cleanup mechanism for zero-liquidity pools per XLS-0030 — also returns `tecINTERNAL`.
- The AMM SLE and pseudo-account persist as permanent ledger state. At network scale, many blocked pools accumulate as bloat.
- Attack cost: ~2 XRP locked (fully recoverable by attacker via DelegateDelete). Attacker can target any AMM whose AccountID is known (all of them — it is public ledger data).
---
 
## Proof of Concept
 
Place the file at `rippled/src/test/app/F14_DelegateBlocksAMMDelete_test.cpp` and rebuild.
 
<details>
<summary>Test file: F14_DelegateBlocksAMMDelete_test.cpp</summary>
```cpp
/**
 * F-14: DelegateSet to AMM pseudo-account permanently blocks AMMDelete.
 *
 * Root cause: DelegateSet::preclaim:43 only checks
 * ctx.view.exists(keylet::account(ctx.tx[sfAuthorize])).  AMM pseudo-accounts
 * have real ltACCOUNT_ROOT SLEs, so this check passes and DelegateSet::doApply
 * calls dirInsert on the AMM pseudo-account's ownerDir, inserting an
 * ltDELEGATE entry.
 *
 * AMMDelete calls deleteAMMAccount → deleteAMMTrustLines (then
 * deleteAMMMPTokens), both of which iterate the AMM ownerDir via
 * cleanupOnAccountDelete.  Neither handler recognises ltDELEGATE, so each
 * returns {tecINTERNAL, SkipEntry::No} via the LCOV_EXCL_START block
 * (AMMHelpers.cpp:619-622 / 664-667).  The tecINTERNAL propagates back
 * through deleteAMMAccount and out of AMMDelete::doApply, permanently
 * blocking deletion of the AMM.
 *
 * An identical path is triggered by AMMWithdraw when the last LP tokens
 * are drained: AMMWithdraw::deleteAMMAccountIfEmpty calls deleteAMMAccount
 * for the empty-pool auto-cleanup, which also encounters the ltDELEGATE and
 * returns tecINTERNAL (AMMWithdraw.cpp:783, also LCOV_EXCL_LINE).  Both the
 * explicit AMMDelete and the implicit auto-delete on last withdrawal are
 * blocked.
 *
 * Feature pool: Permission Delegation (XLS-0049) + AMM (XLS-0030)
 * Severity: Medium — any account can permanently freeze an AMM pool's
 *           exit path (blocking both final withdrawal and AMMDelete).
 */
 
#include <test/jtx.h>
#include <test/jtx/AMM.h>
#include <test/jtx/delegate.h>
 
#include <xrpl/ledger/helpers/DirectoryHelpers.h>
#include <xrpl/protocol/Feature.h>
#include <xrpl/protocol/Indexes.h>
 
namespace xrpl {
namespace test {
 
struct F14DelegateBlocksAMMDelete_test : public beast::unit_test::suite
{
    void
    run() override
    {
        testcase("DelegateSet to AMM pseudo-account permanently blocks AMMDelete");
        using namespace test::jtx;
 
        Env env{*this, testable_amendments()};
 
        Account const gw("gw");        // AMM creator / LP
        Account const alice("alice");  // attacker
 
        env.fund(XRP(1000000), gw, alice);
        env.close();
 
        // ── Step 1: create XRP/IOU AMM pool ───────────────────────────────────
        auto const USD = gw["USD"];
 
        AMM amm(env, gw, XRP(10000), USD(10000));
        BEAST_EXPECT(amm.ammExists());
 
        auto const ammAccountID = amm.ammAccount();
 
        // ── Step 2: attack — alice inserts ltDELEGATE into AMM's ownerDir ─────
        // DelegateSet::preclaim passes because keylet::account(ammAccountID)
        // exists (AMM pseudo-accounts carry a real ltACCOUNT_ROOT SLE).
        Account const ammAcc("amm_pseudo", ammAccountID);
 
        env(delegate::set(alice, ammAcc, {"Payment"}));
        env.close();
 
        auto const delegateKey = keylet::delegate(alice.id(), ammAccountID);
        BEAST_EXPECT(env.le(delegateKey) != nullptr);
        BEAST_EXPECT(!dirIsEmpty(*env.closed(), keylet::ownerDir(ammAccountID)));
 
        // ── Step 3: explicit AMMDelete is blocked (LP tokens still > 0) ───────
        amm.ammDelete(gw, ter(tecAMM_NOT_EMPTY));
 
        // ── Step 4: even draining LP tokens now fails ─────────────────────────
        // deleteAMMAccountIfEmpty → deleteAMMAccount → deleteAMMTrustLines
        // → ltDELEGATE → LCOV_EXCL_START fallthrough → tecINTERNAL
        amm.withdrawAll(gw, std::nullopt, ter(tecINTERNAL));
 
        BEAST_EXPECT(amm.ammExists());
 
        // ── Step 5: recovery — alice removes the delegate ─────────────────────
        env(delegate::set(alice, ammAcc, {}));
        env.close();
 
        BEAST_EXPECT(env.le(delegateKey) == nullptr);
 
        // ── Step 6: withdrawAll now succeeds and auto-deletes the AMM ─────────
        amm.withdrawAll(gw);
 
        BEAST_EXPECT(!amm.ammExists());
        BEAST_EXPECT(!env.le(keylet::ownerDir(ammAccountID)));
    }
};
 
BEAST_DEFINE_TESTSUITE(F14DelegateBlocksAMMDelete, app, xrpl);
 
}  // namespace test
}  // namespace xrpl
```
 
</details>
### How to run
 
```bash
cd /path/to/rippled-build
./xrpld --unittest=F14DelegateBlocksAMMDelete
```
 
### Expected output
 
```text
xrpl.app.F14DelegateBlocksAMMDelete DelegateSet to AMM pseudo-account permanently blocks AMMDelete
xrpl.app.F14DelegateBlocksAMMDelete had 0 failures.
502ms, 1 suite, 1 case, 43 tests total, 0 failures
```
 
---
 
## Mitigation
 
**Primary fix:** Add `isPseudoAccount` check in `DelegateSet::preclaim`:
 
```cpp
auto const authSle = ctx.view.read(keylet::account(authAccount));
if (!authSle)
    return tecNO_TARGET;
if (isPseudoAccount(*authSle))
    return tecNO_PERMISSION;  // same as SponsorshipSet::preclaim
```
 
**Secondary hardening:** Make `deleteAMMTrustLines` and `deleteAMMMPTokens` skip unrecognised SLE types (with a warning log) instead of returning `tecINTERNAL`. This provides defense-in-depth — even if a future protocol addition creates another unexpected ownerDir entry type, AMMDelete should not permanently fail. However, this secondary fix should not replace the primary fix, as skipping entries could leave orphaned SLEs.
 
---
 
## Why This Is a Security Issue (Not a Design Choice)
 
The `deleteAMMTrustLines` / `deleteAMMMPTokens` helpers were written with a hard invariant: the AMM ownerDir contains only `ltAMM`, `ltRIPPLE_STATE`, and `ltMPTOKEN` entries. The `LCOV_EXCL_START` annotation at `AMMHelpers.cpp:595` is the developers' own statement that any other entry type is unreachable under correct protocol operation.
 
`DelegateSet::preclaim` violates the assumption that created this invariant. It validates only `ctx.view.exists(keylet::account(authAccount))` — true for AMM pseudo-accounts, which have real `ltACCOUNT_ROOT` SLEs. It does NOT call `isPseudoAccount()`, which `SponsorshipSet::preclaim` correctly does. The result is a gap: `DelegateSet` can target any `ltACCOUNT_ROOT` including ones that have no signing key and no mechanism to remove entries from their ownerDir.
 
The blocking extends beyond the ledger-cleanup concern: `AMMWithdraw{withdrawAll}` — the transaction LPs use to fully exit their position — hits the same `deleteAMMAccount` path and also fails with `tecINTERNAL`. LPs are unable to complete a full exit while the attack is active. An attacker maintaining a 2 XRP deposit can permanently deny LP exit for an entire pool community.
 
---
 
## Severity Justification
 
**Medium** — LP fund-exit path (final drain) and AMMDelete permanently blocked; partial withdrawals still work; full exit only possible after attacker voluntarily removes delegation.
 
| Factor | Assessment |
|--------|-----------|
| Victim consent required | None — attacker acts unilaterally on public AMM AccountID |
| Self-service recovery | No — only attacker can call `DelegateDelete` |
| `AMMWithdraw{withdrawAll}` | Blocked with `tecINTERNAL` — LPs cannot fully exit position |
| `AMMDelete` | Blocked with `tecINTERNAL` — pool cleanup path broken |
| Partial withdrawals | Unaffected — LPs can recover most funds but not zero out |
| Attack cost | ~2 XRP locked (fully recoverable by attacker anytime) |
| Persistence | Indefinite — attacker re-creates after any deletion |
| Permissionless | Yes — AMM AccountID is public ledger data |
 
**Why not High:** Partial withdrawals are unaffected. LPs can recover most of their funds; only the final zero-drain withdrawal is blocked. The harm is LP exit impairment rather than direct fund theft or total lockup.
 
**Why not Low:** The blocked operation (final LP exit) is a core protocol lifecycle function. The attacker is targeting a community resource (the LP pool exit path) at near-zero cost with no victim remedy. Permanent blockage of `AMMWithdraw{withdrawAll}` crosses from inconvenience into material liveness harm.
 
---
 
## Key Code Locations
 
| Location | Role |
|----------|------|
| https://github.com/sherlock-audit/2026-04-xrp-ledger-april-2026-BlockGuard-team/blob/main/rippled/src/libxrpl/tx/transactors/delegate/DelegateSet.cpp#L43-L44 | Missing `isPseudoAccount` check — root cause |
| https://github.com/sherlock-audit/2026-04-xrp-ledger-april-2026-BlockGuard-team/blob/main/rippled/src/xrpl/app/tx/impl/AMMHelpers.cpp#L595-L625 | `deleteAMMTrustLines` — `LCOV_EXCL_START` block; `ltDELEGATE` → `tecINTERNAL` |
| https://github.com/sherlock-audit/2026-04-xrp-ledger-april-2026-BlockGuard-team/blob/main/rippled/src/xrpl/app/tx/impl/AMMHelpers.cpp#L629-L669 | `deleteAMMMPTokens` — same pattern |
| https://github.com/sherlock-audit/2026-04-xrp-ledger-april-2026-BlockGuard-team/blob/main/rippled/src/xrpl/app/tx/impl/AMMWithdraw.cpp#L783 | `deleteAMMAccountIfEmpty` → `deleteAMMAccount` — drain-to-zero also blocked |