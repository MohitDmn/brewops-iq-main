# Part C Brief — Region Settlement (`src/settlement/settle.ts`)

## Function Signature
```ts
export function settleRegion(input: SettleRegionInput): RegionSettlement
```

## Types (from SPEC.md)
```ts
interface SettleRegionInput {
  regionId: string
  date: string
  tickets: Array<{ storeId: string; memberId: string|null; lines: CartLine[] }>
}

interface RegionSettlement {
  regionId: string
  date: string
  grossTotal: number
  lineDiscountTotal: number
  orderDiscountTotal: number
  discountTotal: number
  netTotal: number
  perCategory: Record<string, number>
  offerUsage: Record<string, number>
  bonus: number
  storesVisited: string[]
  storesMissed: string[]
}
```

## Imports
```ts
import { priceTicket } from '../pricing/engine'
import { getRegions, getMenuItem } from '../data'
```

## Algorithm

### 1. Validate Region
- `const region = getRegions().find(r => r.id === input.regionId)`
- If not found → `throw new Error("Unknown region: " + input.regionId)`

### 2. Validate Store IDs
- Get valid storeIds from region: `region.stores.map(s => s.storeId)`
- For each ticket: if `!validStoreIds.includes(ticket.storeId)` → `throw new Error("Store not in region: " + ticket.storeId)`

### 3. Price Each Ticket
```ts
const pricedTickets = input.tickets.map(t => 
  priceTicket({ lines: t.lines, memberId: t.memberId, date: input.date })
)
// Errors from priceTicket propagate automatically
```

### 4. Aggregate Money Totals (all round2)
```ts
function round2(x: number): number {
  return Math.round((x + Number.EPSILON) * 100) / 100
}

let grossTotal = 0
let lineDiscountTotal = 0
let orderDiscountTotal = 0
const perCategory: Record<string, number> = {}
const offerUsage: Record<string, number> = {}

for (const pt of pricedTickets) {
  for (const line of pt.lines) {
    grossTotal += line.gross
    lineDiscountTotal += line.discount
    // perCategory
    const item = getMenuItem(line.productId)
    if (item) {
      perCategory[item.category] = (perCategory[item.category] || 0) + line.net
    }
    // offerUsage (line level)
    if (line.appliedOfferId) {
      offerUsage[line.appliedOfferId] = (offerUsage[line.appliedOfferId] || 0) + 1
    }
  }
  orderDiscountTotal += pt.orderLevel.discount
  // offerUsage (order level)
  if (pt.orderLevel.appliedOfferId) {
    offerUsage[pt.orderLevel.appliedOfferId] = (offerUsage[pt.orderLevel.appliedOfferId] || 0) + 1
  }
}

grossTotal = round2(grossTotal)
lineDiscountTotal = round2(lineDiscountTotal)
orderDiscountTotal = round2(orderDiscountTotal)
const discountTotal = round2(lineDiscountTotal + orderDiscountTotal)
const netTotal = round2(pricedTickets.reduce((sum, pt) => sum + pt.total, 0))

// Round perCategory values, sort keys, remove zeros
const sortedPerCategory: Record<string, number> = {}
for (const cat of Object.keys(perCategory).sort()) {
  const val = round2(perCategory[cat])
  if (val !== 0) sortedPerCategory[cat] = val
}

// Sort offerUsage keys
const sortedOfferUsage: Record<string, number> = {}
for (const id of Object.keys(offerUsage).sort()) {
  sortedOfferUsage[id] = offerUsage[id]
}
```

### 5. Bonus (Marginal Tiers on netTotal)
```ts
const tier1 = Math.min(netTotal, 250) * 0.03
const tier2 = Math.max(0, Math.min(netTotal - 250, 500)) * 0.06
const tier3 = Math.max(0, netTotal - 750) * 0.10
const bonus = round2(tier1 + tier2 + tier3)
```

### 6. Stores Visited & Missed
```ts
// Region stops in order, deduped (keep first occurrence)
const regionStops = region.stores.map(s => s.storeId)
const seen = new Set<string>()
const dedupedStops = regionStops.filter(id => {
  if (seen.has(id)) return false
  seen.add(id)
  return true
})

// Stores with ≥1 ticket
const visitedStoreIds = new Set(input.tickets.map(t => t.storeId))
const storesVisited = dedupedStops.filter(id => visitedStoreIds.has(id))
const storesMissed = dedupedStops.filter(id => !visitedStoreIds.has(id))
```

### 7. Return RegionSettlement
```ts
return {
  regionId: input.regionId,
  date: input.date,
  grossTotal,
  lineDiscountTotal,
  orderDiscountTotal,
  discountTotal,
  netTotal,
  perCategory: sortedPerCategory,
  offerUsage: sortedOfferUsage,
  bonus,
  storesVisited,
  storesMissed
}
```

## Key Points
- **Must import and use `priceTicket` from `../pricing/engine`** — Part C is judged through YOUR Part A
- All money values rounded half-up at each step
- perCategory: line nets only (no order discounts), keys sorted, no zero entries
- offerUsage: counts both line and order applications, keys sorted
- Bonus: marginal tiers (not flat rate), round2 only at end
- storesVisited/Missed: region stop order, deduped

## Compilation Check
```bash
npx tsc -p tsconfig.app.json --noEmit
```