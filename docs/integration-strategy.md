# Integration Strategy: CluedIn MDM, Salesforce, S3 Shortcuts & Fabric IQ

> **Project:** AtlMigration — Enterprise Data Platform Migration to Microsoft Fabric
> **Author:** Drummer (Integration Specialist)
> **Date:** 2026-04-08
> **Status:** ACTIVE
> **Timeline:** 4 weeks (integration work runs in parallel with core migration)
> **Owner:** Brian Hitney

---

## Executive Summary

This document evaluates four integration areas that extend the core Fabric migration: CluedIn for Master Data Management, Salesforce connectivity, Amazon S3 shortcuts, and Fabric IQ ontologies. Each is assessed for feasibility, effort, cost, and project fit.

**Bottom line:**

| Integration | Verdict | Priority | Timeline |
|-------------|---------|----------|----------|
| Amazon S3 Shortcuts | ✅ Implement now | P1 — High | Week 1–2 |
| Salesforce Integration | ✅ Feasible — multiple options | P2 — Medium | Week 2–3 |
| CluedIn MDM | ⏸ Evaluate in parallel, non-blocking | P2 — Medium | Week 2–4 |
| Fabric IQ Ontologies | 📋 Document and defer | P3 — Low | Post-migration |

---

## 1. CluedIn MDM Evaluation

### 1.1 What Is CluedIn?

CluedIn is a **Master Data Management (MDM) platform** built natively on Microsoft Azure and deeply integrated with Microsoft Fabric. It provides:

- **Entity resolution** — Matching records across disparate sources that represent the same real-world entity (e.g., "John Smith" in SQL Server and "J. Smith" in Salesforce).
- **Data deduplication** — Identifying and merging duplicate records using fuzzy matching, ML-based similarity scoring, and configurable match rules.
- **Golden record creation** — Producing a single, authoritative "golden record" for each entity by merging the best attributes from all matched source records using survivorship rules.
- **Data quality** — Profiling, cleansing, standardization, and enrichment of master data.
- **Graph-based data model** — CluedIn models all data as a knowledge graph, enabling relationship discovery between entities.

### 1.2 How CluedIn Integrates with Microsoft Fabric

CluedIn has a **first-party integration with Microsoft Fabric** — it is one of Microsoft's recommended MDM partners:

```
┌─────────────────────────────────────────────────────────────────────┐
│                          DATA SOURCES                               │
│  SQL Server  │  PostgreSQL  │  MongoDB  │  Salesforce  │  S3 Files │
└──────┬───────┴──────┬───────┴─────┬─────┴──────┬───────┴─────┬─────┘
       │              │             │            │             │
       ▼              ▼             ▼            ▼             ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        CluedIn MDM                                  │
│  ┌──────────┐  ┌──────────────┐  ┌───────────┐  ┌───────────────┐  │
│  │ Ingest   │─▶│ Entity       │─▶│ Golden    │─▶│ Export to     │  │
│  │ (200+    │  │ Resolution & │  │ Record    │  │ Fabric        │  │
│  │ connectors)│ │ Dedup        │  │ Creation  │  │ (OneLake)     │  │
│  └──────────┘  └──────────────┘  └───────────┘  └───────┬───────┘  │
└──────────────────────────────────────────────────────────┬──────────┘
                                                           │
                                                           ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      MICROSOFT FABRIC                               │
│                                                                     │
│  Bronze Lakehouse  ──▶  Silver Lakehouse  ──▶  Gold Lakehouse       │
│  (mirrored + raw)      (+ CluedIn golden     (dimensional models   │
│                         records merged in)     with clean master    │
│                                                data)                │
└─────────────────────────────────────────────────────────────────────┘
```

**Integration patterns:**

1. **CluedIn → OneLake Export:** CluedIn exports golden records directly to OneLake as Delta tables. These land in the Silver layer as pre-cleansed, deduplicated master data.
2. **Fabric → CluedIn Ingestion:** CluedIn can ingest data from Fabric Lakehouses/Warehouses using its 200+ built-in connectors, including a native Microsoft Fabric connector.
3. **CluedIn Dataverse Sync:** CluedIn can sync golden records to Dataverse, which Fabric can then access via Dataverse shortcuts.
4. **API-based integration:** CluedIn exposes GraphQL and REST APIs for programmatic access to golden records and entity graphs.

### 1.3 Use Cases for This Project

| Use Case | Description | Value |
|----------|-------------|-------|
| **Customer deduplication** | Merge customer records across SQL Server, PostgreSQL, and Salesforce | Eliminate duplicates in analytics, single customer view |
| **Product master** | Consolidate product data from multiple operational systems | Consistent product hierarchy for reporting |
| **Reference data management** | Standardize codes, categories, and lookup values across sources | Uniform dimensions in Gold layer |
| **Cross-source entity linking** | Link related entities (customer ↔ orders ↔ products) across systems | Enable cross-system lineage and impact analysis |

### 1.4 Implementation Approach

**Parallel track — non-blocking to main migration.**

CluedIn work runs alongside the core Fabric migration. It does NOT create dependencies on the critical path. If CluedIn evaluation takes longer than expected, the core migration proceeds without it.

| Phase | Duration | Activities |
|-------|----------|------------|
| **Phase 1: Evaluate** | Week 2 | Provision CluedIn trial environment. Ingest sample data from 2–3 sources. Test entity resolution and dedup on customer data. |
| **Phase 2: POC** | Week 3 | Configure match rules and survivorship logic. Export golden records to a test Lakehouse. Validate quality against source data. |
| **Phase 3: Decide** | Week 4 | Cost-benefit analysis. Go/no-go decision. If "go": plan production implementation for post-migration sprint. |

### 1.5 Cost Considerations

| Component | Estimated Cost |
|-----------|---------------|
| **CluedIn License** | Per-record pricing. Typically $0.01–$0.05 per record/month for enterprise tiers. Volume discounts apply. Contact CluedIn sales for exact pricing. |
| **Azure Infrastructure** | CluedIn runs on AKS (Azure Kubernetes Service). Expect $500–$2,000/month for a small-to-medium deployment. |
| **Fabric Compute** | Minimal — CluedIn exports are lightweight Delta writes. < 1% of total CU budget. |
| **Professional Services** | Optional. CluedIn offers implementation packages starting ~$25K. Self-service is possible with documentation. |

**Total estimated cost:** $2,000–$5,000/month ongoing (infrastructure + licensing) plus one-time setup effort of 2–4 weeks engineering time.

### 1.6 Recommendation

> **⏸ EVALUATE — Do not commit yet.**
>
> CluedIn is a strong MDM platform with genuine Fabric integration. However:
> - MDM is not required for the core migration to succeed.
> - The 4-week timeline doesn't allow for a full MDM implementation.
> - Cost is non-trivial and needs business justification.
>
> **Action:** Run the POC in parallel. Make a go/no-go decision at Week 4. If data quality issues emerge during migration (duplicate customers, conflicting records), CluedIn becomes a stronger candidate.

---

## 2. Salesforce Integration — Is It Possible?

### The Short Answer

**Yes, Salesforce integration with Microsoft Fabric is absolutely possible.** There are multiple viable options, each with different tradeoffs. Here is a complete evaluation.

### 2.1 Option A: Fabric Dataflows Gen2 (Power Query Connector) ⭐ RECOMMENDED FOR MOST SCENARIOS

**How it works:** Dataflows Gen2 in Fabric uses the Power Query engine, which includes native connectors for **Salesforce Objects** and **Salesforce Reports**. You visually build data extraction flows that pull Salesforce data into Fabric Lakehouses or Warehouses.

| Attribute | Detail |
|-----------|--------|
| **Feasibility** | ✅ Fully supported. GA connector. |
| **Setup complexity** | Low — visual, no-code interface in Fabric portal |
| **Authentication** | OAuth 2.0 (Salesforce username + security token or connected app) |
| **Data freshness** | Scheduled refresh — minimum every 15 minutes |
| **Supported objects** | All standard and custom Salesforce objects (Accounts, Contacts, Opportunities, Cases, etc.) and Salesforce Reports |
| **Incremental refresh** | Supported via query folding on `SystemModstamp` or `LastModifiedDate` |
| **Transformations** | Full Power Query M transformations during extraction (filter, rename, join, type conversion) |
| **Output** | Delta tables in Lakehouse or tables in Warehouse |
| **Compute cost** | CU consumed during refresh — proportional to data volume. Light for typical Salesforce volumes. |
| **Limitations** | Batch only (no real-time streaming). Subject to Salesforce API rate limits (typically 100K API calls/24h for Enterprise Edition). Large object extracts may hit Salesforce SOQL query timeout (120s). |

**Setup steps:**
1. In Fabric workspace, create a new **Dataflow Gen2**.
2. Select **Salesforce Objects** or **Salesforce Reports** as the data source.
3. Enter Salesforce credentials (username, password + security token, or OAuth connected app).
4. Select objects/reports to extract.
5. Apply transformations as needed.
6. Set destination to the Bronze Lakehouse.
7. Configure refresh schedule.

### 2.2 Option B: Fabric Data Pipelines (Copy Activity)

**How it works:** Fabric Data Pipelines (built on Azure Data Factory) include a **Salesforce connector** in the Copy Activity. This is a code-free, pipeline-based approach for batch data extraction.

| Attribute | Detail |
|-----------|--------|
| **Feasibility** | ✅ Fully supported. GA connector. |
| **Setup complexity** | Low-Medium — pipeline authoring, mapping configuration |
| **Authentication** | OAuth 2.0 or username/password + security token |
| **Data freshness** | Scheduled or event-triggered — configurable intervals |
| **Supported objects** | Standard and custom Salesforce objects |
| **Incremental loading** | Manual watermark pattern — track `LastModifiedDate` and filter on subsequent runs |
| **Transformations** | Column mapping during copy. For complex transforms, chain with Dataflows or Spark notebooks. |
| **Output** | Delta tables in Lakehouse, Parquet/CSV in OneLake, or tables in Warehouse |
| **Compute cost** | CU consumed per pipeline run. Very efficient for batch loads. |
| **Limitations** | No real-time. API rate limits apply. Requires manual watermark management for incremental loads. Less transformation flexibility than Dataflows Gen2. |

**When to use over Dataflows Gen2:** When you need pipeline orchestration (dependencies, retries, alerting) or when Salesforce extraction is part of a larger multi-source pipeline.

### 2.3 Option C: Salesforce Data Cloud + Fabric Integration

**How it works:** Salesforce Data Cloud (formerly CDP) can establish **bidirectional data sharing** with external platforms. Microsoft and Salesforce have announced integration capabilities via Zero Copy Partner Network.

| Attribute | Detail |
|-----------|--------|
| **Feasibility** | ⚠️ Partially available. Zero Copy integration announced but requires Salesforce Data Cloud license. |
| **Setup complexity** | High — requires Salesforce Data Cloud provisioning and configuration |
| **Authentication** | Salesforce Data Cloud admin configuration, OAuth flows |
| **Data freshness** | Near real-time for Data Cloud managed data |
| **Bidirectional** | Yes — can push Fabric data back to Salesforce Data Cloud |
| **Cost** | Expensive — Salesforce Data Cloud licensing is significant ($$$). Not justified unless already licensed. |
| **Limitations** | Requires Salesforce Data Cloud (separate product and license from core Salesforce). Not a fit for simple "read Salesforce data" scenarios. Enterprise feature. |

**When to use:** Only if the organization already has Salesforce Data Cloud licensed AND needs bidirectional data sharing (pushing Fabric insights back into Salesforce for CRM workflows).

### 2.4 Option D: Third-Party ETL Tools (Fivetran, Informatica, Airbyte, etc.)

**How it works:** Third-party data integration platforms provide managed Salesforce connectors that extract data and load it into Fabric (typically via OneLake/ADLS Gen2 endpoints or direct Lakehouse connectors).

| Tool | Salesforce Connector | Fabric Destination | Cost Model |
|------|---------------------|--------------------|------------|
| **Fivetran** | ✅ GA — fully managed, schema drift handling, incremental | ✅ OneLake / ADLS Gen2 destination | Per-row pricing (Monthly Active Rows) |
| **Informatica IDMC** | ✅ GA — enterprise-grade, complex transformations | ✅ Azure-native destinations | License-based |
| **Airbyte** | ✅ GA — open-source option available | ✅ S3/ADLS Gen2 → OneLake shortcut | Per-row or self-hosted (free) |
| **Hevo Data** | ✅ GA — no-code, automated schema management | ✅ Azure destinations | Event-volume pricing |

| Attribute | Detail |
|-----------|--------|
| **Feasibility** | ✅ Proven at scale. Many organizations use these tools. |
| **Setup complexity** | Low-Medium — managed services handle schema changes, retries, error handling |
| **Data freshness** | Near real-time (5-minute intervals) to scheduled batch |
| **Incremental** | Fully automatic incremental extraction (CDC-like) |
| **Cost** | Additional licensing cost on top of Fabric. Fivetran ~$1–$5 per million rows/month. |
| **Limitations** | Adds another vendor to manage. Cost can scale with data volume. Overkill for simple, low-volume Salesforce extracts. |

**When to use:** When Salesforce data volumes are high, schema changes are frequent, or the team already uses one of these tools for other integrations.

### 2.5 Option E: Fabric Mirroring for Salesforce

| Attribute | Detail |
|-----------|--------|
| **Feasibility** | ❌ **Not currently supported.** |
| **Status** | As of April 2026, Fabric Mirroring supports: SQL Server, Azure SQL Database, Azure Cosmos DB, Snowflake, Azure Databricks, PostgreSQL, MySQL, and MongoDB. **Salesforce is not on the supported mirroring list.** |
| **Outlook** | Microsoft has not announced Salesforce mirroring on the public roadmap. Given the REST-API-based nature of Salesforce (vs. database CDC), mirroring is architecturally unlikely in the near term. |

**Verdict:** Not an option today. Use Dataflows Gen2 or Data Pipelines instead.

### 2.6 Option F: OneLake Shortcuts to Salesforce

| Attribute | Detail |
|-----------|--------|
| **Feasibility** | ❌ **Not directly supported.** |
| **Status** | OneLake shortcuts support: ADLS Gen2, Amazon S3, Google Cloud Storage, Dataverse, and other OneLake locations. Salesforce is not a supported shortcut source. |
| **Workaround** | Export Salesforce data to S3 (via Salesforce Data Export or third-party tool) → create S3 shortcut → access in Fabric. This adds latency and complexity. |

**Verdict:** Not recommended as a primary approach. Use only if Salesforce data is already landing in S3 for other reasons.

### 2.7 Salesforce Integration — Recommended Approach

```
┌──────────────────┐     Dataflows Gen2       ┌──────────────────┐
│   Salesforce      │  ──────────────────────▶ │   Bronze         │
│   (Objects/       │   Power Query connector  │   Lakehouse      │
│    Reports)       │   OAuth 2.0 auth         │   (Delta tables) │
│                   │   Scheduled refresh       │                  │
└──────────────────┘   (every 15–60 min)       └──────────────────┘
```

> **✅ RECOMMENDATION: Use Fabric Dataflows Gen2 with the Salesforce Objects connector.**
>
> **Why:**
> - Native to Fabric — no additional licensing or tooling.
> - Visual, no-code setup — fastest time to value.
> - Power Query transformations during extraction.
> - Incremental refresh on `SystemModstamp` keeps loads efficient.
> - Scheduled refresh every 15 minutes is sufficient for analytical use cases.
>
> **If you need orchestration** (dependencies on other data sources, complex error handling), wrap the Dataflow in a **Data Pipeline** for scheduling and monitoring.
>
> **If you need near real-time** (< 5 minute latency), consider Fivetran or a Salesforce Platform Event → Azure Event Hub → Fabric Eventstream pattern.

### 2.8 Salesforce Implementation Plan

| Phase | Duration | Activities |
|-------|----------|------------|
| **Phase 1: Connect** | Week 2 (2 days) | Create Salesforce connected app (OAuth). Build Dataflow Gen2 extracting key objects (Account, Contact, Opportunity, Case). Land in Bronze. |
| **Phase 2: Transform** | Week 2–3 (2 days) | Build Silver layer transformations: dedup, type standardization, null handling. Join Salesforce data with other sources. |
| **Phase 3: Model** | Week 3 (2 days) | Integrate Salesforce dimensions into Gold layer star schemas. Add to Direct Lake semantic model. |
| **Phase 4: Validate** | Week 3 (1 day) | Row count validation, sample comparison against Salesforce reports, refresh schedule tuning. |

**Total estimated effort:** 7 engineering days.

---

## 3. Amazon S3 Shortcuts

### 3.1 What Are OneLake Shortcuts?

OneLake shortcuts are **pointers** to external data that appear as native folders within a Fabric Lakehouse. Data remains in the source (Amazon S3) — **no data movement or duplication occurs.** Fabric reads data directly from S3 at query time.

This is the zero-copy integration pattern: S3 data participates in Fabric queries, notebooks, and pipelines without ETL.

```
┌──────────────────┐                          ┌──────────────────┐
│   Amazon S3       │◀── Direct read at ──────│   Fabric         │
│   Bucket          │    query time            │   Lakehouse      │
│                   │    (no data movement)    │   (Bronze layer) │
│  /data/           │                          │  /Files/          │
│    events.parquet  │                          │    s3-shortcut/  │
│    logs.csv        │                          │      → events.parquet │
│    images/         │                          │      → logs.csv       │
└──────────────────┘                          └──────────────────┘
```

### 3.2 Step-by-Step Setup Guide

#### Prerequisites
- A Fabric workspace with at least Contributor access.
- A Fabric Lakehouse (the Bronze Lakehouse for this project).
- An Amazon S3 bucket with the data to access.
- AWS IAM credentials with read access to the bucket (see Section 3.3).

#### Step 1: Prepare AWS IAM Credentials

**Option A: IAM User with Access Keys (simpler)**

1. In the AWS IAM Console, create a new IAM user (e.g., `fabric-onelake-reader`).
2. Attach a policy granting read access:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "s3:GetBucketLocation",
                "s3:ListBucket"
            ],
            "Resource": [
                "arn:aws:s3:::your-bucket-name",
                "arn:aws:s3:::your-bucket-name/*"
            ]
        }
    ]
}
```

3. Generate an **Access Key ID** and **Secret Access Key**. Store securely.

**Option B: Cross-Account IAM Role (more secure, recommended for production)**

1. In the target AWS account, create an IAM role with a trust policy allowing the Microsoft Fabric service principal.
2. Microsoft's Fabric service uses the AWS STS `AssumeRole` API to obtain temporary credentials.
3. This avoids storing long-lived access keys. Requires configuring the Fabric connection with the role ARN.
4. Consult Microsoft documentation for the exact external ID and principal to trust.

#### Step 2: Create the Connection in Fabric

1. Go to **Fabric Settings → Manage connections and gateways**.
2. Click **+ New connection**.
3. Select **Amazon S3** as the connection type.
4. Enter:
   - **S3 bucket URL:** `https://your-bucket-name.s3.amazonaws.com` or `s3://your-bucket-name`
   - **Authentication method:** Key (Access Key ID + Secret Access Key) or Role ARN
   - **Access Key ID:** (from Step 1)
   - **Secret Access Key:** (from Step 1)
5. Test the connection. Click **Create**.

#### Step 3: Create the Shortcut in the Lakehouse

1. Open the **Bronze Lakehouse** in the Fabric portal.
2. In the **Explorer** pane, right-click on the **Files** folder (for unstructured data) or **Tables** folder (for Delta/Parquet structured data).
3. Select **New shortcut**.
4. Choose **Amazon S3** as the external source.
5. Select the connection created in Step 2.
6. Browse or enter the S3 path:
   - **Bucket:** `your-bucket-name`
   - **Subfolder (optional):** `data/events/` (to scope the shortcut to a specific prefix)
7. Name the shortcut (e.g., `s3-events`).
8. Click **Create**.

#### Step 4: Verify Access

1. In the Lakehouse Explorer, the shortcut appears as a folder with a special shortcut icon (🔗).
2. Click into the shortcut to browse files.
3. For Parquet/Delta files in the **Tables** section: the SQL analytics endpoint will automatically expose them as queryable tables.
4. Test with a Spark notebook:

```python
df = spark.read.parquet("Files/s3-events/")
df.show(5)
df.count()
```

5. Test with SQL analytics endpoint:

```sql
SELECT TOP 10 * FROM [dbo].[s3_events]
```

### 3.3 Authentication Requirements

| Method | Security Level | Best For | Key Management |
|--------|---------------|----------|----------------|
| **IAM User + Access Keys** | Medium | Dev/test, quick setup | Keys must be rotated regularly. Stored in Fabric connection. |
| **IAM Role (AssumeRole)** | High | Production | No long-lived keys. Temporary credentials auto-rotate. |
| **IAM Role + External ID** | Highest | Multi-tenant, cross-account | Prevents confused-deputy attacks. Recommended for enterprise. |

**Minimum required S3 permissions:**

| Permission | Purpose |
|-----------|---------|
| `s3:GetObject` | Read files from the bucket |
| `s3:ListBucket` | List files and folders in the bucket |
| `s3:GetBucketLocation` | Determine the bucket's AWS region |

**NOT required:** `s3:PutObject`, `s3:DeleteObject`. Shortcuts are **read-only** — Fabric cannot write to S3 via shortcuts.

### 3.4 Supported File Formats and Limitations

**Supported formats (auto-detected):**

| Format | Table Shortcut | Files Shortcut | Notes |
|--------|---------------|----------------|-------|
| **Parquet** | ✅ | ✅ | Best performance. Columnar, compressed. |
| **Delta Lake** | ✅ | ✅ | Full Delta table support including time travel. |
| **CSV** | ❌ (schema required) | ✅ | Must define schema manually for SQL queries. |
| **JSON** | ❌ | ✅ | Read via Spark. Not exposed as SQL tables. |
| **Avro** | ❌ | ✅ | Read via Spark. |
| **ORC** | ❌ | ✅ | Read via Spark. |
| **Images/Binary** | ❌ | ✅ | Accessible as files, not queryable. |

**Limitations:**

| Limitation | Detail |
|-----------|--------|
| **Read-only** | Cannot write data back to S3 through shortcuts. |
| **No CDC/streaming** | Shortcuts do not detect file additions in real-time. Requires query-time discovery. |
| **Cross-region latency** | If S3 bucket is in a different region than Fabric capacity, latency increases. Co-locate for best performance. |
| **File listing overhead** | Large buckets (millions of files) may experience slow directory listing. Use partitioned folder structures. |
| **Authentication scope** | One connection per bucket (or prefix). Cannot mix credentials within a single shortcut. |
| **No file-level security** | The Fabric connection has access to everything under the shortcut path. Use S3 prefixes to scope access. |
| **Delta table metadata** | For Delta shortcuts in Tables, the `_delta_log` must be accessible. Ensure IAM policy covers the log path. |

### 3.5 Performance Considerations

| Factor | Recommendation |
|--------|----------------|
| **File format** | Use **Parquet or Delta** for best query performance. Columnar formats enable predicate pushdown and column pruning. |
| **File size** | Target **128 MB – 1 GB per file**. Too many small files degrades performance (S3 list overhead). Too few large files limits parallelism. |
| **Partitioning** | Use **Hive-style partitioning** (`/year=2026/month=04/day=08/data.parquet`) for partition pruning on date-based queries. |
| **Compression** | **Snappy** (default for Parquet) is the best balance of compression ratio and decompression speed. |
| **Caching** | OneLake may cache frequently accessed shortcut data. Subsequent queries on the same data are faster. |
| **Concurrent access** | Shortcuts support concurrent reads from multiple Fabric engines (Spark, SQL, pipelines). |
| **Network** | Fabric reads from S3 over the public internet (or AWS PrivateLink if configured). Ensure sufficient bandwidth for large datasets. |

### 3.6 Security Considerations

| Concern | Mitigation |
|---------|-----------|
| **Data in transit** | All S3 access uses HTTPS (TLS 1.2+). Data is encrypted in transit. |
| **Data at rest** | Data remains in S3 with its existing encryption (SSE-S3, SSE-KMS, or SSE-C). Fabric does not alter S3 encryption. |
| **Credential storage** | Fabric stores S3 credentials in Azure Key Vault (managed by the platform). Credentials are not exposed to end users. |
| **Access control** | Apply Fabric workspace roles to control who can create/view shortcuts. S3 bucket policies provide source-side access control. |
| **Cross-cloud compliance** | Data leaves AWS and enters Microsoft's network at query time. Evaluate data residency and compliance requirements. |
| **Audit trail** | S3 access is logged in AWS CloudTrail. Fabric access is logged in Microsoft 365 audit logs. Both sides have visibility. |
| **VNet/Private Endpoints** | For maximum security, configure AWS PrivateLink to S3 and Fabric managed VNet. Eliminates public internet transit. |

---

## 4. Fabric IQ Ontologies

### 4.1 What Is Fabric IQ?

**Fabric IQ** (also referred to as **Copilot in Fabric** with business context features) is a set of capabilities in Microsoft Fabric that enables organizations to layer **business process semantics** on top of their data artifacts. The core idea: data assets in Fabric (tables, columns, measures) map to real-world business concepts through formal **ontology definitions**.

Think of it as giving Fabric a "business vocabulary" — when someone asks Copilot "what were our sales last quarter?", the ontology tells Fabric exactly which tables, columns, and measures constitute "sales."

### 4.2 How Ontologies Work

An ontology in Fabric IQ maps **business concepts** to **data artifacts**:

```
┌────────────────────────┐        ┌────────────────────────┐
│   BUSINESS CONCEPTS    │        │   DATA ARTIFACTS       │
│                        │        │                        │
│  "Customer"            │ ──────▶│  Gold.DimCustomer      │
│  "Revenue"             │ ──────▶│  Gold.FactSales.Amount │
│  "Product Category"    │ ──────▶│  Gold.DimProduct.Cat   │
│  "Order Date"          │ ──────▶│  Gold.FactSales.Date   │
│  "Customer Lifetime    │ ──────▶│  SemanticModel.CLV     │
│   Value"               │        │  (DAX measure)         │
└────────────────────────┘        └────────────────────────┘
```

**Key components:**

| Component | Description |
|-----------|-------------|
| **Business entities** | Named concepts (Customer, Product, Order) mapped to tables/views |
| **Attributes** | Properties of entities mapped to columns |
| **Measures** | Business metrics mapped to DAX measures or computed columns |
| **Relationships** | How entities relate (Customer has many Orders) — mirrors semantic model relationships |
| **Business rules** | Constraints and logic (Revenue = Quantity × Unit Price) |
| **Synonyms** | Alternative names for concepts (Revenue = Sales = Turnover = Income) |
| **Descriptions** | Natural language descriptions that help Copilot understand intent |

### 4.3 Current Feature Maturity

| Feature | Status (as of April 2026) | Notes |
|---------|---------------------------|-------|
| **Copilot in Power BI** (Q&A with semantic models) | ✅ GA | Uses semantic model metadata (descriptions, synonyms) as a lightweight ontology |
| **Semantic Model linguistic schema** | ✅ GA | Define synonyms and phrasings in semantic models — a form of ontology |
| **Fabric Copilot for data engineering** | ✅ GA | Context-aware code generation using Lakehouse/Warehouse metadata |
| **Business glossary / formal ontology editor** | ⚠️ Preview / Roadmap | A dedicated ontology management UI in Fabric is in early stages. Limited availability. |
| **Enterprise knowledge graph** | 🔮 Roadmap | Microsoft has signaled intent but no firm release date |
| **Purview integration for business terms** | ✅ GA | Microsoft Purview Data Catalog provides business glossary terms that can tag Fabric assets |

**Assessment:** Fabric's ontology capabilities are **maturing but not yet fully realized.** The strongest practical option today is to:
1. Richly annotate your **semantic models** (descriptions, synonyms, display folders) — this is the de facto ontology.
2. Use **Microsoft Purview** to manage a formal business glossary and apply terms to Fabric assets.
3. Wait for the dedicated ontology editor to reach GA before investing in custom ontology tooling.

### 4.4 Relationship to Semantic Models and Medallion Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                    ONTOLOGY LAYER (Logical)                          │
│                                                                     │
│   Business Glossary (Purview)  ──▶  Semantic Model Metadata          │
│   Synonyms, Descriptions       ──▶  Copilot Context                  │
│   Business Rules               ──▶  DAX Measures                     │
│                                                                     │
├─────────────────────────────────────────────────────────────────────┤
│                    GOLD LAYER (Physical)                              │
│                                                                     │
│   DimCustomer, DimProduct, DimDate                                   │
│   FactSales, FactOrders                                              │
│   Star/Snowflake Schema                                              │
│                                                                     │
├─────────────────────────────────────────────────────────────────────┤
│                    SEMANTIC MODEL (Analytical)                        │
│                                                                     │
│   Direct Lake mode on Gold tables                                    │
│   DAX measures, calculated columns                                   │
│   Relationships, hierarchies                                         │
│   Descriptions + synonyms (= working ontology)                       │
│                                                                     │
├─────────────────────────────────────────────────────────────────────┤
│                    COPILOT / FABRIC IQ (Interaction)                  │
│                                                                     │
│   Natural language ──▶ ontology lookup ──▶ DAX/SQL generation         │
│   "Show me revenue by region" → Revenue measure + DimGeography        │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

The ontology sits **above** the Gold layer and the semantic model. It is the bridge between business language and data artifacts. In the medallion architecture:

- **Bronze/Silver** layers have technical metadata (column names, types, lineage).
- **Gold** layer has business-ready structures (star schemas, named dimensions).
- **Semantic model** layer has analytical metadata (measures, relationships, hierarchies).
- **Ontology** layer adds business semantics (synonyms, descriptions, business rules) that enable natural language querying.

### 4.5 Practical Examples of Ontology Definitions

**Example 1: Annotating a Semantic Model (available today)**

In the Power BI semantic model properties:

| Table/Column | Description | Synonyms |
|-------------|-------------|----------|
| `FactSales.Revenue` | Total revenue in USD after discounts and returns | Sales, Income, Turnover, Total Sales |
| `DimCustomer.CustomerName` | Full legal name of the customer account | Client, Account Name, Customer |
| `DimDate.FiscalQuarter` | Fiscal quarter (Q1=Jul-Sep, Q2=Oct-Dec, Q3=Jan-Mar, Q4=Apr-Jun) | Quarter, FQ, Fiscal Quarter |
| `FactSales.OrderDate` | Date the order was placed by the customer | Order Date, Sale Date, Transaction Date |

**Example 2: Purview Business Glossary Terms**

| Term | Definition | Steward | Tagged Assets |
|------|-----------|---------|---------------|
| Revenue | Net revenue after discounts and returns, in reporting currency (USD) | Finance team | FactSales.Revenue, SemanticModel.TotalRevenue |
| Active Customer | Customer with at least one order in the last 12 months | Sales ops | DimCustomer.IsActive, DAX.ActiveCustomerCount |
| Product Category | Top-level product grouping per the official product taxonomy v3 | Product team | DimProduct.Category, DimProduct.CategoryKey |

**Example 3: Hypothetical Fabric IQ Ontology Definition (future)**

```yaml
# Fabric IQ Ontology (hypothetical future format)
ontology:
  name: "AtlMigration Business Ontology"
  version: 1.0

  entities:
    Customer:
      table: Gold.DimCustomer
      key: CustomerKey
      synonyms: [Client, Account, Buyer]
      attributes:
        name: { column: CustomerName, synonyms: [Client Name, Account] }
        region: { column: Region, synonyms: [Territory, Area, Geography] }
        segment: { column: CustomerSegment, synonyms: [Tier, Type] }

    Sale:
      table: Gold.FactSales
      key: SalesKey
      synonyms: [Order, Transaction, Purchase]
      measures:
        revenue: { expression: "SUM(Revenue)", synonyms: [Sales, Income] }
        quantity: { expression: "SUM(Quantity)", synonyms: [Units, Volume] }
      relationships:
        - { entity: Customer, via: CustomerKey, type: many-to-one }
        - { entity: Date, via: OrderDateKey, type: many-to-one }
```

### 4.6 Recommendation

> **📋 DOCUMENT AND DEFER — Invest in semantic model annotations now. Wait for formal ontology tooling.**
>
> **What to do now (Week 3–4):**
> 1. Add **descriptions and synonyms** to every table and column in Gold layer semantic models. This is the best available ontology mechanism today and directly improves Copilot accuracy.
> 2. If Microsoft Purview is available, define **business glossary terms** and tag Fabric assets. This creates an organizational knowledge base independent of individual semantic models.
> 3. Document the business vocabulary (entity names, metric definitions, synonyms) in a shared reference that can be imported into formal ontology tooling when it becomes available.
>
> **What to defer:**
> - Custom ontology tooling or frameworks. The formal Fabric IQ ontology editor is not GA yet, and building custom tooling risks obsolescence.
> - Enterprise knowledge graph investment. Wait for Microsoft's roadmap to materialize.
>
> **Estimated effort for semantic model annotation:** 2–3 days of work by the BI team, integrated into the semantic model build process.

---

## 5. Integration Architecture Summary

### 5.1 How Everything Connects

```
┌───────────────────────────────────────────────────────────────────────────────┐
│                        EXTERNAL DATA SOURCES                                  │
│                                                                               │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────┐  │
│  │ SQL Server  │  │ PostgreSQL  │  │  MongoDB    │  │  Amazon S3          │  │
│  │ (on-prem)   │  │             │  │             │  │  (cloud storage)    │  │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘  └──────────┬──────────┘  │
│         │ Mirroring       │ Mirroring      │ Pipelines          │ Shortcut    │
│         │ (CDC)           │ (WAL)          │ (Copy)             │ (zero-copy) │
│  ┌──────┴──────┐  ┌──────┴──────┐  ┌──────┴──────┐            │             │
│  │ Synapse DW  │  │ Salesforce  │  │  CluedIn    │            │             │
│  │             │  │             │  │  MDM        │            │             │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘            │             │
│         │ Migration       │ Dataflows      │ Golden records    │             │
│         │ Assistant       │ Gen2           │ (Delta export)    │             │
└─────────┼────────────────┼────────────────┼───────────────────┼─────────────┘
          │                │                │                   │
          ▼                ▼                ▼                   ▼
┌───────────────────────────────────────────────────────────────────────────────┐
│                        MICROSOFT FABRIC                                       │
│                                                                               │
│  ┌─────────────────────────────────────────────────────────────────────────┐  │
│  │                        BRONZE LAYER                                     │  │
│  │  Mirrored DBs  │  S3 Shortcuts  │  Salesforce (Dataflows)  │  Pipelines │  │
│  └────────────────────────────────────────────────────┬────────────────────┘  │
│                                                       │                       │
│  ┌────────────────────────────────────────────────────▼────────────────────┐  │
│  │                        SILVER LAYER                                     │  │
│  │  Cleaned + Conformed  │  CluedIn Golden Records  │  Joined + Deduped   │  │
│  └────────────────────────────────────────────────────┬────────────────────┘  │
│                                                       │                       │
│  ┌────────────────────────────────────────────────────▼────────────────────┐  │
│  │                        GOLD LAYER                                       │  │
│  │  Star Schemas  │  Dimensional Models  │  Aggregated Business Tables     │  │
│  └────────────────────────────────────────────────────┬────────────────────┘  │
│                                                       │                       │
│  ┌────────────────────────────────────────────────────▼────────────────────┐  │
│  │                    SEMANTIC + ONTOLOGY LAYER                             │  │
│  │  Direct Lake Models  │  Descriptions/Synonyms  │  Purview Glossary      │  │
│  └────────────────────────────────────────────────────┬────────────────────┘  │
│                                                       │                       │
│  ┌────────────────────────────────────────────────────▼────────────────────┐  │
│  │                        POWER BI                                         │  │
│  │  Reports  │  Dashboards  │  Copilot (natural language via ontology)     │  │
│  └─────────────────────────────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────────────────────────────┘
```

### 5.2 Data Flow Patterns

| Integration | Pattern | Flow | Latency | Compute Cost |
|-------------|---------|------|---------|-------------|
| **SQL Server → Fabric** | Mirroring (CDC) | SQL Server → Gateway → Bronze Lakehouse | ~15 seconds | Free |
| **PostgreSQL → Fabric** | Mirroring (WAL) | PostgreSQL → Bronze Lakehouse | ~15 seconds | Free |
| **MongoDB → Fabric** | Data Pipelines | MongoDB → Copy Activity → Bronze Lakehouse | Scheduled (hourly) | CU per run |
| **Synapse DW → Fabric** | Migration Assistant | Synapse → Fabric Warehouse (Gold) | One-time migration | CU during migration |
| **Amazon S3 → Fabric** | OneLake Shortcut | S3 → Bronze Lakehouse (zero-copy) | Real-time (query-time) | CU at query time only |
| **Salesforce → Fabric** | Dataflows Gen2 | Salesforce API → Power Query → Bronze Lakehouse | 15–60 minute refresh | CU per refresh |
| **CluedIn → Fabric** | Delta Export | CluedIn → OneLake → Silver Lakehouse | Near real-time | Minimal CU |

### 5.3 Critical Path vs Nice-to-Have

| Integration | Classification | Rationale |
|-------------|---------------|-----------|
| **SQL Server Mirroring** | 🔴 Critical Path | Core migration — all operational data depends on this |
| **PostgreSQL Mirroring** | 🔴 Critical Path | Core migration — required data source |
| **MongoDB Pipelines** | 🔴 Critical Path | Core migration — required data source |
| **Synapse DW Migration** | 🔴 Critical Path | Core migration — existing warehouse must be migrated |
| **Amazon S3 Shortcuts** | 🟡 High Priority | Referenced in architecture — needed for Bronze layer completeness |
| **Salesforce Integration** | 🟡 Medium Priority | Extends analytical coverage — not blocking core migration |
| **CluedIn MDM** | ⚪ Nice-to-Have | Parallel evaluation — deferred implementation acceptable |
| **Fabric IQ Ontologies** | ⚪ Nice-to-Have | Forward-looking — adds value but not blocking anything |

---

## Appendix A: Decision Matrix

| Question | Answer |
|----------|--------|
| Should we implement CluedIn for MDM? | **Evaluate in parallel.** Run a POC, decide at Week 4. Don't commit resources until data quality needs are confirmed. |
| Can Salesforce integrate with Fabric? | **Yes.** Multiple options. Dataflows Gen2 is the recommended starting point. |
| What's the best approach for S3 data? | **OneLake shortcuts.** Zero data movement, zero additional cost, minimal setup. |
| Should we invest in Fabric IQ ontologies? | **Not yet.** Annotate semantic models now (descriptions + synonyms). Wait for formal ontology tooling GA. |
| Which integrations block the migration? | **None of these four.** All integration work (S3, Salesforce, CluedIn, IQ) runs in parallel and is non-blocking. |

---

## Appendix B: Prerequisites Checklist

| Prerequisite | Owner | Status | Needed By |
|-------------|-------|--------|-----------|
| AWS IAM credentials for S3 bucket access | Infra team | ⬜ Not started | Week 1 |
| Salesforce connected app (OAuth credentials) | Salesforce admin | ⬜ Not started | Week 2 |
| CluedIn trial environment provisioning | Drummer | ⬜ Not started | Week 2 |
| Microsoft Purview access for business glossary | Data governance | ⬜ Not started | Week 3 |
| S3 bucket path and file format inventory | Data engineering | ⬜ Not started | Week 1 |
| Salesforce object inventory (which objects to sync) | Business analysts | ⬜ Not started | Week 2 |

---

*Document authored by Drummer (Integration Specialist) — AtlMigration project.*
*Last updated: 2026-04-08*
