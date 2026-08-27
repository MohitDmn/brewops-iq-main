# Part B Brief — Store Audit (`src/audit/storeAudit.ts`)

## Function Signature
```ts
export function auditStores(asOf: string): StoreAudit[]
```

## Types (from SPEC.md)
```ts
interface StoreAudit {
  storeId: string
  weightedScore: number | null
  trend: 'up' | 'down' | 'flat' | null
  daysSinceLastTicket: number | null
  dormant: boolean
  status: 'thriving' | 'attention' | 'critical' | 'inactive'
}
```

## Data Loaders
```ts
import { getStores, getTickets } from '../data'
```

## Algorithm

### 1. Validate `asOf`
- Must match `/^\d{4}-\d{2}-\d{2}$/` → else `throw new Error("Invalid date: " + asOf)`

### 2. For Each Store (sorted by storeId asc)
Get all stores from `getStores()`, sort by `id` asc.

For each store:
- **Counted tickets**: `getTickets().filter(t => t.storeId === store.id && t.date <= asOf)`
- **Sort counted**: date desc, then id desc (most recent first)

### 3. Weighted Score
- Take up to 4 most recent counted tickets
- Weights: [4, 3, 2, 1] (most recent gets 4)
- `weightedScore = round2(sum(weight * csat) / sum(weights))`
- Divisors: 4 tickets→10, 3→9, 2→7, 1→4
- 0 tickets → `weightedScore = null`

### 4. Trend
- Need ≥2 counted tickets
- `s1 = counted[0].csat` (latest)
- `prevMean = round2(mean of counted[1..3].csat)` (up to 3 previous)
- Compare: `s1 > prevMean` → 'up', `<` → 'down', `===` → 'flat'
- <2 tickets → `trend = null`

### 5. Days Since Last Ticket
- 0 counted tickets → `null`
- Else: calendar days from `counted[0].date` to `asOf`
  - Parse both as UTC dates: `new Date(date + 'T00:00:00Z')`
  - Diff in milliseconds / (1000*60*60*24)
  - Same day = 0

### 6. Dormant
- `dormant = daysSinceLastTicket === null || daysSinceLastTicket > 21`
- Exactly 21 days is NOT dormant

### 7. Status (using ROUNDED weightedScore)
- 0 tickets → 'inactive'
- `weightedScore < 3.0` → 'critical'
- `weightedScore < 4.0` → 'attention'
- `weightedScore >= 4.0` → 'thriving'
- Boundaries: exactly 3.0 → attention, exactly 4.0 → thriving

### 8. Rounding Helper (Half-Up)
```ts
function round2(x: number): number {
  return Math.round((x + Number.EPSILON) * 100) / 100
}
```

## Output
Array of `StoreAudit` for ALL stores, sorted by `storeId` ascending.

## Compilation Check
```bash
npx tsc -p tsconfig.app.json --noEmit
```