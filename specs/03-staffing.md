# Spec 03 — Staffing vs. Demand

**Status:** APPROVED 2026-09-03 (D3.1, D3.3, D3.4 explicitly verified)
**Depends on:** `00-foundation.md` (schema §3 incl. staff/shift/`assigned`,
conventions C1–C4). Independent of specs 01–02 (different subgraph); shares
only the order/menu-item revenue math.

## 1. Business question

> "Are the right number of people rostered when customers actually show up —
> and does each shift earn its labor cost?"

Two views in one spec (they share the order↔shift time-join):

- **V1 (coverage):** per shift per day — headcount vs. order volume;
  find understaffed rushes and overstaffed lulls.
- **V2 (profitability):** per shift per day — revenue taken vs. wages paid;
  labor-cost ratio per shift.

**Why a knowledge graph:** the orders subgraph and the staffing subgraph
share no keys at all — no CSV column connects them. The connection *is
knowledge*, not data: it exists only through time-window reasoning
(`order.created-at` falls inside a shift on the `assigned.work-date`).
Encoding that once as a function joins two previously disconnected domains,
which is the "turn data into knowledge" headline of the whole demo.

## 2. Model additions & decisions

No new entities/relations — the foundation's `staff`, `shift`, `assigned`
suffice.

- **D3.1 — Order↔shift mapping is computed, not stored.** A function matches
  an order to the shift-on-date where the calendar date of `created-at` equals `work-date` (a `date`, foundation §3.3)
  and time-of-day falls in the shift window. Windows are half-open
  `[start, end)` **except the last shift of each day, which is closed**
  `[start, end]` — required because the dataset's final order lands at
  exactly 17:00:00. (Verified against data.)
- **D3.2 — Labor cost of a shift-on-date** = Σ over staff assigned that date
  to that shift of `shift-duration-hours × hourly-wage`. Durations come from
  start/end times (6h mornings, 4h afternoons).
- **D3.3 — Coverage metric = orders per staff-hour.** Normalizes the 6h/4h
  shift asymmetry so morning and afternoon shifts compare fairly. Flags:
  a shift-on-date is *understaffed* if its orders-per-staff-hour exceeds
  1.5× the week's median, *overstaffed* below 0.5× (both parameters).
- **D3.4 — Shift times as strings compare lexically** (foundation D8):
  zero-padded `HH:MM:SS` makes string comparison correct. The order's
  time-of-day is extracted from `created-at` at query time; if TypeDB 3.x
  datetime accessors prove awkward at implementation, the fallback (owning a
  redundant `created-time` string attribute on `order`, set at load) is
  pre-approved by this spec.

## 3. Functions

```typeql
# Revenue of one order: sum of price × line-quantity over its lines.
fun order_value($o: order) -> double:
  match
    contains (order: $o, item: $m), has line-quantity $q;
    $m has price $p;
    let $v = $p * $q;
  return sum($v);

# Orders landing in a given shift on a given date (D3.1 boundary rule).
fun orders_in($sh: shift, $d: date) -> { order };

# Headcount, labor cost (D3.2), order count, revenue, orders per
# staff-hour (D3.3) for a shift-on-date — each a small function over
# `assigned` and `orders_in`.
fun headcount($sh: shift, $d: date) -> integer;
fun labor_cost($sh: shift, $d: date) -> double;
fun shift_revenue($sh: shift, $d: date) -> double;
fun orders_per_staff_hour($sh: shift, $d: date) -> double;
```

(`order_value` shown in full as the pattern; the rest are one-pattern
aggregations declared here as signatures — bodies at implementation, per
acceptance A3.)

## 4. Queries

**Q1 — Coverage board:** every (work-date, shift): day-of-week, window,
headcount, order count, orders-per-staff-hour, flagged
understaffed/overstaffed per D3.3. Sorted by orders-per-staff-hour
descending. Includes an hourly order histogram for the flagged shifts as a
drill-down (same traversal, grouped by hour).

**Q2 — Shift P&L:** every (work-date, shift): `shift_revenue`, `labor_cost`,
gross margin over labor (`revenue − labor`), and labor-cost ratio
(`labor / revenue`). Sorted by ratio descending — the top rows are shifts
that barely (or don't) pay for themselves.

**Q3 — Staff exposure (small bonus):** per staff member: shifts worked,
total hours, total orders handled during their shifts — evidence for who
carries the rushes.

## 5. Expected finding shape (the demo takeaway)

- *"Monday 07:00–13:00 runs at N orders per staff-hour — X× the weekly
  median — with only H staff rostered: your clearest understaffed rush.
  Wednesday afternoon runs at M with the same headcount: overstaffed."*
- *"Afternoon shifts average a labor-cost ratio of R%, vs. S% for mornings;
  shift ⟨day, window⟩ is the only one where wages exceed takings."*
- *"⟨Name⟩ worked the two busiest shifts of the week; ⟨Name⟩ none of them."*

## 6. Acceptance criteria

- [ ] A1: the order↔shift join exists **only** in `orders_in` — every query
      reuses it; no query re-implements time-window logic.
- [ ] A2: Q1/Q2 are single TypeQL queries over §3 functions; the only
      parameters are the D3.3 flag multipliers (1.5 / 0.5).
- [ ] A3: all §3 functions (incl. bodies for the signature-only ones)
      compile against the foundation schema in TypeDB 3.x; if D3.4's
      fallback attribute is needed, the foundation spec gains a one-line
      amendment noting it.
- [ ] A4: hand-verification: pick one shift-on-date (e.g. Mon 2024-02-12
      morning), count matching orders and compute wages from the CSVs, and
      match the graph's answer exactly — including the 17:00:00 boundary
      order landing in Saturday's afternoon shift.
- [ ] A5: write-up delivers §5-style sentences with real names, days, and
      figures.
- [ ] User has explicitly approved: D3.1 (computed join + boundary rule),
      D3.3 (orders-per-staff-hour with 1.5×/0.5× flags), and D3.4 (string
      time comparison with pre-approved fallback attribute).

## 7. Template notes

- **T3.1 — Generic pattern.** This is *temporal join across disconnected
  domains*: any client with an events stream and a resources calendar
  (support tickets vs. on-call rota, deliveries vs. driver shifts, incidents
  vs. SRE coverage) gets this spec by renaming entities. The `orders_in`
  function is the whole template.
- **T3.2 — Roles per shift.** Real clients staff shifts by role (barista vs.
  manager). `assigned` grows a `role` attribute or the shift links to role
  requirements; coverage becomes per-role — the queries' shape is unchanged.
- **T3.3 — Thresholds.** The 1.5×/0.5× flags are placeholders for a client's
  service-level targets (e.g. max orders per barista-hour before queue times
  degrade); store them as attributes on `shift` when known.
