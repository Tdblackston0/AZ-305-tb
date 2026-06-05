# Azure Messaging Services Cheat Sheet

## AZ-305: Designing Infrastructure & Messaging Solutions

> 📝 **Focus:** Service Bus, Event Hubs, Storage Queues, Event Grid, and Kafka patterns
>
> **Perspective:** Senior Cloud Solution Architect | AZ-305 Exam Focus
> **Last Updated:** 2025

---

## Table of Contents

1. [Quick Service Comparison](#1-quick-service-comparison)
2. [Azure Service Bus](#2-azure-service-bus)
3. [Azure Event Hubs](#3-azure-event-hubs)
   - 3a. [Event Hubs + Time Series Insights (TSI)](#event-hubs--time-series-insights-tsi-pattern)
4. [Azure Storage Queues](#4-azure-storage-queues)
5. [Azure Event Grid](#5-azure-event-grid)
6. [Apache Kafka on Azure](#6-apache-kafka-on-azure)
7. [Decision Tree: Which Service?](#7-decision-tree-which-service)
8. [Integration Patterns](#8-integration-patterns)
9. [Security & Access](#9-security--access)
10. [AZ-305 Scenarios](#10-az305-scenarios)
11. [Quick Reference Trigger Table](#11-quick-reference-trigger-table)

---

## 1. Quick Service Comparison

| Aspect | Service Bus | Event Hubs | Storage Queue | Event Grid | Kafka |
|--------|-------------|-----------|---------------|-----------|-------|
| **Primary Use** | Enterprise messaging | Streaming/IoT | Simple queuing | Event routing | Distributed streaming |
| **Message Size** | 1 MB (standard) / 100 MB (premium) | 1 MB (configurable) | 64 KB (default) | 64 KB | 1 MB (default) |
| **Throughput** | ~1,000 msg/sec per queue | ~1 MB/sec per partition | ~2,000 msg/sec per queue | Millions of events/sec | Highly scalable |
| **Latency** | Milliseconds | Sub-millisecond | Milliseconds | Milliseconds | Milliseconds |
| **Delivery** | At-least-once, Exactly-once (sessions) | At-least-once | At-least-once | Best-effort | At-least-once |
| **Ordering** | Per-session or per-partition | Per-partition | FIFO (within segment) | No guarantee | Per-partition |
| **Pricing** | Per operation + messages | Throughput units | Per transaction | Per operation | Brokers or managed |
| **Dead Letter Queue** | Yes | Yes (optional) | Yes | No | N/A |
| **TTL (Time-to-Live)** | 1 minute - 14 days | 1 - 7 days | 7 days fixed | N/A | Per-broker config |
| **Duplicate Detection** | Yes (20 min window) | Depends on consumer | No | No | Depends on producer |
| **Scaling** | Manual (partitions) | Automatic with TUs | Manual | Automatic | Manual (partitions) |

---

## 2. Azure Service Bus

### Overview

**Enterprise-grade messaging platform** for decoupling application components with guaranteed delivery, ordering, and duplicate detection. Best for business workflows and integration scenarios.

### Core Concepts

| Concept | Description |
|---------|-------------|
| **Namespace** | Logical container for queues, topics, and subscriptions |
| **Queue** | Point-to-point messaging (one producer, one consumer group) |
| **Topic** | Pub-sub messaging (one producer, multiple subscriptions) |
| **Subscription** | Consumer endpoint on a topic (filtering + delivery rules) |
| **Session** | Logical grouping of related messages for ordering guarantees |
| **Dead Letter Queue** | Auto-collection of undeliverable messages |
| **Session State** | Persistent state per session (managed by broker) |

### Queue vs Topic

```
QUEUE (Point-to-Point):
┌────────────┐
│  Producer  │──▶ Queue ──▶ Consumer
└────────────┘

Each message goes to ONE consumer only
Use: Task distribution, background jobs


TOPIC (Pub-Sub):
                  ┌──▶ Subscription 1 ──▶ Consumer A
┌────────────┐    │
│  Producer  │──▶ Topic
└────────────┘    │
                  └──▶ Subscription 2 ──▶ Consumer B

Each message goes to ALL subscribers
Use: Event broadcasting, notifications
```

### Tiers

| Tier | Throughput | Features | Best For |
|------|-----------|----------|----------|
| **Basic** | 1 million msg/day | Queues only, basic features | Dev/test, simple scenarios |
| **Standard** | High (per-unit) | Queues, Topics, FIFO sessions | Most production workloads |
| **Premium** | Very high | Dedicated capacity, TLS inspection, JMS 2.0 | Mission-critical, compliance |

### Key Features

- **Duplicate Detection** — 20-minute window, automatic de-duplication
- **Sessions** — Group related messages; guarantee FIFO within session
- **Scheduled Delivery** — Defer message processing until specific time
- **Auto-Forward** — Chain queues/topics (auto-forward on completion)
- **Dead Letter Queue** — Capture undeliverable messages (max delivery attempts exceeded)
- **Filters** — SQL-based filtering on subscription rules

### Example: Order Processing with Sessions

```csharp
// Order 123 → Messages 1, 2, 3 (same session ID)
// Service Bus guarantees in-order delivery within session
// Order 124 → Messages A, B, C (different session)
// Parallel processing possible across sessions
```

### Pricing

- **Basic:** Flat monthly fee (~$10)
- **Standard:** Operations + messaging (~$0.50 per million operations)
- **Premium:** Dedicated PU (Premium Unit) (~$0.925 per hour)

> 💡 **Tip:** Premium is often cheaper than Standard for high-volume workloads (>1M msg/day)

---

## 3. Azure Event Hubs

### Overview

**Distributed streaming platform** optimized for ingesting massive volumes of events. Built for IoT, telemetry, and real-time processing with automatic partitioning and consumer group isolation.

### Core Concepts

| Concept | Description |
|---------|-------------|
| **Event Hub** | Named entity within namespace (like a topic) |
| **Partition** | Ordered sequence of events (key for scaling) |
| **Partition Key** | Route events to specific partition (ensure ordering) |
| **Consumer Group** | Independent reader view of event stream (time-based offset) |
| **Throughput Unit (TU)** | 1 TU = 1 MB/s ingress, 2 MB/s egress |
| **Epoch** | Consumer group checkpoint (prevents duplicate delivery) |
| **Capture** | Auto-archive events to ADLS Gen2 or Blob Storage |

### Partition Strategy

```
Producer sends with Partition Key = "Device-123"
                    ↓
            Event Hubs Routes to Partition N
                    ↓
    All messages for Device-123 go to Partition N
                    ↓
    Consumer group reads from Partition N in order
```

**Without partition key:** Random distribution (no ordering guarantee)

### Tiers & Scaling

| Tier | Throughput | Auto-scale | Capture | Best For |
|------|-----------|-----------|---------|----------|
| **Basic** | 1 TU max | No | No | Dev/test, <1MB/s |
| **Standard** | 20 TU | Yes (auto-inflate) | Yes | Most production |
| **Dedicated** | Up to 100 MB/s | Yes | Yes | Large-scale, compliance |

### Consumer Groups

```
Single Event Hub, Multiple Consumer Groups:
┌──────────────────────────────────┐
│        Event Hub                 │
│  (Partition 0, 1, 2, 3)          │
└──────────────────────────────────┘
  │                    │                    │
  ▼                    ▼                    ▼
Group A            Group B            Group C
(Analytics)     (Real-time Alerts)  (Archive)
Each reads independently with own offset
```

### Capture Pattern

```
Events → Event Hub → Automatic capture → ADLS Gen2 / Blob
                        (configurable:
                         size or time window)
                        ↓
                    Parquet/Avro files
                    ↓
                    Databricks/Synapse queries
```

### Pricing

- **Throughput Units (TUs):** ~$0.032 per hour (1 TU)
- **Premium:** ~$0.925 per hour (dedicated cluster)
- **Captured data:** Storage charges only

### Event Hubs + Time Series Insights (TSI) Pattern

**Common scenario:** IoT devices send telemetry → Event Hub → TSI for time-series analytics

```
IoT Devices (sensors, equipment)
    ↓
Event Hub (ingestion layer)
    ↓
├─→ Azure Time Series Insights (time-series queries, UI)
├─→ Stream Analytics (real-time alerting)
└─→ Event Hub Capture (archive to ADLS Gen2)
```

**Configuration:**

| Component | Role | Configuration |
|---|---|---|
| **Event Hub** | Central ingestion point | 1-2 TUs, standard tier, auto-inflate |
| **Partition Key** | Device identification | `deviceId` to ensure ordering per device |
| **TSI Instance** | Time-series storage & analytics | Gen2, partition by device ID, 30-day warm retention |
| **TSI Variables** | Computed metrics | Avg temp, humidity alerts, anomaly detection |
| **TSI Hierarchies** | Organize data | Building → Floor → Room → Sensor |

**Example IoT + TSI Data Flow:**

```json
// Raw event from IoT Hub → Event Hub
{
  "deviceId": "sensor-floor2-room201",
  "temperature": 23.5,
  "humidity": 62,
  "co2": 420,
  "timestamp": "2024-06-04T14:30:00Z"
}

// TSI stores and makes queryable:
{
  "Time Series ID": "sensor-floor2-room201",
  "Timestamp": "2024-06-04T14:30:00Z",
  "Variables": {
    "AvgTemperature": 23.5,
    "HighHumidityAlert": 0,
    "CO2Anomaly": false
  },
  "Hierarchy": ["Building-A", "Floor-2", "Room-201"]
}

// TSI Explorer returns trend data, alerts, and aggregations
```

**Exam Scenario:** "Design a solution for 10,000 IoT devices sending 1 reading per minute. Need real-time dashboard showing last 30 days of building temperature trends per floor."

**Answer:** 
- Event Hub with 4-6 TUs (10K devices × 60 readings/hour = ~167K events/hour)
- Partition key = `deviceId` for per-device ordering
- TSI Gen2 instance with warm retention 30 days
- TSI Hierarchies: Building → Floor → Sensors
- TSI Explorer dashboard with time-series aggregations (avg, min, max per floor)
- **Cost:** ~$200-300/month (Event Hub + TSI storage)

**Why NOT alternatives?**
- ❌ Stream Analytics alone: Can't query historical data for trends
- ❌ Data Lake: Overkill for this volume, no pre-built time-series UI
- ❌ Synapse: Slower queries, manual schema management

---

## 4. Azure Storage Queues

### Overview

**Simple, cost-effective** message queue built on Azure Storage. Best for decoupling components when you don't need advanced features of Service Bus.

### Key Features

| Feature | Details |
|---------|---------|
| **Message Size** | Up to 64 KB (configurable with encoding) |
| **TTL** | Up to 7 days |
| **Throughput** | ~2,000 messages/sec per queue |
| **Visibility Timeout** | 0 seconds - 7 days (lock duration) |
| **Queue Name** | 1-63 characters, lowercase alphanumeric + hyphens |
| **Scale** | Automatic (Storage account scale limits apply) |

### Visibility Timeout Pattern

```
Message in queue (visible)
         │
    Consumer reads message
         │
    Message enters "invisible" state (processing)
         │
    TTL expires (e.g., 5 minutes)
         │
    Either:
    ├─ Consumer deletes message (success)
    └─ Message becomes visible again (retry)
```

### Use Case: Background Job

```
Web App → Queue ← Azure Function (trigger)
                     │
                     ├─ Process image
                     ├─ Delete from queue (done)
                     └─ OR: Let TTL expire if failed
```

### Pricing

- **Operations:** ~$0.0004 per 10,000 operations
- **Storage:** ~$0.05 per GB/month
- **Extremely cost-effective** for low-volume scenarios

### Storage Queue vs Service Bus Queue

| Scenario | Storage Queue | Service Bus |
|----------|---------------|------------|
| Simple decoupling | ✅ Best | Overkill |
| Need FIFO ordering | ❌ Weak | ✅ Strong (sessions) |
| Duplicate detection needed | ❌ | ✅ Built-in |
| Long-term retention | ✅ (7 days OK) | Worse (14 days) |
| Cost-sensitive | ✅ Best | More expensive |
| Enterprise workflows | ❌ | ✅ Best |
| DLQ needed | ✅ | ✅ |

---

## 5. Azure Event Grid

### Overview

**Serverless event routing service** for reacting to events across Azure and custom sources. Instant event delivery with filtering and webhooks.

### Core Concepts

| Concept | Description |
|---------|-------------|
| **Event Source** | Azure resource or custom app that emits events |
| **Topic** | Named container for events (system or custom) |
| **Event Subscription** | Connection between topic and handler (with filters) |
| **Event Handler** | Webhook, Logic App, Function, Service Bus, Event Hub, etc. |
| **Delivery Attempt** | Event Grid retries failed deliveries (exponential backoff) |

### System vs Custom Topics

```
SYSTEM TOPIC (Built-in):
Azure Resource → Event Grid System Topic → Event Grid → Handlers
  (e.g., Storage,          (automatic)
   VM, SQL DB)

CUSTOM TOPIC:
Your App → Custom Topic (you manage) → Event Grid → Handlers
```

### Event Filtering

```
Topic receives events:
├─ ResourceGroup: "prod"
├─ ResourceType: "Storage"
└─ EventType: "BlobCreated"

Subscription 1 Filter: EventType = "BlobCreated" ──▶ Handler A
Subscription 2 Filter: ResourceGroup = "prod"   ──▶ Handler B
```

### Common Event Sources

| Source | Event Examples |
|--------|----------------|
| **Blob Storage** | Created, Deleted, Updated |
| **SQL Database** | Tables created, rows updated |
| **Key Vault** | Secret created, certificate expiring |
| **Container Registry** | Image pushed, deleted |
| **App Service** | App published, deployment slots |
| **Azure VMs** | Created, started, deallocated |
| **Custom** | Any event your app sends |

### Pricing

- **First 100,000 operations/month:** Free
- **After:** ~$0.60 per million operations

---

## 6. Apache Kafka on Azure

### Overview

**Distributed streaming platform** for high-throughput, durable message streaming. Available two ways on Azure:

1. **Event Hubs with Kafka Protocol** (managed, easiest)
2. **HDInsight Kafka** (self-managed cluster)

### Event Hubs Kafka Integration

```
Kafka Producer → Event Hubs (Kafka protocol enabled)
                     ↓
              Event Hubs partitions
                     ↓
              Consumer group
                     ↓
            Kafka Consumer / Stream Analytics
```

**Pros:**
- Zero code changes (use standard Kafka SDK)
- No cluster management
- Auto-scaling with Event Hubs TUs
- Capture to ADLS Gen2

**Cons:**
- Some Kafka features not supported (broker config, ACLs)

### HDInsight Kafka

```
You manage:
├─ Cluster provisioning
├─ Broker configuration
├─ Scaling decisions
├─ Upgrades
└─ Security policies
```

**Best for:**
- Advanced Kafka features needed
- Stream processing (Kafka Streams, Flink)
- Multi-cluster federation
- Custom configurations

### Kafka Producer → Consumer Pattern

```
Producer sends to Topic with Partition Key = "User-123"
         │
         ▼
Topic Partition 0: [User-123-msg1, User-123-msg2]
Topic Partition 1: [User-456-msg1]
         │
         ▼
Consumer Group reads from assigned partitions
(offset-based, guaranteed ordering per partition)
```

---

## 7. Decision Tree: Which Service?

```
START: What's your messaging pattern?

├─ Need to DECOUPLE APPLICATION COMPONENTS?
│  ├─ Simple task queue (background jobs)? → STORAGE QUEUE
│  ├─ Enterprise workflows + ordering + DLQ? → SERVICE BUS (Queue)
│  └─ Broadcast event to many subscribers? → SERVICE BUS (Topic)
│
├─ Need HIGH-VOLUME STREAMING (IoT/Telemetry)?
│  ├─ Real-time + Kafka clients available? → EVENT HUBS (Kafka)
│  ├─ Pure Event Hubs (Azure SDK)? → EVENT HUBS
│  └─ Complex stream processing (Flink)? → HDInsight Kafka
│
├─ Need EVENT-DRIVEN AUTOMATION?
│  ├─ React to Azure resource events? → EVENT GRID
│  ├─ Multiple handlers (webhook, function)? → EVENT GRID
│  └─ Custom app events? → EVENT GRID (custom topic)
│
└─ STILL UNSURE?
   ├─ High volume + throughput? → Event Hubs
   ├─ Enterprise patterns + ordering? → Service Bus
   ├─ Cost-sensitive + simple? → Storage Queue
   ├─ React to events + automation? → Event Grid
   └─ Kafka ecosystem needed? → Event Hubs (Kafka) or HDInsight Kafka
```

### Comparative Scenarios

| Scenario | Best Choice | Why |
|----------|------------|-----|
| Process images uploaded to blob storage in batch | Storage Queue | Simple, cost-effective, triggers Functions |
| Guarantee message processing order for payment transactions | Service Bus (Queue + Sessions) | FIFO + duplicate detection critical |
| Ingest 100K sensor readings per second | Event Hubs | Partitioning + throughput needed |
| Notify multiple teams of approval event | Service Bus (Topic) | Pub-sub filtering + reliability |
| Auto-scale function on blob upload | Event Grid | Serverless, instant notification |
| Stream logs to SIEM and retain 30 days | Event Hubs + Capture | High-volume capture + archival |
| Small background job queue (web app) | Storage Queue | Cost, simplicity, no ordering needed |

---

## 8. Integration Patterns

### Pattern 1: Task Queue (Background Jobs)

```
Web Application
    │
    ├─ POST /process-image
    │
    ▼
Storage Queue ──▶ Azure Function (trigger)
                      │
                      ├─ Resize image
                      ├─ Upload to Blob
                      ├─ Delete from queue
                      └─ (On error: retry or DLQ)
```

**Best Practice:** Use Storage Queue for simple, cheap background work.

### Pattern 2: Event Broadcasting (Pub-Sub)

```
Order Service publishes event:
"OrderCreated" → Service Bus Topic
                      │
        ┌─────────────┼─────────────┐
        │             │             │
        ▼             ▼             ▼
    Inventory   Notification    Accounting
    Service     Service          Service
   (filters:    (all orders)    (high-value
    high qty)                    orders)
```

**Best Practice:** Use Service Bus Topic for reliable pub-sub.

### Pattern 3: Real-Time Analytics Pipeline

```
IoT Devices
    │
    ├─ Temperature sensors
    ├─ Pressure gauges
    └─ Flow meters
        │
        ▼
Event Hubs (partitioned by device)
        │
    ┌───┴───┬───────────┐
    │       │           │
    ▼       ▼           ▼
Stream   Power BI   ADLS Gen2
Analytics  (live)   (archive)
```

**Best Practice:** Partition by device ID to preserve ordering and enable parallel processing.

### Pattern 4: Serverless Event-Driven Architecture

```
Blob Upload → Event Grid → Function1 (generate thumbnail)
         │                 Function2 (update database)
         │                 Logic App (send email)
         │                 Webhook (external system)
         └─ All triggered instantly, independently scaled
```

**Best Practice:** Use Event Grid for automatic event routing with no infrastructure.

### Pattern 5: Kafka Streaming with Event Hubs

```
Temperature Sensors
        │
        ▼
Event Hubs (Kafka protocol enabled)
        │
    ┌───┴───┬──────────┐
    │       │          │
    ▼       ▼          ▼
Spark   Kafka Streams  Stream
(batch)  (real-time)   Analytics
```

**Best Practice:** Use Event Hubs Kafka for zero-code Kafka integration.

---

## 9. Security & Access

### Service Bus Security Layers

```
Layer 1: Network
├─ Firewall rules
├─ VNet service endpoints
├─ Private endpoints (premium tier)
└─ IP filtering

Layer 2: Authentication
├─ Shared Access Signature (SAS)
├─ Managed Identity (recommended)
└─ Azure AD / Entra ID

Layer 3: Authorization (RBAC)
├─ Azure Service Bus Data Owner
├─ Azure Service Bus Data Sender
├─ Azure Service Bus Data Receiver
└─ Custom roles

Layer 4: Encryption
├─ At-rest (automatic with CMK option)
├─ In-transit (TLS 1.2+)
└─ Premium tier: TLS inspection available
```

### Managed Identity Pattern

```csharp
// Instead of connection strings with keys:
var serviceBusClient = new ServiceBusClient(
    "myservicebus.servicebus.windows.net",
    new DefaultAzureCredential()  // ← Managed Identity
);
```

**Advantages:**
- No credentials to store/rotate
- Audit trail in Entra ID
- Works in CI/CD pipelines

### RBAC Roles

| Role | Permissions |
|------|-------------|
| **Data Owner** | Full (send, receive, manage) |
| **Data Sender** | Send messages only |
| **Data Receiver** | Receive messages only |
| **Data Contributor** | Send + Receive (not manage) |

### Event Hubs Security

Similar to Service Bus, but also supports:
- **Kafka protocol:** Standard SASL/SSL
- **Consumer group isolation:** Each group has own offset/auth
- **Capture role:** `Storage Blob Data Contributor` to destination

---

## 10. AZ-305 Scenarios

### Scenario 1: E-Commerce Order Processing

**Situation:** Retail platform processes 10,000 orders/day. Orders must be processed in sequence per customer, with guaranteed delivery to inventory and payment systems. If payment fails, order must retry.

**Solution:** Azure Service Bus with Topics and Sessions

```
OrderService → Service Bus Topic: "Orders"
                    │
        ┌───────────┼───────────┐
        │           │           │
        ▼           ▼           ▼
   Subscription  Subscription  Subscription
   "Inventory"   "Payment"    "Notification"
   (filters:     (filters:    (all orders)
    qty>10)      price>$100)

Messages grouped by CustomerID (session)
→ Guaranteed FIFO within customer
→ Duplicate detection (idempotent)
→ Auto-retry to DLQ after max attempts
```

**Why not alternatives:**
- Storage Queue: No topic pub-sub, weak ordering
- Event Hubs: Not designed for guaranteed delivery

---

### Scenario 2: IoT Telemetry Platform

**Situation:** Manufacturing plant has 10,000 IoT sensors sending temperature, pressure, humidity every second. Need real-time dashboards, long-term archival, and anomaly detection.

**Solution:** Event Hubs with Partitioning and Capture

```
10K Sensors
    │ (10K events/sec)
    ▼
Event Hubs (100 partitions) ← Partition key = SensorID
    │
    ├─ Partition by location (100 locations)
    │  → Each partition consumed independently
    │  → Preserve sensor readings in order
    │
    ├─ Real-time stream → Stream Analytics → Alerts
    ├─ Power BI connector → Live dashboard
    └─ Capture → ADLS Gen2 → Parquet → Databricks analysis
```

**Configuration:**
- **Throughput Units:** 10 TU (auto-scale if needed)
- **Partition Count:** 100 (1 per location, or by sensor type)
- **Capture:** Daily Parquet files to ADLS Gen2

**Why not alternatives:**
- Service Bus: Not designed for streaming volumes
- Storage Queue: No partitioning, throughput limits

---

### Scenario 3: Microservices Decoupling

**Situation:** E-commerce platform has 50 microservices. Order Service publishes events; Inventory, Billing, Shipping, and Notification services consume independently with different processing requirements.

**Solution:** Service Bus Topic with Subscriptions + Filters

```
Order Service publishes:
{
  "eventType": "OrderCreated",
  "amount": 500,
  "urgency": "high"
}

    ↓
Service Bus Topic: "Orders"
    │
    ├─ Subscription: Inventory
    │  Filter: eventType = "OrderCreated"
    │  DLQ: Exponential backoff retry
    │
    ├─ Subscription: Billing
    │  Filter: amount > 100
    │  DLQ: Billing failures
    │
    ├─ Subscription: Shipping
    │  Filter: urgency = "high"
    │  MaxDelivery: 3 attempts
    │
    └─ Subscription: Notification
       Filter: None (all events)
       Handler: Azure Function
```

**Benefits:**
- Services added/removed without code changes
- Filtered delivery reduces processing
- Independent scaling

---

### Scenario 4: Event-Driven Application

**Situation:** Healthcare provider needs to react to various Azure resource events: VMs being created, databases provisioned, compliance audits triggered. Each event routes to different handlers.

**Solution:** Event Grid with System Topics

```
Azure Resources emit events
    │
Event Grid System Topics:
├─ Microsoft.Storage/storageAccounts
│  (BlobCreated, BlobDeleted, etc.)
├─ Microsoft.Compute/virtualMachines
│  (VirtualMachineCreated, etc.)
└─ Microsoft.KeyVault/vaults
   (SecretCreated, etc.)
    │
Event Subscriptions (with filters):
├─ Handler: Logic App (send email)
├─ Handler: Function (auto-remediate)
├─ Handler: Webhook (external audit system)
└─ Handler: Service Bus (durable log)
```

**Pricing:** Essentially free (first 100K operations/month free).

---

### Scenario 5: Cost-Optimized Background Job System

**Situation:** Startup with minimal budget. 500 background jobs/day. No ordering requirements. Occasional failures acceptable.

**Solution:** Azure Storage Queue + Functions

```
Web App writes job to Queue ← ~$0.0004 per 100 operations
    │
    ▼
Azure Function (consumption plan)
    │
    ├─ Process job
    ├─ Delete from queue (success)
    └─ OR let TTL expire (failure, auto-retry)

Cost/month:
- Storage: $0.05 (negligible)
- Queue ops: $0.02 (500 jobs × 30 days)
- Function: $0.20 (execution time)
─────────────
TOTAL: ~$0.27/month
```

---

## 11. Quick Reference Trigger Table

### "If the Scenario Says X, Think Y"

| Scenario Keywords | Best Service |
|-------------------|-------------|
| "Decouple components," "simple queue" | Storage Queue |
| "Guaranteed delivery," "enterprise workflow," "FIFO" | Service Bus (Queue) |
| "Broadcast," "pub-sub," "multiple subscribers" | Service Bus (Topic) |
| "High volume," "IoT," "telemetry," "streaming" | Event Hubs |
| "Real-time dashboards," "sensors," "thousands/sec" | Event Hubs + Stream Analytics |
| "React to Azure events," "automation," "webhooks" | Event Grid |
| "Kafka producers," "Kafka consumers," "streaming" | Event Hubs (Kafka) or HDInsight |
| "Background jobs," "Functions," "cost-sensitive" | Storage Queue |
| "Order processing," "duplicate detection," "sessions" | Service Bus (Sessions) |
| "Long-term archival," "Parquet files," "analytics" | Event Hubs + Capture |
| "Complex stream processing," "Flink," "advanced" | HDInsight Kafka or Databricks |
| "Compliance logging," "audit trail," "retention" | Event Grid + Event Hub or Service Bus |

### Anti-Patterns

| ❌ Don't... | ✅ Instead... |
|------------|--------------|
| Use Event Grid for guaranteed delivery | Use Service Bus |
| Use Storage Queue for ordered sequences | Use Service Bus (Sessions) |
| Use Service Bus for 100K+ events/sec | Use Event Hubs |
| Use Storage Queue for complex workflows | Use Service Bus |
| Use Event Hubs for pub-sub filtering | Use Service Bus Topic |
| Store connection strings in code | Use Managed Identities |
| Over-provision throughput | Start small, monitor, scale up |
| Mix Kafka and Event Hub SDKs in same app | Choose one protocol, stay consistent |

---

## Quick Comparison Matrix

| Need | Storage Queue | Service Bus | Event Hubs | Event Grid |
|------|---------------|------------|-----------|-----------|
| Simple decoupling | ✅ Best | Overkill | No | No |
| Guaranteed delivery | ✅ | ✅ Best | ✅ | ❌ |
| Ordering guarantee | ❌ Weak | ✅ Best (sessions) | ✅ (per-partition) | ❌ |
| Pub-sub (topics) | ❌ | ✅ Best | No | ✅ |
| High throughput (1M+/sec) | ❌ | ❌ | ✅ Best | ✅ |
| Cost-effective | ✅ Best | ✗ Higher | Mid | ✅ |
| Event routing/filtering | ❌ | ✅ | ✅ | ✅ Best |
| Built-in capture | ❌ | ❌ | ✅ | ❌ |
| Kafka support | ❌ | ❌ | ✅ | ❌ |

---

## Key Takeaways for AZ-305

1. **Service Bus = Enterprise Messaging** → ordering, delivery guarantees, pub-sub, sessions
2. **Event Hubs = Streaming/IoT** → high-throughput, partitioning, capture to ADLS
3. **Storage Queue = Simple Decoupling** → cost-effective, no complex features
4. **Event Grid = Event Routing** → serverless, automatic, webhook-driven
5. **Kafka = Distributed Streaming** → via Event Hubs or HDInsight
6. **Always use Managed Identities** → never hardcode connection strings
7. **Partition strategy matters** → impacts ordering and throughput
8. **Consider capture + archival** → Event Hubs → ADLS → Analytics
9. **Filter at the service** → Service Bus filters, Event Grid rules (reduce downstream load)
10. **Cost scales with usage** → Queue operations, throughput units, message volume

---

*End of Messaging Cheat Sheet — Good luck on AZ-305! 🎯*
