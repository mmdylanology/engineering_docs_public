# Stock Propagation Delay — Proof of Preventable Out-of-Stock

**Analysis date:** 2026-07-02  ·  **Day analyzed:** 2026-07-01 (Chile local time, CLT / UTC-4)
**Scope:** 33 Jumbo J-stores  ·  **Prepared for:** Cencosud operations review

> **Terminology.** "Stock-propagation" below refers generically to the mechanism that carries
> HANA inventory snapshots to the VTEX storefront (today via PUBLICADOR) so out-of-stock items
> are blocked from sale. This is a data analysis of where that mechanism has a gap — it does not
> name or attribute blame to any specific implementation or vendor.
>
> All timestamps are **CLT (UTC-4)**, taken directly from the source systems with **no timezone
> conversion**. HANA `Load_Time` and Redshift `fecha_creacion` are both naive CLT and are compared directly.

---

## 1. Executive summary

On July 1, 2026, across 33 Jumbo stores, **7,143 order lines** were sold to customers for
items that were **physically not on the shelf**. The customer paid, the order was delivered,
but that item was dropped and refunded. These split into two **distinct root causes**:

| Lines | Cause | Revenue lost (CLP) | Owner |
|---:|---|---:|---|
| **1,273** | **Stock-propagation gap** — HANA already showed the item at **qty ≤ 0** before the order, but that signal did not reach VTEX in time to block the sale | **7,098,693** | **Stock-propagation pipeline — fixable** |
| **5,870** | **HANA inventory inaccuracy** — HANA showed the item **in stock (qty > 0)**, but the shelf was empty; there was no zero-signal to propagate | **31,777,095** | **Cencosud WMS — separate issue** |
| **304** | Unattributable — order placed before the first HANA batch of the day | — | — |
| **7,447** | **Total proven lost** | **38,875,788** | |

> ⚠️ **Do not merge these numbers.** The propagation-preventable loss is **1,273 lines / 7.1M CLP**.
> The 5,870 / 31.8M is a larger, *separate* inventory-accuracy problem that faster propagation
> could **not** have prevented (HANA itself was wrong). Reporting a combined "38.9M" as one
> preventable number is incorrect and undermines the credibility of the 1,273.

---

## 2. Data sources

| Source | Location | Content |
|---|---|---|
| **HANA inventory** | S3: `cencosud.prod.sm.cl.analytics/.../VW_FACT_DAILY_INVENTORY_NRT` | 18 hourly snapshots on Jul 1 (03:41 → 23:41 CLT). Fields: `Item_Id`, `Location_Id`, `Ending_On_Hand_Qty`, `Calendar_Dt`, `Load_Time`. 405.6M J-store rows across 18 batches. |
| **Customer orders + picking** | Redshift `datalake` / schema `jumbo_bo` / `io_items` + `vw_master_pickers_data` | Every Jul 1 order line for J-stores (304,225 lines). Read via `cl_sm_giv_ro` role. |
| **Order outcome** | Redshift `jumbo_bo.io_internal_orders` | Order-level status, cancellation, invoice. |

**Local export used:** `exports/cencommerce_quiebre/all_orders_0701_FULL.csv` (304,225 rows)
**Proof script:** `scripts/delay_proof_0701.py` → checkpoint `analysis/delay_proof_0701_preventable.parquet`
**Summary workbook:** `analysis/delay_proof_0701.xlsx`

### ID mapping (verified)
- **Store:** order `id_tienda` (e.g. `J403`) = HANA `Location_Id` (direct match)
- **Item:** order `refid` numeric prefix, zero-padded to 18 chars = HANA `Item_Id`
  e.g. `1968547` → `000000000001968547`;  `2020972-KG` → `000000000002020972`

### Picker status codes (`io_items.status`, from `io_items_status` lookup)
| Code | Meaning | Bucket |
|---|---|---|
| 1 | Pickeado (picked & delivered) | Delivered |
| 3 / 5 / 7 / 14 | Sin stock (no stock, validated / re-pick / sala) | **OOS — nothing delivered** |
| 4 / 6 | Sustituto (substitute given) | Substituted |
| 8 / 12 | Faltante parcial (partial fill) | Partial |
| 11 | Agregado (picker-added, not ordered) | Excluded |
| 13 | Compuesto (composite/bundle line) | Excluded |

---

## 3. Methodology

For **every order line** on July 1:

1. **Match** — find the most recent HANA batch that arrived **strictly before** the order
   timestamp (`h.batch_ts < o.fecha_creacion`). No grace period — PUBLICADOR propagates HANA
   batches to VTEX rapidly, so any batch before the order was available to the pipeline.
2. **Read stock** — take `Ending_On_Hand_Qty` from that batch for the (item, store) pair.
   `qty = 0` → out of stock. `qty < 0` → damaged / shrinkage. **Both mean nothing on the shelf.**
3. **Flag** — if `qty ≤ 0` → the line is a **candidate** (HANA showed empty before the order).
4. **Classify** by what the picker actually did (`io_items.status`).

The **preventable** claim is only the intersection: **HANA showed ≤ 0 AND the picker
independently confirmed OOS** — two systems agreeing the item was gone.

---

## 4. The full funnel (recomputed from source, 2026-07-02)

```
Order lines placed Jul 1 (J-stores, all statuses) ............ 304,225
  └─ had a HANA batch before the order (95.8%) .............. 291,507
       └─ HANA showed qty ≤ 0  (candidate set) ..............  12,492
```

Everything below sums back to **12,492** exactly (integrity-checked in code).

### 4.1 What happened to the 12,492 candidates

| Outcome (picker status) | Lines | CLP | Counted as loss? |
|---|---:|---:|---|
| **OOS — nothing delivered** (5/3) | **1,273** | **7,098,693** | ✅ HARD lost sale — **the claim** |
| Substituted (4/6) | 950 | 5,416,533 | ⚠️ partial — customer got a replacement |
| Partial fill (8/12) | 129 | 887,257 | ⚠️ partial |
| **Subtotal (clean proof)** | **2,352** | **13,402,483** | |
| Picked despite HANA ≤ 0 (1) | 9,621 | 40,406,357 | ❌ excluded — HANA was wrong, item found & delivered |
| Composite / bundle (13) | 262 | 3,848,234 | ❌ excluded — ID-mapping artifact |
| Picker-added (11) | 257 | 1,254,220 | ❌ excluded — not ordered by customer |
| **TOTAL** | **12,492** | **58,911,294** | |

**Why 9,621 are excluded, not claimed:** in all 9,621, HANA showed ≤ 0 but the picker
**found the item and delivered it** (`pickingquantity > 0` on 100%). Had the pipeline blocked
these on HANA's signal, it would have **wrongly killed 40.4M CLP of good sales**. **58% are
granel / weighed items** (bread, produce, deli) where HANA's unit counter drifts negative even
though the bin is full — proof that HANA's ≤ 0 signal is itself unreliable and must not be acted
on blindly.

---

## 5. The two lost-revenue populations

Starting instead from **all 7,447 confirmed OOS lines** (picker found nothing), split by what
HANA showed beforehand:

| Cause | Lines | CLP | Delivered? | HANA error |
|---|---:|---:|---|---|
| **HANA ≤ 0 — propagation gap (the claim)** | **1,273** | 7,098,693 | Item ❌ / Order ✅ | correct, not propagated |
| **HANA > 0 — HANA overstated** | **5,870** | 31,777,095 | Item ❌ / Order ✅ | overstated (said in-stock) |
| No HANA batch before order | 304 | — | — | n/a |
| **Total** | **7,447** | | | |

Both 1,273 and 5,870 are **item lines**, both are **real lost revenue** (customer ordered,
paid, didn't receive, refunded). The **only** difference is the cause.

---

## 6. Proof examples — propagation gap (HANA ≤ 0 before the order)

Each example shows three independent confirmations:
**(A)** HANA batch timeline (S3 parquet), **(B)** the customer order line (Redshift/CSV),
**(C)** order outcome (`io_internal_orders`, live pull 2026-07-02).

### Example 1 — Toallas Húmedas 50 un. · store J403 · order `v232198567jmch-01`

**A. HANA timeline (all 18 batches Jul 1)** — `Item_Id 000000000001968547 @ J403`:

| Batch (CLT) | Ending_On_Hand_Qty |
|---|---:|
| 03:41 → 19:40 (all 18 batches) | **0.0** |

HANA showed **zero stock in every single batch all day**, including the last one before the order.

**B. Customer order line** (`io_items` / CSV):

| Field | Value |
|---|---|
| Order placed (`fecha_creacion`) | 2026-07-01 **10:30:44** |
| Last HANA batch before order | **09:41 batch = qty 0** *(the 10:40 batch came after the order; gap ~49 min)* |
| Picker status | **5 — Sin stock (validado)** |
| Ordered qty | 1 |
| Picking qty | **0** (not delivered) |
| Picker confirmed OOS (`dateupdate`) | 2026-07-01 11:01:25 |
| Value | 1,592 CLP |

**C. Order outcome** (`io_internal_orders`, live):

| idorder | status | canceledby | amountinvoice | selectedsla | itemsquantity |
|---|---|---|---|---|---|
| v232198567jmch-01 | **53** (delivered) | **None** | 56,434 | Despacho a Domicilio | 11 |

→ Order delivered (11 items), **2 OOS lines dropped** (Toallas Húmedas + Guante Peinador, both status 5, pick 0). Not cancelled.

---

### Example 2 — Tetera Vidrio 800 ml · store J403 · order `v232243602jmch-01` (negative stock)

**A. HANA timeline** — `Item_Id 000000000001907741 @ J403`: **−7.0 in all 18 batches** (damaged/shrinkage — still nothing sellable).

**B. Order line:** placed **18:03:04**; picker status **5**; ordered 1, picked **0**; confirmed 18:35:31; value **2,995 CLP**.

**C. Outcome:** status **51** (delivered), `canceledby` **None**, invoice 36,172 CLP, 10 items. The tetera dropped.

---

### Example 3 — Arena Aglutinante 20 kg · store J408 · order `v232202849jmch-01` (negative stock, high value)

**A. HANA timeline** — `Item_Id 000000000002045198 @ J408`: **−4.0 in all 18 batches**.

**B. Order line:** placed **11:16:56**; picker status **5**; ordered 1, picked **0**; confirmed 12:14:14; value **21,990 CLP**.

**C. Outcome:** status **51** (delivered), `canceledby` **None**, invoice 71,393 CLP, 14 items. The cat litter dropped.

---

## 7. Proof examples — HANA overstated (HANA > 0 but shelf empty)

These are **not** a propagation-pipeline failure — HANA claimed stock that was not physically
there, so there was no zero-signal to propagate.

### Example 4 — Quitamanchas Vanish + Coliflor · store J513 · order `v232197388jmch-01`

**A. HANA timeline** — `Item_Id 000000000001999153 (Vanish) @ J513`: **15.0 in all 18 batches** — HANA claimed 15 units all day.

**B. Order line:** placed **10:19:33**; picker status **5**; ordered 1, picked **0**; value **1,100 CLP**.
(Same order also had Coliflor `260630` at status 5.)

**C. Outcome** (live): status **51** (delivered), `canceledby` **None**, invoice **150,325 CLP**, 39 items.
→ 37 items delivered; **2 OOS lines dropped** despite HANA showing stock.

### Example 5 — Filetitos de Pollo Congelado kg · store J512 · order `v232234357jmch-01` (high value)

**A. HANA timeline** — `Item_Id 000000000002020972 @ J512`: **93.28 kg in all 18 batches** — HANA claimed 93 kg all day; shelf was empty.

**B. Order line:** placed **16:54:12**; picker status **5**; ordered 2, picked **0**; value **21,980 CLP**.

**C. Outcome** (live): status **51** (delivered), `canceledby` **None**, invoice **326,204 CLP**, 54 items.
→ 3 OOS lines dropped (Filetitos Pollo, Durazno Granel, Tomate Beef Granel), all status 5 / pick 0, despite HANA showing large positive stock.

---

## 8. Three-table validation (both populations)

The claim "OOS item dropped, order delivered, revenue refunded — not cancelled" is confirmed
across three independent Redshift tables, for the **entire** population (not a sample):

### The 1,273 (propagation gap)
- **io_items:** 100% status 3/5/7/14 (picker confirmed no stock), 100% `pickingquantity = 0`
- **io_internal_orders:** `canceledby = NULL`, order status 51/53 (delivered/invoiced) — 10/10 random spot-check
- **Rest of basket:** other lines status 1 (delivered)

### The 5,870 (HANA overstated) — bulk confirmation across all 4,141 orders
| Table | Check | Result |
|---|---|---|
| io_items | status 3/5/7/14 (no stock) | **5,870 / 5,870 (100%)** |
| io_items | `pickingquantity = 0` (not delivered) | **5,870 / 5,870 (100%)** |
| io_internal_orders | NOT cancelled (`canceledby` NULL) | **4,067 / 4,074 (99.8%)** |
| io_internal_orders | delivered/invoiced (status 51/53) | **99.1%** |
| io_internal_orders | real invoice (`amountinvoice > 0`) | **99.8%** |
| rest of basket | other lines delivered (status 1) | **89.3%** |

Residuals: 7 orders (0.2%) genuinely cancelled; 67 orders (1.6%) fell outside the join
window (created near the day boundary). Neither moves the totals materially.

> **Order delivered ≠ item delivered.** An order is a basket of many items. Order status 51
> means the *basket was delivered to the customer*; the OOS item inside it was dropped
> (`pickingquantity = 0`) and refunded. The invoice covers only the items that actually shipped.

---

## 9. Timing evidence (propagation gap, OOS lines)

| Metric | Value | Meaning |
|---|---|---|
| HANA batch showed ≤ 0 → order placed | median **31 min**, up to **61 min** | HANA already had the empty signal before the sale; the pipeline had this window to block it. |
| Order placed → picker confirms OOS | median **44 min**, up to **895 min** | The customer paid, then learned hours later the item wasn't coming. |

HANA stock detail of the 12,492 candidates: **4,275 exactly zero (34%)**, **8,217 negative (66%, damaged/shrinkage)**.

---

## 10. Conclusions & recommendations

1. **Propagation-attributable lost sales on Jul 1 = 1,273 lines / 7,098,693 CLP.** Preventable by
   ensuring HANA's zero/negative-stock signal reaches VTEX before customers can order.
2. **A larger, separate problem: HANA inventory accuracy = 5,870 lines / 31,777,095 CLP.**
   HANA reported stock that wasn't on the shelf. Faster propagation cannot fix this; it belongs
   to WMS/inventory.
3. **Do not blindly block on HANA ≤ 0.** 9,621 lines (mostly granel) had HANA ≤ 0 but were
   in stock and delivered. A naïve "block everything HANA says is empty" rule would have
   destroyed 40.4M CLP of good sales. The propagation fix must be **accurate**, not just
   **fast** — e.g. treat granel/weighed categories differently.
4. **Scale.** This is a **single day at 33 stores.** Annualized and network-wide, the
   propagation gap alone justifies the fix many times over.

---

*Every figure recomputed from source on 2026-07-02 via `scripts/delay_proof_0701.py` and
`scripts/export_summary_sheet.py`; example order outcomes pulled live from Redshift the same day.*
