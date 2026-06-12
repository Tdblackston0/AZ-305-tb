# 🗄️ AZ-305: Design Data Storage Solutions

> **Exam Domain Weight: 20–25%** of the AZ-305 exam focuses on designing data storage solutions.

## Overview

This domain tests your ability to recommend storage solutions based on requirements for **performance, cost, scalability, availability, durability, and compliance**. You must understand when to choose relational vs. NoSQL, structured vs. unstructured, and how to integrate storage with compute and analytics services.

**Official skills measured (Study guide):**
- Design a data storage solution for relational data
- Design a data storage solution for non-relational data
- Design data integration

---

## 📊 Storage Types Overview Table

| Service | Type | Best For | Max Scale | SLA | Global Distribution |
|---------|------|----------|-----------|-----|---------------------|
| **Blob Storage (Block)** | Object | Unstructured data, media, backups, data lake | 5 PiB per account | 99.99% (RA-GRS) | Geo-replication (LRS/ZRS/GRS/RA-GRS/GZRS) |
| **Blob Storage (Append)** | Object | Log files, audit trails, streaming data | 195 GiB per blob | 99.99% (RA-GRS) | Same as Block Blob |
| **Blob Storage (Page)** | Object | VM disks (unmanaged), random read/write | 8 TiB per blob | 99.99% (RA-GRS) | Same as Block Blob |
| **Azure Files (SMB)** | File Share | Lift-and-shift, shared config, home directories | 100 TiB per share | 99.99% (ZRS) | Azure File Sync for hybrid |
| **Azure Files (NFS)** | File Share | Linux workloads, HPC, media rendering | 100 TiB per share | 99.99% (ZRS) | Premium tier only |
| **Data Lake Storage Gen2** | Object/Analytics | Big data analytics, hierarchical namespace | 5 PiB per account | 99.99% (RA-GRS) | Geo-replication |
| **Queue Storage** | Messaging | Async messaging, decoupling, task queues | 500 TiB per account | 99.99% (RA-GRS) | Geo-replication |
| **Table Storage** | NoSQL (Key-Value) | Semi-structured data, logs, device data | 500 TiB per account | 99.99% (RA-GRS) | Geo-replication |
| **Azure SQL Database** | Relational | OLTP, web apps, SaaS multitenant | 100 TB (Hyperscale) | 99.995% (BC + AZ) | Active geo-replication, failover groups |
| **Azure SQL Managed Instance** | Relational | SQL Server migration, cross-DB queries, CLR | 16 TB | 99.99% | Failover groups |
| **Azure Cosmos DB** | NoSQL (Multi-model) | Global apps, single-digit ms latency, IoT | Unlimited (partitioned) | 99.999% (multi-region) | Native multi-region writes |
| **Azure Synapse Analytics** | Analytics | Data warehousing, big data, integrated analytics | Unlimited (distributed) | 99.9% | N/A (single region) |
| **Microsoft Fabric** | Analytics | Unified analytics, lakehouse, real-time intelligence | SaaS-managed | 99.9% | Multi-region capacity |
| **Azure Databricks** | Analytics | ML/AI, Spark workloads, collaborative notebooks | Auto-scaling clusters | 99.95% | N/A (single region) |
| **Azure Cache for Redis** | Cache (In-memory) | Session state, caching, pub/sub, leaderboards | 1.2 TB (Enterprise) | 99.999% (Enterprise AZ) | Active geo-replication (Enterprise) |

---

## 🧠 Design Considerations Framework

### ⚡ Performance

#### Latency Requirements → Service Selection

| Latency Requirement | Recommended Service | Notes |
|---------------------|-------------------|-------|
| < 1 ms | Azure Cache for Redis | In-memory, local to compute |
| 1–10 ms | Cosmos DB, SQL DB (memory-optimized) | Single-digit ms guaranteed (Cosmos DB) |
| 10–50 ms | Azure SQL (General Purpose), Blob (Premium) | Standard transactional workloads |
| 50–200 ms | Standard Blob, Azure Files | Acceptable for batch/background |
| > 200 ms (acceptable) | Cool/Archive storage | Offline or batch access patterns |

#### IOPS Considerations

| Service | Max IOPS | Key Factor |
|---------|----------|------------|
| Premium SSD (Managed Disk) | 160,000 | Disk size determines IOPS allocation |
| Ultra Disk | 400,000 | Independently configurable IOPS/throughput |
| Azure Files (Premium) | 100,000 | Provisioned share size |
| Blob Storage (Premium) | Varies | Block blob size and concurrency |
| Cosmos DB | Unlimited | RU/s provisioning per partition |

#### Throughput vs. Latency Trade-offs

- **High throughput, latency-tolerant** → Blob Storage with large block sizes, Synapse for analytics
- **Low latency, moderate throughput** → Redis Cache, Cosmos DB with direct mode
- **Balanced** → Azure SQL with connection pooling, read replicas

#### Caching Strategies

```
┌─────────────────────────────────────────────────────────┐
│                    Caching Layers                         │
├─────────────────────────────────────────────────────────┤
│  L1: In-Process Cache (IMemoryCache, local dictionaries)│
│  L2: Distributed Cache (Azure Cache for Redis)          │
│  L3: CDN (Azure Front Door / CDN for static content)    │
│  L4: Database Read Replicas (SQL, Cosmos DB)            │
└─────────────────────────────────────────────────────────┘
```

| Pattern | Use When | Technology |
|---------|----------|------------|
| Cache-Aside | Data changes infrequently, read-heavy | Redis + app logic |
| Write-Through | Consistency critical, moderate writes | Redis + write pipeline |
| Write-Behind | High write volume, eventual consistency OK | Redis + background sync |
| Read Replica | SQL read-heavy, reporting workloads | SQL read scale-out |
| CDN | Static/semi-static global content | Azure CDN / Front Door |

#### Read Replicas & Read Scale-Out

- **Azure SQL Business Critical / Premium** → Built-in read replica (free), `ApplicationIntent=ReadOnly`
- **Azure SQL Hyperscale** → Up to 4 named replicas, geo-replicas
- **Cosmos DB** → Multi-region reads, consistent prefix for read replicas
- **Azure SQL Geo-Replication** → Up to 4 readable secondaries

#### Connection Pooling

- Always use connection pooling for SQL workloads (ADO.NET default: 100 connections)
- Azure Functions: Use static/singleton `SqlConnection` patterns
- Cosmos DB: One `CosmosClient` per application lifetime
- Redis: Use `ConnectionMultiplexer` as singleton

---

### 💰 Cost

#### Pay-Per-Use vs. Provisioned Capacity

| Model | Services | Best For | Risk |
|-------|----------|----------|------|
| **Pay-per-use** | Cosmos DB Serverless, Synapse Serverless, Blob (per-transaction) | Variable/unpredictable workloads | Cost spikes |
| **Provisioned** | Cosmos DB provisioned, SQL DTU/vCore, Premium storage | Predictable steady workloads | Over-provisioning waste |
| **Autoscale** | Cosmos DB Autoscale, SQL Serverless (vCore) | Variable with known ceiling | Minimum charge |

#### Reserved Capacity Discounts

| Service | 1-Year Discount | 3-Year Discount | Notes |
|---------|----------------|-----------------|-------|
| Azure SQL (vCore) | ~33% | ~55% | Exchangeable |
| Cosmos DB (RU/s) | ~20% | ~30% | Per-region |
| Synapse (DWU) | ~25% | ~40% | Committed use |
| Redis (Enterprise) | ~20% | ~35% | Per instance |
| Storage (capacity) | ~25% | ~38% | 100 TB+ commitments |

#### Storage Tiers & Lifecycle Management

```
Hot ──(30 days)──► Cool ──(90 days)──► Cold ──(180 days)──► Archive
 │                  │                    │                      │
 High access cost   Lower access cost   Lower access cost    Lowest storage cost
 Lowest storage $   Lower storage $     Lower storage $      Highest access cost
                                                              Hours to rehydrate
```

**Lifecycle management rules** automate tier transitions. Key exam point: Define rules based on `last modified` or `last accessed` time.

#### Transaction Costs vs. Storage Costs

| Scenario | Optimize For | Strategy |
|----------|-------------|----------|
| Large files, rare access | Storage cost | Cool/Archive tier |
| Small files, frequent access | Transaction cost | Hot tier, batch operations |
| Write-once, read-many | Balance | Hot tier → Cool after 30 days |
| Analytics data | Storage cost | ADLS Gen2 + Synapse serverless |

#### Egress & Cross-Region Transfer

- **Ingress**: Always free
- **Egress to internet**: $0.087/GB (first 10 TB/month)
- **Cross-region**: $0.02/GB (same continent)
- **Same region**: Free (within same VNet)
- **RA-GRS reads from secondary**: Charged as egress

> 💡 **Exam Tip**: When comparing multi-region architectures, always factor in egress costs. This is a common exam trap.

#### Right-Sizing Strategies

- Start with serverless/consumption, move to provisioned when patterns stabilize
- Monitor DTU/vCore usage — scale down if consistently < 40%
- Use elastic pools for SQL databases with variable usage patterns
- Cosmos DB: Monitor RU consumption, use autoscale to avoid 429s

---

### 📈 Scalability

#### Vertical vs. Horizontal Scaling

| Service | Vertical (Scale Up) | Horizontal (Scale Out) | Recommendation |
|---------|--------------------|-----------------------|----------------|
| Azure SQL DB | Up to 128 vCores (BC) | Read replicas, sharding | Scale up first, shard when limits hit |
| Azure SQL MI | Up to 80 vCores | Failover groups | Scale up; limited horizontal |
| Cosmos DB | N/A (always distributed) | Add regions, increase RU/s | Native horizontal |
| Blob Storage | N/A | Built-in (partition by path) | Design partition keys |
| Redis | Larger SKU | Clustering (up to 10 shards) | Cluster for > 53 GB |
| Synapse | More DWUs | MPP (built-in distribution) | Native horizontal |

#### Partitioning Strategies

| Service | Partition Key | Strategy |
|---------|--------------|----------|
| Cosmos DB | High-cardinality field | Avoid hot partitions; query within single partition |
| Table Storage | PartitionKey + RowKey | Group related entities; avoid single-partition bottleneck |
| Synapse | Distribution column | Hash, round-robin, or replicated tables |
| Blob/ADLS | Path hierarchy | Organize by date/entity for parallel access |
| Event Hubs | Partition ID | Scale consumers with partitions |

#### Sharding Patterns

| Pattern | Description | Use Case |
|---------|-------------|----------|
| **Range-based** | Shard by date range or ID range | Time-series, sequential IDs |
| **Hash-based** | Consistent hash of shard key | Even distribution, random access |
| **Geo-based** | Shard by region/geography | Multi-region compliance |
| **Tenant-based** | One shard per tenant (or group) | SaaS multi-tenancy |

#### Auto-Scale Capabilities

| Service | Auto-Scale Type | Min/Max | Scaling Speed |
|---------|----------------|---------|---------------|
| Cosmos DB (Autoscale) | RU/s (10% to 100% of max) | 100–unlimited RU/s | Instant |
| Azure SQL Serverless | vCores (auto-pause capable) | 0.5–80 vCores | Seconds |
| Synapse Serverless | Automatic | No provisioning needed | Instant |
| Azure Functions | Instances (event-driven) | 0–200 (Consumption) | Seconds |
| Redis (Enterprise) | Manual | Tier-based | Minutes |

#### Scale Units Table

| Service | Scale Unit | Limit per Unit | Max Units |
|---------|-----------|---------------|-----------|
| Storage Account | Account | 500 TB, 20K IOPS (std) | 250 per subscription |
| Cosmos DB | Physical Partition | 50 GB, 10K RU/s | Unlimited partitions |
| Azure SQL (Hyperscale) | Compute replica | 80 vCores each | 4 named + geo |
| Synapse Dedicated | DWU | 100–30000 DWU | Per pool |
| Event Hubs | Throughput Unit | 1 MB/s in, 2 MB/s out | 40 TUs (standard) |

---

### 🔀 Access Patterns

#### Read-Heavy vs. Write-Heavy Workloads

| Pattern | Characteristics | Recommended Services | Design Strategy |
|---------|----------------|---------------------|-----------------|
| **Read-heavy (90/10)** | Catalogs, CMS, reference data | SQL + Redis, Cosmos DB, CDN | Cache aggressively, read replicas |
| **Write-heavy (10/90)** | Logging, IoT telemetry, event streams | Event Hubs, Append Blob, Cosmos DB | Write-optimized paths, async writes |
| **Balanced (50/50)** | E-commerce, social apps | Cosmos DB, Azure SQL | Careful indexing, partitioning |
| **Write-once/Read-many** | Archives, compliance, audit | Blob (WORM), Immutable storage | Lifecycle policies, cheap storage |

#### Random vs. Sequential Access

| Access Pattern | Storage Choice | Why |
|---------------|---------------|-----|
| Random read/write | Cosmos DB, SQL, Redis | Indexed lookup, O(1) access |
| Sequential scan | ADLS Gen2, Synapse, Blob | Streaming reads, columnar formats |
| Random read + Sequential write | Append Blob, Table Storage | Append-only + key lookup |

#### Hot/Warm/Cold Data Classification

| Classification | Access Frequency | Latency Tolerance | Storage Choice |
|---------------|-----------------|-------------------|----------------|
| **Hot** | Multiple times/day | < 10 ms | Redis, SQL (in-memory), Hot Blob |
| **Warm** | Weekly/monthly | < 100 ms | Cool Blob, SQL (standard), Cosmos DB |
| **Cold** | Quarterly/yearly | < 1 hour | Cold Blob, Archive Blob |
| **Frozen** | Rarely/compliance only | Hours to days | Archive Blob with rehydration |

#### Time-Series Data Patterns

| Approach | Service | Best For |
|----------|---------|----------|
| Append-only log | Append Blob, ADLS Gen2 | Raw ingestion, cheap storage |
| Windowed aggregation | Azure Data Explorer (Kusto) | Real-time analytics on time-series |
| Partitioned by time | Cosmos DB (partition by hour/day) | Query-intensive time-series |
| Columnar analytics | Synapse (Parquet files) | Historical time-series analytics |

#### OLTP vs. OLAP Workloads

| Characteristic | OLTP | OLAP |
|---------------|------|------|
| **Operations** | INSERT, UPDATE, DELETE | SELECT (complex aggregations) |
| **Data model** | Normalized (3NF) | Denormalized (star/snowflake) |
| **Query scope** | Single/few rows | Millions of rows |
| **Concurrency** | Thousands of users | Few analysts/reports |
| **Service** | Azure SQL, Cosmos DB | Synapse, Fabric, Databricks |
| **Latency** | Milliseconds | Seconds to minutes |

#### Event-Driven Patterns

| Pattern | Trigger | Services | Use Case |
|---------|---------|----------|----------|
| Change feed → Process | Document change | Cosmos DB Change Feed + Functions | Real-time materialized views |
| Blob trigger → Transform | New blob | Blob Storage + Functions/Logic Apps | Image processing, ETL |
| Queue → Worker | New message | Queue Storage/Service Bus + Functions | Decoupled background processing |
| Event Grid → React | Resource event | Any Azure resource + Event Grid | Cross-service orchestration |

---

## 🔗 Data Platform Integration

### Storage → Compute Patterns

```
┌──────────────────────────────────────────────────────────────────┐
│                      Compute Services                             │
├────────────────┬───────────────┬──────────────┬─────────────────┤
│ Azure Functions│ App Service   │Container Apps│ AKS             │
├────────────────┴───────────────┴──────────────┴─────────────────┤
│                       ▲ ▲ ▲ ▲                                    │
│            ┌──────────┘ │ │ └──────────┐                        │
│            │            │ │            │                         │
│   ┌────────▼──┐  ┌─────▼─▼───┐  ┌────▼─────┐  ┌────────────┐ │
│   │ Blob/ADLS │  │ SQL/Cosmos │  │  Redis   │  │ Event Hubs │ │
│   └───────────┘  └───────────┘  └──────────┘  └────────────┘ │
└──────────────────────────────────────────────────────────────────┘
```

| Compute | Best Storage Pair | Connection Pattern |
|---------|-------------------|-------------------|
| Azure Functions | Queue, Blob, Cosmos DB, Event Hubs | Bindings (trigger + input/output) |
| App Service | SQL, Redis, Blob | Connection strings, Managed Identity |
| Container Apps | Cosmos DB, Redis, Service Bus | Dapr state stores, direct SDK |
| AKS | Any (via CSI drivers, SDKs) | Persistent Volumes, SDK connections |

### Storage → Analytics Patterns

| Source | Analytics Engine | Integration |
|--------|-----------------|-------------|
| Blob/ADLS Gen2 | Synapse Serverless | External tables, OPENROWSET |
| Blob/ADLS Gen2 | Databricks | Direct mount, Unity Catalog |
| Blob/ADLS Gen2 | Fabric | OneLake shortcuts |
| Cosmos DB | Synapse Link | No-ETL analytical store |
| SQL Database | Synapse Pipelines | Copy activity, dataflows |
| Event Hubs | Fabric Real-Time Intelligence | Eventhouse KQL |

### Event-Driven Integration

| Service | Event Mechanism | Consumers | Latency |
|---------|----------------|-----------|---------|
| Blob Storage | Event Grid events | Functions, Logic Apps | Seconds |
| Cosmos DB | Change Feed | Functions, custom processor | Near real-time |
| Queue Storage | Queue trigger | Functions, WebJobs | Milliseconds–seconds |
| Service Bus | Topic/Queue trigger | Functions, Logic Apps | Milliseconds |
| Event Hubs | Event processor | Functions, Stream Analytics | Milliseconds |
| SQL Database | Change tracking | Custom sync, ADF | Minutes |

### ETL vs. ELT Patterns

| Pattern | Description | When to Use | Azure Services |
|---------|-------------|-------------|----------------|
| **ETL** | Extract → Transform → Load | Structured sources, schema-on-write | ADF + Dataflows, SSIS |
| **ELT** | Extract → Load → Transform | Big data, schema-on-read, data lakes | ADF → ADLS → Synapse/Databricks |
| **Streaming ETL** | Continuous transform | Real-time requirements | Event Hubs → Stream Analytics → SQL |
| **Change Data Capture** | Incremental sync | Keep target in sync | ADF CDC, Debezium, SQL CT |

> 💡 **Exam Tip**: ELT is the modern pattern for analytics. ETL is legacy but still valid for structured OLTP targets.

### Data Movement Services

| Service | Best For | Scale | Use Case |
|---------|----------|-------|----------|
| **Azure Data Factory** | Orchestrated pipelines | Cloud-scale | Scheduled ETL/ELT, 90+ connectors |
| **AzCopy** | Bulk transfer (CLI) | Multi-threaded | Migration, one-time copies |
| **Data Box** | Offline transfer | 100 TB–1 PB | Initial migration, poor connectivity |
| **Azure Migrate** | Database migration | Online/offline | SQL Server → Azure SQL |
| **Storage Mover** | File share migration | Millions of files | NAS → Azure Files |

### Hybrid & Multi-Cloud Data Strategies

| Strategy | Services | Scenario |
|----------|----------|----------|
| **Azure Arc-enabled SQL** | SQL MI, PostgreSQL on-prem | Unified management, hybrid |
| **Azure File Sync** | On-prem file server + Azure Files | Hybrid file access, tiering |
| **ExpressRoute + Private Endpoint** | Any storage service | Secure hybrid connectivity |
| **Azure Stack HCI** | Storage Spaces Direct | On-premises Azure-consistent |

---

## 🌳 Decision Flowchart: Which Azure Data Service?

```
                         ┌─────────────────────┐
                         │ What type of data?   │
                         └──────────┬──────────┘
                                    │
                 ┌──────────────────┼──────────────────┐
                 ▼                  ▼                  ▼
          ┌────────────┐    ┌────────────┐     ┌────────────┐
          │ Structured │    │Semi-struct/ │     │Unstructured│
          │ (Relational)│    │  NoSQL     │     │(Files/Blobs)│
          └──────┬─────┘    └──────┬─────┘     └──────┬─────┘
                 │                  │                   │
                 ▼                  ▼                   ▼
    ┌────────────────────┐  ┌──────────────┐   ┌──────────────┐
    │Need SQL Server     │  │Global dist.? │   │Analytics/    │
    │compatibility?      │  │Multi-model?  │   │Big Data?     │
    └─────┬────────┬─────┘  └──┬───────┬──┘   └──┬───────┬───┘
          │        │            │       │          │       │
         YES      NO          YES     NO         YES     NO
          │        │            │       │          │       │
          ▼        ▼            ▼       ▼          ▼       ▼
    ┌──────────┐ ┌─────┐  ┌────────┐ ┌─────┐ ┌───────┐ ┌──────┐
    │SQL MI or │ │Azure│  │Cosmos  │ │Table│ │ADLS   │ │Blob  │
    │SQL on VM │ │SQL  │  │DB      │ │Stg  │ │Gen2   │ │Stg   │
    └──────────┘ │DB   │  └────────┘ └─────┘ └───────┘ └──────┘
                 └─────┘

    ┌─────────────────────────────────────────────────────────────┐
    │              ANALYTICS DECISION                              │
    ├─────────────────────────────────────────────────────────────┤
    │                                                             │
    │  Need real-time + historical? ──► Fabric Real-Time Intel.  │
    │  Enterprise DW + governed? ──────► Synapse Dedicated Pool  │
    │  Ad-hoc queries on data lake? ──► Synapse Serverless       │
    │  ML/AI + collaborative? ─────────► Databricks              │
    │  Unified SaaS analytics? ────────► Microsoft Fabric        │
    │  Sub-second caching? ────────────► Redis Cache             │
    │                                                             │
    └─────────────────────────────────────────────────────────────┘
```

### Quick Decision Matrix

| Requirement | → Service |
|-------------|-----------|
| Need ACID transactions + joins | Azure SQL Database |
| SQL Server features (Agent, CLR, cross-DB) | SQL Managed Instance |
| Global distribution, < 10ms latency | Cosmos DB |
| Store blobs/files cheaply | Blob Storage (Cool/Archive) |
| Shared file system (SMB) | Azure Files |
| Big data analytics (Spark) | ADLS Gen2 + Databricks/Synapse |
| Data warehouse (star schema) | Synapse Dedicated Pools |
| Serverless SQL over data lake | Synapse Serverless |
| Unified analytics platform (SaaS) | Microsoft Fabric |
| Session state / caching | Azure Cache for Redis |
| Message queue (simple) | Queue Storage |
| Message queue (enterprise) | Service Bus |
| IoT time-series telemetry | Azure Data Explorer / Cosmos DB |

---

## 📖 Cheat Sheet Navigation

| Topic | Link | Covers |
|-------|------|--------|
| 🗃️ Azure Storage Accounts | [Azure-Storage-Accounts.md](./Azure-Storage-Accounts.md) | Blob, Files, Queue, Table, Data Lake, redundancy, access tiers, lifecycle |
| 🗄️ Azure SQL | [Azure-SQL.md](./Azure-SQL.md) | SQL Database, Managed Instance, Hyperscale, Elastic Pools, VMs, DTU vs vCore |
| 🌐 Azure Cosmos DB | [Azure-CosmosDB.md](./Azure-CosmosDB.md) | Multi-model APIs, global distribution, consistency levels, partitioning, RU/s |
| 📊 Azure Data & Analytics | [Azure-Data-Analytics.md](./Azure-Data-Analytics.md) | Synapse, Fabric, Databricks, Data Lake, ETL/ELT, real-time analytics |
| 🚚 Azure Migration Strategies | [Azure-Migration-Strategies.md](./Azure-Migration-Strategies.md) | All database migrations, DMS, SSMA, Data Box, BACPAC, PowerShell/CLI commands |

---

## 🧪 Labs Navigation

| Lab | Link | Focus |
|-----|------|-------|
| 🗃️ Azure Storage Labs | [Azure-Storage-Labs.md](../Labs/Azure-Storage-Labs.md) | Create storage accounts, configure tiers, lifecycle policies, SAS tokens |
| 🗄️ Azure SQL Labs | [Azure-SQL-Labs.md](../Labs/Azure-SQL-Labs.md) | Deploy SQL DB, configure geo-replication, elastic pools, failover groups |
| 🌐 Azure Cosmos DB Labs | [Azure-CosmosDB-Labs.md](../Labs/Azure-CosmosDB-Labs.md) | Create Cosmos DB, configure consistency, partition strategies, change feed |
| 📊 Azure Data & Analytics Labs | [Azure-Data-Analytics-Labs.md](../Labs/Azure-Data-Analytics-Labs.md) | Synapse workspace, Spark pools, Fabric lakehouse, data pipelines |
| 🚚 Azure Migration Labs | [Azure-Migration-Labs.md](../Labs/Azure-Migration-Labs.md) | DMS, BACPAC, LRS, AzCopy, File Sync, PostgreSQL, MySQL, MongoDB, Oracle |

---

## 🎯 AZ-305 Exam Tips for Data Storage

### Common Exam Traps

| Trap | What They Test | Correct Answer |
|------|---------------|----------------|
| "Need SQL Server Agent" | SQL DB vs MI | → **SQL Managed Instance** (DB doesn't have Agent) |
| "Globally distributed, strong consistency" | Cosmos DB consistency | → Possible but **highest latency & lowest throughput** |
| "Minimize cost for infrequently accessed data" | Tier selection | → **Cool or Archive** (not Hot) with lifecycle rules |
| "HIPAA/compliance + cross-database queries" | SQL option | → **SQL MI** (compliance + cross-DB) |
| "Serverless, unpredictable traffic" | Provisioned vs serverless | → **Cosmos DB Serverless** or **SQL Serverless** |
| "99.999% availability" | SLA comparison | → **Cosmos DB multi-region** (only service with 5-nines) |
| "Migrate on-prem SQL with minimal changes" | Migration strategy | → **SQL MI** (near 100% compatibility) |
| "Need to query JSON natively" | SQL vs Cosmos DB | → Could be either (SQL supports JSON), read carefully |
| "Large-scale analytics, Parquet files" | Analytics service | → **Synapse Serverless** or **Databricks** over ADLS Gen2 |
| "Real-time dashboard from Cosmos DB" | Integration | → **Synapse Link** (no ETL needed) |

### Decision Patterns Microsoft Expects

1. **"Least cost" + requirements met** → Always pick the cheapest option that satisfies ALL requirements
2. **"Minimize administrative effort"** → Prefer PaaS/Serverless over IaaS
3. **"Ensure data residency"** → Consider sovereign regions, geo-replication constraints
4. **"Maximize availability"** → Multi-region + zone-redundant + appropriate SLA tier
5. **"Minimize latency globally"** → Cosmos DB multi-region writes OR Azure Front Door + regional deployments

### Key Concepts to Memorize

#### Cosmos DB Consistency Levels (Strongest → Weakest)
```
Strong → Bounded Staleness → Session → Consistent Prefix → Eventual
  │              │                │              │               │
  │     Configurable lag    Default (most    Reads never    Lowest latency
  │     (time or ops)       common choice)   see out-of-   highest throughput
  │                                          order writes
Linearizable
reads
```

#### Azure SQL Purchasing Models
```
DTU Model                          vCore Model
├── Basic (5 DTUs)                 ├── General Purpose (serverless/provisioned)
├── Standard (10-3000 DTUs)        ├── Business Critical (local SSD, read replica)
└── Premium (125-4000 DTUs)        └── Hyperscale (100TB, rapid scale, HA replicas)
```

#### Storage Redundancy (Know the RPO/RTO)
| Option | Copies | Durability | RPO (Geo) | Use Case |
|--------|--------|-----------|-----------|----------|
| LRS | 3 (single DC) | 11 nines | N/A | Dev/test, non-critical |
| ZRS | 3 (3 zones) | 12 nines | N/A | High availability, single region |
| GRS | 6 (2 regions) | 16 nines | < 15 min | DR, cross-region protection |
| RA-GRS | 6 (2 regions) | 16 nines | < 15 min | DR + read from secondary |
| GZRS | 6 (3 zones + 1 region) | 16 nines | < 15 min | Maximum durability + HA |
| RA-GZRS | 6 (3 zones + 1 region) | 16 nines | < 15 min | Maximum everything |

### Final Exam Reminders

- ✅ **Always read ALL answer options** — Microsoft loves "almost right" answers
- ✅ **Look for constraints** — "minimize cost," "minimize latency," "maximize availability"
- ✅ **Multi-service solutions are common** — It's rarely just one service
- ✅ **Know the limits** — 5 PiB storage account, 100 TB Hyperscale, 50 GB Cosmos partition
- ✅ **Managed Identity > connection strings** — Always prefer passwordless
- ✅ **Private endpoints > service endpoints** — For exam, private endpoints are preferred
- ✅ **Lifecycle management is automatic** — Don't manually move blobs between tiers at scale

---

> 📅 *Last updated: July 2025*
>
> 📝 *Part of the AZ-305 Azure Solutions Architect Expert study guide series*
