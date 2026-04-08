# Architecture Proposal: Microsoft Fabric Migration

> **Author:** Holden (Lead / Architect)
> **Date:** 2026-04-08
> **Status:** DRAFT — Pending team review
> **Timeline:** 4 weeks
> **Owner:** Brian Hitney

---

## Executive Summary

We're migrating four data source types — SQL Server on-prem, Azure Synapse DW, PostgreSQL, and MongoDB — into Microsoft Fabric using a medallion architecture (Bronze/Silver/Gold). The Gold layer serves Direct Lake semantic models for Power BI. Amazon S3 data is accessed via OneLake shortcuts with zero data movement.

This proposal makes concrete recommendations for every source. The guiding principle: **use mirroring where we can, ETL where we must, and shortcuts where the data doesn't need to move.**

---

## 1. Overall Architecture — Medallion Design

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        MICROSOFT FABRIC                                 │
│                                                                         │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐               │
│  │    BRONZE     │    │    SILVER     │    │     GOLD     │               │
│  │  (Raw/Land)   │───▶│ (Cleaned/    │───▶│ (Curated/    │               │
│  │              │    │  Conformed)   │    │  Modeled)    │               │
│  │  Lakehouse   │    │  Lakehouse    │    │  Lakehouse   │               │
│  │              │    │              │    │  + Warehouse  │               │
│  └──────┬───────┘    └──────────────┘    └──────┬───────┘               │
│         │                                        │                       │
│  Mirrored DBs ──┐                         ┌─────┴──────┐               │
│  Shortcuts   ───┤                         │ Direct Lake │               │
│  Data Pipelines─┤                         │  Semantic   │               │
│  Dataflows Gen2─┘                         │   Models    │               │
│                                           └─────┬──────┘               │
│                                                  │                       │
│                                           ┌──────┴──────┐               │
│                                           │  Power BI    │               │
│                                           │  Reports     │               │
│                                           └─────────────┘               │
└─────────────────────────────────────────────────────────────────────────┘
```

### Layer Definitions

| Layer | Purpose | Fabric Item | Format | Transformations |
|-------|---------|-------------|--------|----------------|
| **Bronze** | Raw landing zone. Data arrives as-is from sources. | Lakehouse | Delta (Parquet) | None — schema-on-read. Preserve source fidelity. |
| **Silver** | Cleaned, deduplicated, conformed. Standard naming, data types, null handling. | Lakehouse | Delta | Light transforms: type casting, dedup, joins, filtering. Spark notebooks or Dataflows Gen2. |
| **Gold** | Business-ready, star/snowflake modeled. Optimized for queries. | Lakehouse + Warehouse | Delta | Heavy transforms: dimensional modeling, aggregations, business logic. |

### Key Design Decisions

1. **Bronze = Lakehouse only.** Raw data is semi-structured and unstructured. Lakehouse handles both. No T-SQL DML needed here.
2. **Silver = Lakehouse.** Spark notebooks for transformation. SQL analytics endpoint available for ad-hoc querying without a separate Warehouse.
3. **Gold = Lakehouse (primary) + Warehouse (for T-SQL heavy workloads).** The Lakehouse serves Direct Lake semantic models. A Fabric Warehouse is added where we need multi-table transactions or heavy T-SQL stored procedures (primarily the Synapse DW migration path).
4. **Mirrored databases land at the Bronze layer.** They produce read-only Delta tables in OneLake — perfect for raw landing.
5. **Shortcuts appear in the Bronze layer.** S3 data is referenced without movement.

---

## 2. Lakehouse vs Warehouse Decision

### Decision Guide Summary

Per the [Microsoft Fabric Lakehouse vs Warehouse Decision Guide](https://learn.microsoft.com/en-us/fabric/fundamentals/decision-guide-lakehouse-warehouse):

| Criterion | Lakehouse | Warehouse |
|-----------|-----------|-----------|
| Development style | Spark / Python / no-code | T-SQL |
| Multi-table transactions | No | Yes |
| Data types | Structured + unstructured | Structured only |
| DML support | Via Spark | Full T-SQL DML/DDL |
| Storage format | Delta (open) | Delta (open) |
| Direct Lake support | ✅ Yes | ✅ Yes |

### Our Recommendation

**Use Lakehouse as the primary artifact across all medallion layers, with a Warehouse for specific T-SQL workloads migrated from Synapse.**

Rationale:
- **Direct Lake semantic models work with both** Lakehouse and Warehouse, so this isn't a differentiator.
- Our Bronze and Silver layers are ingestion/transformation-heavy → Spark-native → **Lakehouse**.
- The Synapse DW migration brings existing T-SQL stored procedures and views → **Warehouse** for that workload at Gold.
- PostgreSQL and MongoDB data is non-relational or varied schema → **Lakehouse** is the natural fit.
- Lakehouse gives us the SQL analytics endpoint for free — T-SQL read access without a Warehouse.

**Decision: Lakehouse-first architecture. Warehouse only for migrated Synapse T-SQL workloads at the Gold layer.**

---

## 3. Migration Strategy Per Source

### 3.1 SQL Server On-Prem → Fabric

```
┌──────────────┐      Mirroring (CDC)       ┌──────────────┐
│  SQL Server   │ ─────────────────────────▶ │   Bronze     │
│  On-Prem      │   via On-Prem Gateway      │  Lakehouse   │
│  (2016-2025)  │                            │  (Delta)     │
└──────────────┘                             └──────────────┘
```

**Options Evaluated:**

| Approach | Latency | Complexity | CU Cost | Transformation | Best For |
|----------|---------|------------|---------|----------------|----------|
| **Mirroring** | Near real-time (~15s) | Low — turnkey setup | Free replication compute; free storage up to 1TB/CU | None (raw replication) | Ongoing sync, operational analytics |
| **Data Pipelines** | Batch (scheduled) | Medium — pipeline authoring | CU consumed per pipeline run | Yes — Copy + transforms | Selective table loads, heavy reshaping |
| **Dataflows Gen2** | Batch (scheduled) | Medium — Power Query UI | CU consumed per dataflow refresh | Yes — Power Query M | Citizen developer transforms |

**Recommendation: Mirroring.**

- SQL Server 2016–2025 is fully supported for Fabric Mirroring.
- Mirroring uses CDC (2016-2022) or the native Fabric change feed (2025) to replicate into OneLake as Delta tables.
- Replication compute is **free**. Storage is **free up to 1TB per CU**.
- Requires an on-premises data gateway if the SQL Server isn't publicly accessible (likely our case).
- Data lands as read-only Delta tables with a SQL analytics endpoint — this IS our Bronze layer.
- No ETL pipelines to build, monitor, or maintain.
- Transform from Bronze → Silver → Gold using Spark notebooks.

**When to use ETL instead:**
- If we only need a subset of columns (mirroring replicates full tables).
- If the source SQL Server can't support CDC overhead (older/undersized instances).
- If we need to reshape data on ingest (flatten, join, filter before landing).

**Gateway Requirement:** An on-premises data gateway or VNet data gateway is required for mirroring SQL Server behind a firewall. Plan gateway installation in Week 1.

### 3.2 Azure Synapse DW → Fabric

```
┌──────────────┐    Fabric Migration        ┌──────────────┐
│  Synapse DW   │    Assistant (DDL/Data)     │   Gold       │
│  (Dedicated   │ ─────────────────────────▶ │  Warehouse   │
│   SQL Pool)   │                            │  (T-SQL)     │
└──────────────┘                             └──────────────┘
       │
       │  SpectralCore SQL Tran
       │  (T-SQL translation)
       ▼
  ┌─────────────┐
  │ Refactored   │
  │ T-SQL Code   │
  └─────────────┘
```

**The Synapse → Fabric path is the most complex migration.** This is a dedicated SQL pool to Fabric Warehouse migration, not a simple replication.

**Options Evaluated:**

| Approach | What It Does | Pros | Cons |
|----------|-------------|------|------|
| **Fabric Migration Assistant** | Automated DDL conversion + data copy from Synapse to Fabric Warehouse | Microsoft-native tool; handles schema + data; manages data type mapping | May not handle all T-SQL constructs; requires manual review |
| **SpectralCore SQL Tran** | Translates T-SQL from Synapse-specific syntax to Fabric-compatible T-SQL | Automates code refactoring; handles Synapse-specific constructs (distributions, indexes, CTAS patterns) | Separate tool to license; output still needs validation |
| **Manual Migration** | Hand-migrate schemas, code, and data | Full control | Extremely time-consuming for 4-week timeline |

**Recommendation: Fabric Migration Assistant for DDL/data + SpectralCore SQL Tran for T-SQL code.**

Detailed approach:

1. **Schema (DDL):** Use the Fabric Migration Assistant to convert table definitions. It handles data type mapping (e.g., `money` → `decimal(19,4)`, `datetime` → `datetime2`, `nvarchar` → `varchar`).

2. **Data Migration:** The Migration Assistant can copy data from Synapse to Fabric Warehouse. For large datasets, use Data Pipelines with the Synapse connector.

3. **Database Code (DML):** This is where SpectralCore SQL Tran earns its keep:
   - Synapse dedicated SQL pools use constructs not supported in Fabric: `DISTRIBUTION`, `CLUSTERED COLUMNSTORE INDEX`, `CTAS` patterns, certain `IDENTITY` column behaviors.
   - SQL Tran analyzes stored procedures, views, and functions, translating Synapse-specific T-SQL to Fabric-compatible T-SQL.
   - It produces a compatibility report identifying manual fixes needed.
   - **Use SQL Tran when you have significant stored procedure/view logic.** If you only have simple tables + views, the Migration Assistant alone may suffice.

4. **Post-Migration Validation:** Run queries in parallel against both Synapse and Fabric Warehouse to compare results.

**Key data type mappings (Synapse → Fabric Warehouse):**

| Synapse DW | Fabric Warehouse |
|------------|-----------------|
| `money` | `decimal(19,4)` |
| `smallmoney` | `decimal(10,4)` |
| `datetime` | `datetime2` |
| `smalldatetime` | `datetime2` |
| `nchar` | `char` |
| `nvarchar` | `varchar` |
| `tinyint` | `smallint` |
| `binary` | `varbinary` |
| `datetimeoffset` | `datetime2` (⚠️ loses timezone offset — extract to separate column) |

### 3.3 PostgreSQL → Fabric

```
┌──────────────┐      Mirroring (WAL)       ┌──────────────┐
│  Azure DB for │ ─────────────────────────▶ │   Bronze     │
│  PostgreSQL   │   (native support, GA)     │  Lakehouse   │
│  Flex Server  │                            │  (Delta)     │
└──────────────┘                             └──────────────┘
```

**Options Evaluated:**

| Approach | Availability | Notes |
|----------|-------------|-------|
| **Mirroring** | ✅ GA for Azure DB for PostgreSQL Flexible Server | Near real-time, CDC via WAL. Free replication compute. |
| **Data Pipelines** | ✅ PostgreSQL connector available | Batch copy. Good for one-time loads. |
| **Dataflows Gen2** | ✅ PostgreSQL connector available | Power Query-based. Good for light transforms on ingest. |

**Recommendation: Mirroring (if Azure DB for PostgreSQL Flexible Server).**

- Fabric Mirroring for Azure Database for PostgreSQL is **generally available**.
- Uses WAL-based CDC for near real-time replication.
- Supports publicly accessible servers AND network-isolated servers (via VNet data gateway).
- Supports General Purpose and Memory Optimized tiers (not Burstable).
- Supports high availability configurations — replication continues across failover events.
- Same free compute/storage benefits as SQL Server mirroring.

**If self-hosted PostgreSQL (not Azure-managed):**
- Mirroring is NOT supported for self-hosted PostgreSQL instances.
- Use **Data Pipelines** with the PostgreSQL connector for batch ingestion.
- Use **Dataflows Gen2** if business users need to apply Power Query transforms on ingest.

**Action item: Confirm whether our PostgreSQL is Azure-managed or self-hosted.** This determines our path.

### 3.4 MongoDB → Fabric

```
┌──────────────┐    Data Pipelines /         ┌──────────────┐
│   MongoDB     │    Dataflows Gen2           │   Bronze     │
│   (Atlas or   │ ─────────────────────────▶ │  Lakehouse   │
│   self-hosted)│    (batch copy)             │  (Delta)     │
└──────────────┘                             └──────────────┘
```

**Options Evaluated:**

| Approach | Availability | Notes |
|----------|-------------|-------|
| **Mirroring** | ❌ Not supported for MongoDB | No mirroring connector exists for MongoDB. |
| **Data Pipelines (Copy Activity)** | ✅ MongoDB Atlas connector available | Supports full load, append, upsert. Basic auth. Supports on-prem and VNet gateways. |
| **Dataflows Gen2** | ✅ MongoDB connector available | Power Query-based extraction. |
| **Custom (Spark + PyMongo)** | ✅ Any Spark notebook | Full control. Handles complex document flattening. |

**Recommendation: Data Pipelines (Copy Activity) for initial load + incremental sync.**

- The **MongoDB Atlas connector** in Fabric Data Factory supports Copy Activity as both source and destination.
- Supports **full load, append, and upsert** modes — covers initial + incremental scenarios.
- Works with on-premises data gateway and VNet data gateway for network-restricted MongoDB instances.
- Authentication: Basic.

**For complex document structures:** If MongoDB collections have deeply nested documents that need flattening, use a **Spark notebook** in the Lakehouse to read via PyMongo and write Delta tables. This gives full control over schema flattening that Copy Activity can't handle.

**Incremental strategy:**
- Use the `_id` field or a timestamp field as the watermark for incremental loads.
- Schedule pipeline runs at appropriate intervals (e.g., every 15 minutes to every hour depending on data velocity).

---

## 4. Mirroring vs ETL Decision Framework

### Decision Matrix

| Factor | Mirroring | ETL (Data Pipelines / Dataflows Gen2) |
|--------|-----------|--------------------------------------|
| **Latency** | Near real-time (~15 seconds) | Batch (scheduled intervals) |
| **Compute cost** | Free replication | CU consumed per run |
| **Storage cost** | Free up to 1TB/CU | Standard OneLake rates |
| **Transformation on ingest** | None — raw replication | Yes — reshape, filter, join |
| **Table selection** | Full tables or selected tables | Any query/subset |
| **Column selection** | Full columns only | Any column subset |
| **Setup complexity** | Low — point-and-click | Medium — pipeline design |
| **Maintenance** | Low — fully managed | Medium — monitor pipeline runs |
| **Source impact** | CDC overhead on source DB | Read load during extraction |
| **Supported sources** | SQL Server, Azure SQL, PostgreSQL, Cosmos DB, Snowflake, Oracle, others | Any connector in Data Factory |
| **Schema evolution** | Auto-detects new tables | Requires pipeline updates |

### Decision Rules

```
START
  │
  ├─ Is the source supported for Mirroring?
  │   ├─ NO ──▶ Use ETL (Data Pipelines or Dataflows Gen2)
  │   │
  │   └─ YES
  │       ├─ Do you need near real-time data?
  │       │   └─ YES ──▶ Use Mirroring
  │       │
  │       ├─ Do you need to transform data on ingest?
  │       │   └─ YES ──▶ Use ETL, transform Bronze→Silver via Spark
  │       │             (or Mirror to Bronze, then transform separately)
  │       │
  │       ├─ Do you need only a subset of columns?
  │       │   └─ YES ──▶ Consider ETL if subset is small
  │       │             (Mirror is still viable — just ignore extra columns)
  │       │
  │       ├─ Is minimizing CU cost a priority?
  │       │   └─ YES ──▶ Use Mirroring (free replication compute)
  │       │
  │       └─ Default ──▶ Use Mirroring (simpler, cheaper, less maintenance)
END
```

### Per-Source Recommendations Summary

| Source | Mirroring Available? | Recommended Approach | Rationale |
|--------|---------------------|---------------------|-----------|
| SQL Server on-prem | ✅ Yes (2016+) | **Mirroring** | Free compute, near real-time, minimal maintenance |
| Azure Synapse DW | ❌ No (different migration path) | **Migration Assistant + SQL Tran** | Schema/code translation required |
| PostgreSQL (Azure) | ✅ Yes (Flex Server, GA) | **Mirroring** | Free compute, WAL-based CDC |
| PostgreSQL (self-hosted) | ❌ No | **Data Pipelines** | Batch connector |
| MongoDB | ❌ No | **Data Pipelines** | MongoDB Atlas connector, supports upsert |
| Amazon S3 | N/A | **Shortcuts** | Zero data movement |

---

## 5. Shortcuts Strategy — Amazon S3

```
┌──────────────┐                            ┌──────────────┐
│  Amazon S3    │ ◀── OneLake Shortcut ────▶ │   Bronze     │
│  Buckets      │    (zero copy, live ref)   │  Lakehouse   │
│               │                            │  /Tables or  │
│               │                            │  /Files      │
└──────────────┘                             └──────────────┘
```

### What Shortcuts Do
- OneLake shortcuts are **symbolic links** to external storage. They appear as folders inside a Lakehouse.
- **No data is copied.** Spark, SQL, and Power BI read directly from S3 through the shortcut.
- If S3 data is in Delta/Parquet format and placed in the `/Tables` folder, the Lakehouse auto-registers it as a table.
- Unstructured data (CSVs, JSON, images) goes in the `/Files` folder.

### Architecture Implications
1. **Egress costs:** Reading from S3 via shortcuts incurs AWS egress charges. Use **shortcut caching** (1–28 day retention) to minimize repeated reads.
2. **No transformation:** Shortcuts provide raw access. Transform in Silver layer via Spark.
3. **Security:** Shortcuts use Cloud Connections for auth. Requires S3 credentials stored in Fabric.
4. **Direct Lake compatibility:** If S3 data is Delta format in `/Tables`, Direct Lake semantic models can read it. Otherwise, materialize to Delta in a notebook first.
5. **Latency:** Shortcut access depends on S3 API performance + network. Cached files serve faster on repeat access.

### Recommendation
- Create S3 shortcuts in the **Bronze Lakehouse** under `/Tables` (for Delta/Parquet data) or `/Files` (for raw files).
- Enable **shortcut caching** with a 7-day retention to balance egress cost vs freshness.
- If S3 data needs transformation, process it from shortcuts into the Silver Lakehouse via Spark notebooks. The shortcut data remains untouched in S3.

---

## 6. Cost Considerations

### CU Consumption by Approach

| Activity | CU Impact | Notes |
|----------|-----------|-------|
| **Mirroring replication** | 🟢 Free | Background replication compute is not charged. |
| **Mirroring storage** | 🟢 Free up to 1TB/CU | Exceeding limit or pausing capacity triggers standard OneLake rates. |
| **Data Pipeline runs** | 🟡 Moderate | CU consumed per Copy Activity execution. Scales with data volume. |
| **Dataflow Gen2 refreshes** | 🟡 Moderate | CU consumed per refresh. Power Query engine overhead. |
| **Spark notebook execution** | 🟡 Moderate–High | CU consumed based on cluster size × duration. Use starter pools. |
| **Warehouse queries (T-SQL)** | 🟡 Moderate | CU consumed per query. Auto-scales. |
| **SQL analytics endpoint queries** | 🟡 Moderate | Same engine as Warehouse, same CU model. |
| **Direct Lake semantic model** | 🟢 Low | Reads Delta directly — no import/refresh CU cost. |
| **Shortcut reads (S3)** | 🟡 Moderate | OneLake API calls consume CU. Plus AWS egress. |
| **Power BI report rendering** | 🟢 Low | Standard Power BI capacity consumption. |

### Cost Optimization Strategy

1. **Maximize mirroring** — It's the cheapest ingestion path. Every source we mirror instead of ETL saves CU on recurring pipeline runs.
2. **Use Direct Lake, not Import mode** — Direct Lake reads Delta files in place. No scheduled refresh burns. This is a significant CU savings over Import semantic models.
3. **Right-size Spark clusters** — Use Starter Pools for Silver transforms. Avoid over-provisioning.
4. **Batch transforms in off-peak hours** — Schedule Silver → Gold transforms during low-usage windows to avoid CU contention.
5. **Enable shortcut caching for S3** — Reduces repeated egress and OneLake API calls.
6. **Minimize Warehouse usage** — Only use the Warehouse item for Synapse-migrated T-SQL workloads. Everything else uses Lakehouse + SQL analytics endpoint.

### SKU Sizing Guidance

| SKU | CUs | Mirroring Storage (Free) | Recommended For |
|-----|-----|--------------------------|-----------------|
| F2 | 2 | 2 TB | Dev/test only |
| F16 | 16 | 16 TB | Small workloads |
| F64 | 64 | 64 TB | **Recommended starting point** for this migration |
| F128 | 128 | 128 TB | If data volumes exceed 50TB or heavy concurrent queries |

**Recommendation: Start with F64.** This gives us 64TB of free mirroring storage and sufficient CU for the migration workloads. Scale up if CU utilization exceeds 80% during peak.

---

## 7. Risks and Mitigations

| # | Risk | Impact | Likelihood | Mitigation |
|---|------|--------|------------|------------|
| 1 | **On-prem data gateway instability** — Gateway is a single point of failure for SQL Server and PostgreSQL mirroring. | High | Medium | Deploy gateway in high-availability mode (2+ nodes). Monitor gateway health. Have Data Pipeline fallback ready. |
| 2 | **Synapse T-SQL incompatibilities** — Not all Synapse T-SQL constructs translate cleanly to Fabric. | Medium | High | Run SQL Tran compatibility report in Week 1. Identify manual refactoring scope early. Budget 30% of Synapse migration time for manual fixes. |
| 3 | **MongoDB document complexity** — Deeply nested documents may not flatten cleanly via Copy Activity. | Medium | Medium | Prototype MongoDB ingestion in Week 1 with representative collections. Fall back to Spark + PyMongo if Copy Activity doesn't handle nesting. |
| 4 | **CU throttling under load** — Concurrent mirroring + transforms + queries may exceed capacity. | High | Medium | Start with F64. Monitor CU utilization via Fabric Capacity Metrics app. Scale up proactively if utilization exceeds 70% sustained. |
| 5 | **4-week timeline pressure** — Scope creep or unexpected data quality issues. | High | High | Strict scope control. Prioritize: (1) Mirroring setup, (2) Synapse migration, (3) Gold layer models, (4) Power BI reports. Defer CluedIn MDM and Salesforce to Phase 2 if needed. |
| 6 | **Direct Lake limitations** — Not all DAX functions or data shapes are compatible with Direct Lake mode. | Medium | Low | Test semantic models early. Have Import mode fallback for specific models if needed. |
| 7 | **S3 egress costs** — Uncontrolled shortcut reads from S3 can generate high AWS bills. | Medium | Medium | Enable shortcut caching. Monitor egress via AWS Cost Explorer. Consider materializing high-frequency tables to OneLake if egress exceeds budget. |
| 8 | **PostgreSQL environment uncertainty** — If PostgreSQL is self-hosted (not Azure-managed), mirroring is unavailable. | Medium | Unknown | Confirm PostgreSQL hosting in Week 1 sprint planning. If self-hosted, pivot to Data Pipeline approach immediately. |

---

## 8. Implementation Sequence (4-Week Roadmap)

### Week 1: Foundation
- [ ] Provision Fabric workspace and F64 capacity
- [ ] Install and configure on-premises data gateway (HA mode)
- [ ] Set up Bronze/Silver/Gold Lakehouses
- [ ] Create Fabric Warehouse for Synapse workloads
- [ ] Configure SQL Server mirroring (start with 1-2 databases)
- [ ] Confirm PostgreSQL hosting environment
- [ ] Run SpectralCore SQL Tran compatibility assessment on Synapse DW
- [ ] Prototype MongoDB ingestion with Copy Activity

### Week 2: Ingestion
- [ ] Complete SQL Server mirroring for all databases
- [ ] Configure PostgreSQL mirroring (or Data Pipelines if self-hosted)
- [ ] Complete MongoDB Data Pipeline setup (full load + incremental)
- [ ] Create S3 shortcuts in Bronze Lakehouse
- [ ] Begin Synapse DW schema migration (Migration Assistant)
- [ ] Begin T-SQL code translation (SQL Tran)

### Week 3: Transformation + Modeling
- [ ] Build Silver layer transforms (Spark notebooks)
- [ ] Build Gold layer dimensional models
- [ ] Complete Synapse T-SQL migration and validation
- [ ] Create Direct Lake semantic models on Gold layer
- [ ] Begin Power BI report development

### Week 4: Validation + Go-Live
- [ ] End-to-end data validation (source → Bronze → Silver → Gold → Power BI)
- [ ] Performance testing and CU optimization
- [ ] Security and permissions audit
- [ ] Documentation finalization
- [ ] Stakeholder UAT
- [ ] Go-live decision

---

## Appendix A: Technology Decision Summary

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Primary Fabric item | Lakehouse | Handles structured + unstructured, Spark-native, Direct Lake support |
| Gold layer for Synapse workloads | Warehouse | T-SQL compatibility for migrated stored procs |
| SQL Server ingestion | Mirroring | Free compute, near real-time, minimal maintenance |
| PostgreSQL ingestion | Mirroring (Azure) / Data Pipelines (self-hosted) | Same rationale as SQL Server; fallback if not Azure-managed |
| MongoDB ingestion | Data Pipelines (Copy Activity) | Only connector available; supports upsert |
| Synapse DW migration | Migration Assistant + SQL Tran | Schema automation + T-SQL translation |
| S3 data access | Shortcuts | Zero data movement, Direct Lake compatible with Delta |
| Semantic model mode | Direct Lake | CU-efficient, no scheduled refresh |
| Starting SKU | F64 | Balance of CU headroom and cost |

## Appendix B: Key References

- [Fabric Lakehouse vs Warehouse Decision Guide](https://learn.microsoft.com/en-us/fabric/fundamentals/decision-guide-lakehouse-warehouse)
- [Fabric Mirroring Overview](https://learn.microsoft.com/en-us/fabric/database/mirrored-database/overview)
- [Mirroring SQL Server](https://learn.microsoft.com/en-us/fabric/database/mirrored-database/sql-server)
- [Mirroring Azure Database for PostgreSQL](https://learn.microsoft.com/en-us/fabric/database/mirrored-database/azure-database-postgresql)
- [Synapse → Fabric Migration Planning](https://learn.microsoft.com/en-us/fabric/data-warehouse/migration-synapse-dedicated-sql-pool-warehouse)
- [MongoDB Atlas Connector](https://learn.microsoft.com/en-us/fabric/data-factory/connector-mongodb-atlas-overview)
- [OneLake Shortcuts](https://learn.microsoft.com/en-us/fabric/onelake/onelake-shortcuts)
- [Fabric Adoption Roadmap](https://learn.microsoft.com/en-us/power-bi/guidance/fabric-adoption-roadmap)
