# Cencosud EKS Cluster — Trino & S3 Quickstart

> Access inventory and stock data directly from S3 via Trino on the Cencosud `pro-evolution` EKS cluster.
> Last validated: **2026-05-12**.

---

## Prerequisites

```bash
# Install tools (macOS)
brew install awscli kubectl
```

You also need access to the `Cencosud-Infra/dpt-tas-magic-utils-cli-token` repo — this contains `k8s_v2.sh`, the team's credential refresh script. Ask Anusha or DevOps if you don't have it.

---

## Step 1 — One-time DNS fix (macOS only)

macOS's Go binaries (including `kubectl`) ignore `/etc/resolver/` and sometimes fail to resolve the EKS endpoint via VPN. Fix this permanently:

```bash
echo "172.23.208.215 2F8F0BD6A8899E4E42A47D67474C9522.gr7.us-east-1.eks.amazonaws.com" \
  | sudo tee -a /etc/hosts
```

Verify:
```bash
ping -c 1 2F8F0BD6A8899E4E42A47D67474C9522.gr7.us-east-1.eks.amazonaws.com
# Should resolve to 172.23.208.215 (packet loss is fine — VPN doesn't ICMP)
```

> You only need to do this **once**. It survives reboots.

---

## Step 2 — Daily auth (every ~8 hours)

```bash
cd ~/Cencosud-Infra/dpt-tas-magic-utils-cli-token
bash k8s_v2.sh -l
```

This opens a browser SSO login and updates your kubeconfig. When prompted:

```
######### Autorizar Acceso CLI #########
Entorno:
1) production   ← type 1 and press Enter
2) staging
3) engineering
4) Salir
```

After success your kubeconfig points to `pro-evolution`, namespace `composable-supply-chain`.

Verify:
```bash
kubectl get pods -n composable-supply-chain | grep trino
# Should show: trino-coordinator-... Running
#              trino-worker-...     Running (2 workers)
```

---

## Step 3 — Find the Trino coordinator pod

Pod names rotate on every deploy. Always look it up:

```bash
kubectl get pods -n composable-supply-chain \
  -l "app.kubernetes.io/name=trino,app.kubernetes.io/component=coordinator" \
  -o jsonpath="{.items[0].metadata.name}"
```

Or save to a shell variable for the session:

```bash
TRINO=$(kubectl get pods -n composable-supply-chain \
  -l "app.kubernetes.io/name=trino,app.kubernetes.io/component=coordinator" \
  -o jsonpath="{.items[0].metadata.name}")
echo $TRINO   # e.g. trino-coordinator-c45c844bd-9pfdc
```

---

## Step 4 — Run Trino queries

### Interactive shell
```bash
kubectl exec -it -n composable-supply-chain $TRINO -- trino
```

Inside the shell:
```sql
SHOW SCHEMAS FROM hive;            -- list all schemas
SHOW TABLES FROM hive.s3_access_test;  -- list our registered tables
SELECT * FROM hive.s3_access_test.fact_iosa_nrt LIMIT 5;
```

### One-shot query (non-interactive)
```bash
kubectl exec -n composable-supply-chain $TRINO -- \
  trino --execute "SELECT COUNT(*) FROM hive.s3_access_test.fact_iosa_nrt"
```

### Export to TSV (for Excel / Python analysis)
```bash
kubectl exec -n composable-supply-chain $TRINO -- \
  trino --execute "SELECT * FROM hive.s3_access_test.fact_iosa_nrt LIMIT 100000" \
  --output-format TSV_HEADER > ~/Desktop/iosa_nrt.tsv
```

> Remove `LIMIT` to get the full dataset. Large exports can take several minutes.

---

## Step 5 — Available tables

All live tables are registered in `hive.s3_access_test`. These are **external tables** — they point directly at S3 and are always current.

| Table | Rows | Freshness | Description |
|-------|------|-----------|-------------|
| `fact_iosa_nrt` | ~3.3M | NRT (updates ~10:10 UTC daily) | On-shelf availability per SKU × store. Key KPIs: in-stock %, IOSA accuracy % |
| `fact_in_stock_latest` | ~3.3M | Daily | Same scope as fact_iosa_nrt but from Redshift copy — use fact_iosa_nrt instead |
| `item_inventory` | ~several M | 2024-04-17 snapshot | Begin/end on-hand qty, unit cost, unit price from raw.new bucket |
| `stock_adjustment_sample` | ~141k | 2025-01 → present | Vendor stock corrections uploaded to smds.cl.analytics — processed Parquet |

### Useful queries

**Overall in-stock and IOSA KPIs (today):**
```sql
SELECT
  COUNT(*) AS total_rows,
  COUNT(DISTINCT item_id) AS unique_skus,
  COUNT(DISTINCT location_id) AS unique_stores,
  ROUND(SUM(CAST(instock AS integer)) * 100.0 / COUNT(*), 1) AS pct_instock,
  ROUND(SUM(CAST(iosa AS integer)) * 100.0 / COUNT(*), 1) AS pct_iosa,
  CAST(MAX(calendar_dt) AS varchar) AS latest_date
FROM hive.s3_access_test.fact_iosa_nrt;
```

**Out-of-stock by store (bottom 20):**
```sql
SELECT
  location_id,
  COUNT(*) AS total_skus,
  SUM(CAST(instock AS integer)) AS in_stock,
  ROUND(SUM(CAST(instock AS integer)) * 100.0 / COUNT(*), 1) AS pct_instock
FROM hive.s3_access_test.fact_iosa_nrt
GROUP BY location_id
ORDER BY pct_instock ASC
LIMIT 20;
```

**Stock adjustments by supplier:**
```sql
SELECT
  supplier_dni,
  COUNT(*) AS total_adjustments,
  COUNT(DISTINCT location_id) AS stores_affected,
  COUNT(DISTINCT CAST(item_id AS varchar)) AS unique_skus,
  status,
  CAST(MAX(dia) AS varchar) AS latest_adjustment
FROM hive.s3_access_test.stock_adjustment_sample
GROUP BY supplier_dni, status
ORDER BY total_adjustments DESC;
```

**FACT_IN_STOCK for a specific date** (you need to register the table first):
```sql
-- Register once per date you want to query:
CREATE TABLE IF NOT EXISTS hive.s3_access_test.fact_in_stock_20260512 (
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
) WITH (
  external_location = 's3://cencosud.prod.sm.cl.raw/CHI_SUPER_DIM_TB/FACT_IN_STOCK/calendar_dt=2026-05-12/',
  format = 'PARQUET'
);

-- Then query:
SELECT location_name, COUNT(*) AS oos_items
FROM hive.s3_access_test.fact_in_stock_20260512
WHERE enabled_ind = '1' AND (stock_q = '0.0000' OR stock_q IS NULL)
GROUP BY location_name
ORDER BY oos_items DESC;
```

---

## Step 6 — List S3 bucket contents (without Trino)

When you need to explore a bucket or prefix directly:

```bash
# Spin up a temporary pod with the namespace's IRSA credentials
kubectl run s3-probe \
  --image=amazon/aws-cli --restart=Never \
  --namespace=composable-supply-chain \
  --overrides='{
    "spec": {
      "serviceAccountName": "composable-supply-chain",
      "nodeSelector": {"node.ccom.io/role": "andinolabs"},
      "containers": [{
        "name": "c",
        "image": "amazon/aws-cli",
        "command": ["aws", "s3", "ls", "s3://cencosud.prod.smds.cl.analytics/STOCK_ADJUSTMENT/", "--human-readable"]
      }]
    }
  }'

# Wait ~30-60 seconds for the pod to pull the image and run, then get output and clean up:
kubectl logs s3-probe -n composable-supply-chain
kubectl delete pod s3-probe -n composable-supply-chain
```

Replace the S3 path and `aws s3` subcommand as needed (`ls`, `ls --recursive --summarize`, etc.).

> ⚠️ **ListBucket permissions vary by bucket.** This ephemeral pod approach only works where `s3:ListBucket` is granted. Buckets with ListBucket access: `cencosud.prod.smds.cl.analytics`, `cencosud.prod.sm.cl.raw.new`. The main raw bucket (`cencosud.prod.sm.cl.raw`) does NOT allow ListBucket from pods — use Trino queries instead to access its data.

---

## S3 Bucket Access Map

| Bucket | Encryption | Access Status | Notes |
|--------|-----------|---------------|-------|
| `cencosud.prod.sm.cl.raw` | SSE-S3 | ⚠️ GetObject ✅ / ListBucket ❌ | `FACT_IOSA_NRT`, `FACT_IN_STOCK` — query via Trino only, not `aws s3 ls` |
| `cencosud.prod.sm.cl.raw.new` | SSE-S3 | ✅ Full (GetObject + ListBucket) | `ITEM_INVENTORY` snapshot |
| `cencosud.prod.smds.cl.analytics` | SSE-KMS | ✅ Full (KMS granted) | `STOCK_ADJUSTMENT/MASTER/` Parquet files |
| `cencosud.prod.sm.cl.analytics` | SSE-KMS | ⚠️ ListBucket ✅ / GetObject ❌ | `VW_FACT_DAILY_INVENTORY_NRT` — blocked by KMS. Pending Gustavo Tejeda fix. |
| `cencosud.prod.sm.cl.ldm` | SSE-S3 | ❌ Access denied | POS `sales_transaction_line` lives here |
| `cl-jumbo-tools-sm` | SSE-S3 | ❌ Different AWS account | Historical ecommerce data (2020-2021). Hive metastore has metadata but Trino queries fail. |

---

## Shell aliases (add to ~/.zshrc)

```bash
# ── Cencosud EKS ───────────────────────────────────────────────────────────────

# Daily auth — run this every morning or after token expiry
alias cenco-login='cd ~/Cencosud-Infra/dpt-tas-magic-utils-cli-token && bash k8s_v2.sh -l'

# Get live Trino coordinator pod name
alias trino-pod='kubectl get pods -n composable-supply-chain -l "app.kubernetes.io/name=trino,app.kubernetes.io/component=coordinator" -o jsonpath="{.items[0].metadata.name}"'

# Interactive Trino SQL shell
alias trino-shell='kubectl exec -it -n composable-supply-chain $(trino-pod) -- trino'

# One-shot query: trino-q "SELECT COUNT(*) FROM hive.s3_access_test.fact_iosa_nrt"
trino-q() {
  kubectl exec -n composable-supply-chain $(trino-pod) -- trino --execute "$1"
}

# Export query to TSV: trino-export "SELECT * FROM ..." ~/Desktop/out.tsv
trino-export() {
  kubectl exec -n composable-supply-chain $(trino-pod) -- \
    trino --execute "$1" --output-format TSV_HEADER > "$2"
  echo "Exported to $2 ($(wc -l < "$2") lines)"
}

# List an S3 prefix via ephemeral pod (inherits IRSA)
# Usage: s3-ls s3://cencosud.prod.sm.cl.raw/CHI_SUPER_NRT_TB/
s3-ls() {
  local podname="s3-probe-$$"
  kubectl run "$podname" \
    --image=amazon/aws-cli --restart=Never \
    --namespace=composable-supply-chain \
    --overrides="{
      \"spec\": {
        \"serviceAccountName\": \"composable-supply-chain\",
        \"nodeSelector\": {\"node.ccom.io/role\": \"andinolabs\"},
        \"containers\": [{
          \"name\": \"c\",
          \"image\": \"amazon/aws-cli\",
          \"command\": [\"aws\", \"s3\", \"ls\", \"$1\", \"--human-readable\"]
        }]
      }
    }" 2>/dev/null
  sleep 5
  kubectl logs "$podname" -n composable-supply-chain
  kubectl delete pod "$podname" -n composable-supply-chain 2>/dev/null
}
```

---

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| `no such host` on kubectl commands | macOS Go DNS ignores VPN resolver | Add EKS endpoint to `/etc/hosts` (see Step 1) |
| `context deadline exceeded` on kubectl | Session token expired | Re-run `bash k8s_v2.sh -l` → select `1) production` |
| `Connect timeout on STS endpoint` | VPN issue or token expired | Re-run `bash k8s_v2.sh -l` |
| `k8s_v2.sh -l` hangs/appears stuck | Script waiting for menu input | Type `1` and press Enter — the menu may not appear visibly |
| Trino query `Failed to open S3 file` | KMS GetObject denied | That bucket's KMS key doesn't grant `kms:Decrypt` to our role — see bucket map above |
| `Failed fetching partitions` | Hive metastore has metadata but S3 bucket is in different account | That schema's S3 is inaccessible (e.g. `cl-jumbo-tools-sm`) — use a different table |
| `Run-as identity must be set` | Trino view missing owner identity | Query the underlying table instead of the view |
| `Unsupported Trino column type` | Parquet files in same prefix have different schemas across vendors | Use broader type (`double` instead of `bigint`, `varchar` as fallback) |

---

## Data Sources Reference

See [S3_TRINO_CATALOG.md](../data_catalog/S3_TRINO_CATALOG.md) for the complete catalog of all S3 datasets, column definitions, sample queries, and cross-system join keys.

See [REDSHIFT_CATALOG.md](../data_catalog/REDSHIFT_CATALOG.md) for the Cencosud IT Redshift warehouse schema.
