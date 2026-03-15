# CardVault Browser Coverage And Gap Analysis

## 1. Executive Summary

This report is grounded in what the app currently does in-browser, not just what the code suggests it should do.

Current browser baseline:
- 12 Playwright tests passing in deterministic mock mode
- 117 unit tests already passing
- the app shell, key portfolio views, core filters, rewards editing, settings dark-mode persistence, alerts interactions, strategy interactions, and mobile bottom navigation are now browser-verified

What that means:
- the product has a real, testable frontend foundation
- the biggest remaining risk is not whether the UI renders
- the biggest remaining risk is whether stateful flows stay correct when real auth, sync, household state, imports, and destructive actions are involved

Bottom-line assessment:
- current user-facing shell and presentation layer: mostly in place
- current workflow coverage: materially improved, but still incomplete
- current backend-integrated trust level: still limited

## 2. Scope And Method

This analysis is based on:
- direct repository inspection
- running the app locally under Playwright mock mode
- using Playwright to exercise the same surfaces a user sees
- observing which interactions were straightforward, brittle, or still missing

Important limit:
- these tests validate browser-visible behavior in a deterministic environment
- they do not yet validate live Supabase auth, live cloud sync, live household collaboration, or real seeded-account data mutations end to end

## 3. What Is Currently In Place

### 3.1 Browser-tested functionality

The following flows are currently verified in-browser:
- guest users land on the login screen
- mock sign-in enters the app shell
- authenticated users bypass login
- primary navigation across overview, cards, rewards, travel, and strategy works
- cards search filters visible card inventory
- cards person/type filters combine correctly
- rewards balance edits persist across reload
- trip-goal entry point opens from the dashboard
- alerts view renders urgency controls and annual-fee actions
- strategy view renders major recommendation modules and dismiss behavior
- settings modal opens and dark-mode preference persists across reload
- mobile viewport navigation remains usable across primary tabs

### 3.2 Product capability visible from the current UI

From a user’s point of view, the project already has:
- a coherent login and app-shell experience
- a strong cards portfolio view
- a rewards surface with immediate editable state
- deadline/alert management UI
- a strategy page with static and dynamic advice surfaces
- a settings modal with account and data-management functions
- responsive navigation that still works on a phone-sized viewport

## 4. What The Browser Testing Suggests Is Working Well

### 4.1 Product strengths

- The app feels like a real product, not a stitched-together prototype.
- Cards filtering is fast and legible, which is important because that screen is one of the clearest user-value surfaces.
- Rewards editing feels immediate and survives reload, which builds trust in the local persistence model.
- Strategy and alerts both expose user-facing decision surfaces instead of only static summaries.
- The mobile shell remains navigable without collapsing into a broken desktop layout.

### 4.2 Engineering strengths

- The deterministic mock-mode harness is a meaningful improvement to testability.
- The app is now structured well enough to support stable browser coverage without requiring the live backend for every run.
- State persistence is testable at the UI layer, not just at store-unit-test level.

## 5. What Is Still Missing

### 5.1 High-priority missing E2E coverage

| Area | Current state | Why it matters |
|---|---|---|
| real Supabase authentication | not run end to end | no production-trust proof for sign-in/session behavior |
| onboarding wizard | unverified in browser | first-run success path is still a blind spot |
| add card flow | unverified | high-value write path |
| edit card flow | unverified | core maintenance path |
| delete/close card flow | unverified | destructive lifecycle path |
| settings import/export/reset | unverified | highest-risk local data operations |
| full trip-goal create/edit flow | partially verified | current coverage stops at modal entry |
| notifications panel behavior | unverified | visible app signal channel |
| sub check-in modal workflow | unverified | recurring guidance and nudging logic |
| balance gate real behavior | not truly tested | one of the most interruption-prone flows |
| household create/invite/accept | unverified | important product differentiator |
| cloud sync and realtime updates | unverified | highest correctness risk in the app |

### 5.2 Medium-priority missing coverage

| Area | Current state | Why it matters |
|---|---|---|
| accessibility behavior | only indirectly observed | weak labels and modal semantics can hurt both users and tests |
| notification CTA routing | not covered | can silently fail without obvious regressions |
| admin/review flows | not covered | operational workflows remain outside the safety net |
| mobile deep interactions | partially covered | shell is proven, task flows are not |
| empty-state behavior | mostly untested | many apps fail most obviously on sparse accounts |
| import edge cases | untested | user trust risk |

## 6. Gap Analysis By Category

### 6.1 Features Currently In Place

| Area | Status | Notes |
|---|---|---|
| login and shell | in place | browser-verified |
| portfolio overview | in place | visible, dense, and interactive |
| cards inventory and filtering | in place | browser-verified |
| rewards tracking | in place | browser-verified for editing persistence |
| alerts view | in place | browser-verified for core urgency/action visibility |
| strategy view | in place | browser-verified for major modules |
| settings basics | in place | browser-verified for modal open and dark mode persistence |
| mobile navigation | in place | browser-verified |

### 6.2 Features Missing Or Only Partial

| Area | Status | Observation |
|---|---|---|
| travel tab | partial | route still reads like a placeholder while trip-goal capability lives elsewhere |
| trip-goal management | partial | entry point exists, end-to-end task flow still incomplete |
| household collaboration | partial | architecture exists, trust level does not |
| admin curation loop | partial | not part of a validated operational system |
| recommendation explainability | partial | advice is visible, rationale is still too implicit |

### 6.3 Foundational Improvements Recommended

| Foundation area | Current state | Recommendation |
|---|---|---|
| source of truth for balances and valuations | mixed | centralize how dashboard, rewards, and strategy derive numbers |
| backend-integrated test lane | scaffolded only | finish and run seeded live E2E regularly |
| sync confidence | weak | add scenario tests for deletes, empty states, and multi-device refresh |
| modal/accessibility semantics | uneven | add explicit accessible labels and less brittle interaction patterns |
| data-destructive flows | under-tested | verify import, reset, delete, and sync reconciliation in browser |

### 6.4 UX Improvements Recommended

| UX area | Observation | Recommendation |
|---|---|---|
| dashboard density | too much stacked decision-making on one screen | split summary from management tasks |
| travel information architecture | travel tab and trip-goal behavior are split | move trip-goal ownership into travel, keep dashboard as summary only |
| modal interaction model | key tasks are heavily modal | reduce blocking prompts and provide clearer escape paths |
| terminology certainty | labels can overstate confidence in calculated values | distinguish exact, estimated, and heuristic values |
| recommendation trust | users can see advice faster than they can understand it | add “why this recommendation exists” detail |

## 7. What Looked Weak During Automation

These are not necessarily bugs, but they are signals worth taking seriously because they showed friction while driving the app like a user:

- The settings dropdown interaction was harder to automate than it should be.
- Several controls depend on visual text placement rather than robust accessible targeting.
- The dashboard remains the gravitational center of too many responsibilities.
- The `Travel` route still under-delivers relative to the rest of the product.
- Some of the most important behaviors are still only trustworthy in local/mock state, not real backend state.

## 8. Recommended Next Steps

### 8.1 Next testing steps

1. Add browser coverage for add/edit/delete card flows.
2. Add browser coverage for settings import, export, and reset.
3. Complete the trip-goal flow from create through edit/update.
4. Add notification-panel tests and balance-gate tests.
5. Run the live-backend Playwright lane once seeded credentials are available.

### 8.2 Next feature/foundation steps

1. Turn `Travel` into a first-class feature area.
2. Make recommendation logic more explainable to the user.
3. Centralize number derivation so dashboard and strategy use the same truth model.
4. Harden cloud sync around empty states, deletes, and first-sync scoping.
5. Reduce reliance on modal-heavy interaction for primary workflows.

## 9. Bottom Line

The project does not need a restart.

It already has:
- a credible frontend foundation
- meaningful user-facing functionality
- enough testability to keep improving safely

What it still needs is:
- coverage for the highest-risk write paths
- proof against the real backend
- stronger source-of-truth discipline around state and calculations
- cleaner UX separation between overview, management, and planning

In practical terms:
- the visible product is ahead of its trust infrastructure
- the right move is to keep building on this base, not replace it
