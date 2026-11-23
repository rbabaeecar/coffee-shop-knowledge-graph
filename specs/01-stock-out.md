# Spec 01 — Stock-Out Impact Tracing

**Status:** APPROVED 2026-09-01 (D1.1 and D1.2 explicitly verified)
**Revised:** 2026-11-23 — D1.1 demo default raised 20 → 50 servings (the
lowest coverage in the data is 25 servings, so 20 returned no rows);
`servings_supported` returns the raw ratio, not floored (comment corrected).
Both explicitly approved by the user.
**Depends on:** `00-foundation.md` (schema §3, conventions C1–C4)

## 1. Business question

> "Which menu items — and how much revenue — are at risk from my current
> ingredient stock levels? And if a specific ingredient runs out, what
> exactly breaks?"

Two deliverable views, one traversal pattern:

- **V1 (alert):** ingredients whose stock covers fewer than *N* servings of
  some menu item → the endangered items → the weekly revenue attached to them.
- **V2 (what-if):** for a named ingredient, assume stock = 0 and list every
  menu item that becomes unsellable, with its historical revenue.

**Why a knowledge graph:** both views are a single traversal
(`inventory-item —requires(ingredient→product)→ menu-item —contains→ order`)
with aggregation at each end. In SQL this is a 4-table join (inventory,
recipes, menu, orders) repeated per metric; in TypeQL it is one pattern
reused by two functions.

## 2. Model additions

None. Uses the foundation schema unchanged.

- **D1.1 — "At risk" is parameterized, not stored.** The data has no
  reorder-level column, so the alert threshold *N* (servings coverage) is a
  function argument, defaulting to **50 servings** in the demo (was 20;
  revised 2026-11-23 because no item falls below 20). (Template
  note T1.1: clients with reorder levels store them as an `inventory-item`
  attribute and the function reads it instead.)
- **D1.2 — Revenue exposure window = the full 6 trading days** (convention
  C3). "Revenue at risk" means *observed revenue over the window for items
  that depend on the ingredient* — an exposure proxy, not a forecast
  (forecasting is spec 02's job; the two specs must not duplicate this).

## 3. Functions (schema-level, per convention C2)

```typeql
# Stock of an inventory item in base units (C1).
fun stock_units($i: inventory-item) -> double:
  match
    $i has packs-in-stock $p, has pack-size $s;
    let $u = $p * $s;
  return first $u;

# How many servings of one menu item the current stock of ONE of its
# ingredients supports: stock_units / amount-per-serving (raw, not floored).
fun servings_supported($i: inventory-item, $m: menu-item) -> double:
  match
    requires (product: $m, ingredient: $i), has amount $a;
    let $n = stock_units($i) / $a;
  return first $n;

# Bottleneck coverage for an item: the minimum servings_supported over all
# of its ingredients. The argmin ingredient is the "limiting ingredient".
fun max_servable($m: menu-item) -> double:
  match
    requires (product: $m, ingredient: $i);
    let $n = servings_supported($i, $m);
  return min($n);

# Observed revenue for an item over the window (C3).
fun item_revenue($m: menu-item) -> double:
  match
    $m has price $p;
    contains (order: $o, item: $m), has line-quantity $q;
    let $rev = $p * $q;
  return sum($rev);
```

*(Function bodies are the spec's intent in TypeDB 3.x syntax; exact syntax is
validated at implementation time — acceptance criterion A3.)*

## 4. Queries

**Q1 — Alert view** (parameter: threshold `N`, default 50):
match every menu-item where `max_servable($m) < N`; return item name, size,
coverage (servings), `item_revenue($m)`, and the limiting ingredient(s) —
those with `servings_supported($i, $m) = max_servable($m)`. Sort by revenue
descending: *the top row is the most money endangered by the least stock.*

**Q2 — What-if view** (parameter: `inventory-item-id`, e.g. `"ING001"`):
match every menu-item playing `product` in a `requires` where the
inventory item plays `ingredient`; return item name/size, weekly units sold
(`sum(line-quantity)`), and `item_revenue($m)`; plus totals: number of
unsellable items and total revenue exposure.

## 5. Expected finding shape (the demo takeaway)

One sentence per view, e.g.:

- *"Espresso beans are 4 packs from empty — that stock caps you at ~X
  cappuccino/latte/espresso servings against £Y of last week's revenue; beans
  are the limiting ingredient for Z of your 24 menu items."*
- *"If ING001 hits zero, N items (~P% of the menu) become unsellable,
  representing £Q of a typical week's takings."*

The finding must name concrete items, counts, and money — "actionable
knowledge", not query output.

## 6. Acceptance criteria

*(All passed 2026-11-23 on TypeDB 3.13. Implementation:
`functions/01-stock-out.tql`, `queries/01-stock-out/`. The A5 write-up is
kept locally and is not tracked in git.)*

- [x] A1: Q1 and Q2 are each answered by **one** TypeQL query invoking the
      §3 functions (no client-side joins or post-processing beyond sorting).
      *(Q2 is a single `fetch` that also returns its totals and the menu size.)*
- [x] A2: Q1's threshold and Q2's ingredient are the only parameters; changing
      them requires no query edits beyond the value.
      *(Tested at N = 20, 50, 60, 201 and on ING001, ING002, ING018.)*
- [x] A3: functions compile against the foundation schema in TypeDB 3.x.
- [x] A4: unit math follows C1 exactly (packs × pack-size vs. `requires` amount) —
      verified by hand-computing one item (e.g. Medium Cappuccino) end-to-end.
      *(423.08 servings, milk-limited; £72.45.)*
- [x] A5: the write-up delivers §5-style sentences with real numbers.
- [x] User has explicitly approved: D1.1 (parameterized threshold, default 50
      servings; revised from 20 on 2026-11-23) and D1.2 (exposure = observed
      6-day revenue, no forecasting).

## 7. Template notes

- **T1.1 — Threshold source.** Swap the parameter for a stored
  `reorder-level` attribute when the client has one; the function signature
  is the only change.
- **T1.2 — Generic pattern.** This is *upstream dependency impact*: "which
  products/services/deliverables are exposed if resource X degrades?" Works
  unchanged for supplier-risk (supplier → component → product → order book)
  and skills-risk (person → capability → service → contract) domains — the
  demo script should state this mapping explicitly.
