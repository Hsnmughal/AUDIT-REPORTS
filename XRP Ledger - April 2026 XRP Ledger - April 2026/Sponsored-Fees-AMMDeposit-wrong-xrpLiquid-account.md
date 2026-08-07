# [Sponsored Fees · XLS-0068] · Low — `AMMDeposit::deposit()` checks sponsor's `xrpLiquid` instead of depositor's; reserve gate enforced against the wrong account
 
**Feature label:** Sponsored Fees and Reserves (XLS-0068) / AMM (XLS-0030) — cross-feature interaction
**Severity:** Low (spec/invariant break; functional inconsistency in sponsored path; no direct adversarial fund extraction)
 
---
 
## Summary
 
`AMMDeposit::deposit()` evaluates the XRP-liquidity gate against the **reserve sponsor's** AccountID rather than the **depositor's** AccountID whenever a reserve sponsorship is attached to the transaction.
 
The result is two distinct failure modes:
 
| Case | Depositor XRP | Sponsor XRP | Correct result | Bug result |
|------|--------------|-------------|----------------|------------|
| **Bypass** | insufficient | sufficient | `tecUNFUNDED_AMM` | `tesSUCCESS` — deposit succeeds when it must not |
| **False reject** | sufficient | insufficient | `tesSUCCESS` | `tecUNFUNDED_AMM` — deposit fails when it should succeed |
 
Neither case constitutes direct adversarial fund extraction: in the bypass, the reserve is correctly charged to the sponsor (who consented to sponsorship); in the false-reject, the depositor's funds are safe. The finding is a spec invariant break — the `xrpLiquid` check is applied to the wrong account — with no direct monetary loss path.
 
---
 
## Root Cause
 
**File:** https://github.com/sherlock-audit/2026-04-xrp-ledger-april-2026-BlockGuard-team/blob/main/rippled/src/libxrpl/tx/transactors/dex/AMMDeposit.cpp#L538-L540
 
```cpp
// AMMDeposit.cpp:524-540 — deposit() inner lambda
auto const sponsor = getTxReserveSponsorAccountID(ctx_.tx);
...
auto const ownerCountAdj = trustlineExists ? 0 : 1;
if (xrpLiquid(view, sponsor.value_or(account_), sponsor ? ownerCountAdj : 0, j_) >=
    depositAmount)
    return tesSUCCESS;
```
 
`sponsor.value_or(account_)` returns `sponsor` (the sponsor's AccountID) when `sponsor` has a value. The intent of the call is to verify that the **depositor** (`account_`) has sufficient liquid XRP for the deposit. The sponsor's role is to cover **reserve increments** (new owner-object slots); it does not and should not cover the deposit's XRP contribution.
 
The corresponding non-sponsored path — `xrpLiquid(view, accountID, 1, ctx.j)` in `AMMDeposit::preclaim` (https://github.com/sherlock-audit/2026-04-xrp-ledger-april-2026-BlockGuard-team/blob/main/rippled/src/libxrpl/tx/transactors/dex/AMMDeposit.cpp#L355) — correctly uses the depositor's AccountID. The `deposit()` helper diverges from this at line 538.
 
---
 
## Pre-Conditions
 
**Bypass case:**
1. An XRP/IOU AMM pool exists.
2. Alice has insufficient XRP to cover the LP-token trust-line reserve increment.
3. Bob has created `SponsorshipSet{Account: Bob, Sponsee: Alice, ReserveCount: ≥1}`.
4. Alice submits `AMMDeposit` with `sfSponsor: Bob, sfSponsorFlags: spfSponsorReserve`.
**False-reject case:**
1. Same pool. Alice has ample XRP.
2. Bob (sponsor) has near-zero liquid XRP.
3. Alice submits the same sponsored `AMMDeposit`.
---
 
## Attack Path
 
### Bypass variant
 
1. Alice's balance ≈ `accountReserve(1)` + 1 drop — cannot afford one more owner-object increment.
2. Bob's balance ≈ 1,000,000 XRP.
3. Without sponsorship: `AMMDeposit` preclaim calls `xrpLiquid(view, alice, 1, j)` → insufficient → `tecINSUF_RESERVE_LINE`.
4. With Bob as reserve sponsor: `deposit()` at line 538 calls `xrpLiquid(view, bob, 0, j)` → 1,000,000 XRP ≥ depositAmount → passes. Deposit succeeds.
5. The LP-token trust-line reserve increment is charged to Bob's ownerCount (as expected for a reserve sponsor). The `xrpLiquid` check against Bob's balance is the spec deviation — it should have fired against Alice's balance.
### False-reject variant
 
1. Alice's balance = 1,000,000 XRP — no reserve issue.
2. Bob's balance ≈ minimum reserve + 1 drop.
3. Alice submits sponsored `AMMDeposit{Amount: 10 XRP}`. `deposit()` line 538: `xrpLiquid(view, bob, 0, j)` = 1 drop < 10 XRP → `tecUNFUNDED_AMM`. Rejected despite Alice having ample XRP.
---
 
## Impact
 
**Functional inconsistency in the sponsored `AMMDeposit` path.** The `xrpLiquid` check is applied to the wrong account, producing incorrect outcomes in both directions. No funds are stolen: the bypass allows a deposit within the scope of a consented sponsorship relationship; the false-reject denies a legitimate deposit with no financial harm to Alice's balance.
 
This is a spec invariant break: `deposit()` must validate the depositor's liquidity, not the sponsor's. The sponsor covers reserve increments; the depositor is always responsible for covering the deposit amount itself.
 
---
 
## Proof of Concept
 
Place the file at `rippled/src/test/app/TN8_AMMDepositWrongXRPLiquid_test.cpp` and rebuild.
 
<details>
<summary>Test file: TN8_AMMDepositWrongXRPLiquid_test.cpp</summary>
```cpp
/**
 * TN-8: AMMDeposit::deposit() XRP balance check uses sponsor's xrpLiquid
 * instead of depositor's.
 *
 * Root cause: AMMDeposit.cpp:538:
 *   if (xrpLiquid(view, sponsor.value_or(account_), sponsor ? ownerCountAdj : 0, j_) >= ...)
 *
 * When a reserve sponsor is present, sponsor.value_or(account_) returns the
 * SPONSOR's AccountID, not the DEPOSITOR's.  The liquidity gate therefore
 * validates the wrong account:
 *
 *   Bypass case  — depositor has insufficient XRP but sponsor has plenty;
 *                  the check passes against the sponsor's balance, so a
 *                  deposit that should fail with tecUNFUNDED_AMM succeeds.
 *
 *   False-reject — depositor has sufficient XRP but sponsor has almost none;
 *                  the check fails against the sponsor's balance, so a
 *                  deposit that should succeed returns tecUNFUNDED_AMM.
 *
 * Feature pool:  Sponsored Fees (XLS-0068) + AMM (XLS-0030)
 * Severity:      Low — spec/invariant break; wrong account in xrpLiquid check;
 *                no direct adversarial fund extraction.
 */
 
#include <test/jtx.h>
#include <test/jtx/AMM.h>
#include <test/jtx/sponsor.h>
#include <test/jtx/trust.h>
 
#include <xrpl/protocol/Feature.h>
#include <xrpl/protocol/TxFlags.h>
 
namespace xrpl {
namespace test {
 
struct TN8AMMDepositWrongXRPLiquid_test : public beast::unit_test::suite
{
    // ── helpers ──────────────────────────────────────────────────────────────
 
    static void
    drainTo(jtx::Env& env, jtx::Account const& account, STAmount const target)
    {
        using namespace jtx;
        auto const current = env.balance(account);
        auto const baseFee = env.current()->fees().base;
        if (current <= target + baseFee)
            return;
        STAmount const surplus = current - target - baseFee;
        if (surplus <= beast::zero)
            return;
        env(pay(account, env.master, surplus));
        env.close();
    }
 
    void
    run() override
    {
        using namespace jtx;
 
        testcase("AMMDeposit checks sponsor xrpLiquid instead of depositor");
 
        Env env{*this, testable_amendments()};
 
        Account const gw("gw");       // AMM creator / IOU issuer
        Account const alice("alice"); // depositor — kept XRP-poor
        Account const bob("bob");     // reserve sponsor — XRP-rich
 
        env.fund(XRP(1'000'000), gw, alice, bob);
        env.close();
 
        // ── Step 1: trust-line for USD and initial AMM pool ──────────────────
        auto const USD = gw["USD"];
 
        env(trust(alice, USD(100'000)));
        env(trust(bob, USD(100'000)));
        env.close();
 
        env(pay(gw, alice, USD(10'000)));
        env(pay(gw, bob, USD(10'000)));
        env.close();
 
        AMM amm(env, gw, XRP(10'000), USD(10'000));
        BEAST_EXPECT(amm.ammExists());
        env.close();
 
        // ── Step 2: create the sponsorship (bob sponsors alice's reserves) ───
        env(sponsor::set_reserve(bob, 0, 10), sponsor::sponseeAcc(alice),
            ter(tesSUCCESS));
        env.close();
 
        auto const sponsorKeylet = keylet::sponsor(bob, alice);
        BEAST_EXPECT(env.le(sponsorKeylet) != nullptr);
 
        // ── Step 3: BYPASS CASE ──────────────────────────────────────────────
        // Drain alice to just above base reserve — she cannot cover a new
        // trust-line reserve increment.  bob keeps his large balance.
        // Correct behavior:  check alice's xrpLiquid → tecUNFUNDED_AMM.
        // Bug behavior:      check bob's  xrpLiquid → tesSUCCESS (bypass).
 
        auto const baseReserve      = env.current()->fees().accountReserve(0);
        auto const incrementReserve = env.current()->fees().increment;
        drainTo(env, alice, baseReserve + incrementReserve + drops(1));
 
        auto const aliceXRPBefore = env.balance(alice);
        auto const bobXRPBefore   = env.balance(bob);
 
        Json::Value jvDeposit;
        jvDeposit[jss::Account]         = alice.human();
        jvDeposit[jss::TransactionType] = jss::AMMDeposit;
        jvDeposit[jss::Asset]  = STIssue(sfAsset, XRP).getJson(JsonOptions::none);
        jvDeposit[jss::Asset2] = STIssue(sfAsset, USD).getJson(JsonOptions::none);
        jvDeposit[jss::Amount] = XRP(10).value().getJson(JsonOptions::none);
        jvDeposit[jss::Flags]  = tfSingleAsset;
 
        // Without sponsorship: alice cannot afford the LP-token trust-line reserve.
        env(jvDeposit, ter(tecINSUF_RESERVE_LINE));
        env.close();
 
        // With bob as reserve sponsor: deposit() checks bob's xrpLiquid → passes.
        env(jvDeposit,
            sponsor::as(bob, spfSponsorReserve),
            sig(sfSponsorSignature, bob),
            ter(tesSUCCESS));  // BUG: wrong account checked
        env.close();
 
        BEAST_EXPECT(env.balance(alice) <= aliceXRPBefore);
        BEAST_EXPECT(env.balance(bob)   <= bobXRPBefore);
 
        // ── Step 4: FALSE-REJECTION CASE ────────────────────────────────────
        Account const alice2("alice2");
        Account const bob2("bob2");
 
        env.fund(XRP(1'000'000), alice2, bob2);
        env.close();
 
        auto const EUR = gw["EUR"];
        env(trust(alice2, EUR(100'000)));
        env.close();
        env(pay(gw, alice2, EUR(10'000)));
        env.close();
 
        AMM amm2(env, gw, XRP(5'000), EUR(5'000));
        BEAST_EXPECT(amm2.ammExists());
        env.close();
 
        env(sponsor::set_reserve(bob2, 0, 10), sponsor::sponseeAcc(alice2),
            ter(tesSUCCESS));
        env.close();
 
        auto const sponsorKeylet2 = keylet::sponsor(bob2, alice2);
        BEAST_EXPECT(env.le(sponsorKeylet2) != nullptr);
 
        // Drain bob2 to near-zero.
        drainTo(env, bob2, baseReserve + incrementReserve + drops(1));
 
        Json::Value jvDeposit2;
        jvDeposit2[jss::Account]         = alice2.human();
        jvDeposit2[jss::TransactionType] = jss::AMMDeposit;
        jvDeposit2[jss::Asset]  = STIssue(sfAsset, XRP).getJson(JsonOptions::none);
        jvDeposit2[jss::Asset2] = STIssue(sfAsset, EUR).getJson(JsonOptions::none);
        jvDeposit2[jss::Amount] = XRP(10).value().getJson(JsonOptions::none);
        jvDeposit2[jss::Flags]  = tfSingleAsset;
 
        // Without sponsorship: alice2 has enough XRP → succeeds.
        env(jvDeposit2, ter(tesSUCCESS));
        env.close();
 
        // With bob2 as reserve sponsor: deposit() checks bob2's xrpLiquid → fails.
        env(jvDeposit2,
            sponsor::as(bob2, spfSponsorReserve),
            sig(sfSponsorSignature, bob2),
            ter(tecUNFUNDED_AMM));  // BUG: alice2's deposit rejected because bob2 is poor
        env.close();
    }
};
 
BEAST_DEFINE_TESTSUITE(TN8AMMDepositWrongXRPLiquid, app, xrpl);
 
}  // namespace test
}  // namespace xrpl
```
 
</details>
### How to run
 
```bash
cd /path/to/rippled-build
./xrpld --unittest=TN8AMMDepositWrongXRPLiquid
```
 
### Expected output
 
```text
xrpl.app.TN8AMMDepositWrongXRPLiquid AMMDeposit checks sponsor xrpLiquid instead of depositor
xrpl.app.TN8AMMDepositWrongXRPLiquid had 0 failures.
802ms, 1 suite, 1 case, 99 tests total, 0 failures
```
 
---
 
## Mitigation
 
Replace `sponsor.value_or(account_)` with `account_` unconditionally:
 
```cpp
// Current (buggy):
if (xrpLiquid(view, sponsor.value_or(account_), sponsor ? ownerCountAdj : 0, j_) >=
    depositAmount)
 
// Fixed:
if (xrpLiquid(view, account_, ownerCountAdj, j_) >= depositAmount)
```
 
The sponsor's role is to cover the **reserve increment** for any new owner objects created by the deposit. The sponsor does not contribute the deposited XRP itself. The depositor must always have sufficient liquid XRP for the deposit amount regardless of sponsorship status.
 
---
 
## Why This Is a Spec Violation (Not Acceptable Behaviour)
 
The `deposit()` helper was written assuming the sponsor covers reserve increments only. The `xrpLiquid` call at line 538 was intended to check whether the depositor can afford the deposit amount — the `ownerCountAdj` parameter adjusts for the new trust-line reserve, and the first argument identifies whose balance to evaluate. Using `sponsor.value_or(account_)` as that first argument causes the entire check to evaluate the wrong account's liquidity.
 
The correct call in `preclaim` (line 355) uses `accountID` directly — the depositor. The divergence in `deposit()` is an implementation error: the two paths must evaluate the same account for the same invariant to hold.
 
The false-reject case demonstrates the invariant break most clearly: a fully-funded depositor is denied a valid operation purely because the code reads the wrong account's balance. No protocol design intent produces this outcome.
 
---
 
## Severity Justification
 
**Low** — spec/invariant break (wrong account in `xrpLiquid` check); no direct adversarial fund extraction; functional inconsistency confined to the sponsored AMM deposit path.
 
| Factor | Assessment |
|--------|-----------|
| Monetary loss (bypass) | None — reserve correctly charged to sponsor who consented to sponsorship |
| Monetary loss (false-reject) | None — depositor's funds are safe; operation denied but not stolen |
| Victim in bypass | Sponsor (Bob) consented to the relationship; reserve charged as agreed |
| Victim in false-reject | Depositor (Alice) denied a valid operation; no financial harm |
| Spec invariant | `xrpLiquid` must check depositor's account; preclaim does this correctly; `deposit()` does not |
| Attack requirement | Requires an existing consented sponsorship relationship |
| Self-recovery | Alice can deposit without sponsorship if she has sufficient XRP |
 
**Why not Medium:** Neither case produces a direct monetary loss or a permanent denial of access to a third party without their consent. The bypass operates within a consented sponsorship relationship. The false-reject is a functional bug that Alice can work around by removing the sponsorship or depositing without it. There is no adversarial extraction vector.
 
---
 
## Key Code Locations
 
| Location | Role |
|----------|------|
| https://github.com/sherlock-audit/2026-04-xrp-ledger-april-2026-BlockGuard-team/blob/main/rippled/src/libxrpl/tx/transactors/dex/AMMDeposit.cpp#L538 | **Bug:** `xrpLiquid(view, sponsor.value_or(account_), ...)` — wrong account |
| https://github.com/sherlock-audit/2026-04-xrp-ledger-april-2026-BlockGuard-team/blob/main/rippled/src/libxrpl/tx/transactors/dex/AMMDeposit.cpp#L355 | Correct reference: `xrpLiquid(ctx.view, accountID, 1, ctx.j)` in preclaim |
| https://github.com/sherlock-audit/2026-04-xrp-ledger-april-2026-BlockGuard-team/blob/main/rippled/src/libxrpl/tx/transactors/dex/AMMDeposit.cpp#L524 | `getTxReserveSponsorAccountID(ctx_.tx)` — extracts sponsor AccountID |