# AmplifyPoints (AmplifyPoints) — Phased Fix Plan

> **Purpose:** Machine-readable remediation plan for use with Claude Code CLI (Opus 4.6, 1M context).
> Load this file alongside the full codebase. Execute phases in order.
> Each phase is designed to be completed in a single session.
>
> **Date:** 2026-03-15 | **Overall Score:** 52/100 | **Target Score After All Phases:** 80+/100

---

## How To Use This File

```bash
# In the cardvault repo root:
claude --model claude-opus-4-6 --context 1000000

# Then paste or reference this file:
# "Read CARDVAULT_FIX_PLAN.md and execute Phase 1"
```

Each phase contains:
- **Goal:** What we're fixing and why
- **Files:** Exact paths to modify
- **Changes:** Specific code changes with before/after
- **Tests:** How to verify the fix
- **Acceptance Criteria:** How to know it's done

---

## Phase 1: Fix Dashboard Balance Source-of-Truth (CRITICAL)

**Goal:** The dashboard reads `RewardProgram.balance` for ALL programs, but loyalty programs (Delta, AA, Hyatt, JetBlue, Alaska) store their real balances in `loyaltyMemberships[]`. This causes the dashboard to show stale, incorrect portfolio values. Fix the dashboard to use the same canonical balance resolution used by `transferRecommendations.ts`.

**Impact:** Fixes incorrect portfolio value, total points, month-over-month delta, and top reward programs display.

### Files to Modify

1. `src/hooks/useAmplifyPoints.ts` — lines 35-79 (stats + topRewardPrograms)
2. `src/lib/transferRecommendations.ts` — extract shared helper (already has correct logic)

### Changes

#### Step 1: Create canonical balance resolver

Create a new export in `src/lib/transferRecommendations.ts` (or a new file `src/lib/balanceResolver.ts`):

```typescript
/**
 * Canonical balance resolver: returns the correct total balance and
 * portfolio value for all reward programs, avoiding double-counting
 * loyalty membership programs.
 *
 * Rules:
 * - Loyalty-membership programs (Delta, AA, Hyatt, JetBlue, Alaska):
 *     balances come from loyaltyMemberships[] (per-person, summed)
 * - Card-linked currencies (Chase UR, Amex MR, Citi TYP, Capital One):
 *     balances come from RewardProgram.balance
 */
export function getResolvedPortfolioStats(
  programs: RewardProgram[],
  loyaltyMemberships: LoyaltyMembership[],
  mode: 'conservative' | 'optimistic'
): { totalPoints: number; portfolioValueMin: number; portfolioValueMax: number } {
  const loyaltyProgramIds = new Set(loyaltyMemberships.map((m) => m.programId))

  // Loyalty membership balances (derived at read time)
  let loyaltyPoints = 0
  let loyaltyValueMin = 0
  let loyaltyValueMax = 0

  // Group memberships by programId and sum balances
  const membershipTotals = new Map<string, number>()
  for (const m of loyaltyMemberships) {
    membershipTotals.set(m.programId, (membershipTotals.get(m.programId) ?? 0) + m.balance)
  }

  for (const [programId, balance] of membershipTotals) {
    const prog = programs.find((p) => p.id === programId)
    if (!prog || balance <= 0) continue
    loyaltyPoints += balance
    loyaltyValueMin += (balance * (prog.conservativeCpp ?? prog.cppRange[0])) / 100
    loyaltyValueMax += (balance * (prog.optimisticCpp ?? prog.cppRange[1])) / 100
  }

  // Card-linked programs (skip loyalty-membership program IDs)
  let cardLinkedPoints = 0
  let cardLinkedValueMin = 0
  let cardLinkedValueMax = 0

  for (const prog of programs) {
    if (loyaltyProgramIds.has(prog.id) || prog.balance <= 0) continue
    cardLinkedPoints += prog.balance
    cardLinkedValueMin += (prog.balance * (prog.conservativeCpp ?? prog.cppRange[0])) / 100
    cardLinkedValueMax += (prog.balance * (prog.optimisticCpp ?? prog.cppRange[1])) / 100
  }

  return {
    totalPoints: loyaltyPoints + cardLinkedPoints,
    portfolioValueMin: loyaltyValueMin + cardLinkedValueMin,
    portfolioValueMax: loyaltyValueMax + cardLinkedValueMax,
  }
}
```

#### Step 2: Update useAmplifyPoints.ts stats computation

In `src/hooks/useAmplifyPoints.ts`, import the new resolver and the loyaltyMemberships from the rewards store:

```typescript
// Add to imports:
import { getResolvedPortfolioStats } from '@/lib/transferRecommendations'

// Add to store selectors (after line 31):
const loyaltyMemberships = useRewardsStore((s) => s.loyaltyMemberships)
const valuationMode = useRewardsStore((s) => s.valuationMode)
```

Replace lines 43-46 in the stats useMemo:

```typescript
// BEFORE (WRONG):
// const totalPoints = rewardPrograms.reduce((sum, p) => sum + p.balance, 0)
// const portfolioValueMin = rewardPrograms.reduce((sum, p) => sum + (p.balance * (p.conservativeCpp ?? p.cppRange[0])), 0) / 100
// const portfolioValueMax = rewardPrograms.reduce((sum, p) => sum + (p.balance * (p.optimisticCpp ?? p.cppRange[1])), 0) / 100

// AFTER (CORRECT):
const { totalPoints, portfolioValueMin, portfolioValueMax } = getResolvedPortfolioStats(
  rewardPrograms, loyaltyMemberships, valuationMode
)
```

Add `loyaltyMemberships` and `valuationMode` to the useMemo dependency array (line 70):

```typescript
// BEFORE:
}, [cards, rewardPrograms])

// AFTER:
}, [cards, rewardPrograms, loyaltyMemberships, valuationMode])
```

#### Step 3: Fix topRewardPrograms to also use resolved balances

Replace lines 74-79:

```typescript
// BEFORE:
const topRewardPrograms = useMemo(() => {
  return [...rewardPrograms]
    .filter((p) => p.balance > 0)
    .map((p) => ({ ...p, estimatedValue: (p.balance * p.cppRange[1]) / 100 }))
    .sort((a, b) => b.estimatedValue - a.estimatedValue)
}, [rewardPrograms])

// AFTER:
const topRewardPrograms = useMemo(() => {
  const loyaltyProgramIds = new Set(loyaltyMemberships.map((m) => m.programId))
  const membershipTotals = new Map<string, number>()
  for (const m of loyaltyMemberships) {
    membershipTotals.set(m.programId, (membershipTotals.get(m.programId) ?? 0) + m.balance)
  }

  return [...rewardPrograms]
    .map((p) => {
      const resolvedBalance = loyaltyProgramIds.has(p.id)
        ? (membershipTotals.get(p.id) ?? 0)
        : p.balance
      return { ...p, balance: resolvedBalance, estimatedValue: (resolvedBalance * p.cppRange[1]) / 100 }
    })
    .filter((p) => p.balance > 0)
    .sort((a, b) => b.estimatedValue - a.estimatedValue)
}, [rewardPrograms, loyaltyMemberships])
```

### Tests

1. Unit test `getResolvedPortfolioStats()` with:
   - Mixed loyalty + card-linked programs
   - Edge case: loyalty program with zero balance
   - Edge case: no loyalty memberships at all
2. Verify dashboard total points matches transfer engine's calculation
3. Run `npm run build` — no TypeScript errors

### Acceptance Criteria

- [ ] Dashboard "Total Points" matches the sum from transfer recommendations page
- [ ] Dashboard "Portfolio Value" uses loyalty membership balances, not stale RewardProgram.balance
- [ ] Monthly snapshot and dashboard use the same canonical resolver
- [ ] No TypeScript errors, build passes

---

## Phase 2: Fix Cloud Sync Device-Level Key + Empty State (CRITICAL)

**Goal:** The sync initialization flag (`amplifypoints-cloud-synced`) is global per device, not per user. This causes cross-user data contamination on shared devices. Additionally, when the cloud returns empty results (user deleted all data elsewhere), local state is not cleared.

### Files to Modify

1. `src/lib/cloudSync.ts` — lines 39, 224-230
2. `src/hooks/useCloudSync.ts` (if it exists, or wherever `hasCompletedInitialSync` is called)

### Changes

#### Step 1: Make sync key user-scoped

```typescript
// BEFORE (line 39):
const SYNCED_KEY = 'amplifypoints-cloud-synced'

// AFTER:
function getSyncedKey(userId: string, householdId?: string): string {
  const scope = householdId ?? userId
  return `amplifypoints-cloud-synced-${scope}`
}
```

Update `hasCompletedInitialSync` and `markInitialSyncComplete`:

```typescript
// BEFORE:
export function hasCompletedInitialSync(): boolean {
  return localStorage.getItem(SYNCED_KEY) === 'true'
}
export function markInitialSyncComplete() {
  localStorage.setItem(SYNCED_KEY, 'true')
}

// AFTER:
export function hasCompletedInitialSync(userId: string, householdId?: string): boolean {
  return localStorage.getItem(getSyncedKey(userId, householdId)) === 'true'
}
export function markInitialSyncComplete(userId: string, householdId?: string) {
  localStorage.setItem(getSyncedKey(userId, householdId), 'true')
}
```

#### Step 2: Update all callers to pass userId/householdId

Search for all usages of `hasCompletedInitialSync()` and `markInitialSyncComplete()` and update them to pass the current user's ID and optional household ID from the sync scope.

#### Step 3: Handle empty cloud results

In the initial sync merge logic (likely in `useCloudSync.ts`), when cloud returns empty cards/programs:

```typescript
// When fetching cloud state for initial merge:
const cloudCards = await fetchCards(scope)
const cloudPrograms = await fetchRewardPrograms(scope)

// BEFORE (implicit): if cloudCards is empty, keep local state (no-op)

// AFTER: empty cloud = valid state, should clear local
if (cloudCards.length === 0 && cloudPrograms.length === 0) {
  // Cloud is intentionally empty — clear local state to match
  setCardsStore([])
  setRewardProgramsStore([])  // or keep seed programs based on product decision
} else {
  // Normal merge logic...
}
```

#### Step 4: Add offline queue deduplication

In the `enqueue` function:

```typescript
// BEFORE:
export function enqueue(op: SyncOp) {
  const q = readQueue()
  q.push(op)
  writeQueue(q)
}

// AFTER:
export function enqueue(op: SyncOp) {
  const q = readQueue()
  // Dedup: remove any existing op for the same entity
  const deduped = q.filter((existing) => {
    if (op.type === 'upsert_card' && existing.type === 'upsert_card') {
      return existing.card.id !== op.card.id
    }
    if (op.type === 'delete_card' && existing.type === 'delete_card') {
      return existing.cardId !== op.cardId
    }
    if (op.type === 'upsert_program' && existing.type === 'upsert_program') {
      return existing.program.id !== op.program.id
    }
    if (op.type === 'delete_program' && existing.type === 'delete_program') {
      return existing.programId !== op.programId
    }
    // Also: upsert supersedes prior delete for same entity, and vice versa
    if (op.type === 'upsert_card' && existing.type === 'delete_card') {
      return existing.cardId !== op.card.id
    }
    if (op.type === 'delete_card' && existing.type === 'upsert_card') {
      return existing.card.id !== op.cardId
    }
    if (op.type === 'upsert_program' && existing.type === 'delete_program') {
      return existing.programId !== op.program.id
    }
    if (op.type === 'delete_program' && existing.type === 'upsert_program') {
      return existing.program.id !== op.programId
    }
    return true
  })
  deduped.push(op)
  writeQueue(deduped)
}
```

### Tests

1. Unit test: different users get different sync keys
2. Unit test: enqueue deduplicates by entity ID
3. Integration test: empty cloud state clears local data
4. Run `npm run build`

### Acceptance Criteria

- [ ] Sync key includes user ID (and household ID when applicable)
- [ ] Switching users on same device triggers fresh initial sync
- [ ] Deleting all cards on device A shows empty state on device B after refresh
- [ ] Offline queue does not grow unboundedly with retries

---

## Phase 3: Security Fixes (HIGH)

**Goal:** Close the invite token vulnerability, add scope to deleteCard, fix household concurrency.

### Files to Modify

1. `supabase/migrations/` — new migration for accept_household_invite fix
2. `src/lib/cloudSync.ts` — deleteCard function
3. `supabase/migrations/` — new migration for member limit constraint

### Changes

#### Fix 1: Validate email in accept_household_invite RPC

Create new migration:

```sql
-- Migration: fix_accept_household_invite_email_validation.sql

CREATE OR REPLACE FUNCTION public.accept_household_invite(invite_token uuid)
RETURNS jsonb
LANGUAGE plpgsql
SECURITY DEFINER
AS $$
DECLARE
  v_invite RECORD;
  v_user_email text;
BEGIN
  -- Look up invite
  SELECT * INTO v_invite FROM household_invites
    WHERE token = invite_token AND status = 'pending' AND expires_at > now();

  IF NOT FOUND THEN
    RAISE EXCEPTION 'Invalid or expired invite token';
  END IF;

  -- NEW: Validate that accepting user's email matches invited_email
  SELECT email INTO v_user_email FROM auth.users WHERE id = auth.uid();
  IF v_user_email IS DISTINCT FROM v_invite.invited_email THEN
    RAISE EXCEPTION 'Email mismatch. This invite was sent to a different email address.';
  END IF;

  -- ... rest of existing logic (member count check, insert, etc.)
END;
$$;
```

#### Fix 2: Add scope filter to deleteCard

```typescript
// BEFORE (src/lib/cloudSync.ts line 133):
export async function deleteCard(cardId: string) {
  const { error } = await supabase
    .from('user_cards')
    .delete()
    .eq('id', cardId)
  if (error) throw error
}

// AFTER:
export async function deleteCard(cardId: string, scope?: SyncScope) {
  let query = supabase.from('user_cards').delete().eq('id', cardId)
  if (scope) {
    const { column, value } = scopeFilter(scope)
    query = query.eq(column, value)
  }
  const { error } = await query
  if (error) throw error
}
```

Update all callers of `deleteCard` to pass scope. In `deleteCardFromCloud`:

```typescript
export function deleteCardFromCloud(cardId: string) {
  if (!_scope) return
  _suppressRealtime?.()
  const scope = _scope
  if (isOnline()) {
    deleteCard(cardId, scope).catch(() => {
      enqueue({ type: 'delete_card', cardId })
    })
  } else {
    enqueue({ type: 'delete_card', cardId })
  }
}
```

Also update the `flushQueue` case for `delete_card` to pass scope (add scope to the SyncOp type).

#### Fix 3: Add database-level member limit constraint

```sql
-- Migration: add_household_member_limit_constraint.sql

-- Add a unique partial index to enforce max 2 active members per household
-- This prevents race conditions where concurrent accepts bypass the count check
CREATE UNIQUE INDEX IF NOT EXISTS idx_household_max_two_active
  ON household_members (household_id, user_id)
  WHERE status = 'active';

-- Also add a trigger for belt-and-suspenders enforcement:
CREATE OR REPLACE FUNCTION enforce_household_member_limit()
RETURNS trigger AS $$
BEGIN
  IF (SELECT count(*) FROM household_members
      WHERE household_id = NEW.household_id AND status = 'active') >= 2
  THEN
    RAISE EXCEPTION 'Household already has maximum 2 active members';
  END IF;
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_household_member_limit
  BEFORE INSERT OR UPDATE ON household_members
  FOR EACH ROW
  WHEN (NEW.status = 'active')
  EXECUTE FUNCTION enforce_household_member_limit();
```

### Tests

1. Test: accepting invite with wrong email returns error
2. Test: deleteCard with scope only deletes within scope
3. Test: concurrent accepts do not create 3-member household
4. Run full E2E suite

### Acceptance Criteria

- [ ] Invite accept verifies email matches
- [ ] deleteCard includes household_id filter
- [ ] Database enforces max 2 active members per household
- [ ] Existing tests still pass

---

## Phase 4: Fix Chase Freedom Race Condition + Recommendation Engine (HIGH)

**Goal:** Fix the useEffect dependency array for Chase Freedom auto-switch, and make the recommendation engine actually use the stats parameter.

### Files to Modify

1. `src/hooks/useAmplifyPoints.ts` — lines 93-106
2. `src/lib/utils.ts` — `generateRecommendations()` function

### Changes

#### Fix 1: Add cards to Chase Freedom dependency array

```typescript
// BEFORE (line 106):
}, [hasSapphire, isInitialized])

// AFTER:
}, [hasSapphire, isInitialized, cards, updateCardStore])
```

Also add a user-visible notification:

```typescript
useEffect(() => {
  if (!isInitialized) return
  const targetProgram = hasSapphire ? 'chase_ur' : 'cashback'
  cards.forEach(card => {
    if (
      card.bank === 'Chase' &&
      card.cardName.toLowerCase().includes('freedom') &&
      (card.rewardsProgram === 'chase_ur' || card.rewardsProgram === 'cashback') &&
      card.rewardsProgram !== targetProgram
    ) {
      updateCardStore(card.id, { rewardsProgram: targetProgram })
      // TODO: Add toast notification here:
      // toast.info(`${card.cardName} rewards program updated to ${targetProgram === 'chase_ur' ? 'Chase Ultimate Rewards' : 'Cashback'}`)
    }
  })
}, [hasSapphire, isInitialized, cards, updateCardStore])
```

#### Fix 2: Make generateRecommendations use stats

```typescript
// BEFORE: _stats parameter is completely ignored

// AFTER: Use stats for personalized thresholds
export function generateRecommendations(cards: CreditCard[], stats: AmplifyPointsStats): StrategyRecommendation[] {
  const recommendations: StrategyRecommendation[] = [];

  // Use stats to personalize high-fee threshold
  // If total annual fees are high relative to card count, lower the threshold
  const avgFeePerCard = stats.totalCards > 0 ? stats.totalAnnualFees / stats.totalCards : 0
  const highFeeThreshold = avgFeePerCard > 300 ? 350 : 400

  cards.forEach(card => {
    if (card.annualFee > highFeeThreshold && card.keepCancelStatus === 'keep') {
      recommendations.push({
        id: `high-fee-${card.id}`,
        type: 'cancel',
        title: 'Review high-fee card',
        description: `${card.cardName} has a ${formatCurrency(card.annualFee)} annual fee (your average is ${formatCurrency(avgFeePerCard)}). Review if benefits outweigh the cost.`,
        cardId: card.id,
        cardName: card.cardName,
        dismissed: false,
      });
    }

    // ... rest of existing logic
  });

  // Use stats for portfolio-level recommendations
  if (stats.totalPoints >= 100_000 && stats.portfolioValueMax > 2000) {
    recommendations.push({
      id: 'portfolio-review',
      type: 'optimize',
      title: 'Large portfolio — consider transfers',
      description: `You have ${formatPoints(stats.totalPoints)} points worth up to ${formatCurrency(stats.portfolioValueMax)}. Review transfer opportunities to maximize value.`,
      dismissed: false,
    });
  }

  // ... rest of existing logic
  return recommendations.slice(0, 6);
}
```

#### Fix 3: Switch Chase 5/24 to prefer applicationDate

```typescript
// BEFORE:
const cardsOpenedLast24Months = cards.filter(card => {
  if (!card.dateOpened) return false
  const opened = parseISO(card.dateOpened)
  return isValid(opened) && isAfter(opened, cutoff) && card.cardType !== 'authorized_user'
}).length;

// AFTER:
const cardsOpenedLast24Months = cards.filter(card => {
  // Prefer applicationDate when available (more accurate for 5/24 counting)
  const resolvedDate = resolveApplicationDate(card)
  const dateStr = resolvedDate ? format(resolvedDate, 'yyyy-MM-dd') : card.dateOpened
  if (!dateStr) return false
  const parsed = parseISO(dateStr)
  return isValid(parsed) && isAfter(parsed, cutoff) && card.cardType !== 'authorized_user'
}).length;
```

Import `resolveApplicationDate` at the top of the file if not already imported.

### Tests

1. Test: Adding a Freedom card after initial load triggers auto-switch
2. Test: generateRecommendations produces different output based on stats
3. Test: 5/24 count uses applicationDate when available
4. Run `npm run build`

### Acceptance Criteria

- [ ] Freedom cards added after Sapphire correctly switch to chase_ur
- [ ] Recommendations reference the user's actual stats
- [ ] 5/24 count uses application date when confidence is 'exact' or 'month'

---

## Phase 5: Household Data Integrity (HIGH)

**Goal:** Fix orphaned data on household departure, add invite rate limiting, handle household create/render bug found in E2E.

### Files to Modify

1. `src/hooks/useHousehold.ts` — leaveHousehold function
2. `supabase/migrations/` — new migration for departure cleanup
3. `supabase/migrations/` — new migration for invite rate limit
4. Investigate and fix household creation rendering bug (household-live.spec.ts failure)

### Changes

#### Fix 1: Clean up data on household departure

In `leaveHousehold()`, before marking the member as 'left', migrate their data to individual scope:

```typescript
const leaveHousehold = useCallback(async () => {
  if (!household || !user) return { error: 'No household' }

  // Step 1: Copy user's cards to individual scope before leaving
  try {
    const { data: myCards } = await supabase
      .from('user_cards')
      .select('card_data')
      .eq('household_id', household.id)
    // Filter to cards belonging to this user (by cardholder field in card_data)
    // and re-insert with user_id scope
    if (myCards?.length) {
      for (const row of myCards) {
        const card = row.card_data as CreditCard
        if (card.cardholder?.toLowerCase() === profile?.display_name?.toLowerCase()) {
          await supabase.from('user_cards').upsert({
            id: card.id,
            user_id: user.id,
            household_id: null,
            card_data: card,
          }, { onConflict: 'id' })
        }
      }
    }
  } catch (err) {
    console.warn('Data migration on leave failed:', err)
    // Continue with leave even if migration fails
  }

  // Step 2: Mark member as left
  const { error } = await supabase
    .from('household_members')
    .update({ status: 'left', left_at: new Date().toISOString() })
    .eq('household_id', household.id)
    .eq('user_id', user.id)

  if (error) return { error: error.message }

  await fetchHousehold()
  return { error: null }
}, [household, user, profile, fetchHousehold])
```

#### Fix 2: Add invite rate limiting

```sql
-- Migration: add_invite_rate_limit.sql

-- Add a function to check invite rate before insert
CREATE OR REPLACE FUNCTION check_invite_rate_limit()
RETURNS trigger AS $$
BEGIN
  IF (
    SELECT count(*) FROM household_invites
    WHERE invited_by = NEW.invited_by
      AND created_at > now() - interval '24 hours'
  ) >= 5 THEN
    RAISE EXCEPTION 'Rate limit: maximum 5 invites per 24 hours';
  END IF;
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_invite_rate_limit
  BEFORE INSERT ON household_invites
  FOR EACH ROW
  EXECUTE FUNCTION check_invite_rate_limit();
```

#### Fix 3: Debug household creation rendering

The live E2E test shows that after submitting "Create & Send Invite", the household name never becomes visible. Likely causes:

1. `fetchHousehold()` is called but the INSERT hasn't committed yet
2. The `createHousehold` function calls `fetchHousehold()` but the state update hasn't propagated to the UI

**Investigation steps:**
- Check if `createHousehold` awaits the full insert chain
- Verify that `fetchHousehold()` is called AFTER both household insert AND member insert succeed
- Check if there's a race between `createHousehold` return and `inviteMember` call in the wizard
- Look at the wizard component to see if it calls createHousehold + inviteMember atomically

**Likely fix:** The wizard may call `createHousehold` then immediately call `inviteMember` before `fetchHousehold` completes. Need to await the full chain:

```typescript
// In the wizard component (likely OnboardingWizard or HouseholdSetup):
const result = await createHousehold(name, isLegalSpouse)
if (result.error) { /* handle */ return }
// MUST wait for household state to update before inviting
// Pass householdId explicitly instead of relying on state:
const inviteResult = await inviteMember(email, result.householdId, name)
```

### Tests

1. E2E test: leaving household migrates user's cards to individual scope
2. Test: 6th invite in 24 hours is rejected
3. Re-run `household-live.spec.ts` after fixing the rendering bug

### Acceptance Criteria

- [ ] Departing user's cards are copied to individual scope
- [ ] Invite creation is rate-limited to 5 per 24 hours
- [ ] `household-live.spec.ts` passes
- [ ] No orphaned data after household dissolution

---

## Phase 6: Error Handling & Resilience (MEDIUM)

**Goal:** Replace silent catch blocks with proper error handling, add auth retry logic, add missing Sentry integration.

### Files to Modify

1. `src/contexts/AuthContext.tsx` — add retry with backoff for profile fetch
2. Multiple files — standardize error handling
3. `src/lib/cloudSync.ts` — add error context to catch blocks

### Changes

#### Fix 1: Add retry logic to profile fetch

```typescript
// In AuthContext.tsx, replace the profile fetch:

async function fetchProfileWithRetry(userId: string, retries = 3): Promise<Profile | null> {
  for (let attempt = 1; attempt <= retries; attempt++) {
    try {
      const timeout = new Promise<null>((resolve) =>
        setTimeout(() => resolve(null), 8000 * attempt)  // Increase timeout per attempt
      )
      const fetch = supabase
        .from('profiles')
        .select('*')
        .eq('id', userId)
        .single()
        .then(({ data }) => data)

      const result = await Promise.race([fetch, timeout])
      if (result) return result

      if (attempt < retries) {
        console.warn(`Profile fetch attempt ${attempt} timed out, retrying...`)
        await new Promise(resolve => setTimeout(resolve, 1000 * attempt))  // Backoff
      }
    } catch (err) {
      if (attempt === retries) {
        captureAuthError(`fetchProfile failed after ${retries} attempts`, err)
      }
    }
  }
  return null
}
```

#### Fix 2: Create standardized error capture utility

Create `src/lib/errorCapture.ts`:

```typescript
/**
 * Standardized error capture for use across the app.
 * Logs to console in dev, sends to Sentry in production.
 */
export function captureError(context: string, error: unknown): void {
  const message = error instanceof Error ? error.message : String(error)
  console.error(`[${context}]`, message, error)

  // When Sentry is configured:
  // if (typeof Sentry !== 'undefined') {
  //   Sentry.captureException(error, { tags: { context } })
  // }
}
```

#### Fix 3: Replace silent catch blocks

Search for all instances of `catch {}` or `catch { return false }` and replace with:

```typescript
// BEFORE:
} catch {
  return false
}

// AFTER:
} catch (err) {
  captureError('useAmplifyPoints.importData', err)
  return false
}
```

Files to update (12+ catch blocks):
- `src/hooks/useAmplifyPoints.ts` (importData)
- `src/lib/cloudSync.ts` (readQueue, flushQueue)
- `src/hooks/useBalanceGate.ts` (ackMonthly)
- `src/hooks/useHousehold.ts` (fetchHousehold)
- `src/contexts/AuthContext.tsx` (multiple)
- `src/stores/useCardsStore.ts` (rehydration)

### Tests

1. Test: Profile fetch retries 3 times before giving up
2. Test: captureError logs context + message
3. Verify no silent swallowed errors remain (grep for `catch {` and `catch(`)

### Acceptance Criteria

- [ ] Profile fetch retries with exponential backoff
- [ ] All catch blocks log error context
- [ ] Error capture utility is used consistently
- [ ] App shows user-friendly message when profile fetch fails entirely

---

## Phase 7: UX & Polish (MEDIUM)

**Goal:** Fix the Travel tab stub, improve accessibility, consolidate duplicated code.

### Files to Modify

1. `src/pages/TravelPage.tsx` — either move trip goals here or remove from nav
2. `src/components/BottomNav.tsx` — increase touch targets
3. `src/components/tabs/DashboardTab.tsx` + `src/components/tabs/RewardsTab.tsx` — consolidate TPG_CPP
4. Various modal components — add focus traps and ARIA attributes

### Changes

#### Fix 1: Consolidate TPG_CPP

Create `src/data/valuations.ts`:

```typescript
/** TPG-style cents-per-point valuations. Single source of truth. */
export const TPG_CPP: Record<string, number> = {
  chase_ur: 2.0,
  amex_mr: 2.0,
  citi_typ: 1.8,
  capital_one: 1.85,
  delta_skymiles: 1.2,
  american_airlines: 1.5,
  united_mileageplus: 1.35,
  southwest_rr: 1.4,
  jetblue_trueblue: 1.3,
  alaska_mileageplan: 1.8,
  hyatt: 2.3,
  marriott_bonvoy: 0.8,
  hilton_honors: 0.5,
  ihg_one_rewards: 0.5,
}
```

Replace the duplicated TPG_CPP in DashboardTab and RewardsTab with:

```typescript
import { TPG_CPP } from '@/data/valuations'
```

#### Fix 2: Increase bottom nav touch targets

```typescript
// In BottomNav.tsx, change nav button sizing:
// BEFORE: className="... p-1 ..." (results in ~20px touch target)
// AFTER:  className="... p-3 min-h-[44px] min-w-[44px] ..." (44px minimum)
```

#### Fix 3: Travel tab — move trip goals here

Instead of the "Coming soon" stub, move the trip goal management from the Dashboard modal into the Travel tab. The trip goal components already exist; they just need to be relocated.

#### Fix 4: Add aria attributes

Add to all collapsible/expandable elements:
```typescript
aria-expanded={isExpanded}
aria-controls="content-id"
```

Add to all icon-only buttons:
```typescript
aria-label="descriptive label"
```

### Tests

1. Verify TPG_CPP is imported from single source
2. Test touch targets are >= 44px with Playwright viewport
3. Test Travel tab renders trip goals
4. Accessibility audit with axe-core

### Acceptance Criteria

- [ ] No duplicated TPG_CPP constants
- [ ] Bottom nav touch targets >= 44px
- [ ] Travel tab is functional (not a stub)
- [ ] ARIA attributes on interactive elements

---

## Phase 8: Test Coverage (MEDIUM)

**Goal:** Write tests for the highest-risk untested areas.

### Files to Create

1. `src/lib/__tests__/transferRecommendations.test.ts`
2. `src/lib/__tests__/balanceResolver.test.ts`
3. `src/lib/__tests__/cloudSync.test.ts`
4. `src/hooks/__tests__/useBalanceGate.test.ts`
5. `tests/e2e/live/onboarding-live.spec.ts`

### Test Specifications

#### transferRecommendations.test.ts

```typescript
describe('getTransferRecommendations', () => {
  it('recommends transfer when effective CPP exceeds cash CPP')
  it('does not recommend transfer when effective CPP is lower')
  it('returns best single destination per source program')
  it('handles empty programs array')
  it('handles zero-balance programs')
  it('flags transfer fees when present')
  it('flags expiry warnings for short-lived points')
})

describe('getJointStrategy', () => {
  it('returns native_pool strategy for supported programs')
  it('returns au_workaround strategy for AU programs')
  it('returns primary_saver when combined >= 100K for fee_transfer programs')
  it('returns parallel_savers when combined < 100K for fee_transfer programs')
  it('returns guidance even when only one person has membership')
  it('returns null when no memberships exist')
})

describe('captureMonthlySnapshot', () => {
  it('does not double-count loyalty membership programs')
  it('is idempotent for same month')
  it('caps at 36 months')
  it('uses correct CPP based on valuation mode')
})
```

#### cloudSync.test.ts

```typescript
describe('enqueue', () => {
  it('deduplicates upsert_card by card ID')
  it('delete supersedes prior upsert for same entity')
  it('upsert supersedes prior delete for same entity')
})

describe('getSyncedKey', () => {
  it('includes userId in key')
  it('includes householdId when provided')
  it('different users get different keys')
})
```

### Acceptance Criteria

- [ ] Transfer recommendation engine has > 80% branch coverage
- [ ] Cloud sync dedup logic has unit tests
- [ ] Balance resolver has unit tests proving no double-counting
- [ ] All new tests pass

---

## Phase 9: Card Catalog Completion (MEDIUM)

**Goal:** The card catalog is only ~48% complete (26 of 54 referenced cards). Complete the missing entries.

### Files to Modify

1. `src/data/cardCatalog.ts`
2. `src/data/referralLinks.ts` — fix 3 mismatched IDs

### Missing Cards to Add

Based on the GPT-4o audit, these categories are missing or incomplete:

1. **Capital One cards:** Venture X, SavorOne, Quicksilver, Venture
2. **Bank of America cards:** Premium Rewards, Customized Cash, Travel Rewards
3. **Barclays cards:** AAdvantage Aviator Red, JetBlue Plus, Hawaiian Airlines
4. **Business cards:** Chase Ink Business Preferred, Ink Cash, Ink Unlimited, Amex Business Gold, Amex Business Platinum, Capital One Spark
5. **Other missing:** Citi Custom Cash, Citi Double Cash, Discover it, Bilt Mastercard (if not present)

Each card entry needs:
- `id`: unique kebab-case identifier
- `name`: full marketing name
- `issuer`: bank name
- `annualFee`: number
- `rewardsProgram`: linked program ID
- `cppRange`: [conservative, optimistic]
- `categories`: earn rate categories
- `signupBonus`: (if applicable)

### Referral Link Fixes

Fix the 3 mismatched referral IDs:
- `amex_bce` → match to actual catalog ID for Blue Cash Everyday
- `capitalone_venture_x` → match to actual catalog ID for Venture X
- `amex_hilton_honors` → match to actual catalog ID for Hilton Honors card

### Acceptance Criteria

- [ ] All 54 referenced cards have catalog entries
- [ ] All referral link IDs match catalog IDs
- [ ] No broken references between catalog and referral data
- [ ] Cross-reference unit test passes (if created in Phase 8)

---

## Phase Summary

| Phase | Focus | Severity | Effort | Score Impact |
|-------|-------|----------|--------|-------------|
| 1 | Dashboard balance source-of-truth | CRITICAL | 1-2 days | +8-10 points |
| 2 | Cloud sync key + empty state | CRITICAL | 1-2 days | +6-8 points |
| 3 | Security fixes (invite, delete, concurrency) | HIGH | 1-2 days | +4-5 points |
| 4 | Chase Freedom + recommendation engine | HIGH | 0.5-1 day | +3-4 points |
| 5 | Household data integrity | HIGH | 1-2 days | +3-4 points |
| 6 | Error handling & resilience | MEDIUM | 1-2 days | +2-3 points |
| 7 | UX & polish | MEDIUM | 1-2 days | +2-3 points |
| 8 | Test coverage | MEDIUM | 2-3 days | +2-3 points |
| 9 | Card catalog completion | MEDIUM | 2-3 days | +1-2 points |

**Total estimated effort: 11-19 developer-days**
**Projected score after all phases: 83-92/100**

---

## Appendix: Files Most Frequently Referenced

| File | Phases | Role |
|------|--------|------|
| `src/hooks/useAmplifyPoints.ts` | 1, 4 | Central data hook; stats, alerts, recommendations |
| `src/lib/cloudSync.ts` | 2, 3, 6 | Cloud sync CRUD, offline queue, scope management |
| `src/lib/transferRecommendations.ts` | 1, 8 | Transfer engine, balance resolution, snapshots |
| `src/lib/utils.ts` | 4 | Date utilities, alert generation, recommendations |
| `src/hooks/useHousehold.ts` | 3, 5 | Household CRUD, invite flow, departure |
| `src/contexts/AuthContext.tsx` | 6 | Auth, profile fetch, session management |
| `src/stores/useCardsStore.ts` | 2 | Card store, migrations, persistence |
| `src/stores/useRewardsStore.ts` | 1 | Reward programs, loyalty memberships, snapshots |
| `src/components/tabs/DashboardTab.tsx` | 1, 7 | Dashboard UI, portfolio display |
| `src/components/BottomNav.tsx` | 7 | Navigation, touch targets |
| `src/data/cardCatalog.ts` | 9 | Card product definitions |
| `supabase/migrations/` | 3, 5 | RPC functions, RLS policies, constraints |
