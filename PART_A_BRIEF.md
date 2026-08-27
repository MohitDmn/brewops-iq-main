# Part A Brief — Pricing Engine (`src/pricing/engine.ts`)

## Function Signature
```ts
export function priceTicket(input: PriceTicketInput): PricedTicket
```

## Types (from SPEC.md)
```ts
interface CartLine { productId: string; qty: number }
interface PriceTicketInput { lines: CartLine[]; memberId: string|null; date: string }
interface PricedLine {
  productId: string; qty: number; unitPrice: number; gross: number;
  appliedOfferId: string|null; discount: number; net: number;
}
interface PricedTicket {
  lines: PricedLine[];
  orderLevel: { appliedOfferId: string|null; discount: number };
  subtotal: number; total: number;
}
```

## Data Loaders (use these, not direct JSON imports)
```ts
import { getMenu, getMembers, getOffers, getMenuItem, getMember } from '../data'
```

## Step-by-Step Algorithm

### 1. Validate Input
- For each line: `getMenuItem(line.productId)` → throw if undefined
- If `memberId !== null`: `getMember(memberId)` → throw if undefined
- Each `qty` must be integer ≥ 1 → throw if not

### 2. Filter Active Offers
```ts
const offers = getOffers().filter(o => {
  // Date active (inclusive)
  if (o.validFrom > input.date || o.validTo < input.date) return false
  // Day of week (UTC)
  if (o.dayOfWeek) {
    const [y,m,d] = input.date.split('-').map(Number)
    const wd = new Date(Date.UTC(y, m-1, d)).getUTCDay() // 0=Sun
    const dayNames = ['Sun','Mon','Tue','Wed','Thu','Fri','Sat']
    if (!o.dayOfWeek.includes(dayNames[wd])) return false
  }
  // Eligible tiers
  if (o.eligibleTiers) {
    if (!member) return false // walk-in excluded
    if (!o.eligibleTiers.includes(member.tier)) return false
  }
  return true
})
```

### 3. Compute Line-Level Offers Per Line
For each cart line:
- Get menu item (category, basePrice)
- `gross = round2(basePrice * qty)`
- Find all active offers that match this line:
  - `percent_off`: scope.category === item.category OR scope.productIds?.includes(item.id)
  - `bundle`: item.id === offer.products[0] (buy product)
- For each matching offer, compute **clamped discount**:
  - `percent_off`: `round2(gross * percent / 100)`, clamped at gross
  - `bundle`: 
    - Count get-product qty in cart: `cart.find(l => l.productId === offer.products[1])?.qty ?? 0`
    - pairs = min(line.qty, getQty)
    - `discount = round2(pairs * amountOff)`, clamped at gross
    - If pairs === 0 → discount = 0 (not applicable)
- Pick offer with **largest clamped discount**
- Tie-break: earlier `validFrom` → lexicographically smaller `id`
- If best discount === 0 → `appliedOfferId = null`, `discount = 0`
- `net = round2(gross - discount)`, clamped at 0

### 4. Compute Order-Level Offer
After all lines have `net`:
- `subtotal = round2(sum of line nets)`
- Filter active `spend_threshold` offers:
  - Relevant amount = offer.category ? sum of line nets for that category : subtotal
  - Qualifies if relevant ≥ offer.minSubtotal
- Pick qualifying offer with largest `amountOff`
- Tie-break: earlier `validFrom` → lexicographically smaller `id`
- If none qualify → `orderLevel = { appliedOfferId: null, discount: 0 }`

### 5. Final Totals
- `total = round2(subtotal - orderLevel.discount)`, clamped at 0

### 6. Rounding Helper (Half-Up)
```ts
function round2(x: number): number {
  return Math.round((x + Number.EPSILON) * 100) / 100
}
```

## Key Differences from Legacy (DO NOT REPLICATE LEGACY)
| Legacy | SPEC |
|--------|------|
| Banker's rounding | Half-up rounding |
| Cumulative stacking | One line offer per line (best wins) |
| Auto tier discount (5%/10%) | NO auto tier discount |
| validTo exclusive | validTo inclusive |
| Bundle once per ticket | Bundle per completed pair |
| dayOfWeek ignored | dayOfWeek enforced (UTC) |
| eligibleTiers ignored | eligibleTiers enforced |
| spend_threshold on gross | spend_threshold on line nets |
| All order offers stack | One order offer (largest wins) |

## Error Messages (exact strings)
- `Unknown product: <id>`
- `Unknown member: <id>`
- `Invalid qty for <productId>`

## Compilation Check
```bash
npx tsc -p tsconfig.app.json --noEmit
```
Must pass with no errors.