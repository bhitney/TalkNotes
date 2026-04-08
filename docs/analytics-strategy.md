# Analytics & BI Strategy: Microsoft Fabric Migration

> **Project:** AtlMigration — Enterprise Data Platform Migration to Microsoft Fabric
> **Author:** Alex (BI & Analytics Engineer)
> **Date:** 2026-04-08
> **Status:** ACTIVE
> **Timeline:** 4 weeks (semantic models depend on Gold layer availability)
> **Owner:** Brian Hitney

---

## Executive Summary

This document defines the analytics and BI strategy for the Fabric migration. We're building a modern analytics layer on top of the Gold medallion tier using Direct Lake semantic models, Power BI reports, Excel integration, and NLP-based sentiment analysis. Every design decision serves a single goal: **get the right data to the right people in the format they actually use.**

Five pillars:

1. **Direct Lake semantic models** on Gold layer Delta tables — sub-second query performance without Import mode refresh overhead.
2. **Power BI reports** — operational dashboards, executive summaries, and self-service exploration.
3. **Excel integration** — because half your users live in Excel, and that's fine.
4. **Sentiment analysis** — favorability scoring and keyword extraction on text columns from source databases.
5. **Fabric IQ ontologies** — business process semantics mapped to the semantic layer.

---

## 1. Direct Lake Semantic Models

### 1.1 What is Direct Lake?

Direct Lake is a storage mode for Power BI semantic models in Microsoft Fabric. It reads Delta/Parquet files **directly from OneLake** without importing data into the Power BI engine's in-memory VertiPaq cache and without sending queries back to the source (DirectQuery). It combines the performance of Import mode with the freshness of DirectQuery.

```
┌──────────────────┐         ┌──────────────────┐         ┌──────────────────┐
│   Gold Layer     │         │  Direct Lake     │         │   Power BI       │
│   (OneLake)      │────────▶│  Semantic Model  │────────▶│   Reports        │
│   Delta Tables   │  reads  │  (VertiPaq-on-   │  serves │   Excel          │
│                  │ parquet │   demand)        │ queries │   Third-party    │
└──────────────────┘         └──────────────────┘         └──────────────────┘
```

**How it works:**

1. The Gold layer stores curated, business-ready Delta tables in OneLake (Parquet files under the hood).
2. The semantic model is defined with tables pointing to these Delta tables.
3. When a report queries the model, Direct Lake reads the Parquet files directly into the VertiPaq engine — no ETL, no copy, no scheduled refresh.
4. Column segments are loaded on demand and cached in memory. Subsequent queries hit the cache.
5. When the underlying Delta table is updated (e.g., a Spark notebook writes new data), the model's "framing" is updated to point to the latest Parquet files. The cache is transparently refreshed.

### 1.2 Why Direct Lake Is the Right Choice

| Factor | Import Mode | DirectQuery | Direct Lake |
|--------|-------------|-------------|-------------|
| **Data freshness** | Stale between refreshes (schedule-based) | Real-time | Near real-time (on next frame update) |
| **Query performance** | ⚡ Fast (in-memory VertiPaq) | 🐌 Depends on source engine | ⚡ Fast (VertiPaq-on-demand) |
| **Memory usage** | High (full dataset in memory) | None | Moderate (on-demand column loading) |
| **Refresh cost** | 🟠 CU-intensive scheduled refresh | None | 🟢 Minimal (framing only, no data copy) |
| **Data duplication** | Yes (copy into PBIX/PBIT) | No | No |
| **Max dataset size** | Limited by memory/SKU | Unlimited | Unlimited (reads from OneLake) |
| **DAX support** | Full | Limited (no complex measures) | Full |
| **Composite model support** | Yes | Yes | Yes (since Fabric GA) |

**For this project, Direct Lake wins because:**

- **Gold layer is already in Delta format** — the architecture stores curated star schemas as Delta tables in OneLake. No additional data movement needed.
- **No refresh scheduling overhead** — Import mode on an F64 capacity would consume significant CUs for scheduled refreshes. Direct Lake eliminates this.
- **Near real-time analytics** — SQL Server mirroring delivers ~15-second latency to Bronze. After Silver/Gold transforms, Direct Lake picks up changes on the next framing cycle.
- **Full DAX support** — Unlike DirectQuery, Direct Lake supports the full DAX language, including complex measures, time intelligence, and calculated tables.
- **Simpler architecture** — One less ETL step. No "refresh the model" pipeline to maintain.

### 1.3 Requirements for Direct Lake

Direct Lake has specific requirements that the architecture must satisfy:

| Requirement | Our Status | Notes |
|-------------|-----------|-------|
| Data must be in OneLake | ✅ Met | Gold layer Lakehouse stores Delta tables in OneLake |
| Delta/Parquet format | ✅ Met | All medallion layers use Delta format |
| Fabric Lakehouse or Warehouse | ✅ Met | Gold = Lakehouse (primary) + Warehouse (Synapse workloads) |
| F64+ SKU for production | ✅ Met | Project uses F64 capacity |
| Tables must be in same workspace (or use shortcuts) | ⚠️ Plan for it | Semantic model and Gold Lakehouse should share a workspace |
| No calculated tables in the model | ⚠️ Constraint | Use Gold layer transforms instead of DAX calculated tables |
| No row-level calculations | ⚠️ Constraint | Computed columns must be in Delta, not DAX |

### 1.4 Direct Lake Limitations and Fallback Behavior

**Critical behavior: Direct Lake can fall back to DirectQuery mode.** When this happens, queries go through the SQL analytics endpoint instead of reading Parquet directly, and performance degrades significantly.

**Fallback triggers:**

| Trigger | Description | Mitigation |
|---------|-------------|------------|
| **Unsupported DAX** | Certain DAX functions force DirectQuery fallback | Test all measures; avoid `USERELATIONSHIP` on large tables if possible |
| **Row count exceeds guardrails** | Tables exceeding row-count limits per SKU (F64: ~1.5B rows per table) | Partition large tables; aggregate where possible |
| **Column count exceeds guardrails** | Too many columns in a single table (F64: ~1,500 columns) | Design narrow tables in the Gold layer |
| **Cardinality limits** | Unique values per column exceeds limits | Monitor high-cardinality columns (e.g., transaction IDs) |
| **RLS with certain patterns** | Some RLS configurations trigger fallback | Test RLS rules against Direct Lake compatibility |
| **Composite model with Import/DQ tables** | Adding Import or DirectQuery tables to a Direct Lake model forces mixed mode | Minimize composite model usage |

**SKU-specific guardrails for F64:**

| Guardrail | F64 Limit |
|-----------|-----------|
| Max rows per table | ~1.5 billion |
| Max total rows (all tables) | ~3 billion |
| Max columns per table | ~1,500 |
| Max tables per model | Workspace-level limit |
| Max Parquet file size per column segment | 256 MB |

**How to detect fallback:** Monitor the `DirectLakeFallback` event in Fabric Capacity Metrics. If a model falls back, the Capacity Metrics app shows which queries triggered DirectQuery mode.

### 1.5 Performance Optimization

#### V-Order Optimization

V-Order is a Fabric-specific write-time optimization that reorders data within Parquet files for faster Direct Lake reads. It's the single most impactful performance lever.

- **Enable V-Order on all Gold layer writes.** Spark notebooks should use `.option("vorder", "true")` when writing Delta tables.
- **Impact:** 10–50% faster query performance on Direct Lake reads depending on data characteristics.
- **Cost:** Slightly slower write times (5–15% write overhead). Worth it for read-heavy analytics workloads.

```python
# Gold layer Spark notebook — V-Order enabled
df.write \
    .format("delta") \
    .option("vorder", "true") \
    .mode("overwrite") \
    .save("Tables/gold_fact_sales")
```

#### Column Statistics

Delta Lake maintains column-level min/max statistics that enable data skipping during queries. Direct Lake leverages these statistics.

- **Ensure statistics are maintained** on all Gold layer Delta tables.
- **Set statistics columns** to include all commonly filtered columns (dates, categories, keys).
- **Default:** Delta maintains stats on the first 32 columns. Reorder columns so filter columns come first, or explicitly set `delta.dataSkippingNumIndexedCols`.

#### Partitioning Strategy

Partitioning affects both Direct Lake read performance and Spark write performance.

| Table Type | Partitioning Strategy | Rationale |
|------------|----------------------|-----------|
| **Fact tables** (large) | Partition by date (year/month) | Time-based filtering is the most common query pattern |
| **Dimension tables** (small) | No partitioning | Overhead isn't worth it for small tables |
| **Aggregate tables** | No partitioning | Already small; partitioning adds file overhead |

**Anti-patterns to avoid:**
- ❌ Over-partitioning (e.g., partition by day on a 2-year table = 730 partitions = too many small files)
- ❌ Partitioning by high-cardinality columns (e.g., customer ID)
- ❌ Partitioning dimension tables

#### File Compaction

Delta tables accumulate small files over time (especially with streaming/incremental writes). Small files kill Direct Lake performance.

- **Run OPTIMIZE regularly** on Gold layer tables to compact small files.
- **Enable auto-compaction** where supported: `delta.autoOptimize.autoCompact = true`.

```sql
-- Run in Spark SQL or Lakehouse SQL analytics endpoint
OPTIMIZE gold_lakehouse.fact_sales;
```

### 1.6 Recommended Semantic Model Structure

Based on the medallion architecture and source systems (SQL Server, Synapse DW, PostgreSQL, MongoDB), the semantic model should follow a **star schema** design aligned with the Gold layer.

```
┌──────────────────────────────────────────────────────────┐
│                   SEMANTIC MODEL                          │
│                   (Direct Lake)                           │
│                                                          │
│   ┌──────────┐   ┌──────────┐   ┌──────────┐            │
│   │ dim_date  │   │dim_region│   │dim_entity│            │
│   └────┬─────┘   └────┬─────┘   └────┬─────┘            │
│        │              │              │                    │
│        └──────────────┼──────────────┘                    │
│                       │                                   │
│              ┌────────┴────────┐                          │
│              │   fact_metrics  │                          │
│              │   fact_surveys  │                          │
│              │   fact_feedback │                          │
│              └─────────────────┘                          │
│                       │                                   │
│   ┌──────────┐   ┌────┴─────┐   ┌──────────┐            │
│   │dim_source│   │dim_status│   │dim_categry│            │
│   └──────────┘   └──────────┘   └──────────┘             │
│                                                          │
│   Measures:                                              │
│   - [Total Records]       = COUNTROWS(fact_metrics)      │
│   - [Avg Favorability]    = AVERAGE(fact_surveys[score]) │
│   - [Keyword Count]       = SUM(fact_feedback[kw_count]) │
│   - [YoY Growth %]        = Time intelligence calc       │
│   - [Rolling 30-Day Avg]  = CALCULATE + DATESINPERIOD    │
└──────────────────────────────────────────────────────────┘
```

**Design principles:**

1. **One semantic model per business domain** — Don't create a single monolithic model. Separate by subject area (e.g., operational metrics, survey/feedback, financial).
2. **All computed columns in the Gold layer, not DAX** — Direct Lake doesn't support DAX calculated columns efficiently. Pre-compute in Spark.
3. **Measures in the semantic model** — DAX measures are fine and encouraged. They execute at query time and don't affect Direct Lake compatibility.
4. **Relationships defined in the model** — Star schema relationships (1-to-many from dimension to fact) are defined in the semantic model, not in the Delta tables.
5. **Avoid bi-directional relationships** — They cause ambiguity and performance issues. Use single-direction (dimension → fact).

---

## 2. Report Strategy

### 2.1 Approach

All Power BI reports will be built on the Direct Lake semantic models. No reports should connect directly to the Lakehouse SQL analytics endpoint or Warehouse — the semantic model is the single source of truth for business metrics.

**Why this matters:**
- Consistent metric definitions across all reports (DAX measures defined once).
- Row-level security applied at the model level, inherited by all reports.
- Direct Lake performance benefits flow to every consumer.

### 2.2 Report Types

| Report Type | Audience | Purpose | Refresh Expectation | Complexity |
|-------------|----------|---------|---------------------|------------|
| **Operational Dashboards** | Analysts, managers | Real-time status of data pipelines, mirroring health, data quality metrics | Near real-time (Direct Lake) | Medium |
| **Executive Summaries** | Brian Hitney, leadership | High-level KPIs, trends, sentiment summaries | Daily / on-demand | Low-Medium |
| **Self-Service Exploration** | Power users, analysts | Ad-hoc analysis with slicers, drill-through, export to Excel | On-demand | High (interactive) |
| **Embedded Reports** | External stakeholders (if needed) | Specific views shared via Power BI Embedded or Publish to Web | Depends on audience | Medium |
| **Paginated Reports** | Finance, compliance | Pixel-perfect reports for print/export (PDF, Excel) | On-demand | Low |

### 2.3 Report Design Standards

- **Every report answers a specific business question.** If it doesn't have a clear question, it doesn't get built.
- **Mobile-first layout** for executive summaries (Power BI mobile layout).
- **Consistent color palette** and branding across all reports.
- **Bookmarks** for guided navigation (e.g., "Show me pipeline health" → "Show me data quality").
- **Tooltips** with contextual detail (hover over a metric → see the breakdown).
- **Drill-through pages** for deep dives (click a region → see region-specific detail).

### 2.4 Composite Models

Composite models combine Direct Lake tables with Import or DirectQuery tables in a single semantic model. Use sparingly.

**When to use composite models:**

| Scenario | Example | Approach |
|----------|---------|----------|
| **Budget/target data in Excel** | Budget figures maintained in Excel by finance | Add as Import table to the Direct Lake model |
| **External benchmarks** | Industry benchmarks from a third-party API | Add as Import table (small dataset) |
| **Cross-workspace data** | Reference data from another Fabric workspace | Use OneLake shortcut + Direct Lake (preferred) or DirectQuery |
| **Real-time streaming** | Live metrics from Eventstream | Out of scope (Phase 2) |

**Warning:** Adding Import or DirectQuery tables to a Direct Lake model creates a composite model and may trigger partial fallback. Test thoroughly.

### 2.5 Deployment Strategy

```
┌────────────────────────────────────────────────────────────────────┐
│                    WORKSPACE ARCHITECTURE                           │
│                                                                    │
│   ┌──────────────────┐    ┌──────────────────┐                     │
│   │  DEV Workspace    │    │  PROD Workspace   │                    │
│   │                  │    │                   │                    │
│   │  Gold Lakehouse  │    │  Gold Lakehouse   │                    │
│   │  Semantic Models │───▶│  Semantic Models  │                    │
│   │  Reports (draft) │DP  │  Reports (live)   │                    │
│   │                  │    │                   │                    │
│   └──────────────────┘    └──────┬────────────┘                    │
│                                  │                                  │
│                                  ▼                                  │
│                          ┌───────────────┐                          │
│                          │  Power BI App  │                          │
│                          │  (published)   │                          │
│                          │               │                          │
│                          │  - Dashboards │                          │
│                          │  - Reports    │                          │
│                          │  - Audiences  │                          │
│                          └───────────────┘                          │
│                                                                    │
│   DP = Deployment Pipeline (DEV → TEST → PROD)                     │
└────────────────────────────────────────────────────────────────────┘
```

**Key decisions:**

1. **Deployment Pipelines** — Use Fabric Deployment Pipelines to promote semantic models and reports from DEV → PROD. This gives us version control and rollback.
2. **Power BI Apps** — Publish curated collections of reports as Power BI Apps. Users access the App, not the workspace directly.
3. **Row-Level Security (RLS)** — Define RLS roles in the semantic model using DAX filters. Test with "View As" before publishing.
4. **Object-Level Security (OLS)** — Hide sensitive columns (e.g., PII) from unauthorized roles. Configure in XMLA endpoint or Tabular Editor.
5. **Workspace roles** — Admin (Holden, Alex), Member (report developers), Contributor (analysts), Viewer (everyone else).

### 2.6 Row-Level Security Design

RLS ensures users only see data they're authorized to access.

```dax
-- Example: Region-based RLS
[Region] = USERPRINCIPALNAME()
-- Maps the logged-in user's email to their assigned region

-- Example: Department-based RLS using a security table
FILTER(
    dim_security,
    dim_security[UserEmail] = USERPRINCIPALNAME()
    && dim_security[Department] = RELATED(dim_department[Department])
)
```

**Implementation approach:**

1. Create a `dim_security` table in the Gold layer mapping users to their data access scope.
2. Define RLS roles in the semantic model using DAX filters on the security table.
3. Test with "View as Role" in Power BI Desktop and Service.
4. Assign Azure AD security groups to RLS roles (not individual users).

---

## 3. Excel Integration Options

Many users live in Excel. We need to meet them where they are. Here's a comprehensive evaluation of every integration path between Excel and Fabric.

### 3.1 Option Comparison Matrix

| Option | Direction | Data Freshness | User Skill Required | Setup Complexity | Best For |
|--------|-----------|---------------|--------------------|-----------------| ---------|
| **Analyze in Excel** | Fabric → Excel | Live (connected) | Low | Low | PivotTable/PivotChart analysis on semantic models |
| **Power BI Publisher for Excel** | Excel → Power BI | Snapshot | Medium | Low | Publishing Excel ranges to Power BI dashboards |
| **Dataflows Gen2** | Excel → Fabric | Scheduled | Medium | Medium | Loading Excel files as data sources into Fabric |
| **OneLake File API** | Bidirectional | Manual/Scheduled | High | Medium | Storing Excel files in OneLake for Spark processing |
| **Power Query in Excel** | Fabric → Excel | On-demand refresh | Medium | Low | Direct queries to Lakehouse/Warehouse from Excel |
| **M365 Integration** | Bidirectional | Live/Near-real-time | Low | Low (when GA) | Native Excel-Fabric connectivity via Microsoft 365 |

### 3.2 Detailed Evaluation

#### Option 1: Analyze in Excel ⭐ PRIMARY RECOMMENDATION

**What it is:** Users connect to a published semantic model from Excel and build PivotTables, PivotCharts, and formulas that query the model live.

**How it works:**
1. User opens Power BI Service → navigates to the semantic model → clicks "Analyze in Excel."
2. Excel opens with a live connection to the semantic model.
3. User builds PivotTables using the model's tables, columns, and DAX measures.
4. Data refreshes when the user refreshes the PivotTable (or on workbook open).

**Pros:**
- ✅ Zero data duplication — queries run live against the Direct Lake model.
- ✅ Inherits RLS — users only see their authorized data.
- ✅ Familiar UX — PivotTables are second nature for Excel power users.
- ✅ DAX measures available — all business logic from the semantic model is accessible.
- ✅ No additional licensing — works with Pro license + F64 capacity.

**Cons:**
- ❌ Requires network connectivity (no offline use).
- ❌ Large PivotTable queries can be slow if the model is very large.
- ❌ Users can't modify the data model from Excel.

**Recommendation:** **Implement in Week 3.** This is the primary Excel integration path. Every semantic model should be Analyze-in-Excel ready on day one.

---

#### Option 2: Power BI Publisher for Excel

**What it is:** An Excel add-in that lets users pin Excel ranges (tables, charts, PivotTables) to Power BI dashboards.

**How it works:**
1. User selects a range in Excel → clicks "Pin to Power BI."
2. The range appears as a tile on a Power BI dashboard.
3. The tile shows a snapshot (not live) — it updates when the user re-pins.

**Pros:**
- ✅ Users can share Excel-based analysis via Power BI dashboards.
- ✅ Simple workflow — no migration of Excel logic required.

**Cons:**
- ❌ Snapshot-based — not live. Requires manual re-pin for updates.
- ❌ Limited interactivity — it's an image tile, not a live visual.
- ❌ Being deprecated in favor of native Excel-Power BI integration.

**Recommendation:** **Do not invest.** This feature is being phased out. Use Analyze in Excel or M365 integration instead.

---

#### Option 3: Dataflows Gen2 (Excel Files as Source)

**What it is:** Use Fabric Dataflows Gen2 (Power Query Online) to ingest data from Excel files into the Lakehouse.

**How it works:**
1. Upload Excel files to OneLake or a SharePoint/OneDrive location.
2. Create a Dataflow Gen2 that reads the Excel file, applies Power Query transforms, and writes to a Lakehouse table.
3. Schedule the Dataflow to run on a cadence (e.g., daily).

**Pros:**
- ✅ Brings Excel-maintained data (budgets, targets, reference data) into Fabric.
- ✅ Power Query transformations — clean, reshape, merge before loading.
- ✅ Scheduled refresh — automate the ingestion.
- ✅ No-code / low-code — accessible to analysts.

**Cons:**
- ❌ Batch only — no real-time.
- ❌ CU consumption per refresh.
- ❌ Excel file format quirks (merged cells, named ranges) require careful Power Query handling.

**Recommendation:** **Implement in Week 3 for budget/target data.** Use this for any Excel-maintained reference data that needs to join with the Gold layer (e.g., budget figures, departmental targets, lookup tables).

---

#### Option 4: OneLake File API

**What it is:** Upload Excel files directly to OneLake's Files section in a Lakehouse, then process them with Spark notebooks.

**How it works:**
1. Upload `.xlsx` files to `Files/excel/` in the Gold Lakehouse.
2. Spark notebook reads the files using `openpyxl` or `pandas` and writes to Delta tables.
3. Delta tables become part of the Gold layer and are included in semantic models.

**Pros:**
- ✅ Full programmatic control over Excel file processing.
- ✅ Can handle complex Excel structures (multiple sheets, formulas, macros).
- ✅ Files stored in OneLake — centralized, versioned.

**Cons:**
- ❌ Requires Spark notebook development — higher skill requirement.
- ❌ No built-in scheduling for file upload (manual or API-driven).
- ❌ Overkill for simple Excel data.

**Recommendation:** **Use only for complex Excel files** that Dataflows Gen2 can't handle (e.g., multi-sheet workbooks with business logic). Not a primary integration path.

---

#### Option 5: Power Query in Excel (Direct Connection)

**What it is:** Excel's built-in Power Query connects directly to the Fabric Lakehouse SQL analytics endpoint or Warehouse.

**How it works:**
1. In Excel → Data → Get Data → Azure → Azure Synapse Analytics Workspace.
2. Connect to the Lakehouse SQL analytics endpoint or Warehouse.
3. Build Power Query queries that pull data into Excel tables.

**Pros:**
- ✅ Users can pull raw data from Fabric directly into Excel.
- ✅ Power Query transformations in Excel — familiar for power users.
- ✅ Can connect to both Lakehouse and Warehouse endpoints.

**Cons:**
- ❌ Bypasses the semantic model — no DAX measures, no RLS.
- ❌ Users may create inconsistent metrics (their own calculations vs. official measures).
- ❌ Pulls raw data into Excel — potential performance and governance issues with large datasets.

**Recommendation:** **Allow for advanced users only.** Prefer Analyze in Excel (which goes through the semantic model) for standard users. Power Query direct connections should be restricted to data engineers and analysts who need raw data access.

---

#### Option 6: Microsoft 365 Integration (Excel ↔ Fabric)

**What it is:** Native integration between Excel and Fabric through the Microsoft 365 platform. Includes features like Excel connected to Fabric semantic models via the M365 app, and Fabric data types in Excel.

**Current state:** Several features are in preview or rolling out through 2026:
- **Excel add-in for Power BI** — GA, allows browsing and inserting semantic model data into Excel.
- **Fabric data types in Excel** — Preview. Lets users define custom data types from Fabric data.
- **Direct editing of Fabric data from Excel** — Preview/planned.

**Pros:**
- ✅ Seamless experience — Excel and Fabric feel like one product.
- ✅ Governed — data flows through the semantic model layer.
- ✅ Microsoft's strategic direction — this will only get better.

**Cons:**
- ❌ Some features still in preview — maturity varies.
- ❌ Requires M365 E5 or equivalent licensing for some features.
- ❌ Feature availability varies by update channel (monthly enterprise vs. current channel).

**Recommendation:** **Monitor and adopt incrementally.** Use GA features now (Analyze in Excel add-in). Pilot preview features in DEV workspace. Plan for full adoption as features reach GA.

### 3.3 Excel Integration Rollout Plan

| Priority | Option | Timeline | Target Users |
|----------|--------|----------|-------------|
| **P0** | Analyze in Excel | Week 3 | All report consumers |
| **P1** | Dataflows Gen2 (Excel → Fabric) | Week 3 | Finance, planning teams |
| **P2** | M365 Integration (GA features) | Week 4 | Early adopters |
| **P3** | Power Query in Excel (direct) | Week 4 | Advanced analysts (governed) |
| **Defer** | OneLake File API | Phase 2 | Data engineers (as needed) |
| **Skip** | Power BI Publisher for Excel | — | Deprecated |

---

## 4. Sentiment Analysis Options

The source databases contain text data (survey responses, feedback, comments, notes). We need to extract:

1. **Favorability scores** — A numeric score (e.g., 0–1 or 1–5) indicating positive/negative/neutral sentiment.
2. **Keyword extraction** — Key phrases and topics mentioned in the text.

### 4.1 Option Comparison Matrix

| Option | Accuracy | Scale | Cost | Setup Complexity | Maintenance | Custom Training |
|--------|----------|-------|------|-----------------|-------------|----------------|
| **Azure AI Language Service** | ⭐⭐⭐⭐⭐ | High | Per-API-call | Low | Low | Limited (custom models available) |
| **SynapseML in Fabric Notebooks** | ⭐⭐⭐⭐⭐ | High | CU only | Medium | Low | No (uses Azure AI under the hood) |
| **Azure OpenAI in Fabric** | ⭐⭐⭐⭐⭐ | Medium-High | Per-token | Medium | Medium | Yes (prompt engineering) |
| **Custom Python (NLTK/spaCy/HF)** | ⭐⭐⭐–⭐⭐⭐⭐⭐ | High | CU only | High | High | Full control |
| **Dataflows Gen2 AI Insights** | ⭐⭐⭐⭐ | Low-Medium | CU + AI | Low | Low | No |

### 4.2 Detailed Evaluation

#### Option A: Azure AI Language Service (Text Analytics API)

**What it is:** A managed Azure cognitive service providing sentiment analysis, key phrase extraction, opinion mining, language detection, and entity recognition via REST API.

**How it works:**
1. Provision an Azure AI Language resource in the same region as Fabric.
2. From a Fabric Spark notebook, call the Text Analytics API with text data from the Gold layer.
3. API returns sentiment scores (positive/negative/neutral/mixed with confidence scores) and key phrases.
4. Write results to Delta tables in the Gold layer.
5. Semantic model includes the sentiment scores and keywords.

```python
# Example: Azure AI Language from a Fabric notebook
from azure.ai.textanalytics import TextAnalyticsClient
from azure.core.credentials import AzureKeyCredential

client = TextAnalyticsClient(
    endpoint="https://<resource>.cognitiveservices.azure.com/",
    credential=AzureKeyCredential("<key>")
)

documents = ["Great experience, very helpful staff", "Terrible wait times"]
response = client.analyze_sentiment(documents, show_opinion_mining=True)
```

**Pros:**
- ✅ Production-grade accuracy — trained on massive corpora, handles nuance well.
- ✅ Opinion mining — extracts aspect-level sentiment (e.g., "staff was helpful" → aspect: staff, sentiment: positive).
- ✅ Multi-language support — 12+ languages out of the box.
- ✅ Low setup — managed service, no model training required.
- ✅ Key phrase extraction built-in — one API for both tasks.

**Cons:**
- ❌ API call cost — $1 per 1,000 text records (S tier). Free tier: 5,000 calls/month.
- ❌ External dependency — requires Azure AI resource outside Fabric.
- ❌ Rate limits — 1,000 documents per request, max 5,120 characters per document.
- ❌ Latency — API round-trip adds latency compared to local processing.

**Cost estimate for this project:**
- Assuming 500K text records: ~$500 one-time for initial analysis.
- Incremental: depends on new data volume.
- Free tier covers development and testing.

**Accuracy expectation:** 85–92% agreement with human annotators on standard English text.

---

#### Option B: SynapseML in Fabric Notebooks ⭐ RECOMMENDED

**What it is:** SynapseML (formerly MMLSpark) is a library pre-installed in Fabric Spark that provides distributed transformers for text analytics. It calls Azure AI Services under the hood but integrates natively with Spark DataFrames.

**How it works:**
1. In a Fabric Spark notebook, import SynapseML text analytics transformers.
2. Load text data from the Gold layer as a Spark DataFrame.
3. Apply the `TextSentiment` and `KeyPhraseExtractor` transformers.
4. Results are added as new columns to the DataFrame.
5. Write the enriched DataFrame back to the Gold layer as a Delta table.

```python
from synapse.ml.cognitive import TextSentiment, KeyPhraseExtractor

# Sentiment Analysis
sentiment = (TextSentiment()
    .setTextCol("comment_text")
    .setOutputCol("sentiment_result")
    .setLinkedService("AzureAILanguage")  # or .setSubscriptionKey() + .setLocation()
)

# Key Phrase Extraction
keyphrases = (KeyPhraseExtractor()
    .setTextCol("comment_text")
    .setOutputCol("key_phrases")
    .setLinkedService("AzureAILanguage")
)

# Apply to DataFrame
result_df = sentiment.transform(text_df)
result_df = keyphrases.transform(result_df)

# Write to Gold layer
result_df.write.format("delta").option("vorder", "true").mode("overwrite") \
    .save("Tables/gold_fact_feedback_sentiment")
```

**Pros:**
- ✅ Native Spark integration — operates on DataFrames, scales with the cluster.
- ✅ Same accuracy as Azure AI Language Service (uses the same API under the hood).
- ✅ Batch processing — can process millions of records in a single Spark job.
- ✅ Pre-installed in Fabric — no pip install required.
- ✅ Distributed — Spark parallelizes the API calls across the cluster.
- ✅ Integrated with Fabric linked services — credential management through Fabric, not hardcoded keys.

**Cons:**
- ❌ Still requires an Azure AI Language resource (API calls under the hood).
- ❌ API costs still apply (same pricing as Option A).
- ❌ CU consumption for the Spark notebook execution.

**Cost estimate:**
- API costs: Same as Option A (~$500 for 500K records).
- CU costs: Spark notebook execution (~0.5–2 CU-hours depending on cluster size).

**Why this is the recommendation:**
SynapseML gives us the best of both worlds — Azure AI Language accuracy + Spark scale + native Fabric integration. We don't have to manage API pagination, rate limiting, or error handling manually. SynapseML handles it. The API cost is the same as calling the service directly, but the developer experience is dramatically better.

**Accuracy expectation:** 85–92% (identical to Option A — same underlying model).

---

#### Option C: Azure OpenAI in Fabric

**What it is:** Use GPT models (GPT-4, GPT-4o) via Azure OpenAI Service from Fabric notebooks for sentiment analysis with natural language prompts.

**How it works:**
1. Provision an Azure OpenAI resource with a GPT model deployment.
2. From a Fabric Spark notebook, send text to the GPT model with a prompt like: "Rate the sentiment of this text on a scale of 1-5 and extract key topics."
3. Parse the structured response (JSON) and write to Delta tables.

```python
import openai

openai.api_type = "azure"
openai.api_base = "https://<resource>.openai.azure.com/"
openai.api_key = "<key>"

response = openai.ChatCompletion.create(
    deployment_id="gpt-4o",
    messages=[{
        "role": "system",
        "content": "Analyze sentiment and extract keywords. Return JSON: {score: 1-5, keywords: [...]}"
    }, {
        "role": "user",
        "content": "The staff was incredibly helpful but the wait time was too long."
    }]
)
```

**Pros:**
- ✅ Highest contextual understanding — GPT handles nuance, sarcasm, and complex sentences better than traditional NLP.
- ✅ Flexible — can customize the scoring scale, extract custom entities, generate summaries.
- ✅ Can handle domain-specific jargon with prompt engineering.
- ✅ JSON-mode output for structured results.

**Cons:**
- ❌ **Expensive at scale** — GPT-4o: ~$2.50 per 1M input tokens + $10 per 1M output tokens. For 500K text records: ~$1,500–5,000 depending on text length.
- ❌ Slower — each API call takes 0.5–3 seconds (vs. batched sentiment API at ~100ms per batch).
- ❌ Token limits — max context window; long documents may need chunking.
- ❌ Non-deterministic — same input can produce slightly different scores.
- ❌ Overkill for standard sentiment analysis.
- ❌ Requires Azure OpenAI quota approval.

**Cost estimate:** $1,500–5,000 for 500K records (3–10x more expensive than Azure AI Language).

**Recommendation:** **Reserve for specialized use cases** — domain-specific sentiment that the standard API doesn't handle well, or when you need rich summaries alongside scores. Not the default choice for bulk favorability scoring.

**Accuracy expectation:** 88–95% (higher contextual accuracy, but non-deterministic).

---

#### Option D: Custom Python (NLTK, TextBlob, spaCy, Hugging Face)

**What it is:** Run open-source NLP libraries directly in Fabric Spark notebooks. No external API calls.

**Libraries:**

| Library | Sentiment | Keywords | Accuracy | Speed |
|---------|-----------|----------|----------|-------|
| **TextBlob** | ✅ Polarity (-1 to 1) | ✅ Noun phrases | ⭐⭐⭐ (basic) | Fast |
| **VADER (NLTK)** | ✅ Compound score | ❌ | ⭐⭐⭐ (social media) | Very fast |
| **spaCy** | ❌ (needs extension) | ✅ Entity extraction | ⭐⭐⭐⭐ | Fast |
| **Hugging Face Transformers** | ✅ (distilbert, roberta) | ✅ (with NER models) | ⭐⭐⭐⭐⭐ | Slow (GPU ideal) |

```python
# TextBlob example (simple, fast, moderate accuracy)
from textblob import TextBlob

blob = TextBlob("Great experience, very helpful staff")
print(blob.sentiment.polarity)   # 0.65 (positive)
print(blob.noun_phrases)         # ['great experience', 'helpful staff']
```

```python
# Hugging Face example (high accuracy, needs GPU for speed)
from transformers import pipeline

classifier = pipeline("sentiment-analysis", model="distilbert-base-uncased-finetuned-sst-2-english")
result = classifier("Great experience, very helpful staff")
# [{'label': 'POSITIVE', 'score': 0.9998}]
```

**Pros:**
- ✅ No API costs — runs entirely on Fabric CUs.
- ✅ No external dependencies — all processing stays in Fabric.
- ✅ Full customization — train your own models, fine-tune for domain-specific text.
- ✅ Data privacy — text never leaves your Fabric environment.

**Cons:**
- ❌ Lower accuracy (TextBlob/VADER) or high compute cost (Hugging Face transformers).
- ❌ Requires Python NLP expertise to implement and tune.
- ❌ Hugging Face models need GPU clusters for reasonable throughput — not available on standard Fabric Spark.
- ❌ Maintenance burden — you own the model updates and drift monitoring.
- ❌ Library installation complexity — may need custom Spark environments.

**Cost estimate:** CU costs only (~1–4 CU-hours for 500K records with TextBlob; much higher with transformers).

**Recommendation:** **Use TextBlob/VADER as a quick baseline or fallback** if Azure AI Language is not available. For production-quality results, SynapseML + Azure AI Language is better.

**Accuracy expectation:** TextBlob: 70–80%. VADER: 75–85% (social media optimized). Hugging Face: 85–93%.

---

#### Option E: Dataflows Gen2 with AI Insights

**What it is:** Power Query Online in Dataflows Gen2 includes built-in AI Insights functions for sentiment analysis and key phrase extraction.

**How it works:**
1. Create a Dataflow Gen2 in Fabric.
2. Connect to the Gold layer table containing text data.
3. In the Power Query editor, add an "AI Insights" step → select "Sentiment Analysis" or "Key Phrase Extraction."
4. The function calls Azure AI Services in the background.
5. Output is written to a Lakehouse table.

**Pros:**
- ✅ No-code — accessible to business analysts.
- ✅ Built into Power Query — familiar interface.
- ✅ Integrated with Fabric — results flow directly to Lakehouse.

**Cons:**
- ❌ Limited scale — Dataflows Gen2 are not designed for millions of rows.
- ❌ Slow — Power Query Online is significantly slower than Spark for large datasets.
- ❌ Limited configuration — can't customize prompts, models, or scoring scales.
- ❌ Premium capacity required for AI Insights.
- ❌ Black box — less visibility into what model is being used.

**Cost estimate:** CU costs + AI Insights consumption (billed separately).

**Recommendation:** **Use only for small datasets or prototyping.** For production-scale sentiment analysis, use SynapseML.

**Accuracy expectation:** 85–90% (same underlying Azure AI models, but less control).

### 4.3 Sentiment Analysis Recommendation

**Primary: SynapseML in Fabric Notebooks (Option B)**

| Decision | Choice |
|----------|--------|
| **Approach** | SynapseML `TextSentiment` + `KeyPhraseExtractor` in Fabric Spark notebooks |
| **Underlying service** | Azure AI Language Service (provisioned as a linked service) |
| **Scale** | Batch processing on Spark — handles millions of records |
| **Output** | Favorability score (0.0–1.0 positive/negative/neutral), key phrases array |
| **Storage** | Results written to Gold layer Delta tables (`gold_fact_sentiment`) |
| **Semantic model** | Sentiment scores and keywords included in the analytics semantic model |
| **Schedule** | Run after Gold layer refresh — triggered by pipeline orchestration |
| **Cost** | ~$500 for 500K records (API) + ~$10 CU (Spark) |
| **Fallback** | TextBlob for quick prototyping; Azure OpenAI for domain-specific edge cases |

**Favorability Score Schema:**

```
gold_fact_sentiment
├── record_id (INT) — FK to source record
├── source_table (STRING) — origin table name
├── text_column (STRING) — which column was analyzed
├── original_text (STRING) — the raw text
├── sentiment_label (STRING) — "positive" / "negative" / "neutral" / "mixed"
├── sentiment_score_positive (DOUBLE) — confidence score 0.0–1.0
├── sentiment_score_negative (DOUBLE) — confidence score 0.0–1.0
├── sentiment_score_neutral (DOUBLE) — confidence score 0.0–1.0
├── favorability_score (DOUBLE) — computed: positive - negative (range -1.0 to 1.0)
├── key_phrases (ARRAY<STRING>) — extracted key phrases
├── analyzed_at (TIMESTAMP) — when the analysis was run
└── model_version (STRING) — Azure AI model version for reproducibility
```

### 4.4 Sentiment Analysis Pipeline

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│  Gold Layer  │     │  Spark       │     │  Azure AI    │     │  Gold Layer  │
│  Text Data   │────▶│  Notebook    │────▶│  Language    │────▶│  Sentiment   │
│  (Delta)     │     │  (SynapseML) │     │  Service     │     │  Results     │
│              │     │              │     │  (API)       │     │  (Delta)     │
└──────────────┘     └──────────────┘     └──────────────┘     └──────────────┘
                                                                     │
                                                                     ▼
                                                              ┌──────────────┐
                                                              │  Semantic    │
                                                              │  Model       │
                                                              │  (Direct     │
                                                              │   Lake)      │
                                                              └──────────────┘
```

---

## 5. Fabric IQ Ontologies

### 5.1 What is Fabric IQ?

Fabric IQ (formerly known as Copilot in Fabric) is Microsoft's AI-powered intelligence layer that enables natural language interaction with data in Fabric. **Ontologies** are a component of Fabric IQ that define business process semantics — they map business concepts (e.g., "customer," "order," "revenue") to the underlying data structures (tables, columns, measures) in semantic models and lakehouses.

**What ontologies do:**
- Define business entities and their relationships in a structured vocabulary.
- Map business terms to physical data objects (tables, columns, measures).
- Enable natural language queries — users ask "What was revenue last quarter?" and the ontology resolves "revenue" to the correct DAX measure.
- Provide context for Copilot-generated answers, ensuring accuracy.

### 5.2 How Ontologies Map to Semantic Models

```
┌─────────────────────────────────────────────────────────────────────┐
│                         FABRIC IQ ONTOLOGY                          │
│                                                                     │
│   Business Concept          Maps To                                 │
│   ─────────────────         ──────────────────────────              │
│   "Customer"           →    dim_customer table                      │
│   "Revenue"            →    [Total Revenue] DAX measure             │
│   "Region"             →    dim_region[RegionName]                  │
│   "Last Quarter"       →    Time intelligence filter                │
│   "Top Customers"      →    TOPN + [Total Revenue]                  │
│   "Favorability"       →    [Avg Favorability] DAX measure          │
│                                                                     │
│   Relationships:                                                    │
│   "Customer Revenue"   →    dim_customer ──▶ fact_sales ──▶ measure │
│   "Regional Sentiment" →    dim_region ──▶ fact_sentiment ──▶ score │
└─────────────────────────────────────────────────────────────────────┘
```

### 5.3 Current State Assessment

| Feature | Status | Notes |
|---------|--------|-------|
| **Copilot in Power BI** | GA | Natural language Q&A on semantic models |
| **Copilot in Fabric notebooks** | GA | Code generation, explanation |
| **Ontology definitions** | Preview | Define business vocabularies |
| **Custom ontology authoring** | Preview | Create your own entity-to-data mappings |
| **Ontology-powered Q&A** | Preview | Natural language queries using ontology context |
| **Ontology sharing across workspaces** | Not yet available | Currently workspace-scoped |
| **Ontology versioning** | Not yet available | No built-in version control |
| **API for ontology management** | Limited preview | Programmatic ontology creation is early-stage |

### 5.4 Evaluation for This Project

**Benefits of adopting ontologies now:**
- Improves Copilot accuracy for natural language questions against our semantic models.
- Documents business terminology formally — useful even if the AI features are immature.
- Early adoption means our team builds competency before GA features mature.

**Risks of adopting now:**
- Preview features may change or be deprecated — risk of rework.
- Limited tooling — authoring experience is still rough.
- No versioning or cross-workspace sharing limits enterprise-scale deployment.
- Time investment competes with higher-priority deliverables (semantic models, reports, sentiment analysis).

### 5.5 Recommendation: Monitor, Don't Invest Heavily

| Decision | Rationale |
|----------|-----------|
| **Create basic ontology definitions** | Define key business entities (customer, region, metric, sentiment) in the ontology preview. Low effort (~2 hours). |
| **Do NOT build comprehensive ontologies** | Feature is too immature for production reliance. Focus effort on semantic model design instead. |
| **Monitor monthly** | Check Fabric release notes monthly for ontology GA announcements and new capabilities. |
| **Invest heavily when:** | (1) Ontology authoring reaches GA, (2) cross-workspace sharing is available, (3) versioning is supported. |
| **Assign to:** | Drummer (Integration Specialist) — owns Fabric IQ evaluation per the project plan. |

**The semantic model IS the ontology for now.** A well-designed semantic model with clear table names, measure descriptions, and relationship documentation serves 90% of the same purpose as a formal ontology. When Fabric IQ matures, the semantic model metadata will be the foundation for ontology definitions.

---

## 6. Implementation Timeline

| Week | Activity | Owner | Deliverable |
|------|----------|-------|-------------|
| **Week 3 — Mon** | Create Direct Lake semantic models on Gold layer | Alex | Semantic models published in DEV workspace |
| **Week 3 — Mon** | Configure relationships, DAX measures, RLS | Alex | Model validated with test queries |
| **Week 3 — Wed** | Build first Power BI reports (operational dashboard) | Alex | Draft reports in DEV |
| **Week 3 — Thu** | Run sentiment analysis notebook (SynapseML) | Alex | Sentiment results in Gold layer |
| **Week 3 — Fri** | Enable Analyze in Excel on semantic models | Alex | Excel connectivity tested |
| **Week 3 — Fri** | Set up Dataflow Gen2 for Excel budget files | Alex | Budget data in Gold layer |
| **Week 4 — Mon** | Complete all Power BI reports | Alex | Full report suite |
| **Week 4 — Mon** | Create basic Fabric IQ ontology definitions | Drummer | Ontology draft |
| **Week 4 — Tue** | Performance testing — Direct Lake, report load times | Alex + Holden | Performance report |
| **Week 4 — Wed** | Promote to PROD via Deployment Pipeline | Alex | Reports live in PROD |
| **Week 4 — Thu** | Stakeholder UAT — reports, Excel, sentiment | Alex + All | UAT feedback |

---

## 7. Risk Register

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| Direct Lake falls back to DirectQuery on large tables | Medium | High | Monitor F64 guardrails; partition large tables; run `DirectLakeFallback` monitoring |
| Azure AI Language Service rate limits during bulk sentiment analysis | Low | Medium | SynapseML handles batching and retries; provision S-tier for higher limits |
| Excel users bypass semantic model (direct Lakehouse queries) | Medium | Medium | Restrict SQL analytics endpoint access; train users on Analyze in Excel |
| Gold layer schema changes break semantic model | Medium | High | Use Deployment Pipelines; test in DEV before PROD promotion |
| Sentiment analysis accuracy insufficient for business decisions | Low | Medium | Validate with human-annotated sample; consider Azure OpenAI for edge cases |
| Fabric IQ ontology preview features change or get deprecated | Medium | Low | Minimal investment; basic definitions only |

---

## 8. Success Metrics

| Metric | Target | How We Measure |
|--------|--------|---------------|
| Report load time (P95) | < 5 seconds | Fabric Capacity Metrics + Power BI performance analyzer |
| Direct Lake fallback rate | < 5% of queries | `DirectLakeFallback` events in Capacity Metrics |
| Semantic model refresh latency | < 30 seconds (framing) | Power BI refresh history |
| Sentiment analysis accuracy | > 85% vs human baseline | Sample 200 records, compare automated vs. manual scores |
| Excel integration adoption | > 50% of Excel users using Analyze in Excel within 30 days | Power BI usage metrics |
| Report adoption | > 80% of identified stakeholders accessing reports within 2 weeks of go-live | Power BI usage metrics |

---

*This document is maintained by Alex (BI & Analytics Engineer). Last updated: 2026-04-08.*
