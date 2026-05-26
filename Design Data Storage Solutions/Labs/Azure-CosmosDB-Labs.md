# Azure Cosmos DB Hands-On Labs (AZ-305)

> 📖 **Cheat Sheet:** [Azure Cosmos DB Cheat Sheet](../CheatSheets/Azure-CosmosDB.md)

## Prerequisites

- Azure subscription with Contributor access
- Azure CLI installed (`az --version` ≥ 2.50)
- Azure Functions Core Tools (for Lab 5)
- .NET 6+ or Python 3.9+ (for SDK labs)

```bash
# Login and set subscription
az login
az account set --subscription "<YOUR_SUBSCRIPTION_ID>"

# Register resource providers
az provider register --namespace Microsoft.DocumentDB
az provider register --namespace Microsoft.Synapse
```

---

## Infrastructure as Code Starter Templates (Bicep & Terraform)

Use these templates to deploy a repeatable Cosmos DB base environment for labs.

```bicep
param location string = resourceGroup().location
param accountName string

resource cosmos 'Microsoft.DocumentDB/databaseAccounts@2024-05-15' = {
  name: accountName
  location: location
  kind: 'GlobalDocumentDB'
  properties: {
    databaseAccountOfferType: 'Standard'
    locations: [
      {
        locationName: location
        failoverPriority: 0
        isZoneRedundant: false
      }
    ]
    consistencyPolicy: {
      defaultConsistencyLevel: 'Session'
    }
  }
}
```

```hcl
resource "azurerm_cosmosdb_account" "lab" {
  name                = var.cosmos_account_name
  location            = var.location
  resource_group_name = var.resource_group_name
  offer_type          = "Standard"
  kind                = "GlobalDocumentDB"
  consistency_policy {
    consistency_level = "Session"
  }
  geo_location {
    location          = var.location
    failover_priority = 0
  }
}
```

---

## Lab 1: Create Cosmos DB Account with NoSQL API

### Objective
Create a Cosmos DB account using the NoSQL (formerly SQL) API, configure a database and container with an effective partition key, and observe the RU cost difference between single-partition and cross-partition queries.

### Key AZ-305 Concepts Practiced
- Choosing the correct API (NoSQL for flexible schema, SQL-like queries)
- Partition key selection strategy (high cardinality, even distribution)
- Request Unit (RU) cost model and query optimization
- Session consistency as the default balanced choice

### Steps

#### 1.1 Create Resource Group and Cosmos DB Account

```bash
# Variables
RG="rg-cosmosdb-lab01"
LOCATION="eastus"
ACCOUNT="cosmos-lab01-$RANDOM"

# Create resource group
az group create --name $RG --location $LOCATION

# Create Cosmos DB account with Session consistency (default, balanced choice)
az cosmosdb create \
  --name $ACCOUNT \
  --resource-group $RG \
  --default-consistency-level Session \
  --locations regionName=$LOCATION failoverPriority=0 isZoneRedundant=false \
  --kind GlobalDocumentDB
```

#### 1.2 Create Database and Container

```bash
# Create database with shared throughput (400 RU/s minimum)
az cosmosdb sql database create \
  --account-name $ACCOUNT \
  --resource-group $RG \
  --name "OrdersDB"

# Create container with partition key /customerId
# Good partition key: high cardinality, frequently used in WHERE clauses
az cosmosdb sql container create \
  --account-name $ACCOUNT \
  --resource-group $RG \
  --database-name "OrdersDB" \
  --name "Orders" \
  --partition-key-path "/customerId" \
  --throughput 400
```

#### 1.3 Insert Sample Documents

```bash
# Get connection string for SDK operations
CONN_STRING=$(az cosmosdb keys list \
  --name $ACCOUNT \
  --resource-group $RG \
  --type connection-strings \
  --query "connectionStrings[0].connectionString" -o tsv)

echo "Connection String: $CONN_STRING"

# Insert documents using REST API (or use Data Explorer in Portal)
# Get the primary key
PRIMARY_KEY=$(az cosmosdb keys list \
  --name $ACCOUNT \
  --resource-group $RG \
  --query "primaryMasterKey" -o tsv)

echo "Primary Key: $PRIMARY_KEY"
echo "Account: $ACCOUNT"
echo ""
echo "Use Azure Portal Data Explorer to insert these documents:"

cat << 'EOF'
Document 1:
{
  "id": "order-001",
  "customerId": "cust-100",
  "product": "Widget A",
  "quantity": 5,
  "price": 29.99,
  "region": "East"
}

Document 2:
{
  "id": "order-002",
  "customerId": "cust-100",
  "product": "Widget B",
  "quantity": 2,
  "price": 49.99,
  "region": "West"
}

Document 3:
{
  "id": "order-003",
  "customerId": "cust-200",
  "product": "Widget A",
  "quantity": 10,
  "price": 29.99,
  "region": "East"
}
EOF
```

#### 1.4 Compare Query Costs (Single-Partition vs Cross-Partition)

```bash
echo "Run these queries in the Azure Portal Data Explorer and compare RU charges:"
echo ""
echo "--- Single-Partition Query (efficient - targets one logical partition) ---"
echo "SELECT * FROM c WHERE c.customerId = 'cust-100'"
echo "Expected: ~2-3 RUs (fan-out to 1 partition only)"
echo ""
echo "--- Cross-Partition Query (expensive - must fan out to ALL partitions) ---"
echo "SELECT * FROM c WHERE c.region = 'East'"
echo "Expected: ~5-10 RUs (fan-out to all physical partitions)"
echo ""
echo "--- Point Read (most efficient) ---"
echo "Read document by id='order-001' and partitionKey='cust-100'"
echo "Expected: ~1 RU (direct lookup, no query engine)"
```

### Verification

1. In Azure Portal → Cosmos DB → Data Explorer, run both queries
2. Check the "Query Stats" tab for each → compare "Request Charge" (RU/s)
3. Single-partition queries should show significantly lower RU cost
4. Point reads (by id + partition key) should be ~1 RU

### Cleanup

```bash
az group delete --name $RG --yes --no-wait
```

---

## Lab 2: Consistency Levels Exploration

### Objective
Understand the five consistency levels in Cosmos DB, their trade-offs between latency/availability and consistency guarantees, and how to override consistency per-request.

### Key AZ-305 Concepts Practiced
- Five consistency levels: Strong → Bounded Staleness → Session → Consistent Prefix → Eventual
- Trade-off spectrum: consistency vs. latency vs. availability vs. throughput
- Per-request consistency override (can only weaken, never strengthen)
- RU cost implications (Strong/Bounded Staleness = 2x read cost in multi-region)
- AZ-305 often tests: when to choose Strong vs. Session vs. Eventual

### Steps

#### 2.1 Create Accounts with Different Consistency Levels

```bash
# Variables
RG="rg-cosmosdb-lab02"
LOCATION="eastus"

az group create --name $RG --location $LOCATION

# Account 1: Strong Consistency (linearizable reads, highest latency)
ACCOUNT_STRONG="cosmos-strong-$RANDOM"
az cosmosdb create \
  --name $ACCOUNT_STRONG \
  --resource-group $RG \
  --default-consistency-level Strong \
  --locations regionName=eastus failoverPriority=0 \
  --locations regionName=westus failoverPriority=1

# Account 2: Session Consistency (default, per-session guarantees)
ACCOUNT_SESSION="cosmos-session-$RANDOM"
az cosmosdb create \
  --name $ACCOUNT_SESSION \
  --resource-group $RG \
  --default-consistency-level Session \
  --locations regionName=eastus failoverPriority=0 \
  --locations regionName=westus failoverPriority=1

# Account 3: Eventual Consistency (lowest latency, highest availability)
ACCOUNT_EVENTUAL="cosmos-eventual-$RANDOM"
az cosmosdb create \
  --name $ACCOUNT_EVENTUAL \
  --resource-group $RG \
  --default-consistency-level Eventual \
  --locations regionName=eastus failoverPriority=0 \
  --locations regionName=westus failoverPriority=1
```

#### 2.2 Observe Consistency Configuration

```bash
# Check consistency level for each account
az cosmosdb show --name $ACCOUNT_STRONG --resource-group $RG \
  --query "{name:name, consistency:consistencyPolicy.defaultConsistencyLevel}" -o table

az cosmosdb show --name $ACCOUNT_SESSION --resource-group $RG \
  --query "{name:name, consistency:consistencyPolicy.defaultConsistencyLevel}" -o table

az cosmosdb show --name $ACCOUNT_EVENTUAL --resource-group $RG \
  --query "{name:name, consistency:consistencyPolicy.defaultConsistencyLevel}" -o table
```

#### 2.3 Update Consistency Level (Change Without Recreating)

```bash
# You CAN change the default consistency level on an existing account
az cosmosdb update \
  --name $ACCOUNT_SESSION \
  --resource-group $RG \
  --default-consistency-level BoundedStaleness \
  --max-interval 5 \
  --max-staleness-prefix 100

# Verify the change
az cosmosdb show --name $ACCOUNT_SESSION --resource-group $RG \
  --query "consistencyPolicy" -o json
```

#### 2.4 Per-Request Consistency Override (SDK Example)

```bash
cat << 'EOF'
// C# SDK Example - Override consistency per request (can only WEAKEN)
// Account default: Session → Can override to Consistent Prefix or Eventual

using Microsoft.Azure.Cosmos;

CosmosClient client = new CosmosClient(connectionString);
Container container = client.GetContainer("OrdersDB", "Orders");

// Read with weaker consistency (Eventual) for lower latency
ItemRequestOptions options = new ItemRequestOptions
{
    ConsistencyLevel = ConsistencyLevel.Eventual  // Weaker than account default
};

var response = await container.ReadItemAsync<Order>(
    "order-001",
    new PartitionKey("cust-100"),
    options
);

Console.WriteLine($"RU Charge: {response.RequestCharge}");
// Eventual reads cost less RU in multi-region scenarios

// NOTE: You CANNOT strengthen consistency per-request
// If account is Session, you cannot request Strong per-request
EOF
```

#### 2.5 RU Cost Comparison Table

```bash
echo "=== Consistency Level Impact on RU Cost ==="
echo ""
echo "| Consistency Level     | Read Cost (Multi-Region) | Write Latency | Availability |"
echo "|-----------------------|--------------------------|---------------|--------------|"
echo "| Strong                | 2x RU (quorum read)     | Highest       | Lower (needs quorum) |"
echo "| Bounded Staleness     | 2x RU (quorum read)     | High          | Lower        |"
echo "| Session               | 1x RU                   | Low           | High         |"
echo "| Consistent Prefix     | 1x RU                   | Low           | High         |"
echo "| Eventual              | 1x RU                   | Lowest        | Highest      |"
echo ""
echo "KEY EXAM INSIGHT: Strong & Bounded Staleness cost 2x RU for reads"
echo "because they require quorum acknowledgment from replicas."
```

### Verification

1. Compare account configurations in the Portal under Settings → Default Consistency
2. Note that Strong consistency limits single-region write accounts to 1 read region
3. Check that Bounded Staleness shows maxStalenessPrefix and maxIntervalInSeconds

### Cleanup

```bash
az group delete --name $RG --yes --no-wait
```

---

## Lab 3: Multi-Region Distribution & Failover

### Objective
Configure Cosmos DB for global distribution with multiple regions, set up automatic failover, enable multi-region writes, and understand conflict resolution policies.

### Key AZ-305 Concepts Practiced
- Multi-region distribution for low-latency global access
- Automatic vs. manual failover and priority configuration
- Multi-region writes (multi-master) for write availability
- Conflict resolution policies (Last Writer Wins, Custom)
- SLA implications: single-region (99.99%) vs. multi-region (99.999%)

### Steps

#### 3.1 Create Multi-Region Cosmos DB Account

```bash
# Variables
RG="rg-cosmosdb-lab03"
ACCOUNT="cosmos-global-$RANDOM"

az group create --name $RG --location eastus

# Create account with 3 regions
az cosmosdb create \
  --name $ACCOUNT \
  --resource-group $RG \
  --default-consistency-level Session \
  --locations regionName=eastus failoverPriority=0 isZoneRedundant=true \
  --locations regionName=westus failoverPriority=1 isZoneRedundant=false \
  --locations regionName=northeurope failoverPriority=2 isZoneRedundant=false \
  --enable-automatic-failover true
```

#### 3.2 View and Modify Failover Priorities

```bash
# View current failover priorities
az cosmosdb show --name $ACCOUNT --resource-group $RG \
  --query "locations[].{region:locationName, priority:failoverPriority, zone:isZoneRedundant}" -o table

# Update failover priorities (swap West US and North Europe)
az cosmosdb failover-priority-change \
  --name $ACCOUNT \
  --resource-group $RG \
  --failover-policies "eastus=0" "northeurope=1" "westus=2"

# Verify new priorities
az cosmosdb show --name $ACCOUNT --resource-group $RG \
  --query "locations[].{region:locationName, priority:failoverPriority}" -o table
```

#### 3.3 Perform Manual Failover

```bash
# Manual failover: promote West US (priority=2) to write region
# This simulates a disaster recovery drill
az cosmosdb failover-priority-change \
  --name $ACCOUNT \
  --resource-group $RG \
  --failover-policies "westus=0" "eastus=1" "northeurope=2"

echo "Manual failover initiated. West US is now the write region."
echo "This operation is zero-data-loss and takes ~1-2 minutes."

# Verify new write region
az cosmosdb show --name $ACCOUNT --resource-group $RG \
  --query "writeLocations[0].locationName" -o tsv
```

#### 3.4 Enable Multi-Region Writes (Multi-Master)

```bash
# Enable multi-region writes
az cosmosdb update \
  --name $ACCOUNT \
  --resource-group $RG \
  --enable-multiple-write-locations true

echo "Multi-region writes enabled. All regions can now accept writes."
echo "This provides 99.999% write availability SLA."
```

#### 3.5 Configure Conflict Resolution Policy

```bash
# Create a database and container with conflict resolution
az cosmosdb sql database create \
  --account-name $ACCOUNT \
  --resource-group $RG \
  --name "GlobalDB"

# Option A: Last Writer Wins (LWW) - default, uses _ts property
az cosmosdb sql container create \
  --account-name $ACCOUNT \
  --resource-group $RG \
  --database-name "GlobalDB" \
  --name "LWWContainer" \
  --partition-key-path "/region" \
  --throughput 400 \
  --conflict-resolution-policy '{"mode":"LastWriterWins","conflictResolutionPath":"/_ts"}'

# Option B: LWW with custom path (e.g., priority field)
az cosmosdb sql container create \
  --account-name $ACCOUNT \
  --resource-group $RG \
  --database-name "GlobalDB" \
  --name "PriorityContainer" \
  --partition-key-path "/region" \
  --throughput 400 \
  --conflict-resolution-policy '{"mode":"LastWriterWins","conflictResolutionPath":"/priority"}'

# Option C: Custom stored procedure (manual conflict resolution)
az cosmosdb sql container create \
  --account-name $ACCOUNT \
  --resource-group $RG \
  --database-name "GlobalDB" \
  --name "CustomContainer" \
  --partition-key-path "/region" \
  --throughput 400 \
  --conflict-resolution-policy '{"mode":"Custom","conflictResolutionProcedure":"dbs/GlobalDB/colls/CustomContainer/sprocs/resolveConflict"}'

echo ""
echo "=== Conflict Resolution Policies ==="
echo "LWW (/_ts): Highest timestamp wins (default)"
echo "LWW (/priority): Highest priority value wins"
echo "Custom: Stored procedure decides winner; unresolved go to conflicts feed"
```

#### 3.6 Add/Remove Regions Dynamically

```bash
# Add a new region (Southeast Asia)
az cosmosdb update \
  --name $ACCOUNT \
  --resource-group $RG \
  --locations regionName=eastus failoverPriority=0 \
  --locations regionName=westus failoverPriority=1 \
  --locations regionName=northeurope failoverPriority=2 \
  --locations regionName=southeastasia failoverPriority=3

# Remove a region (North Europe)
az cosmosdb update \
  --name $ACCOUNT \
  --resource-group $RG \
  --locations regionName=eastus failoverPriority=0 \
  --locations regionName=westus failoverPriority=1 \
  --locations regionName=southeastasia failoverPriority=2
```

### Verification

1. In Portal → Cosmos DB → Replicate Data Globally: verify regions on map
2. Check that automatic failover is enabled
3. Confirm multi-region writes status
4. View conflict resolution policy on each container

### Cleanup

```bash
az group delete --name $RG --yes --no-wait
```

---

## Lab 4: Autoscale vs Provisioned Throughput

### Objective
Compare fixed provisioned throughput with autoscale throughput, understand when to use each, and observe how autoscale responds to varying workload patterns.

### Key AZ-305 Concepts Practiced
- Provisioned throughput: predictable cost, manual scaling
- Autoscale: automatic 1/10 to max RU/s scaling, pay for peak
- Serverless: pay-per-request, no minimum (for dev/test/spiky)
- Cost optimization: choose model based on workload pattern
- Shared throughput (database-level) vs. dedicated (container-level)

### Steps

#### 4.1 Create Cosmos DB Account

```bash
# Variables
RG="rg-cosmosdb-lab04"
LOCATION="eastus"
ACCOUNT="cosmos-scale-$RANDOM"

az group create --name $RG --location $LOCATION

az cosmosdb create \
  --name $ACCOUNT \
  --resource-group $RG \
  --default-consistency-level Session \
  --locations regionName=$LOCATION failoverPriority=0
```

#### 4.2 Create Container with Fixed Provisioned Throughput

```bash
# Create database
az cosmosdb sql database create \
  --account-name $ACCOUNT \
  --resource-group $RG \
  --name "ScaleTestDB"

# Container with FIXED 400 RU/s (manual scaling)
az cosmosdb sql container create \
  --account-name $ACCOUNT \
  --resource-group $RG \
  --database-name "ScaleTestDB" \
  --name "FixedContainer" \
  --partition-key-path "/tenantId" \
  --throughput 400

echo "Fixed throughput container: always consumes 400 RU/s (billed hourly)"
```

#### 4.3 Create Container with Autoscale Throughput

```bash
# Container with AUTOSCALE max 4000 RU/s (scales between 400-4000)
az cosmosdb sql container create \
  --account-name $ACCOUNT \
  --resource-group $RG \
  --database-name "ScaleTestDB" \
  --name "AutoscaleContainer" \
  --partition-key-path "/tenantId" \
  --max-throughput 4000

echo "Autoscale container: scales between 400 (10% of max) and 4000 RU/s"
echo "Billed at highest RU/s reached each hour"
```

#### 4.4 Create Serverless Account (Alternative)

```bash
# Serverless account (separate account - cannot mix with provisioned)
ACCOUNT_SERVERLESS="cosmos-serverless-$RANDOM"

az cosmosdb create \
  --name $ACCOUNT_SERVERLESS \
  --resource-group $RG \
  --default-consistency-level Session \
  --locations regionName=$LOCATION failoverPriority=0 \
  --capabilities EnableServerless

# Serverless container (no throughput specification needed)
az cosmosdb sql database create \
  --account-name $ACCOUNT_SERVERLESS \
  --resource-group $RG \
  --name "ServerlessDB"

az cosmosdb sql container create \
  --account-name $ACCOUNT_SERVERLESS \
  --resource-group $RG \
  --database-name "ServerlessDB" \
  --name "PayPerUseContainer" \
  --partition-key-path "/userId"

echo "Serverless: pay only for RUs consumed per request (no minimum)"
echo "Max burst: 5000 RU/s, Max storage: 1 TB"
```

#### 4.5 View and Modify Throughput

```bash
# View current throughput on fixed container
az cosmosdb sql container throughput show \
  --account-name $ACCOUNT \
  --resource-group $RG \
  --database-name "ScaleTestDB" \
  --name "FixedContainer" \
  --query "{throughput:resource.throughput, isAutoscale:resource.autoscaleSettings}" -o json

# Scale up fixed throughput manually
az cosmosdb sql container throughput update \
  --account-name $ACCOUNT \
  --resource-group $RG \
  --database-name "ScaleTestDB" \
  --name "FixedContainer" \
  --throughput 800

# View autoscale settings
az cosmosdb sql container throughput show \
  --account-name $ACCOUNT \
  --resource-group $RG \
  --database-name "ScaleTestDB" \
  --name "AutoscaleContainer" \
  --query "resource.autoscaleSettings" -o json

# Modify autoscale max RU/s
az cosmosdb sql container throughput update \
  --account-name $ACCOUNT \
  --resource-group $RG \
  --database-name "ScaleTestDB" \
  --name "AutoscaleContainer" \
  --max-throughput 8000

echo "Autoscale now ranges from 800 (10%) to 8000 RU/s"
```

#### 4.6 Migrate Between Throughput Types

```bash
# Migrate fixed → autoscale
az cosmosdb sql container throughput migrate \
  --account-name $ACCOUNT \
  --resource-group $RG \
  --database-name "ScaleTestDB" \
  --name "FixedContainer" \
  --throughput-type autoscale

# Migrate autoscale → fixed (manual)
az cosmosdb sql container throughput migrate \
  --account-name $ACCOUNT \
  --resource-group $RG \
  --database-name "ScaleTestDB" \
  --name "AutoscaleContainer" \
  --throughput-type manual
```

#### 4.7 Cost Comparison

```bash
echo "=== Throughput Model Decision Guide ==="
echo ""
echo "| Model       | Best For                           | Min Cost/mo (~) | Max RU/s    |"
echo "|-------------|------------------------------------|-----------------|-------------|"
echo "| Fixed       | Predictable, steady workloads      | ~\$24 (400 RU)  | Unlimited*  |"
echo "| Autoscale   | Variable with unpredictable spikes | ~\$36 (400 min) | Unlimited*  |"
echo "| Serverless  | Dev/test, sporadic, <5000 RU/s     | \$0 (pay-per-use)| 5000 RU/s   |"
echo ""
echo "* Unlimited = can scale to millions of RU/s (in increments of 100 for fixed)"
echo ""
echo "EXAM TIP: Autoscale minimum is always 10% of max."
echo "If you set max=10000, minimum billing = 1000 RU/s even at idle."
echo ""
echo "EXAM TIP: Serverless limitations:"
echo "  - Single region only (no geo-replication)"
echo "  - Max 1 TB storage"
echo "  - Max 5000 RU/s burst"
echo "  - Cannot use with Synapse Link"
```

### Verification

1. In Portal → Cosmos DB → Scale & Settings for each container
2. Confirm FixedContainer shows manual throughput
3. Confirm AutoscaleContainer shows autoscale with max RU/s
4. Observe the "Throughput" metric in Monitoring to see scaling behavior

### Cleanup

```bash
az group delete --name $RG --yes --no-wait
```

---

## Lab 5: Change Feed & Event-Driven Architecture

### Objective
Implement the Change Feed pattern to react to data changes in real-time using Azure Functions with a Cosmos DB trigger, and build a materialized view pattern.

### Key AZ-305 Concepts Practiced
- Change Feed: ordered stream of inserts and updates (not deletes by default)
- Event-driven architecture with Azure Functions
- Materialized view pattern for read-optimized denormalized data
- Change Feed Processor library for custom consumers
- Lease container for tracking progress

### Steps

#### 5.1 Create Infrastructure

```bash
# Variables
RG="rg-cosmosdb-lab05"
LOCATION="eastus"
ACCOUNT="cosmos-feed-$RANDOM"
FUNC_APP="func-cosmosdb-$RANDOM"
STORAGE="stfunclab05$RANDOM"

az group create --name $RG --location $LOCATION

# Create Cosmos DB account
az cosmosdb create \
  --name $ACCOUNT \
  --resource-group $RG \
  --default-consistency-level Session \
  --locations regionName=$LOCATION failoverPriority=0

# Create database and source container
az cosmosdb sql database create \
  --account-name $ACCOUNT \
  --resource-group $RG \
  --name "ECommerceDB"

# Source container (orders)
az cosmosdb sql container create \
  --account-name $ACCOUNT \
  --resource-group $RG \
  --database-name "ECommerceDB" \
  --name "Orders" \
  --partition-key-path "/customerId" \
  --throughput 400

# Materialized view container (order summaries by product)
az cosmosdb sql container create \
  --account-name $ACCOUNT \
  --resource-group $RG \
  --database-name "ECommerceDB" \
  --name "ProductSummary" \
  --partition-key-path "/productId" \
  --throughput 400

# Lease container (required for Change Feed Processor / Function trigger)
az cosmosdb sql container create \
  --account-name $ACCOUNT \
  --resource-group $RG \
  --database-name "ECommerceDB" \
  --name "leases" \
  --partition-key-path "/id" \
  --throughput 400
```

#### 5.2 Create Azure Function with Cosmos DB Trigger

```bash
# Create storage account for Function App
az storage account create \
  --name $STORAGE \
  --resource-group $RG \
  --location $LOCATION \
  --sku Standard_LRS

# Create Function App
az functionapp create \
  --name $FUNC_APP \
  --resource-group $RG \
  --storage-account $STORAGE \
  --consumption-plan-location $LOCATION \
  --runtime dotnet-isolated \
  --functions-version 4

# Get Cosmos DB connection string
COSMOS_CONN=$(az cosmosdb keys list \
  --name $ACCOUNT \
  --resource-group $RG \
  --type connection-strings \
  --query "connectionStrings[0].connectionString" -o tsv)

# Set connection string as app setting
az functionapp config appsettings set \
  --name $FUNC_APP \
  --resource-group $RG \
  --settings "CosmosDBConnection=$COSMOS_CONN"
```

#### 5.3 Function Code (Cosmos DB Trigger → Materialized View)

```bash
cat << 'EOF'
// Azure Function: Cosmos DB Trigger (C# .NET 8 Isolated Worker)
// File: ProcessOrderChanges.cs

using Microsoft.Azure.Functions.Worker;
using Microsoft.Extensions.Logging;
using System.Text.Json;

namespace CosmosChangeFeed;

public class ProcessOrderChanges
{
    private readonly ILogger _logger;

    public ProcessOrderChanges(ILoggerFactory loggerFactory)
    {
        _logger = loggerFactory.CreateLogger<ProcessOrderChanges>();
    }

    [Function("ProcessOrderChanges")]
    [CosmosDBOutput(
        databaseName: "ECommerceDB",
        containerName: "ProductSummary",
        Connection = "CosmosDBConnection")]
    public object? Run(
        [CosmosDBTrigger(
            databaseName: "ECommerceDB",
            containerName: "Orders",
            Connection = "CosmosDBConnection",
            LeaseContainerName = "leases",
            CreateLeaseContainerIfNotExists = true)]
        IReadOnlyList<Order> orders)
    {
        if (orders == null || orders.Count == 0) return null;

        _logger.LogInformation($"Processing {orders.Count} order changes");

        // Build materialized view: aggregate by product
        var summaries = orders
            .GroupBy(o => o.ProductId)
            .Select(g => new ProductSummary
            {
                Id = $"summary-{g.Key}",
                ProductId = g.Key,
                TotalQuantity = g.Sum(o => o.Quantity),
                TotalRevenue = g.Sum(o => o.Quantity * o.Price),
                LastUpdated = DateTime.UtcNow
            })
            .ToList();

        return summaries;  // Output binding writes to ProductSummary container
    }
}

public record Order(string Id, string CustomerId, string ProductId, int Quantity, decimal Price);
public record ProductSummary
{
    public string Id { get; init; }
    public string ProductId { get; init; }
    public int TotalQuantity { get; init; }
    public decimal TotalRevenue { get; init; }
    public DateTime LastUpdated { get; init; }
}
EOF
```

#### 5.4 Enable Change Feed with All Versions and Deletes (Preview)

```bash
# For capturing deletes, enable "All Versions and Deletes" mode
# This requires the container to have a retention period set
echo "=== Change Feed Modes ==="
echo ""
echo "1. Latest Version (default): Captures inserts and updates only"
echo "   - Most common for materialized views"
echo "   - Documents appear in modification order"
echo ""
echo "2. All Versions and Deletes (preview): Captures inserts, updates, AND deletes"
echo "   - Requires continuous backup mode"
echo "   - Useful for audit trails and sync scenarios"
echo "   - Must be enabled at container creation time"
echo ""
echo "EXAM TIP: Change Feed does NOT capture deletes by default."
echo "Use soft-delete pattern (TTL field) or All Versions mode."
```

#### 5.5 Monitor Change Feed Progress

```bash
# View the lease container to see processing progress
echo "Check the 'leases' container in Data Explorer for:"
echo "  - Continuation tokens (progress checkpoints)"
echo "  - Lease ownership (which function instance owns which partition)"
echo "  - Last processed timestamp"
echo ""
echo "Monitor function execution in:"
echo "  Portal → Function App → Monitor → Invocations"
```

### Verification

1. Insert a document into the Orders container via Data Explorer
2. Check Function App Monitor for trigger execution
3. Verify a corresponding document appears in ProductSummary container
4. Check the leases container for progress tracking documents

### Cleanup

```bash
az group delete --name $RG --yes --no-wait
```

---

## Lab 6: Security & Private Access

### Objective
Implement a defense-in-depth security model for Cosmos DB using Entra ID RBAC, private endpoints, IP firewall, customer-managed keys, and disabling key-based auth.

### Key AZ-305 Concepts Practiced
- Zero Trust: disable keys, use Entra ID RBAC for data plane
- Private endpoints for network isolation (no public internet)
- IP firewall for restricting public access
- Customer-managed keys (CMK) for encryption at rest
- Built-in roles vs custom role definitions

### Steps

#### 6.1 Create Cosmos DB Account

```bash
# Variables
RG="rg-cosmosdb-lab06"
LOCATION="eastus"
ACCOUNT="cosmos-secure-$RANDOM"
VNET="vnet-cosmos"
SUBNET="snet-cosmos-pe"

az group create --name $RG --location $LOCATION

# Create account with IP firewall allowing only Azure Portal and your IP
MY_IP=$(curl -s ifconfig.me)

az cosmosdb create \
  --name $ACCOUNT \
  --resource-group $RG \
  --default-consistency-level Session \
  --locations regionName=$LOCATION failoverPriority=0 \
  --ip-range-filter "$MY_IP,104.42.195.92,40.76.54.131,52.176.6.30,52.169.50.45,52.187.184.26"

# Note: The IP addresses above are Azure Portal IPs (required for Portal access)
echo "Account created with IP firewall. Only your IP and Azure Portal can access."
```

#### 6.2 Configure Entra ID RBAC (Data Plane)

```bash
# Get your user object ID
USER_OBJECT_ID=$(az ad signed-in-user show --query id -o tsv)
ACCOUNT_ID=$(az cosmosdb show --name $ACCOUNT --resource-group $RG --query id -o tsv)

# Built-in data plane roles:
# - Cosmos DB Built-in Data Reader (00000000-0000-0000-0000-000000000001)
# - Cosmos DB Built-in Data Contributor (00000000-0000-0000-0000-000000000002)

# Assign Data Contributor role (read/write data, no management)
az cosmosdb sql role assignment create \
  --account-name $ACCOUNT \
  --resource-group $RG \
  --role-definition-id "00000000-0000-0000-0000-000000000002" \
  --principal-id $USER_OBJECT_ID \
  --scope "/"

echo "Assigned Cosmos DB Built-in Data Contributor role to current user"

# List role assignments
az cosmosdb sql role assignment list \
  --account-name $ACCOUNT \
  --resource-group $RG -o table
```

#### 6.3 Create Custom RBAC Role (Read-Only Specific Container)

```bash
# Create a custom role definition for read-only access to a specific container
cat << 'EOF' > cosmos-custom-role.json
{
  "RoleName": "OrdersReader",
  "Type": "CustomRole",
  "AssignableScopes": ["/dbs/ECommerceDB/colls/Orders"],
  "Permissions": [{
    "DataActions": [
      "Microsoft.DocumentDB/databaseAccounts/readMetadata",
      "Microsoft.DocumentDB/databaseAccounts/sqlDatabases/containers/items/read",
      "Microsoft.DocumentDB/databaseAccounts/sqlDatabases/containers/executeQuery",
      "Microsoft.DocumentDB/databaseAccounts/sqlDatabases/containers/readChangeFeed"
    ]
  }]
}
EOF

az cosmosdb sql role definition create \
  --account-name $ACCOUNT \
  --resource-group $RG \
  --body @cosmos-custom-role.json

rm cosmos-custom-role.json
echo "Custom role 'OrdersReader' created with least-privilege access"
```

#### 6.4 Disable Key-Based Authentication

```bash
# Disable primary/secondary key access (enforce Entra ID only)
az cosmosdb update \
  --name $ACCOUNT \
  --resource-group $RG \
  --disable-key-based-metadata-write-access true

# Fully disable local auth (keys stop working for data plane too)
az resource update \
  --ids $ACCOUNT_ID \
  --set properties.disableLocalAuth=true

echo "Key-based authentication DISABLED."
echo "Only Entra ID RBAC can access this account now."
echo ""
echo "EXAM TIP: disableLocalAuth=true is the Zero Trust best practice."
echo "Connection strings and keys will return 401 Unauthorized."
```

#### 6.5 Set Up Private Endpoint

```bash
# Create VNet and subnet
az network vnet create \
  --name $VNET \
  --resource-group $RG \
  --location $LOCATION \
  --address-prefix 10.0.0.0/16 \
  --subnet-name $SUBNET \
  --subnet-prefix 10.0.1.0/24

# Disable subnet private endpoint network policies
az network vnet subnet update \
  --name $SUBNET \
  --resource-group $RG \
  --vnet-name $VNET \
  --disable-private-endpoint-network-policies true

# Create private endpoint
az network private-endpoint create \
  --name "pe-cosmos" \
  --resource-group $RG \
  --vnet-name $VNET \
  --subnet $SUBNET \
  --private-connection-resource-id $ACCOUNT_ID \
  --group-ids "Sql" \
  --connection-name "cosmos-pe-connection"

# Create private DNS zone
az network private-dns zone create \
  --resource-group $RG \
  --name "privatelink.documents.azure.com"

# Link DNS zone to VNet
az network private-dns zone vnet-link create \
  --resource-group $RG \
  --zone-name "privatelink.documents.azure.com" \
  --name "cosmos-dns-link" \
  --virtual-network $VNET \
  --registration-enabled false

# Create DNS records
PE_NIC_ID=$(az network private-endpoint show \
  --name "pe-cosmos" --resource-group $RG \
  --query "networkInterfaces[0].id" -o tsv)

PE_IP=$(az network nic show --ids $PE_NIC_ID \
  --query "ipConfigurations[0].privateIpAddress" -o tsv)

az network private-dns record-set a add-record \
  --resource-group $RG \
  --zone-name "privatelink.documents.azure.com" \
  --record-set-name $ACCOUNT \
  --ipv4-address $PE_IP

echo "Private endpoint created at IP: $PE_IP"
echo "Cosmos DB now accessible via private IP within the VNet"
```

#### 6.6 Disable Public Network Access

```bash
# After private endpoint is set up, disable public access entirely
az cosmosdb update \
  --name $ACCOUNT \
  --resource-group $RG \
  --public-network-access DISABLED

echo "Public network access DISABLED."
echo "Account only accessible via private endpoint within the VNet."
echo ""
echo "EXAM TIP: Defense-in-depth = Private Endpoint + Disable Public Access"
echo "  + Entra ID RBAC + Disable Local Auth + CMK"
```

#### 6.7 Customer-Managed Keys (CMK) - Informational

```bash
echo "=== Customer-Managed Keys (CMK) ==="
echo ""
echo "CMK must be configured at account CREATION time (cannot add later)."
echo ""
echo "Prerequisites:"
echo "  1. Azure Key Vault with soft-delete and purge protection"
echo "  2. RSA key (2048, 3072, or 4096 bit)"
echo "  3. Cosmos DB managed identity with Key Vault access"
echo ""
echo "Example creation command:"
echo "  az cosmosdb create \\"
echo "    --name \$ACCOUNT \\"
echo "    --resource-group \$RG \\"
echo "    --key-uri 'https://myvault.vault.azure.net/keys/mykey' \\"
echo "    --assign-identity '[system]' \\"
echo "    --default-identity 'SystemAssignedIdentity'"
echo ""
echo "EXAM TIP: CMK = account-level encryption. Always-on encryption"
echo "with Microsoft-managed keys is the default (no action needed)."
```

### Verification

1. Try accessing account with connection string → should get 401 (keys disabled)
2. In Portal → Networking: verify private endpoint and public access disabled
3. In Portal → Data Plane RBAC: verify role assignments
4. DNS resolution: `nslookup <account>.documents.azure.com` should resolve to private IP (from within VNet)

### Cleanup

```bash
az group delete --name $RG --yes --no-wait
```

---

## Lab 7: Cosmos DB Synapse Link (HTAP)

### Objective
Enable Azure Synapse Link for Cosmos DB to run analytics on operational data without impacting transactional performance (Hybrid Transactional/Analytical Processing - HTAP).

### Key AZ-305 Concepts Practiced
- HTAP: separate analytical store (column-oriented) from transactional store (row-oriented)
- Zero-ETL: no data movement pipelines needed
- Synapse serverless SQL and Spark for analytics
- Analytical store auto-sync (~2 min latency)
- No RU consumption on the transactional store for analytical queries

### Steps

#### 7.1 Create Cosmos DB Account with Analytical Store Enabled

```bash
# Variables
RG="rg-cosmosdb-lab07"
LOCATION="eastus"
ACCOUNT="cosmos-synapse-$RANDOM"
SYNAPSE_WS="synapse-cosmos-$RANDOM"
SYNAPSE_STORAGE="stsynapse$RANDOM"

az group create --name $RG --location $LOCATION

# Create Cosmos DB account with Analytical Storage enabled
az cosmosdb create \
  --name $ACCOUNT \
  --resource-group $RG \
  --default-consistency-level Session \
  --locations regionName=$LOCATION failoverPriority=0 \
  --enable-analytical-storage true

echo "Analytical storage enabled at account level"
```

#### 7.2 Create Container with Analytical Store TTL

```bash
# Create database
az cosmosdb sql database create \
  --account-name $ACCOUNT \
  --resource-group $RG \
  --name "SalesDB"

# Create container with analytical TTL enabled (-1 = no expiry)
az cosmosdb sql container create \
  --account-name $ACCOUNT \
  --resource-group $RG \
  --database-name "SalesDB" \
  --name "Transactions" \
  --partition-key-path "/category" \
  --throughput 400 \
  --analytical-storage-ttl -1

echo "Container created with analytical store (TTL=-1, data never expires in analytical store)"
echo ""
echo "Analytical Store TTL options:"
echo "  -1: No expiry (keep all data in analytical store)"
echo "   0: Disabled (no analytical store)"
echo "  >0: Expire after N seconds"
```

#### 7.3 Insert Sample Data

```bash
echo "Insert sample transactions via Data Explorer:"
cat << 'EOF'
[
  {"id":"t1","category":"Electronics","product":"Laptop","amount":1299.99,"timestamp":"2024-01-15T10:00:00Z","region":"East"},
  {"id":"t2","category":"Electronics","product":"Phone","amount":899.99,"timestamp":"2024-01-15T11:00:00Z","region":"West"},
  {"id":"t3","category":"Clothing","product":"Jacket","amount":149.99,"timestamp":"2024-01-15T12:00:00Z","region":"East"},
  {"id":"t4","category":"Electronics","product":"Tablet","amount":599.99,"timestamp":"2024-01-16T09:00:00Z","region":"North"},
  {"id":"t5","category":"Clothing","product":"Shoes","amount":89.99,"timestamp":"2024-01-16T14:00:00Z","region":"West"}
]
EOF
echo ""
echo "Wait ~2 minutes for data to sync to analytical store"
```

#### 7.4 Create Synapse Workspace

```bash
# Create storage for Synapse
az storage account create \
  --name $SYNAPSE_STORAGE \
  --resource-group $RG \
  --location $LOCATION \
  --sku Standard_LRS \
  --kind StorageV2 \
  --hns true

# Create Synapse workspace
az synapse workspace create \
  --name $SYNAPSE_WS \
  --resource-group $RG \
  --location $LOCATION \
  --storage-account $SYNAPSE_STORAGE \
  --file-system "synapsefs" \
  --sql-admin-login-user "sqladmin" \
  --sql-admin-login-password "P@ssw0rd1234!"

# Open firewall for your IP
az synapse workspace firewall-rule create \
  --name "AllowMyIP" \
  --workspace-name $SYNAPSE_WS \
  --resource-group $RG \
  --start-ip-address $MY_IP \
  --end-ip-address $MY_IP

# Allow Azure services
az synapse workspace firewall-rule create \
  --name "AllowAzureServices" \
  --workspace-name $SYNAPSE_WS \
  --resource-group $RG \
  --start-ip-address 0.0.0.0 \
  --end-ip-address 0.0.0.0
```

#### 7.5 Create Linked Service (Connect Synapse to Cosmos DB)

```bash
# Get Cosmos DB account key
COSMOS_KEY=$(az cosmosdb keys list \
  --name $ACCOUNT --resource-group $RG \
  --query primaryReadonlyMasterKey -o tsv)

COSMOS_ENDPOINT=$(az cosmosdb show \
  --name $ACCOUNT --resource-group $RG \
  --query documentEndpoint -o tsv)

echo "In Synapse Studio:"
echo "1. Go to Manage → Linked Services → New"
echo "2. Select 'Azure Cosmos DB (SQL API)'"
echo "3. Enter:"
echo "   - Account endpoint: $COSMOS_ENDPOINT"
echo "   - Database: SalesDB"
echo "   - Use account key or managed identity"
echo ""
echo "Or create via CLI (linked service JSON):"

cat << EOF > cosmos-linked-service.json
{
  "name": "CosmosDbLink",
  "properties": {
    "type": "CosmosDb",
    "typeProperties": {
      "connectionString": "AccountEndpoint=$COSMOS_ENDPOINT;AccountKey=$COSMOS_KEY;Database=SalesDB"
    }
  }
}
EOF

echo "Linked service definition saved. Upload via Synapse Studio or REST API."
rm cosmos-linked-service.json
```

#### 7.6 Query with Synapse Serverless SQL

```bash
cat << 'EOF'
-- Run this in Synapse Studio → SQL Script (serverless SQL pool)
-- OPENROWSET queries the Cosmos DB analytical store directly

SELECT
    category,
    COUNT(*) as transaction_count,
    SUM(amount) as total_revenue,
    AVG(amount) as avg_transaction
FROM OPENROWSET(
    'CosmosDB',
    'Account=<your-account>;Database=SalesDB;Key=<your-key>',
    Transactions
) WITH (
    category VARCHAR(50),
    product VARCHAR(100),
    amount FLOAT,
    region VARCHAR(50),
    timestamp VARCHAR(30)
) AS transactions
GROUP BY category
ORDER BY total_revenue DESC;

-- This query hits the ANALYTICAL STORE (column-format)
-- It does NOT consume RUs from the transactional store!
EOF
```

#### 7.7 Query with Synapse Spark

```bash
cat << 'EOF'
# Run this in Synapse Studio → Notebook (Spark pool)
# PySpark - Read from Cosmos DB analytical store

df = spark.read \
    .format("cosmos.olap") \
    .option("spark.synapse.linkedService", "CosmosDbLink") \
    .option("spark.cosmos.container", "Transactions") \
    .load()

# Show schema (auto-inferred from analytical store)
df.printSchema()

# Aggregate analytics
from pyspark.sql.functions import sum, avg, count

summary = df.groupBy("category", "region") \
    .agg(
        count("*").alias("count"),
        sum("amount").alias("total"),
        avg("amount").alias("average")
    ) \
    .orderBy("total", ascending=False)

summary.show()

# Write results back (e.g., to Data Lake for reporting)
summary.write.mode("overwrite").parquet("abfss://synapsefs@<storage>.dfs.core.windows.net/reports/sales_summary")
EOF
```

### Verification

1. Portal → Cosmos DB → Azure Synapse Link: verify it shows "Enabled"
2. Container settings → Analytical Storage TTL should show -1
3. Synapse Studio → SQL script: OPENROWSET query returns data
4. Cosmos DB Metrics: confirm no RU spike during analytical queries

### Cleanup

```bash
az group delete --name $RG --yes --no-wait
```

---

## Lab 8: Backup & Point-in-Time Restore

### Objective
Configure continuous backup for Cosmos DB and perform a point-in-time restore (PITR) to recover accidentally deleted data.

### Key AZ-305 Concepts Practiced
- Continuous backup (7-day or 30-day retention) vs. periodic backup
- Point-in-time restore to any second within retention window
- Restore creates a NEW account (non-destructive)
- Periodic backup: 4-hour interval, 2 copies, geo-redundant storage
- Backup mode selected at account creation (can migrate periodic → continuous)

### Steps

#### 8.1 Create Account with Continuous Backup

```bash
# Variables
RG="rg-cosmosdb-lab08"
LOCATION="eastus"
ACCOUNT="cosmos-backup-$RANDOM"

az group create --name $RG --location $LOCATION

# Create account with Continuous backup (30-day retention)
az cosmosdb create \
  --name $ACCOUNT \
  --resource-group $RG \
  --default-consistency-level Session \
  --locations regionName=$LOCATION failoverPriority=0 \
  --backup-policy-type Continuous \
  --continuous-tier Continuous30Days

echo "Account created with Continuous 30-day backup"
echo "Point-in-time restore available to any second in last 30 days"
```

#### 8.2 Create Data to Protect

```bash
# Create database and container
az cosmosdb sql database create \
  --account-name $ACCOUNT \
  --resource-group $RG \
  --name "CriticalDB"

az cosmosdb sql container create \
  --account-name $ACCOUNT \
  --resource-group $RG \
  --database-name "CriticalDB" \
  --name "ImportantData" \
  --partition-key-path "/departmentId" \
  --throughput 400

echo "Insert documents via Data Explorer:"
cat << 'EOF'
{"id":"doc-001","departmentId":"engineering","data":"Critical config A","createdAt":"2024-01-15"}
{"id":"doc-002","departmentId":"engineering","data":"Critical config B","createdAt":"2024-01-15"}
{"id":"doc-003","departmentId":"sales","data":"Q1 pipeline data","createdAt":"2024-01-15"}
EOF

echo ""
echo "Record the current UTC time BEFORE deletion:"
echo "RESTORE_TIME=$(date -u +%Y-%m-%dT%H:%M:%SZ)"
echo ""
echo "Wait 5 minutes, then simulate accidental deletion..."
```

#### 8.3 Simulate Data Loss

```bash
echo "=== Simulating accidental data loss ==="
echo ""
echo "1. Note the current time (UTC): $(date -u +%Y-%m-%dT%H:%M:%SZ 2>/dev/null || Get-Date -Format 'yyyy-MM-ddTHH:mm:ssZ')"
echo "2. Wait at least 5 minutes (for restore point availability)"
echo "3. Delete documents from Data Explorer:"
echo "   - Delete doc-001"
echo "   - Delete doc-002"
echo "   - Delete doc-003"
echo ""
echo "4. Or delete the entire container:"
echo "   az cosmosdb sql container delete \\"
echo "     --account-name $ACCOUNT \\"
echo "     --resource-group $RG \\"
echo "     --database-name CriticalDB \\"
echo "     --name ImportantData --yes"
```

#### 8.4 Perform Point-in-Time Restore

```bash
# Set restore time to BEFORE the deletion (use time noted in step 8.3)
RESTORE_TIME="2024-01-15T10:00:00Z"  # Replace with your noted time
RESTORED_ACCOUNT="cosmos-restored-$RANDOM"

# Restore entire account to a new account
az cosmosdb restore \
  --name $RESTORED_ACCOUNT \
  --resource-group $RG \
  --account-name $ACCOUNT \
  --restore-timestamp $RESTORE_TIME \
  --location $LOCATION

echo "Restore initiated. This creates a NEW account: $RESTORED_ACCOUNT"
echo "Restore typically takes 30-90 minutes depending on data size."

# Check restore status
az cosmosdb show --name $RESTORED_ACCOUNT --resource-group $RG \
  --query "{name:name, status:provisioningState, restoreInfo:restoreParameters}" -o json
```

#### 8.5 Restore Specific Database/Container Only

```bash
# Restore only a specific database and container (granular restore)
RESTORED_ACCOUNT2="cosmos-restored2-$RANDOM"

az cosmosdb restore \
  --name $RESTORED_ACCOUNT2 \
  --resource-group $RG \
  --account-name $ACCOUNT \
  --restore-timestamp $RESTORE_TIME \
  --location $LOCATION \
  --databases-to-restore name="CriticalDB" collections="ImportantData"

echo "Granular restore: only CriticalDB/ImportantData will be in the new account"
```

#### 8.6 Check Restorable Resources

```bash
# List available restore timestamps
az cosmosdb restorable-database-account list \
  --location $LOCATION \
  --query "[?name=='$ACCOUNT'].{name:accountName, oldestRestore:oldestRestorableTime}" -o table

# List restorable databases
INSTANCE_ID=$(az cosmosdb restorable-database-account list \
  --location $LOCATION \
  --query "[?accountName=='$ACCOUNT'].id" -o tsv)

az cosmosdb sql restorable-database list \
  --instance-id $INSTANCE_ID \
  --location $LOCATION -o table

# List restorable containers
az cosmosdb sql restorable-container list \
  --instance-id $INSTANCE_ID \
  --location $LOCATION \
  --database-rid "<database-resource-id>" -o table
```

#### 8.7 Migrate from Periodic to Continuous Backup

```bash
# If you have an existing account with periodic backup, migrate:
echo "=== Migrating Periodic → Continuous Backup ==="
echo ""
echo "az cosmosdb update \\"
echo "  --name \$ACCOUNT \\"
echo "  --resource-group \$RG \\"
echo "  --backup-policy-type Continuous \\"
echo "  --continuous-tier Continuous7Days"
echo ""
echo "Notes:"
echo "  - Migration is one-way (cannot go back to periodic)"
echo "  - Takes effect immediately"
echo "  - Choose Continuous7Days (free tier included) or Continuous30Days"
```

#### 8.8 Backup Mode Comparison

```bash
echo "=== Backup Mode Comparison ==="
echo ""
echo "| Feature                | Periodic              | Continuous 7-Day     | Continuous 30-Day    |"
echo "|------------------------|-----------------------|----------------------|----------------------|"
echo "| Retention              | Configurable          | 7 days               | 30 days              |"
echo "| Granularity            | Full backup only      | Any second (PITR)    | Any second (PITR)    |"
echo "| Backup interval        | ≥1 hour (default 4h)  | Continuous           | Continuous           |"
echo "| Restore target         | Support ticket        | Self-service (CLI)   | Self-service (CLI)   |"
echo "| Cost                   | 2 copies free         | Included*            | Additional cost      |"
echo "| Multi-region writes    | Supported             | Supported            | Supported            |"
echo "| Restore creates        | New account           | New account          | New account          |"
echo ""
echo "* Continuous 7-day is included in the Cosmos DB pricing (no extra charge)"
echo ""
echo "EXAM TIP: Continuous backup is required for self-service PITR."
echo "Periodic backup requires a support ticket to restore."
echo "Restore ALWAYS creates a new account (never overwrites existing)."
```

### Verification

1. Portal → Cosmos DB → Backup & Restore: verify "Continuous" mode
2. Check that the restored account appears in the resource group
3. Open restored account → Data Explorer: verify documents exist at restore point
4. Compare restored data with the (now-deleted) data in the original account

### Cleanup

```bash
az group delete --name $RG --yes --no-wait
```

---

## Summary: AZ-305 Cosmos DB Decision Framework

| Decision Point | Options | When to Choose |
|---|---|---|
| **API** | NoSQL, MongoDB, Cassandra, Gremlin, Table | NoSQL for new apps; others for migration compatibility |
| **Consistency** | Strong → Eventual | Strong for financial; Session for most apps; Eventual for counters |
| **Throughput** | Fixed / Autoscale / Serverless | Fixed=steady; Autoscale=variable; Serverless=dev/sporadic |
| **Regions** | Single / Multi-read / Multi-write | Multi-write for 99.999% SLA and global write access |
| **Backup** | Periodic / Continuous 7d / Continuous 30d | Continuous for self-service PITR; Periodic for cost savings |
| **Security** | Keys / Entra RBAC / Private Endpoint | Zero Trust: Entra RBAC + Private Endpoint + Disable Keys |
| **Analytics** | Direct query / Synapse Link | Synapse Link for HTAP (no RU impact on transactional workload) |
| **Partition Key** | High cardinality, even distribution | Choose field used in most WHERE clauses with many distinct values |

---

> **Next Steps:** Practice these labs in sequence. Each builds on concepts tested in AZ-305. Focus on the *why* behind each design decision—the exam tests architectural reasoning, not just CLI syntax.
