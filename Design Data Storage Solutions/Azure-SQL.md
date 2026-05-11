# Azure SQL - AZ-305 Comprehensive Cheat Sheet

> 📝 **Hands-On Labs:** [Azure SQL Labs](../Labs/Azure-SQL-Labs.md)

> 🎯 **Exam Focus:** AZ-305 tests your ability to **choose** the right Azure SQL deployment option, tier, and configuration based on business requirements — not just know what they are.

---

## 1. Azure SQL Family Overview

| Feature | SQL Database (Single) | SQL Database (Elastic Pool) | SQL Managed Instance | SQL Server on Azure VM (IaaS) |
|---------|----------------------|---------------------------|---------------------|-------------------------------|
| **Deployment Model** | PaaS - Single database | PaaS - Shared resources | PaaS - Instance-scoped | IaaS - Full VM control |
| **SQL Server Compatibility** | ~95% | ~95% | ~99% | 100% |
| **Cross-database queries** | ❌ (use Elastic Query) | ✅ Within pool | ✅ Full support | ✅ Full support |
| **CLR** | ❌ | ❌ | ✅ | ✅ |
| **SQL Agent** | ❌ (use Elastic Jobs) | ❌ (use Elastic Jobs) | ✅ | ✅ |
| **Service Broker** | ❌ | ❌ | ✅ (within instance) | ✅ |
| **Linked Servers** | ❌ | ❌ | ✅ | ✅ |
| **FILESTREAM** | ❌ | ❌ | ❌ | ✅ |
| **Database Mail** | ❌ | ❌ | ✅ | ✅ |
| **Max Size** | 128 TB (Hyperscale) | 4 TB (GP) / 4 TB (BC) | 16 TB | Unlimited |
| **Maintenance Control** | Limited | Limited | Maintenance windows | Full control |
| **OS Access** | ❌ | ❌ | ❌ | ✅ Full RDP/SSH |
| **Pricing** | Per database | Shared eDTU/vCore | Per instance | VM + License |
| **Best For** | Modern cloud apps | Multi-tenant SaaS | Lift-and-shift | Legacy apps, full control |

> 💡 **AZ-305 Tip:** If the question mentions "lift and shift" with minimal code changes, think **Managed Instance**. If it mentions FILESTREAM or full OS access, it's **SQL on VM**.

### Azure SQL Database (Single Database)
A fully managed PaaS relational database engine that handles patching, backups, monitoring, and high availability with zero administration. Each database is isolated with its own guaranteed resources. Supports **Serverless** compute (auto-pause/auto-scale for intermittent workloads) and **Hyperscale** tier (up to 128 TB with near-instant backups and read scale-out replicas). Ideal for **modern cloud-native applications** that need a single, self-contained database without instance-level features like SQL Agent or CLR. You design your schema, Microsoft manages everything else.

**Real-World Examples:**
- An **e-commerce startup** launches a new product catalog API backed by a single SQL database — scales from General Purpose to Hyperscale as the catalog grows to millions of products
- A **mobile app backend** (e.g., fitness tracker) uses Serverless tier so the database auto-pauses overnight when no users are active, keeping dev/test costs near zero
- A **healthcare portal** stores patient appointment data in a single database with Business Critical tier for sub-millisecond reads and zone-redundant HA

### Azure SQL Database (Elastic Pool)
Shares a pool of compute and storage resources across multiple databases within the same logical server. Databases in the pool can burst up to the pool's resource limits, making it **cost-effective when you have many databases with unpredictable, spiky usage patterns** — some databases peak while others are idle. Perfect for **multi-tenant SaaS applications** where each tenant gets their own database. You manage one resource pool instead of individually sizing dozens or hundreds of databases. Supports both DTU and vCore purchasing models.

**Real-World Examples:**
- A **SaaS HR platform** provisions a separate database per company customer (200+ tenants) — each tenant's usage is unpredictable, but in aggregate the pool stays steady at ~60% utilization
- A **property management company** runs 50 databases (one per building) — month-end reporting spikes on a few buildings at a time, while the rest sit idle; the pool absorbs bursts without over-provisioning each database
- A **consulting firm** builds internal apps where each department gets its own database — the elastic pool caps total spend while letting any single department burst during quarter-end

### Azure SQL Managed Instance
The closest PaaS migration target to an on-premises SQL Server. Provides **near 100% compatibility** with the SQL Server engine, including support for SQL Agent jobs, CLR assemblies, cross-database queries, Service Broker, linked servers, and Database Mail. Deployed into a dedicated VNet subnet, giving you network isolation by default. Designed for **lift-and-shift migrations** where applications rely on instance-scoped features and you want to move to the cloud with minimal code changes. Microsoft handles OS patching, backups, and HA, but you get the familiar SQL Server surface area.

**Real-World Examples:**
- A **bank** migrates its on-premises core banking SQL Server that uses 30+ SQL Agent jobs for nightly ETL, CLR stored procedures for custom business logic, and cross-database queries between accounts and transactions databases — Managed Instance runs it all without code changes
- An **insurance company** moves a legacy claims processing system that uses Service Broker for asynchronous message processing between databases — the only PaaS option that supports this
- A **government agency** requires full VNet isolation for compliance — Managed Instance deploys directly into their private subnet with no public endpoint, while still being fully managed

### SQL Server on Azure VMs (IaaS)
A full SQL Server instance running on an Azure virtual machine, giving you **100% feature compatibility** and complete control over the OS, SQL Server configuration, maintenance schedules, and storage layout. Required when you need features unavailable in PaaS options: FILESTREAM, SSIS/SSRS installed on the same server, distributed transactions across instances, or custom third-party software alongside SQL Server. Also the right choice for databases exceeding PaaS size limits or for applications requiring specific SQL Server versions. You're responsible for patching, backups, and HA configuration — though Azure provides automated backup and patching extensions to reduce the burden.

**Real-World Examples:**
- A **manufacturing company** runs a legacy ERP system (e.g., older SAP version) that requires FILESTREAM for storing engineering drawings alongside relational data, plus a third-party monitoring agent installed on the same server — only IaaS supports this
- A **media company** runs SSIS packages that extract data from 15 external FTP sources, transform it, and load into SQL Server, with SSRS reports generated on-box — the full BI stack needs a VM
- A **hospital system** runs SQL Server 2016 with a custom Always On Availability Group topology across 5 replicas with specific OS-level tuning and a third-party encryption tool that requires kernel access — full VM control is the only option
- An **analytics firm** runs a 200 TB data warehouse on SQL Server that exceeds PaaS size limits and needs specific tempdb and filegroup configurations for query performance

---

## 2. When to Choose Which — Decision Tree

```
┌─────────────────────────────────────────────────────────┐
│           Need 100% SQL Server compatibility?            │
│        (FILESTREAM, bulk insert from file, SSIS,        │
│         cross-instance distributed transactions)         │
└─────────────────────┬───────────────────────────────────┘
                      │
            ┌─────────┴─────────┐
            │ YES               │ NO
            ▼                   ▼
   ┌────────────────┐  ┌───────────────────────────────┐
   │ SQL Server on  │  │ Need instance-scoped features? │
   │   Azure VM     │  │ (SQL Agent, CLR, cross-DB      │
   │   (IaaS)       │  │  queries, Service Broker,      │
   └────────────────┘  │  linked servers, DB Mail)      │
                       └───────────────┬───────────────┘
                                       │
                             ┌─────────┴─────────┐
                             │ YES               │ NO
                             ▼                   ▼
                    ┌────────────────┐  ┌───────────────────────┐
                    │ SQL Managed    │  │ Multiple DBs with      │
                    │  Instance      │  │ variable workloads?    │
                    └────────────────┘  └───────────┬───────────┘
                                                    │
                                          ┌─────────┴─────────┐
                                          │ YES               │ NO
                                          ▼                   ▼
                                 ┌────────────────┐  ┌────────────────┐
                                 │ Elastic Pool   │  │ Single Database│
                                 └────────────────┘  └────────────────┘
```

### Quick Decision Matrix

| Scenario | Choose |
|----------|--------|
| New cloud-native microservice | SQL Database (Single) |
| Multi-tenant SaaS with 50+ tenant DBs | Elastic Pool |
| Migrating on-prem SQL Server with Agent jobs & CLR | Managed Instance |
| Need SSIS, SSRS, FILESTREAM | SQL Server on VM |
| Need >100TB database | SQL Server on VM |
| Minimal refactoring, many features used | Managed Instance |
| Unpredictable, intermittent workloads | SQL Database Serverless |
| Need fastest possible failover | Business Critical tier |
| Near-unlimited scale + fast backup | Hyperscale |

---

## 3. Purchasing Models — DTU vs vCore

### DTU (Database Transaction Unit)

- **Bundled** measure of CPU + Memory + I/O
- **Simple, predictable** pricing — choose a DTU level
- **Cannot independently scale** compute vs storage
- Available in: Basic, Standard, Premium tiers

### vCore (Virtual Core)

- **Independent control** over compute (vCores) and storage
- **Azure Hybrid Benefit** eligible (use existing SQL licenses)
- **Reserved capacity** discounts (1-year or 3-year)
- Available in: General Purpose, Business Critical, Hyperscale tiers

### Comparison Table

| Aspect | DTU | vCore |
|--------|-----|-------|
| **Pricing Model** | Bundled (CPU+RAM+IO) | Separate compute + storage |
| **Scalability** | Fixed tiers | Granular vCore selection |
| **Azure Hybrid Benefit** | ❌ | ✅ (save up to 55%) |
| **Reserved Capacity** | ❌ | ✅ (save up to 33%) |
| **Serverless Option** | ❌ | ✅ (General Purpose only) |
| **Hyperscale** | ❌ | ✅ |
| **Hardware Selection** | Fixed | Gen5, Fsv2, DC-series, M-series |
| **Best For** | Simple workloads, getting started | Production, cost optimization, hybrid |

> 💡 **AZ-305 Tip:** If the question mentions **Azure Hybrid Benefit** or **existing SQL Server licenses**, the answer requires **vCore**. DTU does NOT support AHB.

### When to Choose DTU
- Simple, predictable workloads
- Don't need fine-grained resource control
- Getting started / dev/test
- Don't have existing SQL Server licenses

### When to Choose vCore
- Need Azure Hybrid Benefit (existing licenses)
- Want reserved capacity discounts
- Need independent compute/storage scaling
- Need Hyperscale or Serverless
- Want hardware flexibility (memory-optimized, etc.)
- Migration from on-premises (easier to map cores)

---

## 4. Service Tiers — Detailed Comparison

### vCore Service Tiers

| Feature | General Purpose | Business Critical | Hyperscale |
|---------|----------------|-------------------|------------|
| **Target Workload** | Budget-friendly, balanced | High IOPS, low-latency | Large scale, flexible |
| **Compute** | 2-128 vCores | 2-128 vCores | 2-128 vCores |
| **Max Storage** | 4 TB (single) / 16 TB (MI) | 4 TB (single) / 16 TB (MI) | 128 TB |
| **Storage Type** | Remote (Azure Premium Storage) | Local SSD (fast) | Tiered (local SSD + remote) |
| **IO Latency** | ~5-10 ms | ~1-2 ms | ~1-2 ms (comparable to BC for hot data) |
| **Availability SLA** | 99.99% (with zone redundancy) | 99.99% (with zone redundancy) | 99.99% (with zone redundancy) |
| **Read Replicas** | ❌ (0 included) | 1 included (free) | Up to 4 HA replicas + 30 named replicas |
| **Serverless** | ✅ | ❌ | ✅ |
| **Zone Redundancy** | ✅ (optional) | ✅ (optional) | ✅ (optional) |
| **Backup Storage** | Up to 4 TB | Up to 4 TB | Up to 128 TB |
| **PITR Retention** | 1-35 days | 1-35 days | 1-35 days |
| **HA Architecture** | Remote storage + failover | Always On AG (synchronous) | Distributed page servers |
| **Failover Time** | ~30 seconds | ~30 seconds | ~30 seconds |
| **Best For** | Most workloads, cost-sensitive | OLTP, mission-critical | Large DBs, fast scaling, fast backup |

### DTU Service Tiers

| Feature | Basic | Standard (S0-S12) | Premium (P1-P15) |
|---------|-------|-------------------|------------------|
| **Max DTUs** | 5 | 3000 | 4000 |
| **Max Storage** | 2 GB | 1 TB | 4 TB |
| **PITR** | 7 days | 35 days | 35 days |
| **Read Scale-Out** | ❌ | ❌ | ✅ (P1+) |
| **In-Memory OLTP** | ❌ | ❌ | ✅ |
| **Zone Redundancy** | ❌ | ❌ | ✅ (optional) |
| **Approx. Mapping** | — | General Purpose | Business Critical |

> 💡 **AZ-305 Tip:** Premium DTU ≈ Business Critical vCore. Standard DTU ≈ General Purpose vCore. Use this mapping when converting.

---

## 5. Hyperscale Deep Dive

### Architecture

```
┌──────────────────────────────────────────────────────────────┐
│                    HYPERSCALE ARCHITECTURE                     │
├──────────────────────────────────────────────────────────────┤
│                                                               │
│  ┌─────────────┐     ┌─────────────┐     ┌─────────────┐   │
│  │  Primary    │     │ HA Replica  │     │ HA Replica  │   │
│  │  Compute    │     │   (up to 4) │     │             │   │
│  └──────┬──────┘     └──────┬──────┘     └──────┬──────┘   │
│         │                    │                    │           │
│         ▼                    ▼                    ▼           │
│  ┌─────────────────────────────────────────────────────┐    │
│  │              LOG SERVICE (Distributed)                │    │
│  └─────────────────────────┬───────────────────────────┘    │
│                             │                                 │
│         ┌───────────────────┼───────────────────┐            │
│         ▼                   ▼                   ▼            │
│  ┌─────────────┐   ┌─────────────┐   ┌─────────────┐      │
│  │ Page Server │   │ Page Server │   │ Page Server │      │
│  │  (1-n)      │   │  (1-n)      │   │  (1-n)      │      │
│  └─────────────┘   └─────────────┘   └─────────────┘      │
│         │                   │                   │            │
│         ▼                   ▼                   ▼            │
│  ┌─────────────────────────────────────────────────────┐    │
│  │          AZURE BLOB STORAGE (Snapshots)              │    │
│  └─────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────┘
```

### Key Capabilities

| Feature | Details |
|---------|---------|
| **Max Database Size** | 128 TB |
| **Near-Instant Backups** | Snapshot-based — minutes regardless of DB size |
| **Fast Restore** | Restore a 10 TB DB in minutes, not hours |
| **Rapid Scale-Out** | Add read replicas in ~2 minutes |
| **Rapid Scale-Up** | Vertical scaling is fast (data doesn't move) |
| **HA Replicas** | Up to 4 (synchronous, automatic failover) |
| **Named Replicas** | Up to 30 (independent compute, own connection string) |
| **Geo-Replicas** | Up to 4 geo-replicas for DR |
| **Read Scale-Out** | Free with HA replicas; Named replicas for granular control |
| **Serverless** | ✅ Supported (auto-pause, auto-scale) |
| **Zone Redundancy** | ✅ Supported |
| **Reverse Migration** | ✅ Can migrate back to General Purpose (if <1 TB) |

### Named Replicas vs HA Replicas

| Aspect | HA Replicas | Named Replicas |
|--------|-------------|----------------|
| **Purpose** | High availability | Read scale-out, workload isolation |
| **Compute** | Same as primary | Independent (can differ) |
| **Connection** | Via primary or read-only routing | Own connection string |
| **Count** | Up to 4 | Up to 30 |
| **Billing** | Included in primary | Billed separately |
| **Use Case** | Failover, basic read scale | Reporting, analytics, per-tenant isolation |
| **Access Control** | Same as primary | Can have different users/permissions |

> 💡 **AZ-305 Tip:** If a scenario requires **independent security contexts** for different read workloads or **per-tenant read isolation**, use **Named Replicas**. If it just needs HA or basic read scale-out, HA replicas suffice.

### When to Choose Hyperscale

- Database will grow beyond 4 TB
- Need fast backups regardless of DB size
- Need rapid scale up/down without data movement
- Need many read replicas (up to 30 named)
- Need fast point-in-time restore for large databases
- Want to start small and grow without tier changes

---

## 6. Elastic Pools

### Concept

Elastic Pools allow multiple databases to **share a pool of resources** (eDTUs or vCores), optimizing cost when databases have **variable and unpredictable** usage patterns.

```
┌──────────────────────────────────────────────────┐
│           ELASTIC POOL (200 eDTUs)                │
│                                                   │
│  ┌─────┐  ┌─────┐  ┌─────┐  ┌─────┐           │
│  │DB-A │  │DB-B │  │DB-C │  │DB-D │           │
│  │Peak:│  │Peak:│  │Peak:│  │Peak:│           │
│  │100  │  │ 80  │  │ 50  │  │ 70  │           │
│  │eDTU │  │eDTU │  │eDTU │  │eDTU │           │
│  └─────┘  └─────┘  └─────┘  └─────┘           │
│                                                   │
│  Total peak demand if simultaneous = 300          │
│  But peaks don't overlap → 200 eDTUs works!      │
└──────────────────────────────────────────────────┘
```

### Key Parameters

| Setting | DTU Model | vCore Model |
|---------|-----------|-------------|
| **Pool Resources** | 50-4000 eDTUs | 2-128 vCores |
| **Per-DB Min** | 0-pool max | 0-pool max |
| **Per-DB Max** | 5-pool max | 0.25-pool max |
| **Max DBs per pool** | 500 (GP) | 500 (GP) |
| **Max Storage** | 4 TB | 4 TB |
| **Service Tiers** | Standard, Premium | General Purpose, Business Critical, Hyperscale |

### When to Use Elastic Pools

✅ **Good Fit:**
- Multi-tenant SaaS (database-per-tenant model)
- Multiple databases with **different peak times**
- Average utilization per DB is < 50% of peak
- Unpredictable workloads across databases
- Want to manage cost for many small databases

❌ **Bad Fit:**
- All databases peak simultaneously
- Single database with consistent high utilization
- Databases need different service tiers
- Need Hyperscale features (use Hyperscale named replicas instead)

### Cost Optimization Formula

```
If: (Number of DBs) × (avg DTU per DB) < Pool eDTUs needed
And: Sum of individual DB costs > Pool cost
Then: Elastic Pool saves money
```

> 💡 **AZ-305 Tip:** Classic scenario — "Company has 50 databases that each spike to S3 (100 DTU) but average only 20 DTU." → **Elastic Pool** (200-400 eDTU pool instead of 50 × 100 DTU individual).

---

## 7. Serverless Compute

### How It Works

| Aspect | Details |
|--------|---------|
| **Auto-Scale** | Scales compute up/down based on workload (within min/max vCore range) |
| **Auto-Pause** | Pauses after configurable inactivity period (min 1 hour) |
| **Resume** | Automatic on first connection (~1 minute cold start) |
| **Billing** | Per-second for compute used + storage (always billed) |
| **Min vCores** | 0.5 vCores |
| **Max vCores** | Up to 80 vCores |
| **Available Tiers** | General Purpose, Hyperscale |
| **Available For** | SQL Database (Single) only — NOT pools or MI |

### Billing Model

```
Cost = (vCores used × per-second rate) + (Storage × per-GB rate)
       ↑ Only when active                  ↑ Always charged

When paused: Only storage is billed (significant savings!)
```

### Serverless vs Provisioned

| Scenario | Serverless | Provisioned |
|----------|-----------|-------------|
| Dev/Test environments | ✅ Ideal | ❌ Overpaying |
| Intermittent, unpredictable usage | ✅ Ideal | ❌ Overpaying |
| Light usage apps (< 25% average) | ✅ Saves money | ❌ Overpaying |
| Consistent, predictable workload | ❌ May cost more | ✅ Better value |
| Cannot tolerate cold start latency | ❌ ~1 min pause | ✅ Always ready |
| 24/7 high-utilization production | ❌ More expensive | ✅ Better value |

### Auto-Pause Configuration

- **Minimum delay:** 1 hour of inactivity
- **Maximum delay:** 7 days
- **Disable auto-pause:** Set to -1 (always running but still auto-scales)
- **What triggers resume:** Any database connection attempt
- **Cold start impact:** First query after resume takes ~1 minute

> 💡 **AZ-305 Tip:** Serverless is the answer when the scenario says "intermittent usage," "dev/test," "only used during business hours," or "minimize cost for light workloads." If it says "always on" or "cannot tolerate latency," choose provisioned.

---

## 8. High Availability (Built-In)

### HA Architecture by Tier

| Tier | Architecture | Failover Time | SLA |
|------|-------------|---------------|-----|
| **General Purpose** | Remote storage + single compute + standby | ~30 sec | 99.99% (ZR) / 99.95% |
| **Business Critical** | Always On AG (local SSD, sync replicas) | ~30 sec | 99.99% (ZR) / 99.995% |
| **Hyperscale** | Distributed page servers + HA replicas | ~30 sec | 99.99% (ZR) / 99.95% |

### General Purpose HA

```
┌────────────┐         ┌──────────────────┐
│  Primary   │────────▶│  Azure Premium   │
│  Compute   │         │  Storage (Remote)│
└────────────┘         └──────────────────┘
       │ failover
       ▼
┌────────────┐
│  Standby   │ (provisioned but idle)
│  Compute   │
└────────────┘
```

### Business Critical HA

```
┌────────────┐  sync   ┌────────────┐  sync   ┌────────────┐
│  Primary   │────────▶│  Replica 1 │────────▶│  Replica 2 │
│ (Local SSD)│         │ (Local SSD)│         │ (Local SSD)│
└────────────┘         └────────────┘         └────────────┘
                              │
                              ▼
                       Read Scale-Out
                       (free, included)
```

### Zone Redundancy

| Feature | Without ZR | With ZR |
|---------|-----------|---------|
| **Replica Placement** | Same datacenter | Across 3 availability zones |
| **Protection** | Hardware/rack failure | Full zone failure |
| **SLA** | 99.95% (GP) / 99.995% (BC) | 99.99% (GP & BC) |
| **Additional Cost** | Baseline | ~small premium |
| **Available For** | All tiers | All tiers (GP, BC, Hyperscale) |

> 💡 **AZ-305 Tip:** Zone redundancy is the key to achieving **99.99% SLA** in General Purpose tier. Without it, GP is only 99.95%. Business Critical without ZR is already 99.995%.

---

## 9. Disaster Recovery

### Options Comparison

| Feature | Active Geo-Replication | Auto-Failover Groups |
|---------|----------------------|---------------------|
| **Scope** | Per database | Group of databases / Instance |
| **Failover** | Manual (app manages) | Automatic + Manual |
| **Endpoints** | Different per replica | Listener endpoints (auto-redirect) |
| **Read/Write Listener** | ❌ | ✅ `<fog-name>.database.windows.net` |
| **Read-Only Listener** | ❌ | ✅ `<fog-name>.secondary.database.windows.net` |
| **Replicas** | Up to 4 (any region) | 1 secondary region |
| **SQL MI Support** | ❌ | ✅ |
| **Grace Period** | N/A | Configurable (min 1 hour) |
| **App Changes Needed** | Yes (connection strings) | No (uses listener) |
| **Best For** | Granular per-DB control | Production DR, minimal app changes |

### RPO/RTO by Configuration

| Configuration | RPO | RTO |
|--------------|-----|-----|
| **Geo-Replication** (async) | < 5 seconds | < 30 seconds (manual) |
| **Auto-Failover Group** (async) | < 5 seconds | < 1 hour (auto) |
| **Geo-Restore** (from backup) | < 1 hour | < 12 hours |
| **LTR Restore** | Up to 1 week | Hours |

### Auto-Failover Group Architecture

```
┌─────────────────────┐                    ┌─────────────────────┐
│   PRIMARY REGION    │                    │  SECONDARY REGION   │
│                     │   Async Repl.      │                     │
│  ┌──────────────┐  │ ──────────────────▶ │  ┌──────────────┐  │
│  │  SQL Server  │  │                    │  │  SQL Server  │  │
│  │  (Primary)   │  │                    │  │  (Secondary) │  │
│  │  ┌────┐┌────┐│  │                    │  │  ┌────┐┌────┐│  │
│  │  │DB-A││DB-B││  │                    │  │  │DB-A││DB-B││  │
│  │  └────┘└────┘│  │                    │  │  └────┘└────┘│  │
│  └──────────────┘  │                    │  └──────────────┘  │
└─────────────────────┘                    └─────────────────────┘
            │                                         │
            ▼                                         ▼
┌─────────────────────────────────────────────────────────────────┐
│  Read-Write Listener: <fog>.database.windows.net                 │
│  Read-Only Listener:  <fog>.secondary.database.windows.net       │
│  (Automatically routes to current primary/secondary)             │
└─────────────────────────────────────────────────────────────────┘
```

### Key DR Decisions for AZ-305

| Requirement | Solution |
|-------------|----------|
| Automatic failover with no app changes | Auto-Failover Group |
| DR for SQL Managed Instance | Auto-Failover Group (only option) |
| Need replicas in multiple regions | Active Geo-Replication (up to 4) |
| Lowest RPO possible | Active Geo-Replication (< 5 sec) |
| Cost-effective DR (acceptable longer RTO) | Geo-Restore from backup |
| Need read workloads in secondary region | Both options support readable secondary |

> 💡 **AZ-305 Tip:** **Auto-Failover Groups** are almost always the correct answer for DR questions because they provide automatic failover AND listener endpoints that don't require application connection string changes.

---

## 10. Security

### Defense in Depth Layers

```
┌─────────────────────────────────────────────────┐
│  NETWORK SECURITY                                │
│  • Firewall Rules (IP-based)                     │
│  • Virtual Network Service Endpoints             │
│  • Private Endpoints (Private Link)              │
│  • Deny Public Network Access                    │
├─────────────────────────────────────────────────┤
│  IDENTITY & ACCESS                               │
│  • Microsoft Entra ID Authentication             │
│  • SQL Authentication                            │
│  • Managed Identity support                      │
│  • Entra-only mode (disable SQL auth)            │
├─────────────────────────────────────────────────┤
│  DATA PROTECTION                                 │
│  • TDE (Transparent Data Encryption)             │
│  • Always Encrypted (client-side encryption)     │
│  • Dynamic Data Masking                          │
│  • Row-Level Security (RLS)                      │
│  • TLS 1.2+ in transit                           │
├─────────────────────────────────────────────────┤
│  MONITORING & THREAT DETECTION                   │
│  • Microsoft Defender for SQL                    │
│  • Auditing (to Storage/Log Analytics/Event Hub) │
│  • Advanced Threat Protection                    │
│  • Vulnerability Assessment                      │
│  • SQL Ledger (tamper-evident)                   │
└─────────────────────────────────────────────────┘
```

### Security Feature Comparison

| Feature | Purpose | Protects Against | Key Detail |
|---------|---------|-----------------|------------|
| **TDE** | Encryption at rest | Physical media theft | Enabled by default, service-managed or CMK |
| **Always Encrypted** | Column encryption | DBA snooping, cloud operator access | Client-side; DB never sees plaintext; with secure enclaves for richer queries |
| **Dynamic Data Masking** | Obfuscate data display | Unauthorized viewing | Does NOT encrypt; admin users see unmasked; masks in query results |
| **Row-Level Security** | Filter rows per user | Unauthorized row access | Security predicate (filter + block) |
| **TLS** | Encryption in transit | Man-in-the-middle | Enforced by default (TLS 1.2+) |
| **Auditing** | Activity logging | Compliance violations | Azure Storage, Log Analytics, Event Hubs |
| **Microsoft Defender** | Threat detection | SQL injection, anomalous access | Alerts + vulnerability assessment |
| **Ledger** | Tamper evidence | Data tampering | Cryptographic proof of data integrity |

### Network Security Decision

| Requirement | Solution |
|-------------|----------|
| Allow specific public IPs | Server-level firewall rules |
| Allow Azure services | "Allow Azure services" toggle |
| Restrict to specific VNet/subnet | VNet service endpoints |
| Full private connectivity (no public IP) | Private Endpoint (Private Link) |
| Block all public access | Deny public network access + Private Endpoint |
| On-premises connectivity | Private Endpoint + ExpressRoute/VPN |

### Always Encrypted vs TDE

| Aspect | TDE | Always Encrypted |
|--------|-----|-----------------|
| **Encryption scope** | Entire database (at rest) | Specific columns |
| **Who holds keys** | Server/service | Client application |
| **DBA can see data?** | ✅ Yes | ❌ No |
| **Performance impact** | Minimal | Higher (client-side crypto) |
| **Query limitations** | None | Limited queries on encrypted columns (without enclaves) |
| **Use case** | Compliance checkbox | True separation of duties |
| **Secure Enclaves** | N/A | Enables range queries, sorting on encrypted data |

> 💡 **AZ-305 Tip:** If the scenario says "DBA should NOT be able to see sensitive data" or "separation of duties between data owners and DBAs" → **Always Encrypted**. If it says "encrypt at rest for compliance" → **TDE** (already enabled by default).

---

## 11. Backup & Restore

### Backup Types

| Backup Type | Frequency | Purpose |
|-------------|-----------|---------|
| **Full** | Weekly | Complete database backup |
| **Differential** | Every 12-24 hours | Changes since last full |
| **Transaction Log** | Every 5-10 minutes | Point-in-time recovery |

### Point-in-Time Restore (PITR)

| Tier | Retention | RPO |
|------|-----------|-----|
| Basic (DTU) | 1-7 days | ~5-10 min |
| Standard/Premium (DTU) | 1-35 days | ~5-10 min |
| General Purpose | 1-35 days (default 7) | ~5-10 min |
| Business Critical | 1-35 days (default 7) | ~5-10 min |
| Hyperscale | 1-35 days (default 7) | ~5-10 min |

### Long-Term Retention (LTR)

- Store **full backups** for up to **10 years**
- Configured as policies: Weekly (W), Monthly (M), Yearly (Y)
- Example: `W=4, M=12, Y=5` = keep 4 weekly, 12 monthly, 5 yearly
- Stored in Azure Blob Storage (RA-GRS)
- Use for: compliance, regulatory, legal hold requirements

### Backup Storage Redundancy

| Option | Protection | Use Case |
|--------|-----------|----------|
| **LRS** (Local) | 3 copies in same datacenter | Dev/test, lowest cost |
| **ZRS** (Zone) | 3 copies across availability zones | Production, same region DR |
| **GRS** (Geo) | 6 copies (3 local + 3 in paired region) | Cross-region DR |
| **GZRS** (Geo-Zone) | Zone + Geo redundancy | Highest protection |

> ⚠️ **Important:** Backup storage redundancy is configured at **database creation** and can be changed later, but requires a new full backup cycle.

### Restore Options Summary

| Method | Speed | Data Loss | Use Case |
|--------|-------|-----------|----------|
| **PITR** | Minutes-hours | Since last log backup (5-10 min) | Accidental deletion/corruption |
| **Geo-Restore** | Up to 12 hours | Up to 1 hour | Region-wide outage |
| **LTR Restore** | Hours | Since backup was taken | Compliance, old data recovery |
| **Deleted DB Restore** | Minutes | Since deletion point | Accidental DB drop |
| **Copy** | Varies by size | None (snapshot point) | Testing with production data |

> 💡 **AZ-305 Tip:** **Geo-Restore** uses geo-replicated backup storage — it's the cheapest DR option but has the longest RTO (up to 12 hours) and RPO (up to 1 hour). If the scenario needs faster recovery, use **Auto-Failover Groups**.

---

## 12. Migration

### Migration Path Decision

```
┌────────────────────────────────────────────────────┐
│              SOURCE: SQL Server On-Premises          │
└────────────────────────┬───────────────────────────┘
                         │
            ┌────────────┴────────────┐
            │  Run Assessment First!   │
            │  (DMA / Azure Migrate)   │
            └────────────┬────────────┘
                         │
         ┌───────────────┼───────────────┐
         │               │               │
    No blockers     Some issues      Many issues
    found           fixable          /blockers
         │               │               │
         ▼               ▼               ▼
   SQL Database    SQL Managed      SQL on VM
   or MI           Instance         (IaaS)
```

### Migration Tools

| Tool | Purpose | Supports |
|------|---------|----------|
| **Azure Migrate** | Discover, assess, migrate | Full assessment + migration |
| **Data Migration Assistant (DMA)** | Assess compatibility | SQL DB, MI, VM |
| **Azure Database Migration Service (DMS)** | Online/offline migration | Minimal downtime |
| **SQL Server Migration Assistant (SSMA)** | Non-SQL sources | MySQL, Oracle, Access → Azure SQL |
| **Azure Data Studio** | Migration extension | Assessment + migration |
| **BACPAC** | Export/Import | Small databases, schema + data |
| **Transactional Replication** | Online migration | Near-zero downtime |
| **Log Replay Service (LRS)** | Backup restore to MI | Managed Instance only |

### Migration Methods Comparison

| Method | Downtime | Best For |
|--------|----------|----------|
| **DMS Online** | Minutes | Production workloads |
| **DMS Offline** | Hours | Simple migrations |
| **BACPAC Import** | Hours-Days | Small DBs (<200 GB) |
| **Transactional Replication** | Seconds | Zero-downtime requirement |
| **Log Replay Service** | Minutes | MI with native backup files |
| **Backup/Restore** | Hours | SQL on VM |
| **Bulk Copy (BCP)** | Varies | Data-only migration |

### Compatibility Levels

- SQL Database supports compatibility levels 100-160
- Higher levels enable newer query optimizer features
- Doesn't change T-SQL surface area (that's the version)
- Recommendation: Test at higher level, fall back if issues

> 💡 **AZ-305 Tip:** For "minimal downtime migration," the answer is **Azure Database Migration Service (Online mode)** or **Transactional Replication**. For Managed Instance specifically, **Log Replay Service** is also an option.

---

## 13. Scaling

### Vertical Scaling (Scale Up/Down)

| Aspect | SQL Database | Managed Instance | SQL on VM |
|--------|-------------|-----------------|-----------|
| **Method** | Change tier/vCores/DTU | Change vCores/storage | Resize VM |
| **Downtime** | Brief (~seconds for online) | Minutes-hours | VM restart required |
| **Online** | ✅ Most changes | ✅ (but can take time) | ❌ |
| **Storage Scale** | Up only (cannot shrink) | Up only (cannot shrink) | Both |
| **Automation** | Azure Automation, Logic Apps, Alerts | Azure Automation | Azure Automation |

### Horizontal Scaling (Scale Out)

| Method | Type | Tier Required | Use Case |
|--------|------|---------------|----------|
| **Read Scale-Out** | Built-in read replica | BC / Premium / Hyperscale | Offload read queries |
| **Named Replicas** | Independent read replicas | Hyperscale only | Per-tenant, analytics isolation |
| **Geo-Replication** | Cross-region replicas | All tiers | Global read distribution |
| **Elastic Database Tools** | Application-level sharding | Any | Massive scale, custom sharding |
| **Azure SQL Database Elastic Pool** | Resource sharing | GP, BC | Multi-database cost optimization |

### Sharding with Elastic Database Tools

| Component | Purpose |
|-----------|---------|
| **Shard Map Manager** | Tracks shard-to-data mapping |
| **Shard Map** | List map or range map |
| **Shardlets** | Individual data partitions |
| **Split-Merge Service** | Move data between shards |
| **Elastic Query** | Query across shards |
| **Elastic Jobs** | Run T-SQL across all shards |

> 💡 **AZ-305 Tip:** Sharding is rarely the answer on AZ-305 unless the scenario explicitly mentions **millions of tenants** or **extreme write scale**. Usually, **Hyperscale** or **Elastic Pools** are the simpler, preferred answer.

---

## 14. AZ-305 Decision Scenarios

### Scenario 1: SaaS Multi-Tenant Application

> **Situation:** A company builds a SaaS application with 200 small tenants. Each tenant has their own database (avg 5 GB). Usage is unpredictable — some tenants spike during business hours, others are barely used. The company wants to minimize costs while maintaining isolation.

**Solution:** **Elastic Pool (vCore, General Purpose)**
- Database-per-tenant provides isolation
- Shared vCore pool handles variable peaks efficiently
- Set per-database min=0, max=4 vCores
- Estimated: 200 DBs × 5 GB = 1 TB total storage
- Much cheaper than 200 individual S2 databases

**Why not alternatives?**
- Individual DBs: 200 × S2 cost is prohibitive
- Managed Instance: No resource sharing between DBs
- Hyperscale named replicas: For read scenarios, not cost sharing

---

### Scenario 2: Mission-Critical OLTP with Zero Tolerance

> **Situation:** A financial trading platform requires <2ms read latency, 99.995% availability, automatic failover within the same region, and the ability to offload reporting queries.

**Solution:** **SQL Database, Business Critical tier with Zone Redundancy**
- Local SSD storage provides <2ms latency
- Built-in Always On AG with synchronous replicas
- 99.995% SLA (99.99% with ZR)
- Free read-scale-out replica for reporting
- Zone redundancy for zone-level failure protection

**For cross-region DR:** Add Auto-Failover Group to secondary region.

---

### Scenario 3: Large Data Warehouse Migration

> **Situation:** A company has a 20 TB SQL Server database on-premises used for analytics. They need to migrate to Azure with minimal downtime. The database is growing 2 TB/year and they need fast backups.

**Solution:** **SQL Database, Hyperscale tier**
- Supports up to 128 TB (handles growth)
- Near-instant backups regardless of size
- Fast PITR restore (minutes vs. hours)
- Named replicas for analytics workloads
- Migrate using DMS Online mode for minimal downtime

**Why not alternatives?**
- General Purpose: Max 4 TB (too small)
- Business Critical: Max 4 TB (too small)
- SQL on VM: Would work but more management overhead
- Synapse: Better for true data warehouse workloads (star schema, complex analytics)

---

### Scenario 4: Lift-and-Shift with SQL Agent Jobs

> **Situation:** A company has 15 SQL Server databases using SQL Agent jobs, CLR assemblies, cross-database queries, Service Broker, and Database Mail. They want to move to Azure with minimal code changes and reduce management overhead.

**Solution:** **SQL Managed Instance (General Purpose, vCore)**
- Supports SQL Agent, CLR, cross-DB queries, Service Broker, DB Mail
- Near 100% compatibility with on-premises SQL Server
- PaaS — no OS/patching management
- Use Azure Hybrid Benefit with existing licenses
- Migrate using DMS Online or Log Replay Service

**Why not alternatives?**
- SQL Database: Missing SQL Agent, CLR, cross-DB queries
- SQL on VM: Works but more management (OS patching, HA setup)

---

### Scenario 5: Dev/Test with Cost Optimization

> **Situation:** A development team needs SQL databases for testing. Databases are used Monday-Friday, 9 AM - 6 PM. On weekends and nights, no activity. They want to minimize costs.

**Solution:** **SQL Database, General Purpose, Serverless**
- Auto-pause after 1 hour of inactivity (saves compute costs)
- Only pay for storage on nights/weekends
- Auto-scale during business hours based on demand
- Per-second billing for actual usage

**Additional savings:**
- Use Dev/Test pricing (if available via subscription)
- Consider Azure Hybrid Benefit if applicable
- Set min vCores = 0.5 for minimal baseline

---

### Scenario 6: Global Application with Low-Latency Reads

> **Situation:** A global e-commerce company has users in North America, Europe, and Asia. The primary database is in East US. They need low-latency reads in all regions while writes go to the primary. Budget is a concern.

**Solution:** **SQL Database (Hyperscale) with Geo-Replication** OR **Auto-Failover Groups**
- Primary in East US (all writes)
- Geo-replicas in West Europe and Southeast Asia (reads)
- Application routes reads to nearest replica
- Auto-Failover Group provides automatic DR

**Alternative approach:**
- Active Geo-Replication for read replicas in 3+ regions
- Application-level read routing to nearest replica
- RPO < 5 seconds for each replica

**Why not alternatives?**
- Azure Cosmos DB: If they need multi-region writes
- Azure Front Door + CDN: Only helps for static content

---

### Scenario 7: Regulatory Compliance - Data Sovereignty

> **Situation:** A healthcare company must keep patient data encrypted at rest AND in transit. DBAs should not be able to view patient PII. Data must remain in specific regions. Audit trails required.

**Solution:**
- **SQL Database, Business Critical** (for HA)
- **Always Encrypted** with secure enclaves (DBA can't see PII)
- **TDE with Customer-Managed Keys** (control encryption keys in Azure Key Vault)
- **Row-Level Security** (restrict access per user/role)
- **Auditing** enabled → Log Analytics
- **Microsoft Defender for SQL** (threat detection)
- **Private Endpoint** (no public internet exposure)
- **Deny public network access** = true
- **Geo-restriction** — deploy only in compliant regions

---

### Scenario 8: Database Consolidation

> **Situation:** A company acquired 3 subsidiaries, each with their own SQL Server instances (total 45 databases). They want to consolidate into Azure, reduce licensing costs (they have Software Assurance), and simplify management. Some databases use linked servers between instances.

**Solution:** **SQL Managed Instance** (vCore, General Purpose)
- Consolidate all 45 DBs into 1-2 Managed Instances
- Linked server support for cross-instance queries
- **Azure Hybrid Benefit** — use existing SA licenses (save ~55%)
- **Reserved capacity** — commit 3-year for additional savings
- Instance-level management vs 45 individual databases
- Migrate using DMS Online for minimal downtime

---

## 15. Quick Reference Trigger Table

| If the scenario says... | Think... |
|------------------------|----------|
| "Lift and shift" / "minimal code changes" | **SQL Managed Instance** |
| "SQL Agent jobs" / "CLR" / "cross-database queries" | **SQL Managed Instance** |
| "FILESTREAM" / "SSIS" / "full OS control" | **SQL Server on VM** |
| "Existing SQL Server licenses" / "Software Assurance" | **vCore + Azure Hybrid Benefit** |
| "Variable workloads" / "multiple databases" / "SaaS" | **Elastic Pool** |
| "Intermittent" / "dev/test" / "only used sometimes" | **Serverless** |
| "Growing beyond 4 TB" / "fast backups" / "100+ TB" | **Hyperscale** |
| "Sub-millisecond latency" / "mission critical" / "OLTP" | **Business Critical** |
| "Cost-effective" / "budget-friendly" / "most workloads" | **General Purpose** |
| "Zero downtime migration" | **DMS Online** / **Transactional Replication** |
| "No connection string changes for failover" | **Auto-Failover Group** |
| "Multiple geo-read replicas" (>1 secondary region) | **Active Geo-Replication** |
| "Managed Instance DR" | **Auto-Failover Group** (only option for MI) |
| "DBA must not see data" / "separation of duties" | **Always Encrypted** |
| "Encrypt at rest" / "compliance checkbox" | **TDE** (default, already on) |
| "Hide/obfuscate data from non-privileged users" | **Dynamic Data Masking** |
| "No public internet access" | **Private Endpoint + Deny Public** |
| "Per-tenant read isolation" / "independent compute" | **Hyperscale Named Replicas** |
| "Automatic failover" + "listener endpoints" | **Auto-Failover Group** |
| "99.99% SLA" + "General Purpose" | **Enable Zone Redundancy** |
| "Read-only routing" / "offload reporting" | **Read Scale-Out** (BC) or **Named Replicas** (HS) |
| "Regulatory" / "10 year retention" | **Long-Term Retention (LTR)** |
| "Cheapest DR option" | **Geo-Restore** (from geo-redundant backups) |
| "Near-zero RPO + automatic failover" | **Auto-Failover Group** (<5 sec RPO) |
| "Multi-master" / "multi-region writes" | **NOT Azure SQL** → Cosmos DB |
| "Data tampering protection" / "immutable audit" | **SQL Ledger** |
| "Consolidate many small DBs cheaply" | **Elastic Pool** |
| "Independent scaling per read workload" | **Hyperscale Named Replicas** |

---

## 16. Pricing Considerations

### Cost Optimization Strategies

| Strategy | Savings | Applies To |
|----------|---------|-----------|
| **Azure Hybrid Benefit** | Up to 55% | vCore models (existing SQL licenses + SA) |
| **Reserved Capacity** | Up to 33% | vCore (1-year or 3-year commitment) |
| **AHB + Reserved** | Up to 80% combined | vCore models |
| **Serverless** | Variable | Intermittent workloads (<25% avg utilization) |
| **Elastic Pool** | 40-70% | Multiple DBs with variable peaks |
| **Dev/Test Pricing** | Up to 55% | Non-production (VS Enterprise subscription) |
| **Free Offer** | 100% (limited) | 100K vCore-seconds/month + 32 GB storage |
| **Right-sizing** | 20-40% | Over-provisioned databases |

### Pricing Components

```
Total Cost = Compute + Storage + Backup Storage + Networking + Add-ons

Compute:
  ├─ Provisioned: Fixed monthly rate per vCore/DTU
  ├─ Serverless: Per-second usage + auto-pause savings
  └─ Reserved: Discounted fixed rate (1yr/3yr)

Storage:
  ├─ Data storage (per GB/month)
  ├─ Log storage
  └─ tempdb (free, local SSD for BC)

Backup:
  ├─ PITR: Free up to 100% of DB size (then per GB)
  └─ LTR: Per GB/month in blob storage

Networking:
  ├─ Egress charges (cross-region replication)
  └─ Private Link: Per endpoint + data processed

Add-ons:
  ├─ Microsoft Defender for SQL
  ├─ Auditing storage
  └─ Additional read replicas
```

### Cost Comparison Example (Rough Estimates)

| Configuration | ~Monthly Cost |
|--------------|--------------|
| GP Serverless, 2-4 vCores, 50 GB (light usage) | $50-150 |
| GP Provisioned, 4 vCores, 100 GB | $400-500 |
| GP Provisioned, 4 vCores, 100 GB + AHB | $200-250 |
| BC Provisioned, 4 vCores, 100 GB | $1,200-1,500 |
| Hyperscale, 4 vCores, 500 GB, 1 replica | $800-1,000 |
| Elastic Pool GP, 8 vCores (20 DBs) | $700-900 |
| Managed Instance GP, 8 vCores | $1,200-1,500 |

> ⚠️ Prices are approximate and vary by region. Always use the [Azure Pricing Calculator](https://azure.microsoft.com/pricing/calculator/) for accurate estimates.

### Key Pricing Rules for AZ-305

1. **Storage cannot be reduced** — you can scale up but not down
2. **Backup storage** — first allocation (= DB size) is free; extra is charged
3. **Geo-replication** — secondary is billed as a separate database at same tier
4. **Failover Groups** — secondary is billed (it's an active replica)
5. **Zone Redundancy** — adds ~25% premium to compute cost
6. **Hyperscale named replicas** — each billed independently
7. **Serverless min cost** — storage is ALWAYS billed, even when paused
8. **License costs** — often 50%+ of total; AHB dramatically reduces this

---

## 🎯 Final AZ-305 Exam Tips

1. **Always consider the simplest PaaS option first** — only go to IaaS (VM) when PaaS truly can't work
2. **Managed Instance is the "lift and shift" answer** — unless they need FILESTREAM/SSIS/OS access
3. **Auto-Failover Groups > Active Geo-Replication** for most DR scenarios (listener endpoints = no app changes)
4. **Hyperscale is the answer for large/growing databases** — don't be tricked by "just use a bigger tier"
5. **Zone Redundancy is cheap insurance** — enables 99.99% SLA even on General Purpose
6. **Always Encrypted ≠ TDE** — understand the DBA separation of duties angle
7. **Serverless ≠ Consumption plan** — Serverless SQL DB auto-pauses; Azure Functions Consumption is event-driven
8. **Elastic Pool doesn't support Hyperscale directly** — use Named Replicas for multi-tenant read scale
9. **vCore is almost always the right purchasing model** — AHB + Reserved = massive savings
10. **Read the scenario carefully for "hidden" requirements** — CLR, SQL Agent, linked servers → MI; FILESTREAM → VM

---

*Last Updated: 2025 | Target: AZ-305 Designing Microsoft Azure Infrastructure Solutions*