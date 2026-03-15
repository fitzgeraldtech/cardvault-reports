# CardVault Logic Review

This document translates the project logic into plain English.

Scope:
- What the app calculates
- What conditions trigger behavior
- Where math and decision thresholds come from
- Where the current logic looks risky or inconsistent

Important note:
- Most of the "backend math" in this repo is actually client-side logic in React hooks and utility modules.
- The Supabase backend is mostly persistence, auth, and sync. The business rules are primarily in `src/lib/*.ts` and `src/hooks/*.ts`.

## 1. Core Date Logic

### `calculateDaysRemaining(dateString)`

If:
- `dateString` is empty
- or `dateString` cannot be parsed

Then:
- return `9999`
- this acts as "no meaningful deadline"

Else:
- parse the date
- set today to local midnight
- return `differenceInCalendarDays(target, today)`

Implication:
- Negative number = already passed
- `0` = today
- Large sentinel `9999` is used instead of `null`

### Relative date formatting

If `daysRemaining >= 9999`:
- show `None`

If `daysRemaining < 0`:
- show `X days ago`

If `daysRemaining === 0`:
- show `Today`

If `daysRemaining === 1`:
- show `Tomorrow`

If `daysRemaining <= 30`:
- show `in X days`

If `daysRemaining <= 60`:
- show `in floor(days / 7) weeks`

Else:
- show `in floor(days / 30) month(s)`

## 2. Dashboard / Portfolio Math

Source: `useCardVault()`

### Active-card counts

Active cards are:
- cards where `status === 'active'`

Then:
- `totalCards` = number of active cards
- `chrisCards` = active cards where `cardholder === 'Chris'`
- `vanessaCards` = active cards where `cardholder === 'Vanessa'`

### Annual-fee totals

Using active cards only:
- `totalAnnualFees` = sum of `annualFee`
- `chrisFees` = sum of Chris active-card `annualFee`
- `vanessaFees` = sum of Vanessa active-card `annualFee`

### Points totals

`totalPoints`:
- sum of every `rewardProgram.balance`

Important limitation:
- This does not honor the more nuanced transfer engine rule that some balances are supposed to come from loyalty memberships instead of `RewardProgram.balance`.
- That means dashboard-level totals can diverge from transfer/strategy math if loyalty programs are tracked separately.

### Portfolio value

For each reward program:
- conservative value contribution = `balance * (conservativeCpp ?? cppRange[0])`
- optimistic value contribution = `balance * (optimisticCpp ?? cppRange[1])`

Then:
- divide by `100` to convert cents-per-point into dollars

Formulas:
- `portfolioValueMin = sum(balance * conservativeCpp) / 100`
- `portfolioValueMax = sum(balance * optimisticCpp) / 100`

### Cards needing action

A card counts as needing action if:
- card is not closed
- and either:
  - `daysRemaining <= 60`
  - or `keepCancelStatus === 'cancel'`

### Upcoming fees

Included if:
- `annualFee > 0`
- and `daysRemaining <= 120`

Then:
- sort ascending by `daysRemaining`

### Fees due in next 12 months

Using active cards only:

If:
- `annualFee > 0`
- and `nextFeeDate` exists
- and `0 <= daysRemaining <= 365`

Then:
- include that `annualFee` in `upcomingFeesNext12Months`

### Lifetime signup bonus value

Using active cards only:
- sum `enrollBonusValue ?? 0`

Important limitation:
- This is not realized value, cash value, or current available value.
- It is just a raw sum of the stored signup-bonus estimates on active cards.

## 3. Alert Logic

Source: `generateAlerts(cards)`

Severity rules:
- if `daysRemaining < 60` -> `red`
- else if `daysRemaining <= 120` -> `amber`
- else -> `green`

### Annual fee alert

Create alert if:
- `annualFee > 0`
- and `nextFeeDate` exists

Then:
- alert type = `annual_fee`
- description = `Annual fee of $X due`

Important:
- No upper time cap
- A fee 300 days out still creates an alert

### Cancel deadline alert

Create alert if:
- `status === 'active'`
- and `keepCancelStatus === 'cancel'`

Then:
- alert type = `cancel_deadline`
- description depends on fee:
  - if annual fee > 0 -> `Cancel before annual fee posts`
  - else -> `Marked for cancellation`

Important:
- No days limit here either

### Welcome bonus deadline alert

Create alert if:
- `status === 'active'`
- `subStatus` is one of:
  - `targeting`
  - `on_track`
  - `not_sure`
  - `worried`
- and `subDeadlineDate` exists

Then:
- alert type = `bonus_deadline`
- if `subSpendRequirement` exists:
  - description = `Spend $X to earn Y points`
- else:
  - description = `Complete welcome bonus spend requirement`

### Year-1 fee-waiver expiry alert

Create alert if:
- `annualFeeWaivedYear1 === true`
- and `annualFee > 0`

Then determine `firstChargeDate`:
- if `annualFeeFirstChargeDate` exists, use it
- else if `dateOpened` is valid, use `dateOpened + 1 year`

Then if:
- `0 <= daysToFirstCharge <= 45`

Create alert:
- type = `fee_waiver_expiry`
- description explains first-year waiver expires soon

## 4. Recommendation Logic

Source: `generateRecommendations(cards, stats)`

### High-fee keep recommendation

If:
- `annualFee > 400`
- and `keepCancelStatus === 'keep'`

Then:
- recommend reviewing/canceling the card

This is a blunt threshold.
- It does not compare annual fee to benefit value
- It does not inspect actual usage

### Cancel-action recommendation

If:
- `keepCancelStatus === 'cancel'`

Then:
- create a recommendation saying action is required
- wording depends on `daysRemaining`

Cases:
- if `daysRemaining >= 9999` -> `No fee date set.`
- else if `daysRemaining > 0` -> `Cancel within X days to avoid the fee.`
- else -> `Fee may have already posted.`

### Chase 5/24 status

The code counts cards opened in the last 24 months.

A card counts if:
- `dateOpened` exists
- parsed date is valid
- `opened > now - 24 months`
- `cardType !== 'authorized_user'`

Then:
- if count >= 4, emit a recommendation
- if count == 4 -> approaching 5/24
- if count >= 5 -> already at 5/24

Important concern:
- This uses `dateOpened`, not true application date
- Citi/Chase-style eligibility often depends on application/bonus timing, not merely open date

### Companion Pass recommendation

The function also always injects a generic companion-pass recommendation.

This means:
- it is not conditional on Southwest cards
- it is more of a standing suggestion than calculated advice

## 5. Chase Freedom Rule

Source: `useCardVault()`

### Rule

If any active card satisfies all three:
- `bank === 'Chase'`
- `status === 'active'`
- `cardName` contains `sapphire`

Then:
- all Chase Freedom cards should use `rewardsProgram = 'chase_ur'`

Else:
- Chase Freedom cards should use `rewardsProgram = 'cashback'`

### Affected cards

A card gets rewritten if:
- `bank === 'Chase'`
- `cardName` contains `freedom`
- current `rewardsProgram` is either `chase_ur` or `cashback`
- current program differs from the target

Important concern:
- This is automatic mutation logic
- It runs on initialization and on Sapphire presence changes
- It assumes all Freedom products should flip as a group

## 6. Application-Date and Reapplication Logic

Source: `utils.ts`

### Application date resolution

If `applicationDateConfidence === 'exact'`:
- return `applicationDate`

If `applicationDateConfidence === 'month'`:
- return last day of `applicationMonth`

If `applicationDateConfidence === 'year'`:
- return December 31 of `applicationYear`

If `applicationDateConfidence === 'unknown'` or missing:
- return `null`

Meaning:
- the app intentionally uses conservative estimates
- month-level data resolves to end-of-month
- year-level data resolves to end-of-year

### SUB received-date estimate

If:
- `approvalDate ?? dateOpened` exists
- and `subDeadlineMonths` exists

Then:
- estimated SUB received date = `openDate + subDeadlineMonths + 30 days`

Else:
- return `null`

Important concern:
- This is heuristic math, not actual receipt tracking
- Any downstream logic using this should be treated as approximate

### Issuer reapplication rules

Static rules:
- Chase -> 48 months from SUB received
- American Express -> lifetime
- Citi -> 24 months
- Capital One -> 6 months
- Bank of America -> none documented
- Barclays -> 24 months
- Wells Fargo -> none documented

Important concern:
- These are hardcoded guidance values
- They are not dynamically sourced from issuer terms

## 7. SUB Check-In Logic

Source: `useSubCheckins()`

This is a cadence engine for prompting follow-up on welcome bonuses.

### A card is eligible for check-in if

All must be true:
- `status === 'active'`
- `subStatus` is one of:
  - `targeting`
  - `on_track`
  - `not_sure`
  - `worried`
- `subDeadlineDate` exists
- `daysRemaining >= 0`
- and `isCheckinDue(card.id, daysRemaining)` is true

### Check-in cadence

If no prior local record exists:
- check-in is due immediately

If prior record state is terminal:
- `achieved`
- `achieved_early`
- `missed`

Then:
- no further prompts

Else cadence rules:
- `not_sure` -> every 7 days
- `worried` -> every 3 days
- `on_track` -> every `floor(daysToDeadline / 2)`, minimum 1 day
- default / `targeting` -> every 30 days

### Dismiss behavior

When dismissed with a new state:
- write `{ date, state }` to localStorage
- update the card’s `subStatus`
- move to next queued check-in

### Mark done behavior

When marked done:
- write localStorage record as `achieved`
- update card:
  - `subStatus = 'achieved'`
  - `subReceived = true`
  - `subReceivedDate = date`

Important concern:
- The due list is only computed on mount and window focus
- It does not automatically recompute on every card change

## 8. Transfer Recommendation Math

Source: `transferRecommendations.ts`

## Balance source rules

The app has two balance systems:

### Loyalty-membership programs

Examples in comments:
- Delta
- AA
- Hyatt
- JetBlue
- Alaska

Rule:
- derive balances from `loyaltyMemberships[]`
- do not trust `RewardProgram.balance` for these

### Card-linked currencies

Examples in comments:
- Chase UR
- Amex MR
- Citi TYP
- Capital One

Rule:
- use `RewardProgram.balance`

This split is important because some screens follow it and some do not.

### Best transfer destination

For each currency program with `balance > 0`:
- find all transfer relationships where `fromProgramId` matches
- inspect each destination program

For each candidate route:
- `destCpp` = conservative or optimistic CPP for destination
- `effectiveCpp = destCpp * (toMultiplier / fromMultiplier)`
- `cashCpp` = conservative or optimistic CPP for source currency

If:
- `effectiveCpp <= cashCpp`

Then:
- discard the route

Else:
- compute transfer value:
  - `transferValueDollars = round(balance * effectiveCpp / 100)`
- compute cash value:
  - `cashValueDollars = round(balance * cashCpp / 100)`

Then:
- keep only the single best route per source currency
- best means highest `transferCpp`

Finally:
- sort recommendations by `transferValueDollars` descending

Important concern:
- This is static cents-per-point optimization
- It does not model award availability, taxes, booking constraints, or actual itinerary pricing

### Per-person balances

If program is represented in loyalty memberships:
- sum those membership balances
- optionally filter by Chris or Vanessa

Else if it is a household currency program:
- take the `RewardProgram.balance`

Important concern:
- Some views can show household totals for currencies and per-person totals for loyalty programs in the same dataset
- This is intentional, but easy to misunderstand

### Monthly snapshot math

If snapshot already exists for current `yyyy-MM`:
- return unchanged

Else:
- compute loyalty value from `loyaltyMemberships`
- compute card-linked value from `RewardProgram.balance`
- avoid double counting by excluding loyalty program IDs from the second pass
- save `totalValue` rounded to cents
- cap history at 36 months

### Month-over-month delta

If fewer than 2 snapshots:
- return `0`

Else:
- sort by month descending
- return `current.totalValue - previous.totalValue`

### Joint strategy logic

Look at `poolingType` on the reward program.

If `poolingType === 'native_pool'`:
- always show strategy if there is at least one membership
- recommendation = pool points natively

If `poolingType === 'au_workaround'`:
- always show strategy if there is at least one membership
- recommendation = combine via authorized-user workaround

If `poolingType === 'fee_transfer'` or default:
- require both Chris and Vanessa memberships
- compute combined balance
- if combined balance >= 100,000:
  - strategy = `primary_saver`
- else:
  - strategy = `parallel_savers`

Important concern:
- The 100,000 threshold is a hardcoded planning rule, not dynamic optimization

## 9. Referral Optimizer Logic

Source: `referralUtils.ts`

For a given `catalogCardId`:

If no referral and no elevated public offer:
- recommendation = `no_data`

If referral exists and elevated does not:
- recommendation = `use_referral`

If elevated exists and referral does not:
- recommendation = `use_elevated_public`

If both exist:
- if `referral.referralPoints >= elevated.points`
  - recommendation = `use_referral`
- else
  - recommendation = `use_elevated_public`

Important concern:
- This compares raw points only
- It does not compare statement credits, annual-fee timing, minimum spend, or approval odds

## 10. Balance Gate Logic

Source: `useBalanceGate()`

This is behavioral gating, not financial math.

### Initial gate

If:
- `cardsCount > 0`

Then on first local initialization:
- force `GATE_KEY = true`
- treat the gate as acknowledged

Else:
- gate is acknowledged only if localStorage says so

### One-time profile sync

If:
- `profileBalanceStatus === 'carries_balance'`
- and `PROFILE_SYNCED_KEY` is not already true
- and `DISABLED_KEY` is not already set

Then:
- set `PROFILE_SYNCED_KEY = true`
- set `DISABLED_KEY = true`
- set `GATE_KEY = true`

### Visibility

Show intro gate if:
- `!gateAcked`
- and `cardsCount === 0`

Show monthly check-in if:
- `gateAcked`
- and not checked in for current month

### Monthly ack

If user says they are paying in full:
- mark current month checked in
- remove `DISABLED_KEY`

If user says they need help:
- set `DISABLED_KEY`
- set `GATE_KEY`

Important concern:
- This is device-local state with partial profile sync
- Different browsers may behave differently until profile and localStorage converge

## 11. Cloud Sync / Data-Movement Logic

This is not financial math, but it strongly affects what data the math runs on.

### Scope mode

If user is in a household:
- sync scope = household

Else:
- sync scope = individual

### First sync

If local `amplifypoints-cloud-synced` is not true:
- fetch cloud cards and programs
- merge local-only items into cloud
- write merged data back into local state
- mark sync complete locally

Else:
- fetch cloud and treat it as source of truth

Important concern:
- This flag is device-level, not user-level or household-level

### Realtime sync suppression

When app writes its own cloud changes:
- suppress realtime refresh for 2 seconds

Purpose:
- avoid immediate echo updates from Supabase realtime

### Household migration

If scope changes from individual -> household:
- fetch individual cloud data
- copy into household scope
- delete old individual rows

If scope changes from household -> individual:
- copy current local cards/programs into individual scope

Important concern:
- Any mistake here changes the dataset feeding all downstream calculations

## 12. Known Logic Risks

These are the parts most worth auditing if you think the math is "off."

### Risk 1: inconsistent balance source rules

Different parts of the app use different balance sources:
- dashboard totals use `rewardPrograms.balance`
- transfer logic uses a split between `rewardPrograms.balance` and `loyaltyMemberships`

Result:
- totals and strategy recommendations may not line up perfectly

### Risk 2: dashboard value math is simplistic

Portfolio values are:
- static balances
- multiplied by static CPP assumptions

They are not:
- market prices
- actual redemption quotes
- award-search-derived values

### Risk 3: recommendation rules are threshold-based, not optimization-based

Examples:
- annual fee > 400
- Chase 5/24 warning at 4+
- joint strategy threshold at 100,000

These are simple rules, not adaptive calculations.

### Risk 4: dateOpened is used where application or SUB timing may matter more

Several card-eligibility ideas are approximated using:
- `dateOpened`
- inferred dates
- confidence buckets

That is useful, but not precise.

### Risk 5: cloud sync bugs can look like math bugs

If stale cards or reward programs remain in local state:
- totals
- alert counts
- fee forecasts
- transfer values

will all appear wrong even if the formulas themselves are correct.

## 13. Practical Audit Order

If you want to verify correctness efficiently, audit in this order:

1. Confirm the source data is right.
   - cards
   - balances
   - annual fees
   - next fee dates
   - reward CPP assumptions

2. Confirm which balance source each screen uses.
   - `RewardProgram.balance`
   - `loyaltyMemberships`
   - merged household data

3. Confirm sentinel cases.
   - empty dates become `9999`
   - negative dates mean overdue
   - empty cloud fetches do not always clear local state

4. Confirm threshold logic.
   - 60 / 120 / 365 day cutoffs
   - 45-day fee-waiver window
   - 4+ count for 5/24 warning
   - 100,000 threshold for joint strategy

5. Confirm whether a "wrong number" is:
   - bad source data
   - stale synced data
   - rough heuristic by design
   - actual formula bug

## 14. Bottom Line

The logic is mostly straightforward threshold-and-sum logic, not deep financial modeling.

The places most likely to create "that number feels wrong" are:
- mixed balance source rules
- static CPP valuation math
- inferred dates
- threshold-based recommendations
- cloud-sync state inconsistencies

If you want, the next useful step is for me to turn this into a second document called `LOGIC_DECISION_TABLES.md` with compact rule tables only:
- input
- condition
- output
- risk
