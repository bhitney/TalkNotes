# Project Plan: Microsoft Fabric Migration

> **Project:** AtlMigration — Enterprise Data Platform Migration to Microsoft Fabric
> **Owner:** Brian Hitney
> **Author:** Avasarala (Documentation & Strategy Lead)
> **Date:** 2026-04-08
> **Status:** ACTIVE
> **Timeline:** 4 weeks (Weeks of April 7 – May 2, 2026)
> **Version:** 1.0

---

## 1. Executive Summary

We are migrating the organization's data estate from multiple disparate sources — SQL Server on-premises, Azure Synapse DW, PostgreSQL, and MongoDB — into a unified Microsoft Fabric platform using a medallion architecture (Bronze/Silver/Gold). Amazon S3 data is accessed via OneLake shortcuts with zero data movement.

### Why Now

- Azure Synapse dedicated SQL pools are approaching end-of-mainstream support. Microsoft is consolidating on Fabric.
- Multiple disconnected data sources create silos, redundant ETL, and inconsistent metrics.
- Fabric's unified compute model (Capacity Units) eliminates the need to manage separate services for ingestion, transformation, warehousing, and BI.

### What We're Delivering

1. **All operational databases** mirrored or migrated into Fabric (Bronze layer).
2. **Cleaned, conformed data** in a Silver layer (Spark-based transforms).
3. **Business-ready dimensional models** in a Gold layer (star/snowflake schemas).
4. **Direct Lake semantic models** on the Gold layer for high-performance analytics.
5. **New Power BI reports** replacing existing reports with modern visuals.
6. **Integrations** with Amazon S3 (shortcuts), CluedIn MDM, Salesforce, and Fabric IQ.
7. **Sentiment analysis** capabilities on text data (favorability scores, keyword extraction).
8. **Comprehensive documentation** of every decision, migration path, and operational procedure.

### Success Criteria (Summary)

| # | Criterion | Measure |
|---|-----------|---------|
| 1 | All source data accessible in Fabric | 100% of identified tables/collections available in Bronze layer |
| 2 | Data quality maintained | Row counts, checksums, and sample validation match within 0.1% |
| 3 | Reports operational | All planned Power BI reports rendering correctly on Direct Lake |
| 4 | Performance acceptable | Report load times < 5 seconds for 95th percentile |
| 5 | Documentation complete | Project plan, cost analysis, runbooks, architecture docs delivered |
| 6 | Stakeholder sign-off | UAT passed, go-live approved by Brian Hitney |

---

## 2. Project Scope

### 2.1 Workstream Inventory

| # | Workstream | Owner | Priority | Dependencies |
|---|-----------|-------|----------|--------------|
| WS-1 | SQL Server on-prem → Fabric (Mirroring) | Naomi | P0 — Critical | Gateway setup |
| WS-2 | Synapse DW → Fabric Warehouse (Migration Assistant + SQL Tran) | Naomi | P0 — Critical | Architecture decisions |
| WS-3 | PostgreSQL → Fabric (Mirroring or Pipelines) | Naomi | P1 — High | Hosting environment confirmation |
| WS-4 | MongoDB → Fabric (Data Pipelines) | Naomi | P1 — High | Document schema analysis |
| WS-5 | Medallion Architecture (Bronze/Silver/Gold) | Holden + Naomi | P0 — Critical | WS-1 through WS-4 (Bronze) |
| WS-6 | Mirroring vs ETL evaluation and implementation | Holden | P0 — Critical | Source inventory |
| WS-7 | Amazon S3 Shortcuts | Drummer | P1 — High | S3 bucket access credentials |
| WS-8 | Direct Lake semantic models on Gold layer | Alex | P0 — Critical | WS-5 (Gold layer) |
| WS-9 | New Power BI reports | Alex | P0 — Critical | WS-8 (semantic models) |
| WS-10 | MDM with CluedIn (parallel, non-blocking) | Drummer | P2 — Medium | CluedIn environment provisioning |
| WS-11 | Salesforce integration evaluation | Drummer | P2 — Medium | Salesforce API credentials |
| WS-12 | Fabric IQ ontologies for business process semantics | Drummer | P2 — Medium | Gold layer schema definitions |
| WS-13 | Sentiment analysis on text data | Alex | P1 — High | Text data in Gold layer |
| WS-14 | Excel integration options | Alex | P1 — High | Semantic models published |

### 2.2 In Scope — Detail

**Database Migrations:**
- **SQL Server on-prem:** Full database mirroring via CDC with on-premises data gateway. Near real-time replication to Bronze Lakehouse. SQL Server 2016–2025 supported.
- **Azure Synapse DW:** Schema migration via Fabric Migration Assistant. T-SQL code translation via SpectralCore SQL Tran. Data copied to Fabric Warehouse at Gold layer.
- **PostgreSQL:** Mirroring via WAL-based CDC (if Azure DB for PostgreSQL Flexible Server) or Data Pipelines (if self-hosted). Requires Week 1 environment confirmation.
- **MongoDB:** Batch ingestion via Data Pipelines Copy Activity with MongoDB Atlas connector. Spark notebooks for complex document flattening.

**Architecture:**
- **Medallion layers:** Three Lakehouse artifacts (Bronze, Silver, Gold) plus one Fabric Warehouse for Synapse T-SQL workloads.
- **Mirroring vs ETL:** Mirroring-first strategy. ETL only where mirroring is unavailable (MongoDB) or requires transformation on ingest.

**Analytics & BI:**
- **Direct Lake semantic models** on Gold layer Delta tables — no Import mode refresh cost.
- **Power BI reports** built on semantic models. Modern visuals, row-level security.
- **Sentiment analysis:** Favorability scoring and keyword extraction on text columns using Fabric notebooks (Python NLP libraries or Azure AI Services).
- **Excel integration:** Analyze in Excel, connected to published semantic models.

**Integrations:**
- **Amazon S3:** OneLake shortcuts. Zero data movement. Shortcut caching enabled (7-day retention).
- **CluedIn MDM:** Parallel track. Data quality and master data matching. Non-blocking to core migration.
- **Salesforce:** Evaluation of Salesforce connector in Fabric Data Factory. Integration scope to be determined by end of Week 2.
- **Fabric IQ:** Business process ontologies for semantic layer enrichment. Defines relationships between business concepts and Gold layer entities.

### 2.3 Out of Scope (Phase 2+)

- Real-time streaming ingestion (Eventstream/KQL Database)
- Machine learning model training and deployment (beyond sentiment analysis)
- Data governance framework (Microsoft Purview integration)
- Multi-tenant capacity architecture
- Reverse ETL / data activation to operational systems

---

## 3. Four-Week Timeline

### Week 1: Discovery, Architecture, Environment Setup (Apr 7–11)

| Day | Activity | Owner | Deliverable |
|-----|----------|-------|-------------|
| Mon | Kick-off meeting. Confirm scope, access, credentials. | All | Signed-off scope document |
| Mon | Provision Fabric workspace and F64 capacity in Azure | Holden | Fabric environment live |
| Mon | Install and configure on-premises data gateway (HA mode, 2+ nodes) | Naomi | Gateway operational |
| Tue | Set up Bronze, Silver, Gold Lakehouses | Naomi | 3 Lakehouse artifacts created |
| Tue | Create Fabric Warehouse for Synapse T-SQL workloads | Naomi | Warehouse artifact created |
| Tue | Confirm PostgreSQL hosting (Azure-managed vs self-hosted) | Drummer | Hosting decision documented |
| Wed | Configure SQL Server mirroring (pilot: 1-2 databases) | Naomi | First mirrored tables visible in Bronze |
| Wed | Run SpectralCore SQL Tran compatibility assessment on Synapse DW | Naomi | Compatibility report generated |
| Thu | Prototype MongoDB ingestion (Copy Activity, 1-2 collections) | Naomi | Test data in Bronze Lakehouse |
| Thu | Create S3 shortcut configuration (test with 1 bucket) | Drummer | S3 data visible in Bronze via shortcut |
| Fri | Architecture review — team alignment on decisions | Holden (all) | Architecture proposal finalized |
| Fri | CluedIn environment setup begins (parallel) | Drummer | CluedIn sandbox provisioned |

**Week 1 Milestone:** ✅ Fabric environment operational. First data flowing from SQL Server to Bronze. Synapse compatibility assessment complete.

### Week 2: Bronze/Silver Layer Implementation, Migration Execution (Apr 14–18)

| Day | Activity | Owner | Deliverable |
|-----|----------|-------|-------------|
| Mon | Complete SQL Server mirroring — all production databases | Naomi | All SQL tables in Bronze |
| Mon | Configure PostgreSQL mirroring or Data Pipelines | Naomi | PostgreSQL data in Bronze |
| Tue | Complete MongoDB Data Pipeline setup (full load + incremental) | Naomi | MongoDB collections in Bronze |
| Tue | Create all S3 shortcuts in Bronze Lakehouse | Drummer | All S3 buckets accessible |
| Wed | Begin Silver layer Spark notebooks — type casting, dedup, null handling | Naomi | First Silver tables populated |
| Wed | Begin Synapse DW schema migration (Migration Assistant) | Naomi | DDL converted to Fabric Warehouse |
| Thu | Begin T-SQL code translation (SQL Tran output → manual fixes) | Naomi | Stored procs migrated (first batch) |
| Thu | Salesforce connector evaluation — test connectivity | Drummer | Salesforce feasibility documented |
| Fri | Silver layer progress review — data quality checks | Naomi + Holden | Silver QA report |
| Fri | Evaluate Fabric IQ ontology requirements | Drummer | Ontology design draft |

**Week 2 Milestone:** ✅ All source data landed in Bronze. Silver layer transformations 50% complete. Synapse schema migration underway.

### Week 3: Gold Layer, Semantic Models, Integrations (Apr 21–25)

| Day | Activity | Owner | Deliverable |
|-----|----------|-------|-------------|
| Mon | Complete Silver layer transforms | Naomi | All Silver tables validated |
| Mon | Begin Gold layer dimensional modeling (star schemas) | Naomi + Holden | Dimension and fact tables designed |
| Tue | Build Gold layer tables (Spark notebooks or Warehouse T-SQL) | Naomi | Gold tables populated |
| Tue | Complete Synapse T-SQL migration and run validation queries | Naomi | Synapse vs Fabric comparison report |
| Wed | Create Direct Lake semantic models on Gold layer | Alex | Semantic models published |
| Wed | Configure row-level security in semantic models | Alex | RLS rules applied |
| Thu | Begin Power BI report development (initial reports) | Alex | First reports in development |
| Thu | Sentiment analysis notebook — favorability scores, keywords | Alex | Sentiment pipeline functional |
| Fri | Excel integration testing — Analyze in Excel | Alex | Excel connectivity verified |
| Fri | CluedIn MDM matching rules configuration | Drummer | MDM rules configured |

**Week 3 Milestone:** ✅ Gold layer complete. Semantic models published. First Power BI reports rendering. Sentiment analysis operational.

### Week 4: Reports, Testing, Documentation, Cutover (Apr 28–May 2)

| Day | Activity | Owner | Deliverable |
|-----|----------|-------|-------------|
| Mon | Complete all Power BI reports | Alex | Full report suite |
| Mon | End-to-end data validation (source → Bronze → Silver → Gold → PBI) | Naomi | Validation report |
| Tue | Performance testing — report load times, CU utilization | Alex + Holden | Performance test results |
| Tue | CU optimization — right-size Spark, adjust schedules | Holden | Optimization applied |
| Wed | Security and permissions audit | Holden | Security audit report |
| Wed | Fabric IQ ontology finalization | Drummer | Ontologies published |
| Thu | Stakeholder UAT sessions | All | UAT feedback collected |
| Thu | Documentation finalization — all docs reviewed and approved | Avasarala | Complete documentation set |
| Fri | Go/No-Go decision meeting | All + Brian | Go-live decision |
| Fri | Cutover execution (if approved) — update DNS, retire legacy | Holden + Naomi | Production cutover complete |

**Week 4 Milestone:** ✅ All reports live. UAT passed. Documentation delivered. Go-live decision made.

---

## 4. Roles and Responsibilities (RACI Matrix)

### Team Roster

| Name | Role | Core Expertise |
|------|------|---------------|
| **Brian Hitney** | Project Owner / Sponsor | Executive authority, budget approval |
| **Holden** | Lead / Architect | Architecture decisions, Fabric design, code reviews |
| **Naomi** | Data Engineer | Migrations, pipelines, Spark notebooks, medallion layers |
| **Alex** | BI & Analytics Engineer | Semantic models, Power BI reports, Excel integration, sentiment analysis |
| **Avasarala** | Documentation & Strategy Lead | Project plan, cost analysis, risk register, all documentation |
| **Drummer** | Integration Specialist | CluedIn MDM, Salesforce, S3 shortcuts, Fabric IQ ontologies |

### RACI Matrix

**R** = Responsible (does the work) | **A** = Accountable (owns the outcome) | **C** = Consulted | **I** = Informed

| Activity | Brian | Holden | Naomi | Alex | Avasarala | Drummer |
|----------|-------|--------|-------|------|-----------|---------|
| **Architecture decisions** | I | **A/R** | C | C | C | C |
| **Fabric capacity provisioning** | A | **R** | I | I | C | I |
| **On-prem gateway setup** | I | C | **A/R** | I | I | I |
| **SQL Server mirroring** | I | C | **A/R** | I | I | I |
| **Synapse DW migration** | I | C | **A/R** | I | I | I |
| **PostgreSQL migration** | I | C | **A/R** | I | I | I |
| **MongoDB migration** | I | C | **A/R** | I | I | I |
| **Bronze layer setup** | I | C | **A/R** | I | I | I |
| **Silver layer transforms** | I | R | **A/R** | I | I | I |
| **Gold layer modeling** | I | **A/R** | R | C | I | I |
| **Direct Lake semantic models** | I | C | I | **A/R** | I | I |
| **Power BI reports** | I | C | I | **A/R** | I | I |
| **Sentiment analysis** | I | C | I | **A/R** | I | I |
| **Excel integration** | I | I | I | **A/R** | I | I |
| **S3 shortcuts** | I | C | I | I | I | **A/R** |
| **CluedIn MDM** | I | C | I | I | I | **A/R** |
| **Salesforce integration** | I | C | I | I | I | **A/R** |
| **Fabric IQ ontologies** | I | C | I | C | I | **A/R** |
| **Project plan & docs** | A | C | C | C | **R** | C |
| **Cost analysis** | A | C | I | I | **R** | I |
| **Risk management** | I | C | C | C | **A/R** | C |
| **UAT coordination** | **A** | R | R | R | I | R |
| **Go-live decision** | **A** | R | C | C | C | C |
| **CU monitoring & optimization** | I | **A/R** | C | C | C | I |

---

## 5. Dependencies and Critical Path

### Dependency Map

```
Architecture Decisions (Week 1, Day 1)
    │
    ├──▶ Fabric Capacity Provisioning ──▶ All work depends on this
    │
    ├──▶ Gateway Installation ──▶ SQL Server Mirroring ──▶ Bronze Layer
    │                          ──▶ PostgreSQL Mirroring ──▶ Bronze Layer
    │
    ├──▶ PostgreSQL Hosting Confirmation ──▶ Migration Approach Selection
    │
    └──▶ Synapse Compatibility Assessment ──▶ T-SQL Migration Scope

Bronze Layer (all sources)
    │
    └──▶ Silver Layer Transforms
            │
            └──▶ Gold Layer Dimensional Models
                    │
                    ├──▶ Direct Lake Semantic Models
                    │       │
                    │       ├──▶ Power BI Reports
                    │       ├──▶ Excel Integration
                    │       └──▶ Sentiment Analysis (on text columns)
                    │
                    └──▶ Fabric IQ Ontologies (on Gold schema)

CluedIn MDM ──▶ (parallel, non-blocking — feeds into Silver/Gold as enrichment)
Salesforce  ──▶ (evaluation — feeds into scope decision for Phase 1 or 2)
S3 Shortcuts ──▶ (independent — lands in Bronze, flows through medallion)
```

### Critical Path

The **critical path** — the longest sequence of dependent activities — is:

```
Gateway Install → SQL Server Mirroring → Bronze Complete → Silver Transforms →
Gold Models → Direct Lake Semantic Models → Power BI Reports → UAT → Go-Live
```

**Estimated critical path duration:** 18 working days (3.5 weeks). This leaves 2 days of buffer.

### Key Blockers

| Blocker | Impact | Resolution Timeline |
|---------|--------|-------------------|
| Fabric F64 capacity not provisioned | Blocks everything | Must complete Day 1 |
| On-prem gateway not installed | Blocks SQL Server + PostgreSQL mirroring | Must complete Day 1–2 |
| PostgreSQL hosting unknown | Blocks migration approach selection | Must confirm Day 2 |
| Synapse T-SQL compatibility issues | Could extend Synapse migration | Assess by Day 3, mitigate in Week 2 |
| S3 credentials unavailable | Blocks S3 shortcuts | Drummer to secure by Day 3 |

---

## 6. Risk Register

| # | Risk | Likelihood | Impact | Risk Score | Mitigation | Owner | Contingency |
|---|------|-----------|--------|------------|------------|-------|-------------|
| R1 | **On-prem data gateway instability** — Single point of failure for SQL Server and PostgreSQL mirroring | Medium | High | 🔴 High | Deploy gateway in HA mode (2+ nodes). Monitor gateway health metrics. | Naomi | Fall back to Data Pipelines for batch ingestion if gateway fails. |
| R2 | **Synapse T-SQL incompatibilities** — Not all Synapse constructs translate to Fabric | High | Medium | 🔴 High | Run SQL Tran compatibility report Day 3. Identify manual refactoring scope. Budget 30% of Synapse time for manual fixes. | Naomi | Deprioritize complex stored procs. Rewrite critical ones in Spark. |
| R3 | **MongoDB document complexity** — Deeply nested documents may not flatten via Copy Activity | Medium | Medium | 🟡 Medium | Prototype with representative collections in Week 1. | Naomi | Fall back to Spark + PyMongo for custom flattening. |
| R4 | **CU throttling under load** — Concurrent mirroring + transforms + queries exceed capacity | Medium | High | 🔴 High | Start with F64. Monitor via Fabric Capacity Metrics app. Set CU overage protection. | Holden | Scale up to F128. Stagger workloads to off-peak hours. |
| R5 | **4-week timeline pressure** — Scope creep or unexpected data quality issues | High | High | 🔴 Critical | Strict scope control. Prioritize: (1) Mirroring, (2) Synapse, (3) Gold models, (4) Reports. Defer P2 items if needed. | Avasarala | Defer CluedIn, Salesforce, Fabric IQ to Phase 2. |
| R6 | **Direct Lake limitations** — Some DAX functions or data shapes incompatible | Low | Medium | 🟢 Low | Test semantic models mid-Week 3. | Alex | Fall back to Import mode for specific models. |
| R7 | **S3 egress costs** — Uncontrolled shortcut reads generate high AWS bills | Medium | Medium | 🟡 Medium | Enable shortcut caching (7-day retention). Monitor AWS Cost Explorer. | Drummer | Materialize high-frequency tables to OneLake. |
| R8 | **PostgreSQL hosting uncertainty** — If self-hosted, mirroring unavailable | Unknown | Medium | 🟡 Medium | Confirm hosting Day 2. If self-hosted, pivot to Data Pipelines immediately. | Drummer | Data Pipeline approach is validated and ready. |
| R9 | **CluedIn MDM delays** — MDM platform provisioning may take longer than expected | Medium | Low | 🟢 Low | Start provisioning Day 1. MDM is non-blocking to core migration. | Drummer | Defer MDM integration to Phase 2. |
| R10 | **Key person dependency** — Single team member on critical workstreams | Medium | High | 🔴 High | Cross-train within the team. Document all procedures. | Avasarala | Redistribute work. Extend timeline for that workstream. |

### Risk Scoring Legend

- **Likelihood:** Low / Medium / High
- **Impact:** Low / Medium / High
- **Risk Score:** 🟢 Low (accept) / 🟡 Medium (monitor) / 🔴 High (mitigate actively) / 🔴 Critical (escalate immediately)

---

## 7. Success Criteria

### 7.1 Migration Completeness

| Source | Success Measure | Validation Method |
|--------|----------------|-------------------|
| SQL Server on-prem | All identified databases mirrored to Bronze | Table count match, row count match, spot-check 5% of rows |
| Synapse DW | Schema + data + stored procs migrated to Fabric Warehouse | DDL comparison, query result comparison (10 key queries) |
| PostgreSQL | All tables replicated to Bronze | Row count match, schema comparison |
| MongoDB | All collections ingested to Bronze | Document count match, sample document comparison |
| Amazon S3 | All specified buckets accessible via shortcuts | File listing comparison, sample file read verification |

### 7.2 Architecture Validation

| Criterion | Measure |
|-----------|---------|
| Bronze layer complete | All source data landed as Delta tables or shortcuts |
| Silver layer complete | All Bronze tables transformed: deduped, typed, cleaned |
| Gold layer complete | Star/snowflake dimensional models built and validated |
| Direct Lake models | Semantic models connected to Gold, responsive under load |

### 7.3 BI and Analytics

| Criterion | Measure |
|-----------|---------|
| Power BI reports | All planned reports rendering correctly |
| Report performance | 95th percentile load time < 5 seconds |
| Excel integration | Analyze in Excel connected and functional |
| Sentiment analysis | Favorability scores and keywords generated on target text columns |

### 7.4 Operational Readiness

| Criterion | Measure |
|-----------|---------|
| CU utilization | Sustained utilization < 80% of F64 capacity |
| Mirroring health | All mirrored databases in sync, lag < 30 seconds |
| Documentation | All project docs delivered and reviewed |
| Runbook | Operational runbook covering monitoring, troubleshooting, scaling |
| Security | Workspace permissions and RLS configured and audited |

### 7.5 Stakeholder Approval

- UAT sessions conducted with designated business users
- All critical defects resolved
- Go/No-Go decision approved by Brian Hitney
- Cutover plan executed (or scheduled)

---

## 8. Communication Plan

| Audience | Cadence | Format | Owner |
|----------|---------|--------|-------|
| Project team | Daily standup (15 min) | Teams call | Holden |
| Brian Hitney (sponsor) | Weekly status report | Written report + 30-min call | Avasarala |
| Stakeholders | End of Week 2 + Week 4 | Demo + written update | Alex + Avasarala |
| Technical reviewers | Architecture review (Week 1), Code review (ongoing) | PR reviews | Holden |

---

## 9. Quality Gates

| Gate | When | Pass Criteria | Decision Maker |
|------|------|---------------|----------------|
| **G1: Architecture Approved** | End of Week 1 | Architecture proposal reviewed and signed off | Holden + Brian |
| **G2: Bronze Complete** | End of Week 2, Day 2 | All sources landing in Bronze Lakehouse | Naomi + Holden |
| **G3: Silver/Gold Validated** | End of Week 3 | Data quality checks pass, dimensional models verified | Naomi + Holden |
| **G4: Reports Ready** | Week 4, Day 2 | All reports rendering on Direct Lake, performance tested | Alex + Holden |
| **G5: Go/No-Go** | Week 4, Day 5 | UAT passed, all critical issues resolved, docs complete | Brian Hitney |

---

## 10. Document Inventory

| Document | Owner | Status | Location |
|----------|-------|--------|----------|
| Architecture Proposal | Holden | ✅ Draft complete | `architecture-proposal.md` |
| Project Plan (this document) | Avasarala | ✅ v1.0 | `project-plan.md` |
| Fabric Cost Analysis | Avasarala | ✅ v1.0 | `fabric-cost-analysis.md` |
| Migration Runbook | Naomi | 📝 Planned | `docs/migration-runbook.md` |
| Semantic Model Design | Alex | 📝 Planned | `docs/semantic-model-design.md` |
| Integration Playbook | Drummer | 📝 Planned | `docs/integration-playbook.md` |
| UAT Test Plan | All | 📝 Planned | `docs/uat-test-plan.md` |

---

*This is a living document. Updates will be tracked via git commits with clear change descriptions.*

*Prepared by Avasarala — Documentation & Strategy Lead*
*"If it's not documented, it didn't happen. If it's not planned, it won't happen."*
