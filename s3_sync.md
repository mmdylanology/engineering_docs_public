# S3 → Aurora Inventory Sync — Technical Design
## Hourly VW_DAILY_NRT Sync for GIV Integration

> **Status:** Design finalised — pending Step 1 infra decisions  
> **Date:** 2026-05-31  
> **Scope:** POC → Production path for syncing SAP inventory snapshots into Aurora for GIV supply-demand service  
> **Analysis window:** 13 snapshots, May 21–31 2026 (10-day window)

---

## 0. How It Works — Plain English

This section is the full flow in plain language. Sections 1–10 below contain the technical detail, code, and evidence.

### What happens every hour

**1. SAP exports 36 files to S3.**
Every hour, SAP HANA writes the entire Cencosud Chile inventory state — all 606 stores, all 658K SKUs — into 36 Parquet files. That is 58 million rows per snapshot. The files land in Cencosud's S3 bucket. We replicate them to our POC bucket automatically via S3 replication.

**2. Lambda counts the files.**
Every file arrival fires an S3 event. A Lambda function tracks how many files have arrived for each hourly run. When all 36 are confirmed, Lambda triggers the Glue job exactly once. No partial loads, no race conditions.

**3. Glue filters 58M rows down to ~3–4M.**
The raw 58M rows are mostly noise. Glue applies three filters:
- **Drop ~590K dead SKUs** (~50.7M rows) — SKUs that have zero stock at every store in the entire network. Validated against 3.5 years of history: 99.996% were never stocked. These carry zero signal for GIV.
- **Drop zero rows for active SKUs** (~3.9M rows) — a SKU with stock at store A and zero at store B; the zero row at store B is irrelevant. Absence from our database means the same thing.
- **Keep negatives** — SAP negative quantities are NOT zero stock. They mean the SAP ledger has more outflows than inflows (a data integrity state). 32,820 negative rows recovered to positive stock in our 10-day analysis. If we drop them, we miss those "back in stock" events permanently.

After filtering: ~3–4M rows go into the Aurora `staging` table.

**4. sync_inventory() diffs staging against the last known state.**
A stored procedure compares staging (what SAP says NOW) against `supply_summary` (what we recorded LAST HOUR). Three outcomes per row:
- **New row** (in staging, not in supply_summary) → stock appeared. Emit event.
- **Changed row** (in both, qty differs) → stock moved. Emit event. This catches positive→negative AND negative→positive (SAP correction / goods receipt posted).
- **Gone row** (in supply_summary, not in staging) → stock went to zero or SKU became dead. Emit event.

The ~95% of rows that didn't change produce no events and no writes. `supply_summary` is updated. `staging` is truncated.

**5. Delta Publisher clamps and publishes.**
A polling service reads pending events from `delta_outbox` and publishes them to SQS. Before publishing:
- Raw SAP quantities are clamped: `published_qty = MAX(qty, 0)`. GIV never receives a negative number.
- Events are typed: `BACK_IN_STOCK` (old ≤ 0, new > 0), `OUT_OF_STOCK` (old > 0, new ≤ 0), `QTY_CHANGED` (both positive, level changed).

**6. GIV consumes events from SQS.**
The supply-demand service receives only what changed. `new_quantity > 0` → mark IN STOCK. `new_quantity = 0` → mark OOS (record kept, not deleted). GIV always has an accurate inventory state, updated every hour, without ever processing the 54M rows that never change.

---

### Volume at a glance

| Stage | Rows | Notes |
|-------|------|-------|
| SAP snapshot (raw) | 58,286,222 | Full network dump every hour |
| After dead SKU filter | ~7.6M | ~590K SKUs removed |
| After zero row filter | ~3–4M | Location gaps removed |
| Into staging | ~3–4M | Non-zero active pairs only |
| Deltas generated | ~2–3M peak | During store hours (9AM–10PM CLT) |
| Deltas overnight | Near-zero | Stores closed, SAP unchanged |
| supply_summary size | ~3–4M rows | Live inventory state |

### Naive vs this design

| Approach | Daily writes | Notes |
|----------|-------------|-------|
| Naive (write everything) | ~1.4 billion | 58M × 24 = 140 crore |
| This design (peak) | ~48–72 million | 2–3M × 24 |
| This design (average) | ~20–30 million | Overnight near-zero pulls average down |
| **Reduction** | **~95–98%** | |

---

## 1. Problem Statement

Every hour, SAP HANA exports a full snapshot of Cencosud Chile's inventory network into 36 Parquet files in S3. Each snapshot contains ~58 million rows — every SKU at every store in the network, regardless of whether anything changed.

**The problem:** Naively writing all 58M rows to a database every hour would generate ~140 crore total writes per day (58M × 24 snapshots). Based on analysis of 13 snapshots (May 21–31, 2026):

- **95.17% of rows never change** across any snapshot
- Only **~4.83% of rows** carry any new inventory signal per hour
- Writing all 58M rows is 95% wasted work

**The goal:** Detect only genuinely changed rows and publish those as events to the supply-demand service — so GIV always has an accurate, up-to-date inventory state without hammering the database.

---

## 2. Key Design Decisions

### 2.1 Absence = Zero Stock
The supply-demand service treats a missing row in `supply_summary` as zero stock. This means:
- **Zero-quantity rows (qty = 0)** from SAP never need to enter the system — absence is equivalent
- **Negative-quantity rows are kept** — see Section 2.3 for why
- When a row goes to zero, the sync publishes a `new_quantity = 0` event and removes it from `supply_summary`

This keeps `supply_summary` small (~3–4M rows of non-zero active stock) instead of 58M.

### 2.2 OOS vs Delete
When `new_quantity = 0` is received by the supply-demand service:
- The record is **marked OOS** — it is NOT deleted
- History is preserved
- If the same SKU+store is restocked next hour, a `new_quantity > 0` event flips it back to in-stock

### 2.3 Negative Quantities — Keep Raw, Clamp at Publish

**This is the most important design decision and was validated by data.**

SAP negative quantities do NOT mean "no stock". A negative `Ending_On_Hand_Qty` means the SAP ledger recorded more outflows than inflows for that SKU+store — a data integrity state, not a zero-stock flag. The store may have physical stock while SAP shows -50,000.

**What the data showed (10-day analysis, 206,934 ever-negative SKU+store pairs):**

| Category | Count | % | Meaning |
|----------|-------|---|---------|
| Frozen negative (same value all 13 snapshots) | 84,705 | 40.9% | Chronic SAP ledger corruption — no correction happening |
| Dynamic, still negative at May31 | 56,093 | 27.1% | Partially correcting but not resolved yet |
| Dynamic, resolved to 0 | 33,316 | 16.1% | SAP zeroed it out |
| **Dynamic, recovered to POSITIVE** | **32,820** | **15.9%** | **SAP goods receipt posted — real stock confirmed** |

**The 32,820 recoveries prove why you cannot filter negatives at ingestion.** Real examples:

| SKU | Store | Min seen | Negative snapshots | S14 qty | What happened |
|-----|-------|----------|-------------------|---------|---------------|
| `000000000001817599` | J414 | -1,536 | 2 of 13 | **13,009** | Goods receipt posted, full restock |
| `000000000001960847` | J403 | -5 | 1 of 13 | **9,379** | Single-snapshot glitch, self-corrected |
| `000000000001819203` | J514 | -919 | 4 of 13 | **4,575** | Partial receipt postings, eventually balanced |
| `000000000002017687` | N845 | -63 | 10 of 13 | **4,410** | Negative most of the window, large receipt at end |

If these rows had been filtered at ingestion, GIV would have missed 32,820 "back in stock" signals during this window alone.

**Examples of chronic negative positions that will NOT self-correct:**

| SKU | Store | May21 | May31 | Trend | Action needed |
|-----|-------|-------|-------|-------|--------------|
| `000000000000692822` | J411 | -234,024 | -242,053 | Getting worse ~800/day | SAP ops team — goods receipt misconfigured |
| `000000000000692822` | J403 | -108,339 | -112,223 | Same pattern, 8 Jumbo stores | Same root cause |
| `000000000000001502` | N818 | -91,850 | -94,250 | Step-function drops (batch job) | SAP ops team — batch movement mismatch |
| `000000000000001502` | N669 | -85,418 | -88,678 | Same, 7 SISA stores | Same root cause |

These are SAP process failures. Not in IOSA (no physical audit trail). Not in stock adjustments. Getting worse daily. Flag separately.

**Design rule:**
```
storage layer:  supply_summary.quantity = raw SAP value (can be negative)
publish layer:  published_qty = MAX(quantity, 0)   ← never send negative downstream
event trigger:  old_qty <= 0 AND new_qty > 0  →  BACK_IN_STOCK event
```

### 2.4 Glue for Parquet Conversion
Aurora cannot natively load Parquet files. AWS Glue reads Parquet from S3, applies filtering rules, and loads only the relevant rows into Aurora staging. Glue stays dumb — it only applies structural filters. All business logic lives in the SQL stored procedure.

### 2.5 36-File Coordination via Lambda
Each file arrives in S3 independently and triggers its own S3 event. A Lambda function counts arrivals per `run_id` (embedded in the filename). When all 36 files are confirmed, it fires the Glue job. No file is missed, no partial loads happen.

### 2.6 All Business Logic in One Stored Procedure
The `sync_inventory()` stored procedure in Aurora owns all diff, delta, and state-update logic. This means:
- Rules can be added/changed in one place
- Nothing is spread across Lambda, Glue, and SQL
- Easy to test and debug independently

---

## 3. Architecture Overview

```
SAP HANA (source)
    │
    ▼
Cencosud S3 bucket (36 Parquet files per hour, ~540MB)
    │  S3 Replication (file by file, automated)
    ▼
POC S3 bucket (controlled bucket — source of truth for POC)
    │  S3 event per file → SQS → Lambda
    ▼
Lambda — file coordinator
    │  counts files per run_id
    │  when 36 confirmed → triggers Glue
    ▼
AWS Glue job — pre-processing layer
    │  reads 36 Parquet files
    │  filters: drop dead SKUs (zero everywhere in network, ~590K SKUs)
    │  keeps: positive, negative, and zero rows for active SKUs
    │  loads surviving rows into Aurora staging
    │  calls CALL sync_inventory(run_id)
    ▼
Aurora — sync engine
    │  staging table (current hour's rows, including negatives)
    │  sync_inventory() procedure (diff → delta → state update)
    │  supply_summary table (raw qty, including negatives)
    │  delta_outbox table (pending events, stores raw qty)
    │  sync_jobs table (monitoring)
    ▼
Delta Publisher service
    │  polls delta_outbox for pending rows
    │  publishes MAX(new_quantity, 0) — never sends negative downstream
    │  fires BACK_IN_STOCK event when old_qty <= 0 AND new_qty > 0
    │  marks rows as published
    ▼
SQS (delta-events queue)
    │
    ▼
Supply-Demand Service (GIV consumer)
    │  new_quantity > 0 → update stock level (IN_STOCK or BACK_IN_STOCK)
    │  new_quantity = 0 → mark OOS
```

---

## 4. Infrastructure Setup (One-Time)

These steps are done once before the service runs for the first time.

| Component | What to create |
|-----------|---------------|
| POC S3 bucket | Controlled bucket to receive replicated files |
| S3 Replication rule | Source: Cencosud bucket → Destination: POC bucket |
| Aurora instance | MySQL or PostgreSQL — hosts all sync tables |
| Aurora tables | `staging`, `supply_summary`, `delta_outbox`, `sync_jobs`, `file_manifest` |
| Aurora stored procedure | `sync_inventory()` |
| Lambda function | S3 event handler — file coordinator |
| Glue job | Parquet reader + filter + staging loader |
| SQS queues | `file-arrival` (internal) + `delta-events` (for supply-demand) |
| Delta publisher | Polling service — reads delta_outbox, publishes to SQS |
| IAM roles | Lambda read POC bucket + write Aurora; Glue read POC bucket + write Aurora |

---

## 5. Database Schema

### 5.1 staging
Holds the current hour's filtered rows. Includes negative quantities (raw from SAP). Truncated after every run.

```sql
CREATE TABLE staging (
    item_id     VARCHAR(18)   NOT NULL,
    location_id VARCHAR(10)   NOT NULL,
    -- Raw SAP value. Can be negative (SAP ledger imbalance).
    -- Do NOT coerce here — supply_summary stores the raw value.
    quantity    DECIMAL(18,4) NOT NULL,
    run_id      BIGINT        NOT NULL,
    INDEX idx_stg_item_loc (item_id, location_id)
);
```

### 5.2 supply_summary
The live inventory state. Stores raw SAP quantity including negatives.
Absence from this table = zero stock (truly dead SKU, never active).
Negative quantity = SAP ledger imbalance — published as 0 downstream.

```sql
CREATE TABLE supply_summary (
    item_id     VARCHAR(18)   NOT NULL,
    location_id VARCHAR(10)   NOT NULL,
    -- Raw SAP quantity. Can be negative.
    -- Published downstream as MAX(quantity, 0) by the delta publisher.
    quantity    DECIMAL(18,4) NOT NULL,
    updated_at  DATETIME      NOT NULL,
    PRIMARY KEY (item_id, location_id)
);
```

### 5.3 delta_outbox
Outbox table. Every change detected by `sync_inventory()` lands here first.
Stores raw quantities. The delta publisher applies MAX(qty,0) clamping on the way out.

```sql
CREATE TABLE delta_outbox (
    id               BIGINT AUTO_INCREMENT PRIMARY KEY,
    item_id          VARCHAR(18)   NOT NULL,
    location_id      VARCHAR(10)   NOT NULL,
    -- Raw values from SAP — may be negative.
    old_quantity     DECIMAL(18,4) NOT NULL,
    new_quantity     DECIMAL(18,4) NOT NULL,
    run_id           BIGINT        NOT NULL,
    event_created_at DATETIME      NOT NULL,
    status           ENUM('pending', 'published') NOT NULL DEFAULT 'pending',
    INDEX idx_status      (status),
    INDEX idx_item_loc    (item_id, location_id),
    INDEX idx_run_id      (run_id)
);
```

### 5.4 sync_jobs
One row per hourly run. Tracks every phase for monitoring and debugging.

```sql
CREATE TABLE sync_jobs (
    id               BIGINT AUTO_INCREMENT PRIMARY KEY,
    run_id           BIGINT NOT NULL UNIQUE,
    status           ENUM('collecting','loading','diffing',
                          'publishing','completed','failed') NOT NULL,
    files_received   INT          NOT NULL DEFAULT 0,
    files_expected   INT          NOT NULL DEFAULT 36,
    rows_loaded      INT,
    deltas_generated INT,
    events_published INT,
    started_at       DATETIME,
    completed_at     DATETIME,
    error_message    TEXT,
    INDEX idx_run_id (run_id),
    INDEX idx_status (status)
);
```

### 5.5 file_manifest
Tracks individual file arrivals per run.

```sql
CREATE TABLE file_manifest (
    id         BIGINT AUTO_INCREMENT PRIMARY KEY,
    run_id     BIGINT       NOT NULL,
    file_name  VARCHAR(255) NOT NULL,
    arrived_at DATETIME     NOT NULL,
    INDEX idx_run_id (run_id)
);
```

---

## 6. Step-by-Step Flow

### Step 1 — Files Arrive in POC Bucket

SAP HANA exports 36 Parquet files to the Cencosud S3 bucket every hour. S3 Replication copies each file to the POC bucket as it arrives. Files do **not** arrive simultaneously — replication happens file by file over 2–5 minutes.

Each file is named:
```
run-{unix_ms}-part-block-0-r-{00000–00035}-snappy.parquet
```

The `run_id` (unix millisecond timestamp) is the same for all 36 files in a run. This is how the coordinator knows which files belong together.

---

### Step 2 — Lambda: File Coordination

Every file arrival in the POC bucket triggers an S3 event → SQS → Lambda.

```python
def handle_s3_event(event, context):
    key      = event['Records'][0]['s3']['object']['key']
    filename = key.split('/')[-1]
    run_id   = int(filename.split('-')[1])  # extract unix_ms run_id

    # Record file arrival
    cursor.execute(
        "INSERT INTO file_manifest (run_id, file_name, arrived_at) VALUES (%s, %s, NOW())",
        (run_id, filename)
    )

    # Create or increment sync_job for this run
    cursor.execute("""
        INSERT INTO sync_jobs (run_id, status, files_received)
        VALUES (%s, 'collecting', 1)
        ON DUPLICATE KEY UPDATE
            files_received = files_received + 1,
            status = IF(files_received + 1 >= 36, 'loading', 'collecting')
    """, (run_id,))

    # Check if all 36 files have arrived
    row = cursor.execute(
        "SELECT files_received FROM sync_jobs WHERE run_id = %s", (run_id,)
    ).fetchone()

    if row['files_received'] >= 36:
        trigger_glue_job(run_id)
```

The Glue job is only triggered **once**, when the 36th file arrives. No partial loads, no race conditions.

---

### Step 3 — Glue Job: Parquet → Staging

The Glue job reads all 36 files for the run, applies structural filters, and loads surviving rows into the `staging` table.

**What Glue filters and why:**

| Rule | What it does | Rows removed | Rationale |
|------|-------------|-------------|-----------|
| Cast quantity | `Ending_On_Hand_Qty → DECIMAL(18,4)` | 0 | Type normalisation |
| Drop dead SKUs | SKUs where `MAX(qty) = 0` at every store in network | ~50.7M rows | 590K SKUs confirmed never stocked via 3.5yr FACT_IN_STOCK check (script 29). Zero signal. |
| Drop zero rows | `qty = 0` for any remaining row | ~3.9M rows | Active SKUs with zero at this store (location gaps) carry no signal — absence in supply_summary means the same thing. |
| Drop service SKUs | `Item_Id LIKE '000000010%'` | handful | SAP service entries, not physical products |
| Drop A-prefix stores | `Location_Id LIKE 'A%'` | ~100K | Unknown chain — exclude until confirmed |
| Deduplicate | Safety net | ~0 in practice | Edge case guard |

**What Glue does NOT filter:**
- **Negative quantities** — kept as-is. 32,820 negative rows recovered to positive in 10 days (see Section 2.3). Filtering negatives at this layer would silently drop all "back in stock" recovery signals.

After all filters: staging contains **~3–4M rows** (non-zero active SKU+store pairs).

```python
df = spark.read.parquet(f"s3://poc-bucket/run-{run_id}-*.parquet")

# Cast quantity — keep raw value including negatives
# DO NOT coerce negatives to 0 here. Negatives are SAP ledger states.
# They are stored raw in supply_summary and clamped at publish time.
# Reason: 32,820 negative rows recovered to positive in our 10-day analysis.
# Filtering them would silently drop all "back in stock" recovery events.
df = df.withColumn(
    "quantity",
    col("Ending_On_Hand_Qty").cast("decimal(18,4)")
)

# Drop truly dead SKUs: zero at every store network-wide.
# Computed fresh each run — if a dead SKU gets restocked, it re-enters automatically.
# ~590K SKUs confirmed never stocked via 3.5yr FACT_IN_STOCK history (script 29).
# This removes ~50.7M rows (590K SKUs × ~86 avg stores each).
sku_max = df.groupBy("Item_Id").agg(max(col("quantity")).alias("max_qty"))
dead_skus = sku_max.filter(col("max_qty") <= 0).select("Item_Id")
df = df.join(dead_skus, on="Item_Id", how="left_anti")

# Drop zero rows — active SKU at zero stock at this location.
# Absence from supply_summary means the same thing as qty=0.
# Zero rows for active SKUs ("location gaps") would add ~3.9M rows of no-signal noise.
# CASE 3 in sync_inventory() correctly handles the positive→zero transition
# by detecting rows present in supply_summary but absent from staging.
df = df.filter(col("quantity") != 0)  # keep positives AND negatives, drop zeros

# Drop non-physical SKUs (SAP services / bundles, Item_Id prefix 000000010...)
df = df.filter(~col("Item_Id").startswith("000000010"))

# Drop unconfirmed chain (A-prefix stores) — remove this filter when chain confirmed
df = df.filter(~col("Location_Id").startswith("A"))

# Safety dedup — one row per SKU+store per run
df = df.dropDuplicates(["Item_Id", "Location_Id"])

# Select only what staging needs — raw qty preserved
df = df.select(
    col("Item_Id").alias("item_id"),
    col("Location_Id").alias("location_id"),
    col("quantity"),           # raw SAP value, may be negative
    lit(run_id).alias("run_id")
)

# Load into staging
df.write.jdbc(aurora_jdbc_url, "staging", mode="append", properties=jdbc_props)

# Update job status and kick off diff
cursor.execute(
    "UPDATE sync_jobs SET status='diffing', rows_loaded=%s WHERE run_id=%s",
    (df.count(), run_id)
)
cursor.execute("CALL sync_inventory(%s)", (run_id,))
```

---

### Step 4 — sync_inventory(): The Diff Engine

This stored procedure is the core of the system. It runs once per hour after staging is loaded.

**Four cases it handles:**

```
CASE 1 — New item
  Item is in staging but NOT in supply_summary
  → First time this SKU+store has appeared in the active set
  → old_quantity = 0, new_quantity = staging.quantity
  → Includes negative first appearances (SAP ledger already broken on arrival)

CASE 2 — Changed quantity
  Item is in both staging AND supply_summary, but quantities differ
  → Stock level moved: sale, replenishment, adjustment, or SAP correction
  → Covers all transitions: pos→pos, pos→neg, neg→neg, neg→pos
  → The neg→pos transition is the "back in stock" recovery case
  → old_quantity = supply_summary.quantity, new_quantity = staging.quantity

CASE 3 — Went to zero / disappeared
  Item is in supply_summary but NOT in staging
  → Glue filtered it out (fell into dead pool), or qty truly hit 0
  → old_quantity = supply_summary.quantity, new_quantity = 0
  → Remove from supply_summary

NOTE: No CASE 4 for "still negative, same value" — that is a CASE 2 with
no change, so it produces no delta. Frozen negative rows (84,705 confirmed)
do not generate events unless they change. They sit in supply_summary at
their negative raw value and publish as 0 downstream.
```

```sql
DELIMITER //
CREATE PROCEDURE sync_inventory(IN p_run_id BIGINT)
BEGIN

    -- ── STEP 1: New items + changed quantities (CASE 1 + CASE 2) ─────────────
    -- Catches all transitions: 0→pos, 0→neg, pos→pos, pos→neg, neg→pos, neg→neg
    -- The neg→pos case (old<=0, new>0) is the "back in stock" recovery signal.
    -- Delta publisher handles the clamping and event type labelling downstream.
    INSERT INTO delta_outbox
        (item_id, location_id, old_quantity, new_quantity,
         run_id, event_created_at, status)
    SELECT
        s.item_id,
        s.location_id,
        COALESCE(ss.quantity, 0),   -- 0 if new item; previous raw qty if changed
        s.quantity,                  -- new raw qty from SAP (may be negative)
        p_run_id,
        NOW(),
        'pending'
    FROM staging s
    LEFT JOIN supply_summary ss
           ON s.item_id = ss.item_id
          AND s.location_id = ss.location_id
    WHERE ss.item_id IS NULL          -- CASE 1: new item not seen before
       OR ss.quantity != s.quantity;  -- CASE 2: quantity changed (any direction)

    -- ── STEP 2: Items that went to zero / left active set (CASE 3) ────────────
    -- Covers SKU+store rows that are in supply_summary but missing from staging.
    -- This happens when: qty dropped to 0, SKU moved into dead pool, or
    -- the row was filtered by Glue for another reason.
    INSERT INTO delta_outbox
        (item_id, location_id, old_quantity, new_quantity,
         run_id, event_created_at, status)
    SELECT
        ss.item_id,
        ss.location_id,
        ss.quantity,   -- what supply_summary had (raw, may be negative)
        0,             -- now gone from active set → treat as zero
        p_run_id,
        NOW(),
        'pending'
    FROM supply_summary ss
    WHERE NOT EXISTS (
        SELECT 1 FROM staging s
        WHERE s.item_id = ss.item_id
          AND s.location_id = ss.location_id
    );

    -- ── STEP 3: Update supply_summary to reflect new state ────────────────────
    -- Upsert all rows from staging — raw qty including negatives
    INSERT INTO supply_summary (item_id, location_id, quantity, updated_at)
    SELECT item_id, location_id, quantity, NOW()
    FROM staging
    ON DUPLICATE KEY UPDATE
        quantity   = VALUES(quantity),
        updated_at = NOW();

    -- Remove rows that are no longer in staging (went to zero or left active set)
    DELETE ss FROM supply_summary ss
    WHERE NOT EXISTS (
        SELECT 1 FROM staging s
        WHERE s.item_id = ss.item_id
          AND s.location_id = ss.location_id
    );

    -- ── STEP 4: Update job record ─────────────────────────────────────────────
    UPDATE sync_jobs
    SET status           = 'publishing',
        rows_loaded      = (SELECT COUNT(*) FROM staging),
        deltas_generated = (SELECT COUNT(*) FROM delta_outbox
                            WHERE run_id = p_run_id)
    WHERE run_id = p_run_id;

    -- ── STEP 5: Clean staging for next run ────────────────────────────────────
    TRUNCATE TABLE staging;

END //
DELIMITER ;
```

---

### Step 5 — Delta Publisher

A lightweight polling service that drains `delta_outbox` to SQS.

**This is where raw SAP quantities are clamped before going downstream.**
GIV must never receive a negative availability signal.

```python
def classify_event(old_qty, new_qty):
    """
    Classify the event type for the supply-demand service.

    Raw SAP quantities are clamped here — supply_summary stores negatives,
    but GIV only ever sees MAX(qty, 0).

    Real data example of BACK_IN_STOCK recovery:
      SKU 000000000001817599, store J414:
        - Was -1,536 units for 2 snapshots (SAP ledger imbalance)
        - Goods receipt posted, jumped to +13,009
        - old_qty = -1,536, new_qty = 13,009 → BACK_IN_STOCK event
        - GIV sees: old=0, new=13,009 (negative clamped to 0)
    """
    # Clamp: never publish negative values downstream
    published_old = max(old_qty, 0)
    published_new = max(new_qty, 0)

    if old_qty <= 0 and new_qty > 0:
        # Covers both: new item appearing for first time (old=0, CASE 1)
        # and recovery from negative back to positive (CASE 2 neg→pos).
        # Both are semantically "stock is now available" from GIV's perspective.
        event_type = 'BACK_IN_STOCK'
    elif old_qty > 0 and new_qty <= 0:
        event_type = 'OUT_OF_STOCK'    # was positive, now zero or negative
    else:
        event_type = 'QTY_CHANGED'     # both positive, level changed

    return event_type, published_old, published_new


def publish_loop():
    while True:
        rows = cursor.execute("""
            SELECT id, item_id, location_id,
                   old_quantity, new_quantity, event_created_at, run_id
            FROM delta_outbox
            WHERE status = 'pending'
            ORDER BY id
            LIMIT 500
        """).fetchall()

        if not rows:
            time.sleep(30)
            continue

        messages = []
        for row in rows:
            event_type, pub_old, pub_new = classify_event(
                float(row['old_quantity']),
                float(row['new_quantity'])
            )
            messages.append({
                'Id': str(row['id']),
                'MessageBody': json.dumps({
                    'item_id':          row['item_id'],
                    'location_id':      row['location_id'],
                    'event_type':       event_type,
                    'old_quantity':     pub_old,   # clamped: MAX(raw, 0)
                    'new_quantity':     pub_new,   # clamped: MAX(raw, 0)
                    'event_created_at': row['event_created_at'].isoformat(),
                    'run_id':           row['run_id']
                })
            })

        sqs.send_message_batch(QueueUrl=DELTA_QUEUE_URL, Entries=messages)

        ids = [row['id'] for row in rows]
        cursor.execute(
            f"UPDATE delta_outbox SET status='published' WHERE id IN ({','.join(['%s']*len(ids))})",
            ids
        )

        run_ids = set(row['run_id'] for row in rows)
        for run_id in run_ids:
            pending = cursor.execute(
                "SELECT COUNT(*) FROM delta_outbox WHERE run_id=%s AND status='pending'",
                (run_id,)
            ).fetchone()[0]
            if pending == 0:
                cursor.execute("""
                    UPDATE sync_jobs
                    SET status='completed',
                        events_published=(SELECT COUNT(*) FROM delta_outbox WHERE run_id=%s),
                        completed_at=NOW()
                    WHERE run_id=%s
                """, (run_id, run_id))
```

**SQS message format (clamped values, never negative):**
```json
{
  "item_id":          "000000000001817599",
  "location_id":      "J414",
  "event_type":       "BACK_IN_STOCK",
  "old_quantity":     0,
  "new_quantity":     13009,
  "event_created_at": "2026-05-31T13:13:00",
  "run_id":           1780227801570
}
```

```json
{
  "item_id":          "000000000000462386",
  "location_id":      "J754",
  "event_type":       "QTY_CHANGED",
  "old_quantity":     25,
  "new_quantity":     18,
  "event_created_at": "2026-05-31T13:13:00",
  "run_id":           1780227801570
}
```

---

### Step 6 — Supply-Demand Service Consumes

The supply-demand service receives events from SQS and updates its own state.

**Event handling contract:**

| `event_type` | `new_quantity` | Action |
|-------------|----------------|--------|
| `IN_STOCK` | `> 0` | First stock appearance — create record, mark IN STOCK |
| `BACK_IN_STOCK` | `> 0` | Recovery from zero/negative — flip status to IN STOCK |
| `QTY_CHANGED` | `> 0` | Update stock level |
| `OUT_OF_STOCK` | `= 0` | Mark OOS — **do not delete**, keep record |

```python
def handle_event(message):
    event = json.loads(message['Body'])

    # new_quantity is always >= 0 (clamped by publisher)
    # old_quantity is always >= 0 (clamped by publisher)
    # Raw SAP values never reach this service.

    if event['new_quantity'] > 0:
        # Stock appeared, changed, or recovered from negative
        db.upsert('supply_state', {
            'item_id':     event['item_id'],
            'location_id': event['location_id'],
            'quantity':    event['new_quantity'],
            'status':      'IN_STOCK',
            'updated_at':  event['event_created_at']
        })
    else:
        # Went to zero — mark OOS, preserve history
        db.update('supply_state',
            where={'item_id': event['item_id'], 'location_id': event['location_id']},
            set={'quantity': 0, 'status': 'OOS', 'updated_at': event['event_created_at']}
        )
```

---

## 7. First Run vs Subsequent Runs

### First Run (Bootstrap)

`supply_summary` is empty. `sync_inventory()` sees every row in staging as CASE 1 (new item).

The full 58M SAP snapshot passes through Glue. After removing ~50.7M dead rows (zero-everywhere SKUs) and ~3.9M zero rows for active SKUs, staging contains **~3–4M rows** — positives and negatives only.

```
Glue loads ~3–4M filtered rows into staging (non-zero active SKU+store pairs)
  ↓
sync_inventory():
  → supply_summary is empty
  → ALL staging rows hit CASE 1 (new item, old_quantity=0)
  → Positive rows → BACK_IN_STOCK event (old=0, new>0)
  → Negative rows → no delta emitted (old=0, new<0 → published as 0→0, no signal)
  → ALL staging rows inserted into supply_summary (raw qty including negatives)
  → staging truncated

Delta publisher:
  → publishes BACK_IN_STOCK events for all positive rows (clamped new_qty)
  → negative first-appearance rows publish as 0→0, effectively skipped
  → supply-demand service builds its initial state

After first run:
  supply_summary = full current inventory baseline (~3–4M rows)
```

### Subsequent Runs (Every Hour After)

```
Glue loads ~3–4M filtered rows into staging (non-zero active SKU+store pairs)
  ↓
sync_inventory():
  → CASE 1: new SKU+store combinations         → small (new products, new stores)
  → CASE 2: rows where quantity changed         → ~2–3M during store hours
  → CASE 3: rows that left the active set       → varies
  → ~95% of rows produce NO delta (unchanged)

Delta publisher:
  → clamps and publishes only the delta rows
  → supply-demand service receives only what changed
```

| | First Run | Subsequent Runs |
|--|-----------|----------------|
| `supply_summary` before | Empty | Full baseline |
| Rows into staging | ~3–4M (non-zero active) | ~3–4M (non-zero active) |
| Deltas generated | All ~3–4M rows | Only what changed (~2–3M peak) |
| Events published | Full baseline load | Incremental changes only |
| `supply_summary` after | ~3–4M rows | Updated in place |

---

## 8. Monitoring

Query `sync_jobs` at any time to see the state of every run:

```sql
-- Current run status
SELECT run_id, status, files_received, rows_loaded,
       deltas_generated, events_published,
       started_at, completed_at,
       TIMESTAMPDIFF(MINUTE, started_at, completed_at) AS duration_min
FROM sync_jobs
ORDER BY run_id DESC
LIMIT 10;

-- Stuck runs (collecting > 10 minutes — some files may have been missed)
SELECT * FROM sync_jobs
WHERE status = 'collecting'
  AND started_at < NOW() - INTERVAL 10 MINUTE;

-- Failed runs
SELECT * FROM sync_jobs WHERE status = 'failed';

-- Unpublished deltas older than 5 minutes
SELECT run_id, COUNT(*) AS stuck_events
FROM delta_outbox
WHERE status = 'pending'
  AND event_created_at < NOW() - INTERVAL 5 MINUTE
GROUP BY run_id;

-- Chronic negative positions (SAP process failures to flag to ops team)
-- These are rows with deeply negative qty that haven't changed in days.
-- Example: SKU 692822 at J411 has been -234K → -242K over 10 days.
SELECT item_id, location_id, quantity, updated_at
FROM supply_summary
WHERE quantity < -1000
ORDER BY quantity ASC
LIMIT 20;
```

---

## 9. Open Questions

| Question | Impact |
|----------|--------|
| Aurora MySQL or PostgreSQL? | Determines exact `LOAD DATA` syntax and JDBC driver |
| Glue DPU sizing | Depends on actual Parquet read + JDBC write throughput at ~7.5M rows post-filter |
| Delta publisher: dedicated service or Lambda? | Lambda simpler for POC; dedicated service better for production |
| SQS message batching size | Default 500 events/batch — confirm supply-demand service can handle |
| A-prefix store chain confirmed? | Remove the A-prefix filter once chain identity is confirmed |
| Dead SKU list refresh cadence | `dead_skus` table needs periodic refresh as catalog evolves — monthly? |
| SAP ops escalation for chronic negatives | SKUs 692822 (8 Jumbo stores) and 001502 (7 SISA stores) need SAP investigation |

---

## 10. What the Analysis Proved (Supporting Evidence)

All numbers from analysis of 13 VW_DAILY_NRT snapshots, May 21–31 2026 (10-day window).
Scripts: `scripts/28_inventory_state_full.py`, `scripts/29_dead_sku_historical_check.py`,
         `scripts/30_proof_never_changed.py`

### Snapshot structure

| Finding | Value | Implication |
|---------|-------|-------------|
| Total rows per snapshot | 58,286,222 | Service must handle this scale |
| Rows never changed (all 13 snapshots, 10 days) | 55,473,003 (95.17%) | 95% of writes are wasted without filtering |
| Frozen zeros | 54,594,442 (93.67%) | Dead SKUs — filtered at Glue |
| Frozen positive stock | 793,856 (1.36%) | Real stock, no movement for 10 days — 41.00B CLP |
| Frozen negatives | 84,705 (0.15%) | Chronic SAP corruption — stored raw, publish as 0 |
| Rows changed at least once | 2,813,219 (4.83%) | Upper bound on hourly delta volume |
| Proof methodology | Script 30 — two independent methods agree exactly | Number is verified, not estimated |

### Negative quantity analysis (validated reason to keep negatives)

| Finding | Value | Implication |
|---------|-------|-------------|
| Ever-negative SKU+store pairs (10 days) | 206,934 (0.355% of 58M) | Small set but critical to handle correctly |
| Frozen negative (stuck, same value all 13) | 84,705 (40.9%) | Chronic SAP failures — flag to SAP ops |
| Dynamic, still negative at May31 | 56,093 (27.1%) | In process of correcting |
| Dynamic, resolved to 0 | 33,316 (16.1%) | Corrected and zeroed out |
| **Dynamic, recovered to POSITIVE** | **32,820 (15.9%)** | **Would be missed if negatives filtered at ingestion** |

### Dead SKU validation

| Finding | Value | Implication |
|---------|-------|-------------|
| Dead SKUs (zero everywhere in network) | ~590,000 | Safe to filter — confirmed by FACT_IN_STOCK |
| Never stocked in 3.5 years of history | 590,199 (99.996%) | Filter is permanent and safe |
| Recently depleted (stocked in last 6 months) | 21 | Perishables — small, sync will catch restock |

### Chain breakdown (frozen stock, May21-31)

| Chain | Total Rows | Frozen Stock Rows | Value (CLP) | Change Rate |
|-------|------------|-------------------|-------------|-------------|
| Jumbo (J) | 22,419,389 | 461,962 | 27.85B | 5.67% |
| SISA (N) | 35,412,789 | 321,175 | 12.83B | 4.22% |
| Conveniencia (O) | 353,466 | 10,700 | 0.32B | 13.47% |
| Total | 58,286,222 | 793,856 | 41.00B | 4.83% |

> Conveniencia turns stock fastest (13.47% change rate) — smaller stores, faster SKU cycling.
> Jumbo holds the most frozen value per store — larger locations, higher-value inventory.
