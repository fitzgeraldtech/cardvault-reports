# Live E2E Setup

This project now has two Playwright modes:

- Mock mode: deterministic UI/browser coverage with no real auth dependency
- Live mode: real Supabase auth and real app behavior with a seeded test account

## 1. Commands

Mock suite:

```bash
npm run test:e2e
```

Live suite:

```bash
TEST_USER_EMAIL=you@example.com \
TEST_USER_PASSWORD=your-password \
npm run test:e2e:live
```

## 2. Environment

Live mode needs:

- `TEST_USER_EMAIL`
- `TEST_USER_PASSWORD`

Optional:

- `PLAYWRIGHT_BASE_URL`
- `VITE_SUPABASE_URL`
- `VITE_SUPABASE_ANON_KEY`

Defaults:

- `PLAYWRIGHT_BASE_URL=http://127.0.0.1:4174`
- Supabase URL and anon key default to the current project values

## 3. What Live Mode Covers

Current live specs:

- [auth-live.spec.ts](/home/chris/cardvault/tests/e2e/live/auth-live.spec.ts)
- [cards-live.spec.ts](/home/chris/cardvault/tests/e2e/live/cards-live.spec.ts)

Current intent:

- prove real sign-in works
- prove the real cards shell renders after auth
- create a foundation for further live-backend scenarios

## 4. Recommended Seed Account

The best live E2E account should be:

- isolated from real household data
- safe to mutate
- stable over time
- not used manually for day-to-day product usage

Recommended shape:

- email/password auth
- onboarding already completed
- at least one active card
- at least one reward balance
- no high-risk personal or production-only data

## 5. Recommended Next Live Specs

1. Onboarding flow for a fresh user
2. Card add/edit/close flow
3. Rewards balance edit persistence
4. Alerts page rendering and decision actions
5. Settings import/export/reset
6. Household create/invite/accept with two seeded users
7. Trip-goal create/edit/book flow
8. Cloud-sync consistency across two browser contexts

## 6. Why Both Modes Matter

Mock mode is best for:

- fast local iteration
- deterministic UI regression coverage
- validating what the user sees without backend volatility

Live mode is best for:

- real auth verification
- real backend integration
- catching env, schema, RLS, and sync regressions

Use both:

- mock mode as the fast baseline
- live mode as the production-trust layer
