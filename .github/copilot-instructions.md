# Copilot Instructions — BrewOps IQ Harness

## CRITICAL: Source of Truth
**`SPEC.md` is the ONLY authoritative specification.** Everything in `src/legacy/` and `docs/RETRO.md` describes the OLD system and CONTRADICTS the new spec. Do NOT replicate legacy behavior.

## Three Modules to Build (in order)
1. **Part A — Pricing Engine** → `src/pricing/engine.ts` exporting `priceTicket`
2. **Part B — Store Audit** → `src/audit/storeAudit.ts` exporting `auditStores`
3. **Part C — Region Settlement** → `src/settlement/settle.ts` exporting `settleRegion` (MUST import and reuse `priceTicket` from `../pricing/engine`)

## Data Access
- Read ALL data via loaders in `src/data/index.ts` (`getMenu`, `getMembers`, `getOffers`, `getStores`, `getRegions`, `getTickets`, `getMenuItem`, `getMember`, `getStore`, `getRegion`)
- Do NOT import JSON files directly

## ⚠️ LEGACY TRAPS — DO NOT REPLICATE THESE

| Legacy Behavior (WRONG) | SPEC Requirement (CORRECT) |
|------------------------|---------------------------|
| Banker's rounding (round-half-to-even) | **Half-up rounding** to 2dp (1.005 → 1.01) |
| Cumulative stacking: all matching offers apply | **At most ONE line-level offer per line** (best clamped discount wins) |
| Automatic tier discount: basic 0%, silver 5%, gold 10% | **NO automatic tier discount** — tier only gates offer eligibility |
| `validTo` is EXCLUSIVE (`date < validTo`) | `validTo` is **INCLUSIVE** (`validFrom ≤ date ≤ validTo`) |
| Bundle: at most once per ticket | Bundle applies **per completed pair** (min(qty buy, qty get)) |
| `dayOfWeek` ignored | `dayOfWeek` **must be enforced** (UTC weekday) |
| `eligibleTiers` ignored | `eligibleTiers` **must be enforced** (walk-in never eligible if present) |
| `spend_threshold` qualifies on GROSS category subtotal | Qualifies on **line NETS** (post line-discount) |
| Order-level: all qualifying offers stack | **At most ONE order-level offer** (largest amountOff wins) |

## Rounding Rules (CRITICAL)
- Every money value: round to 2dp, **half-up** (not banker's)
- `gross = round2(unitPrice * qty)`
- `discount = round2(...)` per offer type
- `net = round2(gross - discount)`, clamped at 0
- `subtotal = round2(sum of line nets)`
- `total = round2(subtotal - orderLevel.discount)`, clamped at 0
- Use a proper half-up function, NOT `Math.round(x*100)/100` (fails on 2.175 → 2.17)

## Offer Selection Rules (Part A)
1. Filter offers: date-active (inclusive), dayOfWeek match (UTC), eligibleTiers match (walk-in excluded if tiers present)
2. Line-level: for each line, find all matching active offers → pick **largest clamped discount** → tie-break: earlier validFrom → lexicographic id
3. Discount of 0 = not applicable (e.g., bundle without get line)
4. Order-level: among qualifying `spend_threshold` offers → pick largest `amountOff` → same tie-break
5. Line + order offers DO stack

## Part B — Store Audit Key Points
- `asOf` must be `YYYY-MM-DD` format, else throw
- Count tickets with `date ≤ asOf` (inclusive)
- Sort: date desc, then id desc
- Weighted score: up to 4 most recent, weights 4,3,2,1 → round2(weighted avg)
- Trend: need ≥2 tickets; compare latest csat vs mean of next up to 3 (rounded)
- daysSinceLastTicket: calendar days from latest ticket date to asOf (same day = 0)
- dormant: true if daysSinceLastTicket > 21 OR null
- Status: inactive (0 tickets), critical (<3.0), attention (3.0-3.99), thriving (≥4.0) — using ROUNDED weightedScore

## Part C — Region Settlement Key Points
- Validate regionId exists, all storeIds in region
- Call `priceTicket` for each ticket (propagate errors)
- Aggregate: grossTotal, lineDiscountTotal, orderDiscountTotal, discountTotal, netTotal (all round2)
- perCategory: sum of line nets per category (round2), keys sorted, no zero entries
- offerUsage: count line appliedOfferId + orderLevel appliedOfferId, keys sorted
- Bonus: marginal tiers on netTotal (250@3%, 250-750@6%, >750@10%), round2 only at end
- storesVisited: region's stop storeIds with ≥1 ticket, in region order, deduped
- storesMissed: remaining stop storeIds in region order, deduped

## Run Strategy
- This run may be scoped to a subset of modules (check `PROGRESS.md`)
- Build ONLY the modules designated for this run
- Create files ONLY under `src/pricing`, `src/audit`, `src/settlement`
- Do NOT write or run tests
- Do NOT modify any other files

## Output Requirements
- Each module must export exactly the function signature in SPEC.md
- Types must match SPEC.md exactly
- Code must compile with `npx tsc -p tsconfig.app.json --noEmit`