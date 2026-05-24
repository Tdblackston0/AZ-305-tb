# Azure Cosmos DB - AZ-305 Cheat Sheet

> 📝 **Hands-On Labs:** [Azure Cosmos DB Labs](../Labs/Azure-CosmosDB-Labs.md)

> 🎯 **Exam Weight:** Cosmos DB is heavily tested in AZ-305. Expect 3-5 questions covering consistency levels, partitioning strategy, throughput models, and global distribution decisions.

## Table of Contents

- [1. Cosmos DB Overview](#cosmos-db-overview)
- [2. APIs (Data Models)](#apis-data-models)
- [3. Consistency Levels](#consistency-levels)
- [4. Partitioning](#partitioning)
- [5. Request Units (RU/s)](#request-units-rus)
- [6. Throughput Models](#throughput-models)
- [7. Global Distribution](#global-distribution)
- [8. Indexing](#indexing)
- [9. Change Feed](#change-feed)
- [10. Security](#security)
- [11. Backup & Restore](#backup-restore)
- [12. Cost Optimization](#cost-optimization)
- [13. Cosmos DB vs Other Services](#cosmos-db-vs-other-services)
- [14. AZ-305 Decision Scenarios](#az-305-decision-scenarios)
- [15. Quick Reference Trigger Table](#quick-reference-trigger-table)
- [16. Synapse Link (HTAP)](#synapse-link-htap)
- [17. AZ-305 Exam Tips Summary](#az-305-exam-tips-summary)
- [18. Architecture Decision Flowchart](#architecture-decision-flowchart)

---

<a id="cosmos-db-overview"></a>
## 1. Cosmos DB Overview

Azure Cosmos DB is a **globally distributed, multi-model, NoSQL database** service with guaranteed single-digit millisecond latency at the 99th percentile.

### Key Characteristics

| Feature | Detail |
|---------|--------|
| **Distribution** | Turnkey global distribution across 60+ Azure regions |
| **Availability SLA** | 99.999% (multi-region) / 99.99% (single-region) |
| **Latency SLA** | <10ms reads, <10ms writes at P99 |
| **Throughput SLA** | Guaranteed provisioned throughput |
| **Consistency SLA** | Guaranteed consistency per chosen level |
| **Schema** | Schema-agnostic, automatically indexes all data |
| **Multi-model** | Document, key-value, graph, column-family, table |

### When to Choose Cosmos DB

- ✅ Global distribution required
- ✅ Single-digit millisecond latency at scale
- ✅ Multi-region writes needed
- ✅ Elastic horizontal scale (unlimited throughput/storage)
- ✅ Multiple consistency models needed
- ✅ Guaranteed SLAs for mission-critical workloads

> 💡 **AZ-305 Tip:** If the question mentions "globally distributed," "multi-region," "single-digit millisecond latency," or "five 9s availability" — think Cosmos DB immediately.

---

<a id="apis-data-models"></a>
## 2. APIs (Data Models)

Cosmos DB offers **six API options** chosen at account creation (cannot be changed later).

| API | Data Model | Wire Protocol | Best For |
|-----|-----------|---------------|----------|
| **NoSQL (Core)** | Document (JSON) | REST/SQL-like query | New apps, most flexible, full feature access |
| **MongoDB** | Document (BSON) | MongoDB wire protocol | Existing MongoDB apps, lift-and-shift |
| **Cassandra** | Wide-column | CQL (Cassandra Query Language) | Existing Cassandra workloads |
| **Gremlin** | Graph | Apache TinkerPop/Gremlin | Social networks, recommendation engines, fraud detection |
| **Table** | Key-value | Azure Table Storage protocol | Migrating from Azure Table Storage (better perf/SLAs) |
| **PostgreSQL** | Relational + Document | PostgreSQL (via Citus) | Distributed PostgreSQL, multi-tenant SaaS |

### API Selection Decision Tree

```
Is it a new application with no existing database code?
├── YES → Use API for NoSQL (most features, best SDK support)
├── Existing MongoDB code? → MongoDB API
├── Existing Cassandra code? → Cassandra API
├── Need graph traversals? → Gremlin API
├── Migrating from Table Storage? → Table API
└── Need distributed PostgreSQL? → PostgreSQL API
```

> ⚠️ **AZ-305 Tip:** The API for NoSQL is the default recommendation for new applications. It gets features first and has the richest SDK support. Only choose other APIs for migration/compatibility reasons.

---

<a id="consistency-levels"></a>
## 3. Consistency Levels

Cosmos DB offers **five consistency levels** — a spectrum from strongest to weakest. This is the **most tested Cosmos DB topic on AZ-305**.

### The Five Levels (Strongest → Weakest)

| Level | Guarantee | Analogy | RU Cost | Availability |
|-------|-----------|---------|---------|--------------|
| **Strong** | Linearizability — reads always return most recent committed write | Bank balance: always see latest | 2x | Lowest (single-region writes only) |
| **Bounded Staleness** | Reads lag behind writes by at most K versions or T time | News feed: guaranteed within 5 min | ~2x | High (single-region writes only for <2 regions) |
| **Session** | Within a session: read-your-writes, monotonic reads | Shopping cart: you see your own changes | 1x | High |
| **Consistent Prefix** | Reads never see out-of-order writes | Reading a book in order (but maybe behind) | 1x | Higher |
| **Eventual** | No ordering guarantee; replicas converge eventually | Social media likes: count eventually correct | 1x | Highest |

### Detailed Comparison

```
                CONSISTENCY SPECTRUM
    ←———————————————————————————————————————→
    Strong    Bounded    Session    Prefix    Eventual
    
    More Consistent ←——————————→ More Available
    Higher Latency  ←——————————→ Lower Latency
    Higher RU Cost  ←——————————→ Lower RU Cost
    Lower Throughput←——————————→ Higher Throughput
```

### Real-World Analogies

| Level | Real-World Analogy |
|-------|-------------------|
| **Strong** | ATM withdrawal — everyone sees the updated balance immediately |
| **Bounded Staleness** | Stock ticker — may be up to 15 seconds behind, but never shows older data after newer |
| **Session** | Your email inbox — you see emails you sent, others might not yet |
| **Consistent Prefix** | Watching a live stream with a 30-sec delay — you see everything in order, just delayed |
| **Eventual** | DNS propagation — eventually all servers agree, but temporarily different |

### Key Rules for AZ-305

| Scenario | Choose |
|----------|--------|
| Financial transactions, inventory counts | **Strong** |
| Leaderboard that can be a few seconds behind | **Bounded Staleness** |
| User's own profile edits visible immediately to them | **Session** (default) |
| Activity feed — order matters, slight delay OK | **Consistent Prefix** |
| Like counts, view counters | **Eventual** |

> 🔑 **Critical AZ-305 Facts:**
> - **Session** is the DEFAULT and most commonly used level (~70% of workloads)
> - **Strong** consistency is NOT available with multi-region writes
> - **Bounded Staleness** with multi-region is recommended for apps needing strong-like consistency globally
> - Strong and Bounded Staleness consume **2x RUs** for reads (reads from 2 replicas)
> - Consistency is configurable at the **account level** (default) and can be **relaxed per request** (never strengthened)

---

<a id="partitioning"></a>
## 4. Partitioning

### Architecture

```
Cosmos DB Account
└── Database
    └── Container (Collection/Table/Graph)
        ├── Logical Partition 1 (partition key = "Seattle")
        │   ├── Item 1
        │   └── Item 2
        ├── Logical Partition 2 (partition key = "Portland")
        │   └── Item 3
        └── Physical Partition A (hosts multiple logical partitions)
            └── Max 50 GB storage per physical partition
```

### Key Concepts

| Concept | Detail |
|---------|--------|
| **Logical Partition** | All items sharing the same partition key value; max 20 GB |
| **Physical Partition** | Infrastructure unit; max 50 GB storage, 10,000 RU/s throughput |
| **Partition Key** | Immutable property chosen at container creation |
| **Cross-partition query** | Fan-out query across multiple partitions (expensive) |

### Choosing a Partition Key — The Golden Rules

| ✅ DO | ❌ DON'T |
|--------|----------|
| High cardinality (many distinct values) | Low cardinality (e.g., status: "active/inactive") |
| Even distribution of storage | Values that create hot partitions |
| Even distribution of throughput | Time-based keys (e.g., timestamp) |
| Frequently used in WHERE clauses | Keys rarely used in queries |
| Use in point reads (key + id) | Keys that change over time |

### Partition Key Examples

| Scenario | Good Key | Bad Key | Why |
|----------|----------|---------|-----|
| Multi-tenant SaaS | `tenantId` | `createdDate` | Even distribution per tenant |
| IoT telemetry | `deviceId` | `timestamp` | Avoids hot partition on latest time |
| E-commerce orders | `customerId` | `orderStatus` | High cardinality, natural query filter |
| Social media posts | `userId` | `category` | Distributes evenly |

### Hierarchical Partition Keys (Preview → GA)

Allows up to 3 levels: e.g., `TenantId` → `UserId` → `SessionId`
- Enables sub-partitioning for more granular distribution
- Useful for multi-tenant scenarios with large tenants

### Hot Partition Detection & Mitigation

```
Symptoms:
- 429 (rate limiting) on some requests while overall RU/s not maxed
- Uneven RU consumption across partitions
- One partition at 10,000 RU/s limit

Solutions:
1. Choose a higher-cardinality partition key
2. Use synthetic partition key (concatenate fields)
3. Use hierarchical partition keys
4. Add randomization suffix (write-heavy, sacrifice point reads)
```

> 💡 **AZ-305 Tip:** If the question describes "hot partitions" or "throttling on specific partition key values," the answer is almost always to change the partition key strategy.

---

<a id="request-units-rus"></a>
## 5. Request Units (RU/s)

### What is a Request Unit?

A **Request Unit (RU)** is a normalized measure combining CPU, IOPS, and memory for a database operation.

| Operation | Approximate Cost |
|-----------|-----------------|
| Point read (1 KB item by ID + partition key) | **1 RU** |
| Write (1 KB item) | **~5 RUs** |
| Query (returning 1 KB) | **~3 RUs** (varies with complexity) |
| Replace (1 KB item) | **~10 RUs** |
| Delete (1 KB item) | **~5 RUs** |

### Factors That Increase RU Cost

| Factor | Impact |
|--------|--------|
| Item size | Larger items = more RUs |
| Indexing | More indexed properties = more write RUs |
| Consistency level | Strong/Bounded = 2x read RUs |
| Query complexity | Cross-partition, sorts, aggregates = more RUs |
| Number of results | More items returned = more RUs |
| UDFs/Stored procedures | Complex logic = more RUs |

### How to Estimate RU/s

1. **Capacity Calculator** — [cosmos.azure.com/capacitycalculator](https://cosmos.azure.com/capacitycalculator)
2. **Response headers** — Every response includes `x-ms-request-charge`
3. **Azure Monitor** — Track actual RU consumption
4. **Rule of thumb:** Start with 400 RU/s, monitor, adjust

> 💡 **AZ-305 Tip:** The minimum provisioned throughput is **400 RU/s** per container (or 100 RU/s with database-level shared throughput with 25 containers).

---

<a id="throughput-models"></a>
## 6. Throughput Models

### Three Throughput Models

| Model | Billing | Best For | Min/Max |
|-------|---------|----------|---------|
| **Provisioned** | Per hour (set RU/s) | Predictable, sustained workloads | 400–unlimited RU/s |
| **Autoscale** | Per hour (0.1x–1x of max) | Variable workloads with peaks | 100–unlimited RU/s (max) |
| **Serverless** | Per request (RUs consumed) | Dev/test, low/sporadic traffic | 5,000 RU/s burst max |

### Provisioned Throughput

```
Fixed RU/s allocated → pay whether used or not
- Database-level: Shared across containers (min 100 RU/s per container)
- Container-level: Dedicated to one container
- Can scale up/down manually or programmatically
```

### Autoscale Throughput

```
Set a MAX RU/s → system scales between 10% and 100% of max
- Example: Max = 10,000 RU/s → scales between 1,000–10,000
- Billed at highest RU/s reached per hour
- No throttling within the max range
- Best for: unpredictable traffic, variable loads
```

### Serverless

```
No provisioning → pay only for RUs consumed per request
- Max 5,000 RU/s burst capacity
- Max 1 TB storage per container (50 GB per partition)
- Single-region only (no geo-replication)
- Best for: development, testing, light/sporadic workloads
```

### Decision Matrix

| Scenario | Model |
|----------|-------|
| Predictable traffic, production, cost-sensitive | **Provisioned** |
| Traffic with peaks 2-10x baseline | **Autoscale** |
| Dev/test, infrequent traffic, new apps | **Serverless** |
| Need multi-region | **Provisioned or Autoscale** (not Serverless) |
| Traffic is consistently >50% of max | **Provisioned** (cheaper than Autoscale) |

### Database vs Container Level Throughput

| Scope | Use Case | Consideration |
|-------|----------|---------------|
| **Database-level** | Multiple containers with shared workload | Min 100 RU/s per container; up to 25 containers share |
| **Container-level** | Predictable per-container load, isolation needed | Guaranteed throughput; containers with dedicated 400+ RU/s |
| **Mixed** | Some containers need guaranteed, others can share | Combine both in same database |

> ⚠️ **AZ-305 Tip:** Autoscale is **1.5x more expensive** per RU/s than manual provisioned. Choose provisioned if load is predictable and consistently above 66% of peak.

---

<a id="global-distribution"></a>
## 7. Global Distribution

### Multi-Region Configuration

```
                    ┌──────────────┐
                    │  East US     │ ← Write Region (Primary)
                    │  (Read+Write)│
                    └──────┬───────┘
                           │ Automatic Replication
              ┌────────────┼────────────┐
              ▼            ▼            ▼
     ┌────────────┐ ┌────────────┐ ┌────────────┐
     │ West Europe│ │ Southeast  │ │ Australia  │
     │ (Read)     │ │ Asia (Read)│ │ East (Read)│
     └────────────┘ └────────────┘ └────────────┘
```

### Multi-Region Writes

| Feature | Single-Region Write | Multi-Region Write |
|---------|--------------------|--------------------|
| Write latency | Single region latency | Local region latency |
| Availability | 99.99% | 99.999% |
| Conflict resolution | N/A | Required |
| Strong consistency | ✅ Available | ❌ Not available |
| Cost | Base price | ~25% more (additional write region cost) |

### Automatic Failover

- **Automatic failover:** System promotes read region to write region during outage
- **Manual failover:** Admin-triggered for planned maintenance
- **Service-managed failover:** Azure detects outage and promotes automatically
- **Failover priority:** Configurable priority list for region promotion

### Conflict Resolution Policies (Multi-Region Writes)

| Policy | Behavior | Use Case |
|--------|----------|----------|
| **Last Writer Wins (LWW)** | Highest timestamp wins (default) | Most scenarios, simple |
| **Custom (Stored Procedure)** | Custom merge logic in SP | Complex merge requirements |
| **Custom (Async)** | Conflicts written to conflict feed for app resolution | Human review needed |

> 🔑 **AZ-305 Tip:** Multi-region writes require a conflict resolution policy. Default is LWW using `_ts` (timestamp). Strong consistency is NOT supported with multi-region writes.

---

<a id="indexing"></a>
## 8. Indexing

### Automatic Indexing (Default)

By default, Cosmos DB **automatically indexes every property** in every item. This provides maximum query flexibility with no schema management.

### Custom Indexing Policy

```json
{
  "indexingMode": "consistent",
  "automatic": true,
  "includedPaths": [
    { "path": "/name/?" },
    { "path": "/age/?" }
  ],
  "excludedPaths": [
    { "path": "/description/?" },
    { "path": "/*" }
  ],
  "compositeIndexes": [
    [
      { "path": "/city", "order": "ascending" },
      { "path": "/age", "order": "descending" }
    ]
  ],
  "spatialIndexes": [
    { "path": "/location/*", "types": ["Point", "Polygon"] }
  ]
}
```

### Index Types

| Index Type | Purpose | When to Use |
|-----------|---------|-------------|
| **Range** | Equality, range, ORDER BY on single property | Default for most properties |
| **Composite** | ORDER BY on multiple properties, multi-property filters | Sorting by city ASC, age DESC |
| **Spatial** | Geospatial queries (ST_DISTANCE, ST_WITHIN) | Location-based queries |
| **Vector** | Vector similarity search | AI/ML embeddings |

### Indexing Optimization Tips

| Strategy | Benefit |
|----------|---------|
| Exclude write-heavy, rarely-queried paths | Reduce write RU cost |
| Add composite indexes for multi-sort queries | Avoid expensive sorts |
| Use `indexingMode: none` for pure key-value access | Minimize write cost |
| Use `indexingMode: lazy` | ❌ Deprecated — don't use |

> 💡 **AZ-305 Tip:** Indexing policy changes don't cause downtime. Reducing indexed paths lowers write RU costs but may increase read RU costs for queries on excluded paths.

---

<a id="change-feed"></a>
## 9. Change Feed

### What is Change Feed?

A **persistent, ordered log of inserts and updates** to items in a container. Deletes are NOT captured by default (use soft-delete pattern or TTL with change feed).

### Architecture

```
┌──────────────┐     ┌──────────────┐     ┌──────────────────────┐
│ Cosmos DB    │────▶│ Change Feed  │────▶│ Azure Functions      │
│ Container    │     │ (ordered log)│     │ (trigger)            │
└──────────────┘     └──────────────┘     ├──────────────────────┤
                                          │ Event Hubs           │
                                          ├──────────────────────┤
                                          │ Materialized View    │
                                          ├──────────────────────┤
                                          │ Real-time Analytics  │
                                          └──────────────────────┘
```

### Change Feed Processing Options

| Option | Complexity | Use Case |
|--------|-----------|----------|
| **Azure Functions trigger** | Low | Simple event processing |
| **Change Feed Processor (SDK)** | Medium | Custom processing, state management |
| **Change Feed pull model** | Medium | Batch processing, on-demand reads |
| **Kafka Connect (preview)** | High | Integration with Kafka ecosystems |

### Use Cases

- 📊 **Materialized views** — Denormalized read-optimized views
- 🔔 **Event-driven architectures** — Trigger downstream actions
- 📈 **Real-time analytics** — Stream processing pipelines
- 🗄️ **Data replication** — Sync to other stores or regions
- 🔍 **Full-text search sync** — Keep Azure Cognitive Search updated
- 📋 **Audit logs** — Track all changes

### Change Feed Modes

| Mode | Behavior |
|------|----------|
| **Latest version** (default) | Only the latest version of each change |
| **All versions and deletes** | Full history including intermediate changes and deletes |

> 💡 **AZ-305 Tip:** If the question asks about "event-driven," "materialized views," or "real-time sync," Change Feed with Azure Functions is usually the answer.

---

<a id="security"></a>
## 10. Security

### Authentication & Authorization

| Method | Use Case |
|--------|----------|
| **Primary/Secondary keys** | Full access (admin); rotate regularly |
| **Resource tokens** | Scoped, time-limited access to specific resources |
| **Microsoft Entra ID (RBAC)** | Identity-based access; preferred for production |
| **Managed Identity** | App-to-Cosmos without storing credentials |

### RBAC Built-in Roles

| Role | Permissions |
|------|------------|
| Cosmos DB Built-in Data Reader | Read-only data plane access |
| Cosmos DB Built-in Data Contributor | Read/write data plane access |
| Cosmos DB Account Reader | Read account metadata (control plane) |
| Cosmos DB Operator | Manage account (no data access) |

### Network Security

| Feature | Description |
|---------|-------------|
| **VNet Service Endpoints** | Restrict access to specific VNets/subnets |
| **Private Endpoints (Private Link)** | Private IP in your VNet; no public internet exposure |
| **IP Firewall** | Allow-list specific IP addresses/ranges |
| **VNet integration** | Accessible only from configured virtual networks |

### Encryption

| Layer | Detail |
|-------|--------|
| **At rest** | Encrypted by default (Microsoft-managed keys) |
| **CMK (Customer-Managed Keys)** | Bring your own key via Azure Key Vault |
| **In transit** | TLS 1.2 enforced |
| **Client-side encryption** | Application-level encryption (Always Encrypted preview) |

> 🔑 **AZ-305 Tip:** For zero-trust / defense-in-depth questions: Private Endpoint + Entra ID RBAC + CMK + disabled public access = maximum security posture.

---

<a id="backup-restore"></a>
## 11. Backup & Restore

### Backup Modes

| Feature | Continuous Backup | Periodic Backup |
|---------|------------------|-----------------|
| **Type** | Point-in-Time Restore (PITR) | Snapshots |
| **RPO** | Any point in last 7 or 30 days | Backup interval (min 1 hour) |
| **RTO** | Hours (depends on data size) | Submit support request |
| **Granularity** | Container or database level | Account level |
| **Self-service restore** | ✅ Yes (portal/CLI) | ❌ No (support ticket) |
| **Cost** | Included (7-day) / Extra (30-day) | Included (2 copies) |
| **Tiers** | Tier 1: 7 days / Tier 2: 30 days | N/A |

### Continuous Backup (PITR) Details

- Restore to any point within retention window
- Restores to a **new account** (not in-place)
- Available for NoSQL, MongoDB, Gremlin, Table APIs
- Can restore specific containers or entire database

### Periodic Backup Details

- Default: every 4 hours, retain 2 copies
- Configurable interval (1–24 hours) and retention (1–720 hours)
- Stored in geo-redundant blob storage
- Restore requires Azure support ticket

> 💡 **AZ-305 Tip:** If the question mentions "point-in-time restore" or "self-service recovery," the answer is Continuous Backup mode.

---

<a id="cost-optimization"></a>
## 12. Cost Optimization

### Cost Reduction Strategies

| Strategy | Savings | How |
|----------|---------|-----|
| **Reserved Capacity** | Up to 65% | 1-year or 3-year commitment on RU/s |
| **Right-size RU/s** | Variable | Monitor actual usage, reduce over-provisioned |
| **Autoscale** | Up to 60% vs peak provisioned | Scale down during off-peak |
| **Serverless** | Significant for <1M RU/month | Pay only for consumed RUs |
| **TTL (Time to Live)** | Storage savings | Automatically expire old data |
| **Custom indexing** | 10-30% write savings | Exclude unnecessary paths |
| **Analytical Store** | Avoid heavy OLAP on transactional store | Column-store for analytics (Synapse Link) |
| **Multi-region: fewer regions** | ~$X per region | Only add regions you need |

### Reserved Capacity

| Term | Discount |
|------|----------|
| 1-year | ~20% savings |
| 3-year | ~30-65% savings |

> Applies to provisioned throughput (RU/s) across all APIs and regions.

### TTL (Time to Live)

```
Container-level: Set default TTL for all items
Item-level: Override per item (-1 = never expire)

TTL = -1  → Never expires (even if container has TTL)
TTL = 0   → Inherits container default
TTL = N   → Expires N seconds after last modified
```

> 💡 **AZ-305 Tip:** TTL deletes don't consume RU/s from your provisioned throughput — they use leftover/"background" RUs.

---

<a id="cosmos-db-vs-other-services"></a>
## 13. Cosmos DB vs Other Services

### Decision Matrix

| Requirement | Cosmos DB | Azure SQL | Table Storage | MongoDB Atlas |
|-------------|-----------|-----------|---------------|---------------|
| Global distribution | ✅ Native | ⚠️ Geo-rep (read-only) | ❌ | ⚠️ Manual |
| Multi-region writes | ✅ | ❌ | ❌ | ⚠️ Limited |
| Guaranteed <10ms latency | ✅ SLA | ❌ | ❌ | ❌ |
| Relational/ACID | ⚠️ Limited (within partition) | ✅ Full | ❌ | ⚠️ Limited |
| Schema flexibility | ✅ | ❌ | ⚠️ | ✅ |
| Complex joins | ❌ | ✅ | ❌ | ⚠️ |
| Graph queries | ✅ (Gremlin) | ❌ | ❌ | ❌ |
| Cost at low scale | $$$ | $ | ¢ | $$ |
| Elastic scale | ✅ Unlimited | ⚠️ 100 DTU/vCore limits | ⚠️ | ⚠️ |
| 99.999% SLA | ✅ | ❌ | ❌ | ❌ |

### When NOT to Use Cosmos DB

- ❌ Complex relational queries with many JOINs → **Azure SQL**
- ❌ Simple key-value with minimal throughput → **Table Storage** (much cheaper)
- ❌ Budget-constrained, single-region, relational → **Azure SQL/PostgreSQL**
- ❌ Need full ACID across multiple entities → **Azure SQL**
- ❌ Data warehouse/OLAP → **Synapse Analytics**

### When TO Use Cosmos DB

- ✅ Mission-critical with guaranteed latency SLAs
- ✅ Multi-region active-active writes
- ✅ IoT ingestion at massive scale
- ✅ Real-time personalization/recommendations
- ✅ Gaming leaderboards (global, low-latency)
- ✅ Retail/e-commerce catalogs (flexible schema, global)

---

<a id="az-305-decision-scenarios"></a>
## 14. AZ-305 Decision Scenarios

### Scenario 1: Global E-Commerce Platform

> **Requirement:** A retail company needs a product catalog accessible from North America, Europe, and Asia with <10ms read latency and 99.999% availability.

**Answer:** Cosmos DB with API for NoSQL, multi-region reads (3 regions), Session consistency, partition key = `categoryId` or `productId`.

**Why:** Global distribution with SLA guarantees, flexible JSON schema for varying product attributes.

---

### Scenario 2: Financial Transaction Ledger

> **Requirement:** A bank needs a globally distributed ledger where all readers must see the most recent committed write. The system operates in two regions.

**Answer:** Cosmos DB with **Strong consistency**, single-region write with read replicas. If multi-region writes are required, use **Bounded Staleness** (Strong is not supported with multi-region writes).

**Why:** Strong consistency guarantees linearizability; financial data cannot be stale.

---

### Scenario 3: IoT Telemetry Ingestion

> **Requirement:** 100,000 IoT devices sending telemetry every 5 seconds. Data must be retained for 30 days, then deleted. Need real-time analytics.

**Answer:** Cosmos DB with Autoscale throughput, partition key = `deviceId`, TTL = 30 days (2,592,000 seconds), Synapse Link for analytics, Change Feed for real-time processing.

**Why:** Autoscale handles variable ingestion spikes, deviceId distributes evenly, TTL manages lifecycle, Synapse Link avoids expensive analytical queries on operational store.

---

### Scenario 4: Social Media Platform with Likes/Comments

> **Requirement:** A social platform needs to store posts and track likes. Like counts don't need to be perfectly accurate in real-time. Posts should be visible immediately to the author.

**Answer:** Cosmos DB, **Session consistency** (author sees own writes), **Eventual consistency** relaxed per-request for like counts. Partition key = `userId` for posts container, `postId` for likes/comments container.

**Why:** Session gives read-your-writes for authors; Eventual reduces cost for non-critical counters.

---

### Scenario 5: Multi-Tenant SaaS Application

> **Requirement:** A SaaS platform with 10,000 tenants. Some tenants have 100x more data than others. Need per-tenant isolation and cost efficiency.

**Answer:** Cosmos DB with **hierarchical partition keys** (`tenantId/userId`), database-level shared throughput for small tenants, dedicated containers for large tenants. Consider Autoscale.

**Why:** Hierarchical keys prevent hot partitions from large tenants; mixed throughput model balances cost and performance.

---

### Scenario 6: Event-Driven Microservices

> **Requirement:** An order processing system where placing an order must trigger inventory update, payment processing, and notification services. Each service needs its own data view.

**Answer:** Cosmos DB with **Change Feed** → Azure Functions triggers for each downstream service. Each service maintains its own materialized view. Partition key = `orderId`.

**Why:** Change Feed provides reliable, ordered event stream; materialized views give each service its own read-optimized data model; no coupling between services.

---

### Scenario 7: Low-Traffic Development Environment

> **Requirement:** A team needs a Cosmos DB instance for development and testing. Traffic is sporadic — sometimes no requests for hours, then bursts during testing.

**Answer:** Cosmos DB **Serverless** mode.

**Why:** Pay only for consumed RUs; no idle cost during periods of no activity; supports up to 5,000 RU/s burst which is sufficient for dev/test.

---

### Scenario 8: Migrating from On-Premises MongoDB

> **Requirement:** A company wants to migrate their existing MongoDB 4.x application to Azure with minimal code changes. They need multi-region distribution.

**Answer:** Cosmos DB **API for MongoDB** (v4.x compatible). Use Azure Database Migration Service for migration. Configure multi-region reads.

**Why:** Wire-protocol compatibility means existing MongoDB drivers and queries work. Minimal code changes required.

---

<a id="quick-reference-trigger-table"></a>
## 15. Quick Reference Trigger Table

**"If the scenario says X, think Y"**

| If You See... | Think... |
|---------------|----------|
| "Globally distributed" + low latency | Cosmos DB |
| "Multi-region writes" | Cosmos DB multi-region write + conflict resolution |
| "99.999% availability" | Cosmos DB (multi-region) |
| "Single-digit millisecond latency" | Cosmos DB |
| "Most recent write must be visible" | Strong consistency |
| "Reads can be slightly behind" | Bounded Staleness |
| "User sees their own updates" | Session consistency (default) |
| "Order doesn't matter, eventual OK" | Eventual consistency |
| "Hot partition" / "throttling one key" | Change partition key strategy |
| "Variable/unpredictable traffic" | Autoscale throughput |
| "Dev/test, sporadic traffic" | Serverless |
| "Predictable, steady traffic" | Provisioned throughput |
| "Expire old data automatically" | TTL |
| "Trigger downstream on data change" | Change Feed |
| "Real-time analytics without ETL" | Synapse Link |
| "Event-driven architecture" | Change Feed + Azure Functions |
| "Migrate MongoDB with no code change" | API for MongoDB |
| "Graph traversals / relationships" | Gremlin API |
| "Minimize write costs" | Custom indexing policy |
| "Zero-trust / maximum security" | Private Endpoint + Entra ID + CMK |
| "Self-service point-in-time restore" | Continuous backup (PITR) |
| "Cost savings on steady workload" | Reserved capacity (1yr/3yr) |
| "No public internet exposure" | Private Endpoint |
| "Strong consistency + multi-region writes" | ❌ Not possible (use Bounded Staleness) |
| "Relational data + complex joins" | ❌ Not Cosmos DB → Azure SQL |
| "Full ACID across entities" | ❌ Not Cosmos DB → Azure SQL |

---

<a id="synapse-link-htap"></a>
## 16. Synapse Link (HTAP)

### What is Synapse Link?

**Azure Synapse Link for Cosmos DB** enables **no-ETL analytics** over operational data by automatically syncing data to a column-store **analytical store**.

### Architecture

```
┌─────────────────────────────────────────────────┐
│              Cosmos DB Container                  │
│                                                   │
│  ┌─────────────────┐    ┌─────────────────────┐ │
│  │  Transactional  │───▶│  Analytical Store    │ │
│  │  Store (Row)    │    │  (Column-oriented)   │ │
│  │  OLTP Workloads │    │  Auto-sync, no ETL   │ │
│  └─────────────────┘    └──────────┬──────────┘ │
│                                     │            │
└─────────────────────────────────────┼────────────┘
                                      │ Synapse Link
                                      ▼
                         ┌─────────────────────────┐
                         │   Azure Synapse         │
                         │   Analytics             │
                         │  ┌───────────────────┐  │
                         │  │ Synapse SQL        │  │
                         │  │ Serverless Pool    │  │
                         │  ├───────────────────┤  │
                         │  │ Synapse Spark      │  │
                         │  │ Pools              │  │
                         │  └───────────────────┘  │
                         └─────────────────────────┘
```

### Key Benefits

| Benefit | Detail |
|---------|--------|
| **No ETL** | Automatic sync, no pipelines to build/maintain |
| **No performance impact** | Analytical queries don't consume transactional RU/s |
| **Near real-time** | ~2 minute sync latency |
| **Cost-effective** | Analytical store uses cheaper column storage |
| **Full fidelity** | Schema, including nested structures, preserved |
| **HTAP** | Run analytics on live operational data |

### Analytical Store Details

| Feature | Transactional Store | Analytical Store |
|---------|--------------------|--------------------|
| Format | Row-oriented | Column-oriented |
| Optimized for | Point reads, writes | Aggregations, scans |
| Indexing | All properties indexed | Column-based compression |
| TTL | Configurable | Independent TTL (or infinite) |
| Cost | RU-based | Storage cost only |
| Query engine | Cosmos DB SQL | Synapse SQL/Spark |

### When to Use Synapse Link

- ✅ Need real-time BI dashboards over operational data
- ✅ Complex analytical queries (aggregations, joins) that would be expensive in RUs
- ✅ Want to eliminate ETL pipelines
- ✅ Historical analytics (set Analytical TTL > Transactional TTL)
- ✅ Data science/ML training on live data

> 💡 **AZ-305 Tip:** If the question mentions "analytics on Cosmos DB data without impacting operational performance" or "no-ETL analytics," Synapse Link is the answer. It's also the answer when analytical queries would consume too many RU/s.

---

<a id="az-305-exam-tips-summary"></a>
## 17. 🎯 AZ-305 Exam Tips Summary

### Top 10 Things to Remember

1. **Session consistency is the default** — most workloads use it
2. **Strong consistency ≠ multi-region writes** — they're mutually exclusive
3. **Partition key is immutable** — choose carefully, can't change later
4. **Serverless = single region only** — no geo-replication
5. **Autoscale scales 10% to max** — billed at peak per hour
6. **Change Feed captures inserts/updates** — not deletes (by default)
7. **Synapse Link = no-ETL analytics** — no RU impact on operational store
8. **Private Endpoint** = best network security (not service endpoints)
9. **Continuous backup (PITR)** = self-service restore; Periodic = support ticket
10. **Reserved capacity** = 1yr/3yr commitment for up to 65% savings

### Common Exam Traps

| Trap | Reality |
|------|---------|
| "Use Cosmos DB for everything" | SQL databases are better for complex relational queries |
| "Strong consistency everywhere" | Overkill and expensive; Session suffices for most apps |
| "Autoscale is always cheaper" | Only cheaper if peak:average ratio > 1.5x |
| "Serverless for production" | Limited to 5,000 RU/s burst, single region only |
| "Change partition key" | Can't change it — must recreate container |
| "TTL consumes provisioned RUs" | Background TTL deletions use leftover system RUs |

---

<a id="architecture-decision-flowchart"></a>
## 18. 📐 Architecture Decision Flowchart

```
Need a database on Azure?
│
├── Need complex JOINs / full ACID / relational schema?
│   └── YES → Azure SQL Database or PostgreSQL
│
├── Need global distribution with guaranteed low latency?
│   └── YES → Cosmos DB
│       ├── New app? → API for NoSQL
│       ├── Existing MongoDB? → MongoDB API
│       ├── Graph relationships? → Gremlin API
│       └── Existing Cassandra? → Cassandra API
│
├── Simple key-value, minimal cost, single region?
│   └── YES → Azure Table Storage (or Cosmos DB Table API if need SLAs)
│
├── Time-series / telemetry at massive scale?
│   └── YES → Cosmos DB (with TTL) or Azure Data Explorer
│
└── Data warehouse / OLAP?
    └── YES → Azure Synapse Analytics (+ Synapse Link if source is Cosmos DB)
```

---

*Last Updated: 2025 | Aligned with AZ-305 exam objectives*
