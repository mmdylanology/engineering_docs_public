# JUMBO Inventory OOS Investigation — System Map & Findings

> Written 2026-07-02. Covers end-to-end trace of darkstore OOS orders across HANA, VTEX, and Cencommerce Redshift.

---

## 1. The Three Systems

```
HANA (ERP/WMS)        →   GIV pipeline   →   VTEX (ecommerce)
Batch parquets in S3      periodic sync       Customer-facing catalog
                                              + SNS stock events

                              ↓ (Cencommerce Redshift picks up all order activity)
                          io_internal_orders / io_items / vw_master_pickers_data
```

| System | Role | Source |
|---|---|---|
| **HANA** | ERP/WMS — physical inventory snapshots | `data_samples/vw_daily_nrt_MMDD_HHMM/*.parquet` |
| **VTEX** | Ecommerce platform — accepts customer orders, manages its own stock counter | `analysis/vtex_events_*.jsonl` (SNS archive) |
| **Cencommerce Redshift** | Order management + picker workflow | `jumbo_bo.*` tables in `datalake` DB |

---

## 2. HANA Parquets (`vw_daily_nrt_MMDD_HHMM`)

### What it is
Hourly full snapshot of HANA's `Ending_On_Hand_Qty` per (item, location). Cencosud exports these from HANA to S3 via AWS Glue (~4-5 min lag after the HANA snapshot).

### Key fields
| Field | Type | Description |
|---|---|---|
| `Item_Id` | string | 18-digit zero-padded SAP material number. e.g. `000000000001964576` |
| `Location_Id` | string | Store code with J-prefix for Jumbo. e.g. `J411`, `J414` |
| `Ending_On_Hand_Qty` | decimal | Stock quantity per item per store at snapshot time |
| `Calendar_Dt` | date | Date of the snapshot (CLT — Chile local time) |
| `Load_Time` | string | Time of the HANA snapshot in CLT (`HH:MM:SS`) |

### Folder naming convention
`vw_daily_nrt_MMDD_HHMM` where MMDD = Calendar_Dt (CLT date), HHMM = Load_Time (CLT hour:min)

Example: `vw_daily_nrt_0622_0741` = Jun 22 snapshot at 07:40 CLT

### Jun 22 batches available locally
```
vw_daily_nrt_0622_0540  → 05:40 CLT
vw_daily_nrt_0622_0641  → 06:41 CLT
vw_daily_nrt_0622_0741  → 07:40 CLT
vw_daily_nrt_0622_0940  → 09:40 CLT
```

### How to query
```python
import duckdb, glob
item_id = '1964576'.zfill(18)  # zero-pad refid to 18 chars
parquets = glob.glob('data_samples/vw_daily_nrt_0622_0741/*.parquet')
con = duckdb.connect()
con.execute(f"""
    SELECT Item_Id, Location_Id, Load_Time, Ending_On_Hand_Qty
    FROM read_parquet({parquets!r})
    WHERE Item_Id = '{item_id}' AND Location_Id = 'J411'
""").df()
```

### Important caveats
- `Ending_On_Hand_Qty` = ERP book stock, NOT necessarily physical shelf stock
- ERP can show 17 units while the physical shelf is empty (committed/reserved stock)
- ~4-5 min glue gap between HANA snapshot and S3 availability

---

## 3. VTEX SNS Events (`vtex_events_*.jsonl`)

### What it is
VTEX fires SNS stock events whenever its internal stock counter changes — either because a customer ordered (depletion) or because a GIV sync pushed new stock numbers from HANA.

### Format (one JSON object per line)
```json
{
  "receivedAt": 1750585215000,
  "attributes": {
    "skuId": "127862",
    "storeId": "jumboclj411",
    "outOfStock": false,
    "stockLevel": "in-stock"
  }
}
```

### Key fields
| Field | Description |
|---|---|
| `receivedAt` | Unix milliseconds UTC — when the event was received |
| `attributes.skuId` | VTEX SKU ID — maps directly to `io_items.sku` in Redshift |
| `attributes.storeId` | VTEX store code e.g. `jumboclj411` |
| `attributes.outOfStock` | `true` = OOS event, `false` = back in stock |
| `attributes.stockLevel` | `"out-of-stock"` or `"in-stock"` |

### Store ID mapping
```
VTEX storeId:    jumboclj411  →  strip "jumboclj"  →  J411
HANA Location_Id: J411
Redshift storename: J411
```

### How to query
```python
import json, glob
from datetime import datetime, timezone, timedelta

CLT = timezone(timedelta(hours=-4))
sku = '127862'
store = 'jumboclj411'

for f in sorted(glob.glob('analysis/vtex_events_*.jsonl')):
    with open(f) as fp:
        for line in fp:
            ev = json.loads(line)
            a = ev.get('attributes', {})
            if str(a.get('skuId','')) == sku and a.get('storeId','') == store:
                ts_clt = datetime.fromtimestamp(ev['receivedAt']/1000, tz=timezone.utc).astimezone(CLT)
                print(ts_clt, a['outOfStock'], a['stockLevel'])
```

### Important notes
- Multiple duplicate events fire per stock change (3-5 at same timestamp) — deduplicate by timestamp
- `outOfStock=False` events fire far more frequently than OOS events
- VTEX events are NOT GIV-originated — they are VTEX's own stock tracking events, triggered by:
  1. Customer orders (VTEX depletes its counter)
  2. GIV sync pushes HANA stock → overrides VTEX counter

---

## 4. Cencommerce Redshift Tables

**Connection:** `172.22.194.225:5439`, database=`datalake`, schema=`jumbo_bo`  
**Credentials:** in `infrastructure/mcp_server/.env` as `CC_RS_*`

### 4.1 `jumbo_bo.io_internal_orders`

**What it is:** One row per order. The authoritative "orders received by VTEX" table. This is the correct denominator for "how many orders did VTEX accept."

**Key fields:**
| Field | Description |
|---|---|
| `idorder` | Order ID e.g. `v231531616jmch-01` — joins to all other tables |
| `storename` | Store code e.g. `J414` |
| `creationdate` | **When customer placed the order** (CLT) — most important timestamp |
| `assigndate` | When the order was assigned to a picker (CLT) |
| `starttime` | When picker started working the order (CLT) |
| `updatedat` | Last update to the order (CLT) |
| `amountinvoice` | Total order value in CLP |
| `itemsquantity` | Number of distinct SKUs ordered |
| `selectedsla` | Delivery type e.g. `Despacho a Domicilio` |
| `status` | Order status code (51 = delivered/completed) |

**Scope:** All Jumbo stores (J-prefix) + other channels. Total ~72K orders vs ~64K in pickers data — the delta is non-J stores.

### 4.2 `jumbo_bo.vw_master_pickers_data`

**What it is:** One row per order. Picker workflow summary — how the picking went, who did it, found rate.

**Key fields:**
| Field | Description |
|---|---|
| `id_raw` | Order ID — same as `io_internal_orders.idorder` |
| `id_tienda` | Store code |
| `tienda` | Store name in Spanish |
| `fecha_creacion` | Order creation time (CLT) — mirrors `io_internal_orders.creationdate` |
| `estado` | Order state: `Entregado` = delivered |
| `items_solicitados` | Total units requested |
| `items_pickeados` | Total units actually picked |
| `items_faltantes` | Units that couldn't be found (OOS) |
| `found_rate` | `items_pickeados / items_solicitados × 100` |
| `inicio_picking` | Picker started (CLT) |
| `fin_picking` | Picker finished (CLT) |
| `fecha_facturacion` | Invoice generated (CLT) — order effectively closed |
| `venta_neta` | Net sale amount in CLP |

**Note:** `items_solicitados` counts total units; `itemsquantity` in io_internal_orders counts distinct SKUs. They differ when qty > 1 per SKU.

### 4.3 `jumbo_bo.io_items`

**What it is:** N rows per order (one per SKU line). Shows every product in the order and its outcome.

**Key fields:**
| Field | Description |
|---|---|
| `key` | Order ID — joins to `io_internal_orders.idorder` |
| `refid` | SAP material number (matches HANA `Item_Id` when zero-padded to 18 chars) |
| `sku` | **VTEX skuId** — maps directly to VTEX SNS events `attributes.skuId` |
| `name` | Product name in Spanish |
| `originalquantity` | Units requested by customer |
| `invoiceprice` | Price in centavos (divide by 100 for CLP) |
| `status` | Outcome code — see below |
| `dateupdate` | When this line was processed by picker (CLT) |
| `pickingquantity` | Units actually picked |

**Status codes:**
| Code | Meaning |
|---|---|
| `1` | ✅ Picked successfully |
| `3` | ❌ NO_STOCK |
| `5` | ❌ VALIDATE_NO_STOCK |
| `7` | ❌ RE_PICKING_VALIDATE_NO_STOCK |
| `14` | ❌ ROOM_DELETION |
| `4` | 🔄 Substituted |
| `11` | ➕ Added item |

**Monto formula:** `ROUND((originalquantity * invoiceprice) / 100.0, 0)` → CLP

---

## 5. ID Mapping Chain

```
VTEX sku (skuId)    ←→   io_items.sku          (direct — no mapping needed)
io_items.refid      ←→   HANA Item_Id           (zero-pad refid to 18 chars)
io_items.key        =    io_internal_orders.idorder
                    =    vw_master_pickers_data.id_raw
```

**Cross-reference example:**
```
Order:    v231531616jmch-01
Store:    J414  (VTEX: jumboclj414)
OOS item: refid=1699105  →  HANA Item_Id = 000000000001699105
          sku=89998      →  VTEX skuId = '89998'
```

---

## 6. Darkstore vs Normal Store

| | Darkstore (J408/J410/J411/J414) | Normal store |
|---|---|---|
| Picking speed | Same-day, within 2-4 hours of order | Can take 3-5 days |
| Order window | 24h | Scheduled delivery windows |
| OOS impact | Critical — order and pick same day | Order today, OOS problem days later |
| Examples found | J411 COSTANERA, J414 LA DEHESA | J521 LA SERENA (5 days lag) |

---

## 7. Full Order Timeline (how to read it)

```
io_internal_orders.creationdate   → Customer places order on VTEX
io_internal_orders.assigndate     → Order assigned to picker
vw_master_pickers_data.inicio_picking  → Picker starts
io_items.dateupdate               → Each SKU line processed (all at same time, batch commit)
vw_master_pickers_data.fin_picking     → Picker done
vw_master_pickers_data.fecha_facturacion → Invoice generated
io_internal_orders.updatedat      → Order closed
```

---

## 8. Case Studies

### Case Study 1 — Phantom Inventory
**Order:** `v231512019jmch-01` | **Store:** J411 DARKSTORE COSTANERA | **Date:** Jun 22

| Timestamp (CLT) | Event | Source |
|---|---|---|
| 09:00:09 | Customer orders | io_internal_orders |
| 09:00:15 | VTEX fires: `outOfStock=False, in-stock` | VTEX SNS |
| 09:40 | HANA batch: qty=**17** at J411 | HANA parquet |
| 10:25:08 | Picker assigned & starts | vw_master_pickers_data |
| 10:36:51 | Picker processes items | io_items |
| 10:44:07 | Picking done | vw_master_pickers_data |
| 11:18 | VTEX still fires: in-stock | VTEX SNS |
| Jun 23 10:08 | VTEX still fires: in-stock | VTEX SNS |

**OOS item:** `refid=1964576` / `sku=127862` — Acondicionador Herbal Essences 865ml — 8,704 CLP

**Root cause — Phantom Inventory:**
- HANA showed 17 units → GIV never triggered OOS → VTEX showed in-stock all day
- Physical shelf was empty; HANA's ERP count (17) ≠ physical reality
- VTEX never sent a single OOS event for this SKU+store, even after picker confirmed OOS
- Any subsequent customer could order this same item and also get OOS at picking

---

### Case Study 2 — GIV Override of Correct VTEX Depletion
**Order:** `v231531616jmch-01` | **Store:** J414 JUMBO LA DEHESA | **Date:** Jun 22

| Timestamp (CLT) | Event | Source |
|---|---|---|
| 06:41 | HANA: **95 units** at J414 | HANA parquet |
| 07:40 | HANA: **17 units** (78 sold/consumed) | HANA parquet |
| 07:48 | VTEX: in-stock | VTEX SNS |
| 08:49 | VTEX: **`outOfStock=True`** ← VTEX counter hits zero from orders | VTEX SNS |
| 09:40 | HANA: still **17 units** (ERP committed, not physical) | HANA parquet |
| 10:48 | VTEX: **`in-stock`** ← GIV sync pushes HANA's 17 → overrides VTEX's zero | VTEX SNS |
| 12:36:47 | Customer orders ← VTEX shows in-stock (false) | io_internal_orders |
| 12:36:49 | VTEX: in-stock (at moment of order) | VTEX SNS |
| 14:47 | Picker starts | vw_master_pickers_data |
| 15:03:47 | Picker confirms OOS | io_items |
| 14:56 | VTEX: still in-stock (never corrected) | VTEX SNS |

**OOS items:**
- `refid=1699105` / `sku=89998` — Frambuesas Pote 70g — 2,190 CLP
- `refid=2045680` / `sku=156028` — Agua Saborizada Frambuesa lata — 1,100 CLP (phantom inventory case)

**Root cause — GIV Override:**
- VTEX correctly tracked orders depleting stock to zero at 08:49 — this is correct behavior
- GIV then ran a sync pushing HANA's 17 units (ERP/committed stock) → VTEX flipped back to in-stock
- Customer ordered 1h48m into that false in-stock window
- GIV is overwriting VTEX's accurate order-depletion tracking with lagging ERP book values

---

## 9. Failure Mode Summary

| Failure Mode | Root Cause | HANA shows | VTEX fires OOS? | Impact |
|---|---|---|---|---|
| **Phantom Inventory** | HANA ERP book stock ≠ physical shelf stock | Positive (wrong) | ❌ Never | Customers order indefinitely; no signal to fix |
| **GIV Override** | Stale HANA batch overwrites VTEX's correct zero | Positive (committed, not physical) | ✅ Yes, then reversed by GIV | Customer orders in false in-stock window |

Both result in the same outcome: picker confirms OOS, customer disappointed.

---

## 10. Quantified Impact (Jun 22-24)

From `exports/cencommerce_quiebre/quiebre_2026-06-22_to_2026-06-24_FULL.csv`:
- **21,620** OOS order lines across all stores
- **7,833** OOS lines from darkstores (J408/J410/J411/J414)
- **5,803** distinct darkstore orders with at least 1 OOS item

From `analysis/preventable_oos_proof_FINAL_60min.xlsx`:
- **1,574** proven preventable OOS events (HANA showed zero ≥60min before order)
- 100/100 spot-check passed against live Redshift
