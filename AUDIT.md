# Canton Patterns v1.0 — Security Audit Report

**Date:** 2026-02-16
**Auditor:** AI-assisted via Claude Opus 4.6 (Anthropic)
**Scope:** All 6 Daml pattern modules and their test suites in `src/`
**SDK Version:** Daml 2.9.7

---

## Executive Summary

A full security audit was performed on all 6 contract patterns in the canton-patterns library. The audit reviewed signatory/observer/controller permissions, ensure clauses, time handling, edge cases, and test coverage.

**3 bugs were found and fixed:**

| Severity | Pattern | Issue |
|----------|---------|-------|
| **High** | Vesting | `Claim` choice accepted caller-supplied time, allowing beneficiaries to claim unvested tokens by passing a future timestamp |
| **Medium** | Vesting | `RevokeUnvested` accepted caller-supplied time, allowing grantors to backdate revocations |
| **Medium** | Voting | `CancelBallot` had no deadline check, allowing proposers to cancel ballots after voting ended |

**2 hardening improvements were made:**

| Pattern | Improvement |
|---------|-------------|
| Multisig | Added `dedup signers == signers` ensure clause to reject duplicate signers |
| Escrow | Added `Refund` choice for post-deadline fund recovery by payer |

**18 new tests were added** across all 6 patterns to cover authorization edge cases, state machine violations, and temporal boundary conditions.

---

## Pattern: AccessControl

**Files:** `src/Patterns/AccessControl.daml`, `src/TestAccessControl.daml`

### What Was Reviewed
- Signatory/observer declarations on `RoleAssignment`, `RoleRequest`, `RoleGrant`
- Controller permissions on `Revoke`, `Approve`, `Reject`, `Accept`, `Withdraw`
- Ensure clauses (none present — roles are enum-typed, no numeric invariants)

### Findings

**No bugs found.** Authorization model is correct:
- `RoleAssignment`: signatory is admin, observer is grantee. Only admin can `Revoke`.
- `RoleRequest`: signatory is requester, observer is admin. Only admin can `Approve`/`Reject`.
- `RoleGrant`: signatory is admin, observer is grantee. Only grantee can `Accept`, only admin can `Withdraw`.

**Risk (Low):** No ensure clauses prevent `admin == grantee` (self-grant). This is a valid design choice but worth noting — an admin can grant themselves any role.

### Tests Added (3)
- `testNonAdminCannotApprove` — outsider party cannot approve a role request
- `testNonAdminCannotRevoke` — grantee cannot revoke their own role assignment
- `testNonGranteeCannotAccept` — outsider party cannot accept someone else's grant

### Status: AI-Audited — Pending Community Review

---

## Pattern: Escrow

**Files:** `src/Patterns/Escrow.daml`, `src/TestEscrow.daml`

### What Was Reviewed
- State machine transitions (AwaitingDeposit -> Funded -> Released/Disputed)
- Controller permissions on all choices
- Ensure clauses (`amount > 0.0`)
- Timeout/deadline handling

### Findings

**Bug (Medium) — Fixed:** `CheckTimeout` was a nonconsuming read-only query that returned a Bool but could not actually enforce a timeout or return funds. There was no mechanism for the payer to reclaim funds after a deadline.

**Fix:** Added a new `Refund` choice that allows the payer to reclaim funds after the deadline has passed, guarded by `now >= deadline` and `state == Funded`.

**Risk (Low):** `receiver` is only an observer, not a signatory. This means the receiver cannot independently prove on-ledger that they released the escrow. In production, the receiver should be added as a signatory on the Released state. This was not changed because it would require a propose-accept workflow for every state transition, which is a design tradeoff.

**Risk (Low):** The `EscrowState` data type includes `Released` and `Disputed`, but there are no choices to transition out of `Disputed` state (e.g., to an arbitrator resolution). This is acceptable for a pattern library but should be noted.

### Tests Added (5)
- `testCannotReleaseBeforeDeposit` — receiver cannot release an unfunded escrow
- `testCannotDisputeBeforeDeposit` — receiver cannot dispute an unfunded escrow
- `testRefundAfterDeadline` — payer can refund after deadline
- `testCannotRefundBeforeDeadline` — payer cannot refund before deadline
- `testReceiverCannotDeposit` — receiver cannot exercise payer-only Deposit choice

### Status: AI-Audited — Pending Community Review

---

## Pattern: Multisig

**Files:** `src/Patterns/Multisig.daml`, `src/TestMultisig.daml`

### What Was Reviewed
- Signatory accumulation via `signatory proposer :: approvals`
- Controller permissions on `Sign`, `Execute`, `Cancel`
- Ensure clauses for threshold validation
- Duplicate signer handling

### Findings

**Bug (Medium) — Fixed:** The ensure clause did not check for unique signers. A malicious proposer could create `signers = [alice, alice, alice]` with `minSigners = 3`, making it appear like a 3-of-3 multisig when only 1 party is actually required.

**Fix:** Added `dedup signers == signers` to the ensure clause, rejecting any proposal with duplicate entries in the signers list.

**Design Note:** The `Sign` choice already has a `dedup` call on approvals (line 35), which is redundant now that signers must be unique. The existing `dedup` is harmless and provides defense-in-depth.

**Risk (Low):** The proposer is not required to be in the signers list, meaning the proposer can create and execute multisig proposals without being one of the signers. This is by design but should be documented for users.

### Tests Added (3)
- `testDuplicateSignersRejected` — creating a proposal with duplicate signers fails
- `testDoubleSignRejected` — same signer cannot sign twice
- `testNonProposerCannotExecute` — outsider cannot execute a fully-signed proposal

### Status: AI-Audited — Pending Community Review

---

## Pattern: Vesting

**Files:** `src/Patterns/Vesting.daml`, `src/TestVesting.daml`

### What Was Reviewed
- Linear vesting calculation (microsecond precision)
- Time handling in `CalculateVested`, `Claim`, `RevokeUnvested`
- Ensure clauses (time ordering, amount bounds)
- Signatory/controller permissions

### Findings

**Bug (High) — Fixed:** The `Claim` choice accepted a `currentTime : Time` parameter from the caller instead of using `getTime`. A beneficiary could pass any future time to calculate a higher vested amount and claim tokens that hadn't actually vested yet. This is a direct financial exploit.

**Fix:** Removed the `currentTime` parameter. The choice now calls `getTime` internally to get the authoritative ledger time.

**Bug (Medium) — Fixed:** The `RevokeUnvested` choice accepted a `revokeTime : Time` parameter from the grantor. While the grantor controls this choice, they could backdate the revocation to before the cliff, setting `totalAmount = 0.0` and effectively stealing already-vested tokens. The ensure clause would catch some cases (if `endTime < cliffTime`), but not all.

**Fix:** Removed the `revokeTime` parameter. The choice now uses `getTime` and adds two guards:
1. `now >= cliffTime` — cannot revoke before the cliff (nothing to revoke)
2. `vested >= claimedAmount` — ensures the new total doesn't go below what's already been claimed

**Design Note:** The vesting calculation uses microsecond-precision integer arithmetic (`convertRelTimeToMicroseconds`). For very large `totalAmount` values and very short vesting periods, there could be precision loss in the `intToDecimal` conversion. This is unlikely to be an issue in practice but should be noted.

### Tests Added (3)
- `testClaimAfterCliff` — beneficiary claims a partial amount after cliff
- `testCannotOverclaim` — claiming more than vested amount fails
- `testCannotClaimBeforeCliff` — claiming before cliff fails (0 vested)

### Status: AI-Audited — Pending Community Review

---

## Pattern: Timelock

**Files:** `src/Patterns/Timelock.daml`, `src/TestTimelock.daml`

### What Was Reviewed
- Time guards on `ExecuteAction` and `CancelAction`
- Signatory/controller permissions
- `TimelockReceipt` signatory model
- Proposal accept/reject flow

### Findings

**No bugs found.** Time handling is correct:
- `ExecuteAction` checks `now >= unlockTime` using `getTime` (not caller-supplied time)
- `CancelAction` checks `now < unlockTime` using `getTime`
- `TimelockReceipt` has `signatory owner, beneficiary` — both parties attest to execution

**Design Note:** `TimelockReceipt` requires both owner and beneficiary as signatories. Since only the owner is a signatory on `TimelockAction`, the `ExecuteAction` choice creates a `TimelockReceipt` signed by both — this works because the beneficiary (controller) authorizes via the choice exercise, and the owner authorized by creating the `TimelockAction`. This is correct Daml authorization.

### Tests Added (3)
- `testOwnerCancelsBeforeUnlock` — owner successfully cancels before unlock
- `testCannotCancelAfterUnlock` — owner cannot cancel after unlock time passes
- `testNonBeneficiaryCannotExecute` — outsider cannot execute the action

### Status: AI-Audited — Pending Community Review

---

## Pattern: Voting

**Files:** `src/Patterns/Voting.daml`, `src/TestVoting.daml`

### What Was Reviewed
- Signatory accumulation via `signatory proposer :: fmap (\vr -> vr.voter) votes`
- Voter eligibility and double-vote prevention
- Deadline enforcement on `CastVote` and `Tally`
- `CancelBallot` authorization and timing
- Vote counting logic

### Findings

**Bug (Medium) — Fixed:** `CancelBallot` had no deadline check. The proposer could cancel a ballot after the deadline, after voters had already cast their votes, effectively discarding legitimate votes.

**Fix:** Added `now <- getTime` and `assertMsg "Cannot cancel after deadline" $ now < deadline` to the `CancelBallot` choice.

**Design Note:** The `passed` criterion is simple majority (`yesVotes > noVotes`). Abstentions do not count toward either side. A tie (equal yes and no) results in failure. This is a deliberate design choice that should be documented for users who may expect different quorum rules.

**Design Note:** `BallotResult` has only `signatory proposer`. Voters cannot independently verify the result on-ledger since they are not signatories on the result. In high-stakes scenarios, voters should also be signatories on `BallotResult`.

### Tests Added (4)
- `testDoubleVoteRejected` — same voter cannot vote twice
- `testCannotVoteAfterDeadline` — voting after deadline fails
- `testCannotTallyBeforeDeadline` — tallying before deadline fails
- `testTieResultsInFailure` — tied vote correctly results in `passed = False`

### Status: AI-Audited — Pending Community Review

---

## Known Limitations

This audit was performed by an AI model (Claude Opus 4.6) with the following inherent limitations:

1. **No runtime execution.** The Daml SDK was not available in the audit environment. Code was validated structurally but not compiled or executed. Findings are based on static analysis of Daml semantics.

2. **No formal verification.** The audit did not use formal methods, model checking, or symbolic execution. Properties like "no token inflation" or "funds conservation" were reasoned about but not mechanically proven.

3. **Single-reviewer bias.** AI review lacks the diversity of perspectives that a multi-person audit team provides. Subtle architectural issues or domain-specific risks may be missed.

4. **No cross-contract interaction analysis.** Each pattern was reviewed in isolation. In production, patterns may be composed together (e.g., a Multisig controlling a Vesting schedule), and composition-level vulnerabilities were not assessed.

5. **Daml runtime guarantees assumed.** The audit assumes the Daml runtime correctly enforces signatory authorization, privacy, and the contract model. Bugs in the Daml interpreter or Canton protocol are out of scope.

6. **No adversarial testing.** While negative tests (submitMustFail) were added, no systematic fuzzing, property-based testing, or adversarial simulation was performed.

7. **Time oracle trust.** Several patterns depend on `getTime` for security-critical decisions. In Canton, ledger time is bounded by the participant's local time, but participants can skew time within bounds. The audit did not assess time manipulation attacks specific to Canton's time model.

---

## Community Review Requested

This library is intended as a public good for Canton Network developers. The AI audit above is a starting point, not a replacement for human review.

**We explicitly invite Daml and Canton developers to:**

- Review the signatory/observer/controller model for each pattern
- Check the ensure clauses for missing invariants
- Assess whether the authorization model is appropriate for production use cases
- Review the vesting arithmetic for precision edge cases
- Evaluate whether the Escrow pattern needs a full state machine (with arbitration)
- Consider whether `BallotResult` should include voter signatories
- Test the patterns against Canton-specific behaviors (time bounds, privacy, sub-transactions)
- File issues at the project repository for any findings

**Every pattern is marked "AI-Audited — Pending Community Review" until reviewed by at least one experienced Daml developer.**

---

## Test Coverage Summary

| Pattern | Tests Before Audit | Tests After Audit | Coverage Gaps Closed |
|---------|-------------------|------------------|---------------------|
| AccessControl | 4 | 7 | Non-admin authorization, non-grantee authorization |
| Escrow | 4 | 9 | State machine violations, refund lifecycle, cross-party authorization |
| Multisig | 4 | 7 | Duplicate signers, double signing, non-proposer execution |
| Vesting | 4 | 7 | Claim lifecycle, overclaim prevention, pre-cliff claim prevention |
| Timelock | 4 | 7 | Cancel lifecycle, post-unlock cancel prevention, non-beneficiary execution |
| Voting | 4 | 8 | Double voting, post-deadline voting, pre-deadline tally, tie handling |
| **Total** | **24** | **45** | |
