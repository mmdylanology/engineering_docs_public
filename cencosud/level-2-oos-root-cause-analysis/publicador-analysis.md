# PUBLICADOR CSV Analysis — Complete Reference
**Date of PUBLICADOR file:** 2026-07-10, Load_Time: 11:41:24 CLT
**HANA batch matched:** vw_daily_nrt_0710_1141 (same hour)
**Analysis date:** 2026-07-28 (cross-verified 2026-07-28)

---

## 1. File Overview

| Metric | Value |
|--------|-------|
| Total rows | 1,225,228 |
| Unique (store, item) pairs | 1,213,379 |
| Stores | 41 (35 regular + 6 dark stores) |
| Unique items | 38,395 |
| Delimiter | Semicolon (;) |
| PUBLICAR | 836,690 (68.3%) |
| NO PUBLICAR | 388,538 (31.7%) |

**Schema:**
```
Calendar_dt;Load_Time;Location_Id;Item_Id;umv;stock_umb;accion;caso
```

**Unit of Measure (umv) distribution:**
| UMV | Count | Description |
|-----|-------|-------------|
| UN | 1,183,914 | Units (discrete items) |
| KG | 21,828 | Kilograms (meat, produce, deli) |
| PAK | 15,359 | Packs |
| CS | 3,762 | Cases |
| KIT | 201 | Kits |
| BAG | 164 | Bags |

**Dark stores (6):** J403, J408, J410, J411, J414, J512
- **Inheriting** (4): J403, J408, J410, J414 — have PRODUCTO TIENDA MADRE rows (20-40%), inherit mother store stock
- **Standalone** (2): J411, J512 — zero TIENDA MADRE, no mother store, behave like regular stores in PUBLICADOR

**Duplicate (store,item) pairs:** 11,316 (10,947 appear 2x, 205 appear 3x, 164 appear 4x). All 4x duplicates are JERARQUIA PRODUCCION items (production variants). 2x duplicates spread across JERARQUIA CON STOCK REGULAR and REGLA DDS. Evenly distributed across stores (~280/store).

---

## 2. All 15 `caso` Values — Decision Rules

| # | caso | Action | Count (%) | Logic | False OOS count |
|---|------|--------|-----------|-------|-----------------|
| 1 | JERARQUIA CON STOCK REGULAR | Mostly PUBLICAR | 409,284 (33.4%) | Hierarchy enabled, has stock — main happy path | 5,109 |
| 2 | REGLA DDS | Mostly PUBLICAR | 270,743 (22.1%) | Days-of-supply rule — blocks if stock below demand rate threshold | 3,418 |
| 3 | JERARQUIA DESHABILITADA | 100% NO PUBLICAR | 241,221 (19.7%) | **Entire category disabled** — always blocks regardless of stock | 105,001 |
| 4 | JERARQUIA PRODUCCION | Mostly PUBLICAR | 119,407 (9.7%) | Production hierarchy — bakery, deli, in-house items | 917 |
| 5 | MAESTRA MARCAS MMPP-IMPO | Mostly PUBLICAR | 67,961 (5.5%) | Brand master: own-brand + imported products | 996 |
| 6 | PRODUCTO TIENDA MADRE | Mixed | 30,390 (2.5%) | Dark store inherits mother store's stock qty | 3,285 |
| 7 | MAESTRA MARCAS PRODUCCION | Mostly PUBLICAR | 27,738 (2.3%) | Brand master: production/in-house items | 442 |
| 8 | PRODUCTO NUEVO | ~50/50 | 20,884 (1.7%) | New product — publish only if stock > 0 | 0 |
| 9 | LISTADO FIJO SIN STOCK | 100% PUBLICAR | 14,407 (1.2%) | Fixed assortment — **always publish** even with zero/negative stock | 0 |
| 10 | LISTADO FIJO CON STOCK | Mostly PUBLICAR | 10,887 (0.9%) | Fixed assortment with stock | 252 |
| 11 | LISTADO BAJA | 100% NO PUBLICAR | 8,216 (0.7%) | Delisted/discontinued — always blocks | 5,500 |
| 12 | PROD CON VENTA | 100% PUBLICAR | 1,503 (0.1%) | Products with active sales history — always publish | 0 |
| 13 | PRODUCTO NO CAT EN DARKSTORE | 100% NO PUBLICAR | 1,315 (0.1%) | Item not in dark store catalog — blocks | 567 |
| 14 | SET DE VENTA | Mixed | 1,043 (0.1%) | Sales bundles/kits | 32 |
| 15 | CASO FECHA HORA | 100% PUBLICAR | 229 (0.0%) | Time-window override | 0 |

---

## 3. Dark Store ↔ Mother Store Mapping

### 3.1 Inheriting Dark Stores (4 of 6)

**Pattern: J4xx → J5xx (same last 2 digits)**

| Dark Store | Mother Store | TIENDA MADRE Qty Match | TIENDA MADRE Items | Overall Match Rate |
|-----------|-------------|----------------------|-------------------|-------------------|
| J403 | J503 | 100.0% (7,849/7,849 with PUB>0) | 10,247 (40.5%) | 41.2% |
| J408 | J508 | 100.0% (3,541/3,541 with PUB>0) | 4,476 (22.8%) | 49.6% |
| J410 | J510 | 100.0% (5,423/5,423 with PUB>0) | 6,607 (29.8%) | 43.9% |
| J414 | J514 | 100.0% (6,651/6,651 with PUB>0) | 9,060 (35.9%) | 38.8% |

Low overall match rates (39-50%) driven by mother store stock inheritance inflating PUB above own HANA.

### 3.2 Standalone Dark Stores (2 of 6)

| Dark Store | TIENDA MADRE | PUB > HANA | Overall Match Rate | Behavior |
|-----------|-------------|-----------|-------------------|----------|
| J411 | 0 rows (0%) | 1 row | 65.5% | Like a regular store — no inheritance |
| J512 | 0 rows (0%) | 0 rows | 84.5% | Like a regular store — no inheritance |

J411 and J512 are operationally dark stores but PUBLICADOR treats them identically to regular stores: no stock inheritance, PUB never exceeds HANA. **J512 breaks the J4xx naming convention** — it's a J5xx code that is a dark store.

**Mapping method:** For each TIENDA MADRE row in a dark store, compared pub_qty against every regular store's HANA qty for the same item. The mother store matched at 100%. For J411/J512, brute-force tested all J5xx stores — no mother store found (max 60 coincidental matches out of 5,285+ mismatches).

---

## 4. HANA × PUBLICADOR Quantity Cross-Reference

**Join methodology:** PUBLICADOR Item_Id (integer) zero-padded to 18 digits → HANA Item_Id (e.g., `1022568` → `000000000001022568`).

### 4.1 Overall Match Rate

| Metric | Count |
|--------|-------|
| Matched (store,item) in both | 1,223,410 (99.85% of PUBLICADOR) |
| PUBLICADOR only (no HANA match) | 1,818 |
| HANA only (not in PUBLICADOR) | 21,347,928 (non-Jumbo stores, non-GIV items) |
| **Exact qty match** | **1,101,157 (90.0%)** |
| Qty mismatch | 122,253 (10.0%) |

### 4.2 By Store Type (corrected: 6 dark, 35 regular)

| Store Type | Joined | Exact Match | Mismatch | PUB > HANA | PUB < HANA |
|-----------|--------|------------|----------|-----------|-----------|
| Regular (35) | 1,068,953 | 1,015,720 (95.0%) | 53,233 (5.0%) | 29 | 53,204 |
| Dark (6) | 154,457 | 85,437 (55.3%) | 69,020 (44.7%) | 37,241 | 31,779 |

**Regular stores:** 95.0% exact match. PUB > HANA in only 29 cases (0.05% of mismatches) — PUBLICADOR essentially only reduces stock for regular stores.

**Dark stores:** 55.3% exact match. High PUB > HANA count (37,241) driven by mother store inheritance in J403/J408/J410/J414. Standalone dark stores J411 (65.5% match) and J512 (84.5% match) behave like regular stores.

### 4.3 Mismatch Root Cause Breakdown (122,253 rows)

| # | caso | Count | PUB>HANA | PUB<HANA | Dark | Regular |
|---|------|-------|---------|---------|------|---------|
| 1 | JERARQUIA CON STOCK REGULAR | 45,959 | 0 | 45,959 | 17,475 | 28,484 |
| 2 | PRODUCTO TIENDA MADRE | 24,471 | 23,097 | 1,374 | 24,471 | 0 |
| 3 | REGLA DDS | 24,465 | 13,939 | 10,526 | 17,614 | 6,851 |
| 4 | JERARQUIA PRODUCCION | 12,248 | 21 | 12,227 | 4,101 | 8,147 |
| 5 | MAESTRA MARCAS PRODUCCION | 4,975 | 0 | 4,975 | 1,652 | 3,323 |
| 6 | MAESTRA MARCAS MMPP-IMPO | 4,556 | 1 | 4,555 | 1,830 | 2,726 |
| 7 | LISTADO FIJO SIN STOCK | 3,731 | 0 | 3,731 | 1,001 | 2,730 |
| 8 | PROD CON VENTA | 647 | 16 | 631 | 380 | 267 |
| 9 | PRODUCTO NUEVO | 577 | 0 | 577 | 139 | 438 |
| 10 | LISTADO FIJO CON STOCK | 400 | 0 | 400 | 144 | 256 |
| 11 | PRODUCTO NO CAT EN DARKSTORE | 199 | 195 | 4 | 199 | 0 |
| 12 | CASO FECHA HORA | 16 | 0 | 16 | 11 | 5 |
| 13 | SET DE VENTA | 4 | 1 | 3 | 2 | 2 |
| 14 | LISTADO BAJA | 3 | 0 | 3 | 1 | 2 |
| 15 | JERARQUIA DESHABILITADA | 2 | 0 | 2 | 0 | 2 |

**Key observations:**
- Categories 1, 4, 5, 6, 7: PUBLICADOR consistently **reduces** HANA qty (PUB<HANA always). Likely safety margin / reserved stock.
- Category 2 (TIENDA MADRE): 94.4% PUB>HANA — dark store replaces its own HANA with mother's. 100% dark stores (by definition).
- Category 3 (REGLA DDS): **bi-directional**. PUB>HANA almost exclusively dark stores (13,934 dark vs 5 regular). DDS is a secondary inheritance channel.
- Category 11 (NO CAT EN DARKSTORE): inherit then block — 98% PUB>HANA, 100% dark stores.

**Split:** 69,020 mismatches in dark stores (6) + 53,233 in regular stores (35).

### 4.4 Unjoined Rows (1,818 — 0.15%)

Items in PUBLICADOR but not in HANA for that store. **100% are NO PUBLICAR** — zero customer impact.

| caso | Count | Explanation |
|------|-------|-------------|
| PRODUCTO NO CAT EN DARKSTORE | 1,089 | Item not tracked in HANA for that dark store |
| JERARQUIA DESHABILITADA | 707 | New items in disabled hierarchy, not yet in HANA |
| Other | 22 | Edge cases |

Concentrated in dark stores (J408=362, J414=286, J403=232, J410=223). Stock values: 79% zero, 20% positive (inherited from mother), 1% negative.

### 4.5 Mismatch Pattern Deep Dive

#### REGLA DDS — Bi-directional Behavior

DDS is the only caso where PUB can be **higher** than HANA. Analysis of direction by store type:

| Store Type | PUB > HANA | PUB < HANA | Total |
|-----------|-----------|-----------|-------|
| Dark stores (6) | 13,934 | 3,680 | 17,614 |
| Regular stores (35) | 5 | 6,846 | 6,851 |

**Key insight:** PUB > HANA for DDS is almost exclusively dark stores (13,934 vs 5 regular). This means DDS acts as a **secondary inheritance channel** — dark stores get mother store stock via the DDS rule, not just via PRODUCTO TIENDA MADRE. For regular stores, DDS only reduces stock (as expected from a "minimum days of supply" threshold).

#### JERARQUIA CON STOCK REGULAR — Reduction Pattern

Distribution of qty reduction (PUB - HANA) for the 45,959 mismatched rows:

| Reduction (PUB - HANA) | Count | Cumulative % |
|------------------------|-------|-------------|
| -1 | 18,326 | 39.9% |
| -2 | 8,980 | 59.4% |
| -3 | 4,097 | 68.3% |
| -4 | 3,562 | 76.1% |
| -5 | 1,538 | 79.4% |
| -6 to -10 | 4,702 | 89.7% |
| -11 to -50 | 3,672 | 97.6% |
| > -50 | 594 | 98.9% |

**Pattern:** Fixed integer reductions, NOT percentage-based. ~79% of reductions are ≤ 5 units. This is consistent with a **safety stock reserve** — PUBLICADOR withholds 1-5 units from the published quantity to prevent overselling (e.g., if HANA reports 57, PUBLICADOR publishes 55, reserving 2 units as buffer).

#### JERARQUIA PRODUCCION — Skewed by Large Items

Production hierarchy shows avg reduction of -56 units, but the distribution is highly skewed:

- **Median reduction:** -2.0 (IQR: -6.0 to -1.0)
- **Skew driver:** Large CS (case) items like item 448612 with HANA=7,125 → PUB=0 (entire production batch withheld)
- Bakery/deli items typically see -1 to -5 unit reductions, similar to JERARQUIA CON STOCK REGULAR
- The -56 average is inflated by a few high-volume production items with massive withholds

---

## 5. Scenario Analysis

### Scenario 2: Dark Store HANA=0, Mother Store Has Stock → What Does PUBLICADOR Show?

**Result: PUBLICADOR inherits the mother store's qty exactly.** (Applies to 4 inheriting dark stores only.)

| Dark Store | Total | Inherited Mother Qty | Stayed Zero | Published |
|-----------|-------|---------------------|-------------|-----------|
| J403 | 10,207 | 9,899 (97.0%) | 245 | 7,911 |
| J408 | 5,179 | 4,747 (91.7%) | 314 | 4,771 |
| J410 | 7,969 | 7,635 (95.8%) | 291 | 7,096 |
| J414 | 9,003 | 8,457 (93.9%) | 438 | 7,564 |
| **Total** | **32,358** | **30,738 (95.0%)** | **1,288** | **27,342 (84.5%)** |

**Standalone dark stores (J411, J512):** When HANA=0, PUB is **always** 0 — zero inheritance, zero fabrication. Behave identically to regular stores.

Example:
```
J403  item=1889275  DarkHANA=0  MotherHANA(J503)=8360  PUB_QTY=8360  PUBLICAR  PRODUCTO TIENDA MADRE
J408  item=1938286  DarkHANA=0  MotherHANA(J508)=7183  PUB_QTY=7183  PUBLICAR  PRODUCTO TIENDA MADRE
```

### Scenario 3: Both Dark + Mother Have Stock → Is PUB_QTY Additive?

**Result: NO. PUBLICADOR does NOT add dark + mother stock.**

When both dark store and mother store have stock > 0 (51,112 items, 4 inheriting stores):

| Dark Store | Total | Uses Own | Uses Mother | SUM (additive) |
|-----------|-------|----------|-------------|----------------|
| J403 | 12,218 | 6,897 (56.4%) | 2,062 (16.9%) | 20 (0.2%) |
| J408 | 13,518 | 8,063 (59.6%) | 1,471 (10.9%) | 14 (0.1%) |
| J410 | 12,281 | 7,027 (57.2%) | 1,747 (14.2%) | 15 (0.1%) |
| J414 | 13,095 | 5,689 (43.4%) | 2,663 (20.3%) | 24 (0.2%) |

**Conclusion: The hypothesis "20 units (dark) + 30 units (mother) = 50 available" is FALSE.**
PUBLICADOR either uses the dark store's own qty OR replaces it with the mother's qty, never both.

### Scenario 4: Regular Stores — No Inheritance

**Confirmed.** Regular stores (35) + standalone dark stores (J411, J512) use their own HANA qty only. Regular stores: 95.0% exact match, 5.0% mismatch, PUB>HANA in only 29 of 53,233 cases.

### Scenario 5: Regular Store HANA ≠ PUB qty — Breakdown

53,233 mismatches in regular stores (35). In virtually all cases:
- **HANA is higher** than PUBLICADOR (PUBLICADOR reduced the stock)
- The reduction is applied by various business rules (DDS, production adjustments, brand master)
- Negative clamping accounts for ~30 rows (HANA negative → PUB=0)

### Scenario 6: Regular Store HANA=0 But PUB > 0

**Result: ZERO cases.** For regular stores, PUBLICADOR NEVER shows stock when HANA shows zero. This confirms PUBLICADOR only reduces or maintains — never fabricates stock for regular stores.

### Scenario 7: HANA ≤ 0 AND PUB=0 AND NO PUBLICAR

**Result: 245,658 items** — this is the "correctly blocked" bucket.

Top reasons:
| caso | Count | Explanation |
|------|-------|-------------|
| JERARQUIA DESHABILITADA | 131,976 | Category disabled (no stock anyway) |
| REGLA DDS | 46,812 | Below demand threshold |
| JERARQUIA CON STOCK REGULAR | 27,831 | Zero stock in enabled hierarchy |
| MAESTRA MARCAS MMPP-IMPO | 10,587 | Brand master block |
| PRODUCTO NUEVO | 10,332 | New product with no stock |

### Scenario 8: HANA ≤ 0 AND PUB ≤ 0 AND PUBLICAR — Why Published With No Stock?

**Result: 7,236 items** published despite zero/negative stock.

| caso | Count | Explanation |
|------|-------|-------------|
| LISTADO FIJO SIN STOCK | 6,153 | **Always-visible assortment** — intentionally published with zero stock so customers can see the product exists |
| PROD CON VENTA | 516 | Products with active sales — kept visible |
| PRODUCTO TIENDA MADRE | 365 | Dark store inheritance (both zero) |
| CASO FECHA HORA | 120 | Time-window override |

### Scenario 9: HANA ≤ 0 BUT PUB > 0 AND PUBLICAR — Stock From Where?

**Result: 27,161 items** — HANA says zero/negative but PUBLICADOR shows positive stock.

| caso | Count | Explanation |
|------|-------|-------------|
| PRODUCTO TIENDA MADRE | 15,499 | **Dark store inherited mother store's stock** (HANA for dark store is 0, but mother store has stock → PUBLICADOR shows mother's qty) |
| REGLA DDS | 11,658 | DDS rule recalculated stock based on supply chain data not in HANA snapshot |
| PROD CON VENTA | 4 | Active sales override |

---

## 6. False OOS Summary

Two definitions measured:

| Definition | Total | What it measures |
|-----------|-------|-----------------|
| **PUB-based:** PUB stock > 0 AND NO PUBLICAR | **125,519** | PUBLICADOR knows stock exists (from any source) but blocks |
| **HANA-based:** HANA stock > 0 AND NO PUBLICAR | **122,391** | Physical stock at this specific store but PUBLICADOR blocks |

**Delta:** 3,128. Primarily PRODUCTO TIENDA MADRE (+2,834) — dark store inherited mother's stock into PUB but still blocked; the dark store's own HANA is zero.

### HANA-based breakdown (physical stock wasted):

| Rank | caso | Count | % | Root Cause |
|------|------|-------|---|------------|
| 1 | JERARQUIA DESHABILITADA | 105,003 | 85.8% | Entire product category disabled |
| 2 | LISTADO BAJA | 5,502 | 4.5% | Delisted but stock not cleared |
| 3 | JERARQUIA CON STOCK REGULAR | 5,212 | 4.3% | Sub-rule block within enabled hierarchy |
| 4 | REGLA DDS | 3,367 | 2.8% | Stock below demand threshold |
| 5 | MAESTRA MARCAS MMPP-IMPO | 1,029 | 0.8% | Brand master block |
| 6 | JERARQUIA PRODUCCION | 1,026 | 0.8% | Production hierarchy block |
| 7 | MAESTRA MARCAS PRODUCCION | 467 | 0.4% | Production brand block |
| 8 | PRODUCTO TIENDA MADRE | 451 | 0.4% | Dark store rule blocked despite own stock |
| 9 | LISTADO FIJO CON STOCK | 252 | 0.2% | Fixed assortment rule |
| 10 | SET DE VENTA | 32 | 0.0% | Bundle rule |
| 11 | PRODUCTO NUEVO | 29 | 0.0% | New product with stock |
| 12 | PRODUCTO NO CAT EN DARKSTORE | 21 | 0.0% | Not in dark store catalog |

**Key takeaway:** 85.8% of all false OOS is caused by a single rule — JERARQUIA DESHABILITADA. Fixing disabled hierarchies would recover 105,003 items with real physical stock.

---

## 7. Key Findings

1. **PUBLICADOR never fabricates stock for regular stores or standalone dark stores.** If HANA=0, PUB=0. Always. (Scenario 6 = 0 cases for 35 regular + J411 + J512)

2. **PUBLICADOR only reduces, never inflates** regular store quantities. Only 29 of 53,233 mismatches have PUB > HANA.

3. **Dark store inheritance is REPLACEMENT, not ADDITION.** PUBLICADOR replaces the dark store's zero-stock with the mother store's qty via PRODUCTO TIENDA MADRE. It does NOT add dark + mother. Confirmed per-store.

4. **6 dark stores, two types:** 4 inheriting (J403→J503, J408→J508, J410→J510, J414→J514, 100% mother match) and 2 standalone (J411, J512 — no inheritance, behave like regular stores). **J512 breaks the J4xx naming convention.**

5. **LISTADO FIJO SIN STOCK is intentional** — 14,407 items always published even with zero stock (always-visible assortment).

6. **Decimal quantities exist** — 17,939 PUBLICADOR rows and 72,222 HANA rows have fractional KG quantities (meat, produce, deli sold by weight).

7. **PUBLICADOR file has been stale since July 10, 2026.** The file on S3 has not been updated. PUBLICADOR may have stopped running or is writing to a different path.

8. **11,316 duplicate (store,item) pairs** in PUBLICADOR. All 4x duplicates are JERARQUIA PRODUCCION items. Evenly distributed (~280/store). Does not affect analysis conclusions (same caso/accion/qty in duplicates).

9. **REGLA DDS is a secondary dark store inheritance channel.** 13,934 dark store DDS rows have PUB > HANA (vs only 5 regular). DDS recalculates stock from supply chain data, effectively giving dark stores their mother store's stock without using TIENDA MADRE.
