# Microsoft Fabric Cost Analysis & Capacity Planning

> **Project:** AtlMigration — Enterprise Data Platform Migration to Microsoft Fabric
> **Owner:** Brian Hitney
> **Author:** Avasarala (Documentation & Strategy Lead)
> **Date:** 2026-04-08
> **Status:** ACTIVE
> **Version:** 1.0

---

## 1. Fabric SKU Comparison Table

Microsoft Fabric uses Capacity Units (CUs) as a unified compute currency. All workloads — ingestion, transformation, warehousing, BI — draw from the same CU pool. SKUs are Azure F-series resources, billed per-second with a one-minute minimum.

### 1.1 Complete SKU Reference

| SKU | Capacity Units (CUs) | Power BI Equivalent | Approx. Monthly Cost (PAYG)¹ | Approx. Monthly Cost (1-Year Reserved)¹ | Savings (Reserved) | Free Mirroring Storage | Max Concurrent Operations² |
|-----|---------------------|--------------------|-----------------------------|----------------------------------------|--------------------|-----------------------|---------------------------|
| **F2** | 2 | — | ~$262/mo | ~$155/mo | ~41% | 2 TB | Very limited — dev/test only |
| **F4** | 4 | — | ~$524/mo | ~$310/mo | ~41% | 4 TB | Light workloads |
| **F8** | 8 | EM/A1 (1 v-core) | ~$1,048/mo | ~$620/mo | ~41% | 8 TB | Small team, light transforms |
| **F16** | 16 | EM2/A2 (2 v-cores) | ~$2,097/mo | ~$1,240/mo | ~41% | 16 TB | Small-medium workloads |
| **F32** | 32 | EM3/A3 (4 v-cores) | ~$4,194/mo | ~$2,480/mo | ~41% | 32 TB | Medium workloads |
| **F64** | 64 | P1/A4 (8 v-cores) | ~$8,387/mo | ~$4,960/mo | ~41% | 64 TB | **Production workloads. Free Power BI viewing for Free-license users.** |
| **Trial** | 64 | — (P1 equivalent) | **Free (60 days)** | N/A | 100% | None (no free mirroring) | Same as F64 but time-limited |
| **F128** | 128 | P2/A5 (16 v-cores) | ~$16,774/mo | ~$9,920/mo | ~41% | 128 TB | Large-scale migrations, heavy concurrency |
| **F256** | 256 | P3/A6 (32 v-cores) | ~$33,549/mo | ~$19,840/mo | ~41% | 256 TB | Enterprise-scale |
| **F512** | 512 | P4/A7 (64 v-cores) | ~$67,097/mo | ~$39,680/mo | ~41% | 512 TB | Very large enterprise |
| **F1024** | 1,024 | P5/A8 (128 v-cores) | ~$134,195/mo | ~$79,360/mo | ~41% | 1,024 TB | Massive scale |
| **F2048** | 2,048 | — (256 v-cores) | ~$268,390/mo | ~$158,720/mo | ~41% | 2,048 TB | Maximum scale |

> ¹ Pricing is approximate based on US East region, USD. Actual pricing varies by region and agreement. Check the [Azure Pricing Calculator](https://azure.microsoft.com/en-us/pricing/calculator/?service=microsoft-fabric) for current rates. Base rate is approximately ~$0.18/CU/hour for PAYG.
>
> ² Concurrent operations scale with CU count. Fabric uses a background/interactive job scheduler that queues work when CU demand exceeds capacity.

### 1.2 Key SKU Thresholds

| Threshold | SKU | Why It Matters |
|-----------|-----|---------------|
| **Minimum for Fabric workloads** | F2 | Smallest SKU. Can run all Fabric experiences. Dev/test only. |
| **Free Power BI viewing** | F64+ | Users with a Free license can view Power BI content (viewer role) at F64 and above. Below F64, every viewer needs a Pro or PPU license. |
| **Production recommendation** | F64 | Balance of compute, free mirroring storage (64 TB), and licensing simplification. |
| **Heavy migration loads** | F128+ | Required if sustained CU utilization exceeds 80% on F64 during concurrent migrations + queries. |

### 1.3 Additional Storage Costs

| Storage Type | Cost | Notes |
|-------------|------|-------|
| **OneLake storage** | ~$0.023/GB/month | Standard storage for all Lakehouse and Warehouse data |
| **OneLake BCDR storage** | ~$0.046/GB/month | Business continuity/disaster recovery replication |
| **OneLake cache** | ~$0.095/GB/month | KQL cache and Data Activator retention |
| **SQL storage** | ~$0.023/GB/month | Fabric Warehouse native storage |
| **SQL backup storage** | ~$0.05/GB/month | Point-in-time restore backups |
| **Mirroring storage** | **Free** up to SKU limit | 1 TB free per CU. Exceeding limit or pausing capacity triggers OneLake rates. |

### 1.4 Per-User License Costs

| License | Approx. Monthly Cost | Needed For |
|---------|---------------------|-----------|
| **Fabric Free** | $0 | Auto-assigned. Can create non-PBI Fabric items on F capacity. |
| **Power BI Pro** | ~$10/user/month | Publishing and sharing Power BI content. Required for all PBI publishers. |
| **Power BI Premium Per User (PPU)** | ~$20/user/month | PBI Premium features per user. Does NOT provision Fabric capacity. |

**Licensing strategy for this project:** F64+ capacity with Pro licenses for content creators. Free-license viewers can access reports on F64+.

---

## 2. Capacity Planning — Understanding CU Consumption

### 2.1 What Consumes CUs

Every Fabric workload draws from the same CU pool. Consumption is measured in CU-seconds.

| Workload | CU Consumption Pattern | Typical Impact |
|----------|----------------------|----------------|
| **Mirroring (replication)** | 🟢 **Free** — Background replication compute is not charged | Zero CU impact during active replication |
| **Data Pipeline (Copy Activity)** | 🟡 Moderate — CU per copy execution. Scales with data volume and parallelism | 0.5–4 CU-hours per GB depending on source/complexity |
| **Dataflow Gen2 (Power Query)** | 🟡 Moderate — CU per refresh. Power Query engine overhead | Slightly higher than Copy Activity for equivalent data |
| **Spark Notebooks** | 🟠 Moderate–High — CU based on cluster size × duration. Starter pools reduce startup cost | Biggest CU consumer during Silver/Gold transforms |
| **Warehouse Queries (T-SQL)** | 🟡 Moderate — CU per query. Auto-scales with query complexity | Varies widely by query. Complex joins consume more. |
| **SQL Analytics Endpoint** | 🟡 Moderate — Same engine as Warehouse | Read-only queries on Lakehouse tables |
| **Direct Lake Semantic Models** | 🟢 Low — Reads Delta directly. No import refresh CU cost. | Dramatically lower than Import mode |
| **Import Semantic Models** | 🟠 Moderate–High — CU consumed on each scheduled refresh | Avoid if possible — use Direct Lake instead |
| **Power BI Report Rendering** | 🟢 Low — Standard rendering cost | Scales with concurrent users and query complexity |
| **Shortcut Reads (S3)** | 🟡 Moderate — OneLake API calls consume CU | Plus AWS egress costs on the S3 side |

### 2.2 Smoothing and Bursting

Fabric uses **CU smoothing** to flatten short bursts of compute usage over time:

- **Interactive operations** (queries, report renders): Smoothed over a **5-minute window**. A 30-second spike of 128 CU on an F64 capacity is averaged to 12.8 CU/s over 5 minutes — well within capacity.
- **Background operations** (pipeline runs, Spark jobs, dataflow refreshes): Smoothed over a **24-hour window**. A 1-hour Spark job consuming 256 CU on an F64 is spread across 24 hours — effectively 10.7 CU/s.

**What this means for our migration:**
- Short, intensive Spark notebook runs (Silver/Gold transforms) are smoothed over 24 hours. An F64 can handle occasional heavy Spark jobs without throttling.
- Concurrent report queries are smoothed over 5 minutes. Standard BI workloads rarely cause issues on F64.
- **Sustained heavy workloads** (multiple large Spark jobs running continuously) will accumulate debt. If cumulative consumption exceeds capacity over the smoothing window, throttling begins.

### 2.3 Throttling Behavior

When CU consumption exceeds capacity:

| Throttle Level | Trigger | Effect |
|----------------|---------|--------|
| **None** | < 100% cumulative utilization | Normal operation |
| **Delayed** | > 100% for background jobs | Background jobs (Spark, pipelines) are delayed 10–20 minutes |
| **Rejected** | > 120% sustained | Background job submissions are rejected. Interactive queries still run but may slow. |

**Mitigation options:**
1. **Scale up** the SKU temporarily (e.g., F64 → F128)
2. **Fabric capacity overage** — opt-in to pay for excess consumption instead of throttling
3. **Pause and resume** — pause capacity during non-working hours to reset CU balance
4. **Autoscale Billing for Spark** — opt-in for Spark workloads to use separate, dedicated CU billing

### 2.4 Monitoring Consumption

Use the **Microsoft Fabric Capacity Metrics app** (available from AppSource):

| Metric | What to Watch | Action Threshold |
|--------|--------------|-----------------|
| **CU utilization %** | Percentage of capacity consumed over time | Investigate if sustained > 70%, scale up at > 85% |
| **Throttling events** | Count and duration of throttle episodes | Any throttling requires immediate investigation |
| **Background vs interactive** | Split of CU between background jobs and interactive queries | If background dominates, schedule jobs to off-peak |
| **Per-item consumption** | CU consumed by each Fabric item (pipeline, notebook, warehouse) | Identify top consumers for optimization |

**Setup in Week 1:** Install the Capacity Metrics app, configure alerts at 70% and 85% utilization thresholds.

---

## 3. Cost Impact of Key Architectural Decisions

### 3.1 Mirroring vs ETL — CU Consumption

| Scenario | Mirroring | ETL (Data Pipelines) | CU Difference |
|----------|-----------|---------------------|--------------|
| **10 GB database, ongoing sync** | Free (replication compute + storage up to limit) | ~5 CU-hours initial load + ~0.5 CU-hours per incremental run | Mirroring saves ~100% on compute |
| **100 GB database, ongoing sync** | Free | ~50 CU-hours initial + ~5 CU-hours per incremental | Mirroring saves hundreds of CU-hours/month |
| **Transformation on ingest** | Not possible — must transform in separate Spark step | Included in pipeline | Wash — transform CU consumed either way |
| **Column subset extraction** | Not possible — mirrors full tables | Selective columns only | ETL saves storage but costs CU for each run |

**Our decision:** Mirroring-first (SQL Server, PostgreSQL). ETL only for MongoDB (no mirroring support) and Synapse (different migration path). **Estimated savings: hundreds of CU-hours per month** vs an all-ETL approach.

### 3.2 Lakehouse vs Warehouse — Cost Implications

| Factor | Lakehouse | Warehouse | Impact |
|--------|-----------|-----------|--------|
| **Storage format** | Delta (open format) | Delta (open format) | Same storage cost |
| **Query engine** | Spark (heavy transforms), SQL analytics endpoint (ad-hoc reads) | T-SQL engine | Warehouse T-SQL queries may consume slightly different CU patterns |
| **DML operations** | Via Spark (CU for Spark cluster) | Via T-SQL (CU for query engine) | Spark clusters consume CU for duration; T-SQL is per-query |
| **Direct Lake support** | ✅ Yes | ✅ Yes | No cost difference |
| **Best for** | Ingestion, transformation, semi-structured data | T-SQL workloads, stored procedures, multi-table transactions | Use each where it fits |

**Our decision:** Lakehouse-first. Warehouse only for Synapse T-SQL workloads at Gold. This minimizes Warehouse query CU by restricting T-SQL to one specific workstream.

### 3.3 Direct Lake vs Import vs DirectQuery — Consumption Patterns

| Mode | CU on Refresh | CU on Query | Memory Usage | Best For |
|------|--------------|-------------|-------------|----------|
| **Direct Lake** | 🟢 None (no refresh) | 🟢 Low (reads Delta in place) | Loads columns on demand | **Our default.** Best for Gold layer tables. |
| **Import** | 🟠 High (full data load into memory) | 🟢 Low (cached in memory) | High — full dataset in memory | Legacy pattern. Avoid unless Direct Lake has limitations. |
| **DirectQuery** | 🟢 None | 🟠 High (query hits source every time) | Low — no caching | Real-time requirements only. High query CU cost. |
| **Composite** | Mixed | Mixed | Mixed | When some tables need Import, others Direct Lake or DirectQuery |

**Cost analysis for Direct Lake:**
- **No scheduled refresh cost.** Import mode on a 1 GB semantic model refreshing 8×/day consumes significant CU. Direct Lake eliminates this entirely.
- **Lower memory pressure.** Direct Lake loads columns on demand rather than caching entire datasets.
- **Trade-off:** First query after data change may be slightly slower (column cache miss). Subsequent queries are fast.

**Estimated monthly CU savings (Direct Lake vs Import for our workload):**

| Metric | Import Mode (8 refreshes/day) | Direct Lake | Monthly Savings |
|--------|------------------------------|-------------|----------------|
| Refresh CU per semantic model (1 GB) | ~4 CU-hours/day | 0 | ~120 CU-hours/month |
| If we have 5 semantic models | ~20 CU-hours/day | 0 | ~600 CU-hours/month |
| Query CU | Low (cached) | Low (column cache) | Comparable |

**Direct Lake is the clear winner for this project.** We save an estimated 600 CU-hours/month on refresh alone.

### 3.4 Shortcuts vs Data Copy — Storage and Compute Tradeoffs

| Factor | Shortcuts (S3) | Data Copy (Pipeline to Lakehouse) |
|--------|---------------|----------------------------------|
| **Data movement** | None — reads in place | Full copy to OneLake |
| **OneLake storage cost** | None (data stays in S3) | ~$0.023/GB/month |
| **Compute cost on read** | CU for OneLake API + AWS egress | CU for copy activity (one-time) + standard query CU |
| **Latency** | Depends on S3 API + network | Local reads — fast |
| **Freshness** | Always current (reads S3 live) | As fresh as last pipeline run |
| **Shortcut caching** | Reduces repeat reads. ~$0.095/GB/month for cache | N/A |

**Our decision:** Shortcuts for S3 data with 7-day cache retention. Best for data we query occasionally. If any S3 dataset is queried heavily (>10× per day), materialize it to OneLake to avoid egress costs.

**Estimated S3 cost impact (example: 50 GB of S3 data):**

| Approach | Monthly Cost |
|----------|-------------|
| Shortcut (no cache, 5 reads/day) | ~$15–45 AWS egress + CU for reads |
| Shortcut (7-day cache) | ~$4.75 cache + reduced egress (~$5–10) |
| Full copy to OneLake | ~$1.15 OneLake storage + one-time CU for copy |

---

## 4. Recommended SKU and Budget

### 4.1 SKU Recommendation: F64

**We recommend starting with F64.** Here's the justification:

| Factor | Requirement | F64 Capability | Assessment |
|--------|------------|----------------|------------|
| **CU headroom** | Multiple concurrent migrations + transforms + queries | 64 CU with 24-hour smoothing for background jobs | ✅ Sufficient for migration phase with scheduling |
| **Free mirroring storage** | SQL Server + PostgreSQL databases (est. 10–50 TB) | 64 TB free | ✅ Ample headroom |
| **Power BI licensing** | Report viewers with Free licenses | F64+ enables Free viewer access | ✅ Meets threshold |
| **Spark transforms** | Silver/Gold layer notebooks | Starter pools + smoothing handle intermittent jobs | ✅ With scheduling |
| **T-SQL workloads** | Synapse migrated stored procs | Auto-scaling query engine | ✅ Sufficient |
| **Cost** | Budget-conscious | ~$8,387/mo PAYG or ~$4,960/mo reserved | ✅ Reasonable for scope |

### 4.2 When to Scale Up

Monitor these triggers during the migration:

| Trigger | Action |
|---------|--------|
| CU utilization > 80% sustained for 4+ hours | Scale to F128 temporarily |
| Throttling events > 3 per day | Scale to F128 or enable capacity overage |
| Multiple Spark jobs competing for resources | Consider Autoscale Billing for Spark |
| Post-migration steady state < 30% utilization | Scale down to F32 (if PBI Free viewer access not required) or keep F64 |

### 4.3 Projected Monthly Cost

| Line Item | Monthly Cost (PAYG) | Monthly Cost (1-Year Reserved) |
|-----------|--------------------|-----------------------------|
| **F64 capacity** | ~$8,387 | ~$4,960 |
| **OneLake storage (est. 100 GB Gold + Silver)** | ~$2.30 | ~$2.30 |
| **Mirroring storage** | Free (within 64 TB) | Free (within 64 TB) |
| **SQL backup storage (est. 50 GB)** | ~$2.50 | ~$2.50 |
| **Power BI Pro licenses (5 creators)** | ~$50 | ~$50 |
| **AWS S3 egress (est. with caching)** | ~$10–30 | ~$10–30 |
| | | |
| **Total (PAYG)** | **~$8,452–$8,472/mo** | |
| **Total (1-Year Reserved)** | | **~$5,025–$5,045/mo** |

> **Annual cost:** ~$101K (PAYG) or ~$60K (reserved). The 1-year reservation saves approximately $41K/year.

### 4.4 Cost Optimization Strategies

| Strategy | Estimated Monthly Savings | Implementation |
|----------|--------------------------|---------------|
| **1-Year Reserved capacity** | ~$3,400/mo (~41%) | Commit after validating workload during first month of PAYG |
| **Pause during non-work hours** | ~$2,800/mo (assuming 12 hrs/day, 5 days/week) | Azure Automation runbook to pause/resume. ⚠️ Caution: Mirroring pauses too. |
| **Direct Lake instead of Import** | ~100–600 CU-hours/mo (avoids throttle risk) | Default for all semantic models |
| **Mirroring instead of ETL** | Hundreds of CU-hours/mo | Already our strategy for SQL/PostgreSQL |
| **Spark Starter Pools** | Avoids custom cluster startup CU | Use for all notebook workloads |
| **Off-peak scheduling** | Reduces peak CU contention | Schedule Silver→Gold transforms for evenings/weekends |

### 4.5 Pause/Resume Strategy

Fabric F SKUs can be paused and resumed via the Azure portal or REST API. When paused, billing stops (per-second, minimum 1 minute).

**Important considerations for pausing:**
- **Mirroring stops** when capacity is paused. Replication resumes when capacity is resumed, but there will be a sync gap.
- **Free mirroring storage** is billed at standard OneLake rates while capacity is paused.
- **Cumulative CU overages** are settled when capacity is paused.
- **Workspaces become read-only** during pause — no queries, no pipelines, no reports.

**Recommendation for this migration:**
- **During the 4-week migration:** Do NOT pause. Mirroring needs continuous operation. Transforms run at various times.
- **Post-migration (steady state):** Evaluate pause/resume schedule. If mirroring latency gaps are acceptable (e.g., overnight), implement an Azure Automation runbook for nightly pause (10 PM–6 AM) and weekend pause.

### 4.6 Trial Capacity Option

Fabric offers a **60-day free trial** with F64-equivalent capacity (64 CUs). However:

| Factor | Trial | Paid F64 |
|--------|-------|----------|
| Duration | 60 days | Unlimited |
| CUs | 64 | 64 |
| Free mirroring storage | ❌ No | ✅ 64 TB |
| Pause/Resume | ❌ No | ✅ Yes |
| Scale up/down | ❌ No | ✅ Yes |
| Capacity overage | ❌ No | ✅ Yes |
| Production use | ⚠️ Not recommended | ✅ Yes |

**Recommendation:** Use the trial for initial exploration and prototyping only. Provision paid F64 on Day 1 for the actual migration.

---

## 5. Cost Impact Summary — Migration Scenarios

### Scenario A: Current Architecture (Status Quo)

| Component | Est. Monthly Cost |
|-----------|------------------|
| SQL Server on-prem (licensing, hardware) | $2,000–$5,000 (varies) |
| Azure Synapse dedicated SQL pool (DW100c–DW500c) | $1,200–$6,000 |
| PostgreSQL (Azure managed) | $200–$800 |
| MongoDB (Atlas or self-hosted) | $200–$1,000 |
| Power BI Premium (P1 or PPU) | $4,995 (P1) or $20/user |
| AWS S3 storage | $50–$200 |
| ETL tooling / SSIS | $500–$2,000 |
| **Estimated total** | **$9,145–$20,000/mo** |

### Scenario B: Microsoft Fabric (Post-Migration)

| Component | Est. Monthly Cost |
|-----------|------------------|
| Fabric F64 (reserved) | ~$4,960 |
| OneLake storage | ~$5 |
| Power BI Pro (5 users) | ~$50 |
| AWS S3 egress | ~$20 |
| On-prem SQL Server (kept for source of truth during transition) | $2,000–$5,000 |
| **Estimated total (transition period)** | **~$7,035–$10,035/mo** |
| **Estimated total (post-decommission)** | **~$5,035/mo** |

### Cost Trajectory

```
Month 1-2:  $12K-15K/mo  (running both old and new systems)
Month 3:    $8K-10K/mo   (decommission Synapse, reduce on-prem)
Month 6+:   $5K-6K/mo    (fully on Fabric, on-prem decommissioned)
Annual:     $60K-72K     (vs $110K-240K current state)
```

**Projected annual savings: $50K–$170K** depending on current infrastructure costs.

---

## 6. References

- [Microsoft Fabric Licensing](https://learn.microsoft.com/en-us/fabric/enterprise/licenses)
- [Buy a Microsoft Fabric Subscription](https://learn.microsoft.com/en-us/fabric/enterprise/buy-subscription)
- [Azure Pricing — Microsoft Fabric](https://azure.microsoft.com/en-us/pricing/details/microsoft-fabric/)
- [Fabric Capacity Metrics App](https://learn.microsoft.com/en-us/fabric/enterprise/metrics-app)
- [Fabric Capacity Reservations](https://learn.microsoft.com/en-us/azure/cost-management-billing/reservations/fabric-capacity)
- [Pause and Resume Capacity](https://learn.microsoft.com/en-us/fabric/enterprise/pause-resume)
- [Fabric Operations and CU Consumption](https://learn.microsoft.com/en-us/fabric/enterprise/fabric-operations)
- [Fabric Adoption Roadmap](https://learn.microsoft.com/en-us/power-bi/guidance/fabric-adoption-roadmap)
- [Fabric Capacity Estimator](https://www.microsoft.com/en-us/microsoft-fabric/capacity-estimator)

---

*Prepared by Avasarala — Documentation & Strategy Lead*
*"Every dollar spent without a documented rationale is a dollar wasted twice — once when you spend it, and again when you have to explain it."*
