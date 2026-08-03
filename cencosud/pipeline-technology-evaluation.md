# Technology Research Report — Jumbo Analytical Pipeline (Phase 1)

**Status:** Research only. No HLD/LLD/ADRs in this document.
**Date:** 2026-07-31
**Scope:** Ingestion & filtering engine · Storage tiering · Serving target · Orchestration · Semantic layer & catalog · Multi-cloud posture
**Explicitly out of scope:** VTEX, Publicador, all prior PoC artifacts

---

## 0. Confirmed Volume Profile — A Small-Data Problem in a Big-Data Costume

One hourly batch = 36 files, ~58M rows, **~500 MB total compressed size** across all 36 files.

| Measure | Value |
| :--- | :--- |
| **Bytes per row (compressed)** | ~8.6 B |
| **Average file size** | ~13.9 MB (~1.6M rows) |
| **Daily Volume** | 12 GB · 1.39B rows |
| **Annual Raw Volume (all chains)** | 4.4 TB · 508B rows |
| **Jumbo Slice @ 10% survival** | 1.2 GB/day · 139M rows/day · **438 GB/yr · ~51B rows/yr** |

### Critical Architectural Consequences

1. **Horizontal Scaling is Unnecessary for Ingestion**: 4.4 TB/year of raw data is a rounding error on S3. The entire multi-year historical backfill can be run sequentially on a single large EC2 instance (e.g. `c7g.8xlarge` sustained at ~2 GB/s read from S3), taking roughly 75 minutes of pure I/O and perhaps 4–6 hours end-to-end with transformations. This single fact eliminates Spark cluster architectures (EMR/Glue) from the steady-state ingestion path.
2. **Snapshot Semantics Hint**: The 8.6 bytes/row compression ratio in Parquet requires near-perfect dictionary + RLE encoding, typical of a sorted full-snapshot stock-position table (`store, sku, qty, status, ts`) with low column cardinality. If this is a full-snapshot feed where ~99% of what arrives each hour is unchanged state, the Silver representation should be an SCD2 / change-only table to reduce downstream row counts by 100×.

---

## 1. Workload Characterization

Stripped of vendor framing, this is the pipeline:
1. **Arrival**: 36 immutable Parquet objects land on S3, hourly, at a predictable cadence.
2. **Selection**: Keep records matching a chain predicate (`J*`), discard ~90%.
3. **Normalization**: Type coercion, deduplication, late-arrival handling, chain/store/SKU key resolution.
4. **Persistence**: Append to a conformed store, queryable by an analytical serving layer.
5. **Downstream (Future)**: Join against Redshift sales/orders, VTEX events, business rules, store master, SKU master, and pricing matrices behind a semantic layer serving LLM-mediated queries.

---

## 2. Ingestion & Filtering Engine

### 2.1 Multi-Engine Comparative Evaluation

| Engine | Execution Model | Startup / Cold Start | Operational Burden | S3 Pushdown Support | Monthly Cost |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **DuckDB** (Fargate/EC2) | Single-node, in-process, vectorized | Sub-second (< 1s) | **Low** | Natively reads S3 Parquet using HTTP range requests | **~$3.00** (4 vCPU/16GB Fargate) |
| **Polars / Daft** | Single-node (Daft scales out) | Sub-second | Low | Supported | ~$3.00 |
| **AWS Glue 5.0** | Serverless Spark | High (30 - 60s) | Low-Medium | Supported (Glue Catalog integration) | **~$35.00** (Min 2 DPU charge) |
| **EMR Serverless** | Serverless Spark | High (30 - 120s) | Medium | Supported | **~$30.00** |
| **AWS Athena** | Serverless Presto | Low (1 - 3s) | Very Low | Supported ($5/TB scanned billing) | **~$2.00** (12 GB scanned/day) |
| **GCP Dataflow / Dataproc** | Distributed Beam / Spark | Medium (60s+) | Medium-High | Incurs cross-cloud S3 egress costs | High (Egress tax per run) |

### 2.2 Cost vs. Operational Differentiators
At 500 MB/batch, compute costs across all engines are trivial (under $40/month). The decision must be made on engineering throughput, latency, and debuggability:

* **Startup Coordination Tax**: Glue and EMR Serverless spend more time starting up than working. Cold starts account for 50–100% of the job runtime. Single-node container tasks start immediately.
* **Local Reproducibility**: A DuckDB/Polars transformation runs identically on a developer laptop, in CI, and in production. Spark-on-Glue requires a cloud round-trip for every logic iteration.
* **Ceiling & Partition Fan-out**: If single-node memory becomes a bottleneck, the workload is naturally partitionable. We can process files in parallel chunks (36 independent S3 objects) or dynamically size Fargate tasks.

### 2.3 Recommendation
**Primary: DuckDB in a single ECS Fargate container, reading all 36 files as a single atomic task per hourly batch.**
* *Escape Hatches*: If the team's existing Spark fluency significantly exceeds its DuckDB capability, AWS Glue 5.0 is a stable second choice for ~$35/month extra.

---

## 3. Storage Architecture: Active Jumbo Data & Digital Decisions

### 3.1 Table Format: Apache Iceberg as the Standard
We select **Apache Iceberg** as the conformed storage format. 
* *Ecosystem Reach*: Iceberg is production-stable and natively integrated across Redshift, Athena, ClickHouse, Trino, Snowflake, and BigQuery. For a multi-tenant platform, format compatibility is a key requirement.

### 3.2 Catalog Selection
We recommend the **AWS Glue Data Catalog** configured as the Iceberg REST Catalog. It provides Lake Formation row/column-level access control, which is essential for multi-tenant isolation.

### 3.3 Storage Tiering & Update Mechanics

* **Bronze Tier (Raw Archive - S3 Landing)**: 
  * Managed by AWS S3 Replication. Stores 100% of the raw, hourly Parquet files (all chains, 58M rows/hour). It is kept purely as an immutable historical archive and is not cataloged or managed as database tables.
* **Silver Tier (Current State tables - S3 Iceberg)**:
  * Resides on S3 Standard. Instead of storing massive historical snapshots that bloat storage, the Silver layer holds the **current state** (updated hourly in-place):
    1. **`silver.physical_stock_active`**: Contains physical inventory for **all Jumbo SKUs** where quantity is not zero (`Ending_On_Hand_Qty != 0`), retaining positive and negative stock for supply and demand planning (~1.69M rows). Updated hourly via in-place `MERGE/DELETE` in the Fargate task.
    2. **`silver.publicador_state`**: Contains the direct e-commerce decisions from Cencosud's pipeline, ingested **as-is** from the hourly output CSV (`source.csv`) (~1.2M rows, 90MB). Updated hourly via a **complete atomic Overwrite/Replace** (reloading the entire 90MB file in under 3 seconds in Fargate to simplify change tracking and deletions).
* **Gold Tier (Analytical Serving - ClickHouse)**:
  * ClickHouse pulls from both Silver tables every hour to build a consolidated **Unified Analytical Truth** table (`gold.inventory_reconciliation`). It does not apply virtual stock overrides or safety buffers yet, serving purely as the raw analytical comparison target.

```
                  S3 Landing Zone (Bronze / Replicated Archive)
                  ┌─────────────────────────────────────────┐
                  │  raw/hana/      |   raw/publicador/     │
                  │  (58M rows, PK) |   (1.2M rows, CSV)    │
                  └─────────────────┬───────────────────────┘
                                    │
                       ┌────────────▼────────────┐
                       │   ECS Fargate Task      │
                       │   (Python + DuckDB)     │
                       └────┬────────────────┬───┘
                            │                │
     ┌──────────────────────▼──┐          ┌──▼──────────────────────┐
     │ silver.physical_stock   │          │ silver.publicador_state │
     │ - Jumbo & HANA != 0     │          │ - Ingest CSV as-is      │
     │ - Merge/Delete (1.69M)  │          │ - Overwrite/Replace(1.2M)│
     └──────────────┬──────────┘          └──────────┬──────────────┘
                    │                                │
                    └────────────────┬───────────────┘
                                     │ (LEFT JOIN + Metadata mapping)
                                     ▼
                          ┌─────────────────────────┐
                          │ Gold Tier (ClickHouse)  │
                          │ - Analytical Truth DB   │
                          │ - Compare HANA vs. Pub  │
                          └─────────────────────────┘
```

---

## 4. Serving Target: The Case Against Defaulting to Redshift

For an agentic platform where queries are generated by LLMs, query patterns are unpredictable and undetected.

### In-Depth Target Evaluation

* **ClickHouse (Recommended)**: Runs on **fixed-capacity compute** (e.g. reserved EC2 instances). Runaway queries compiled by LLMs cost latency, not money. It is purpose-built for fast time-series aggregation over high-cardinality `store * SKU * hour` dimensions.
* **Redshift Serverless**: Charges based on RPU-hours. Incurring RPU charges on a bursty, agent-driven query pattern creates an unbounded cost model. 
* **AWS Athena**: At $5/TB scanned, Athena is excellent for ad-hoc exploration, but dangerous as the production agent endpoint. A poorly compiled query could scan the entire database, leading to unpredictable billing.
* **Google BigQuery / Snowflake**: Excellent platforms, but introduce high cross-cloud egress and data transfer costs since the source files and Redshift sales data live on AWS.

### Serving Recommendation
**Use ClickHouse (1x `m7g.2xlarge` with HA replica) as the Gold serving tier.** 

ClickHouse operates as the **governed analytical source of truth**. Every hour, it runs a raw LEFT JOIN matching the e-commerce catalog (`silver.publicador_state`) to the physical store stock (`silver.physical_stock_active`), resolving names and VTEX SKU IDs using metadata mapping tables. 

It does **not** apply virtual override or inheritance rules at this layer. It is designed to expose raw values (HANA qty, e-commerce qty, case, and action) side-by-side. Runaway queries generated by LLM agents cost CPU/latency rather than dollars due to fixed-capacity instance sizing. Redshift remains in the architecture purely as a source to federate against for sales/orders data (via scheduled micro-batch S3/Iceberg extracts or JDBC external table links).

---

## 5. Semantic Layer & Metadata Catalog

We split the "Semantic Catalog" into three distinct technical layers to avoid single-vendor coupling:

```
┌──────────────────────────────────────────────────────────────────────────┐
│  C. Semantic Layer (Cube Core / MetricFlow)                              │
│  - Maps business metrics, defines join paths, enforces RLS policies.      │
└────────────────────────────────────┬─────────────────────────────────────┘
                                     ▼
┌──────────────────────────────────────────────────────────────────────────┐
│  B. Governance Layer (SageMaker DataZone / Future Open Source)           │
│  - Business glossaries, data ownership, lineage tracking.                │
└────────────────────────────────────┬─────────────────────────────────────┘
                                     ▼
┌──────────────────────────────────────────────────────────────────────────┐
│  A. Technical Catalog (Glue Data Catalog + Lake Formation)               │
│  - Tracks database schemas, formats, partitions, and permissions.        │
└──────────────────────────────────────────────────────────────────────────┘
```

### 5.1 Comparative Semantic & Metadata Tooling

* **AWS Glue Data Catalog + Lake Formation (Technical Catalog)**: 
  * *Verdict*: **Must use on day one**. It maps schemas and partitions for Athena/Redshift Spectrum and handles cell-level tenant security.
* **GCP Dataplex (Governance & Discovery)**:
  * *Verdict*: Excellent catalog profiling, but locked into GCP. Defer this layer entirely until the catalog schema exceeds ~50 tables.
* **Colrows (AI Semantic Graph)**:
  * *Verdict*: **Evaluate only as a competitor / design reference**. Their compile-then-execute model (resolving joins and policies before running SQL) is a solid architectural pattern. However, relying on a Pune-based commercial startup in the same space creates unnecessary operational dependencies for a proprietary product like Insights.
* **Cube Core (Semantic Substrate)**:
  * *Verdict*: **Recommended to build the in-house semantic compiler on top of Cube Core (Apache 2.0)**. Cube handles SQL compilation, pre-aggregation caching, and row-level multi-tenant security out of the box, exposing clean MCP interfaces to the LLM agent.

---

## 6. Orchestration & Engine Control

* **Dagster (Recommended)**: Asset-centric orchestration is a perfect fit. Since 90% of the raw data is historical, the pipeline must support partition backfills. Running "re-run partitions 2025-01-01 through 2026-06-30 for asset `silver_jumbo_stock`" is a native, single-command operation in Dagster.
* **Airflow 3.x (Alternative)**: If the engineering team has deep existing Airflow fluency, use managed Airflow (MWAA) to reduce operational learning curves.

---

## 7. Multi-Cloud Verdict

**GCP loses on physics, not on merit.** 
Because Cencosud's source S3 bucket and Redshift databases reside in AWS, running GCP-based pipelines (Dataflow, BigQuery) would incur constant cross-cloud egress and transfer fees. 

The correct hedge against cloud lock-in is **format openness (Apache Iceberg)**, not multi-cloud deployment. By saving all processed data in S3 Iceberg format, the query and warehouse engines can be swapped later without moving the physical files.

---

## 8. Sizing & Year-One Cost Model

Based on a steady-state run in `us-east-1` (24 hourly runs/day):

| Component | Architecture Role | Monthly Cost |
| :--- | :--- | :--- |
| **Ingestion** | ECS Fargate (4 vCPU / 16 GB, ~1 min runtime) | ~$5.00 |
| **Bronze Storage** | 2.2 TB average accumulated (Intelligent-Tiering) | ~$50.00 |
| **Silver Storage** | 220 GB conformed Jumbo data | ~$5.00 |
| **Technical Catalog**| Glue Data Catalog + Lake Formation | ~$0.00 |
| **Serving Tier** | ClickHouse (1 node + HA replica) | ~$480.00 |
| **Orchestration** | Dagster (self-hosted on small ECS task) | ~$30.00 |
| **Infrastructure Overhead** | S3 API requests, compaction, logging | ~$10.00 |
| **TOTAL** | | **~$580.00 / month** |

*Note: Infrastructure costs represent a minor fraction of the total platform value. The architecture should be optimized for developer iteration speed, testability, and query correctness rather than squeezing infrastructure costs below this baseline.*

---

## 9. Known Limitations and Trade-offs

1. **Single-Node Fargate Execution Limit**: If the hourly batch volume grows by more than 50×, single-node containers will hit memory ceilings. The mitigation path is fanning out the Fargate tasks to process the 36 Parquet files in parallel chunks.
2. **ClickHouse Operational Complexity**: Self-managing ClickHouse requires dedicated database expertise (merging, TTL policies, HA replication). If the estate stays single-tenant in Phase 1, a simpler DuckLake/DuckDB serving process is a viable alternative before scaling up.
3. **Compaction Latency in S3 Tables**: Managed compaction in S3 Tables can introduce a 2.5–3 hour lag. If immediate data freshness is a business requirement, compaction must be managed manually in the Silver/Gold pipelines.

---

## 10. Open Questions Blocking the HLD/LLD Phase

| # | Open Question | Design Blocker | Owner |
| :---: | :--- | :--- | :--- |
| **1** | Is the Jumbo `"J"` prefix in the S3 object key/path or inside the Parquet column values? | Determines if filtering is an S3 list operation (zero download cost) or a full data scan. | Andino |
| **2** | Are the hourly HANA files full snapshots or CDC/deltas? (Verify by running rows-per-store query). | Determines if the Silver layout is a snapshot table, an SCD2 table, or an event log. | Cencosud / Andino |
| **3** | What is the actual freshness SLA? Is hourly processing required, or is daily batch sufficient? | Determines the pipeline run frequency and partition sizes. | Andino / Cencosud |
| **4** | What is the multi-tenant isolation requirement? (Shared tables with row-level policies vs. separate storage). | Determines Lake Formation governance and semantic layer security configurations. | Andino Product |
| **5** | Are there data residency or PII restrictions on the HANA extracts? | Determines region selection (e.g. `us-east-1` vs. `sa-east-1` Chile). | Cencosud Legal |

---

## 11. Proposed ADR Docket

This append-only Architectural Decision Record (ADR) log maps out the key technical decisions that will permanently settle the system design:

| ADR | Decision Description | Rationale | Status |
| :---: | :--- | :--- | :--- |
| **001** | **Single-Cloud on AWS** | Source data gravity (S3) and destination data gravity (Redshift) are on AWS; avoids cross-cloud network egress fees. | Approved |
| **002** | **Apache Iceberg Format** | Open table format with native support across Athena, Redshift, ClickHouse, and Snowflake, avoiding lock-in. | Approved |
| **003** | **Single-Node DuckDB Ingestion** | Vectorized, vectorized C++ engine. Avoids distributed Spark JVM cold starts (EMR/Glue), processing 58M rows in seconds on Fargate. | Approved |
| **004** | **Glue Catalog as REST Catalog** | Direct integration with AWS IAM and Lake Formation for multi-tenant row-level access permissions. | Approved |
| **005** | **Current-State-Only Silver Tier** | Stores flat, current-state tables (`physical_stock_active` where qty != 0, and `publicador_state` from source.csv via full overwrite), avoiding S3 partition bloat. | Approved |
| **006** | **ClickHouse Gold Serving Layer** | Purpose-built OLAP engine. Exposes raw joined values side-by-side (HANA qty, pub qty, cases, cost) for analytical audit. | Approved |
| **007** | **Fixed-Capacity Compute Pricing** | Running queries on fixed EC2 VM capacity ensures cost determinism, protecting against LLM agent query volume spikes. | Approved |
| **008** | **Cube Core Semantic Layer** | Scaffold on Cube Core (Apache 2.0) to handle metric modeling, query caching, and MCP interface for agents. | Approved |
| **009** | **Dagster Orchestrator** | Asset-based orchestration. Optimizes partition backfills if rules or catalog mappings need to be reprocessed. | Approved |
| **010** | **Retain Raw Data in Landing S3** | Replicated S3 raw files are archived natively (Lifecycle to Glacier IR), preserving future onboarding of SISA/Easy. | Approved |

