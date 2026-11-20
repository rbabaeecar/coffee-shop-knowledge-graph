# Spec 02 — Demand-Driven Restocking

**Status:** APPROVED 2026-09-03 (D2.1 and D2.2 explicitly verified)
**Depends on:** `00-foundation.md` (schema §3, conventions C1–C4);
reuses `stock_units` from `01-stock-out.md` §3 (shared schema function).

## 1. Business question

> "Given what customers actually ordered, how fast is each ingredient really
> being consumed, when does each run out, and what should this week's
> restock order be?"

Spec 01 looks at *exposure* (what breaks if stock fails); this spec looks at
*rate* (when it will fail and what to buy). No overlap by design (D1.2).

**Why a knowledge graph:** consumption is *derived* knowledge — no CSV
records it. It only exists by traversing
`order → contains(line-quantity) → menu-item → requires(amount) →
inventory-item` and multiplying quantities found on two different relations
along the path. That per-edge-attribute math across a multi-table path is
exactly what join-table SQL obscures and TypeQL functions make declarative
and reusable.

## 2. Model additions

None. Uses the foundation schema unchanged.

- **D2.1 — Burn rate = simple mean over the 6 trading days** (C3): total
  consumed base units ÷ 6. No weekday weighting or trend — deliberately
  simple for a one-week dataset (template note T2.2 covers seasonality).
- **D2.2 — Restock advice is pack-quantized and costed.** The suggestion is
  whole packs to reach a target coverage of **T days** (parameter, default
  **14**), priced via `pack-price`, so the finding is a purchasable order
  with a cash total — not an abstract rate.

## 3. Functions

```typeql
# Total base units of an inventory item consumed by all orders in the window:
# for each order line, line-quantity × the per-serving amount of this item.
fun consumed_units($i: inventory-item) -> double:
  match
    requires (product: $m, ingredient: $i), has amount $a;
    contains (order: $o, item: $m), has line-quantity $q;
    let $c = $a * $q;
  return sum($c);

# Mean consumption per trading day (D2.1).
fun daily_burn($i: inventory-item) -> double:
  match let $d = consumed_units($i) / 6.0;
  return first $d;

# Days until stock-out at the observed burn rate.
fun days_of_stock($i: inventory-item) -> double:
  match let $d = stock_units($i) / daily_burn($i);   # from spec 01
  return first $d;

# Whole packs to buy to reach $t days of coverage (0 if already covered).
fun packs_to_buy($i: inventory-item, $t: double) -> integer:
  match
    $i has pack-size $s;
    let $deficit = $t * daily_burn($i) - stock_units($i);
    let $packs = ceil(max($deficit, 0.0) / $s);
  return first $packs;
```

## 4. Queries

**Q1 — Burn-down board:** for every ingredient: name, stock in base units,
`daily_burn`, `days_of_stock`, sorted ascending by `days_of_stock`. The top
rows are the next stock-outs, with dates (window end + days_of_stock).

**Q2 — Restock order** (parameter `T`, default 14): every ingredient with
`packs_to_buy($i, T) > 0`: packs to order and line cost
(`packs × pack-price`), plus the basket total. Output is literally a
purchase order.

**Q3 — Consistency cross-check (analytical bonus):** ingredients where
`days_of_stock` is high but spec 01's coverage is low or vice versa — i.e.
slow-burning ingredients that nevertheless bottleneck a high-revenue item.
This is the cross-scenario insight only possible because both specs share
one graph and one function library.

## 5. Expected finding shape (the demo takeaway)

- *"At last week's demand, espresso beans run out in ~N days — first of all
  17 ingredients. Milk follows at ~M days."*
- *"To hold 14 days of coverage, order: X packs beans (£a), Y packs milk
  (£b), … — basket total £Z."*
- *"Ingredient W burns slowly but is still your #1 risk: it bottlenecks
  your two best-selling items (cross-check with spec 01)."*

## 6. Acceptance criteria

- [ ] A1: `consumed_units` is one traversal — no intermediate tables,
      exports, or client-side math.
- [ ] A2: Q1/Q2 invoke only §3 + shared functions; `T` is the only Q2
      parameter.
- [ ] A3: functions compile against the foundation schema in TypeDB 3.x.
- [ ] A4: hand-verification per C1: pick one ingredient (e.g. ING001),
      compute consumption from the CSVs in a spreadsheet/script, and match
      the graph's answer exactly.
- [ ] A5: Q2's output totals to a cash figure; §5-style sentences with real
      numbers in the write-up.
- [ ] User has explicitly approved: D2.1 (simple 6-day mean burn rate) and
      D2.2 (pack-quantized, costed restock with default T = 14 days).

## 7. Template notes

- **T2.1 — Generic pattern.** This is *derived resource consumption*:
  transactions × composition = resource demand. Identical shape for cloud
  cost per feature (request → service → dependency → compute), or materials
  planning (sales order → BOM → parts).
- **T2.2 — Seasonality.** With a longer window, replace `daily_burn`'s mean
  with a grouped-by-weekday function; `days_of_stock` and `packs_to_buy`
  stay untouched — a selling point for function-composed semantics.
- **T2.3 — Live stock.** Here `packs-in-stock` is a snapshot. Clients with
  transactional inventory would append stock-movement relations and compute
  stock as a function too; consumers of `stock_units` never change.
