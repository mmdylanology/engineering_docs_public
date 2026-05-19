# S3 / Trino Data Catalog — Jumbo Chile

> Authored 2026-05-06. Last validated: **2026-05-19**. Covers all data sources accessible via the Cencosud EKS
> `pro-evolution` cluster, `composable-supply-chain` namespace.
>
> Access method: AWS SSO `production` profile → AssumeRole
> `arn:aws:iam::103413823818:role/EKSDefaultPolicyFor-Composable-Supply-Chain` →
> `kubectl exec` into Trino coordinator pod.
>
> See `infrastructure/CENCOSUD_CLUSTER_QUICKSTART.md` for connection instructions.

---

## Architecture Overview

```
S3 buckets  ──►  Hive Metastore  ──►  Trino (hive catalog)  ──►  You
                                                  │
                        Direct S3 access ─────────┘
                     (GetObject on specific prefixes)
```

**Trino catalogs:** `hive` (only one relevant)
**Trino coordinator pod:** rotates on deploy — always look up with label selector:
```bash
kubectl get pods -n composable-supply-chain \
  -l "app.kubernetes.io/name=trino,app.kubernetes.io/component=coordinator" \
  -o jsonpath="{.items[0].metadata.name}"
```
**Trino workers:** 2 (as of 2026-05-06)

---

## S3 Buckets — Access Map

| Bucket | Encryption | ListBucket | GetObject | Prefixes Granted | Notes |
|--------|-----------|-----------|-----------|-----------------|-------|
| `cencosud.prod.sm.cl.raw` | SSE-S3 | ✅ (prefix-scoped) | ✅ | `CHI_SUPER_NRT_TB/FACT_IOSA_NRT/*`, `CHI_SUPER_DIM_TB/FACT_IN_STOCK/*` | Primary NRT + daily inventory |
| `cencosud.prod.sm.cl.raw.new` | SSE-S3 | ✅ (prefix-scoped) | ✅ | `CHI_SUPER_REL_TB/ITEM_INVENTORY/*` | Item inventory snapshots |
| `cencosud.prod.sm.cl.analytics` | SSE-KMS (CMK) | ✅ (prefix-scoped) | ✅ | `CHI_SUPER_DATASHARE_DIM_VW/VW_FACT_DAILY_INVENTORY_NRT/*` | ✅ KMS fixed 2026-05-18 by Gustavo Tejeda. Cross-account datashare from acct 265763412542 |
| `cencosud.prod.smds.cl.analytics` | SSE-KMS | ✅ (prefix-scoped) | ✅ | `STOCK_ADJUSTMENT/*` | ✅ KMS fixed 2026-05-12. Contains MASTER/, RAW/, STAGING/ sub-prefixes |
| `cencosud.prod.sm.cl.ldm` | ? | ❌ | ❌ | — | POS `sales_transaction_line` lives here — no access |
| `cl-jumbo-tools-sm` | SSE-S3 | ❌ | Via Trino IRSA | Historical prefixes | Historical ecommerce raw (2020) |
| `cencosud.prod.ccom.cl.raw` | SSE-S3 | ❌ | Via Trino IRSA | — | Cencommerce JSON orders (via Trino) |
| `data-lake-cencosud-dev` | SSE-S3 | ❌ | Via Trino IRSA | — | Legacy BCV/POS NRT (2018–2021) |

> **Note**: ListBucket is prefix-scoped via IAM condition. Root `aws s3 ls s3://bucket/` fails, but `aws s3 ls s3://bucket/GRANTED_PREFIX/` works. Trino only needs GetObject (uses Hive Metastore for paths) and never touches ListBucket.

**KMS key** (smds.cl.analytics): `arn:aws:kms:us-east-1:265763412542:key/9e7d4c72-b9df-46cb-b62b-89843c63cbf6` (in Cencosud IT acct 265763412542)

---

## Live Inventory Data (PRIMARY USE)

These are the most valuable tables for supply chain analytics — daily/NRT data.

### 1. FACT_IN_STOCK — Daily Inventory Snapshot

| Property | Value |
|----------|-------|
| **S3 bucket** | `cencosud.prod.sm.cl.raw` |
| **S3 prefix** | `CHI_SUPER_DIM_TB/FACT_IN_STOCK/` |
| **Format** | Parquet (SSE-S3, ~715 MB/day) |
| **Partition key** | `calendar_dt=YYYY-MM-DD/` |
| **Coverage** | 2022-10-14 → present (daily, no gaps observed) |
| **Row count** | ~5.78M rows per day (item × location combinations) |
| **Trino table** | `hive.s3_access_test.fact_in_stock` (partitioned, registered 2026-05-19) — ⚠️ see note below |
| **Updated** | Daily (exact time TBD, file LastModified was 08:35 UTC on 2026-05-06) |

**Columns:**

| Column | Type | Description |
|--------|------|-------------|
| `item_id` | varchar | SKU / item identifier |
| `item_desc` | varchar | Product description (Spanish) |
| `item_class_cd` / `item_class_name` | varchar | Category (rubro) |
| `item_subclass_cd` / `item_subclass_name` | varchar | Sub-category |
| `location_id` | varchar | Store ID (e.g., `J988`, `N711`) |
| `location_name` | varchar | Store full name |
| `org_name` | varchar | Vendor/supplier name |
| `stock` | varchar | Stock value (CLP, qty × unit_price) |
| `stock_q` | varchar | Physical stock quantity (units) |
| `brand_cd` / `brand_name` | varchar | Brand code and name |
| `enabled_ind` | varchar | 1 = item active at this location |
| `channel_cd` | varchar | Sales channel code |
| `dispo_minex` | varchar | Disposición mínima de existencias (1=met) |
| `item_replenishment_type_cd` | varchar | `RR`=regular replenishment, `ND`=no demand, etc. |
| `min_safety_stock_qty` | varchar | Safety stock threshold |
| `operational_division_cd` | varchar | 1=food, 2=non-food, 3=fresh (approx) |
| `section_cd` | varchar | Store section |
| `top_venta` | varchar | Top seller flag (`NORMAL`, `TOP`) |
| `vendor_party_id` | varchar | Vendor party ID (links to SAP) |
| `calendar_year_id` | varchar | Year |
| `calendar_week_id` | varchar | ISO week (e.g., `202619`) |
| `day_of_week_num` | varchar | Day of week (1=Monday) |

**Sample rows (`hive.s3_access_test.fact_in_stock_latest`, store J988, 2026-05-12):**

| item_id | item_desc | item_class | location_id | location_name | stock | stock_q | brand | enabled | channel | replen_type | vendor_party_id |
|---|---|---|---|---|---|---|---|---|---|---|---|
| 000000001921242001 | BOCADO POSTRE ABUELA SOPROLE 120G, CHOCO | RUBRO POSTRES LÁCTEOS | J988 | J988-JUMBO SUPER PUERTAS DE CHICUREO | 0.0000 | 0.0000 | SOPROLE | 0 | 13 | ND | V_0084472400 |
| 000000001959602002 | YOGURT COLUN ORIGEN 120G, VAINILLA | RUBRO YOG Y L.CULTIVADAS | J988 | J988-JUMBO SUPER PUERTAS DE CHICUREO | 0.0000 | 0.0000 | COLUN | 1 | 13 | RR | V_0081094100 |
| 000000001960540002 | JALEA GOODNES MULTIPACK 4X110GR, MAN DUR | RUBRO POSTRES LÁCTEOS | J988 | J988-JUMBO SUPER PUERTAS DE CHICUREO | -20859.0000 | -17.0000 | NESTLE | 1 | 13 | RR | V_0090703000 |

> Note: `stock` is value in CLP, `stock_q` is units. Negative stock_q (e.g. `-17.0000`) indicates a correction/adjustment has driven it below zero — real scenario in Jumbo stores.

**Sample query (create Trino table for a specific day first):**
```sql
-- Register the table for a specific date:
CREATE TABLE IF NOT EXISTS hive.s3_access_test.fact_in_stock_20260505 (
  item_id varchar, item_desc varchar, item_class_cd varchar,
  item_class_name varchar, location_id varchar, location_name varchar,
  stock varchar, stock_q varchar, brand_name varchar,
  enabled_ind varchar, top_venta varchar
)
WITH (
  external_location = 's3://cencosud.prod.sm.cl.raw/CHI_SUPER_DIM_TB/FACT_IN_STOCK/calendar_dt=2026-05-05/',
  format = 'PARQUET'
);

-- Out-of-stock items (enabled but no stock):
SELECT location_name, COUNT(*) AS oos_items
FROM hive.s3_access_test.fact_in_stock_20260505
WHERE enabled_ind = '1' AND (stock_q = '0.0000' OR stock_q IS NULL)
GROUP BY location_name
ORDER BY oos_items DESC;
```

---

### 2. FACT_IOSA_NRT — On-Shelf Availability (Near Real-Time)

| Property | Value |
|----------|-------|
| **S3 bucket** | `cencosud.prod.sm.cl.raw` |
| **S3 prefix** | `CHI_SUPER_NRT_TB/FACT_IOSA_NRT/` |
| **Format** | Parquet (SSE-S3, ~113 MB) |
| **Partition** | Single file, replaced nightly (not date-partitioned) |
| **Last seen** | 2026-05-18 (live NRT) |
| **Row count** | ~3.27M rows (item × location) — validated 2026-05-18 |
| **Trino table** | `hive.s3_access_test.fact_iosa_nrt` (manual registration) |

**Columns:**

| Column | Description |
|--------|-------------|
| `item_id` | Item/SKU identifier |
| `location_id` | Store ID |
| `stock_qty` | Current stock quantity |
| `iosa` | IOSA flag: 1=inventory count verified accurate, 0=not verified |
| `instock` | In-stock flag: 1=has stock, 0=out of stock |
| `calendar_dt` | Date of measurement (YYYY-MM-DD) |
| `hora` | Time of last measurement (NRT — updates throughout day) |
| `cplanif` | Replenishment planning type (`RR`, `YN`, etc.) |
| `vendor_party_id` | Vendor ID |

**Observed metrics (latest 2026-05-18, cross-validated vs Redshift 2026-05-12):**

| Metric | S3/Trino (2026-05-12) | Redshift `fact_in_stock_latest` | Delta |
|--------|---------------|----------------------------------|-------|
| Total rows | 3,294,696 | ~3,300,000 | -5k (-0.15%) |
| Unique SKUs | 34,335 | — | — |
| Unique stores | 296 | — | — |
| **In-stock rate** | **93.8%** | **93.4%** | +0.4pp |
| **IOSA accuracy** | **81.5%** | **81.1%** | +0.4pp |
| Latest date | 2026-05-12 | 2026-05-12 | ✅ same day |

> ✅ **Cross-validated 2026-05-12**: S3/Trino and Redshift are consistent within 0.4pp. The delta is explained by snapshot timing (S3 captured later in the day).

**Sample rows (2026-05-12):**

| item_id | location_id | stock_qty | iosa | instock | calendar_dt | hora | cplanif | vendor_party_id |
|---------|-------------|-----------|------|---------|-------------|------|---------|-----------------|
| 000000000000260641 | J403 | 8.0000 | 1 | 1 | 2026-05-12 | 17:10:08 | YN | V_0078610160 |
| 000000000000260698 | J403 | 3.0000 | 1 | 1 | 2026-05-12 | 17:10:08 | YN | V_0078610160 |
| 000000000000260711 | J403 | 11.0000 | 1 | 1 | 2026-05-12 | 17:10:08 | YN | V_0078610160 |

**Key use:** On-shelf availability (OSA) KPI. Measures whether inventory records match physical shelf reality. Direct input to the "Fill Rate" and "Sin Stock" metrics used in Cencommerce reports.

```sql
-- IOSA accuracy by store:
SELECT location_id,
       SUM(CASE WHEN iosa='1' THEN 1 ELSE 0 END) AS iosa_accurate,
       COUNT(*) AS total,
       ROUND(100.0 * SUM(CASE WHEN iosa='1' THEN 1 ELSE 0 END) / COUNT(*), 2) AS pct
FROM hive.s3_access_test.fact_iosa_nrt
GROUP BY location_id
ORDER BY pct ASC
LIMIT 20;
```

---

### 3. ITEM_INVENTORY — Stock Levels Snapshot

| Property | Value |
|----------|-------|
| **S3 bucket** | `cencosud.prod.sm.cl.raw.new` |
| **S3 prefix** | `CHI_SUPER_REL_TB/ITEM_INVENTORY/` |
| **Format** | Parquet (279 MB) |
| **Partition** | `item_inv_dt=2024-04-17/` (single snapshot — not regularly updated) |
| **Row count** | 6,845,788 rows |
| **Trino table** | `hive.s3_access_test.item_inventory` (registered) |

**Columns:** `item_id`, `location_id`, `base_uom_cd`, `begin_on_hand_unit_qty`, `end_on_hand_unit_qty`, `unit_cost_amt`, `unit_price_amt`, `total_cost_amt`, `stock_type_cd`, `storage_area_cd`, `special_stock_type_id`, `inventory_trx_source_cd`, `loadnbr`

**Sample rows (2024-04-17 snapshot, store N764):**

| item_id | location_id | base_uom_cd | begin_on_hand | end_on_hand | unit_cost | unit_price | total_cost | stock_type | storage_area |
|---|---|---|---|---|---|---|---|---|---|
| 000000000001946759 | N764 | UN | 2.0000 | 2.0000 | 1250.0000 | 490.0000 | 2500.0000 | B | 0001 |
| 000000000001946864 | N764 | UN | 19.0000 | 19.0000 | 1609.0000 | 3990.0000 | 30562.0000 | B | 0001 |
| 000000000001947036 | N764 | UN | 10.0000 | 10.0000 | 1840.0000 | 3059.0000 | 18395.0000 | B | 0001 |

**Note:** Only one partition exists (2024-04-17). Not a live feed — this appears to be a one-time extract. Use FACT_IN_STOCK for current inventory state.

---

### 4. STOCK_ADJUSTMENT — Vendor Stock Corrections

| Property | Value |
|----------|-------|
| **S3 bucket** | `cencosud.prod.smds.cl.analytics` |
| **S3 prefix** | `STOCK_ADJUSTMENT/MASTER/` (Parquet) and `STOCK_ADJUSTMENT/RAW/` (XLSX) |
| **Format** | Parquet (MASTER — processed), XLSX (RAW — vendor uploads) |
| **Coverage** | 2025-01-12 → 2026-04-29 |
| **Row count** | 144,754 rows in MASTER/ (validated 2026-05-18) |
| **Trino table** | `hive.s3_access_test.stock_adjustment_sample` |
| **KMS** | ✅ `kms:Decrypt` granted — fully accessible |
| **Updated** | Per vendor upload event (not daily batch) |

**Columns:**

| Column | Type | Description |
|--------|------|-------------|
| `supplier_dni` | varchar | Vendor ID (e.g. `V_0082623500`) |
| `location_id` | varchar | Store ID |
| `item_id` | double | SKU identifier |
| `scan_cd` | varchar | EAN barcode |
| `umv` | varchar | Unit of measure |
| `dia` | timestamp(3) | Date of adjustment |
| `hora` | varchar | Time of adjustment |
| `reason` | varchar | Adjustment reason (e.g. `STOCK FANTASMA`, `Stock negativo`) |
| `stock_supplier` | varchar | Stock level reported by supplier |
| `stock_adjust` | varchar | Adjusted stock value |
| `usuario` | varchar | User who submitted |
| `channel` | varchar | `MASSIVE` (bulk upload) or individual |
| `requested_at` | varchar | Submission timestamp |
| `id` | varchar | UUID of the adjustment record |
| `status` | varchar | `REJECTED`, `APPROVED`, `PENDING` |
| `tm_processed` | boolean | Whether TM (supply chain system) processed it |
| `tm_reason` | bigint | TM processing reason code |
| `tm_status` | bigint | TM status code |
| `tm_updated_by` | varchar | `SMDS_STOCK_ADJ_ETL` (the ETL process) |
| `stock_nrt` | double | NRT stock at time of adjustment |

**Sample rows (2026-03-02, REJECTED adjustments from vendor V_0081094100):**

| supplier_dni | location_id | item_id | scan_cd | umv | dia | hora | reason | stock_supplier | stock_adjust | status | tm_processed | stock_nrt |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| V_0081094100 | J503 | 1.960556002E9 | 78028746 | UN | 2026-03-02 | 11:20:00 | Stock negativo | -96 | 0 | REJECTED | true | -1.0 |
| V_0081094100 | N634 | 1.965565006E9 | 78031913 | UN | 2026-03-02 | 11:20:00 | Stock negativo | -96 | 0 | REJECTED | true | -1.0 |
| V_0081094100 | N636 | 1.960556002E9 | 78028746 | UN | 2026-03-02 | 11:20:00 | Stock negativo | -96 | 0 | REJECTED | true | -1.0 |

> `item_id` is stored as `double` due to schema variance across vendor files (some vendors write int64, others double). Cast to bigint or varchar for joins: `CAST(CAST(item_id AS bigint) AS varchar)`.

**Key stats (2026-05-12):**
- **12 unique suppliers**, **261 stores**, **2,389 unique SKUs**
- Date range: 2025-01-12 → 2026-04-29
- RAW/ prefix: XLSX files are raw vendor uploads (same records, pre-ETL)
- MASTER/ prefix: ETL-processed Parquet — use this for analysis

**Sample query:**
```sql
-- Adjustment volume by supplier and status:
SELECT
  supplier_dni,
  status,
  COUNT(*) AS adjustments,
  COUNT(DISTINCT location_id) AS stores,
  CAST(MAX(dia) AS varchar) AS latest
FROM hive.s3_access_test.stock_adjustment_sample
GROUP BY supplier_dni, status
ORDER BY adjustments DESC;
```

---

### 5. VW_FACT_DAILY_INVENTORY_NRT — Daily Inventory View (Cross-Account Datashare)

| Property | Value |
|----------|-------|
| **S3 bucket** | `cencosud.prod.sm.cl.analytics` |
| **S3 prefix** | `CHI_SUPER_DATASHARE_DIM_VW/VW_FACT_DAILY_INVENTORY_NRT/` |
| **Format** | Parquet, Snappy compression |
| **Source account** | Cencosud IT (`265763412542`) — cross-account data share |
| **Files** | 36 Parquet files, ~538 MB total |
| **Generated** | 2026-05-18 ~16:44–16:45 UTC (run ID: `1779122559192`) |
| **Row count** | **58,277,212 rows** |
| **Trino table** | `hive.s3_access_test.vw_daily_inventory_nrt` |
| **KMS status** | ✅ Fixed 2026-05-18 (Gustavo Tejeda migrated to CMK) |

**Columns:**

| Column | Type | Notes |
|--------|------|-------|
| `item_id` | varchar | SKU identifier |
| `location_id` | varchar | Store ID |
| `stock_qty` | varchar | Stock quantity |
| `calendar_dt` | DATE (Parquet) | ⚠️ Registered as `varchar` in Hive — causes query error if selected. Fix: recreate table with `calendar_dt date` |

**Sample rows (2026-05-18):**

| item_id | location_id | stock_qty |
|---------|-------------|-----------|
| 000000010000098562 | A047 | (empty) |
| 000000000000054575 | A047 | (empty) |
| 000000000000059788 | A047 | (empty) |

> Note: `location_id = A047` suggests this datashare may cover additional store formats beyond standard Jumbo `J*/N*` IDs. Also note `stock_qty` is empty in visible sample rows — the column appears sparsely populated or the sample pulled zero-stock records.

**✅ Fixed 2026-05-19** — table recreated with `calendar_dt date`. Full `SELECT *` works. Note: `calendar_dt` returns NULL for all rows — the column is not embedded in the Parquet files (single-snapshot export, date is implicit from export run).

**Use case:** Broadest inventory view available — 58M rows vs 3.3M in FACT_IOSA_NRT. Likely covers all Cencosud banners (not just Jumbo), all store types. Compare against FACT_IN_STOCK for reconciliation.

---

## Ecommerce Order Data

### 4. jsonrequest_orders — Raw VTEX Order Payload

| Property | Value |
|----------|-------|
| **S3 bucket** | `cencosud.prod.ccom.cl.raw` |
| **S3 prefix** | `jumbo/cl/backoffice/jsonrequest_orders/` |
| **Format** | OPENX_JSON (AVRO-embedded schema) |
| **Trino table** | `hive.chi_super_dl_cs_ecommerce.jsonrequest_orders` |
| **Partitioned by** | `year` / `month` / `day` / `hour` |
| **Coverage** | 2020-01 → 2021-08 (~14,129 total partitions at hour level) |
| **Updated** | ⚠️ Last data: August 2021 — NOT live |

**Structure:** Each row is a complete VTEX order JSON blob. The `_source` field is a deeply nested `ROW` type containing the full order detail, including:
- Order metadata: `id`, `status`, `creationdate`, `saleschannel`, `value`
- Line items: `items[]` (sku, ean, refid, name, quantity, price, listprice, measurementunit, hall, externalinfo)
- Shipping: `shippingdata` (address, logisticsinfo with SLAs and delivery windows)
- Payment: `paymentdata` (transactions, payment systems)
- Customer: `clientprofiledata` (document/RUT, email, name)
- Picking: `pickingtype`, `pickingestimatedtime`, `starttime`
- Promotions: `ratesandbenefitsdata`
- Changes: `changesattachment` (substitutions, add/remove items)

**Key query patterns:**
```sql
-- Access items array from a specific order (Trino nested syntax):
SELECT _id,
       _source.detail.status,
       _source.detail.saleschannel,
       _source.detail.creationdate,
       item.refid,
       item.name,
       item.quantity,
       item.price
FROM hive.chi_super_dl_cs_ecommerce.jsonrequest_orders
CROSS JOIN UNNEST(_source.detail.items) AS t(item)
WHERE year='2021' AND month='08' AND day='01'
LIMIT 10;
```

> ⚠️ Data ends 2021-08. For current order data use **Cencommerce Redshift** (`jumbo_bo.io_internal_orders`, `jumbo_bo.vw_master_data`).

---

### 5. fact_insellout — Order Fill Rate (Historical)

| Property | Value |
|----------|-------|
| **S3 bucket** | `cl-jumbo-tools-sm` |
| **S3 prefix** | `DL_CS_ECOMMERCE_RAW/FACT_INSELLOUT/` |
| **Format** | AVRO |
| **Trino table** | `hive.chi_super_dl_cs_ecommerce.fact_insellout` |
| **Coverage** | 2020-07 only (single partition, historical) |

**Columns** (key fields): `store_id`, `store_title`, `order_op_number`, `order_created_date`, `order_invoiced_date`, `order_status`, `main_item_sku`, `main_item_refid`, `main_item_ean`, `main_item_name`, `main_item_unit`, `main_item_list_price`, `main_item_selling_price`, `main_item_requested`, `main_item_picked`, `main_item_shortage`, `is_substitute`, `shipping_date`, `shipping_zone`, `shipping_commune`, `shipping_region`, `shipping_type`, `main_brand_name`, `main_category_name`, `venta_ingresada`, `venta_facturada`, `was_replaced`, `data_source`

**Purpose:** Item-level order fill rate — shows what was requested vs. picked, plus shortage flag. The predecessor to Cencommerce `io_items`. Most complete fill rate source in S3/Trino but only covers 2020-07.

---

### 6. faltantebrutoitem / faltantebrutoitem_p2 — Raw Picking Shortage

| Property | Value |
|----------|-------|
| **S3 bucket** | `cl-jumbo-tools-sm` |
| **S3 prefixes** | `DL_CS_ECOMMERCE_RAW/FALTANTEBRUTOITEM/`, `FALTANTEBRUTOITEM_P2/`, `Faltantes3/` |
| **Format** | AVRO (single 2.4 GB file each) |
| **Trino tables** | `hive.chi_super_dl_cs_ecommerce.faltantebrutoitem`, `faltantebrutoitem_p2`, `faltantes3` |
| **Coverage** | 2020-07-21 only (historical, single partition) |
| **Status** | ⚠️ Hive partition fetch fails — S3 path confirmed but metastore needs refresh |

**Columns** (key fields): `picking_round`, `picking_start1/2/3`, `picking_end`, `sales_channel`, `seq_id` (order ID), `date_created`, `status`, `product`, `ref_id`, `sku`, `quantity`, `main_item_picked_plain/spec/gram`, `main_item_difference`, `repl_item_substitution`, `repl_item_name`, `repl_item_picking`

`faltantebrutoitem_p2` adds: `invoice_date`, `item_requested`, `item_picked`

**Purpose:** Raw picking session data showing item-level shortage during picking. Source for the `vw_fact_ecommerce_faltante` view in Cencosud IT Redshift. Current equivalent is **Cencommerce Redshift** `io_items`.

---

### 7. eventos_comerciales — Commercial Events / Order Headers

| Property | Value |
|----------|-------|
| **S3 bucket** | `cl-jumbo-tools-sm` |
| **S3 prefix** | `DL_CS_ECOMMERCE_RAW/EVENTOS_COMERCIALES/` |
| **Format** | AVRO |
| **Trino table** | `hive.chi_super_dl_cs_ecommerce.eventos_comerciales` |
| **Coverage** | 2020-07 only |

**Columns:** `id_pedido`, `fcreacion`, `hcreacion`, `fech_despacho`, `tipo_cliente`, `rut_clie`, `cod_prod1`, `descripcion`, `cant_solic`, `precio`, `total_precio_sku`, `nom_clie`, `direccion`, `comuna`, `transportadora`, `medio_pago`, `monto_reservado`, `estado_op`, `store`, `canal`, `fech/hora_ventana_ini/fin`, `servicio`, `costo_despacho`, `data_source`

**Purpose:** Order-level summary — customer, delivery window, store, channel, payment method. Historical counterpart to Cencommerce `io_internal_orders`.

---

## POS Transaction Data

### 8. sales_transaction_line — POS Line Items (Near Real-Time)

| Property | Value |
|----------|-------|
| **S3 bucket** | `cencosud.prod.sm.cl.ldm` |
| **S3 prefix** | `chi_super/chi_super_rel_tb/sales_transaction_line/` |
| **Format** | Parquet |
| **Trino table** | `hive.chi_super_rel_tb_nrt.sales_transaction_line` |
| **Partitioned by** | `year` / `month` / `date` / `hour` |
| **Hive metastore partitions** | Shows 2020-02 (174 partitions) — **actual S3 data may be more recent** |
| **S3 access** | ⚠️ `cencosud.prod.sm.cl.ldm` is access denied — cannot verify dates directly |

**Key columns:** Full POS transaction line with SAP integration:
- `centro_cd`, `nro_caja`, `transaccion_nro` — store/register/transaction
- `fecha`, `hora`, `linea_nro` — timestamp and line number
- `codigo_barra_venta` — barcode scanned at POS
- `plu`, `item_id`, `pos_item_id` — item identifiers
- `cantidad`, `item_qty`, `item_qty_umb` — quantity in various UoMs
- `precio_unitario`, `actual_unit_selling_price_amt` — selling price
- `actual_total_amt` — line total
- **`matnr`** — **SAP material number** (direct join to SAP inventory tables)
- **`ean11`** — EAN barcode
- `meinh`, `umren`, `umrez`, `cant_calc` — SAP unit of measure conversions
- `tran_line_type_cd` — line type (sale, return, etc.)
- `sales_tran_id` — joins to `sales_transaction` header
- `location_id`, `pos_register_id` — store/register
- `fulfillment_type_cd` — fulfillment type (pickup, delivery, etc.)
- `sales_transaction_channel_cd` — channel (in-store, online, etc.)

**Use case:** POS → SAP join via `matnr`. Links in-store sales to SAP inventory levels.

```sql
-- Sales by SAP material number (requires Hive partition sync):
SELECT matnr, SUM(CAST(item_qty AS double)) AS total_qty_sold
FROM hive.chi_super_rel_tb_nrt.sales_transaction_line
WHERE year='2020' AND month='02' AND date='2020-02-15'
GROUP BY matnr
ORDER BY total_qty_sold DESC
LIMIT 20;
```

### 9. sales_transaction — POS Transaction Headers

| Property | Value |
|----------|-------|
| **Trino table** | `hive.chi_super_rel_tb_nrt.sales_transaction` |
| **S3 prefix** | `cencosud.prod.sm.cl.ldm/chi_super/chi_super_rel_tb/sales_transaction/` |
| **Format** | Parquet |

**Key columns:** `location_id`, `pos_register_id`, `tran_num`, `tran_start_dt/tm`, `tran_end_dt/tm`, `total_amt`, `sales_transaction_channel_cd`, `online_transaction_ind`, `voided_transaction_ind`, `primary_payment_type_cd`, `debtor_party_id` (customer RUT)

---

## Historical / Reference Data

### 10. BCV NRT — Legacy POS Data

| Property | Value |
|----------|-------|
| **S3 bucket** | `data-lake-cencosud-dev` |
| **Trino schema** | `hive.chile_pos` |
| **Coverage** | 2018 → 2021 (year-suffixed tables: `_year_2018`, `_year_2019`, etc.) |
| **Format** | AVRO |

Key tables: `bcv_nrt_cl_arcai_year_*` (ARCAI = register + item), `bcv_nrt_cl_nrt_clientes_frecuentes_year_*` (loyalty), `bcv_nrt_cl_nrt_encabezados_year_*` (transaction headers), `bcv_arts_cl_history_*` (transaction history)

**Note:** Superseded by `sales_transaction_line` in `chi_super_rel_tb_nrt`.

### 11. VTEX Historical Orders (2018–2019)

| Property | Value |
|----------|-------|
| **Trino schema** | `hive.chile_vtex` |
| **Tables** | ~70 tables: `ordenes_*`, `productos_*`, `sku_*`, `clientes_*`, `cupones_*`, etc. |
| **Coverage** | 2018–2019 only |

Key tables:
- `ordenes_year_2018/2019` — raw order headers
- `ordenes_detalle_orden_year_*` — order line items
- `productos_year_*` — product catalog
- `vista_detalle_ordenes` — view with: `precio`, `precio_lista`, `name`, `refid`, `quantity`, `orderid`, `saleschannel`, `status`, `userprofileid`, `document`

**Note:** Predates `jsonrequest_orders` format. Superseded by Cencommerce Redshift for current ecommerce data.

### 12. SAP Historical Data

| Property | Value |
|----------|-------|
| **Trino schema** | `hive.chile_sap` |
| **Tables** | `ean_year_*`, `deltas_year_*`, `slt_eina_year_*`, `interfaz_maestroproducto_*`, `interfaz_stock_*` |
| **Coverage** | 2018–2021 |

Primarily EAN ↔ SAP material number mapping files and stock interface files. Useful for historical item master joins.

### 13. Cencommerce Backoffice (JSON)

| Property | Value |
|----------|-------|
| **Trino schema** | `hive.ccom_bo` |
| **Tables** | `bojsonrequest_customers`, `bojsonrequest_customers_2` |
| **S3 bucket** | `cencosud.prod.ccom.cl.raw` |

Customer profiles from the Cencommerce backoffice system.

---

## Data Lineage & Cross-System Joins

```
VTEX (ecommerce orders)
    │
    ├─► jsonrequest_orders (S3/Trino) [2020-2021]
    │       └─► faltantebrutoitem (S3/Trino) [shortage during picking, 2020]
    │               └─► vw_fact_ecommerce_faltante (IT Redshift) [current]
    │                       └─► Cencommerce io_items [current, 638M rows]
    │
    └─► fact_insellout (S3/Trino) [order fill rate, 2020]

Physical stores (POS)
    │
    ├─► sales_transaction_line (S3/Trino) [BCV NRT feed, matnr=SAP]
    │       └─► Cencosud IT Redshift: sales_transaction_line (same data, Redshift copy)
    │
    └─► FACT_IOSA_NRT (S3, NRT) [on-shelf availability per item×store]

Inventory
    │
    ├─► FACT_IN_STOCK (S3, daily) [stock qty per item×store, 2022-present]
    │       └─► joins to IT Redshift: wh_sap.slt_hana_marc* (via item_id)
    │
    └─► ITEM_INVENTORY (S3, snapshot) [begin/end on-hand, 2024-04-17 only]

Key join keys:
    item_id  ──►  IT Redshift: wh_sap columns, pos.item_id
    matnr    ──►  SAP tables (chile_sap.ean_year_*, wh_sap.* in IT Redshift)
    ean11    ──►  product catalog, barcode lookup
    location_id  ──►  store master (IT Redshift: wh_sap.slt_hana_t001w_*)
```

---

## What We Have Access To vs. What We Don't

> Last verified: 2026-05-12 via `kubectl run s3-access-check` pod with IRSA role.

### ✅ Accessible (can query now)

| Dataset | Freshness | Rows | Best For |
|---------|-----------|------|---------|
| `FACT_IN_STOCK` | Daily (2022→present) | 5,778,471/day | Item×store inventory state, OOS analysis |
| `FACT_IOSA_NRT` | NRT (daily refresh) | 3,275,019 | On-shelf accuracy — 93.8% in-stock, 81.5% IOSA (validated) |
| `VW_FACT_DAILY_INVENTORY_NRT` | Daily (generated 2026-05-18) | **58,277,212** | Broadest inventory view — all banners. ✅ KMS fixed 2026-05-18 |
| `STOCK_ADJUSTMENT` | Per vendor upload event | 144,754 | Vendor stock correction analysis |
| `ITEM_INVENTORY` | 2024-04-17 snapshot only | 6,845,788 | begin/end on-hand, cost, price |
| `jsonrequest_orders` | 2020–2021 (historical) | — | Order structure, VTEX payload exploration |
| Cencommerce Redshift `vw_master_data` | Current | — | Order fill rate, picking KPIs |
| IT Redshift (via MCP) | Current | — | Full warehouse: SAP, BCV, VTEX joins |

### ⚠️ Accessible but blocked/stale

| Dataset | Issue |
|---------|-------|
| `sales_transaction_line` | S3 bucket `cencosud.prod.sm.cl.ldm` — access denied from our role |
| `faltantebrutoitem` | Data only 2020-07, Hive partition fetch fails |
| `VW_FACT_DAILY_INVENTORY_NRT` | ⚠️ Hive table has `calendar_dt varchar` but Parquet has `calendar_dt DATE` — type mismatch crashes full `SELECT *`. Use column-specific query or fix the table definition (see fix above). |
| Cencommerce `io_items` direct queries | WLM kills aggregates (638M rows, no SORTKEY) |

### ❌ No access

| Dataset | What it would give us |
|---------|----------------------|
| `cencosud.prod.sm.cl.ldm` (full) | Live POS sales_transaction data |
| IT Redshift S3 export bucket | Redshift UNLOAD outputs |
| `cl-jumbo-tools-sm/STAGING/` | Historical fill rate (STAGING prefix not granted) |
| `cl-jumbo-tools-sm/SURTIDO/` | Category hierarchy (SURTIDO prefix not granted) |

### 📋 Hive Schema Inventory (2026-05-18) — All 60 Schemas

> Discovered via `SHOW SCHEMAS FROM hive` on Trino pod `trino-coordinator-c45c844bd-tzxkl`, 2026-05-18.

**Active / queryable:**

| Schema | Tables | Content | Notes |
|--------|--------|---------|-------|
| `s3_access_test` | 6 | `fact_iosa_nrt`, `fact_in_stock`, `fact_in_stock_latest`, `item_inventory`, `stock_adjustment_sample`, `vw_daily_inventory_nrt` | ✅ Our primary query schema |
| `chi_super_dl_cs_ecommerce` | ~20 | VTEX orders, fill rate, picking data | 2020-2021 archive. jsonrequest_orders, faltantebrutoitem, fact_insellout |
| `chi_super_rel_tb_nrt` | 2 | `sales_transaction_line`, `sales_transaction` | Blocked: `cencosud.prod.sm.cl.ldm` access denied |
| `ccom_bo` | 2 | `bojsonrequest_customers` (×2) | Cencommerce backoffice customer profiles |
| `machine_learning` | 3 | `nrt_clientes`, `nrt_productos`, `vista_pos_ml_chile_nrt` | NRT Chile ML features — likely queryable via Trino |
| `nps` | 7 | `chile_estado_orden`, `chile_nps_nrt_disparos`/`2`, `chile_nps_nrt_tiendafisica`/`2`, `chile_nrt_tiendafisica`, `cl_logs` | NPS survey triggers + physical store NPS |

**Infrastructure / AWS logs:**

| Schema | Tables | Content |
|--------|--------|---------|
| `alb_db` | 2 | `alb_logs`, `smdigital` — ALB access logs |
| `waf_logs` | 1 | `chile_aws_waf_logs_access_2019` — WAF logs |
| `vpcflowlogsathenadatabasefl0064eb65a1c77369b` | 1 | VPC flow logs (2021-05-10) |
| `athenacurcfn_parquet_cur` | 2 | `parquet_cur`, `cost_and_usage_data_status` — AWS Cost & Usage Report |
| `sisa_103413823818_logs` | 1 | `access_logs` — SISA access logs |
| `metadatas3log` | 3 | `s3_log`, `staging_s3_log`, S3 metadata logs |
| `_hb8_inputs` | 4 | `cloudtrail_logs_logssec_prod`, `elb_logs`, CloudFront/Paris logs |
| `default` | 10 | `billing_cencommerce`, `cloudfront_perseo_logs`, CloudTrail logs (×4), `elb_logs`, `lista_cliente_migrados_20190212`, `mkp_ledger`, `waf_logs` |

**Historical data (2018–2019 archives):**

| Schema | Tables (approx) | Content |
|--------|----------------|---------|
| `chile_vtex` | ~70 | Chile VTEX orders, products, SKUs, customers (2018-2019) |
| `chile_analytics` | — | Chile Google Analytics / ecommerce BI |
| `chile_pos` | ~20 | BCV NRT POS data (2018-2021) |
| `chile_sap` | ~10 | SAP EAN/material mappings (2018-2021) |
| `chile_bdjumbo` | ~10 | Old Jumbo BO system (pre-VTEX) |
| `chile_interfacesvtex` | — | VTEX interface data |
| `chile_bnf` | — | BNF data |
| `chile_cmx` | — | CMX data |
| `chile_base_c` | — | Base customer data |
| `chile_janis` | — | Janis (push notification) data |
| `chile_rappi` | — | Rappi Chile orders |
| `chile_sybaseclientes` / `chile_sybaseiqclientes` | — | Legacy Sybase customer data |
| `chile_prisma` | — | Prisma payment data |
| `vtex` | 3 | `vista_producto_promociones`, `vista_producto_promociones_peru`, `vistaclientes_year_2018` |
| `_vtex` | 0 | Empty |
| `chile_control_tower_data_catalog` | — | Empty |

**Multi-country archives:**

| Schema | Tables | Country | Content |
|--------|--------|---------|---------|
| `argentina_analytics` | ~40 | AR | Google Analytics 2018 (Disco, Jumbo, Vea) |
| `argentina_vtex` | ~50 | AR | VTEX orders 2018-2019 |
| `colombia_analytics` | ~65 | CO | Google Analytics + CoreMetrics 2018 (Jumbo CO, JumboFood, JumboNonFood) |
| `colombia_vtex` | ~50 | CO | VTEX orders 2018-2019 |
| `colombia_cupones` | 1 | CO | `data_year_2018` |
| `colombia_puntos` | 1 | CO | `data_year_2018` |
| `peru_analytics` | ~70 | PE | Google Analytics + CoreMetrics 2018 (Wong) |
| `peru_vtex` | ~55 | PE | VTEX 2018-2019 (Wong, WongApp, WongProd) |
| `peru_saphana` | 1 | PE | `ventas_year_2019` — SAP Hana sales data |
| `peru_hb8_inputs` | 1 | PE | `ic_in_precios_201810_year_2019` — pricing data |
| `cl_pim_sm_data` | — | CL | PIM (Product Information Management) data |
| `cl_pim_sm_data_qa` | — | CL | PIM QA env |
| `cl-monitor-vtex` | — | CL | VTEX monitoring data |
| `cl-ccom-backoffice-easy-bo-db2-tables-history-db2-backups` | — | CL | Easy backoffice DB2 backup tables |
| `chi_easy_ext` | — | CL | Easy.cl external data |

**CRO / Other:**

| Schema | Tables | Content |
|--------|--------|---------|
| `cro_database` | 2 | `interactions_paris_cl`, `interactions_paris_cl_tmp` — Paris Chile CRO |
| `cro_database_raw` | 8 | `interactions_jumbo_co_4`, `items_jumbo_co_4`, `interactions_metro_co`, `items_metro_co`, `interactions_easy_co`, `items_easy_co` — Colombia multi-brand |
| `cro_database_stage` | 10 | Staging versions of cro_database_raw tables |
| `cro_database_analytics` | — | CRO analytics (empty in today's scan) |
| `redshift_bcv` | 1 | `csv_datalake_global_materialized_data_vista_historica_ventas_chile` — historical Chile sales |
| `nps` | 7 | NPS survey + physical store NPS triggers |
| `epay` | 1 | `refund` — ePay refund data |
| `epay_athena` | 1 | `epay_refund` |
| `devoluciones_payment` | 1 | `qa` — payment returns (QA data) |
| `ecommerce_bi` | 1 | `day_23` (STAGING prefix, access blocked) |
| `sampledb` | 2 | `elb_logs`, `mdw_comparar` — AWS sample + comparison |
| `paris-dataset` | 0 | Empty |
| `aurora` | — | Aurora DB tables |
| `ean-sap` | 0 | Empty |
| `chile_cmx` / `chi_super_dl_cs_ecommerce` | — | See above |

---

## Quick Reference: Trino External Table Setup

To query `FACT_IN_STOCK` for any date:
```sql
CREATE TABLE hive.s3_access_test.fact_in_stock_<YYYYMMDD> (
  item_id varchar, item_desc varchar,
  item_class_cd varchar, item_class_name varchar,
  item_subclass_cd varchar, item_subclass_name varchar,
  location_id varchar, location_name varchar, org_name varchar,
  stock varchar, stock_q varchar,
  brand_cd varchar, brand_name varchar,
  enabled_ind varchar, channel_cd varchar, dispo_minex varchar,
  item_replenishment_type_cd varchar, min_safety_stock_qty varchar,
  operational_division_cd varchar, section_cd varchar,
  top_venta varchar, vendor_party_id varchar,
  calendar_year_id varchar, calendar_week_id varchar, day_of_week_num varchar
)
WITH (
  external_location = 's3://cencosud.prod.sm.cl.raw/CHI_SUPER_DIM_TB/FACT_IN_STOCK/calendar_dt=<YYYY-MM-DD>/',
  format = 'PARQUET'
);
```

To query `FACT_IOSA_NRT` (current file):
```sql
CREATE TABLE IF NOT EXISTS hive.s3_access_test.fact_iosa_nrt (
  item_id varchar, location_id varchar, stock_qty varchar,
  iosa varchar, instock varchar, calendar_dt varchar,
  hora varchar, cplanif varchar, vendor_party_id varchar
)
WITH (
  external_location = 's3://cencosud.prod.sm.cl.raw/CHI_SUPER_NRT_TB/FACT_IOSA_NRT/',
  format = 'PARQUET'
);
```

---

## Open Questions / Things to Request

### ✅ Resolved (2026-05-18 / 2026-05-19)
- ~~**`VW_FACT_DAILY_INVENTORY_NRT` KMS fix**~~ → ✅ Fixed by Gustavo Tejeda 2026-05-18. Now fully accessible.
- ~~**`STOCK_ADJUSTMENT` KMS fix**~~ → ✅ Fixed 2026-05-12. KMS key `arn:aws:kms:us-east-1:265763412542:key/9e7d4c72-b9df-46cb-b62b-89843c63cbf6` added to role policy.
- ~~**FACT_IN_STOCK Trino registration**~~ → ✅ Partially done 2026-05-19. Table recreated with correct `partitioned_by = ARRAY['calendar_dt']` DDL. Blocked on HMS partition population — see item 3 below.

### 🔲 Still Outstanding
1. ~~**`VW_FACT_DAILY_INVENTORY_NRT` Hive schema fix**~~ → ✅ Fixed 2026-05-19. Recreated with `calendar_dt date`. Fully queryable — 58,279,814 rows, no type crash. Note: `calendar_dt` is NULL across all rows (single-snapshot export, date not embedded in data).
2. **`sales_transaction_line` live access**: Grant GetObject on `cencosud.prod.sm.cl.ldm/chi_super/chi_super_rel_tb/` — NRT POS data with SAP material numbers. Ask Gustavo Tejeda.
3. **FACT_IN_STOCK partition population** ⚠️: Table DDL is correct (`partitioned_by = ARRAY['calendar_dt']`), but HMS cannot enumerate S3 to auto-register partitions (`sync_partition_metadata` runs without error but finds 0 entries). `register_partition` procedure is disabled in Trino config. Queries fail with "Failed fetching partitions". Two paths to fix:
   - **Option A (fast):** Ask Gustavo Tejeda to enable `hive.allow-register-partition-procedure=true` in Trino config — then we can populate from here.
   - **Option B (durable):** Grant HMS service account `s3:ListBucket` on `cencosud.prod.sm.cl.raw/CHI_SUPER_DIM_TB/FACT_IN_STOCK/*` — `sync_partition_metadata` then works permanently.
   - **Workaround:** Per-day tables (`fact_in_stock_YYYYMMDD`) pointing at `calendar_dt=YYYY-MM-DD/` subfolder, no `partitioned_by`. Still works. See Quick Reference below.
4. **`io_items` SORTKEY or materialized view**: Pre-aggregated fill rate view in Cencommerce Redshift to bypass WLM limits.
5. **`machine_learning` schema access**: 3 tables (`nrt_clientes`, `nrt_productos`, `vista_pos_ml_chile_nrt`) are registered — confirm queryability and document schema.
6. **`nps` schema exploration**: 7 NPS tables — explore schema and document. Potentially useful for correlating in-stock issues with customer satisfaction.
7. **Understand VW_FACT_DAILY_INVENTORY_NRT data scope**: Why does `location_id = A047`? Is this multi-banner (Jumbo + Santa Isabel + Easy)? Clarify with Gustavo Tejeda which stores/banners are included.
