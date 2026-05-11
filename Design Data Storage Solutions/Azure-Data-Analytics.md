# Azure Data & Analytics Storage Cheat Sheet

## AZ-305: Designing Data Storage Solutions – Analytics & Big Data

> 📝 **Hands-On Labs:** [Azure Data & Analytics Labs](../Labs/Azure-Data-Analytics-Labs.md)

> **Perspective:** Senior Cloud Solution Architect | AZ-305 Exam Focus
> **Last Updated:** 2025

---

## Table of Contents

1. [Azure Data Lake Storage Gen2](#1-azure-data-lake-storage-gen2)
2. [Azure Synapse Analytics](#2-azure-synapse-analytics)
3. [Microsoft Fabric](#3-microsoft-fabric)
4. [Azure Data Factory](#4-azure-data-factory)
5. [Azure Databricks](#5-azure-databricks)
6. [Azure Stream Analytics](#6-azure-stream-analytics)
7. [Azure HDInsight](#7-azure-hdinsight)
8. [Data Architecture Patterns](#8-data-architecture-patterns)
9. [Data Governance](#9-data-governance)
10. [Security & Access](#10-security--access)
11. [AZ-305 Decision Matrix](#11-az-305-decision-matrix)
12. [AZ-305 Decision Scenarios](#12-az-305-decision-scenarios)
13. [Quick Reference Trigger Table](#13-quick-reference-trigger-table)
14. [Cost Optimization](#14-cost-optimization)
15. [Integration Patterns](#15-integration-patterns)

---

## 1. Azure Data Lake Storage Gen2

### Overview

ADLS Gen2 = Azure Blob Storage + **Hierarchical Namespace (HNS)** — purpose-built for big data analytics workloads.

### Key Features

| Feature | Description |
|---------|-------------|
| **Hierarchical Namespace** | True directory structure (not flat blob namespace), enables atomic directory operations |
| **POSIX ACLs** | Fine-grained access control at file/folder level (owner, owning group, other) |
| **Multi-protocol Access** | Blob API (`wasbs://`), DFS API (`abfss://`), NFS 3.0 |
| **Hadoop Compatible** | `abfss://` driver works with Spark, HDInsight, Databricks, Synapse |
| **Tiering** | Hot, Cool, Cold, Archive tiers available |
| **Unlimited Scale** | No limit on account size, file sizes up to 190.7 TiB |

### Hierarchical Namespace Deep Dive

```
Without HNS (Flat):  container/path/to/file.parquet  (rename = copy + delete all blobs with prefix)
With HNS (ADLS Gen2): container/path/to/file.parquet  (rename = metadata operation, O(1))
```

**Why HNS matters for analytics:**
- Atomic directory rename/delete (critical for Spark job output commits)
- 2-6x performance improvement for analytics workloads
- POSIX ACLs only available with HNS enabled

### POSIX ACL Model

```
Access ACL:   Applied to a specific file or directory
Default ACL:  Applied to a directory; inherited by new child objects

Permissions:  r (read), w (write), x (execute/traverse)
Applied to:   Owning User, Owning Group, Named Users, Named Groups, Other
```

> ⚠️ **AZ-305 Tip:** POSIX ACLs are evaluated AFTER Azure RBAC. If RBAC grants access, ACLs are not checked. Use ACLs for fine-grained control within a data lake when multiple teams share a storage account.

### ADLS Gen2 vs Regular Blob Storage

| Criteria | ADLS Gen2 (HNS Enabled) | Regular Blob Storage |
|----------|--------------------------|---------------------|
| **Directory operations** | Atomic (O(1)) | Simulated (O(n)) |
| **ACL granularity** | File/folder POSIX ACLs | Container-level only |
| **Analytics performance** | Optimized | Standard |
| **Hadoop/Spark** | Native `abfss://` driver | `wasbs://` (slower) |
| **Cost** | ~Same (slight premium for transactions) | Standard |
| **NFS 3.0** | Supported | Not supported |
| **Object replication** | Not supported with HNS | Supported |
| **Blob versioning** | Supported | Supported |
| **Soft delete** | Supported | Supported |
| **Static website** | Not supported with HNS | Supported |

### When to Use ADLS Gen2

✅ **Use ADLS Gen2 when:**
- Building a data lake for analytics (Spark, Synapse, Databricks)
- Need fine-grained ACLs for multi-team data access
- Running big data workloads that perform directory-level operations
- Need atomic rename operations (Spark output commits)
- Implementing medallion architecture (Bronze/Silver/Gold)

❌ **Use regular Blob Storage when:**
- Application workloads (web apps, media storage)
- Need object replication across regions
- Static website hosting
- Simple flat-file storage without analytics needs

### Integration with Analytics Services

```
┌─────────────────────────────────────────────────────┐
│                  ADLS Gen2                           │
├─────────────────────────────────────────────────────┤
│                                                     │
│  ┌──────────┐  ┌──────────┐  ┌──────────────────┐  │
│  │ Synapse  │  │Databricks│  │   HDInsight      │  │
│  │Analytics │  │          │  │                  │  │
│  └──────────┘  └──────────┘  └──────────────────┘  │
│                                                     │
│  ┌──────────┐  ┌──────────┐  ┌──────────────────┐  │
│  │  ADF     │  │  Fabric  │  │ Stream Analytics │  │
│  │          │  │(OneLake) │  │                  │  │
│  └──────────┘  └──────────┘  └──────────────────┘  │
│                                                     │
│  ┌──────────┐  ┌──────────┐  ┌──────────────────┐  │
│  │  Purview │  │Power BI  │  │ Azure ML         │  │
│  │          │  │          │  │                  │  │
│  └──────────┘  └──────────┘  └──────────────────┘  │
└─────────────────────────────────────────────────────┘
```

---

## 2. Azure Synapse Analytics

### Overview

Unified analytics platform combining data warehousing, big data processing, and data integration in a single workspace.

### Components

| Component | Description | Billing Model |
|-----------|-------------|---------------|
| **Dedicated SQL Pool** | Massively Parallel Processing (MPP) data warehouse | DWU (Data Warehouse Units) – always-on or paused |
| **Serverless SQL Pool** | Query data in-place without loading | Per TB processed |
| **Apache Spark Pool** | Managed Spark clusters for big data | Per vCore-hour (auto-pause) |
| **Data Explorer Pool** | Near real-time log/telemetry analytics (Kusto) | Compute + storage |
| **Pipelines** | Data orchestration (based on ADF) | Activity runs + integration runtime |

### Dedicated SQL Pool (formerly SQL DW)

```
Architecture: MPP (Massively Parallel Processing)
- Control node: Query planning, optimization
- Compute nodes: Parallel query execution (1-60 nodes)
- Data Movement Service (DMS): Data redistribution between nodes

Distribution types:
- Hash:       Best for large fact tables (join/aggregation columns)
- Round-robin: Default, good for staging tables
- Replicate:  Best for small dimension tables (<2GB)

Performance tiers: DW100c → DW30000c
```

> 🎯 **AZ-305 Tip:** Dedicated SQL pools are best when you need predictable performance for complex queries on structured data with known patterns. Think "enterprise data warehouse."

### Serverless SQL Pool

```
- No infrastructure to manage
- Pay per TB of data processed (~$5/TB)
- Query Parquet, CSV, JSON directly in ADLS Gen2
- Create external tables/views (logical data warehouse)
- T-SQL syntax
- Cannot store data (read-only from external sources)
- Max concurrency: 1000 concurrent queries
```

**Best for:**
- Ad-hoc exploration of data lake files
- Creating a logical data warehouse over ADLS Gen2
- Light transformation and data discovery
- Cost-effective for infrequent queries

### Spark Pool

```
- Managed Apache Spark (auto-provisioned clusters)
- Auto-pause after configurable idle timeout (5-90 min)
- Auto-scale (3-200 nodes)
- Supports: PySpark, Scala, .NET Spark, SparkSQL
- Delta Lake support built-in
- Shared metadata: Spark tables visible in serverless SQL
```

**Best for:**
- Complex data transformations (ETL/ELT)
- Machine learning model training
- Data exploration with notebooks
- Processing unstructured/semi-structured data

### When to Use Which Pool

| Scenario | Pool Type |
|----------|-----------|
| Enterprise data warehouse with predictable queries | Dedicated SQL |
| Ad-hoc exploration of data lake | Serverless SQL |
| Complex ETL with PySpark/Scala | Spark |
| Real-time log analytics | Data Explorer |
| Simple SQL-based transformations on lake data | Serverless SQL |
| ML model training on large datasets | Spark |
| Low-latency dashboard queries on structured data | Dedicated SQL |
| Infrequent queries, cost-sensitive | Serverless SQL |

### Synapse Pipelines

- Built on Azure Data Factory technology
- Integrated within the Synapse workspace
- Orchestrate Spark notebooks, SQL scripts, data flows
- Same connectors and activities as ADF
- Tight integration with Synapse pools

> ⚠️ **AZ-305 Tip:** If the scenario involves ONLY data integration/movement without needing Synapse compute, use standalone ADF. If the scenario involves unified analytics + integration, use Synapse Pipelines.

---

## 3. Microsoft Fabric

### Overview

Microsoft Fabric is a **unified SaaS analytics platform** that brings together data engineering, data science, real-time analytics, data warehousing, and business intelligence into a single product with a single billing model.

### Core Components

| Component | Description |
|-----------|-------------|
| **OneLake** | Single, unified data lake for the entire organization (built on ADLS Gen2) |
| **Lakehouse** | Combines data lake flexibility with warehouse structure (Delta/Parquet) |
| **Data Warehouse** | Full T-SQL data warehouse with cross-database queries |
| **Real-Time Analytics** | KQL-based analytics for streaming/event data |
| **Data Factory** | Visual data integration and orchestration |
| **Data Science** | ML model training with MLflow integration |
| **Power BI** | Business intelligence and reporting |
| **Data Activator** | Event-driven triggers and alerts |

### OneLake

```
Key Principles:
├── ONE data lake for the entire organization
├── Governed by default (security, compliance built-in)
├── Open data formats (Delta/Parquet)
├── Shortcuts: Reference external data without copying
│   ├── ADLS Gen2 shortcuts
│   ├── S3 shortcuts (AWS)
│   └── Google Cloud Storage shortcuts
└── Automatic Delta format optimization
```

**OneLake vs ADLS Gen2:**

| Feature | OneLake | ADLS Gen2 |
|---------|---------|-----------|
| Management | SaaS (zero config) | PaaS (you manage) |
| Governance | Built-in (Purview integrated) | Must configure |
| Multi-cloud shortcuts | Yes (S3, GCS) | No |
| Billing | Fabric capacity | Storage account |
| Open format | Delta Lake by default | Any format |
| Direct Lake mode | Yes (Power BI) | No |

### Lakehouse vs Data Warehouse in Fabric

| Feature | Lakehouse | Data Warehouse |
|---------|-----------|----------------|
| **Engine** | Spark + SQL endpoint | T-SQL MPP engine |
| **Format** | Delta Lake (open) | Proprietary storage |
| **Schema** | Schema-on-read | Schema-on-write |
| **Language** | PySpark, Spark SQL, T-SQL (read) | T-SQL (full DML) |
| **Best for** | Data engineering, ML, flexible schemas | Traditional BI, complex T-SQL |
| **Multi-table transactions** | Limited | Full ACID |

### Licensing (Capacity Units)

```
Fabric Capacity SKUs:
┌──────────┬───────────┬─────────────────────────────────┐
│ SKU      │ CU        │ Typical Use Case                │
├──────────┼───────────┼─────────────────────────────────┤
│ F2       │ 2 CU      │ Dev/test                        │
│ F4       │ 4 CU      │ Small team                      │
│ F8       │ 8 CU      │ Department                      │
│ F16      │ 16 CU     │ Large department                │
│ F32      │ 32 CU     │ Business unit                   │
│ F64      │ 64 CU     │ Enterprise (small)              │
│ F128     │ 128 CU    │ Enterprise (medium)             │
│ F256     │ 256 CU    │ Enterprise (large)              │
│ F512     │ 512 CU    │ Enterprise (very large)         │
│ F1024    │ 1024 CU   │ Enterprise (massive)            │
│ F2048    │ 2048 CU   │ Enterprise (maximum)            │
└──────────┴───────────┴─────────────────────────────────┘

Key billing concepts:
- Capacity Units (CU) are shared across ALL Fabric workloads
- Burstable: Can temporarily exceed capacity
- Smoothing: Spikes averaged over time windows
- Auto-pause available for F SKUs
- Pay-per-use via Fabric trial or PAYG available
```

> 🎯 **AZ-305 Tip:** Fabric is the answer when the scenario mentions "unified analytics," "single platform for data teams," "reduce data silos," or "self-service analytics with governance."

---

## 4. Azure Data Factory

### Overview

Cloud-based data integration service for creating ETL/ELT workflows at scale. Orchestrates data movement and transformation across 100+ connectors.

### Key Concepts

| Concept | Description |
|---------|-------------|
| **Pipeline** | Logical grouping of activities |
| **Activity** | A unit of work (copy, transform, control flow) |
| **Dataset** | Named reference to data |
| **Linked Service** | Connection string/credentials to data source |
| **Integration Runtime** | Compute infrastructure for activities |
| **Trigger** | Defines when a pipeline executes |
| **Data Flow** | Visual Spark-based transformation (Mapping/Wrangling) |

### Integration Runtimes

| Runtime Type | Location | Use Case |
|--------------|----------|----------|
| **Azure IR** | Azure regions (auto-resolve or fixed) | Cloud-to-cloud data movement, Data Flows |
| **Self-hosted IR** | On-premises / private VNet | On-prem data sources, private networks |
| **Azure-SSIS IR** | Azure (managed cluster) | Lift-and-shift existing SSIS packages |

```
Decision: Which Integration Runtime?
├── Data source is in Azure/public cloud? → Azure IR
├── Data source is on-premises/private network? → Self-hosted IR
├── Existing SSIS packages to migrate? → Azure-SSIS IR
└── Need Spark-based transformations? → Azure IR (Data Flows)
```

### ADF vs Synapse Pipelines

| Feature | Azure Data Factory | Synapse Pipelines |
|---------|-------------------|-------------------|
| **Deployment** | Standalone service | Integrated in Synapse workspace |
| **Connectors** | 100+ | Same (shared codebase) |
| **Data Flows** | Mapping + Wrangling | Mapping only |
| **Git integration** | Full (ARM templates) | Synapse workspace Git |
| **SSIS IR** | Yes | Yes |
| **Spark notebooks** | No (must call externally) | Native orchestration |
| **Dedicated SQL scripts** | No (stored procedure activity) | Native orchestration |
| **Monitoring** | ADF Monitor | Synapse Studio Monitor |
| **CI/CD** | ARM/Bicep deployment | Synapse workspace publish |
| **Cross-workspace** | Share Self-hosted IR | Share IR within workspace |

> 🎯 **AZ-305 Tip:** Use standalone ADF when you need a centralized data integration hub that orchestrates across MULTIPLE analytics platforms. Use Synapse Pipelines when data integration is part of a unified Synapse analytics solution.

### Trigger Types

| Trigger | Description |
|---------|-------------|
| **Schedule** | Time-based (cron-like) |
| **Tumbling Window** | Fixed-size, non-overlapping time intervals with retry/dependency |
| **Event (Blob)** | Fires when blob created/deleted in storage |
| **Event (Custom)** | Fires on custom Event Grid events |

---

## 5. Azure Databricks

### Overview

Fully managed Apache Spark platform optimized for Azure, with collaborative notebooks, ML lifecycle management, and Delta Lake as the foundation.

### Key Components

| Component | Description |
|-----------|-------------|
| **Workspace** | Collaborative environment for notebooks, repos, jobs |
| **Clusters** | Managed Spark compute (interactive or job clusters) |
| **Delta Lake** | Open-source storage layer (ACID transactions on data lake) |
| **Unity Catalog** | Unified governance for data and AI assets |
| **MLflow** | End-to-end ML lifecycle management |
| **SQL Warehouses** | Serverless/Pro/Classic SQL endpoints for BI queries |
| **Workflows** | Job orchestration and scheduling |

### Unity Catalog

```
Hierarchy:
Metastore (account-level)
└── Catalog (logical grouping, e.g., "production", "development")
    └── Schema (database)
        └── Tables / Views / Volumes / Functions / Models
            └── Columns (with column-level security)

Key Features:
- Centralized access control (grants at any level)
- Data lineage tracking
- Data sharing (Delta Sharing – open protocol)
- Audit logging
- Row-level and column-level security
- Attribute-based access control (ABAC)
```

### Delta Lake

```
Delta Lake = Parquet files + Transaction Log (_delta_log/)

Key capabilities:
├── ACID transactions (serializable isolation)
├── Schema enforcement & evolution
├── Time travel (query historical versions)
├── MERGE/UPDATE/DELETE operations
├── Optimistic concurrency control
├── Z-ORDER optimization (multi-dimensional clustering)
├── OPTIMIZE (file compaction)
├── VACUUM (remove old files)
└── Change Data Feed (CDC)
```

### Databricks vs Synapse

| Feature | Azure Databricks | Azure Synapse |
|---------|-----------------|---------------|
| **Primary strength** | Advanced Spark analytics, ML, Delta Lake | Unified analytics + DW |
| **SQL DW** | SQL Warehouses (serverless) | Dedicated SQL Pool (MPP) |
| **Spark** | Full Spark (latest versions, optimized runtime) | Managed Spark pools |
| **ML** | MLflow, AutoML, Feature Store | SparkML (basic) |
| **Real-time** | Structured Streaming | Data Explorer Pool |
| **Governance** | Unity Catalog | Purview integration |
| **Data sharing** | Delta Sharing (open protocol) | N/A |
| **Cost model** | DBU (Databricks Units) + VM cost | DWU/vCore-hour |
| **Multi-cloud** | AWS, Azure, GCP | Azure only |
| **Notebook experience** | Superior (collaborative, real-time) | Good |
| **CI/CD** | Repos, Bundles, CLI | Synapse workspace Git |

### When to Use Databricks vs Synapse

✅ **Choose Databricks when:**
- Advanced ML/AI workloads with MLflow
- Need latest Spark features/performance
- Multi-cloud data strategy
- Heavy PySpark/Scala engineering
- Delta Lake-centric architecture
- Need Delta Sharing for cross-org data sharing
- Team prefers notebook-first development

✅ **Choose Synapse when:**
- Need dedicated MPP SQL data warehouse
- T-SQL skills predominate in the team
- Want serverless SQL over data lake (pay-per-query)
- Need unified workspace with built-in pipelines
- Microsoft-native stack (tighter Azure integration)
- Simpler Spark needs alongside SQL

> 🎯 **AZ-305 Tip:** If the scenario mentions "data science team," "MLflow," "multi-cloud," or "advanced Spark," think Databricks. If it says "enterprise data warehouse," "T-SQL," or "unified analytics platform," think Synapse.

---

## 6. Azure Stream Analytics

### Overview

Real-time analytics service for processing high-throughput event streams using SQL-like query language. No Spark or complex coding required.

### Architecture

```
┌─────────────┐      ┌─────────────────────┐      ┌────────────────┐
│   INPUTS    │──────│  Stream Analytics   │──────│    OUTPUTS     │
│             │      │                     │      │                │
│ Event Hubs  │      │  SQL-like queries   │      │ Power BI       │
│ IoT Hub     │      │  Windowing          │      │ ADLS Gen2      │
│ Blob Storage│      │  Pattern matching   │      │ SQL Database   │
│ Kafka       │      │  ML scoring         │      │ Cosmos DB      │
│             │      │  Reference data     │      │ Event Hubs     │
│             │      │                     │      │ Service Bus    │
│             │      │                     │      │ Azure Functions│
└─────────────┘      └─────────────────────┘      └────────────────┘
```

### Windowing Functions

| Window Type | Description | Use Case |
|-------------|-------------|----------|
| **Tumbling** | Fixed-size, non-overlapping, contiguous | Count events every 5 minutes |
| **Hopping** | Fixed-size, overlapping (hop size < window size) | Moving average (10-min window, 5-min hop) |
| **Sliding** | Output when event enters/exits window | Alert if >3 errors in 10 seconds |
| **Session** | Groups events with variable gap timeout | User session activity |
| **Snapshot** | Groups events with same timestamp | Point-in-time aggregation |

```sql
-- Tumbling Window: Count events every 5 minutes
SELECT COUNT(*) as EventCount, System.Timestamp() as WindowEnd
FROM Input
GROUP BY TumblingWindow(minute, 5)

-- Hopping Window: Average temp, 10-min window, 5-min hop
SELECT AVG(temperature) as AvgTemp
FROM IoTInput
GROUP BY HoppingWindow(minute, 10, 5)

-- Sliding Window: Alert if >100 events in 10 seconds
SELECT COUNT(*) as EventCount
FROM Input
GROUP BY SlidingWindow(second, 10)
HAVING COUNT(*) > 100

-- Session Window: Group user events with 5-min timeout
SELECT UserId, COUNT(*) as Actions
FROM ClickStream
GROUP BY UserId, SessionWindow(minute, 5)
```

### IoT Scenarios

| Scenario | Configuration |
|----------|---------------|
| **Anomaly detection** | Built-in ML anomaly detection functions |
| **Geofencing** | `ST_WITHIN`, `ST_DISTANCE` geospatial functions |
| **Fleet monitoring** | Session windows per vehicle + reference data JOIN |
| **Predictive maintenance** | ML model scoring + tumbling window aggregations |
| **Real-time dashboards** | Power BI output adapter |

### Deployment Options

| Option | Description |
|--------|-------------|
| **Cloud job** | Fully managed in Azure |
| **Edge job (IoT Edge)** | Run on IoT Edge devices for local processing |
| **VS Code local** | Develop and test locally |

> 🎯 **AZ-305 Tip:** Stream Analytics is the answer for "real-time" + "SQL-based" + "low code." If the scenario needs complex Spark Streaming logic, use Databricks Structured Streaming instead.

---

## 7. Azure HDInsight

### Overview

Fully managed, open-source analytics service for running Apache Hadoop, Spark, Hive, Kafka, HBase, and more.

### Cluster Types

| Cluster Type | Engine | Use Case |
|--------------|--------|----------|
| **Hadoop** | MapReduce, YARN, Hive, Pig | Batch processing, legacy Hadoop workloads |
| **Spark** | Apache Spark | Big data processing, ML |
| **Kafka** | Apache Kafka | Event streaming, message broker |
| **HBase** | Apache HBase | NoSQL wide-column store, random read/write |
| **Interactive Query** | LLAP (Hive) | Low-latency interactive Hive queries |
| **Storm** | Apache Storm | Real-time stream processing (deprecated) |

### HDInsight vs Databricks vs Synapse Spark

| Feature | HDInsight | Databricks | Synapse Spark |
|---------|-----------|------------|---------------|
| **Management** | IaaS-like (you size clusters) | Managed (autoscale) | Managed (autoscale) |
| **Spark version** | Typically behind | Latest + optimized runtime | Recent versions |
| **Cost control** | Manual (scale/stop) | Auto-terminate, spot VMs | Auto-pause |
| **Kafka** | Native support | No (use Event Hubs) | No |
| **HBase** | Native support | No | No |
| **Hadoop/Hive** | Native support | Limited (Delta preferred) | Limited |
| **Notebook UX** | Jupyter/Zeppelin | Databricks notebooks (best) | Synapse notebooks |
| **Enterprise security** | ESP (Entra ID + Ranger) | Unity Catalog | Synapse workspace security |
| **Multi-tenancy** | Separate clusters | Shared workspace + clusters | Shared workspace + pools |

### When to Use HDInsight

✅ **Choose HDInsight when:**
- Need managed **Kafka** clusters (not Event Hubs)
- Need **HBase** for wide-column NoSQL at scale
- Legacy Hadoop/Hive workloads to lift-and-shift
- Need specific open-source versions/configurations
- Want full control over cluster configuration

❌ **Don't use HDInsight when:**
- Only need Spark (use Databricks or Synapse)
- Want serverless/auto-pause (HDInsight clusters always run)
- Need advanced ML lifecycle management
- Want collaborative notebook experience

> ⚠️ **AZ-305 Tip:** HDInsight is becoming less common in exam scenarios. It's primarily the answer when specific open-source components (Kafka, HBase) are required and Azure-native alternatives (Event Hubs, Cosmos DB) won't work.

---

## 8. Data Architecture Patterns

### Lambda Architecture

```
                    ┌──────────────────────────────┐
                    │        SERVING LAYER         │
                    │    (Merged batch + speed)    │
                    └──────────┬───────────────────┘
                               │
              ┌────────────────┼────────────────┐
              │                │                │
    ┌─────────▼──────┐  ┌─────▼──────────┐     │
    │  BATCH LAYER   │  │  SPEED LAYER   │     │
    │ (Complete,     │  │ (Real-time,    │     │
    │  accurate)     │  │  approximate)  │     │
    │                │  │                │     │
    │ - Spark batch  │  │ - Stream       │     │
    │ - Synapse DW   │  │   Analytics   │     │
    │ - Scheduled    │  │ - Event Hubs  │     │
    └────────▲───────┘  └────▲───────────┘     │
             │               │                 │
             └───────────────┴─────────────────┘
                             │
                    ┌────────▼────────┐
                    │   DATA SOURCE   │
                    │  (All events)   │
                    └─────────────────┘

Pros: Handles both real-time and historical accurately
Cons: Complex (two codepaths to maintain), data reconciliation needed
```

### Kappa Architecture

```
    ┌─────────────────────────────────────┐
    │           SERVING LAYER             │
    └──────────────────┬──────────────────┘
                       │
    ┌──────────────────▼──────────────────┐
    │         STREAM PROCESSING           │
    │   (Single pipeline for all data)    │
    │                                     │
    │   Event Hubs → Spark Streaming      │
    │              → Stream Analytics     │
    │              → Kafka Streams        │
    └──────────────────▲──────────────────┘
                       │
    ┌──────────────────┴──────────────────┐
    │       IMMUTABLE EVENT LOG           │
    │   (Event Hubs / Kafka / ADLS)       │
    └─────────────────────────────────────┘

Pros: Simpler (one codepath), easier to maintain
Cons: Reprocessing history requires replaying entire log
Best for: Event-driven architectures where all data arrives as events
```

### Medallion Architecture (Bronze / Silver / Gold)

```
┌─────────────────────────────────────────────────────────────────┐
│                     DATA LAKEHOUSE                               │
├─────────────────┬──────────────────┬────────────────────────────┤
│                 │                  │                            │
│   🥉 BRONZE     │   🥈 SILVER      │   🥇 GOLD                  │
│   (Raw)         │   (Cleansed)     │   (Business-Ready)        │
│                 │                  │                            │
│ • Raw ingestion │ • Deduplicated   │ • Aggregated              │
│ • All formats   │ • Schema applied │ • Business metrics        │
│ • Append-only   │ • Quality checks │ • Star schema / cubes     │
│ • Minimal       │ • Standardized   │ • Feature store           │
│   transformation│   types/names    │ • Conformed dimensions    │
│ • Data as-is    │ • Filtered nulls │ • SLA-backed              │
│                 │ • Joined refs    │                            │
│                 │                  │                            │
│ Delta/Parquet   │  Delta/Parquet   │  Delta/Parquet            │
│                 │                  │                            │
└─────────────────┴──────────────────┴────────────────────────────┘
         │                  │                    │
    Raw sources        Data Engineers       Analysts / BI
    IoT, APIs,         Data Scientists      Power BI
    Databases                               ML serving
```

**Implementation in Azure:**

| Layer | Technology | Format |
|-------|-----------|--------|
| Bronze | ADF/Synapse Pipelines → ADLS Gen2 | Raw (JSON, CSV, Parquet) |
| Silver | Databricks/Synapse Spark → ADLS Gen2 | Delta Lake |
| Gold | Databricks/Synapse → ADLS Gen2 or Synapse DW | Delta Lake / SQL tables |
| Serving | Power BI, Synapse Serverless, SQL | DirectQuery / Import |

### Modern Data Lakehouse

```
Combines the best of Data Lake + Data Warehouse:

Data Lake benefits:          Data Warehouse benefits:
├── Open formats             ├── ACID transactions
├── Schema-on-read           ├── Schema enforcement
├── Low cost storage         ├── SQL access
├── ML/AI workloads          ├── BI performance
└── Any data type            └── Governance

Lakehouse = Data Lake + Delta Lake + SQL Engine
Azure Implementation: ADLS Gen2 + Delta Lake + Synapse Serverless/Databricks SQL
```

> 🎯 **AZ-305 Tip:** Medallion architecture is the default pattern Microsoft recommends. If the scenario describes "raw data → cleansed → business ready" layers, this is the answer.

---

## 9. Data Governance

### Microsoft Purview

Unified data governance service for discovering, classifying, and governing data across on-premises, multi-cloud, and SaaS.

### Key Capabilities

| Capability | Description |
|------------|-------------|
| **Data Catalog** | Searchable inventory of all data assets |
| **Data Map** | Automated scanning and classification |
| **Data Lineage** | End-to-end tracking of data flow |
| **Data Estate Insights** | Reports on data estate health |
| **Sensitivity Labels** | Classification and protection (MIP labels) |
| **Access Policies** | Centralized access management |
| **Data Sharing** | In-place data sharing (no copy needed) |
| **Data Quality** | Rules, profiling, scoring |

### Data Lineage

```
┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│  Source  │───▶│   ADF    │───▶│  ADLS    │───▶│ Synapse  │
│  (SQL)   │    │ Pipeline │    │  Gen2    │    │   DW     │
└──────────┘    └──────────┘    └──────────┘    └──────────┘
                                                      │
                                                      ▼
                                               ┌──────────┐
                                               │ Power BI │
                                               │  Report  │
                                               └──────────┘

Lineage is automatically captured from:
- Azure Data Factory / Synapse Pipelines
- Azure Databricks (via Unity Catalog or OpenLineage)
- Power BI
- Azure SQL / Synapse SQL
- ADLS Gen2 operations
```

### Sensitivity Labels & Classification

```
Classification types:
├── System classifications (150+ built-in patterns)
│   ├── PII (SSN, passport, credit card)
│   ├── Financial (bank account, SWIFT)
│   └── Healthcare (medical record numbers)
├── Custom classifications (regex or dictionary-based)
└── Microsoft Information Protection (MIP) labels
    ├── Public
    ├── Internal
    ├── Confidential
    └── Highly Confidential
```

### Purview + Fabric Integration

- Fabric workspaces automatically discoverable in Purview
- OneLake data assets scanned and classified
- Sensitivity labels applied in Fabric flow to downstream reports
- Unified lineage across Fabric and external sources

> 🎯 **AZ-305 Tip:** Purview is the answer whenever the scenario mentions "data catalog," "data lineage," "data classification," "governance across hybrid/multi-cloud," or "compliance requirements for data discovery."

---

## 10. Security & Access

### ADLS Gen2 Security Layers

```
Layer 1: Network Security
├── Private endpoints
├── VNet service endpoints
├── Firewall rules (IP allowlist)
└── Disable public access

Layer 2: Authentication
├── Microsoft Entra ID (Azure AD)
├── Shared Key (storage account key)
├── SAS tokens (limited, time-bound)
└── Managed Identities (recommended)

Layer 3: Authorization
├── Azure RBAC (coarse-grained)
│   ├── Storage Blob Data Owner
│   ├── Storage Blob Data Contributor
│   └── Storage Blob Data Reader
├── POSIX ACLs (fine-grained, with HNS)
│   ├── Access ACLs
│   └── Default ACLs
└── SAS permissions (token-based)

Layer 4: Encryption
├── Encryption at rest (SSE, always on)
│   ├── Microsoft-managed keys (MMK)
│   ├── Customer-managed keys (CMK via Key Vault)
│   └── Encryption scopes (per-container keys)
├── Encryption in transit (TLS 1.2+)
└── Infrastructure encryption (double encryption)
```

### RBAC vs ACLs Decision

| Scenario | Use RBAC | Use ACLs |
|----------|----------|----------|
| Grant access to entire storage account | ✅ | ❌ |
| Grant access to specific container | ✅ | ✅ |
| Grant access to specific folder/file | ❌ (too broad) | ✅ |
| Service-to-service access | ✅ (managed identity) | ✅ |
| Multiple teams, same storage account | RBAC for base | ACLs for fine-grained |

> ⚠️ **Key Rule:** If a user has RBAC **Storage Blob Data Owner/Contributor/Reader** role, those permissions are evaluated FIRST. ACLs are only checked when RBAC does NOT grant access. Use `Storage Blob Delegator` role + ACLs for pure ACL-based access.

### Managed Identities for Analytics

| Service | Identity Type | Access Pattern |
|---------|--------------|----------------|
| Azure Data Factory | System-assigned MI | ADF → ADLS Gen2 |
| Synapse Workspace | System-assigned MI + User MI | Synapse → ADLS Gen2 |
| Databricks | Service Principal or MI via Unity Catalog | Databricks → ADLS Gen2 |
| Stream Analytics | System-assigned MI | ASA → Event Hubs, ADLS |
| Azure Functions | System-assigned MI | Functions → ADLS, SQL |

### VNet Integration Patterns

```
Pattern 1: Private Endpoints (Recommended)
┌──────────────────────────────────────┐
│           Virtual Network            │
│  ┌───────────┐    ┌──────────────┐   │
│  │ Synapse   │    │ Private      │   │
│  │ Workspace │───▶│ Endpoint     │───┼──▶ ADLS Gen2
│  └───────────┘    │ (10.0.1.5)   │   │
│                   └──────────────┘   │
└──────────────────────────────────────┘

Pattern 2: Managed VNet (Synapse / Databricks)
┌──────────────────────────────────────┐
│     Managed Virtual Network          │
│  ┌───────────┐                       │
│  │ Synapse   │──▶ Managed Private ───┼──▶ ADLS Gen2
│  │ Spark     │    Endpoint           │
│  └───────────┘                       │
└──────────────────────────────────────┘

Pattern 3: VNet Injection (Databricks)
┌──────────────────────────────────────┐
│      Customer VNet                   │
│  ┌───────────────────────────────┐   │
│  │  Databricks Workspace         │   │
│  │  (injected into customer VNet)│   │
│  └───────────────────────────────┘   │
│         │                            │
│  ┌──────▼──────┐                     │
│  │ Service     │                     │
│  │ Endpoint /  │──────────────────────┼──▶ ADLS Gen2
│  │ Priv Endpt  │                     │
│  └─────────────┘                     │
└──────────────────────────────────────┘
```

### Encryption Best Practices for Analytics

| Requirement | Solution |
|-------------|----------|
| Default encryption at rest | Always on (SSE with MMK) |
| Customer controls keys | CMK with Azure Key Vault |
| Double encryption | Enable infrastructure encryption |
| Column-level encryption | Dynamic Data Masking or Always Encrypted (SQL) |
| Encrypt specific containers differently | Encryption scopes |
| Encrypt data in Spark processing | Transparent (Delta/Parquet encrypted at storage) |

> 🎯 **AZ-305 Tip:** Default answer for "secure data lake access" = Private endpoints + Managed Identity + RBAC + (ACLs if fine-grained needed). Never use storage account keys in production analytics pipelines.

---

## 11. AZ-305 Decision Matrix

### Comprehensive Service Comparison

| Requirement | Best Service | Runner-Up |
|-------------|-------------|-----------|
| Enterprise data warehouse (structured, known queries) | Synapse Dedicated SQL Pool | Fabric Data Warehouse |
| Ad-hoc queries on data lake files | Synapse Serverless SQL | Databricks SQL |
| Advanced ML/AI on big data | Azure Databricks | Synapse Spark |
| Real-time event processing (simple SQL) | Stream Analytics | Event Hubs + Functions |
| Real-time event processing (complex) | Databricks Structured Streaming | Fabric Real-Time Analytics |
| Data integration/ETL orchestration | Azure Data Factory | Synapse Pipelines |
| Unified SaaS analytics platform | Microsoft Fabric | Azure Synapse |
| Managed Kafka | HDInsight (Kafka) | Event Hubs (Kafka protocol) |
| Data lake storage | ADLS Gen2 | OneLake (Fabric) |
| Data governance/catalog | Microsoft Purview | Unity Catalog (Databricks) |
| Self-service BI | Power BI (Fabric) | Synapse Serverless + Power BI |
| Log/telemetry analytics (KQL) | Azure Data Explorer | Fabric Real-Time Analytics |
| Cross-cloud data sharing | Databricks (Delta Sharing) | Fabric shortcuts |
| NoSQL wide-column at scale | HDInsight HBase | Cosmos DB (Table API) |

### Decision Tree: Choosing an Analytics Service

```
START: What is the primary workload?
│
├── Structured data warehouse (SQL, BI)?
│   ├── Need MPP dedicated compute? → Synapse Dedicated SQL Pool
│   ├── Serverless, pay-per-query? → Synapse Serverless SQL Pool
│   └── Unified SaaS platform? → Fabric Data Warehouse
│
├── Big data processing (Spark)?
│   ├── Advanced ML, Delta Lake, multi-cloud? → Azure Databricks
│   ├── Part of unified Synapse workspace? → Synapse Spark Pool
│   └── Need Hadoop/HBase/Kafka? → HDInsight
│
├── Real-time/streaming?
│   ├── Simple transformations, SQL-based? → Stream Analytics
│   ├── Complex Spark Streaming? → Databricks Structured Streaming
│   └── Log/telemetry analytics (KQL)? → Azure Data Explorer
│
├── Data integration/orchestration?
│   ├── Standalone, multi-platform? → Azure Data Factory
│   ├── Within Synapse? → Synapse Pipelines
│   └── Within Fabric? → Fabric Data Factory
│
└── Governance & catalog?
    ├── Enterprise-wide, multi-cloud? → Microsoft Purview
    └── Databricks-specific? → Unity Catalog
```

### Workload-Based Quick Reference

| Workload Pattern | Recommended Architecture |
|-----------------|-------------------------|
| Traditional EDW modernization | Synapse Dedicated Pool + ADF + Power BI |
| Modern data lakehouse | ADLS Gen2 + Databricks + Delta Lake + Power BI |
| Real-time IoT analytics | IoT Hub → Stream Analytics → ADLS/Power BI |
| Unified self-service analytics | Microsoft Fabric (OneLake + Lakehouse + Power BI) |
| Hybrid cloud data integration | ADF + Self-hosted IR + ADLS Gen2 |
| Multi-cloud analytics | Databricks (Azure + AWS) + Delta Sharing |
| Log analytics at scale | Event Hubs → Azure Data Explorer / Fabric KQL |
| Streaming + batch unified | Databricks (Structured Streaming + Batch on Delta) |

---

## 12. AZ-305 Decision Scenarios

### Scenario 1: Enterprise Data Warehouse Modernization

**Situation:** A retail company has a 50TB on-premises SQL Server data warehouse. They need to migrate to Azure with minimal application changes. The BI team uses complex T-SQL queries and stored procedures. They need predictable performance for 500 concurrent users during business hours.

**Solution:** Azure Synapse Analytics - Dedicated SQL Pool (DW1500c or higher)

**Reasoning:**
- MPP architecture handles 50TB with predictable performance
- T-SQL compatibility minimizes migration effort
- Stored procedures supported
- Workload management for 500 concurrent users
- Can pause during off-hours for cost savings
- ADF with Self-hosted IR for ongoing data loads

**Why not alternatives:**
- Fabric DW: Less T-SQL compatibility, newer service (migration risk)
- Databricks SQL: Would require rewriting stored procedures
- Serverless SQL: Not suitable for 500 concurrent users with complex queries

---

### Scenario 2: Real-Time Fraud Detection

**Situation:** A fintech company processes 1 million transactions per second. They need to detect fraudulent patterns within 2 seconds, enrich transactions with customer profiles, and trigger alerts. The data science team maintains ML models in Python.

**Solution:** Event Hubs → Azure Databricks Structured Streaming → Delta Lake → Alerts (via Event Hubs output)

**Reasoning:**
- 1M events/sec requires Event Hubs (dedicated tier) for ingestion
- Complex ML scoring needs Spark (not simple SQL windowing)
- Python ML models integrate natively with Databricks
- Delta Lake for both streaming and batch state management
- Sub-2-second latency achievable with micro-batch tuning
- Unity Catalog for model governance

**Why not Stream Analytics:**
- Complex ML model scoring not easily expressed in SQL
- Python UDF performance insufficient at 1M events/sec
- Pattern matching across multiple event types is complex

---

### Scenario 3: Multi-Team Data Lake

**Situation:** A healthcare organization has 5 departments that need to share a data lake. Each department needs access only to their own data plus shared reference data. They process both structured claims data and unstructured medical images. Strict HIPAA compliance required.

**Solution:** ADLS Gen2 + Medallion Architecture + POSIX ACLs + Microsoft Purview

**Architecture:**
```
ADLS Gen2 (HNS enabled)
├── /bronze/dept-claims/      (ACL: claims-team rwx)
├── /bronze/dept-imaging/     (ACL: imaging-team rwx)
├── /silver/shared-reference/ (ACL: all-analysts r-x)
├── /gold/analytics/          (ACL: per-department)
└── Private endpoints + CMK encryption
```

**Reasoning:**
- HNS enables POSIX ACLs for per-department access
- Purview for HIPAA compliance (classification, lineage, audit)
- Private endpoints for network isolation
- CMK encryption for data sovereignty
- Managed identities for all service-to-service access

---

### Scenario 4: Self-Service Analytics Platform

**Situation:** A media company wants to empower business analysts to explore data without depending on IT. They need data from SQL databases, cloud storage, and SaaS applications (Salesforce, Google Analytics). The CFO wants one bill and predictable costs.

**Solution:** Microsoft Fabric

**Reasoning:**
- Single SaaS platform (one bill, capacity-based pricing)
- OneLake shortcuts connect to external sources without copying
- Lakehouse for self-service exploration
- Built-in Power BI for visualization
- Low-code Data Factory for analyst-built pipelines
- Governance built-in (no separate Purview configuration needed)
- Capacity model gives predictable costs

**Why not Synapse + ADF + Power BI separately:**
- Multiple services = multiple bills, more complexity
- Fabric provides tighter integration and simpler governance
- Self-service experience is superior in Fabric

---

### Scenario 5: IoT Telemetry Analytics

**Situation:** A manufacturing company has 10,000 sensors sending temperature, pressure, and vibration data every second. They need: (1) real-time alerts when thresholds are exceeded, (2) hourly aggregated dashboards, (3) historical trend analysis over 2 years of data.

**Solution:** IoT Hub → Stream Analytics → Multi-output architecture

```
IoT Hub → Stream Analytics ──┬──▶ Event Hubs (alerts → Azure Functions → notifications)
                             ├──▶ Power BI (real-time dashboard)
                             └──▶ ADLS Gen2 (Bronze layer, raw archive)
                                      │
                            Synapse Serverless SQL (historical queries)
                            Databricks (trend analysis, predictive maintenance ML)
```

**Reasoning:**
- IoT Hub for device management and ingestion (10K devices)
- Stream Analytics for SQL-based windowing (simple threshold alerts)
- Multiple outputs: alerts, dashboards, archive
- ADLS Gen2 for cost-effective long-term storage (2 years, cold tier)
- Synapse Serverless for ad-hoc historical queries (pay-per-query)
- Databricks for advanced trend analysis/ML (only when needed)

---

### Scenario 6: Data Mesh Implementation

**Situation:** A large enterprise (10,000+ employees) wants to implement a data mesh pattern where each domain team owns and publishes their data products. They need federated governance, self-serve data platform, and cross-domain data discovery.

**Solution:** Azure Databricks + Unity Catalog + Microsoft Purview + ADLS Gen2

**Architecture:**
```
┌────────────────────────────────────────────────────────┐
│                 FEDERATED GOVERNANCE                    │
│         (Purview + Unity Catalog)                       │
└────────────────────────────────────────────────────────┘
         │              │              │
┌────────▼─────┐ ┌─────▼──────┐ ┌────▼────────┐
│ Domain: Sales│ │Domain: Ops  │ │Domain: Fin  │
│              │ │             │ │             │
│ Databricks   │ │ Databricks  │ │ Databricks  │
│ Workspace    │ │ Workspace   │ │ Workspace   │
│ + ADLS Gen2  │ │ + ADLS Gen2 │ │ + ADLS Gen2 │
│              │ │             │ │             │
│ Data Products│ │Data Products│ │Data Products│
│ (published   │ │(published   │ │(published   │
│  via Delta   │ │ via Delta   │ │ via Delta   │
│  Sharing)    │ │ Sharing)    │ │ Sharing)    │
└──────────────┘ └─────────────┘ └─────────────┘
```

**Reasoning:**
- Unity Catalog provides federated governance across workspaces
- Delta Sharing enables cross-domain data product publishing (zero-copy)
- Purview provides enterprise-wide catalog and discovery
- Each domain team has autonomy (own workspace, own storage)
- Consistent governance without central bottleneck

---

### Scenario 7: Cost-Sensitive Startup Analytics

**Situation:** A startup with limited budget needs to analyze 500GB of user behavior data. Queries are infrequent (a few times per week). They want to build dashboards and run occasional ML experiments. Team of 3 data analysts who know SQL.

**Solution:** ADLS Gen2 + Synapse Serverless SQL Pool + Power BI

**Reasoning:**
- ADLS Gen2: ~$10/month for 500GB (cool tier)
- Synapse Serverless: Pay only per TB scanned (~$5/TB)
- Infrequent queries = minimal cost (maybe $20-50/month)
- Power BI Pro: $10/user/month for dashboards
- No dedicated infrastructure running 24/7
- SQL-familiar for the team
- Can add Spark pool later (auto-pause) for ML experiments

**Total estimated cost: $50-100/month** vs $1000+/month for always-on solutions

---

## 13. Quick Reference Trigger Table

### "If the Scenario Says X, Think Y"

| If the scenario mentions... | Think... |
|----------------------------|----------|
| "Petabyte-scale data warehouse" | Synapse Dedicated SQL Pool |
| "T-SQL stored procedures migration" | Synapse Dedicated SQL Pool |
| "Query data lake without loading" | Synapse Serverless SQL Pool |
| "Pay only for queries run" | Synapse Serverless SQL Pool |
| "Advanced machine learning on big data" | Azure Databricks |
| "MLflow / model registry" | Azure Databricks |
| "Multi-cloud analytics" | Azure Databricks |
| "Delta Lake / Delta Sharing" | Azure Databricks |
| "Unified SaaS analytics" | Microsoft Fabric |
| "One platform, one bill" | Microsoft Fabric |
| "Self-service for business users" | Microsoft Fabric |
| "Real-time alerts, simple rules" | Stream Analytics |
| "IoT + windowing functions" | Stream Analytics |
| "SQL-based streaming" | Stream Analytics |
| "Complex event processing (Spark)" | Databricks Structured Streaming |
| "Managed Kafka" | HDInsight Kafka or Event Hubs (Kafka protocol) |
| "HBase / wide-column NoSQL" | HDInsight HBase |
| "Hadoop migration lift-and-shift" | HDInsight |
| "Data orchestration / ETL pipelines" | Azure Data Factory |
| "On-premises data integration" | ADF + Self-hosted Integration Runtime |
| "SSIS package migration" | ADF Azure-SSIS IR |
| "Data catalog / discovery" | Microsoft Purview |
| "Data lineage" | Microsoft Purview |
| "HIPAA / GDPR classification" | Purview + Sensitivity Labels |
| "Hierarchical file permissions" | ADLS Gen2 + POSIX ACLs |
| "Fine-grained folder access" | ADLS Gen2 + ACLs |
| "Medallion / Bronze-Silver-Gold" | ADLS Gen2 + Databricks/Synapse Spark |
| "Lakehouse architecture" | Databricks or Fabric Lakehouse |
| "Real-time dashboards" | Stream Analytics → Power BI |
| "Log analytics / KQL" | Azure Data Explorer / Fabric Real-Time Analytics |
| "Reduce data silos" | Microsoft Fabric (OneLake) |
| "Data mesh / domain ownership" | Databricks Unity Catalog + Delta Sharing |
| "Predictable performance, dedicated resources" | Synapse Dedicated Pool or Databricks SQL Warehouse |
| "Cost-sensitive, infrequent queries" | Synapse Serverless SQL |
| "Auto-pause compute" | Synapse Spark Pool or Fabric capacity |
| "Cross-org data sharing (no copy)" | Delta Sharing (Databricks) or Fabric shortcuts |

### Anti-Patterns to Avoid

| ❌ Don't... | ✅ Instead... |
|------------|--------------|
| Use Dedicated SQL Pool for infrequent queries | Use Serverless SQL Pool |
| Use HDInsight when only Spark is needed | Use Databricks or Synapse Spark |
| Use Stream Analytics for complex ML scoring | Use Databricks Structured Streaming |
| Store analytics data without HNS | Enable Hierarchical Namespace (ADLS Gen2) |
| Use storage account keys in pipelines | Use Managed Identities |
| Copy data between services unnecessarily | Use shortcuts / external tables / Delta Sharing |
| Over-provision dedicated pools "just in case" | Start small, monitor, scale up |
| Use ADF for transformations that belong in Spark | Use ADF for orchestration, Spark for transforms |

---

## 14. Cost Optimization

### Reserved Capacity Options

| Service | Reservation Type | Savings | Term |
|---------|-----------------|---------|------|
| Synapse Dedicated SQL Pool | cDWU Reserved | Up to 65% | 1 or 3 year |
| Azure Databricks | DBU Commit | Up to 37% | 1 or 3 year |
| ADLS Gen2 | Storage Reserved Capacity | Up to 38% | 1 or 3 year |
| Azure Data Explorer | Markup Reserved | Up to 50% | 1 or 3 year |
| HDInsight | VM Reserved Instances | Up to 72% | 1 or 3 year |
| Microsoft Fabric | Fabric Capacity Reservation | Up to 40% | 1 year |

### Auto-Pause & Auto-Scale Strategies

| Service | Feature | Configuration |
|---------|---------|---------------|
| Synapse Dedicated SQL | Pause/Resume | Manual, scheduled, or API-based |
| Synapse Spark Pool | Auto-pause | 5-90 minutes idle timeout |
| Synapse Spark Pool | Auto-scale | 3-200 nodes (scale within minutes) |
| Databricks | Auto-terminate | 10-120 minutes idle |
| Databricks | Spot instances | Up to 90% savings on worker nodes |
| Databricks | Photon engine | 2-8x faster (fewer DBU-hours) |
| Stream Analytics | Auto-scale | SU (Streaming Units) auto-adjustment |
| Fabric | Auto-pause (F SKUs) | Pause when idle |

### Right-Sizing Guidelines

**Synapse Dedicated SQL Pool:**
```
Workload Size    | Recommended DWU | Concurrent Queries
─────────────────┼─────────────────┼───────────────────
Small (<1TB)     | DW100c-DW500c   | 4-20
Medium (1-10TB)  | DW500c-DW3000c  | 20-60
Large (10-100TB) | DW3000c-DW6000c | 60-128
XLarge (>100TB)  | DW6000c+        | 128
```

**Databricks Cluster Sizing:**
```
Workload Type        | Instance Series | Cluster Mode
─────────────────────┼─────────────────┼──────────────────
Interactive/Dev      | Standard_DS3_v2 | Single Node or Small
ETL (memory)        | Standard_E8_v3  | Multi-node, autoscale
ML Training (GPU)   | Standard_NC6    | GPU-optimized
SQL Warehouse (BI)  | Serverless      | Auto-managed
```

### Cost Optimization Decision Matrix

| Scenario | Optimization Strategy |
|----------|----------------------|
| Known steady-state workload | Reserved capacity (1-3 year) |
| Variable workload, cost-sensitive | Serverless/pay-per-query |
| Development/test environments | Auto-pause + smallest tier |
| Production with off-hours | Schedule pause/resume |
| Large Spark workloads | Spot instances (workers) + auto-terminate |
| Storing historical data rarely queried | ADLS Gen2 Cool/Archive tier |
| BI queries on data lake | Synapse Serverless (avoid loading into DW) |
| Multiple small teams, varied workloads | Fabric capacity (shared CUs) |

### Cost Monitoring

```
Azure Cost Management:
├── Set budgets per resource group / subscription
├── Tag resources: team, project, environment
├── Alerts at 50%, 75%, 90% of budget
└── Anomaly detection for unexpected spikes

Synapse-specific:
├── Monitor DWU utilization (target 60-80%)
├── Review query performance DMVs
├── Identify and eliminate long-running queries
└── Use result-set caching and materialized views

Databricks-specific:
├── Unity Catalog usage tracking
├── Cluster policies (enforce max sizes)
├── Spot instance fallback policies
└── DBU consumption alerts
```

---

## 15. Integration Patterns

### How Services Connect Together

```
                    ┌─────────────────────────────────────────┐
                    │           DATA SOURCES                   │
                    │  On-prem SQL │ SaaS APIs │ IoT Devices  │
                    │  Cloud DBs   │ Files     │ Event Streams│
                    └───────┬──────────┬──────────┬───────────┘
                            │          │          │
                    ┌───────▼──────────▼──────────▼───────────┐
                    │          INGESTION LAYER                 │
                    │  ADF (batch) │ Event Hubs (streaming)   │
                    │  IoT Hub     │ Kafka (HDInsight)        │
                    └───────┬──────────┬──────────────────────┘
                            │          │
                    ┌───────▼──────────▼──────────────────────┐
                    │         STORAGE LAYER                    │
                    │  ADLS Gen2 (Bronze/Silver/Gold)          │
                    │  OneLake (Fabric)                        │
                    └───────┬──────────┬──────────────────────┘
                            │          │
                    ┌───────▼──────────▼──────────────────────┐
                    │        PROCESSING LAYER                  │
                    │  Databricks │ Synapse │ Stream Analytics │
                    │  HDInsight  │ Fabric  │ Azure Functions  │
                    └───────┬──────────┬──────────────────────┘
                            │          │
                    ┌───────▼──────────▼──────────────────────┐
                    │         SERVING LAYER                    │
                    │  Synapse DW  │ Cosmos DB  │ SQL Database │
                    │  Power BI    │ API Apps   │ ML Endpoints │
                    └───────┬──────────────────────────────────┘
                            │
                    ┌───────▼──────────────────────────────────┐
                    │        GOVERNANCE LAYER                   │
                    │  Microsoft Purview (catalog, lineage,     │
                    │  classification, access policies)         │
                    └──────────────────────────────────────────┘
```

### Event-Driven Pipeline Patterns

#### Pattern 1: Event-Driven ETL (File Arrival)

```
Blob Created Event ──▶ Event Grid ──▶ ADF Pipeline Trigger
                                           │
                                           ▼
                                    ADF Pipeline:
                                    1. Validate file
                                    2. Load to Bronze
                                    3. Transform (Spark)
                                    4. Load to Silver
                                    5. Notify downstream
```

**Implementation:**
- Storage account → Event Grid → ADF Event Trigger
- Or: Storage account → Event Grid → Azure Function → ADF REST API

#### Pattern 2: Real-Time + Batch Unified (Kappa-style)

```
Source Events ──▶ Event Hubs ──┬──▶ Databricks Streaming ──▶ Delta Lake (Silver)
                               │                                     │
                               │                              ┌──────▼──────┐
                               │                              │  Merge into │
                               │                              │  Gold layer │
                               │                              └──────┬──────┘
                               │                                     │
                               └──▶ Stream Analytics ──▶ Real-time alerts
                                                                     │
                                                              ┌──────▼──────┐
                                                              │  Power BI   │
                                                              │  Dashboard  │
                                                              └─────────────┘
```

#### Pattern 3: Change Data Capture (CDC)

```
Source Database ──▶ CDC (Debezium/ADF) ──▶ Event Hubs ──▶ Databricks ──▶ Delta Lake
                                                                              │
                   ┌──────────────────────────────────────────────────────────┘
                   ▼
            Delta Lake Change Data Feed ──▶ Downstream consumers
                                       ──▶ Synapse Serverless (queries)
                                       ──▶ Power BI (DirectQuery)
```

#### Pattern 4: Data API Pattern

```
Client App ──▶ API Management ──▶ Azure Function ──▶ Synapse Serverless SQL
                                                  ──▶ Cosmos DB (low latency)
                                                  ──▶ Redis Cache (hot data)
```

### Service Connectivity Matrix

| From \ To | ADLS Gen2 | Synapse | Databricks | Event Hubs | Power BI |
|-----------|-----------|---------|------------|------------|----------|
| **ADF** | ✅ Native | ✅ Native | ✅ Activity | ✅ Sink/Source | ❌ |
| **Synapse** | ✅ External tables | ✅ Internal | ✅ Linked service | ✅ Data Explorer | ✅ Native |
| **Databricks** | ✅ abfss:// | ✅ JDBC/connector | ✅ Internal | ✅ Spark connector | ✅ Direct |
| **Stream Analytics** | ✅ Output | ✅ Output | ❌ | ✅ Input/Output | ✅ Output |
| **Fabric** | ✅ Shortcuts | ✅ (via OneLake) | ✅ Shortcuts | ✅ Eventstream | ✅ Native |
| **Purview** | ✅ Scan/Lineage | ✅ Scan/Lineage | ✅ Lineage | ✅ Scan | ✅ Lineage |

### Common Integration Topologies

#### Topology 1: Microsoft-Native Stack
```
ADF → ADLS Gen2 → Synapse (Dedicated + Serverless) → Power BI
                          ↕
                     Purview (governance)
```

#### Topology 2: Databricks-Centric
```
ADF → ADLS Gen2 → Databricks (Delta Lake + Unity Catalog) → Power BI / Databricks SQL
                                    ↕
                          Purview + Unity Catalog
```

#### Topology 3: Fabric-Unified
```
Fabric Data Factory → OneLake → Lakehouse/Warehouse → Power BI
                                      ↕
                          Built-in Governance + Purview
```

#### Topology 4: Hybrid (Most Common in Enterprise)
```
ADF (orchestration) → ADLS Gen2 (storage) → Databricks (heavy transforms) 
                                           → Synapse Serverless (ad-hoc SQL)
                                           → Synapse Dedicated (DW)
                                           → Power BI (BI)
                                           → Purview (governance)
```

---

## Appendix: Key Metrics & Limits

| Service | Key Limit |
|---------|-----------|
| ADLS Gen2 | 190.7 TiB max file size, 5 PB per account (soft) |
| Synapse Dedicated SQL | DW30000c max, 128 concurrent queries |
| Synapse Serverless SQL | 1000 concurrent queries, 1 TB max result size |
| Synapse Spark | 200 nodes max per pool |
| Databricks | 10,000 DBU/hour (Standard tier) |
| Stream Analytics | 200 SU max per job (can request more) |
| Event Hubs | 100 TU (Standard), 20 CU (Dedicated) |
| ADF | 500 concurrent pipeline runs (max) |
| Fabric | F2048 max capacity |

---

## Appendix: Exam Day Quick Review

### Top 10 AZ-305 Data & Analytics Rules

1. **ADLS Gen2 + HNS** = Always for data lake analytics (never flat blob)
2. **Synapse Serverless** = Ad-hoc, infrequent, cost-sensitive data lake queries
3. **Synapse Dedicated** = Predictable performance, enterprise DW, complex T-SQL
4. **Databricks** = Advanced Spark/ML, Delta Lake, multi-cloud
5. **Fabric** = Unified SaaS, one bill, self-service, reduce complexity
6. **Stream Analytics** = Real-time + SQL + simple (no Spark needed)
7. **ADF** = Orchestration hub for multi-service data pipelines
8. **Purview** = Governance, catalog, lineage, classification
9. **Managed Identity** = Always for service-to-service auth (never keys)
10. **Medallion Architecture** = Default pattern for data lakehouse layers

### Key Pricing Mental Model

```
Cheapest → Most Expensive (for analytics queries):
1. Synapse Serverless SQL    (~$5/TB scanned, zero when idle)
2. Fabric Lakehouse          (shared capacity, auto-pause)
3. Databricks SQL Serverless (DBU/hour, auto-stop)
4. Synapse Dedicated SQL     (DWU/hour, can pause)
5. Databricks Interactive    (DBU/hour + VM cost)
6. HDInsight                 (VM cost 24/7)
```

---

*End of Cheat Sheet — Good luck on AZ-305! 🎯*
