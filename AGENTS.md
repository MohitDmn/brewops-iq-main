# AGENTS.md — BrewOps IQ Agent Guidance

## Purpose
This file provides additional context for the AI agent building the three Chain Operations Suite modules. Read this AFTER `.github/copilot-instructions.md`.

## Module Dependencies
```
Part A (Pricing) → Part C (Settlement imports priceTicket)
Part B (Audit)   → Independent (reads tickets, stores)
```

**Build order matters**: Part A must be complete and correct before Part C, because Part C is judged through YOUR `priceTicket` implementation.

## Recommended Run Strategy (check PROGRESS.md)
1. **Run 1**: Build Part A (Pricing Engine) only
2. **Run 2**: Build Part B (Store Audit) only  
3. **Run 3**: Build Part C (Region Settlement) only — imports your Part A

## Per-Module Briefs (Distilled from SPEC.md)

### Part A — Pricing Engine (`src/pricing/engine.ts`)

**Exports**: `priceTicket(input: PriceTicketInput): PricedTicket`

**Input**: `{ lines: CartLine[], memberId: string|null, date: string }`

**Key algorithms**:
1. **Offer filtering**: For each offer, check:
   - `validFrom ≤ date ≤ validTo` (inclusive, string compare works for ISO dates)
   - `dayOfWeek`: if present, compute UTC weekday from date, must match
   - `eligibleTiers`: if present, member must exist and tier in list (walk-in excluded)

2. **Line-level offer matching**:
   - `percent_off`: matches if scope.category === item.category OR scope.productIds includes item.id
   - `bundle`: matches if item.id === products[0] (buy product); get product (products[1]) must be in cart at qty≥1
   - `spend_threshold`: NOT line-level

3. **Best-for-customer selection per line**:
   - For each matching line-level offer, compute clamped discount
   - Pick offer with largest clamped discount
   - Tie-break: earlier validFrom → lexicographically smaller id
   - Discount 0 = not applicable

4. **Bundle discount calculation**:
   - pairs = min(qty of buy product, qty of get product) in cart
   - discount = pairs * amountOff, rounded half-up
   - Clamped at buy line's gross

5. **Order-level (spend_threshold)**:
   - After all line discounts applied, compute relevant subtotal:
     - If offer.category: sum of line nets for that category
     - Else: sum of ALL line nets (subtotal)
   - Qualifies if relevant ≥ minSubtotal
   - Pick qualifying offer with largest amountOff
   - Tie-break: earlier validFrom → lexicographically smaller id

6. **Rounding helper** (half-up to 2dp):
```ts
function round2(x: number): number {
  return Math.round((x + Number.EPSILON) * 100) / 100
}
```
Note: `Number.EPSILON` helps with float artifacts like 2.175 → 2.18

**Error cases**:
- Unknown productId → `throw new Error("Unknown product: " + id)`
- Unknown memberId → `throw new Error("Unknown member: " + id)`
- qty ≤ 0 or non-integer → `throw new Error("Invalid qty for " + productId)`

### Part B — Store Audit (`src/audit/storeAudit.ts`)

**Exports**: `auditStores(asOf: string): StoreAudit[]`

**Input validation**: `asOf` must match `/^\d{4}-\d{2}-\d{2}$/` else throw

**Algorithm per store**:
1. Get counted tickets: `tickets.filter(t => t.storeId === store.id && t.date <= asOf)`
2. Sort: date desc, then id desc
3. Weighted score:
   - Take first 4 tickets, weights [4,3,2,1]
   - `round2(sum(weight * csat) / sum(weights))`
   - 0 tickets → null
4. Trend:
   - Need ≥2 tickets
   - s1 = latest.csat
   - prevMean = round2(mean of csat of tickets[1..3])
   - Compare s1 vs prevMean → 'up' | 'down' | 'flat'
   - <2 tickets → null
5. daysSinceLastTicket:
   - 0 tickets → null
   - Else: calendar days from latest.date to asOf (parse as UTC dates, diff in days)
6. dormant: daysSinceLastTicket === null || daysSinceLastTicket > 21
7. Status:
   - 0 tickets → 'inactive'
   - weightedScore < 3.0 → 'critical'
   - weightedScore < 4.0 → 'attention'
   - else → 'thriving'
   (Use ROUNDED weightedScore for boundaries)

**Output**: Array of StoreAudit for ALL stores, sorted by storeId asc

### Part C — Region Settlement (`src/settlement/settle.ts`)

**Exports**: `settleRegion(input: SettleRegionInput): RegionSettlement`

**Imports**: `import { priceTicket } from '../pricing/engine'`

**Validation**:
- regionId must exist in getRegions() → else throw
- Each ticket.storeId must be in region.stores[].storeId → else throw

**Algorithm**:
1. For each ticket in input.tickets:
   - Call `priceTicket({ lines: ticket.lines, memberId: ticket.memberId, date: input.date })`
   - Collect PricedTicket results
2. Aggregate (all round2):
   - grossTotal = sum of all pricedLine.gross
   - lineDiscountTotal = sum of all pricedLine.discount
   - orderDiscountTotal = sum of all ticket.orderLevel.discount
   - discountTotal = lineDiscountTotal + orderDiscountTotal
   - netTotal = sum of all ticket.total
3. perCategory:
   - For each pricedLine, get item.category, accumulate line.net
   - round2 each category sum
   - Keys sorted asc, omit zero categories
4. offerUsage:
   - For each pricedLine with appliedOfferId, count++
   - For each ticket with orderLevel.appliedOfferId, count++
   - Keys sorted asc
5. Bonus (marginal on netTotal):
   - tier1 = min(netTotal, 250) * 0.03
   - tier2 = max(0, min(netTotal - 250, 500)) * 0.06
   - tier3 = max(0, netTotal - 750) * 0.10
   - bonus = round2(tier1 + tier2 + tier3)
6. storesVisited:
   - Region's stops (region.stores) in order
   - Keep storeIds that appear in ≥1 ticket
   - Dedup: if storeId appears multiple times in region.stops, keep first occurrence
7. storesMissed:
   - Region's stops in order, deduped, minus storesVisited

## Common Pitfalls to Avoid
1. **Don't use legacy rounding** — use half-up everywhere
2. **Don't apply automatic tier discounts** — tier only gates offers
3. **Don't stack multiple line offers** — pick best one per line
4. **Don't use exclusive validTo** — inclusive per SPEC
5. **Don't ignore dayOfWeek/eligibleTiers** — enforce them
6. **Don't qualify spend_threshold on gross** — use line nets
7. **Don't stack multiple order offers** — pick best one
8. **Don't forget to round at each step** — gross, discount, net, subtotal, total, all aggregates
9. **Don't allocate order discounts to categories** — perCategory is line nets only
10. **Don't use flat-rate bonus** — marginal tiers

## Testing Your Implementation
- Run `npx tsc -p tsconfig.app.json --noEmit` to verify compilation
- The agent-run.mjs wrapper does this automatically after each run
- No test suite exists in repo — correctness judged against hidden suite at end

## File Structure to Create
```
src/
├── pricing/
│   └── engine.ts          # Part A
├── audit/
│   └── storeAudit.ts      # Part B
└── settlement/
    └── settle.ts          # Part C (imports ../pricing/engine)
```

## Progress Tracking
Check `PROGRESS.md` to see which modules are assigned for this run. Update it after completing each module.