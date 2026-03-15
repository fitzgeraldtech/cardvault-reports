# AUDIT_GPT4O_20260315_FULL

**Model:** GPT-4o (via aider + manual analysis)
**Date:** 2026-03-15
**Focus:** Full codebase review
**Repo:** vredensek1/cardvault @ master (v0.9.0)

---

## 1. Executive Summary

AmplifyPoints is a well-architected React + TypeScript + Zustand + Supabase application with strong type safety, comprehensive RLS policies, and a clean domain-organized structure. The codebase has solid foundations but suffers from several medium-severity issues: cloud sync race conditions, timezone-unsafe date calculations, incomplete card catalog data (50% of referenced cards missing from catalog), silent error swallowing across 12+ catch blocks, and monolithic components exceeding 800+ lines. The project is roughly 70% complete toward v1.0 with 9 of 13 phases done.

---

## 2. Findings Table

| ID | Severity | File | Issue | Recommended Fix | Effort |
|----|----------|------|-------|-----------------|--------|
| CS-01 | HIGH | `src/lib/cloudSync.ts` | Module-level mutable globals (`_scope`, `_suppressRealtime`) create race conditions in async operations | Refactor to React context or singleton class with initialization guards | Medium |
| CS-02 | HIGH | `src/lib/cloudSync.ts` | No dedup logic in offline queue. Failed-then-retried ops can create duplicates | Add idempotency keys or dedup by card ID before flushing | Medium |
| CS-03 | MEDIUM | `src/lib/cloudSync.ts` | `upsertRewardProgram` uses non-atomic delete-then-insert pattern | Use `onConflict` upsert or wrap in transaction | Small |
| DT-01 | HIGH | `src/lib/utils.ts` | `calculateDaysRemaining()` mixes local midnight with UTC-parsed dates, producing off-by-one errors near midnight in US timezones | Use `startOfDay()` from date-fns consistently on both operands | Small |
| CV-01 | MEDIUM | `src/hooks/useCardVault.ts` | `hasSapphire` uses fragile string matching (`.includes('sapphire')`) | Use `catalogCardId` to check against known Sapphire product IDs | Small |
| CV-02 | MEDIUM | `src/hooks/useCardVault.ts` | `importData` catches all errors and returns `false` with no detail | Return `{ success, error, skipped }` object. Add schema version to exports | Small |
| CV-03 | MEDIUM | `src/hooks/useCardVault.ts` | `generateRecommendations()` ignores its `_stats` parameter entirely | Either personalize using stats or convert to static UI copy | Medium |
| AU-01 | HIGH | `src/contexts/AuthContext.tsx` | Profile fetch uses `Promise.race` with 8s hardcoded timeout. If timeout wins, no retry. Users on slow networks get stuck | Add retry with exponential backoff and explicit "connection slow" UI | Medium |
| AU-02 | MEDIUM | `src/contexts/AuthContext.tsx` | `updateProfile()` updates local state optimistically but doesn't roll back on Supabase write failure | Add rollback on error | Small |
| ST-01 | MEDIUM | `src/stores/useCardsStore.ts` | Migration chain (v0-v7) is linear with no guard against skipped versions | Add version assertions or make each migration idempotent | Medium |
| ST-02 | LOW | `src/stores/useCardsStore.ts` | ID generation fallback (`Date.now() + Math.random()`) is collision-prone | Use `crypto.randomUUID()` only, or add nanoid | Small |
| HH-01 | MEDIUM | `src/hooks/useHousehold.ts` | No email format validation before sending invites | Add regex validation | Small |
| HH-02 | MEDIUM | `src/hooks/useHousehold.ts` | `inviteMember` returns `{ error: null, emailError }` on email failure. Caller thinks success | Unify error shape | Small |
| ERR-01 | MEDIUM | Multiple files | 12+ catch blocks silently swallow errors (`catch {}` or `catch { return false }`) | Standardize with `captureError(context, err)` utility | Medium |
| DATA-01 | HIGH | `src/data/cardCatalog.ts` | Only ~26 of 54 referenced cards have catalog entries. Business cards, Capital One, Barclays, BofA largely missing | Complete the remaining ~28 CardProduct entries | Large |
| DATA-02 | MEDIUM | `src/data/referralLinks.ts` | 3 referral IDs don't match catalog (`amex_bce`, `capitalone_venture_x`, `amex_hilton_honors`) | Fix to match catalog IDs exactly | Small |
| DATA-03 | MEDIUM | Cross-file | No runtime or test validation that `rewardsProgramId` references resolve to actual programs | Add cross-reference unit test | Small |
| UX-01 | MEDIUM | `src/pages/TravelPage.tsx` | Travel tab shows "Coming soon" placeholder but is in the main navigation | Remove from nav until ready, or move trip goals here | Small |
| UX-02 | MEDIUM | `src/components/AddCardModal.tsx` | 5-step wizard has no progress indicator and no quick-add option | Add step dots and single-screen quick-add mode | Medium |
| UX-03 | LOW | Multiple tabs | Empty states use different designs across tabs | Extract shared `EmptyState` component | Small |
| UX-04 | LOW | `src/components/tabs/DashboardTab.tsx` | Hardcoded `REDEMPTION_EVENTS` renders fake data as if real | Remove or clearly mark as demo | Small |
| UX-05 | LOW | Multiple files | `formatPointsCompact()` duplicated in DashboardTab and RewardsTab | Extract to `src/lib/formatters.ts` | Small |
| A11Y-01 | MEDIUM | Multiple components | Missing `aria-expanded` on collapsibles, missing `aria-label` on icon buttons | Add ARIA attributes | Small |
| A11Y-02 | MEDIUM | All modals | No focus trap implementation | Add focus-trap-react or equivalent | Medium |
| A11Y-03 | LOW | `src/components/BottomNav.tsx` | Touch targets 20px (below 44px minimum) | Increase to 44px | Small |
| SEC-01 | MEDIUM | `src/hooks/useHousehold.ts` | No rate limiting on invite creation (can spam SES) | Add DB-level throttle (max 5/day/user) | Medium |
| SEC-02 | LOW | Supabase config | Signup open to anyone | Restrict to invite-only | Small |
| PERF-01 | LOW | `src/stores/useRewardsStore.ts` | `addPortfolioSnapshot()` filters entire array for dedup on every call | Use Map or index by month key | Small |

---

## 3. Site Map

```
https://amplifypoints.com/
+-- /login                    [PUBLIC]  Email/password + Google OAuth
+-- /auth/callback            [PUBLIC]  OAuth redirect handler
+-- /household/invite/:token  [PUBLIC]  Household invite acceptance
+-- / (Dashboard)             [AUTH]    Portfolio overview, stats, trip goals, alerts preview
+-- /cards                    [AUTH]    Card inventory, filters, add card wizard, cheat sheet
+-- /rewards                  [AUTH]    Program balances, loyalty memberships, transfers
+-- /travel                   [AUTH]    STUB - "Coming soon"
+-- /strategy                 [AUTH]    Chase 5/24, recommendations, joint strategies
+-- /alerts                   [AUTH]    Annual fees, cancel reminders, SUB deadlines, fee waivers
+-- /admin                    [ADMIN]   Catalog change review queue
```

---

## 4. Data Flow Diagram

```
User Input
    |
    v
React Components (pages/tabs/modals)
    |
    v
Custom Hooks (useCardVault, useHousehold, useSubCheckins, useBalanceGate)
    |
    +---> Zustand Stores (useCardsStore v7, useRewardsStore v3)
    |         |
    |         +---> localStorage (amplifypoints_cards, amplifypoints_rewards, etc.)
    |         |
    |         +---> cloudSync.ts ---> Supabase (user_cards, user_reward_programs)
    |                                     |
    |                                     +---> Realtime subscriptions (broadcast)
    |
    +---> AuthContext ---> Supabase Auth (profiles, sessions)
    |
    +---> useHousehold ---> Supabase (households, household_members, household_invites)
    |                           |
    |                           +---> Edge Functions (send-household-invite, notify-household-joined)
    |
    +---> Alert generation (derived from card dates, computed on render)

Source of Truth:
  - Cards/Programs: localStorage (primary), Supabase (sync target)
  - Auth/Profile: Supabase (primary)
  - Households: Supabase (primary)
  - Trip Goals: Supabase (primary)
  - UI preferences: localStorage only
```

---

## 5. Risk Register

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Cloud sync data loss from race condition (CS-01) | Medium | High | Refactor mutable globals to context with init guards |
| Off-by-one date errors in alerts (DT-01) | High | Medium | Fix timezone handling in calculateDaysRemaining |
| Users can't add 50% of cards from catalog (DATA-01) | High | High | Complete the remaining 28 catalog entries |
| Auth hangs on slow networks (AU-01) | Medium | High | Add retry logic and explicit slow-connection UI |
| Silent data corruption from skipped migrations (ST-01) | Low | High | Add version guards to migration chain |
| Invite spam via SES (SEC-01) | Low | Medium | Add rate limiting at DB level |
| Import of malformed data (CV-02) | Medium | Medium | Add schema version and validation to import |

---

## 6. UX Scorecard

| Tab | Usefulness | Usability | Completeness | Polish | Score |
|-----|-----------|-----------|-------------|--------|-------|
| Dashboard | 9 | 8 | 7 | 7 | 7.8 |
| Cards | 9 | 8 | 8 | 8 | 8.3 |
| Rewards | 8 | 7 | 7 | 7 | 7.3 |
| Travel | 2 | 1 | 1 | 1 | 1.3 |
| Strategy | 6 | 7 | 5 | 6 | 6.0 |
| Alerts | 9 | 9 | 9 | 8 | 8.8 |
| Admin | 7 | 7 | 7 | 6 | 6.8 |
| Settings | 7 | 6 | 7 | 6 | 6.5 |

**Best feature:** Alerts tab - clean urgency visualization, decision tracking, fee-paid muting
**Worst feature:** Travel tab - stub with no content, shouldn't be in nav

---

## 7. Recommendations

### Critical (do first)
| # | Action | Effort | Impact |
|---|--------|--------|--------|
| 1 | Fix timezone bug in `calculateDaysRemaining` | Small | Every alert and deadline affected |
| 2 | Fix cloud sync race condition (mutable globals) | Medium | Data loss prevention |
| 3 | Complete card catalog (28 missing products) | Large | 50% of cards can't be added from catalog |
| 4 | Fix referral link ID mismatches (3 broken) | Small | Referral Optimizer broken for those cards |

### High Priority
| # | Action | Effort | Impact |
|---|--------|--------|--------|
| 5 | Add auth retry with backoff and slow-connection UI | Medium | Prevents user lockout on slow networks |
| 6 | Standardize error handling (replace 12+ silent catches) | Medium | Debuggability across entire app |
| 7 | Add cross-reference validation tests for data files | Small | Prevents silent lookup failures |
| 8 | Add rate limiting on invite creation | Medium | SES spam prevention |

### Medium Priority
| # | Action | Effort | Impact |
|---|--------|--------|--------|
| 9 | Remove or build Travel tab | Small | Eliminates dead end in nav |
| 10 | Add onboarding progress indicator | Small | Reduces user confusion |
| 11 | Add focus traps to modals | Medium | Accessibility compliance |
| 12 | Add ARIA attributes to collapsibles and icon buttons | Small | Screen reader support |
| 13 | Split monolithic components (7 files > 800 lines) | Large | Maintainability |

### Low Priority
| # | Action | Effort | Impact |
|---|--------|--------|--------|
| 14 | Remove fake REDEMPTION_EVENTS from dashboard | Small | Data integrity |
| 15 | Deduplicate formatPointsCompact across files | Small | Code cleanliness |
| 16 | Increase touch targets to 44px minimum | Small | Mobile usability |
| 17 | Extract shared EmptyState component | Small | UI consistency |

---

## 8. Questions for the Team

1. **Cloud sync source of truth:** Is localStorage or Supabase intended to be authoritative? Current design is ambiguous - localStorage is primary but Supabase is the sync target. What happens on conflict?
2. **Travel tab:** Should this be removed from nav entirely until Phase 10, or should trip goals (currently on Dashboard) be moved here?
3. **Signup restriction:** The app is designed for 2 users (Chris + Vanessa). Should signup be restricted to invite-only immediately?
4. **Card catalog completeness:** Is the missing 50% intentional (phased rollout) or an oversight? Earn rates exist for all 54 cards but catalog only has 26.
5. **Recommendation engine:** `generateRecommendations()` ignores its stats parameter. Is there a plan to personalize, or should this be converted to static content?
6. **Household archival cleanup:** `archive_expires_at` is set on archived households but no cron job deletes expired archives. Is this a GDPR concern?
7. **Error monitoring:** Sentry is conditionally loaded (`window.Sentry`) but appears inactive. When will it be activated?

---

## 9. Prior Report Cross-Reference

### vs. E2E_GAP_ANALYSIS.md
- **Confirm:** Auth flow timeout issues identified. My analysis adds specific line references (AuthContext.tsx lines 48-80) and the 8-second hardcoded timeout detail.
- **Confirm:** Cloud sync gaps. My analysis adds the specific race condition mechanism (module-level mutable globals) and duplicate queue entry risk.
- **Additional context:** The gap analysis noted missing features but didn't flag the fake REDEMPTION_EVENTS data being rendered as real user data on the dashboard.

### vs. HOLISTIC_LOGIC_AUDIT.md
- **Confirm:** Date calculation issues. My analysis specifies the exact timezone bug mechanism (local midnight vs UTC-parsed dates with `parseISO`).
- **Confirm:** Error handling inconsistency. My analysis counts 12+ silent catch blocks across specific files.
- **Additional context:** The logic audit didn't flag the `hasSapphire` fragile string matching or the migration chain skip vulnerability.

### vs. LOGIC_DECISION_TABLES.md
- **Confirm:** Alert severity thresholds are hardcoded magic numbers (60/120 days). My analysis recommends centralizing in a constants file.
- **Additional context:** Decision tables didn't cover the import/export validation gap (no schema version, no validation on import).

### vs. LOGIC_REVIEW.md
- **Confirm:** Sync retry logic needs improvement. My analysis identifies the specific non-atomic delete-then-insert pattern in `upsertRewardProgram`.
- **Challenge:** The logic review rates code quality as generally high. My analysis finds that while architecture is strong, the execution has significant AI-generated patterns (fake data, unused parameters, duplicate utilities) that need cleanup.

### vs. LIVE_E2E_VALIDATION_REPORT.md
- **Confirm:** Live testing shows the app is functional end-to-end.
- **Additional context:** Live testing didn't catch that the portfolio sparkline chart renders reverse-engineered fake history, not actual user data.

---

*Report generated 2026-03-15 following the AmplifyPoints LLM Audit Playbook v1.*
