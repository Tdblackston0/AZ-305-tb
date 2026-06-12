# Azure Data & Analytics Hands-On Labs

> 📖 **Cheat Sheet:** [Azure Data & Analytics Cheat Sheet](../Azure-Data-Analytics.md)

> **Purpose:** Reinforce AZ-305 data and analytics design concepts through hands-on labs with real Azure CLI commands.
>
> **Prerequisites:** Azure subscription, Azure CLI installed (`az --version`), Owner or Contributor role on the subscription.
>
> **Cost Warning:** Some resources (Synapse Dedicated Pools, Databricks clusters, Event Hubs) incur significant costs. Always run cleanup commands when finished.

## Exam Domain Mapping

- **Primary domain:** Design data storage solutions (20-25%)
- **Secondary domains:** Design infrastructure solutions (30-35%), Design identity, governance, and monitoring solutions (25-30%)

| Lab | Primary Skill Tested |
|-----|---------------------|
| Lab 1: Data Lake Gen2 | Non-relational storage – hierarchical namespace and medallion architecture |
| Lab 2: Synapse Analytics | Data warehousing service selection and dedicated vs. serverless design |
| Lab 3: Azure Data Factory | Data integration pipeline design and orchestration |
| Lab 4: Stream Analytics | Real-time analytics and event processing architecture |
| Lab 5: Azure Databricks | Big data and ML workload platform selection |
| Lab 6: Microsoft Purview | Data governance, cataloging, and lineage design |
| Lab 7: Synapse Link | HTAP architecture and operational analytics without ETL |
| Lab 8: Microsoft Fabric | Unified SaaS analytics lakehouse platform selection |

---

## Table of Contents

0. [Infrastructure as Code Starter Templates (Bicep & Terraform)](#infrastructure-as-code-starter-templates-bicep--terraform)
1. [Lab 1: Data Lake Storage Gen2 with Medallion Architecture](#lab-1-data-lake-storage-gen2-with-medallion-architecture)
2. [Lab 2: Azure Synapse Analytics Workspace](#lab-2-azure-synapse-analytics-workspace)
3. [Lab 3: Azure Data Factory Pipeline](#lab-3-azure-data-factory-pipeline)
4. [Lab 4: Real-Time Analytics with Stream Analytics](#lab-4-real-time-analytics-with-stream-analytics)
5. [Lab 5: Azure Databricks with Delta Lake](#lab-5-azure-databricks-with-delta-lake)
6. [Lab 6: Microsoft Purview Data Governance](#lab-6-microsoft-purview-data-governance)
7. [Lab 7: Synapse Link for Cosmos DB (End-to-End)](#lab-7-synapse-link-for-cosmos-db-end-to-end)
8. [Lab 8: Microsoft Fabric Lakehouse (Overview Lab)](#lab-8-microsoft-fabric-lakehouse-overview-lab)

---

## Infrastructure as Code Starter Templates (Bicep & Terraform)

Use these templates to bootstrap core analytics lab resources before running detailed steps.

```bicep
param location string = resourceGroup().location
param storageAccountName string
param workspaceName string

resource storage 'Microsoft.Storage/storageAccounts@2023-05-01' = {
  name: storageAccountName
  location: location
  sku: {
    name: 'Standard_LRS'
  }
  kind: 'StorageV2'
  properties: {
    isHnsEnabled: true
  }
}

resource adf 'Microsoft.DataFactory/factories@2018-06-01' = {
  name: workspaceName
  location: location
  identity: {
    type: 'SystemAssigned'
  }
}
```

```hcl
resource "azurerm_storage_account" "datalake" {
  name                     = var.storage_account_name
  resource_group_name      = var.resource_group_name
  location                 = var.location
  account_tier             = "Standard"
  account_replication_type = "LRS"
  account_kind             = "StorageV2"
  is_hns_enabled           = true
}

resource "azurerm_data_factory" "lab" {
  name                = var.data_factory_name
  location            = var.location
  resource_group_name = var.resource_group_name
}
```

---

## Lab 1: Data Lake Storage Gen2 with Medallion Architecture

### Objective

Design and implement a Data Lake Storage Gen2 account with hierarchical namespace, configure the medallion architecture (bronze/silver/gold), apply POSIX ACLs for team-level security, and set up lifecycle management policies for cost optimization.

### Key AZ-305 Concepts Practiced

- **Hierarchical Namespace (HNS):** Enables directory-level operations, POSIX ACLs, and Data Lake semantics on Blob storage
- **Medallion Architecture:** Bronze (raw) → Silver (cleansed) → Gold (curated) pattern for data lakes
- **POSIX ACLs vs. Azure RBAC:** Fine-grained file/directory permissions vs. subscription-level role assignments
- **Lifecycle Management:** Automating tier transitions (Hot → Cool → Archive) to optimize storage costs
- **Data Lake design patterns:** Organizing data for analytics workloads

### Steps

#### 1.1 Set Variables

```bash
# Define variables
RG="rg-datalake-lab"
LOCATION="eastus2"
STORAGE_ACCOUNT="stadatalakelab$(openssl rand -hex 4)"
CONTAINER="datalake"

echo "Storage Account: $STORAGE_ACCOUNT"
```

#### 1.2 Create Resource Group

```bash
az group create \
  --name $RG \
  --location $LOCATION
```

#### 1.3 Create Storage Account with HNS Enabled

```bash
# HNS (hierarchical namespace) = Data Lake Storage Gen2
az storage account create \
  --name $STORAGE_ACCOUNT \
  --resource-group $RG \
  --location $LOCATION \
  --sku Standard_LRS \
  --kind StorageV2 \
  --hns true \
  --access-tier Hot \
  --min-tls-version TLS1_2 \
  --allow-blob-public-access false
```

> **AZ-305 Note:** `--hns true` is what distinguishes Gen2 from standard Blob. Once enabled, HNS cannot be disabled. This enables atomic directory operations and POSIX ACLs.

#### 1.4 Create the Data Lake Container

```bash
az storage container create \
  --name $CONTAINER \
  --account-name $STORAGE_ACCOUNT \
  --auth-mode login
```

#### 1.5 Create Medallion Directory Structure

```bash
# Create bronze/silver/gold directories
az storage fs directory create \
  --name "bronze" \
  --file-system $CONTAINER \
  --account-name $STORAGE_ACCOUNT \
  --auth-mode login

az storage fs directory create \
  --name "bronze/raw-sales" \
  --file-system $CONTAINER \
  --account-name $STORAGE_ACCOUNT \
  --auth-mode login

az storage fs directory create \
  --name "silver" \
  --file-system $CONTAINER \
  --account-name $STORAGE_ACCOUNT \
  --auth-mode login

az storage fs directory create \
  --name "silver/cleansed-sales" \
  --file-system $CONTAINER \
  --account-name $STORAGE_ACCOUNT \
  --auth-mode login

az storage fs directory create \
  --name "gold" \
  --file-system $CONTAINER \
  --account-name $STORAGE_ACCOUNT \
  --auth-mode login

az storage fs directory create \
  --name "gold/sales-aggregates" \
  --file-system $CONTAINER \
  --account-name $STORAGE_ACCOUNT \
  --auth-mode login
```

#### 1.6 Upload Raw Data to Bronze Layer

```bash
# Create sample CSV data
cat > sales_raw.csv << 'EOF'
order_id,customer_id,product,quantity,price,order_date
1001,C001,Widget-A,5,12.99,2024-01-15
1002,C002,Widget-B,3,24.99,2024-01-15
1003,C001,Widget-C,1,99.99,2024-01-16
1004,C003,Widget-A,10,12.99,2024-01-16
1005,C002,Widget-D,2,49.99,2024-01-17
EOF

# Upload to bronze layer
az storage fs file upload \
  --source ./sales_raw.csv \
  --path "bronze/raw-sales/2024/01/sales_raw.csv" \
  --file-system $CONTAINER \
  --account-name $STORAGE_ACCOUNT \
  --auth-mode login

# Clean up local file
rm sales_raw.csv
```

#### 1.7 Set Up POSIX ACLs for Team-Level Access

```bash
# Get AAD Object IDs for teams (replace with actual IDs)
# DATA_ENGINEERS_OID="<data-engineers-group-object-id>"
# DATA_SCIENTISTS_OID="<data-scientists-group-object-id>"
# ANALYSTS_OID="<analysts-group-object-id>"

# Example: Grant data engineers RWX on bronze and silver
# az storage fs access set \
#   --acl "group:$DATA_ENGINEERS_OID:rwx" \
#   --path "bronze" \
#   --file-system $CONTAINER \
#   --account-name $STORAGE_ACCOUNT \
#   --auth-mode login

# Example: Grant analysts Read+Execute on gold only
# az storage fs access set \
#   --acl "group:$ANALYSTS_OID:r-x" \
#   --path "gold" \
#   --file-system $CONTAINER \
#   --account-name $STORAGE_ACCOUNT \
#   --auth-mode login

# View current ACLs on bronze directory
az storage fs access show \
  --path "bronze" \
  --file-system $CONTAINER \
  --account-name $STORAGE_ACCOUNT \
  --auth-mode login

# Set recursive ACL (apply to all children)
# az storage fs access set-recursive \
#   --acl "group:$DATA_ENGINEERS_OID:rwx" \
#   --path "bronze" \
#   --file-system $CONTAINER \
#   --account-name $STORAGE_ACCOUNT \
#   --auth-mode login
```

> **AZ-305 Note:** POSIX ACLs work at the file/directory level. Default ACLs are inherited by new child objects. Use `set-recursive` for existing content. RBAC gives broader access; ACLs provide fine-grained control within.

#### 1.8 Configure Lifecycle Management Policies

```bash
# Create lifecycle policy JSON
cat > lifecycle-policy.json << 'EOF'
{
  "rules": [
    {
      "enabled": true,
      "name": "bronze-to-cool",
      "type": "Lifecycle",
      "definition": {
        "actions": {
          "baseBlob": {
            "tierToCool": {
              "daysAfterModificationGreaterThan": 30
            },
            "tierToArchive": {
              "daysAfterModificationGreaterThan": 180
            }
          }
        },
        "filters": {
          "blobTypes": ["blockBlob"],
          "prefixMatch": ["datalake/bronze/"]
        }
      }
    },
    {
      "enabled": true,
      "name": "silver-to-cool",
      "type": "Lifecycle",
      "definition": {
        "actions": {
          "baseBlob": {
            "tierToCool": {
              "daysAfterModificationGreaterThan": 60
            }
          }
        },
        "filters": {
          "blobTypes": ["blockBlob"],
          "prefixMatch": ["datalake/silver/"]
        }
      }
    },
    {
      "enabled": true,
      "name": "gold-hot-retention",
      "type": "Lifecycle",
      "definition": {
        "actions": {
          "baseBlob": {
            "tierToCool": {
              "daysAfterModificationGreaterThan": 90
            }
          }
        },
        "filters": {
          "blobTypes": ["blockBlob"],
          "prefixMatch": ["datalake/gold/"]
        }
      }
    }
  ]
}
EOF

# Apply lifecycle policy
az storage account management-policy create \
  --account-name $STORAGE_ACCOUNT \
  --resource-group $RG \
  --policy @lifecycle-policy.json

rm lifecycle-policy.json
```

### Verification

```bash
# Verify HNS is enabled
az storage account show \
  --name $STORAGE_ACCOUNT \
  --resource-group $RG \
  --query "isHnsEnabled"

# List directories in the data lake
az storage fs directory list \
  --file-system $CONTAINER \
  --account-name $STORAGE_ACCOUNT \
  --auth-mode login \
  --query "[].name" -o tsv

# Verify lifecycle policy
az storage account management-policy show \
  --account-name $STORAGE_ACCOUNT \
  --resource-group $RG \
  --query "policy.rules[].name" -o tsv

# Verify file was uploaded
az storage fs file list \
  --file-system $CONTAINER \
  --path "bronze/raw-sales/2024/01" \
  --account-name $STORAGE_ACCOUNT \
  --auth-mode login \
  --query "[].name" -o tsv
```

### Cleanup

```bash
# Delete the entire resource group and all resources within it
az group delete --name $RG --yes --no-wait
```

---

## Lab 2: Azure Synapse Analytics Workspace

### Objective

Deploy a Synapse Analytics workspace with integrated ADLS Gen2, query data using serverless SQL pools, create a dedicated SQL pool, perform CETAS transformations, and manage pool compute (pause/resume).

### Key AZ-305 Concepts Practiced

- **Synapse unified analytics:** Single workspace for SQL, Spark, and pipeline orchestration
- **Serverless SQL pools:** Pay-per-query model for ad-hoc exploration (no provisioned compute)
- **Dedicated SQL pools:** MPP engine with DWU-based scaling for predictable workloads
- **CETAS pattern:** Transform and persist query results as external tables (ETL via T-SQL)
- **Cost management:** Pause/resume dedicated pools to control costs
- **Data lake integration:** Direct query of Parquet/CSV files in ADLS Gen2

### Steps

#### 2.1 Set Variables

```bash
RG="rg-synapse-lab"
LOCATION="eastus2"
STORAGE_ACCOUNT="stasynlab$(openssl rand -hex 4)"
CONTAINER="synapse"
SYNAPSE_WS="synws-lab-$(openssl rand -hex 4)"
SQL_ADMIN="sqladmin"
SQL_PASSWORD="P@ssw0rd$(openssl rand -hex 4)!"

echo "Synapse Workspace: $SYNAPSE_WS"
echo "Storage Account: $STORAGE_ACCOUNT"
echo "SQL Password: $SQL_PASSWORD"
```

#### 2.2 Create Resource Group and Storage

```bash
az group create --name $RG --location $LOCATION

# Create ADLS Gen2 storage for Synapse
az storage account create \
  --name $STORAGE_ACCOUNT \
  --resource-group $RG \
  --location $LOCATION \
  --sku Standard_LRS \
  --kind StorageV2 \
  --hns true

az storage container create \
  --name $CONTAINER \
  --account-name $STORAGE_ACCOUNT \
  --auth-mode login
```

#### 2.3 Create Synapse Workspace

```bash
az synapse workspace create \
  --name $SYNAPSE_WS \
  --resource-group $RG \
  --location $LOCATION \
  --storage-account $STORAGE_ACCOUNT \
  --file-system $CONTAINER \
  --sql-admin-login-user $SQL_ADMIN \
  --sql-admin-login-password $SQL_PASSWORD

# Open firewall for your client IP (for testing only)
az synapse workspace firewall-rule create \
  --name "AllowAll" \
  --workspace-name $SYNAPSE_WS \
  --resource-group $RG \
  --start-ip-address 0.0.0.0 \
  --end-ip-address 255.255.255.255
```

> **AZ-305 Note:** In production, use private endpoints and managed VNet integration. The open firewall rule here is for lab purposes only.

#### 2.4 Upload Sample Parquet Data

```bash
# Upload a sample CSV that we'll query with serverless SQL
cat > products.csv << 'EOF'
product_id,product_name,category,unit_price,stock_qty
P001,Widget-A,Electronics,12.99,500
P002,Widget-B,Electronics,24.99,300
P003,Widget-C,Furniture,99.99,50
P004,Widget-D,Clothing,49.99,200
P005,Widget-E,Electronics,199.99,100
EOF

az storage fs file upload \
  --source ./products.csv \
  --path "raw/products/products.csv" \
  --file-system $CONTAINER \
  --account-name $STORAGE_ACCOUNT \
  --auth-mode login

rm products.csv
```

#### 2.5 Query with Serverless SQL Pool

Connect to the serverless SQL endpoint (`<workspace>-ondemand.sql.azuresynapse.net`) using Azure Data Studio or SSMS and run:

```sql
-- Query CSV files directly from the data lake (serverless)
SELECT TOP 100 *
FROM OPENROWSET(
    BULK 'https://<storage_account>.dfs.core.windows.net/synapse/raw/products/products.csv',
    FORMAT = 'CSV',
    HEADER_ROW = TRUE,
    PARSER_VERSION = '2.0'
) AS products;

-- Create a database for serverless queries
CREATE DATABASE SalesDB;
GO

USE SalesDB;
GO

-- Create an external data source pointing to the lake
CREATE EXTERNAL DATA SOURCE DataLake
WITH (
    LOCATION = 'https://<storage_account>.dfs.core.windows.net/synapse'
);
GO

-- Query using the external data source
SELECT *
FROM OPENROWSET(
    BULK 'raw/products/products.csv',
    DATA_SOURCE = 'DataLake',
    FORMAT = 'CSV',
    HEADER_ROW = TRUE
) AS products
WHERE category = 'Electronics';
```

> **AZ-305 Note:** Serverless SQL pools use a pay-per-TB-processed model. No infrastructure to manage. Ideal for exploration and ad-hoc queries on data lake files.

#### 2.6 Create Dedicated SQL Pool

```bash
# Create a dedicated SQL pool (DW100c = smallest)
az synapse sql pool create \
  --name "dedicatedpool" \
  --workspace-name $SYNAPSE_WS \
  --resource-group $RG \
  --performance-level "DW100c"
```

> **AZ-305 Note:** Dedicated SQL pools use DWU (Data Warehouse Units) for scaling. DW100c is minimum. Each DWU increase adds more compute nodes. Cost is per-DWU-hour while running.

#### 2.7 CETAS – Transform Data into External Table

```sql
-- Run this in serverless SQL pool context
USE SalesDB;
GO

-- Create external file format for Parquet output
CREATE EXTERNAL FILE FORMAT ParquetFormat
WITH (
    FORMAT_TYPE = PARQUET,
    DATA_COMPRESSION = 'org.apache.hadoop.io.compress.snappy.SnappyCodec'
);
GO

-- CETAS: Create External Table As Select
-- Transforms CSV to Parquet and persists in the lake
CREATE EXTERNAL TABLE dbo.products_parquet
WITH (
    LOCATION = 'curated/products_parquet/',
    DATA_SOURCE = DataLake,
    FILE_FORMAT = ParquetFormat
)
AS
SELECT
    product_id,
    product_name,
    category,
    CAST(unit_price AS DECIMAL(10,2)) AS unit_price,
    CAST(stock_qty AS INT) AS stock_quantity
FROM OPENROWSET(
    BULK 'raw/products/products.csv',
    DATA_SOURCE = 'DataLake',
    FORMAT = 'CSV',
    HEADER_ROW = TRUE
) AS products;
GO
```

> **AZ-305 Note:** CETAS is a powerful ETL pattern — transforms data inline with T-SQL and persists to data lake in optimized formats (Parquet). No separate ETL tool needed for simple transformations.

#### 2.8 Pause and Resume Dedicated Pool

```bash
# Pause dedicated pool (stops billing for compute)
az synapse sql pool pause \
  --name "dedicatedpool" \
  --workspace-name $SYNAPSE_WS \
  --resource-group $RG

# Resume when needed
az synapse sql pool resume \
  --name "dedicatedpool" \
  --workspace-name $SYNAPSE_WS \
  --resource-group $RG
```

> **AZ-305 Note:** Pausing stops compute billing but storage remains. Automate pause/resume via Azure Automation or Logic Apps for non-production workloads.

### Verification

```bash
# Verify workspace exists
az synapse workspace show \
  --name $SYNAPSE_WS \
  --resource-group $RG \
  --query "{name:name, status:provisioningState, endpoint:connectivityEndpoints.sql}" -o table

# Verify dedicated pool status
az synapse sql pool show \
  --name "dedicatedpool" \
  --workspace-name $SYNAPSE_WS \
  --resource-group $RG \
  --query "{name:name, status:status, sku:sku.name}" -o table

# List files created by CETAS
az storage fs file list \
  --file-system $CONTAINER \
  --path "curated/products_parquet" \
  --account-name $STORAGE_ACCOUNT \
  --auth-mode login \
  --query "[].name" -o tsv
```

### Cleanup

```bash
# Pause dedicated pool first to stop billing immediately
az synapse sql pool pause \
  --name "dedicatedpool" \
  --workspace-name $SYNAPSE_WS \
  --resource-group $RG 2>/dev/null

# Delete everything
az group delete --name $RG --yes --no-wait
```

---

## Lab 3: Azure Data Factory Pipeline

### Objective

Create an Azure Data Factory instance, configure linked services, build a copy pipeline with column mappings, add a data flow transformation, set up scheduling triggers, and monitor pipeline executions.

### Key AZ-305 Concepts Practiced

- **Data Factory as orchestration layer:** Managed ETL/ELT service for data movement and transformation
- **Linked Services:** Connection definitions to data stores (abstraction layer)
- **Datasets and Pipelines:** Logical data references and workflow definitions
- **Mapping Data Flows:** Visual, code-free Spark-based transformations
- **Triggers:** Schedule, tumbling window, and event-based pipeline execution
- **Integration Runtime:** Self-hosted IR for on-premises, Azure IR for cloud-to-cloud

### Steps

#### 3.1 Set Variables

```bash
RG="rg-adf-lab"
LOCATION="eastus2"
ADF_NAME="adf-lab-$(openssl rand -hex 4)"
STORAGE_SOURCE="stasource$(openssl rand -hex 4)"
STORAGE_DEST="stadest$(openssl rand -hex 4)"

echo "ADF Name: $ADF_NAME"
echo "Source Storage: $STORAGE_SOURCE"
echo "Dest Storage: $STORAGE_DEST"
```

#### 3.2 Create Resource Group and Storage Accounts

```bash
az group create --name $RG --location $LOCATION

# Source storage (simulates raw data source)
az storage account create \
  --name $STORAGE_SOURCE \
  --resource-group $RG \
  --location $LOCATION \
  --sku Standard_LRS \
  --kind StorageV2

az storage container create \
  --name "source-data" \
  --account-name $STORAGE_SOURCE \
  --auth-mode login

# Destination storage (simulates target)
az storage account create \
  --name $STORAGE_DEST \
  --resource-group $RG \
  --location $LOCATION \
  --sku Standard_LRS \
  --kind StorageV2

az storage container create \
  --name "dest-data" \
  --account-name $STORAGE_DEST \
  --auth-mode login
```

#### 3.3 Upload Sample Source Data

```bash
cat > orders.json << 'EOF'
{"order_id": "ORD001", "customer": "Contoso", "amount": 1500.00, "date": "2024-03-01"}
{"order_id": "ORD002", "customer": "Fabrikam", "amount": 2300.50, "date": "2024-03-01"}
{"order_id": "ORD003", "customer": "Contoso", "amount": 750.25, "date": "2024-03-02"}
{"order_id": "ORD004", "customer": "Northwind", "amount": 4100.00, "date": "2024-03-02"}
{"order_id": "ORD005", "customer": "Fabrikam", "amount": 980.75, "date": "2024-03-03"}
EOF

az storage blob upload \
  --account-name $STORAGE_SOURCE \
  --container-name "source-data" \
  --file ./orders.json \
  --name "orders/2024/03/orders.json" \
  --auth-mode login

rm orders.json
```

#### 3.4 Create Data Factory

```bash
az datafactory create \
  --name $ADF_NAME \
  --resource-group $RG \
  --location $LOCATION
```

#### 3.5 Create Linked Services

```bash
# Get storage account keys
SOURCE_KEY=$(az storage account keys list \
  --account-name $STORAGE_SOURCE \
  --resource-group $RG \
  --query "[0].value" -o tsv)

DEST_KEY=$(az storage account keys list \
  --account-name $STORAGE_DEST \
  --resource-group $RG \
  --query "[0].value" -o tsv)

# Create source linked service
cat > ls-source.json << EOF
{
  "type": "AzureBlobStorage",
  "typeProperties": {
    "connectionString": "DefaultEndpointsProtocol=https;AccountName=$STORAGE_SOURCE;AccountKey=$SOURCE_KEY;EndpointSuffix=core.windows.net"
  }
}
EOF

az datafactory linked-service create \
  --factory-name $ADF_NAME \
  --resource-group $RG \
  --linked-service-name "ls-source-blob" \
  --properties @ls-source.json

# Create destination linked service
cat > ls-dest.json << EOF
{
  "type": "AzureBlobStorage",
  "typeProperties": {
    "connectionString": "DefaultEndpointsProtocol=https;AccountName=$STORAGE_DEST;AccountKey=$DEST_KEY;EndpointSuffix=core.windows.net"
  }
}
EOF

az datafactory linked-service create \
  --factory-name $ADF_NAME \
  --resource-group $RG \
  --linked-service-name "ls-dest-blob" \
  --properties @ls-dest.json

rm ls-source.json ls-dest.json
```

#### 3.6 Create Datasets

```bash
# Source dataset (JSON)
cat > ds-source.json << 'EOF'
{
  "type": "Json",
  "linkedServiceName": {
    "referenceName": "ls-source-blob",
    "type": "LinkedServiceReference"
  },
  "typeProperties": {
    "location": {
      "type": "AzureBlobStorageLocation",
      "container": "source-data",
      "folderPath": "orders/2024/03",
      "fileName": "orders.json"
    }
  }
}
EOF

az datafactory dataset create \
  --factory-name $ADF_NAME \
  --resource-group $RG \
  --dataset-name "ds-source-orders" \
  --properties @ds-source.json

# Destination dataset (JSON)
cat > ds-dest.json << 'EOF'
{
  "type": "Json",
  "linkedServiceName": {
    "referenceName": "ls-dest-blob",
    "type": "LinkedServiceReference"
  },
  "typeProperties": {
    "location": {
      "type": "AzureBlobStorageLocation",
      "container": "dest-data",
      "folderPath": "processed-orders"
    }
  }
}
EOF

az datafactory dataset create \
  --factory-name $ADF_NAME \
  --resource-group $RG \
  --dataset-name "ds-dest-orders" \
  --properties @ds-dest.json

rm ds-source.json ds-dest.json
```

#### 3.7 Create Copy Pipeline

```bash
cat > pipeline.json << 'EOF'
{
  "activities": [
    {
      "name": "CopyOrdersActivity",
      "type": "Copy",
      "inputs": [
        {
          "referenceName": "ds-source-orders",
          "type": "DatasetReference"
        }
      ],
      "outputs": [
        {
          "referenceName": "ds-dest-orders",
          "type": "DatasetReference"
        }
      ],
      "typeProperties": {
        "source": {
          "type": "JsonSource",
          "storeSettings": {
            "type": "AzureBlobStorageReadSettings",
            "recursive": true
          }
        },
        "sink": {
          "type": "JsonSink",
          "storeSettings": {
            "type": "AzureBlobStorageWriteSettings"
          },
          "formatSettings": {
            "type": "JsonWriteSettings"
          }
        }
      }
    }
  ]
}
EOF

az datafactory pipeline create \
  --factory-name $ADF_NAME \
  --resource-group $RG \
  --pipeline-name "pipeline-copy-orders" \
  --pipeline @pipeline.json

rm pipeline.json
```

#### 3.8 Create a Schedule Trigger

```bash
cat > trigger.json << 'EOF'
{
  "type": "ScheduleTrigger",
  "typeProperties": {
    "recurrence": {
      "frequency": "Day",
      "interval": 1,
      "startTime": "2024-03-01T06:00:00Z",
      "timeZone": "UTC"
    }
  },
  "pipelines": [
    {
      "pipelineReference": {
        "referenceName": "pipeline-copy-orders",
        "type": "PipelineReference"
      }
    }
  ]
}
EOF

az datafactory trigger create \
  --factory-name $ADF_NAME \
  --resource-group $RG \
  --trigger-name "daily-copy-trigger" \
  --properties @trigger.json

# Start the trigger
az datafactory trigger start \
  --factory-name $ADF_NAME \
  --resource-group $RG \
  --trigger-name "daily-copy-trigger"

rm trigger.json
```

#### 3.9 Run Pipeline Manually and Monitor

```bash
# Trigger a manual pipeline run
RUN_ID=$(az datafactory pipeline create-run \
  --factory-name $ADF_NAME \
  --resource-group $RG \
  --pipeline-name "pipeline-copy-orders" \
  --query "runId" -o tsv)

echo "Pipeline Run ID: $RUN_ID"

# Monitor the run (wait a moment then check)
sleep 30

az datafactory pipeline-run show \
  --factory-name $ADF_NAME \
  --resource-group $RG \
  --run-id $RUN_ID \
  --query "{status:status, start:runStart, end:runEnd, pipeline:pipelineName}" -o table
```

#### 3.10 Data Flow (Portal Steps)

> **Note:** Data Flows are best created in the ADF Studio UI. Below are the conceptual steps:

1. Open **Azure Data Factory Studio** → Author → Data Flows → New Data Flow
2. Add **Source** → Select `ds-source-orders`
3. Add **Filter** transformation → Filter where `amount > 1000`
4. Add **Derived Column** → Add `order_category` = `iif(amount > 3000, 'High', 'Medium')`
5. Add **Aggregate** → Group by `customer`, Sum `amount` as `total_amount`
6. Add **Sink** → Select `ds-dest-orders` with a new folder path
7. Add this data flow to a pipeline and run it

> **AZ-305 Note:** Data Flows run on Spark clusters (managed by ADF). They have warm-up time (~5 min). For simple copies, use Copy Activity. For transformations, use Data Flows or Synapse.

### Verification

```bash
# List pipeline runs
az datafactory pipeline-run query-by-factory \
  --factory-name $ADF_NAME \
  --resource-group $RG \
  --last-updated-after "2024-01-01T00:00:00Z" \
  --last-updated-before "2025-12-31T00:00:00Z" \
  --query "value[].{Pipeline:pipelineName, Status:status, Start:runStart}" -o table

# Verify data was copied to destination
az storage blob list \
  --account-name $STORAGE_DEST \
  --container-name "dest-data" \
  --auth-mode login \
  --query "[].name" -o tsv
```

### Cleanup

```bash
# Stop the trigger first
az datafactory trigger stop \
  --factory-name $ADF_NAME \
  --resource-group $RG \
  --trigger-name "daily-copy-trigger" 2>/dev/null

# Delete resource group
az group delete --name $RG --yes --no-wait
```

---

## Lab 4: Real-Time Analytics with Stream Analytics

### Objective

Build a real-time data pipeline using Event Hubs for ingestion, Stream Analytics for processing with windowed aggregations, and output results to Blob storage and/or SQL Database.

### Key AZ-305 Concepts Practiced

- **Event Hubs:** High-throughput event ingestion (millions of events/sec), partitioned for parallelism
- **Stream Analytics:** Real-time query engine with SQL-like syntax for streaming data
- **Windowing functions:** Tumbling, hopping, sliding, session windows for time-based aggregations
- **Lambda architecture:** Hot path (real-time) + cold path (batch) for complete analytics
- **Throughput Units vs. Processing Units:** Scaling mechanisms for Event Hubs and Stream Analytics
- **At-least-once delivery:** Understanding delivery guarantees in streaming

### Steps

#### 4.1 Set Variables

```bash
RG="rg-streaming-lab"
LOCATION="eastus2"
EH_NAMESPACE="ehns-lab-$(openssl rand -hex 4)"
EH_NAME="telemetry-events"
SA_JOB="asa-lab-$(openssl rand -hex 4)"
STORAGE_ACCOUNT="stastream$(openssl rand -hex 4)"

echo "Event Hub Namespace: $EH_NAMESPACE"
echo "Stream Analytics Job: $SA_JOB"
echo "Storage Account: $STORAGE_ACCOUNT"
```

#### 4.2 Create Resource Group and Storage

```bash
az group create --name $RG --location $LOCATION

# Output storage for Stream Analytics
az storage account create \
  --name $STORAGE_ACCOUNT \
  --resource-group $RG \
  --location $LOCATION \
  --sku Standard_LRS \
  --kind StorageV2

az storage container create \
  --name "stream-output" \
  --account-name $STORAGE_ACCOUNT \
  --auth-mode login
```

#### 4.3 Create Event Hub

```bash
# Create namespace (Standard tier for consumer groups and partitions)
az eventhubs namespace create \
  --name $EH_NAMESPACE \
  --resource-group $RG \
  --location $LOCATION \
  --sku Standard \
  --capacity 1

# Create event hub with 4 partitions
az eventhubs eventhub create \
  --name $EH_NAME \
  --namespace-name $EH_NAMESPACE \
  --resource-group $RG \
  --partition-count 4 \
  --message-retention 1

# Create a consumer group for Stream Analytics
az eventhubs eventhub consumer-group create \
  --name "asa-consumer-group" \
  --eventhub-name $EH_NAME \
  --namespace-name $EH_NAMESPACE \
  --resource-group $RG

# Get the connection string
EH_CONN=$(az eventhubs namespace authorization-rule keys list \
  --name "RootManageSharedAccessKey" \
  --namespace-name $EH_NAMESPACE \
  --resource-group $RG \
  --query "primaryConnectionString" -o tsv)

echo "Event Hub Connection: $EH_CONN"
```

> **AZ-305 Note:** Partitions enable parallel reads. Consumer groups allow multiple readers. Standard tier supports up to 32 partitions and 20 consumer groups.

#### 4.4 Create Stream Analytics Job

```bash
az stream-analytics job create \
  --job-name $SA_JOB \
  --resource-group $RG \
  --location $LOCATION \
  --output-error-policy "Drop" \
  --events-outoforder-policy "Adjust" \
  --events-outoforder-max-delay 5 \
  --events-late-arrival-max-delay 10 \
  --compatibility-level "1.2"
```

#### 4.5 Configure Input (Event Hub)

```bash
cat > input.json << EOF
{
  "type": "Stream",
  "datasource": {
    "type": "Microsoft.EventHub/EventHub",
    "properties": {
      "serviceBusNamespace": "$EH_NAMESPACE",
      "sharedAccessPolicyName": "RootManageSharedAccessKey",
      "sharedAccessPolicyKey": "$(az eventhubs namespace authorization-rule keys list --name RootManageSharedAccessKey --namespace-name $EH_NAMESPACE --resource-group $RG --query primaryKey -o tsv)",
      "eventHubName": "$EH_NAME",
      "consumerGroupName": "asa-consumer-group"
    }
  },
  "serialization": {
    "type": "Json",
    "properties": {
      "encoding": "UTF8"
    }
  }
}
EOF

az stream-analytics input create \
  --job-name $SA_JOB \
  --resource-group $RG \
  --input-name "eventhub-input" \
  --properties @input.json

rm input.json
```

#### 4.6 Configure Output (Blob Storage)

```bash
STORAGE_KEY=$(az storage account keys list \
  --account-name $STORAGE_ACCOUNT \
  --resource-group $RG \
  --query "[0].value" -o tsv)

cat > output.json << EOF
{
  "datasource": {
    "type": "Microsoft.Storage/Blob",
    "properties": {
      "storageAccounts": [
        {
          "accountName": "$STORAGE_ACCOUNT",
          "accountKey": "$STORAGE_KEY"
        }
      ],
      "container": "stream-output",
      "pathPattern": "results/{date}/{time}",
      "dateFormat": "yyyy-MM-dd",
      "timeFormat": "HH"
    }
  },
  "serialization": {
    "type": "Json",
    "properties": {
      "encoding": "UTF8",
      "format": "LineSeparated"
    }
  }
}
EOF

az stream-analytics output create \
  --job-name $SA_JOB \
  --resource-group $RG \
  --output-name "blob-output" \
  --properties @output.json

rm output.json
```

#### 4.7 Write Stream Analytics Query (Tumbling Window)

```bash
# The query aggregates events in 60-second tumbling windows
cat > query.json << 'EOF'
{
  "streamingUnits": 3,
  "query": "SELECT\n    System.Timestamp() AS WindowEnd,\n    deviceId,\n    COUNT(*) AS EventCount,\n    AVG(temperature) AS AvgTemperature,\n    MAX(temperature) AS MaxTemperature,\n    MIN(temperature) AS MinTemperature\nINTO\n    [blob-output]\nFROM\n    [eventhub-input]\nGROUP BY\n    deviceId,\n    TumblingWindow(second, 60)"
}
EOF

az stream-analytics transformation create \
  --job-name $SA_JOB \
  --resource-group $RG \
  --transformation-name "main-query" \
  --properties @query.json

rm query.json
```

> **AZ-305 Note:** Tumbling windows are fixed-size, non-overlapping time intervals. Hopping windows overlap. Sliding windows fire when events enter/leave. Session windows group by activity with timeout gaps.

#### 4.8 Start the Stream Analytics Job

```bash
az stream-analytics job start \
  --job-name $SA_JOB \
  --resource-group $RG \
  --output-start-mode "Now"
```

#### 4.9 Send Test Events to Event Hub

```bash
# Install the Azure Event Hubs Python package (or use any SDK)
# Using Azure CLI extension for simplicity:

# Send test events using curl and the Event Hub REST API
# First, generate a SAS token (or use the Python/Node SDK)

# Alternative: Use Azure CLI with the eventhubs extension
# pip install azure-eventhub

cat > send_events.py << 'EOF'
import json
import random
import time
from azure.eventhub import EventHubProducerClient, EventData

# Replace with your connection string
CONNECTION_STR = "YOUR_EH_CONNECTION_STRING"
EVENTHUB_NAME = "telemetry-events"

producer = EventHubProducerClient.from_connection_string(
    conn_str=CONNECTION_STR,
    eventhub_name=EVENTHUB_NAME
)

devices = ["device-001", "device-002", "device-003"]

with producer:
    for i in range(50):
        event_data_batch = producer.create_batch()
        for device in devices:
            event = {
                "deviceId": device,
                "temperature": round(random.uniform(20.0, 45.0), 2),
                "humidity": round(random.uniform(30.0, 80.0), 2),
                "timestamp": time.strftime("%Y-%m-%dT%H:%M:%SZ", time.gmtime())
            }
            event_data_batch.add(EventData(json.dumps(event)))
        producer.send_batch(event_data_batch)
        print(f"Sent batch {i+1}/50")
        time.sleep(1)

print("Done sending events!")
EOF

echo "To send events, install azure-eventhub and run:"
echo "  pip install azure-eventhub"
echo "  Update CONNECTION_STR in send_events.py with: $EH_CONN"
echo "  python send_events.py"
```

#### 4.10 Alternative: Send Events via Azure CLI

```bash
# Quick test with a single event using REST API
# (Requires the event hub connection string)

# Or use the az eventhubs eventhub send command if available in preview:
# az eventhubs eventhub send \
#   --namespace-name $EH_NAMESPACE \
#   --name $EH_NAME \
#   --resource-group $RG \
#   --body '{"deviceId":"device-001","temperature":32.5,"humidity":65.2}'
```

### Verification

```bash
# Check Stream Analytics job status
az stream-analytics job show \
  --job-name $SA_JOB \
  --resource-group $RG \
  --query "{name:name, status:jobState, created:createdDate}" -o table

# After sending events and waiting ~2 minutes, check output
az storage blob list \
  --account-name $STORAGE_ACCOUNT \
  --container-name "stream-output" \
  --auth-mode login \
  --query "[].{Name:name, Size:properties.contentLength}" -o table

# Download and inspect a result file
az storage blob download \
  --account-name $STORAGE_ACCOUNT \
  --container-name "stream-output" \
  --name "results/<date>/<time>" \
  --file ./output-sample.json \
  --auth-mode login 2>/dev/null && cat ./output-sample.json
```

### Cleanup

```bash
# Stop Stream Analytics job first
az stream-analytics job stop \
  --job-name $SA_JOB \
  --resource-group $RG 2>/dev/null

# Delete resource group
az group delete --name $RG --yes --no-wait

# Clean up local files
rm -f send_events.py output-sample.json
```

---

## Lab 5: Azure Databricks with Delta Lake

### Objective

Deploy a Databricks workspace, create a cluster, mount ADLS Gen2 storage, work with Delta Lake tables (including MERGE/UPSERT and time travel), and understand Unity Catalog basics.

### Key AZ-305 Concepts Practiced

- **Databricks workspace:** Managed Spark environment for big data processing and ML
- **Delta Lake:** ACID transactions on data lakes, schema enforcement, time travel
- **MERGE/UPSERT:** Handling incremental data loads with deduplication
- **Time travel:** Querying previous versions of data for auditing and rollback
- **Unity Catalog:** Centralized governance for data and AI assets across workspaces
- **Cluster sizing:** Balancing cost and performance with autoscaling and spot instances

### Steps

#### 5.1 Set Variables

```bash
RG="rg-databricks-lab"
LOCATION="eastus2"
DBX_WORKSPACE="dbx-lab-$(openssl rand -hex 4)"
STORAGE_ACCOUNT="stadbxlab$(openssl rand -hex 4)"
CONTAINER="dbxdata"

echo "Databricks Workspace: $DBX_WORKSPACE"
echo "Storage Account: $STORAGE_ACCOUNT"
```

#### 5.2 Create Resource Group and Storage

```bash
az group create --name $RG --location $LOCATION

# Create ADLS Gen2 for Databricks to mount
az storage account create \
  --name $STORAGE_ACCOUNT \
  --resource-group $RG \
  --location $LOCATION \
  --sku Standard_LRS \
  --kind StorageV2 \
  --hns true

az storage container create \
  --name $CONTAINER \
  --account-name $STORAGE_ACCOUNT \
  --auth-mode login

# Upload sample data
cat > employees.csv << 'EOF'
emp_id,name,department,salary,hire_date
E001,Alice Johnson,Engineering,95000,2021-03-15
E002,Bob Smith,Marketing,72000,2020-06-01
E003,Carol Williams,Engineering,105000,2019-11-20
E004,David Brown,Sales,68000,2022-01-10
E005,Eve Davis,Engineering,88000,2023-02-28
EOF

az storage fs file upload \
  --source ./employees.csv \
  --path "raw/employees/employees.csv" \
  --file-system $CONTAINER \
  --account-name $STORAGE_ACCOUNT \
  --auth-mode login

rm employees.csv
```

#### 5.3 Create Databricks Workspace

```bash
az databricks workspace create \
  --name $DBX_WORKSPACE \
  --resource-group $RG \
  --location $LOCATION \
  --sku standard

# Get workspace URL
DBX_URL=$(az databricks workspace show \
  --name $DBX_WORKSPACE \
  --resource-group $RG \
  --query "workspaceUrl" -o tsv)

echo "Databricks URL: https://$DBX_URL"
```

#### 5.4 Create a Cluster (Portal/UI Steps)

> **Note:** Cluster creation is typically done via the Databricks UI or REST API. Below are the concepts:

1. Navigate to `https://<workspace-url>` → Compute → Create Cluster
2. Configure:
   - **Cluster Name:** `lab-cluster`
   - **Cluster Mode:** Single Node (for lab) or Standard
   - **Databricks Runtime:** Latest LTS (e.g., 14.x LTS)
   - **Node Type:** `Standard_DS3_v2` (smallest general-purpose)
   - **Autoscaling:** Min 1, Max 2 workers
   - **Auto-termination:** 30 minutes of inactivity
3. Click **Create Cluster**

```bash
# Alternative: Use Databricks CLI (if configured)
# databricks clusters create --json '{
#   "cluster_name": "lab-cluster",
#   "spark_version": "14.3.x-scala2.12",
#   "node_type_id": "Standard_DS3_v2",
#   "num_workers": 0,
#   "autotermination_minutes": 30,
#   "spark_conf": {
#     "spark.databricks.cluster.profile": "singleNode"
#   }
# }'
```

> **AZ-305 Note:** Use Standard_DS3_v2 or Standard_F4 for cost-effective labs. Enable auto-termination to prevent runaway costs. In production, use autoscaling with spot instances for 60-90% savings.

#### 5.5 Mount ADLS Gen2 Storage (Notebook Code)

Create a new notebook in Databricks and run these cells:

```python
# Cell 1: Mount ADLS Gen2 using account key (for lab purposes)
# In production, use service principal or Unity Catalog external locations

storage_account = "<your_storage_account>"
container = "dbxdata"
storage_key = "<your_storage_key>"  # Get from portal or CLI

configs = {
    f"fs.azure.account.key.{storage_account}.dfs.core.windows.net": storage_key
}

dbutils.fs.mount(
    source=f"abfss://{container}@{storage_account}.dfs.core.windows.net/",
    mount_point="/mnt/datalake",
    extra_configs=configs
)

# Verify mount
display(dbutils.fs.ls("/mnt/datalake/raw/employees/"))
```

#### 5.6 Create Delta Table

```python
# Cell 2: Read CSV and create Delta table
df = spark.read.format("csv") \
    .option("header", "true") \
    .option("inferSchema", "true") \
    .load("/mnt/datalake/raw/employees/employees.csv")

# Write as Delta table
df.write.format("delta") \
    .mode("overwrite") \
    .save("/mnt/datalake/delta/employees")

# Register as a table
spark.sql("CREATE DATABASE IF NOT EXISTS hr_db")
spark.sql("""
    CREATE TABLE IF NOT EXISTS hr_db.employees
    USING DELTA
    LOCATION '/mnt/datalake/delta/employees'
""")

# Query the table
display(spark.sql("SELECT * FROM hr_db.employees"))
```

#### 5.7 MERGE / UPSERT Operation

```python
# Cell 3: Simulate incoming updates (new hires + salary changes)
from pyspark.sql import Row

updates = spark.createDataFrame([
    Row(emp_id="E002", name="Bob Smith", department="Marketing", salary=78000, hire_date="2020-06-01"),       # Salary update
    Row(emp_id="E006", name="Frank Miller", department="Sales", salary=71000, hire_date="2024-01-15"),        # New hire
    Row(emp_id="E007", name="Grace Lee", department="Engineering", salary=92000, hire_date="2024-02-01"),     # New hire
])

# Create temp view for MERGE
updates.createOrReplaceTempView("employee_updates")

# MERGE (UPSERT) - Update existing, Insert new
spark.sql("""
    MERGE INTO hr_db.employees AS target
    USING employee_updates AS source
    ON target.emp_id = source.emp_id
    WHEN MATCHED THEN
        UPDATE SET
            target.salary = source.salary,
            target.department = source.department
    WHEN NOT MATCHED THEN
        INSERT (emp_id, name, department, salary, hire_date)
        VALUES (source.emp_id, source.name, source.department, source.salary, source.hire_date)
""")

# Verify results
display(spark.sql("SELECT * FROM hr_db.employees ORDER BY emp_id"))
```

> **AZ-305 Note:** MERGE is critical for incremental loading patterns (CDC). Delta Lake handles this atomically with ACID guarantees, unlike raw Parquet files.

#### 5.8 Time Travel Queries

```python
# Cell 4: View table history
display(spark.sql("DESCRIBE HISTORY hr_db.employees"))

# Query previous version (before the merge)
display(spark.sql("SELECT * FROM hr_db.employees VERSION AS OF 0"))

# Query by timestamp
# display(spark.sql("SELECT * FROM hr_db.employees TIMESTAMP AS OF '2024-03-01T10:00:00'"))

# Compare versions
print("=== Version 0 (Original) ===")
v0 = spark.sql("SELECT COUNT(*) as cnt, SUM(salary) as total FROM hr_db.employees VERSION AS OF 0")
display(v0)

print("=== Current Version (After MERGE) ===")
current = spark.sql("SELECT COUNT(*) as cnt, SUM(salary) as total FROM hr_db.employees")
display(current)

# Restore to previous version if needed
# spark.sql("RESTORE TABLE hr_db.employees TO VERSION AS OF 0")
```

> **AZ-305 Note:** Time travel enables auditing, debugging, and rollback. Retention is configurable (default 30 days). Use VACUUM to clean up old files and reduce storage costs.

#### 5.9 Unity Catalog Basics (Conceptual)

> **Note:** Unity Catalog requires a Premium workspace and account-level admin setup. Below are the key concepts:

```python
# Unity Catalog hierarchy:
# Metastore (account-level)
#   └── Catalog (logical grouping)
#       └── Schema (database)
#           └── Table / View / Function

# Example: Create catalog and schema (requires Premium + UC configured)
# spark.sql("CREATE CATALOG IF NOT EXISTS analytics")
# spark.sql("CREATE SCHEMA IF NOT EXISTS analytics.production")
# spark.sql("CREATE TABLE analytics.production.employees AS SELECT * FROM hr_db.employees")

# Grant permissions (centralized governance)
# spark.sql("GRANT SELECT ON TABLE analytics.production.employees TO `data-analysts`")
# spark.sql("GRANT USAGE ON SCHEMA analytics.production TO `data-analysts`")
```

**Unity Catalog Key Points for AZ-305:**
- Centralized metadata and access control across workspaces
- Three-level namespace: `catalog.schema.table`
- Fine-grained permissions (GRANT/REVOKE)
- Data lineage tracking built-in
- External locations for governed access to cloud storage
- Works with Delta Sharing for cross-organization data sharing

### Verification

```bash
# Verify workspace exists and is running
az databricks workspace show \
  --name $DBX_WORKSPACE \
  --resource-group $RG \
  --query "{name:name, sku:sku.name, url:workspaceUrl, state:provisioningState}" -o table

# Verify Delta files exist in storage
az storage fs file list \
  --file-system $CONTAINER \
  --path "delta/employees" \
  --account-name $STORAGE_ACCOUNT \
  --auth-mode login \
  --query "[].name" -o tsv
```

### Cleanup

```bash
# Terminate cluster via UI or Databricks CLI first
# databricks clusters delete --cluster-id <cluster-id>

# Unmount in notebook:
# dbutils.fs.unmount("/mnt/datalake")

# Delete resource group
az group delete --name $RG --yes --no-wait
```

---

## Lab 6: Microsoft Purview Data Governance

### Objective

Deploy a Microsoft Purview account, register data sources, run discovery scans, apply classifications and sensitivity labels, and explore data lineage capabilities.

### Key AZ-305 Concepts Practiced

- **Unified data governance:** Single pane for data discovery, classification, and lineage
- **Data Map:** Automated scanning and metadata discovery across hybrid data estate
- **Classifications:** Auto-detect sensitive data (PII, financial, healthcare)
- **Sensitivity labels:** Integration with Microsoft Information Protection
- **Data lineage:** Visual tracking of data flow from source to destination
- **Collection hierarchy:** Organizing assets for access control and governance

### Steps

#### 6.1 Set Variables

```bash
RG="rg-purview-lab"
LOCATION="eastus2"
PURVIEW_ACCOUNT="purview-lab-$(openssl rand -hex 4)"
STORAGE_ACCOUNT="stapurview$(openssl rand -hex 4)"
SQL_SERVER="sql-purview-lab-$(openssl rand -hex 4)"
SQL_DB="SalesDB"
SQL_ADMIN="sqladmin"
SQL_PASSWORD="P@ssw0rd$(openssl rand -hex 4)!"

echo "Purview Account: $PURVIEW_ACCOUNT"
echo "SQL Server: $SQL_SERVER"
echo "SQL Password: $SQL_PASSWORD"
```

#### 6.2 Create Resource Group and Data Sources

```bash
az group create --name $RG --location $LOCATION

# Create ADLS Gen2 (data source to scan)
az storage account create \
  --name $STORAGE_ACCOUNT \
  --resource-group $RG \
  --location $LOCATION \
  --sku Standard_LRS \
  --kind StorageV2 \
  --hns true

az storage container create \
  --name "customer-data" \
  --account-name $STORAGE_ACCOUNT \
  --auth-mode login

# Upload sample data with PII
cat > customers.csv << 'EOF'
customer_id,name,email,phone,ssn,credit_card,address
C001,John Smith,john.smith@contoso.com,555-0101,123-45-6789,4111-1111-1111-1111,123 Main St Seattle WA
C002,Jane Doe,jane.doe@fabrikam.com,555-0102,987-65-4321,5500-0000-0000-0004,456 Oak Ave Portland OR
C003,Bob Johnson,bob.j@northwind.com,555-0103,456-78-9012,3400-0000-0000-009,789 Pine Rd Denver CO
EOF

az storage fs file upload \
  --source ./customers.csv \
  --path "raw/customers/customers.csv" \
  --file-system "customer-data" \
  --account-name $STORAGE_ACCOUNT \
  --auth-mode login

rm customers.csv

# Create Azure SQL Database (another data source to scan)
az sql server create \
  --name $SQL_SERVER \
  --resource-group $RG \
  --location $LOCATION \
  --admin-user $SQL_ADMIN \
  --admin-password $SQL_PASSWORD

az sql server firewall-rule create \
  --name "AllowAzure" \
  --server $SQL_SERVER \
  --resource-group $RG \
  --start-ip-address 0.0.0.0 \
  --end-ip-address 0.0.0.0

az sql db create \
  --name $SQL_DB \
  --server $SQL_SERVER \
  --resource-group $RG \
  --service-objective S0
```

#### 6.3 Create Microsoft Purview Account

```bash
# Note: The CLI extension for Purview may vary. Use 'az purview' or portal.
az purview account create \
  --account-name $PURVIEW_ACCOUNT \
  --resource-group $RG \
  --location $LOCATION

# Get the Purview account endpoint
PURVIEW_ENDPOINT=$(az purview account show \
  --account-name $PURVIEW_ACCOUNT \
  --resource-group $RG \
  --query "endpoints.catalog" -o tsv)

echo "Purview Portal: https://purview.microsoft.com"
echo "Purview Endpoint: $PURVIEW_ENDPOINT"
```

> **AZ-305 Note:** Microsoft Purview is now accessed through the unified governance portal at `https://purview.microsoft.com`. The managed resource group contains the underlying infrastructure.

#### 6.4 Grant Purview Access to Data Sources

```bash
# Get Purview managed identity
PURVIEW_MI=$(az purview account show \
  --account-name $PURVIEW_ACCOUNT \
  --resource-group $RG \
  --query "identity.principalId" -o tsv)

# Grant Storage Blob Data Reader on ADLS Gen2
STORAGE_ID=$(az storage account show \
  --name $STORAGE_ACCOUNT \
  --resource-group $RG \
  --query "id" -o tsv)

az role assignment create \
  --assignee-object-id $PURVIEW_MI \
  --assignee-principal-type ServicePrincipal \
  --role "Storage Blob Data Reader" \
  --scope $STORAGE_ID
```

#### 6.5 Register Data Sources (Portal Steps)

> **Note:** Source registration and scanning is done in the Purview governance portal.

1. Navigate to **https://purview.microsoft.com** → Data Map → Sources
2. **Register ADLS Gen2:**
   - Click "Register" → Select "Azure Data Lake Storage Gen2"
   - Select your subscription and storage account
   - Name: `adls-customer-data`
   - Click "Register"
3. **Register Azure SQL Database:**
   - Click "Register" → Select "Azure SQL Database"
   - Select your subscription, server, and database
   - Name: `sql-salesdb`
   - Click "Register"

#### 6.6 Run a Scan

1. In the Data Map, select the registered ADLS Gen2 source
2. Click "New Scan" → Configure:
   - **Scan name:** `scan-customer-data`
   - **Integration runtime:** Azure AutoResolve
   - **Credential:** Purview MSI (system-assigned managed identity)
   - **Scope:** Select the `customer-data` container
3. Select a scan rule set (use "System default" for built-in classifications)
4. Set schedule: Once (or recurring)
5. Click "Save and Run"

> **AZ-305 Note:** Scans discover assets and auto-classify data. System classifications include: SSN, Credit Card, Email, Phone Number, IP Address, and 200+ built-in patterns.

#### 6.7 Review Classifications

After the scan completes (~5-10 minutes):

1. Go to **Data Catalog** → Browse assets
2. Find `customers.csv` in the asset list
3. Review auto-detected classifications:
   - `ssn` column → Government ID / Social Security Number
   - `credit_card` column → Financial / Credit Card Number
   - `email` column → Contact Info / Email Address
   - `phone` column → Contact Info / Phone Number
4. You can manually add or remove classifications

#### 6.8 Apply Sensitivity Labels

1. Go to **Data Map** → Classifications → Sensitivity labels
2. Map classifications to Microsoft Information Protection labels:
   - SSN → "Highly Confidential"
   - Credit Card → "Confidential"
   - Email → "General"
3. Labels propagate to Microsoft 365 compliance center

> **AZ-305 Note:** Sensitivity labels require Microsoft 365 E5 or E5 Compliance add-on. They enable consistent protection across Azure, Microsoft 365, and on-premises.

#### 6.9 Explore Data Lineage

1. If you have Data Factory pipelines (Lab 3), lineage is captured automatically
2. Navigate to **Data Catalog** → Search for an asset → Click "Lineage" tab
3. View the visual flow: Source → Transformation → Destination
4. Lineage is captured from: ADF, Synapse pipelines, Databricks (with connector), Power BI

> **AZ-305 Note:** Lineage helps answer "Where did this data come from?" and "What downstream reports are affected if I change this source?" Critical for impact analysis and compliance.

### Verification

```bash
# Verify Purview account
az purview account show \
  --account-name $PURVIEW_ACCOUNT \
  --resource-group $RG \
  --query "{name:name, status:provisioningState, endpoint:endpoints.catalog}" -o table

# Verify role assignment for scanning
az role assignment list \
  --scope $STORAGE_ID \
  --assignee $PURVIEW_MI \
  --query "[].{Role:roleDefinitionName, Principal:principalId}" -o table
```

### Cleanup

```bash
az group delete --name $RG --yes --no-wait
```

---

## Lab 7: Synapse Link for Cosmos DB (End-to-End)

### Objective

Create a Cosmos DB account with analytical store enabled, establish a Synapse Link connection, and query operational data in near real-time using both serverless SQL and Spark pools — without impacting transactional workloads.

### Key AZ-305 Concepts Practiced

- **Synapse Link:** No-ETL bridge between operational and analytical stores (HTAP)
- **Analytical store:** Column-store auto-synced from Cosmos DB row-store (no RU impact)
- **HTAP pattern:** Hybrid Transactional/Analytical Processing without data movement
- **Serverless SQL on Cosmos:** Query Cosmos DB data with T-SQL (no ingestion needed)
- **Cost optimization:** Analytical store avoids expensive Change Feed + ETL pipelines
- **TTL for analytical store:** Independent retention from transactional store

### Steps

#### 7.1 Set Variables

```bash
RG="rg-synapselink-lab"
LOCATION="eastus2"
COSMOS_ACCOUNT="cosmos-lab-$(openssl rand -hex 4)"
COSMOS_DB="salesdb"
COSMOS_CONTAINER="orders"
SYNAPSE_WS="synlink-lab-$(openssl rand -hex 4)"
STORAGE_ACCOUNT="stasynlink$(openssl rand -hex 4)"
SQL_ADMIN="sqladmin"
SQL_PASSWORD="P@ssw0rd$(openssl rand -hex 4)!"

echo "Cosmos DB: $COSMOS_ACCOUNT"
echo "Synapse: $SYNAPSE_WS"
```

#### 7.2 Create Resource Group

```bash
az group create --name $RG --location $LOCATION
```

#### 7.3 Create Cosmos DB with Analytical Store

```bash
# Create Cosmos DB account with analytical storage enabled
az cosmosdb create \
  --name $COSMOS_ACCOUNT \
  --resource-group $RG \
  --location $LOCATION \
  --kind GlobalDocumentDB \
  --enable-analytical-storage true \
  --default-consistency-level Session

# Create database
az cosmosdb sql database create \
  --account-name $COSMOS_ACCOUNT \
  --resource-group $RG \
  --name $COSMOS_DB

# Create container with analytical store TTL (-1 = infinite)
az cosmosdb sql container create \
  --account-name $COSMOS_ACCOUNT \
  --resource-group $RG \
  --database-name $COSMOS_DB \
  --name $COSMOS_CONTAINER \
  --partition-key-path "/customerId" \
  --throughput 400 \
  --analytical-storage-ttl -1
```

> **AZ-305 Note:** `--analytical-storage-ttl -1` enables the analytical store with infinite retention. Set to a specific value (in seconds) to auto-expire analytical data independently from transactional TTL.

#### 7.4 Insert Sample Data

```bash
# Get Cosmos DB connection key
COSMOS_KEY=$(az cosmosdb keys list \
  --name $COSMOS_ACCOUNT \
  --resource-group $RG \
  --query "primaryMasterKey" -o tsv)

# Insert sample orders using the REST API or SDK
# Using Azure CLI data plane commands:

for i in $(seq 1 10); do
  az cosmosdb sql container invoke-stored-procedure \
    --account-name $COSMOS_ACCOUNT \
    --resource-group $RG \
    --database-name $COSMOS_DB \
    --container-name $COSMOS_CONTAINER \
    --stored-procedure-id "__placeholder__" 2>/dev/null || true
done

# Alternative: Insert via the portal Data Explorer or SDK
# Below is a Python script approach:
cat > insert_orders.py << 'EOF'
from azure.cosmos import CosmosClient
import random
import uuid
from datetime import datetime, timedelta

# Replace with your values
ENDPOINT = "https://<cosmos_account>.documents.azure.com:443/"
KEY = "<cosmos_key>"

client = CosmosClient(ENDPOINT, KEY)
database = client.get_database_client("salesdb")
container = database.get_container_client("orders")

customers = ["C001", "C002", "C003", "C004", "C005"]
products = ["Widget-A", "Widget-B", "Widget-C", "Gadget-X", "Gadget-Y"]
regions = ["East", "West", "Central", "South"]

for i in range(100):
    order = {
        "id": str(uuid.uuid4()),
        "orderId": f"ORD-{10000 + i}",
        "customerId": random.choice(customers),
        "product": random.choice(products),
        "quantity": random.randint(1, 20),
        "unitPrice": round(random.uniform(10.0, 200.0), 2),
        "region": random.choice(regions),
        "orderDate": (datetime.now() - timedelta(days=random.randint(0, 30))).isoformat(),
        "status": random.choice(["Completed", "Pending", "Shipped"])
    }
    order["totalAmount"] = round(order["quantity"] * order["unitPrice"], 2)
    container.create_item(body=order)
    
print("Inserted 100 orders into Cosmos DB")
EOF

echo "To insert data, install azure-cosmos and run:"
echo "  pip install azure-cosmos"
echo "  Update ENDPOINT and KEY in insert_orders.py"
echo "  python insert_orders.py"
```

#### 7.5 Create Synapse Workspace

```bash
# Create storage for Synapse
az storage account create \
  --name $STORAGE_ACCOUNT \
  --resource-group $RG \
  --location $LOCATION \
  --sku Standard_LRS \
  --kind StorageV2 \
  --hns true

az storage container create \
  --name "synapse" \
  --account-name $STORAGE_ACCOUNT \
  --auth-mode login

# Create Synapse workspace
az synapse workspace create \
  --name $SYNAPSE_WS \
  --resource-group $RG \
  --location $LOCATION \
  --storage-account $STORAGE_ACCOUNT \
  --file-system "synapse" \
  --sql-admin-login-user $SQL_ADMIN \
  --sql-admin-login-password $SQL_PASSWORD

# Open firewall
az synapse workspace firewall-rule create \
  --name "AllowAll" \
  --workspace-name $SYNAPSE_WS \
  --resource-group $RG \
  --start-ip-address 0.0.0.0 \
  --end-ip-address 255.255.255.255
```

#### 7.6 Set Up Synapse Link Connection (Portal Steps)

1. Open **Synapse Studio** → Manage → Linked services
2. Click "+ New" → Select "Azure Cosmos DB (SQL API)"
3. Configure:
   - **Name:** `cosmos-link`
   - **Account:** Select your Cosmos DB account
   - **Database:** `salesdb`
4. Test connection and create

> **Note:** Synapse Link sync starts automatically. Data appears in the analytical store within 2-5 minutes.

#### 7.7 Query with Serverless SQL Pool

```sql
-- Connect to Synapse serverless SQL endpoint
-- Query Cosmos DB analytical store directly

SELECT TOP 20 *
FROM OPENROWSET(
    'CosmosDB',
    'Account=<cosmos_account>;Database=salesdb;Key=<cosmos_key>',
    orders
) AS orders;

-- Aggregation query (runs on analytical store, no RU impact)
SELECT
    region,
    COUNT(*) AS order_count,
    SUM(totalAmount) AS total_revenue,
    AVG(totalAmount) AS avg_order_value
FROM OPENROWSET(
    'CosmosDB',
    'Account=<cosmos_account>;Database=salesdb;Key=<cosmos_key>',
    orders
) AS orders
GROUP BY region
ORDER BY total_revenue DESC;

-- Create a view for easy reporting
CREATE OR ALTER VIEW vw_OrderSummary AS
SELECT
    customerId,
    product,
    region,
    status,
    CAST(orderDate AS DATE) AS order_date,
    quantity,
    unitPrice,
    totalAmount
FROM OPENROWSET(
    'CosmosDB',
    'Account=<cosmos_account>;Database=salesdb;Key=<cosmos_key>',
    orders
) WITH (
    customerId VARCHAR(50),
    product VARCHAR(100),
    region VARCHAR(50),
    status VARCHAR(20),
    orderDate VARCHAR(30),
    quantity INT,
    unitPrice FLOAT,
    totalAmount FLOAT
) AS orders;
GO

-- Query the reporting view
SELECT * FROM vw_OrderSummary WHERE status = 'Completed';
```

> **AZ-305 Note:** Queries against the analytical store consume zero RU/s from your Cosmos DB. The analytical store is a separate column-store optimized for analytics. Sync latency is typically 2-5 minutes.

#### 7.8 Query with Spark Pool

```python
# In a Synapse Spark notebook:

# Read from Cosmos DB analytical store via Synapse Link
df = spark.read \
    .format("cosmos.olap") \
    .option("spark.synapse.linkedService", "cosmos-link") \
    .option("spark.cosmos.container", "orders") \
    .load()

# Show schema and sample data
df.printSchema()
display(df.limit(10))

# Run analytics
from pyspark.sql.functions import col, sum, avg, count

summary = df.groupBy("region", "status") \
    .agg(
        count("*").alias("order_count"),
        sum("totalAmount").alias("total_revenue"),
        avg("totalAmount").alias("avg_order_value")
    ) \
    .orderBy(col("total_revenue").desc())

display(summary)
```

#### 7.9 Build a Reporting View

```sql
-- Create a database for reporting
CREATE DATABASE CosmosReporting;
GO

USE CosmosReporting;
GO

-- Create a credential (for Cosmos DB connection)
CREATE CREDENTIAL [CosmosDBKey]
WITH IDENTITY = 'SHARED ACCESS SIGNATURE',
SECRET = '<cosmos_key>';
GO

-- Create a daily summary view
CREATE OR ALTER VIEW dbo.DailySalesSummary AS
SELECT
    CAST(orderDate AS DATE) AS sale_date,
    region,
    product,
    COUNT(*) AS order_count,
    SUM(quantity) AS total_units,
    SUM(totalAmount) AS daily_revenue
FROM OPENROWSET(
    'CosmosDB',
    'Account=<cosmos_account>;Database=salesdb;Key=<cosmos_key>',
    orders
) WITH (
    orderDate VARCHAR(30),
    region VARCHAR(50),
    product VARCHAR(100),
    quantity INT,
    totalAmount FLOAT
) AS orders
GROUP BY CAST(orderDate AS DATE), region, product;
GO
```

### Verification

```bash
# Verify Cosmos DB analytical store is enabled
az cosmosdb sql container show \
  --account-name $COSMOS_ACCOUNT \
  --resource-group $RG \
  --database-name $COSMOS_DB \
  --name $COSMOS_CONTAINER \
  --query "{name:name, analyticalTtl:resource.analyticalStorageTtl, partitionKey:resource.partitionKey.paths[0]}" -o table

# Verify Synapse workspace
az synapse workspace show \
  --name $SYNAPSE_WS \
  --resource-group $RG \
  --query "{name:name, state:provisioningState}" -o table
```

### Cleanup

```bash
az group delete --name $RG --yes --no-wait
rm -f insert_orders.py
```

---

## Lab 8: Microsoft Fabric Lakehouse (Overview Lab)

### Objective

Explore Microsoft Fabric's Lakehouse architecture — create a workspace, build a Lakehouse, upload data to OneLake, transform with notebooks, and query via the SQL endpoint.

### Key AZ-305 Concepts Practiced

- **Microsoft Fabric:** Unified SaaS analytics platform (successor to separate services)
- **OneLake:** Single data lake for the entire organization (one copy of data)
- **Lakehouse:** Combines data lake flexibility with data warehouse structure
- **SQL endpoint:** Auto-generated T-SQL layer over Delta tables in the Lakehouse
- **Medallion in Fabric:** Bronze/Silver/Gold implemented natively
- **Capacity model:** CU-based billing (F2 minimum for trial)

> ⚠️ **Prerequisites:** This lab requires a Microsoft Fabric trial or capacity (F2+). Sign up at [https://app.fabric.microsoft.com](https://app.fabric.microsoft.com).

### Steps

#### 8.1 Enable Fabric Trial (Portal)

1. Navigate to [https://app.fabric.microsoft.com](https://app.fabric.microsoft.com)
2. Click your profile icon → "Start trial" (if available)
3. A Fabric trial provides F64 capacity for 60 days
4. Alternatively, ask your admin to assign a Fabric capacity (F2+)

> **AZ-305 Note:** Fabric uses Capacity Units (CUs). F2 = 2 CUs (dev/test). F64 = 64 CUs (trial). CUs are shared across all Fabric workloads (Data Engineering, Data Warehouse, Real-Time Analytics, etc.)

#### 8.2 Create a Fabric Workspace

1. In Fabric portal → Click "Workspaces" → "+ New workspace"
2. Configure:
   - **Name:** `AZ305-DataLab`
   - **License mode:** Trial or Fabric capacity
   - **Description:** "AZ-305 Data & Analytics Lab"
3. Click "Apply"

> **AZ-305 Note:** Workspaces in Fabric are the security and collaboration boundary. They map to Power BI workspaces and hold all Fabric items (Lakehouses, Warehouses, Pipelines, etc.)

#### 8.3 Create a Lakehouse

1. In the workspace → Click "+ New" → Select "Lakehouse"
2. **Name:** `SalesLakehouse`
3. Click "Create"

The Lakehouse automatically provisions:
- **Files section:** Unstructured/semi-structured data (like a traditional data lake)
- **Tables section:** Managed Delta tables (queryable via SQL endpoint)
- **SQL endpoint:** Auto-generated T-SQL views over Delta tables

#### 8.4 Upload Data to OneLake

**Option A: Via UI**
1. In the Lakehouse → Files → Click "Upload" → "Upload files"
2. Upload CSV/Parquet files to the Files section

**Option B: Via OneLake Explorer (Windows)**
1. Install [OneLake File Explorer](https://www.microsoft.com/store/productId/9P3ZQVUV1F1Q)
2. OneLake mounts as a drive in Windows Explorer
3. Drag and drop files to: `OneLake/<workspace>/<lakehouse>/Files/`

**Option C: Via Azure Storage APIs**
```bash
# OneLake supports ADLS Gen2 APIs
# Endpoint: https://onelake.dfs.fabric.microsoft.com/<workspace-id>/<lakehouse-id>

# Upload using az storage (requires Fabric identity auth)
# az storage fs file upload \
#   --source ./data.csv \
#   --path "Files/raw/sales.csv" \
#   --file-system "<lakehouse-id>" \
#   --account-name "onelake" \
#   --auth-mode login
```

#### 8.5 Create Sample Data (Upload This CSV)

Create a file called `sales_2024.csv`:

```csv
transaction_id,date,store_id,product_id,product_name,category,quantity,unit_price,total_amount,payment_method
T001,2024-01-15,S01,P001,Laptop Pro,Electronics,1,1299.99,1299.99,Credit Card
T002,2024-01-15,S02,P002,Wireless Mouse,Electronics,3,29.99,89.97,Debit Card
T003,2024-01-16,S01,P003,Office Chair,Furniture,2,449.99,899.98,Credit Card
T004,2024-01-16,S03,P004,USB-C Hub,Electronics,5,59.99,299.95,Cash
T005,2024-01-17,S02,P005,Standing Desk,Furniture,1,799.99,799.99,Credit Card
T006,2024-01-17,S01,P001,Laptop Pro,Electronics,2,1299.99,2599.98,Credit Card
T007,2024-01-18,S03,P002,Wireless Mouse,Electronics,10,29.99,299.90,Debit Card
T008,2024-01-18,S02,P006,Monitor 27",Electronics,1,599.99,599.99,Credit Card
T009,2024-01-19,S01,P003,Office Chair,Furniture,1,449.99,449.99,Cash
T010,2024-01-19,S03,P007,Keyboard,Electronics,4,79.99,319.96,Debit Card
```

Upload to: `Files/raw/sales_2024.csv`

#### 8.6 Create a Notebook for Transformation

1. In the workspace → "+ New" → "Notebook"
2. Attach to the Lakehouse: Click "Add" → Select `SalesLakehouse`
3. Write PySpark code:

```python
# Cell 1: Read raw data from Files section
df_raw = spark.read.format("csv") \
    .option("header", "true") \
    .option("inferSchema", "true") \
    .load("Files/raw/sales_2024.csv")

display(df_raw)
print(f"Raw records: {df_raw.count()}")
```

```python
# Cell 2: Transform - cleanse and enrich (Silver layer)
from pyspark.sql.functions import col, to_date, upper, when

df_silver = df_raw \
    .withColumn("date", to_date(col("date"), "yyyy-MM-dd")) \
    .withColumn("category", upper(col("category"))) \
    .withColumn("price_tier", 
        when(col("unit_price") > 500, "Premium")
        .when(col("unit_price") > 100, "Standard")
        .otherwise("Budget")
    ) \
    .dropDuplicates(["transaction_id"])

display(df_silver)
```

```python
# Cell 3: Write to Tables section as managed Delta table
df_silver.write.format("delta") \
    .mode("overwrite") \
    .saveAsTable("sales_silver")

print("Silver table created successfully!")
```

```python
# Cell 4: Create Gold aggregation table
df_gold = spark.sql("""
    SELECT 
        date,
        category,
        store_id,
        COUNT(*) AS transaction_count,
        SUM(quantity) AS total_units,
        SUM(total_amount) AS total_revenue,
        AVG(total_amount) AS avg_transaction_value
    FROM sales_silver
    GROUP BY date, category, store_id
    ORDER BY date, total_revenue DESC
""")

df_gold.write.format("delta") \
    .mode("overwrite") \
    .saveAsTable("sales_gold_daily")

display(df_gold)
```

#### 8.7 Query via SQL Endpoint

1. In the Lakehouse → Click "SQL endpoint" (top-right switch)
2. You'll see auto-generated tables: `sales_silver`, `sales_gold_daily`
3. Run T-SQL queries:

```sql
-- Query the gold table
SELECT 
    category,
    SUM(total_revenue) AS revenue,
    SUM(total_units) AS units_sold,
    COUNT(*) AS num_transactions
FROM dbo.sales_gold_daily
GROUP BY category
ORDER BY revenue DESC;

-- Create a reporting view
CREATE VIEW dbo.vw_StorePerformance AS
SELECT
    store_id,
    SUM(total_revenue) AS total_revenue,
    SUM(transaction_count) AS total_transactions,
    AVG(avg_transaction_value) AS avg_transaction
FROM dbo.sales_gold_daily
GROUP BY store_id;

-- Query the view
SELECT * FROM dbo.vw_StorePerformance ORDER BY total_revenue DESC;
```

> **AZ-305 Note:** The SQL endpoint provides read-only T-SQL access to Lakehouse Delta tables. It's auto-generated — no infrastructure to provision. Ideal for BI tools (Power BI, Excel) connecting to the Lakehouse.

#### 8.8 Connect Power BI (Optional)

1. In the SQL endpoint → Click "New report" or "Build a report"
2. Select tables/views for the semantic model
3. Create visuals: Revenue by category, Store performance, Daily trends
4. The semantic model is auto-synced with the Lakehouse data

### Key Architecture Comparison

| Feature | Traditional | Fabric Lakehouse |
|---------|------------|-----------------|
| Storage | Separate ADLS Gen2 | OneLake (unified) |
| Compute | Separate Synapse/Databricks | Shared capacity (CUs) |
| SQL Access | Dedicated/Serverless pool | Auto SQL endpoint |
| Governance | Purview (separate) | Built-in |
| BI Integration | Manual connection | Native Power BI |

### Verification

- ✅ Workspace created with Fabric capacity assigned
- ✅ Lakehouse visible with Files and Tables sections
- ✅ Raw CSV uploaded to Files/raw/
- ✅ Notebook runs successfully, creating Delta tables
- ✅ `sales_silver` and `sales_gold_daily` appear in Tables
- ✅ SQL endpoint queries return data
- ✅ (Optional) Power BI report renders from Lakehouse data

### Cleanup

1. **Delete the Lakehouse:** Workspace → Right-click `SalesLakehouse` → Delete
2. **Delete the Workspace:** Settings → Remove workspace
3. **End Fabric Trial:** Profile → Manage trial → End trial (if desired)

> **Note:** Fabric trial resources are automatically cleaned up when the trial expires. OneLake data is deleted with the workspace.

---

## Summary: AZ-305 Data & Analytics Decision Matrix

| Scenario | Recommended Service | Key Reason |
|----------|-------------------|------------|
| Ad-hoc lake exploration | Synapse Serverless SQL | Pay-per-query, no infra |
| Predictable DW workload | Synapse Dedicated Pool | MPP, pause/resume |
| Complex ETL orchestration | Data Factory / Synapse Pipelines | 90+ connectors, visual |
| Real-time stream processing | Stream Analytics / Event Hubs | SQL-like, windowing |
| Big data + ML + Delta Lake | Databricks | Spark, ACID, Unity Catalog |
| Data governance + lineage | Microsoft Purview | Scan, classify, lineage |
| HTAP (operational analytics) | Synapse Link + Cosmos DB | No ETL, no RU impact |
| Unified SaaS analytics | Microsoft Fabric | OneLake, all-in-one |
| Cost-optimized storage | ADLS Gen2 + Lifecycle | Tier automation |

---

## Exam Tips

1. **Serverless vs. Dedicated:** Serverless = unpredictable/ad-hoc. Dedicated = predictable/high concurrency.
2. **Synapse Link:** Eliminates ETL for Cosmos DB analytics. Always prefer when analytics on operational data is needed.
3. **Event Hubs vs. IoT Hub:** Event Hubs = general telemetry. IoT Hub = device management + telemetry.
4. **Stream Analytics vs. Databricks Streaming:** ASA = simpler SQL queries. Databricks = complex ML on streams.
5. **Fabric vs. Synapse:** Fabric is the strategic direction. Synapse remains for IaaS-level control.
6. **Delta Lake:** Understand ACID, time travel, MERGE/UPSERT for any data lake question.
7. **Purview:** Governance + compliance. Automatic classification. Cross-estate lineage.
8. **Lifecycle policies:** Always mention for cost optimization on data lake questions.

---

> 📖 **Cheat Sheet:** [Azure Data & Analytics Cheat Sheet](../Azure-Data-Analytics.md)

---

## Exam-Style Review Questions

1. A data engineering team needs to run ad-hoc exploratory SQL queries against files stored in Azure Data Lake Gen2 without provisioning any dedicated compute or paying for idle resources. Which service is most appropriate?

   **A)** Synapse Dedicated SQL Pool  
   **B)** Synapse Serverless SQL Pool  
   **C)** Azure Databricks SQL Warehouse  
   **D)** Azure SQL Database  
   > **Answer: B.** Synapse Serverless SQL Pool charges per TB of data scanned with no infrastructure to manage. It is ideal for ad-hoc exploration of data lake files without committed compute.

2. A team runs a predictable daily batch ETL job that requires high concurrency and must complete within 2 hours each night. Which analytics service is most appropriate?

   **A)** Synapse Serverless SQL Pool  
   **B)** Azure Data Factory with an Azure SQL Database sink  
   **C)** Synapse Dedicated SQL Pool  
   **D)** Microsoft Fabric Lakehouse  
   > **Answer: C.** Synapse Dedicated SQL Pool provides MPP compute for high-concurrency, predictable workloads. It can be paused when not in use, making the cost manageable for nightly batch jobs.

3. A streaming application ingests IoT telemetry from Event Hubs and must apply tumbling window aggregations over 5-minute intervals using SQL syntax without writing custom code. Which service should be used?

   **A)** Azure Databricks Structured Streaming  
   **B)** Azure Data Factory with copy activity  
   **C)** Azure Stream Analytics  
   **D)** Synapse Serverless SQL Pool  
   > **Answer: C.** Azure Stream Analytics is purpose-built for real-time stream processing with SQL-like syntax and native windowing functions (tumbling, hopping, sliding). It connects directly to Event Hubs without custom code.

4. A data governance team needs to automatically scan, classify sensitive data (PII, financial), and build a cross-estate data lineage map across Azure SQL, Data Lake, and on-premises SQL Server. Which service should they use?

   **A)** Microsoft Defender for Cloud  
   **B)** Azure Monitor with Log Analytics  
   **C)** Microsoft Purview  
   **D)** Azure Policy with classification initiatives  
   > **Answer: C.** Microsoft Purview provides automated scanning, sensitivity classification, business glossary, and end-to-end data lineage across on-premises and cloud data sources.

5. A company wants to adopt a unified SaaS analytics platform that combines a data lake, data warehouse, data engineering, and native Power BI integration in a single managed service with OneLake as the shared storage. Which service should they evaluate?

   **A)** Azure Synapse Analytics  
   **B)** Azure Databricks + ADLS Gen2  
   **C)** Microsoft Fabric  
   **D)** Azure HDInsight  
   > **Answer: C.** Microsoft Fabric is the strategic SaaS analytics platform that unifies all workloads (lakehouse, warehouse, pipelines, real-time intelligence, Power BI) on a single OneLake storage foundation.
