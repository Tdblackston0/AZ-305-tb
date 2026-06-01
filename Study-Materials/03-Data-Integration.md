# Design Data Integration - CRITICAL PRIORITY ⭐⭐⭐

**Exam Weight:** Part of Data Storage Solutions (20-25%)  
**Your Performance:** ⚠️⚠️ CRITICAL WEAKNESS  
**Potential Points:** +5-10

---

## Overview

**Data integration** is about designing how data moves, transforms, and flows between systems. This includes:
- ETL/ELT patterns
- Data pipeline architecture
- Choosing between batch and real-time processing
- Orchestrating complex data workflows

---

## Core Concepts

### ETL vs ELT: Critical Distinction

| Aspect | ETL | ELT |
|--------|-----|-----|
| **Transform Location** | On-premises/staging server | Cloud data warehouse |
| **When Used** | Legacy, small datasets, complex transformations | Cloud-native, big data, modern analytics |
| **Performance** | Slower (limited compute in middle tier) | Faster (cloud warehouse compute) |
| **Cost** | Lower cloud storage (transformed) | Higher cloud storage (raw data first) |
| **Data Quality** | Checked before loading | Checked after loading |
| **Tools** | SQL Server Integration Services (SSIS), Talend | Spark, Synapse, BigQuery |

**Exam Scenario:** "Your company wants to migrate from on-premises data warehouse (SSIS jobs) to Azure. The company loads 100 GB of raw data daily. Should you use ETL or ELT?"
- **Answer:** ELT - Use Azure Data Factory to copy raw data to Data Lake, use Synapse SQL to transform, much faster and scalable in cloud

---

## Azure Data Integration Services

### 1. Azure Data Factory (ADF)

**What it is:** Serverless data integration service for orchestrating data pipelines

**Key Concepts:**
- **Pipeline** - Workflow of activities
- **Activity** - Individual task (copy data, execute script, run notebook)
- **Linked Service** - Connection to data source/destination
- **Dataset** - Named reference to data in a source
- **Integration Runtime** - Compute that executes activities

**Pipeline Example:**
```
Trigger (schedule or event)
    ↓
Copy from source (SQL DB)
    ↓
Transform (Databricks notebook)
    ↓
Copy to destination (Data Lake)
    ↓
Run stored procedure (validate)
```

**When to Use ADF:**
- Orchestrating multiple data sources
- Scheduling data movement
- Complex multi-step pipelines
- Hybrid scenarios (on-prem + cloud)

**Hands-On Lab:**
```
1. Create linked services (SQL DB, Blob Storage)
2. Define datasets
3. Build pipeline: copy + transform
4. Schedule execution
5. Monitor pipeline runs
```

**Microsoft Learn:** [Azure Data Factory fundamentals](https://learn.microsoft.com/training/modules/data-factory-fundamentals/)

---

### 2. Azure Synapse Analytics

**What it is:** All-in-one analytics platform combining data warehousing, big data, and analytics

**Key Components:**
- **SQL Pools** - Dedicated data warehouse (like SQL Server but cloud-scale)
- **Spark Pools** - Distributed compute for big data processing
- **Pipelines** - Integrated data orchestration (like ADF)
- **Studio** - Integrated development environment

**When to Use Synapse:**
- Large-scale data warehouse (>100GB)
- Unified analytics platform needed
- Real-time + batch analytics
- Interactive queries on big data

**Architecture:**
```
Raw Data (Data Lake)
    ↓
Synapse Pipeline (load/transform)
    ↓
SQL Pool (structured queries) + Spark Pool (ML)
    ↓
Power BI / Analytics Apps
```

**Exam Scenario:** "Your company has 5 TB of raw data in a Data Lake and needs both SQL queries and machine learning. How do you design this?"
- **Answer:** Use Synapse SQL Pool for structured queries, Spark Pool for ML, shared workspace for notebooks/pipelines

**Hands-On Lab:**
```
1. Create Synapse workspace
2. Load data from Data Lake
3. Create SQL pool
4. Run Spark notebook
5. Query across both compute types
```

---

### 3. Data Lake Gen 2

**What it is:** Massively scalable, secure data storage designed for analytics

**Key Advantages over Blob Storage:**
- Hierarchical namespace (folder structure, not just flat blobs)
- Fine-grained access control (POSIX-like permissions)
- Better for big data analytics
- Cost-effective (pay as you go)

**Typical Data Lake Structure:**
```
/
├── raw/ (ingestion layer - unchanged data)
├── processed/ (transformed data)
├── curated/ (business-ready data)
└── archive/ (cold storage for compliance)
```

**Exam Scenario:** "You're building a data lake for IoT data. How do you organize it for both real-time analytics and long-term compliance?"
- **Answer:**
  - /raw/iot/year=2024/month=05/day=31/ (partition by date)
  - /processed/iot_cleaned/ (remove duplicates, fix formats)
  - /curated/iot_aggregated/ (hourly summaries)
  - Use lifecycle management to move old data to archive (cold storage)

---

### 4. Azure Data Explorer (Kusto)

**What it is:** Fast, scalable data exploration and analytics service

**When to Use:**
- Time-series data (IoT, logs, telemetry)
- Real-time analytics required
- Ad-hoc queries on large datasets
- Ingestion rates > 100K events/sec

**Advantages:**
- Extremely fast ingestion and queries
- Native time-series functions
- Real-time alerting capabilities

**Example:** Ingest 1 million IoT events/sec, query for anomalies instantly

---

## Data Pipeline Patterns

### Pattern 1: Batch ETL (Daily/Weekly)

```
Schedule Trigger (e.g., 2 AM daily)
    ↓
Source System (OLTP DB)
    ↓
ADF Copy Activity
    ↓
Data Lake (landing zone)
    ↓
Databricks Transformation
    ↓
SQL Data Warehouse
    ↓
BI Tool Refresh
    ↓
Reports available for business users
```

**Best For:** Finance, HR, periodic reporting  
**RTO:** Hours  
**RPO:** 1 day  
**Cost:** Lowest

### Pattern 2: Real-Time Streaming

```
Event Source (IoT, app events)
    ↓
Event Hub / Kafka
    ↓
Stream Analytics / Spark Streaming
    ↓
Real-time transformations
    ↓
Cosmos DB (real-time insights)
    ↓
Live Dashboard / Alerts
```

**Best For:** Dashboards, fraud detection, anomalies  
**Latency:** Seconds to milliseconds  
**Cost:** Higher (always-on compute)

### Pattern 3: Lambda Architecture (Batch + Streaming)

```
Data Source
    ├─ Speed Layer (Stream Analytics)
    │   ├─ Event Hub
    │   └─ Real-time view (Cosmos DB)
    │
    └─ Batch Layer (Spark)
        ├─ Data Lake
        └─ Batch view (SQL)
        
Serving Layer (merge both views)
    ↓
Reports & Analytics
```

**Best For:** Accuracy + speed needed (financial analytics)  
**Complexity:** High

### Pattern 4: Kappa Architecture (Streaming Only)

```
Event Stream (Event Hub)
    ↓
Stream Processing (Spark Streaming)
    ↓
Serving Layer (Cosmos DB, SQL)
    ↓
Analytics / Queries
```

**Best For:** Real-time-first applications  
**Advantage:** Simpler than Lambda (only one processing path)  
**Challenge:** Reprocessing historical data is complex

---

## Data Transformation Technologies

### 1. SQL (Synapse, Azure SQL)

**Best For:** Structured data, complex queries, traditional analytics  
**Ease:** Easy for SQL developers  
**Scalability:** SQL Pool scales to petabytes  
**Latency:** Sub-second queries

### 2. Apache Spark (Synapse, Databricks)

**Best For:** Unstructured data, ML, graph processing, complex transformations  
**Languages:** Python, Scala, SQL  
**Scalability:** Distributed processing  
**When to use:** When SQL isn't enough

### 3. Data Factory Mapping Data Flows

**Best For:** Visual, low-code data transformations  
**Advantages:** UI-based, no code needed  
**Limitations:** Not for very complex logic  
**Usefulness:** Quick transformations without custom code

### 4. Azure Databricks

**What it is:** Managed Apache Spark platform with Notebooks  
**Best For:** Data science, ML pipelines, collaborative analytics  
**Integration:** Works with Data Lake, Synapse, other services  

---

## Choosing Between Technologies

### "Should I use Batch or Real-Time?"

```
Do you need results immediately?
├─ YES (seconds) → Real-time streaming
└─ NO (hours/days acceptable)
    ├─ Small data (<1TB) → Batch SQL
    └─ Large data (>1TB) → Batch Spark or Synapse
```

### "Should I use SQL or Spark?"

```
Is data structured (tables, columns)?
├─ YES, mostly → Use SQL (simpler, faster for queries)
└─ NO (images, unstructured text)
    ├─ → Use Spark
    
Do I need advanced ML?
├─ YES → Use Spark (Spark MLlib, integration with ML frameworks)
└─ NO → SQL might suffice
```

---

## Data Integration Scenarios

### Scenario 1: Data Migration from On-Premises

**Context:** Company has on-premises SQL Server warehouse (100GB) and needs to migrate to Azure

**Design:**
```
Step 1: Data Assessment (current data size, growth)
Step 2: Choose target platform
  - < 500GB: Use SQL Database
  - > 500GB: Use Synapse or Data Lake + Synapse
Step 3: ADF pipeline for incremental load
  - Full load initially
  - Delta load nightly
Step 4: Validation checks
Step 5: Cutover (migration day)
```

**Tools:** ADF (orchestration) + Data Factory Copy Activity + SQL DW (destination)

### Scenario 2: Real-Time Analytics Dashboard

**Context:** Company needs real-time view of sales as orders come in

**Design:**
```
Order System (source)
    ↓
Event Hub (decoupling)
    ↓
Stream Analytics (aggregations)
    ├─ Tumbling window (1-minute sales)
    └─ Output to Cosmos DB
    ↓
Power BI (real-time dashboard)
    ↓
Business users see updated metrics every minute
```

**RTO:** Immediate / **RPO:** 1 minute

### Scenario 3: Data Lake for Historical Analytics

**Context:** Company wants to explore historical patterns (5 years of data = 50TB)

**Design:**
```
Raw Data Layer
├── /raw/sales/year=2024/month=01/
├── /raw/sales/year=2023/month=12/
└── /raw/sales/year=2020/month=01/

Processed Layer (deduplicated, clean)
├── /processed/sales_clean/

Curated Layer (business aggregations)
├── /curated/daily_sales/
├── /curated/customer_segments/
└── /curated/regional_trends/

Lifecycle Management:
├── /raw/ → 90 days → Archive to cold storage
├── /processed/ → 1 year → Archive
└── /curated/ → Keep hot (frequently accessed)
```

**Tools:** Data Lake Gen 2 + Synapse Spark + Power BI

---

## Common Data Integration Mistakes

❌ **Mistake 1:** Choosing ETL when ELT is better (cloud)  
✅ **Fix:** Use ELT for cloud (Data Lake → transform in cloud)

❌ **Mistake 2:** Building streaming for data that doesn't need it  
✅ **Fix:** Batch is simpler and cheaper for most use cases

❌ **Mistake 3:** No data governance (raw data mixed with processed)  
✅ **Fix:** Use layered Data Lake structure (raw/processed/curated)

❌ **Mistake 4:** Ignoring data quality checks  
✅ **Fix:** Add validation in pipelines (row counts, duplicates, nulls)

❌ **Mistake 5:** Not planning for scaling  
✅ **Fix:** Start with partitioned structure from day 1

---

## Exam Tips for Data Integration

1. **Pattern recognition:** Identify if scenario needs batch, real-time, or both
2. **Service selection:** Match service to scale and complexity (ADF for simple, Synapse for large, Functions for serverless)
3. **Architecture:** Think about data flow - where does it come from, where does it go, what transformations?
4. **Cost vs performance:** Real-time costs more than batch
5. **ETL vs ELT:** Cloud-native usually means ELT

---

## Quick Reference: Service Selection

| Need | Service |
|------|---------|
| Daily batch load (< 50GB) | Data Factory + SQL Database |
| Large data warehouse (> 100GB) | Synapse |
| Scalable storage + queries | Data Lake + Synapse |
| Real-time events | Event Hub + Stream Analytics |
| Complex ML pipeline | Databricks + Data Lake |
| Time-series analytics | Data Explorer |

---

## Key Microsoft Learn Resources

1. **[Design data integration](https://learn.microsoft.com/training/modules/design-data-integration/)** - 50 min
2. **[Azure Data Factory fundamentals](https://learn.microsoft.com/training/modules/data-factory-fundamentals/)** - 45 min
3. **[Azure Synapse SQL pool](https://learn.microsoft.com/training/modules/create-data-warehouse-azure-synapse-analytics/)** - 50 min
4. **[Stream Processing with Spark](https://learn.microsoft.com/training/modules/stream-processing-spark/)** - 60 min

**Total Study Time:** 3-4 hours  
**Hands-On Labs:** 2-3 hours

---

## Practice Scenarios

### Quick Quiz: Choose the Service

1. Load 5GB of data daily from 10 sources into a data warehouse
   - Answer: **ADF + SQL Database**

2. Process 100,000 IoT events/sec in real-time
   - Answer: **Event Hub + Stream Analytics or Spark Streaming**

3. Migrate 500GB on-premises data warehouse to Azure
   - Answer: **ADF + Synapse (destination)**

4. Build ML model from 100GB historical data
   - Answer: **Databricks + Data Lake**

---

## Next Steps

1. ✅ Read this guide
2. **→ Complete Microsoft Learn modules (prioritize Synapse + ADF)**
3. **→ Build a sample pipeline in ADF**
4. **→ Create a Data Lake structure for a scenario**
5. **→ Practice exam questions on data patterns**

**Remember:** Data integration is about matching tools to requirements. Batch vs streaming is the first decision. 🎯
