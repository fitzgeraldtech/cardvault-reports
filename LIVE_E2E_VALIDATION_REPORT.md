# CardVault Live E2E Validation Report

Prepared: 2026-03-15 UTC

## Executive Summary

The live browser audit now covers nine real-backend Playwright specs against the seeded admin account.

Current outcome:
- 7 of 9 live specs passed
- 2 of 9 live specs failed in ways that look like real product issues rather than harness problems

What is validated:
- real Supabase email/password auth works
- protected routing works
- core read-only surfaces load with live data
- card add/edit/delete works
- rewards balance edit/restore works
- export/reset/import backup flow works
- admin review queue access works

What is not validated cleanly:
- household create/invite/delete flow
- cross-context rewards sync consistency

Overall assessment:
- the app is on a usable foundation and does not need a restart
- the product is past the "can it operate?" stage and into the "can users trust the state transitions?" stage
- the highest-value work now is fixing sync, household lifecycle behavior, and blocking modal control flow

## Environment

- app under test: local dev server pointed at the live Supabase backend
- backend: live Supabase project
- test account: seeded admin account with onboarding completed
- auth method: real email/password login
- browser runner: Playwright live config

## Coverage Executed

### Passed Live Specs

1. [auth-live.spec.ts](/home/chris/cardvault/tests/e2e/live/auth-live.spec.ts)

Validated:
- user can sign in with email/password
- signed-in user reaches the dashboard shell

2. [cards-live.spec.ts](/home/chris/cardvault/tests/e2e/live/cards-live.spec.ts)

Validated:
- authenticated user can open cards
- search/filter interaction works against live data

3. [shell-live.spec.ts](/home/chris/cardvault/tests/e2e/live/shell-live.spec.ts)

Validated:
- rewards route loads
- alerts route loads
- strategy route loads
- settings modal opens from the user menu

4. [admin-live.spec.ts](/home/chris/cardvault/tests/e2e/live/admin-live.spec.ts)

Validated:
- admin review queue is reachable behind the admin guard

5. [card-lifecycle-live.spec.ts](/home/chris/cardvault/tests/e2e/live/card-lifecycle-live.spec.ts)

Validated:
- user can add a real card from the catalog
- user can edit card metadata
- user can permanently delete the card

6. [rewards-mutations-live.spec.ts](/home/chris/cardvault/tests/e2e/live/rewards-mutations-live.spec.ts)

Validated:
- user can change a live rewards balance
- edited value persists in-session
- original value can be restored cleanly

7. [settings-backup-live.spec.ts](/home/chris/cardvault/tests/e2e/live/settings-backup-live.spec.ts)

Validated:
- user can export account backup JSON
- reset-to-defaults executes
- import restores the exported dataset
- a temporary sentinel card added after export is removed by restore, which proves the import materially changed state

### Failed Live Specs

8. [household-live.spec.ts](/home/chris/cardvault/tests/e2e/live/household-live.spec.ts)

Observed failure:
- after submitting `Create & Send Invite`, the newly created household name never becomes visible
- the test fails at the first post-submit assertion

Why this looks real:
- the failure is stable
- it occurs after the form is filled and submitted, not during login or setup
- the expected record is simply not rendered afterward

9. [sync-live.spec.ts](/home/chris/cardvault/tests/e2e/live/sync-live.spec.ts)

Observed failure:
- context A updates `Chase UR` from `290.0K` to `290.3K`
- context A reflects the new value
- after reload, context B still shows `290.0K`

Why this looks real:
- the mutation succeeds in the editing context
- the receiving context is explicitly reloaded
- the stale value remains visible after reload, which suggests sync or re-fetch inconsistency rather than flaky UI timing

## Findings

### 1. Live auth and core routing are healthy

Status: GREEN

Evidence:
- real login succeeded repeatedly
- protected routes rendered consistently
- admin gating behaved correctly for the seeded admin account

Why it matters:
- the app is operational against the real backend
- this lowers the risk that the project needs structural rework

### 2. Core mutations are stronger than expected

Status: GREEN

Evidence:
- card lifecycle mutation passed end to end
- rewards balance edit/restore passed end to end
- backup export, reset, and restore passed end to end

Why it matters:
- high-risk user actions are not universally broken
- the data model is sturdy enough to support repair work instead of restart work

### 3. Blocking prompts still interfere with ordinary use

Status: YELLOW

Observed behavior:
- the monthly `Quick check-in` modal can intercept navigation immediately after login
- a SUB progress check-in can also appear during testing and needs dismissal

Why it matters:
- this creates friction before the user starts their intended task
- it also makes automated and manual QA harder because navigation state becomes contingent on hidden cadence rules

Recommendation:
- allow defer/dismiss behavior that does not trap the session
- document recurrence rules and make them observable in settings or logs

### 4. Household lifecycle is not trustworthy yet

Status: RED

Observed behavior:
- household creation did not produce a visible new household record after submission
- downstream invite-cancel and delete steps therefore never became testable

Why it matters:
- this is not cosmetic; it blocks a full multi-user collaboration feature
- household state is foundational to sharing and sync assumptions elsewhere in the product

Recommendation:
- instrument the create-household mutation and post-submit refresh path
- verify success toast, query invalidation, and rendered state all come from the same source of truth
- add server-asserted tests around household creation response payloads

### 5. Cross-context rewards sync is likely wrong today

Status: RED

Observed behavior:
- a rewards edit committed in one browser context did not become visible in another context even after reload

Why it matters:
- stale balances directly undermine trust in portfolio totals, recommendations, and transfer decisions
- this aligns with earlier code-review concern that cloud sync logic has edge cases around refresh and empty-state handling

Recommendation:
- trace the rewards update path from mutation to invalidation to fetch to render
- verify whether the second context is reading stale local state, stale cache, or stale backend rows
- add a dedicated sync regression test that asserts exact before/after values across two contexts

## Updated Coverage Assessment

What is now proven in live browser tests:
- auth
- protected navigation
- cards read flow
- rewards read flow
- alerts read flow
- strategy read flow
- settings entry
- admin review queue
- card create/edit/delete
- rewards edit/restore
- backup export/reset/import

What remains unproven or failing:
- household create/invite/delete
- multi-context sync consistency
- trip goal mutation flows
- referral flows
- import/export behavior across larger and messier real datasets

## Test Infrastructure Improvements Made

Supporting changes added during this audit:
- live helpers support either magic-link or email/password auth, though email/password is the reliable path
- a reusable prompt dismissor handles `Quick check-in` and SUB check-in interruptions
- settings open/close helpers were hardened for modal-based flows
- new live specs were added for admin, card lifecycle, rewards mutation, backup/restore, household, and sync coverage

Files:
- [helpers.ts](/home/chris/cardvault/tests/e2e/live/helpers.ts)
- [admin-live.spec.ts](/home/chris/cardvault/tests/e2e/live/admin-live.spec.ts)
- [card-lifecycle-live.spec.ts](/home/chris/cardvault/tests/e2e/live/card-lifecycle-live.spec.ts)
- [rewards-mutations-live.spec.ts](/home/chris/cardvault/tests/e2e/live/rewards-mutations-live.spec.ts)
- [settings-backup-live.spec.ts](/home/chris/cardvault/tests/e2e/live/settings-backup-live.spec.ts)
- [household-live.spec.ts](/home/chris/cardvault/tests/e2e/live/household-live.spec.ts)
- [sync-live.spec.ts](/home/chris/cardvault/tests/e2e/live/sync-live.spec.ts)
- [LIVE_E2E_SETUP.md](/home/chris/cardvault/LIVE_E2E_SETUP.md)

## Recommended Next Steps

1. Fix the rewards sync defect first.
Reason:
- it affects trust in user-visible numbers across sessions and devices

2. Fix household create/render flow second.
Reason:
- it blocks an entire collaboration feature set

3. Soften or defer blocking check-in prompts.
Reason:
- this improves first-minute usability and reduces QA friction

4. Add one server-observed mutation audit layer.
Reason:
- for household and sync failures, browser-only evidence is enough to flag the defect but not enough to isolate whether the break is in persistence, cache invalidation, or rendering

## Bottom Line

The live testing picture is better than the earlier code review suggested.

This is not a "start over" product. The app has a solid enough operational core to justify targeted fixes. The failing areas are specific and high-impact:
- household lifecycle behavior
- cross-context rewards sync
- blocking modal cadence and control flow

That is a repair plan, not a rewrite plan.
