# Spec 00 — Foundation: Knowledge-Graph Model & Shared Conventions

**Status:** APPROVED 2026-09-01 (naming, order normalization, and `assigned` rota model explicitly verified)
**Revised:** 2026-11-19 — `ingredient` entity renamed `inventory-item` (`ingredient` is now its role); `recipe` entity and `makes` relation removed, `requires` links `menu-item` ↔ `inventory-item` directly; `work-date` is `date`.
**Revised:** 2026-11-20 — §2 ingredient counts corrected 17 → 18 (the CSVs lack a trailing newline, so line counts missed the last row); ING018 noted as used in no recipe.
**Revised:** 2026-11-20 — §2 split into entity sources (`data/`) and relation sources (`archive/`); D5 updated for merged duplicate order lines; drop list completed.
**Re-approved:** 2026-11-20 — revised model (`inventory-item`, `requires (product, ingredient)` with no recipe entity, `work-date` as `date`, merged order lines) explicitly approved by the user.
**Depends on:** nothing
**Depended on by:** `01-stock-out.md`, `02-restocking.md`, `03-staffing.md`

## 1. Purpose

Define the target TypeDB 3.x knowledge-graph schema and the shared conventions
(naming, units, time, derived logic) that the three scenario specs build on.
This spec deliberately contains **no scenario queries** — only the model they
all share.

**Out of scope:** data loading mechanics (CSV2TQL pipeline compatibility is
explicitly ignored for this phase, per project decision), application code,
visualization.

## 2. Source data profile (facts, verified 2026-09-01)

**Entity sources (`data/`)** — one row per entity:

| Source | Rows | Feeds | Notes |
|---|---|---|---|
| `data/inventory.csv` | 18 | `inventory-item` | stock as `quantity` (number of packs) × `ing_weight` per pack. ING018 (Vanilla extract) appears in no recipe, so it has no `requires` relation and zero demand (verified 2026-11-20) |
| `data/menu.csv` | 24 | `menu-item` | name × size variants, category, price; identical to `archive/menu_items.csv` (verified 2026-11-20) |
| `data/orders.csv` | 430 | `order` | one row per order, deduplicated from `archive/orders.csv` keeping the first non-blank `in_or_out`; 2024-02-12 07:04 → 2024-02-17 17:00 (Mon–Sat, one week) |
| `data/staff.csv` | 4 | `staff` | position + hourly wage; identical to `archive/staff.csv` (verified 2026-11-20) |
| `data/shift.csv` | 12 | `shift` | day-of-week template: 2 shifts/day, 07:00–13:00 and 13:00–17:00; identical to `archive/shift.csv` (verified 2026-11-20) |

**Relation sources (`archive/`)** — the join keys and relation attributes live only here:

| Source | Rows | Feeds | Notes |
|---|---|---|---|
| `archive/recipes.csv` | 61 lines, 24 recipes | `requires` (61) | `recipe_id` equals menu `sku` (1:1, verified — no orphans either way), so it serves only as the load-time join key to `menu-item` |
| `archive/orders.csv` | 521 lines | `contains` (517) | rows are **order lines**; 4 orders (ORD041, ORD106, ORD173, ORD243) list the same item on two lines, merged into one `contains` with `line-quantity 2` (see D5) |
| `archive/rota.csv` | 18 assignments | `assigned` (18) | staff↔shift per date; covers exactly the same week as orders |

**Superseded (not loaded):** `archive/ingredients.csv` (identical to
`data/inventory.csv` minus `quantity`), `archive/inventory.csv` (`quantity`
identical to `data/inventory.csv` for all 18 items), `archive/menu_items.csv`,
`archive/staff.csv`, `archive/shift.csv` (identical to their `data/`
counterparts). All re-verified 2026-11-20.

Because orders and rota cover the same single week, all three scenarios can
produce real cross-domain findings from this dataset without synthetic data.

## 3. Target schema (full idiomatic remodel)

Design principles (these ARE the decisions; scenario specs must not deviate):

- **D1 — No foreign-key attributes.** Joins become relations. `ing_id_fk`,
  `item_id_fk`, `*_row_id` do not exist in the target schema.
- **D2 — Quantities live on relations.** A quantity that describes a
  *connection* (how much of an inventory item one serving of a menu item
  uses; how many of an item an order contains) is owned by the relation, not
  by either entity.
- **D3 — One entity instance per real-world thing.** One `order` per
  `order_id` (not per CSV line). A recipe is **not** an entity: it is the set
  of `requires` relations of one `menu-item`. Because `recipe_id` = `sku`
  1:1, a recipe entity would carry no information of its own.
- **D4 — Singular entity names, no `_pk`/`_fk` suffixes.** Identity
  attributes are annotated `@key`. Entities are named for what a thing *is*
  (`inventory-item`). Roles are named for what it *does* in a relation
  (`ingredient`, `product`).
- **D5 — Line-level rows become relation instances.** A recipe CSV line →
  one `requires` relation. Each distinct (order, menu-item) pair → one
  `contains` relation whose `line-quantity` is the sum of that pair's order
  lines. Repeated lines for the same item in the same order are merged, not
  duplicated (521 order lines → 517 `contains`; decided 2026-11-20).

### 3.1 Entities

```typeql
entity inventory-item,
  owns inventory-item-id @key,  # string, e.g. "ING001"
  owns name,
  owns pack-size,            # double — was ing_weight (amount per pack)
  owns unit,                 # string — grams | ml | units
  owns pack-price,           # double — was ing_price (price per pack)
  owns packs-in-stock,       # double — was quantity
  plays requires:ingredient;

entity menu-item,
  owns item-id @key,         # string, e.g. "It001"
  owns sku,                  # = recipes.csv recipe_id (load-time join)
  owns name,
  owns category,
  owns size,
  owns price,                # double — selling price
  plays requires:product,
  plays contains:item;

entity order,
  owns order-id @key,        # string, e.g. "ORD001"
  owns created-at,           # datetime
  owns customer-name,        # string (see template note T2)
  owns channel,              # string — "in" | "out" (was in_or_out)
  plays contains:order;

entity staff,
  owns staff-id @key,
  owns first-name,
  owns last-name,
  owns position,
  owns hourly-wage,          # double
  plays assigned:staff;

entity shift,
  owns shift-id @key,
  owns day-of-week,          # string
  owns start-time,           # see D8 below
  owns end-time,
  plays assigned:shift;
```

### 3.2 Relations

```typeql
relation requires,           # recipe line (was: used + made via recipe entity)
  relates product,           # played by menu-item
  relates ingredient,        # played by inventory-item
  owns amount;               # double, per serving, in the inventory item's `unit`

relation contains,           # order line, one per distinct (order, item) (was: included)
  relates order,
  relates item,
  owns line-quantity;        # integer — was item_quantity

relation assigned,           # rota entry: staff works a shift on a date
  relates staff,
  relates shift,
  owns work-date;            # date — was rota.date
```

`assigned` carrying `work-date` is the idiomatic replacement for the `rota`
join table: the same (staff, shift) pair on different dates is simply two
relation instances. Scenario 03 may extend this (e.g. a `shift-instance`
hyper-relation) — that decision belongs to spec 03, not here.

### 3.3 Attribute definitions

All attributes above declared once with types:
`string` for ids/names/category/size/unit/channel/day-of-week/position;
`double` for pack-size, pack-price, packs-in-stock, amount, price, hourly-wage;
`integer` for line-quantity; `datetime` for created-at; `date` for work-date
(rota rows carry a calendar date only, no time component).

- **D8 — shift times:** stored as `string` (`"07:00:00"`). TypeDB 3.x has no
  standalone time-of-day type; scenario 03 owns the decision of how to compare
  order timestamps with shift windows (string comparison works for
  zero-padded HH:MM:SS).

## 4. Shared conventions

- **C1 — Units.** Inventory-item stock in *base units* is always
  `packs-in-stock × pack-size`, expressed in `unit`. The `requires` `amount`
  is already in base units. All consumption math happens in base units; money is
  converted via `pack-price / pack-size` (cost per base unit).
- **C2 — Derived logic = TypeDB functions.** Every derived metric a scenario
  defines (consumption per order, stock coverage, revenue at risk, shift
  labor cost…) is written as a schema-level **function**, not inlined in
  ad-hoc queries. This is the "semantic layer inside the database" story and
  the core of the demo/template value.
- **C3 — Observation window.** The dataset covers 6 trading days
  (2024-02-12 → 2024-02-17). Any rate ("per day") is computed over trading
  days present in the data, not calendar assumptions.
- **C4 — Spec format.** Each scenario spec contains: Business question →
  Model additions (if any) → Functions → Queries → Expected finding shape →
  Acceptance criteria → Template notes. Small; one scenario only.

## 5. Template notes (generalizing to client data)

- **T1 — The chain pattern.** `resource —composition→ product
  —transaction→ order` (inventory-item —requires→ menu-item —contains→
  order) generalizes to any BOM/manufacturing, pharma-formulation, or
  service-composition client. The transformation (recipe/BOM) is a
  *relation carrying quantities*, not an intermediary entity. Promote it to
  an entity only when the client's recipes have identity of their own, such
  as versions, approvals, or several recipes per product. The scenario specs
  are written against this abstract chain.
- **T2 — Customer as attribute.** `customer-name` stays an attribute because
  the dataset has bare first names with no identity. For a client with real
  customer records, promote it to a `customer` entity playing a role in a
  `placed` relation — nothing else in the model changes (that's the
  polymorphic-model selling point; say so in the demo).
- **T3 — Calendar entities.** `assigned` + `work-date` is the minimal rota
  model. Clients with scheduling complexity (breaks, roles-per-shift,
  locations) upgrade `assigned` to a hyper-relation without touching
  staff/shift entities.

## 6. Acceptance criteria

- [x] Schema in §3 is valid TypeDB 3.x TypeQL (compiles under `define`).
      *(Passed 2026-11-18; re-passed 2026-11-20 after the 2026-11-19 revision.)*
- [x] Every source CSV column is either mapped to an attribute/relation in §3
      or explicitly dropped.
      - Dropped columns: `recipe_row_id`, `order_row_id`, `rota.row_id`,
        `rota.rota_id`.
      - Used only as load-time join keys (not stored): `recipes.recipe_id`
        (→ `menu-item` via `sku`), `recipes.ing_id_fk`
        (→ `inventory-item-id`), `orders.order_id_pk` / `orders.item_id_fk`
        (→ `order-id` / `item-id`), `rota.staff_id` / `rota.shift_id`.
      - In `archive/orders.csv`, only the relation columns (`order_id_pk`,
        `item_id_fk`, `item_quantity`) are read. The order-level columns
        (`created_at`, `cust_name`, `in_or_out`) come from `data/orders.csv`.
      - Dropped files (all columns, superseded by `data/`):
        `archive/inventory.csv` (`inv_id`, `ing_id`, `quantity`),
        `archive/ingredients.csv` (`ing_id_pk`, `ing_name`, `ing_weight`,
        `ing_measure_unit`, `ing_price`), `archive/menu_items.csv`,
        `archive/staff.csv`, `archive/shift.csv`.
- [x] Decisions D1–D5, D8 and conventions C1–C4 are referenced (not
      restated) by the three scenario specs. All three cite "schema §3" and
      "conventions C1–C4" in their headers; D8 is cited inline by spec 03.
      *(D6–D7 are not defined: the numbering gap is historical, and D8 keeps
      its number because spec 03 cites it.)*
- [x] User has explicitly approved: entity/relation naming (D4, §3.2 names),
      order normalization (D3/D5), and the `assigned` rota model (§3.2).
      *(Approved 2026-09-01; revised model re-approved 2026-11-20.)*
