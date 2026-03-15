# Audit Report: Gemini CLI (Flash) - 2026-03-15 - Full Review

## 1. Executive Summary
AmplifyPoints (v0.9.0) is a robust, well-architected mobile-first React SPA for credit card reward tracking. The project has a solid foundation in its data model, date arithmetic, and alert engine. However, as it transitions from a local-first to a cloud-synced multi-user application, it faces significant consistency and synchronization challenges. The most critical issue is the lack of a unified source-of-truth for reward balances, which results in conflicting totals across different screens. While the frontend and catalog data are of high quality, the synchronization logic and end-to-end verification of stateful flows are currently the primary risks to user trust.

## 2. Findings Table

| ID | Finding | Severity | Location | Recommended Fix | Effort |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **LOGIC-01** | Conflicting reward totals due to parallel balance tracking. | **CRITICAL** | `useCardVault.ts`, `transferRecommendations.ts` | Implement a canonical `getResolvedBalance` layer for all read paths. | Medium |
| **SYNC-01** | Empty cloud results do not clear local state during sync. | **HIGH** | `useCloudSync.ts`, `cloudSync.ts` | Treat empty cloud responses as a valid "clear" command rather than a no-op. | Small |
| **SYNC-02** | Sync state is device-level instead of user/household-level. | **HIGH** | `useCloudSync.ts` | Key initial sync state by `user_id` and `household_id` to prevent contamination. | Small |
| **ARCH-01** | `RootLayout.tsx` is an over-burdened "God component". | **MEDIUM** | `src/layouts/RootLayout.tsx` | Refactor modals and global side-effects into dedicated Providers. | Medium |
| **DATA-01** | `cardholder` field uses hardcoded strings ('Chris'/'Vanessa'). | **MEDIUM** | Throughout `src/` | Migrate to real Supabase `owner_user_id` (scheduled for Phase 12). | Large |
| **UX-01** | Hardcoded hex colors in secondary components (AlertsTab, CardsTab). | **LOW** | `AlertsTab.tsx`, `CardsTab.tsx` | Replace all hardcoded hex with `var(--color-*)` tokens for consistent dark mode. | Small |
| **TEST-01** | Missing E2E coverage for core write/destructive paths. | **HIGH** | `tests/e2e/` | Add Playwright tests for Add/Edit/Delete card and Settings reset. | Medium |

## 3. Site Map
- `/` (**DashboardPage**): Summary stats, hero card, trip goals, alert preview.
- `/cards` (**CardsPage**): Filterable/searchable card list, detail expansion, Add/Edit wizard.
- `/rewards` (**RewardsPage**): Program balances, loyalty memberships, transfer recommendations.
- `/travel` (**TravelPage**): Current stub; trip goals currently live on the Dashboard.
- `/strategy` (**StrategyPage**): 5/24 tracker, card recommendations, joint strategy advice.
- `/alerts` (**AlertsPage**): Full fee/SUB/cancel timeline with decision actions.
- `/admin` (**AdminPage**): Catalog change review queue (Admin/Super-Admin only).
- `/login` (**LoginPage**): Auth entry (Google OAuth/Email).
- `/auth/callback` (**AuthCallbackPage**): OAuth redirect handler.
- `/household/invite/:token` (**InviteAcceptPage**): Invite acceptance flow.

## 4. Data Flow Diagram
`Supabase (PostgreSQL)` <-> `Edge Functions (Sync/Ingestion)` <-> `Supabase Client` <-> `useCloudSync Hook` <-> `Zustand Stores (Cards/Rewards)` <-> `LocalStorage (Persistence)` <-> `React UI Components`

## 5. Risk Register

| Risk | Impact | Probability | Mitigation |
| :--- | :--- | :--- | :--- |
| **Data Inconsistency** | **HIGH** | **HIGH** | Centralize balance resolution logic immediately. |
| **Sync Poisoning** | **HIGH** | **MEDIUM** | Implement per-user sync keys and better empty-state handling. |
| **Auth Lock Hangs** | **CRITICAL** | **LOW** | (Resolved) Switched to `processLock` and added watchdog timer. |
| **Scalability (UI)** | **MEDIUM** | **MEDIUM** | Break down `RootLayout` and large tab components. |

## 6. UX Scorecard

| Tab | Usefulness | Usability | Completeness | Polish | Score |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Dashboard** | 9 | 8 | 8 | 9 | **8.5** |
| **Cards** | 10 | 9 | 9 | 9 | **9.3** |
| **Rewards** | 8 | 7 | 6 | 8 | **7.3** |
| **Alerts** | 10 | 8 | 9 | 8 | **8.8** |
| **Strategy** | 7 | 8 | 6 | 7 | **7.0** |
| **Travel** | 3 | 5 | 2 | 4 | **3.5** |

## 7. Recommendations
1. **Unify Balance Logic (High/Medium):** Create a canonical resolution layer for reward balances to fix the discrepancy between loyalty memberships and program totals.
2. **Harden Sync (High/Small):** Fix the "empty result no-op" bug and key sync state by user ID.
3. **E2E Expansion (High/Medium):** Prioritize Playwright tests for the Add/Edit card wizard.
4. **Tokenize UI Colors (Low/Small):** Complete the migration of hardcoded hex colors to CSS variables.
5. **Travel Tab Activation (Medium/Large):** Move trip goal management from the Dashboard to the Travel tab as per the Phase 10 roadmap.

## 8. Questions for the Team
1. Is the `credit_cards` table in Supabase ready for the full Phase 12 migration?
2. Are the `au_insights` and `proxy_card_flags` tables currently being populated by any background process?
3. Should we prioritize the "Private" card sharing tier before finalizing the household sync logic?

## 9. Prior Report Cross-Reference
- **Confirming `HOLISTIC_LOGIC_AUDIT.md`**: Yes, the "Mixed balance source-of-truth" remains a critical RED issue.
- **Confirming `E2E_GAP_ANALYSIS.md`**: Yes, the write paths (Add/Edit/Delete) are currently a significant testing blind spot.
- **Challenge**: The holistic audit flagged "Cloud sync undermined by correctness math" as RED. I'd argue it's a **SYNC** issue (integrity), not a **MATH** issue, but the impact is the same.
