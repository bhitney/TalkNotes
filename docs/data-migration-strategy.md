# Data Migration Strategy: Source-to-Fabric Pathways

> **Project:** AtlMigration — Enterprise Data Platform Migration to Microsoft Fabric
> **Author:** Naomi (Data Engineer)
> **Date:** 2026-04-08
> **Status:** ACTIVE
> **Timeline:** 4 weeks
> **Owner:** Brian Hitney

---

## Executive Summary

This document maps every data source to its Fabric landing path. Four source systems — SQL Server on-prem, Azure Synapse DW, PostgreSQL, and MongoDB — each require a distinct migration approach based on connector availability, latency requirements, transformation complexity, and cost. Amazon S3 data is accessed via OneLake shortcuts with zero data movement.

**Guiding principles:**
1. **Mirror where we can.** Free compute, near real-time, minimal maintenance.
2. **ETL where we must.** Only when mirroring is unavailable or transformations are required on ingest.
3. **Shortcuts where data doesn't need to move.** S3 stays in S3.
4. **Validate at every layer transition.** No data moves forward without quality checks.

---

## 1. Migration Approach Per Source

### 1.1 SQL Server On-Premises → Fabric

```
┌──────────────────┐      Mirroring (CDC)        ┌──────────────────┐
│   SQL Server      │ ────────────────────────▶   │   Bronze         │
│   On-Prem         │   via On-Prem Gateway       │   Lakehouse      │
│   (2016–2025)     │   (near real-time, ~15s)    │   (Delta tables) │
└──────────────────┘                              └──────────────────┘
```

#### Option A: Fabric Mirroring for SQL Server ⭐ RECOMMENDED

| Attribute | Detail |
|-----------|--------|
| **Mechanism** | Change Data Capture (CDC) for SQL Server 2016–2022; native Fabric change feed for SQL Server 2025 |
| **Latency** | Near real-time (~15 seconds) |
| **Output format** | Read-only Delta tables in OneLake |
| **Compute cost** | Free — background replication is not charged against CU |
| **Storage cost** | Free up to 1 TB per CU (64 TB on F64 SKU) |
| **Transformation** | None — raw replication preserving source fidelity |
| **Setup complexity** | Low — point-and-click configuration in Fabric portal |
| **Maintenance** | Low — fully managed, auto-detects schema changes (new tables) |
| **Gateway required** | Yes — on-premises data gateway (HA mode, 2+ nodes recommended) or VNet data gateway |
| **Supported versions** | SQL Server 2016, 2017, 2019, 2022, 2025 |

**How it works:**
1. An on-premises data gateway is installed on a server with network access to the SQL Server instance.
2. Fabric Mirroring is configured to connect through the gateway.
3. CDC is enabled on the source database (if SQL Server 2016–2022). SQL Server 2025 uses the native change feed without requiring CDC.
4. Initial snapshot copies all selected tables to OneLake as Delta/Parquet files.
5. Ongoing changes are captured and replicated in near real-time (~15-second intervals).
6. Delta tables appear in the Bronze Lakehouse with a SQL analytics endpoint for ad-hoc T-SQL queries.

**Strengths:**
- Zero pipeline code to write, monitor, or maintain.
- Free replication compute eliminates ongoing CU cost for ingestion.
- Near real-time means Bronze layer is always fresh for downstream transforms.
- Schema evolution is auto-detected — new tables appear automatically.

**Limitations:**
- Replicates full tables (all columns). Cannot select a column subset at the mirroring level.
- Data lands read-only — no DML allowed on mirrored tables directly.
- Requires CDC overhead on the source SQL Server (minor impact on well-sized instances).
- If the gateway goes down, replication pauses until the gateway recovers.

#### Option B: Data Pipelines (ADF-Based Copy Activity)

| Attribute | Detail |
|-----------|--------|
| **Mechanism** | Scheduled batch copy via Fabric Data Factory Copy Activity |
| **Latency** | Batch — scheduled intervals (e.g., every 15 min, hourly, daily) |
| **Output format** | Delta tables or Parquet files in OneLake |
| **Compute cost** | CU consumed per pipeline run — scales with data volume |
| **Transformation** | Yes — mapping, filtering, column selection during copy |
| **Setup complexity** | Medium — pipeline authoring, watermark management |
| **Maintenance** | Medium — monitor pipeline runs, handle failures |

**When to use instead of mirroring:**
- You need only a small subset of columns (mirroring replicates everything).
- The source SQL Server cannot support CDC overhead (legacy/undersized instances).
- You need to reshape data during ingestion (flatten, join, filter before landing).
- One-time historical loads where mirroring's initial snapshot would be too large.

#### Option C: Dataflows Gen2 (Power Query-Based)

| Attribute | Detail |
|-----------|--------|
| **Mechanism** | Power Query M engine running in Fabric |
| **Latency** | Batch — scheduled refresh |
| **Output format** | Delta tables in Lakehouse or rows in Warehouse |
| **Compute cost** | CU consumed per dataflow refresh |
| **Transformation** | Yes — full Power Query M transformation capabilities |
| **Setup complexity** | Medium — no-code/low-code Power Query UI |
| **Maintenance** | Medium — monitor refresh, handle schema changes |

**When to use:**
- Business analysts or citizen developers need to define transformations.
- Light, well-defined transformations on ingest (type conversions, basic filtering).
- Not recommended for high-volume tables due to Power Query engine overhead.

#### ✅ Recommendation: Mirroring

**Rationale:** SQL Server is the highest-priority source (P0). Mirroring delivers the best combination of cost (free compute, free storage), latency (near real-time), and simplicity (no pipeline code). The on-premises data gateway is the only infrastructure requirement. Keep Data Pipelines as a fallback for any tables where CDC is problematic.

**Action items:**
1. Install on-premises data gateway in HA mode (2+ nodes) — **Week 1, Day 1**
2. Configure mirroring for 1–2 pilot databases — **Week 1, Day 3**
3. Expand to all production databases — **Week 2, Day 1**
4. Validate row counts against source — **Week 2, Day 5**

---

### 1.2 Azure Synapse DW → Fabric

```
┌──────────────────┐   Migration Assistant      ┌──────────────────┐
│   Synapse DW      │   (DDL + Data Copy)        │   Gold           │
│   (Dedicated      │ ────────────────────────▶  │   Warehouse      │
│    SQL Pool)      │                            │   (T-SQL)        │
└──────────────────┘                             └──────────────────┘
         │
         │  SpectralCore SQL Tran
         │  (T-SQL translation)
         ▼
   ┌─────────────┐
   │  Refactored  │
   │  T-SQL Code  │
   └─────────────┘
```

> **This is the most complex migration.** It's not simple replication — it's a platform migration involving schema conversion, data type mapping, and T-SQL code translation.

#### Native Migration: Fabric Migration Assistant

| Attribute | Detail |
|-----------|--------|
| **What it does** | Automated DDL conversion + data copy from Synapse dedicated SQL pool to Fabric Warehouse |
| **Schema handling** | Converts table definitions, handles data type mapping automatically |
| **Data copy** | Copies data from Synapse to Fabric Warehouse |
| **Scope** | Tables, views, basic constraints |

**Key data type mappings (Synapse → Fabric Warehouse):**

| Synapse DW Type | Fabric Warehouse Type | Notes |
|-----------------|-----------------------|-------|
| `money` | `decimal(19,4)` | Precision preserved |
| `smallmoney` | `decimal(10,4)` | Precision preserved |
| `datetime` | `datetime2` | Higher precision in Fabric |
| `smalldatetime` | `datetime2` | Higher precision in Fabric |
| `nchar` | `char` | Unicode handled natively in Fabric |
| `nvarchar` | `varchar` | Unicode handled natively in Fabric |
| `tinyint` | `smallint` | Wider range in Fabric |
| `binary` | `varbinary` | Variable-length in Fabric |
| `datetimeoffset` | `datetime2` | ⚠️ **Loses timezone offset** — extract to separate column before migration |

#### SpectralCore SQL Tran

| Attribute | Detail |
|-----------|--------|
| **What it does** | Analyzes and translates Synapse-specific T-SQL constructs to Fabric-compatible T-SQL |
| **Scope** | Stored procedures, views, functions, complex DML |
| **Output** | Refactored T-SQL code + compatibility report identifying manual fixes needed |

**What SQL Tran handles:**
- `DISTRIBUTION` clauses (hash, round-robin, replicate) → removed (Fabric manages distribution)
- `CLUSTERED COLUMNSTORE INDEX` → converted to Fabric-supported index types
- `CTAS` (CREATE TABLE AS SELECT) patterns → refactored to standard T-SQL
- `IDENTITY` column behavior differences → adjusted for Fabric semantics
- Synapse-specific system views and DMVs → mapped to Fabric equivalents
- Unsupported T-SQL functions → flagged for manual refactoring

**When SQL Tran is needed:**
- You have significant stored procedure logic (>10 procs or complex business rules).
- The Synapse DW uses Synapse-specific constructs (distributions, CTAS patterns).
- You need a compatibility report before committing to the migration.

**When SQL Tran is NOT needed:**
- The Synapse DW contains only simple tables and basic views.
- No stored procedures or functions exist.
- The Migration Assistant alone covers the DDL conversion.

**Limitations:**
- Separate tool to license — evaluate cost vs manual refactoring effort.
- Output still requires human validation — not a blind "run and deploy."
- Cannot translate every construct — some will require manual fixes.
- Budget approximately 30% of Synapse migration time for manual fixes after SQL Tran output.

#### Destination: Fabric Warehouse vs Fabric Lakehouse

| Factor | Fabric Warehouse | Fabric Lakehouse |
|--------|------------------|------------------|
| **Best for Synapse migration** | ✅ Yes — T-SQL native | Requires Spark rewrite |
| **Stored procedure support** | ✅ Full T-SQL DML/DDL | No stored procedures |
| **Multi-table transactions** | ✅ Yes | No |
| **Direct Lake support** | ✅ Yes | ✅ Yes |
| **Development paradigm** | T-SQL | Spark / Python |

**Decision: Synapse DW migrates to Fabric Warehouse (Gold layer).** The T-SQL compatibility makes this the natural landing zone. All other sources use Lakehouse.

#### ✅ Recommendation: Migration Assistant + SpectralCore SQL Tran

**Rationale:** The Synapse DW migration involves both data AND code. The Migration Assistant handles schema and data. SQL Tran handles the T-SQL code translation that the Migration Assistant doesn't cover. Together they minimize manual effort within our 4-week window.

**Execution plan:**
1. Run SQL Tran compatibility assessment — **Week 1, Day 3** → produces a compatibility report
2. Begin schema migration via Migration Assistant — **Week 2, Day 3** → DDL converted
3. Begin T-SQL code translation (SQL Tran output + manual fixes) — **Week 2, Day 4**
4. Data copy to Fabric Warehouse — **Week 2–3** → for large datasets, use Data Pipelines
5. Parallel validation queries (Synapse vs Fabric results comparison) — **Week 3, Day 2**
6. Cutover — **Week 4** (after UAT)

---

### 1.3 PostgreSQL → Fabric

```
Scenario A: Azure Database for PostgreSQL (Flexible Server)

┌──────────────────┐      Mirroring (WAL CDC)    ┌──────────────────┐
│  Azure DB for     │ ────────────────────────▶   │   Bronze         │
│  PostgreSQL       │   (native support, GA)      │   Lakehouse      │
│  Flexible Server  │                             │   (Delta tables) │
└──────────────────┘                              └──────────────────┘

Scenario B: Self-Hosted PostgreSQL

┌──────────────────┐      Data Pipelines         ┌──────────────────┐
│  Self-Hosted      │ ────────────────────────▶   │   Bronze         │
│  PostgreSQL       │   (batch, via gateway)      │   Lakehouse      │
│                   │                             │   (Delta tables) │
└──────────────────┘                              └──────────────────┘
```

#### Fabric Mirroring for PostgreSQL

| Attribute | Detail |
|-----------|--------|
| **Availability** | ✅ Generally Available (GA) for Azure Database for PostgreSQL — Flexible Server |
| **Mechanism** | WAL-based Change Data Capture (Write-Ahead Log) |
| **Latency** | Near real-time |
| **Compute cost** | Free — same as SQL Server mirroring |
| **Storage cost** | Free up to 1 TB per CU |
| **Supported tiers** | General Purpose and Memory Optimized (⚠️ **NOT Burstable**) |
| **HA support** | Yes — replication continues across failover events |
| **Network** | Public access AND network-isolated servers (via VNet data gateway) |

**Requirements for PostgreSQL mirroring:**
- Must be Azure Database for PostgreSQL — Flexible Server (not Single Server, not self-hosted)
- WAL level must be set to `logical` (may require a server restart if not already configured)
- General Purpose or Memory Optimized tier (Burstable tier is not supported)

**What is NOT supported for mirroring:**
- ❌ Self-hosted PostgreSQL instances (on-prem or VM-based)
- ❌ Azure Database for PostgreSQL — Single Server (legacy offering)
- ❌ Amazon RDS for PostgreSQL
- ❌ Burstable tier on Flexible Server

#### Data Pipelines with PostgreSQL Connector

| Attribute | Detail |
|-----------|--------|
| **Availability** | ✅ PostgreSQL connector available in Fabric Data Factory |
| **Mechanism** | Batch copy via Copy Activity |
| **Latency** | Batch — scheduled intervals |
| **Gateway** | On-premises data gateway required for self-hosted or network-restricted instances |
| **Transformation** | Yes — column mapping, filtering, type conversion during copy |

#### Dataflows Gen2 with PostgreSQL Connector

| Attribute | Detail |
|-----------|--------|
| **Availability** | ✅ PostgreSQL connector available |
| **Mechanism** | Power Query M engine |
| **Best for** | Light transformations, citizen developer access |
| **Limitations** | Not ideal for high-volume tables |

#### On-Premises Data Gateway Requirements

If the PostgreSQL instance is self-hosted or behind a firewall:
- An on-premises data gateway or VNet data gateway is required.
- The gateway provides secure connectivity from Fabric to the PostgreSQL instance.
- The same gateway used for SQL Server mirroring can serve PostgreSQL connections.
- Ensure the gateway machine has network access to the PostgreSQL port (default 5432).
- Install the PostgreSQL ODBC driver on the gateway machine.

#### ✅ Recommendation: Mirroring (if Azure Flexible Server) / Data Pipelines (if self-hosted)

**Rationale:**
- If Azure-managed Flex Server: Mirroring gives us the same free compute, near real-time benefits as SQL Server. No reason to use anything else.
- If self-hosted: Mirroring is not available. Data Pipelines with the PostgreSQL connector is the reliable batch alternative.

**⚠️ Critical action item:** Confirm whether PostgreSQL is Azure-managed or self-hosted in **Week 1**. This determines our path and timeline.

**Execution plan (Azure-managed path):**
1. Confirm hosting environment — **Week 1, Day 2**
2. Verify WAL level = `logical` and tier = General Purpose or Memory Optimized — **Week 1**
3. Configure mirroring — **Week 2, Day 1**
4. Validate replication and row counts — **Week 2, Day 5**

**Execution plan (self-hosted path):**
1. Confirm hosting environment — **Week 1, Day 2**
2. Install PostgreSQL ODBC driver on the data gateway — **Week 1**
3. Build Data Pipeline with PostgreSQL connector (full load + incremental via watermark) — **Week 2, Day 1**
4. Schedule pipeline runs (frequency based on data velocity) — **Week 2**
5. Validate row counts — **Week 2, Day 5**

---

### 1.4 MongoDB → Fabric

```
┌──────────────────┐    Data Pipelines           ┌──────────────────┐
│   MongoDB         │    (Copy Activity)          │   Bronze         │
│   (Atlas or       │ ────────────────────────▶   │   Lakehouse      │
│    self-hosted)   │    + Spark for flattening   │   (Delta tables) │
└──────────────────┘                              └──────────────────┘
```

> **No mirroring connector exists for MongoDB.** This is an ETL-only path.

#### Option A: Data Pipelines with MongoDB Atlas Connector ⭐ RECOMMENDED

| Attribute | Detail |
|-----------|--------|
| **Availability** | ✅ MongoDB Atlas connector available in Fabric Data Factory |
| **Mechanism** | Copy Activity — full load, append, or upsert |
| **Authentication** | Basic auth (username/password) |
| **Gateway support** | On-premises data gateway and VNet data gateway for network-restricted instances |
| **Incremental support** | Yes — use `_id` or timestamp field as watermark |
| **Output format** | Delta tables or Parquet in Lakehouse |

**Strengths:**
- Supports full load, append, and upsert modes — covers initial migration + ongoing incremental.
- Works with MongoDB Atlas (cloud) and self-hosted MongoDB (via gateway).
- Familiar ADF pipeline paradigm — monitor runs, set alerts, handle retries.

**Limitations:**
- Deeply nested document structures may not flatten cleanly via Copy Activity alone.
- Copy Activity performs basic tabular mapping — complex arrays and nested objects may land as JSON strings.
- Schema inference may not capture all nested fields on first pass.

#### Option B: Dataflows Gen2 with MongoDB Connector

| Attribute | Detail |
|-----------|--------|
| **Availability** | ✅ MongoDB connector available in Dataflows Gen2 |
| **Mechanism** | Power Query M extraction |
| **Best for** | Simple collections with flat or lightly nested documents |
| **Limitations** | Power Query M has limited support for complex JSON manipulation |

#### Option C: Custom Spark Notebook (PyMongo)

| Attribute | Detail |
|-----------|--------|
| **Mechanism** | Spark notebook using PyMongo library to connect directly to MongoDB |
| **Control** | Full programmatic control over document traversal and schema flattening |
| **Output** | Delta tables written directly to the Bronze Lakehouse |
| **Best for** | Deeply nested documents, complex arrays, polymorphic schemas |

**When to use Spark notebooks:**
- Collections have >3 levels of nesting.
- Documents contain arrays of objects that need to be exploded into separate tables.
- Schema varies significantly across documents (polymorphic collections).
- You need custom logic (e.g., extract embedded timestamps, parse subdocuments).

**Example pattern for complex flattening:**
```python
# Spark notebook approach for nested MongoDB documents
from pyspark.sql import functions as F

# Read from MongoDB
raw_df = spark.read.format("mongodb") \
    .option("uri", connection_string) \
    .option("collection", "orders") \
    .load()

# Flatten nested structures
flattened_df = raw_df \
    .select(
        F.col("_id").alias("order_id"),
        F.col("customer.name").alias("customer_name"),
        F.col("customer.email").alias("customer_email"),
        F.explode("line_items").alias("item")
    ) \
    .select("order_id", "customer_name", "customer_email",
            F.col("item.product_id"), F.col("item.quantity"), F.col("item.price"))

# Write to Bronze Lakehouse as Delta
flattened_df.write.format("delta").mode("overwrite").saveAsTable("bronze.orders_flat")
```

#### JSON/Document Handling Considerations

| Challenge | Strategy |
|-----------|----------|
| **Nested objects** | Dot notation flattening (`customer.name` → `customer_name`) |
| **Arrays** | Explode into child tables with foreign key back to parent |
| **Polymorphic schemas** | Union schema approach — superset of all fields, NULL where absent |
| **Large documents (>1MB)** | Split extraction by top-level fields; avoid single-row wide tables |
| **ObjectId / BSON types** | Convert to string at ingestion; cast downstream as needed |
| **Dates in MongoDB** | ISODate → `timestamp` type; epoch millis → convert in Silver layer |

#### ✅ Recommendation: Data Pipelines (Copy Activity) for initial load + incremental, with Spark notebooks for complex collections

**Rationale:** Start with Copy Activity — it handles the 80% case (flat to moderately nested documents). For collections where Copy Activity produces JSON string columns, fall back to Spark notebooks with PyMongo for full control over flattening. This two-tier approach balances speed with coverage.

**Incremental strategy:**
- Use the `_id` field (monotonically increasing ObjectId) or a `updatedAt` timestamp field as the watermark.
- Schedule pipeline runs at intervals matching data velocity (15 min to 1 hour).
- For upsert mode, configure the `_id` as the key column.

**Execution plan:**
1. Analyze MongoDB collection schemas — identify nesting depth, polymorphism — **Week 1, Day 4**
2. Prototype Copy Activity ingestion on 1–2 representative collections — **Week 1, Day 4**
3. Evaluate output quality — are nested documents flattened or JSON strings? — **Week 1, Day 5**
4. Build full pipeline set (Copy Activity for simple collections, Spark for complex ones) — **Week 2, Day 2**
5. Configure incremental sync (watermark-based) — **Week 2**
6. Validate row counts and schema fidelity — **Week 2, Day 5**

---

## 2. Medallion Architecture Implementation

```
┌─────────────────┐      ┌─────────────────┐      ┌─────────────────┐
│     BRONZE       │      │     SILVER       │      │      GOLD        │
│   (Raw / Land)   │─────▶│  (Cleaned /      │─────▶│  (Curated /      │
│                  │      │   Conformed)     │      │   Modeled)       │
│   Lakehouse      │      │   Lakehouse      │      │   Lakehouse +    │
│                  │      │                  │      │   Warehouse      │
└─────────────────┘      └─────────────────┘      └─────────────────┘
  Mirrored tables           Spark notebooks          Star/snowflake
  Pipeline output           Dataflows Gen2           Direct Lake models
  S3 shortcuts              Type casting             Aggregations
                            Dedup, joins             Business logic
```

### 2.1 Bronze Layer — Raw Data Ingestion

**Purpose:** Land data as-is from source systems. No transformations. Preserve source fidelity.

| Attribute | Detail |
|-----------|--------|
| **Fabric artifact** | Lakehouse (single Bronze Lakehouse) |
| **Storage format** | Delta tables (Parquet underlying) |
| **Transformations** | None — schema-on-read |
| **Access** | SQL analytics endpoint for ad-hoc queries |

#### Data Format by Source

| Source | Ingestion Method | Format in Bronze | Location |
|--------|-----------------|------------------|----------|
| SQL Server (on-prem) | Mirroring (CDC) | Delta tables (read-only, auto-generated) | `/Tables` (auto-registered) |
| PostgreSQL | Mirroring (WAL) or Data Pipelines | Delta tables (read-only if mirrored) | `/Tables` |
| MongoDB | Data Pipelines / Spark notebooks | Delta tables | `/Tables` |
| Amazon S3 | OneLake Shortcuts | Delta/Parquet (if source is Delta/Parquet) or raw files | `/Tables` (Delta/Parquet) or `/Files` (raw) |
| Synapse DW | Migration Assistant | Directly to Gold Warehouse (bypasses Bronze/Silver) | Fabric Warehouse |

#### Naming Conventions

```
Bronze Lakehouse name: lh_bronze

Table naming pattern: {source}_{schema}_{table}

Examples:
  sqlserver_dbo_customers
  sqlserver_dbo_orders
  sqlserver_sales_invoices
  postgres_public_users
  postgres_public_sessions
  mongodb_analytics_events
  mongodb_catalog_products
```

**Rules:**
- All lowercase, underscores as separators.
- Source prefix identifies origin system (aids lineage tracking).
- Schema prefix preserves source schema context.
- Mirrored tables retain their original names within the mirrored database context (Fabric auto-names them).

#### Partitioning Strategy (Bronze)

| Scenario | Partitioning | Rationale |
|----------|-------------|-----------|
| Mirrored tables | No manual partitioning | Mirroring manages Delta file layout automatically |
| Pipeline-loaded tables (large, time-series) | Partition by date (e.g., `year`, `month`) | Enables efficient incremental processing and query pruning |
| Pipeline-loaded tables (small, reference) | No partitioning | Overhead not justified for small tables |
| MongoDB collections | Partition by ingestion date | Enables incremental reprocessing |

#### Retention Policy

- Bronze data is retained indefinitely as the system of record for raw source data.
- Delta table history (time travel) retained for 30 days to enable reprocessing.
- VACUUM operations scheduled weekly to reclaim storage from expired file versions.

---

### 2.2 Silver Layer — Data Cleansing and Conforming

**Purpose:** Clean, deduplicate, standardize, and conform data. Make it query-ready for analysts and Gold layer processing.

| Attribute | Detail |
|-----------|--------|
| **Fabric artifact** | Lakehouse (single Silver Lakehouse) |
| **Storage format** | Delta tables (Parquet underlying) |
| **Processing engine** | Spark notebooks (primary), Dataflows Gen2 (secondary) |
| **Access** | SQL analytics endpoint for analyst queries |

#### Transformation Patterns

| Transformation | Description | Example |
|----------------|-------------|---------|
| **Type casting** | Standardize data types across sources | `VARCHAR` dates → `DATE`; `STRING` amounts → `DECIMAL(18,2)` |
| **Null handling** | Apply consistent null treatment | Replace empty strings with NULL; default values for required fields |
| **Deduplication** | Remove duplicate records | Window function on business key, keep latest by timestamp |
| **Column renaming** | Conform to standard naming conventions | `CustName` → `customer_name`; `ACCT_NO` → `account_number` |
| **Data type normalization** | Align types across sources for the same concept | SQL Server `datetime2` and PostgreSQL `timestamptz` → `TIMESTAMP` |
| **Join enrichment** | Combine related tables from the same source | Customer + Address → enriched customer record |
| **Filtering** | Remove test data, soft-deleted records | `WHERE is_deleted = 0 AND environment = 'production'` |
| **Code standardization** | Map codes to standard values | Country codes → ISO 3166; currency codes → ISO 4217 |

#### Slowly Changing Dimensions (SCD) Handling

| SCD Type | Strategy | Implementation |
|----------|----------|----------------|
| **Type 1 (Overwrite)** | Latest value overwrites previous | `MERGE` statement or Delta `MERGE INTO` with update |
| **Type 2 (History)** | Track full history with effective dates | Add `effective_start_date`, `effective_end_date`, `is_current` columns. Use Delta `MERGE` with insert for new version, update end date for previous. |
| **Type 3 (Previous value)** | Keep current + one previous value | Add `previous_{column}` columns. Rarely used — prefer Type 2. |

**Default:** Use **SCD Type 2** for dimensions that require historical tracking (customers, products, employees). Use **SCD Type 1** for reference data that doesn't need history (country codes, status lookups).

#### Naming Conventions

```
Silver Lakehouse name: lh_silver

Table naming pattern: {domain}_{entity}

Examples:
  sales_customers
  sales_orders
  sales_order_lines
  catalog_products
  catalog_categories
  analytics_user_sessions
  analytics_events
```

**Rules:**
- Source system prefix is dropped — Silver is source-agnostic.
- Domain prefix groups related entities (sales, catalog, analytics, finance, etc.).
- Entity names are plural, descriptive, business-aligned.

---

### 2.3 Gold Layer — Business-Ready Models

**Purpose:** Dimensional models (star/snowflake schemas) optimized for analytics. This is the layer that feeds Direct Lake semantic models and Power BI reports.

| Attribute | Detail |
|-----------|--------|
| **Fabric artifact** | Lakehouse (primary) + Warehouse (for Synapse T-SQL workloads) |
| **Storage format** | Delta tables |
| **Processing engine** | Spark notebooks (Lakehouse) or T-SQL (Warehouse) |
| **Access** | Direct Lake semantic models → Power BI |

#### Denormalization Strategy

Gold layer tables follow **star schema** principles:

| Table Type | Design Pattern | Example |
|------------|---------------|---------|
| **Fact tables** | Narrow, event-grain, foreign keys to dimensions | `fact_sales` (order_key, product_key, customer_key, date_key, quantity, amount) |
| **Dimension tables** | Wide, descriptive attributes, surrogate keys | `dim_customer` (customer_key, customer_id, name, email, segment, region) |
| **Date dimension** | Pre-built calendar table | `dim_date` (date_key, date, day_of_week, month, quarter, year, fiscal_year) |
| **Bridge tables** | Many-to-many relationships | `bridge_customer_product` (for multi-valued dimensions) |
| **Aggregate tables** | Pre-computed summaries for performance | `agg_daily_sales` (date_key, product_key, total_quantity, total_amount) |

#### Grain Decisions

| Fact Table | Grain | Rationale |
|------------|-------|-----------|
| `fact_sales` | One row per order line item | Lowest useful grain for sales analysis |
| `fact_user_sessions` | One row per session | Enables session-level analytics |
| `fact_events` | One row per event | Full granularity for event analysis |
| `agg_daily_sales` | One row per product per day | Pre-aggregated for dashboard performance |

**Rule:** Always model at the lowest useful grain. Pre-aggregate only when performance requires it (and test Direct Lake performance first — it may be fast enough without aggregates).

#### Naming Conventions

```
Gold Lakehouse name: lh_gold
Gold Warehouse name: wh_gold (Synapse T-SQL workloads only)

Table naming pattern:
  Fact tables:      fact_{business_process}
  Dimensions:       dim_{entity}
  Bridges:          bridge_{relationship}
  Aggregates:       agg_{description}
  Date dimension:   dim_date

Examples:
  fact_sales
  fact_user_sessions
  dim_customer
  dim_product
  dim_date
  dim_geography
  bridge_customer_segment
  agg_daily_sales_by_product
```

---

### 2.4 Layer-to-Layer Processing

#### Options Evaluated

| Processing Tool | Best For | Strengths | Limitations |
|----------------|----------|-----------|-------------|
| **Spark Notebooks** | Bronze→Silver, Silver→Gold complex transforms | Full Spark/Python power, handles any data volume, version-controlled | Requires coding, CU intensive for large jobs |
| **Data Pipelines** | Orchestration, scheduling, dependencies | Orchestrate multiple notebooks/dataflows, retry logic, monitoring | Not a transformation engine itself — orchestrates other tools |
| **Dataflows Gen2** | Simple Silver transforms, citizen developer use cases | No-code Power Query UI, familiar to Power BI users | Limited for complex logic, slower than Spark on large datasets |

#### ✅ Recommendation: Spark Notebooks for transformation, Data Pipelines for orchestration

**Bronze → Silver:**
- Use **Spark notebooks** for all transformation logic (type casting, dedup, joins, SCD processing).
- Notebooks are version-controlled (Git integration), testable, and handle any data volume.
- Orchestrate notebook execution with **Data Pipelines** (schedule, dependencies, error handling).

**Silver → Gold:**
- Use **Spark notebooks** for dimensional modeling transforms (surrogate key generation, star schema assembly, aggregations).
- For Synapse-migrated workloads in the Fabric Warehouse, use **T-SQL stored procedures** (migrated via SQL Tran).
- Orchestrate with **Data Pipelines**.

**Why not Dataflows Gen2 for layer processing:**
- Power Query M is limited for complex joins, window functions, and SCD logic.
- Performance degrades on large datasets compared to Spark.
- Reserve Dataflows Gen2 for ad-hoc, citizen-developer use cases — not core pipeline transforms.

---

## 3. Amazon S3 Shortcuts

### How Fabric Shortcuts Work for S3

OneLake shortcuts are **symbolic links** (virtual references) to external storage. They make S3 data appear as local folders inside a Fabric Lakehouse — without copying a single byte.

```
┌──────────────────┐                               ┌──────────────────┐
│   Amazon S3       │ ◀──── OneLake Shortcut ────▶  │   Bronze         │
│   Bucket          │    (zero copy, live ref)      │   Lakehouse      │
│   /data/          │                               │   /Tables or     │
│                   │                               │    /Files         │
└──────────────────┘                                └──────────────────┘
```

**How it works:**
1. Create a Cloud Connection in Fabric with S3 credentials.
2. Create a shortcut inside the Bronze Lakehouse pointing to an S3 bucket/prefix.
3. The S3 data appears as a folder in the Lakehouse.
4. Spark, SQL (via analytics endpoint), and Power BI can read the data through the shortcut as if it were local.
5. If the S3 data is in Delta or Parquet format and placed in `/Tables`, Fabric auto-registers it as a queryable table.

### Authentication

| Method | Detail |
|--------|--------|
| **AWS IAM Access Keys** | Access Key ID + Secret Access Key stored in a Fabric Cloud Connection |
| **Configuration** | Credentials stored securely in Fabric — not in code or notebooks |
| **Permissions needed** | `s3:GetObject`, `s3:ListBucket` on the target bucket/prefix |
| **Cross-account** | Supported via IAM roles with cross-account trust policies |

**Security best practice:** Create a dedicated IAM user with read-only S3 permissions. Do not use root credentials. Rotate keys on a regular schedule.

### Limitations

| Limitation | Detail |
|------------|--------|
| **Read-only** | Shortcuts are read-only. You cannot write back to S3 through a shortcut. |
| **Supported formats** | Delta, Parquet, CSV, JSON, Avro, ORC, and other common formats. Delta and Parquet in `/Tables` are auto-registered as tables. |
| **No transformation** | Shortcuts provide raw access. Any transformation requires reading through the shortcut and writing to a Lakehouse table. |
| **Egress costs** | Every read through the shortcut generates AWS S3 egress charges (data transfer out). |
| **Latency** | Access depends on S3 API performance + cross-cloud network latency. Not as fast as local OneLake data. |
| **Caching** | Shortcut caching (1–28 day retention) reduces repeat reads and egress costs. Cached data is stored in OneLake. |
| **Direct Lake compatibility** | ✅ If S3 data is Delta format in `/Tables`, Direct Lake semantic models can read it directly. Otherwise, materialize to Delta first. |

### When to Use Shortcuts vs Data Copy

| Scenario | Use Shortcut | Use Data Copy |
|----------|-------------|---------------|
| Data stays in S3, read occasionally | ✅ | |
| Need transformation before use | | ✅ Copy to Bronze, transform to Silver |
| High-frequency reads (dashboards) | ⚠️ Costly (egress) | ✅ Copy once, read locally |
| Data in Delta/Parquet format | ✅ Direct Lake compatible | Not needed |
| Data in CSV/JSON (needs schema) | ✅ (in `/Files`) + Spark to materialize | ✅ Pipeline with schema mapping |
| S3 data updated frequently | ✅ Shortcut always reads latest | Copy requires re-sync schedule |
| AWS egress budget is tight | | ✅ One-time copy, no ongoing egress |

### Configuration Recommendations

1. Create shortcuts in the **Bronze Lakehouse** under `/Tables` (for Delta/Parquet) or `/Files` (for raw formats).
2. Enable **shortcut caching** with **7-day retention** to balance egress cost vs data freshness.
3. If S3 data needs transformation, process it from shortcuts into the Silver Lakehouse via Spark notebooks. The shortcut data remains untouched in S3.
4. Monitor AWS egress costs via AWS Cost Explorer. If egress exceeds budget, consider materializing high-frequency tables to OneLake as a one-time copy.

---

## 4. Data Quality Framework

### Philosophy

> Every row that moves between layers is validated. No data advances without a quality check. If a check fails, the pipeline stops and alerts — it does not silently propagate bad data.

### 4.1 Validation Checks at Each Layer Transition

#### Source → Bronze Validation

| Check | Method | Threshold | Action on Failure |
|-------|--------|-----------|-------------------|
| **Row count reconciliation** | Compare source table count vs Bronze table count | Must match within 0.1% | Alert + investigate. If mirroring, check replication lag. If pipeline, check watermark. |
| **Schema match** | Compare source column list/types vs Bronze schema | Must match exactly | Alert. For mirroring, check if schema evolution was captured. For pipelines, update mapping. |
| **Null check on primary keys** | Assert no NULLs in PK columns in Bronze | Zero NULLs | Block downstream processing. Investigate source. |
| **Freshness check** | Verify last modified timestamp is within expected window | Based on source SLA (e.g., within 1 hour for mirrored, within schedule for batch) | Alert if stale. Check mirroring status or pipeline run history. |
| **Sample data spot check** | Compare 100 random rows source vs Bronze | Data matches exactly | Investigate mismatches — potential encoding or type conversion issues. |

#### Bronze → Silver Validation

| Check | Method | Threshold | Action on Failure |
|-------|--------|-----------|-------------------|
| **Row count delta** | Compare Bronze count vs Silver count (accounting for expected filters) | Variance explained by documented filter rules | Alert if unexpected rows lost/gained. |
| **Deduplication verification** | Assert no duplicate business keys in Silver | Zero duplicates on business key | Block Gold processing. Fix dedup logic. |
| **Type conversion validation** | Verify all type casts succeeded (no data truncation or overflow) | Zero conversion errors | Alert. Review type mapping. |
| **Null introduction check** | Compare NULL counts Bronze vs Silver per column | NULLs should not increase unless by design (e.g., null coalescing) | Investigate unexpected NULLs. |
| **Referential integrity** | Foreign key relationships valid in Silver | 100% FK resolution (or documented exceptions) | Alert. Check join logic. |

#### Silver → Gold Validation

| Check | Method | Threshold | Action on Failure |
|-------|--------|-----------|-------------------|
| **Fact-dimension join coverage** | Every FK in fact table has a matching dimension row | 100% (or has a "Unknown" dimension row for unmatched) | Alert. Add Unknown member or investigate missing dimension data. |
| **Aggregate reconciliation** | Sum of fact measures matches Silver source totals | Must match within rounding tolerance (0.01%) | Alert. Review aggregation logic. |
| **Grain validation** | Assert uniqueness on the fact table grain (composite key) | Zero duplicates on grain key | Block publication. Fix Gold transform. |
| **Surrogate key integrity** | All surrogate keys are unique, non-null, sequential | Zero violations | Alert. Review key generation. |
| **Business rule validation** | Domain-specific checks (e.g., amounts > 0, dates in valid range) | Per business rule | Alert with specifics. Route to data steward. |

### 4.2 Row Count Reconciliation

Row count reconciliation is the primary data completeness check. Implemented as a Spark notebook that runs after every layer transition.

**Reconciliation approach:**
```
Source Count  →  Bronze Count  →  Silver Count  →  Gold Fact Count
    ↕                ↕                 ↕                 ↕
  Must match     Must match      Variance          Variance
  within 0.1%    (filters         explained         explained
                 documented)      by grain
```

**Implementation:**
- A reconciliation notebook runs after each pipeline/notebook execution.
- Counts are logged to a `_data_quality.reconciliation_log` Delta table in each Lakehouse.
- Alerts are sent via Fabric Data Activator if thresholds are breached.
- Weekly summary reports show reconciliation trends.

### 4.3 Schema Drift Detection

Schema drift occurs when source systems add, remove, or modify columns without notice.

| Scenario | Detection Method | Response |
|----------|-----------------|----------|
| **New column added in source** | Compare source schema snapshot vs Bronze schema at each pipeline run | If mirroring: auto-captured. If pipeline: alert, update pipeline mapping. Propagate to Silver if needed. |
| **Column removed in source** | Same as above | Alert immediately. Column still exists in Bronze (historical data preserved). Update Silver transforms. |
| **Column type changed** | Type comparison on schema snapshot | Alert immediately. May break Silver transforms. Test impact before updating. |
| **Column renamed** | Column list diff | Alert. May appear as drop + add. Investigate with source team. |

**Schema snapshot strategy:**
- Store a schema snapshot (table name, column name, data type, nullable) in a `_data_quality.schema_snapshots` table after every successful ingestion.
- Compare current schema to previous snapshot. Log diffs.
- Breaking changes (drops, type changes) generate P1 alerts.
- Additive changes (new columns) generate P2 notifications.

### 4.4 Data Type Mapping Reference

#### SQL Server → Fabric (via Mirroring / Delta)

| SQL Server Type | Delta / Spark Type | Notes |
|----------------|--------------------|-------|
| `int` | `INT` | Direct mapping |
| `bigint` | `BIGINT` | Direct mapping |
| `smallint` | `SMALLINT` | Direct mapping |
| `tinyint` | `SMALLINT` | Widened (Delta has no tinyint) |
| `bit` | `BOOLEAN` | Direct mapping |
| `decimal(p,s)` | `DECIMAL(p,s)` | Direct mapping |
| `float` | `DOUBLE` | Direct mapping |
| `real` | `FLOAT` | Direct mapping |
| `money` | `DECIMAL(19,4)` | Explicit precision |
| `varchar(n)` | `STRING` | No length constraint in Delta |
| `nvarchar(n)` | `STRING` | UTF-8 in Delta |
| `char(n)` | `STRING` | No fixed-width in Delta |
| `datetime` | `TIMESTAMP` | Millisecond precision |
| `datetime2` | `TIMESTAMP` | Nanosecond precision preserved |
| `date` | `DATE` | Direct mapping |
| `time` | `STRING` | ⚠️ No native TIME type in Delta — store as string, cast in Silver |
| `uniqueidentifier` | `STRING` | UUID as string |
| `varbinary` | `BINARY` | Direct mapping |
| `xml` | `STRING` | Parse in Silver if needed |
| `geography` / `geometry` | `STRING` (WKT) | Convert to Well-Known Text |

#### PostgreSQL → Fabric (via Mirroring / Delta)

| PostgreSQL Type | Delta / Spark Type | Notes |
|----------------|--------------------|-------|
| `integer` | `INT` | Direct mapping |
| `bigint` | `BIGINT` | Direct mapping |
| `serial` / `bigserial` | `INT` / `BIGINT` | Auto-increment becomes regular int |
| `numeric(p,s)` | `DECIMAL(p,s)` | Direct mapping |
| `double precision` | `DOUBLE` | Direct mapping |
| `boolean` | `BOOLEAN` | Direct mapping |
| `varchar(n)` / `text` | `STRING` | Direct mapping |
| `timestamp` | `TIMESTAMP` | Direct mapping |
| `timestamptz` | `TIMESTAMP` | ⚠️ Timezone offset may be lost — convert to UTC in Silver |
| `date` | `DATE` | Direct mapping |
| `uuid` | `STRING` | UUID as string |
| `jsonb` / `json` | `STRING` | Parse JSON in Silver layer |
| `array` types | `ARRAY<type>` | Spark supports arrays natively |
| `bytea` | `BINARY` | Direct mapping |
| `inet` / `cidr` | `STRING` | Network types as strings |
| `interval` | `STRING` | No native interval in Delta |

#### MongoDB → Fabric (via Copy Activity / Spark)

| MongoDB / BSON Type | Delta / Spark Type | Notes |
|--------------------|--------------------|-------|
| `ObjectId` | `STRING` | Convert 12-byte ObjectId to hex string |
| `String` | `STRING` | Direct mapping |
| `Int32` | `INT` | Direct mapping |
| `Int64` / `Long` | `BIGINT` | Direct mapping |
| `Double` | `DOUBLE` | Direct mapping |
| `Decimal128` | `DECIMAL(34,6)` | Adjust precision as needed |
| `Boolean` | `BOOLEAN` | Direct mapping |
| `Date` (ISODate) | `TIMESTAMP` | Direct mapping |
| `Timestamp` (BSON) | `TIMESTAMP` | Internal MongoDB timestamp |
| `Binary` | `BINARY` | Direct mapping |
| `Array` | `ARRAY<type>` or exploded to child table | Depends on complexity |
| `Object` (nested doc) | `STRUCT` or flattened columns | Flatten in Bronze (Spark) or Silver |
| `Null` | `NULL` | Handled natively |

---

## 5. Migration Execution Order

### Priority Ranking

| Priority | Source | Why First | Dependencies | Week |
|----------|--------|-----------|-------------|------|
| **1** | SQL Server On-Prem | Highest data volume, most business-critical, P0 workstream. Foundation for Silver/Gold layers. | Gateway installation | Week 1–2 |
| **2** | Azure Synapse DW | Most complex migration (schema + code). Needs early start for the 4-week timeline. Compatibility assessment informs effort. | Architecture decisions finalized | Week 1–3 |
| **3** | PostgreSQL | Depends on hosting confirmation (Azure vs self-hosted). Once confirmed, mirroring setup is fast. | PostgreSQL hosting confirmed | Week 1–2 |
| **4** | MongoDB | No mirroring — requires pipeline development. Schema analysis needed first for complex documents. | Document schema analysis | Week 1–2 |
| **5** | Amazon S3 (Shortcuts) | Zero data movement — fastest to configure. Low risk. Can happen in parallel. | S3 bucket credentials | Week 1 |

### Execution Timeline

```
Week 1                    Week 2                    Week 3                    Week 4
├─────────────────────────┼─────────────────────────┼─────────────────────────┼─────────────────┤
│ SQL Server              │ SQL Server              │                         │                 │
│ ├─ Gateway install (D1) │ ├─ Full mirroring (D1)  │                         │ Validation      │
│ ├─ Pilot mirror (D3)    │ └─ Validate (D5)        │                         │                 │
│                         │                         │                         │                 │
│ Synapse DW              │ Synapse DW              │ Synapse DW              │                 │
│ ├─ SQL Tran assess (D3) │ ├─ Schema migrate (D3)  │ ├─ Complete T-SQL (D2)  │ Validation      │
│                         │ ├─ T-SQL translate (D4) │ └─ Parallel validation  │                 │
│                         │                         │                         │                 │
│ PostgreSQL              │ PostgreSQL              │                         │                 │
│ ├─ Confirm hosting (D2) │ ├─ Mirror or pipe (D1)  │                         │ Validation      │
│                         │ └─ Validate (D5)        │                         │                 │
│                         │                         │                         │                 │
│ MongoDB                 │ MongoDB                 │                         │                 │
│ ├─ Schema analysis (D4) │ ├─ Full pipeline (D2)   │                         │ Validation      │
│ ├─ Prototype (D4)       │ └─ Validate (D5)        │                         │                 │
│                         │                         │                         │                 │
│ S3 Shortcuts            │ S3 Shortcuts            │                         │                 │
│ └─ Test shortcut (D4)   │ └─ All buckets (D2)     │                         │                 │
│                         │                         │                         │                 │
│                         │ Silver Layer            │ Gold Layer              │                 │
│                         │ ├─ Transforms (D3)      │ ├─ Star schemas (D1)    │ End-to-end      │
│                         │ └─ QA checks (D5)       │ └─ Direct Lake (D3)     │ validation      │
└─────────────────────────┴─────────────────────────┴─────────────────────────┴─────────────────┘
```

### Parallel vs Sequential Tracks

| Track | Execution Mode | Rationale |
|-------|---------------|-----------|
| **SQL Server mirroring** | Parallel (can start Day 1 after gateway) | Independent — no dependencies on other sources |
| **Synapse DW migration** | Parallel (start assessment Day 3) | Independent path, long lead time needed |
| **PostgreSQL** | Sequential (wait for hosting confirmation) | Path depends on Azure vs self-hosted answer |
| **MongoDB** | Parallel (start analysis Day 4) | Independent — but later start due to lower priority |
| **S3 Shortcuts** | Parallel (start Day 4) | Fast, zero risk, no dependencies on other sources |
| **Silver layer** | Sequential (after Bronze complete) | Cannot transform data that hasn't landed yet |
| **Gold layer** | Sequential (after Silver complete) | Cannot model data that hasn't been cleaned |

### Dependencies Between Sources

```
Gateway Installation ──▶ SQL Server Mirroring ──▶ Bronze Landing
                    └──▶ PostgreSQL (if self-hosted, reuse gateway)

PostgreSQL Hosting Confirmation ──▶ Mirroring (Azure) OR Pipeline (self-hosted)

MongoDB Schema Analysis ──▶ Pipeline Design (simple vs Spark)

All Bronze Sources Complete ──▶ Silver Layer Processing

Silver Complete ──▶ Gold Layer Modeling

Gold Complete ──▶ Direct Lake Semantic Models ──▶ Power BI Reports
```

### Risk-Based Sequencing Rationale

1. **SQL Server first** — Largest data footprint, longest initial snapshot time. Starting early gives mirroring time to stabilize. If the gateway fails, we have time to recover.
2. **Synapse DW early** — Most complex migration with the most unknowns (T-SQL compatibility). The SQL Tran assessment in Week 1 reveals the scope of manual work. Starting late risks blowing the timeline.
3. **PostgreSQL depends on confirmation** — We literally cannot choose a path until we know the hosting. But once confirmed, either mirroring (fast) or pipelines (moderate) can complete quickly.
4. **MongoDB last among sources** — Lower priority (P1 vs P0), requires schema analysis before pipeline design. Complex documents may need Spark notebooks, which take longer to develop.
5. **S3 shortcuts anytime** — Nearly zero-effort configuration. Can be done by any team member in parallel. No risk to timeline.

---

## Appendix A: Decision Summary

| Decision | Choice | Rationale |
|----------|--------|-----------|
| SQL Server ingestion method | Mirroring (CDC) | Free compute, near real-time, minimal maintenance |
| Synapse DW migration method | Migration Assistant + SQL Tran | Schema automation + T-SQL code translation |
| Synapse DW target | Fabric Warehouse (Gold layer) | T-SQL compatibility for migrated stored procs |
| PostgreSQL ingestion (Azure) | Mirroring (WAL) | Same benefits as SQL Server mirroring |
| PostgreSQL ingestion (self-hosted) | Data Pipelines | Mirroring unavailable for self-hosted |
| MongoDB ingestion | Data Pipelines + Spark notebooks | Copy Activity for simple docs, Spark for complex |
| S3 data access | OneLake Shortcuts | Zero data movement, Direct Lake compatible |
| Layer-to-layer processing | Spark notebooks + Data Pipeline orchestration | Full Spark power, orchestrated scheduling |
| Bronze format | Delta tables (Lakehouse) | Universal format, SQL analytics endpoint |
| Silver processing | Spark notebooks | Complex transforms, version-controlled |
| Gold modeling | Star schema in Lakehouse + Warehouse | Optimized for Direct Lake and T-SQL |
| SCD strategy | Type 2 (default for dimensions) | Full history tracking |

## Appendix B: Connector and Gateway Reference

| Source | Connector | Gateway Required | Auth Method |
|--------|-----------|-----------------|-------------|
| SQL Server on-prem | Mirroring (CDC) | ✅ On-prem data gateway | Windows Auth or SQL Auth |
| Azure Synapse DW | Migration Assistant / Data Pipelines | ❌ (Azure-to-Azure) | Azure AD / Service Principal |
| PostgreSQL (Azure Flex) | Mirroring (WAL) | ❌ (public) or VNet gateway (private) | PostgreSQL Auth |
| PostgreSQL (self-hosted) | Data Pipelines (Copy Activity) | ✅ On-prem data gateway | PostgreSQL Auth |
| MongoDB Atlas | Data Pipelines (Copy Activity) | ❌ (cloud-to-cloud) | Basic Auth |
| MongoDB (self-hosted) | Data Pipelines (Copy Activity) | ✅ On-prem data gateway | Basic Auth |
| Amazon S3 | OneLake Shortcut | ❌ | AWS IAM Access Keys |

---

*This document is a living artifact. Update as hosting decisions are confirmed and migration execution reveals new findings.*
