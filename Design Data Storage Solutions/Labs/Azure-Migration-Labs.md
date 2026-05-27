# Azure Migration Hands-On Labs (AZ-305)

> 📖 **Cheat Sheet:** [Azure Migration Strategies](../Design Data Storage Solutions/Azure-Migration-Strategies.md)

> **Purpose:** These labs cover every major Azure migration path tested on AZ-305. Each lab includes both **Azure CLI** and **PowerShell** commands, verification steps, and cleanup. Labs 1–11 are hands-on deployment labs; Lab 12 is a decision-making drill.

---

## Table of Contents

| Lab | Title | Method | Downtime |
|-----|-------|--------|----------|
| [Lab 1](#lab-1-azure-migrate--discovery--assessment) | Azure Migrate – Discovery & Assessment | Assessment | N/A |
| [Lab 2](#lab-2-sql-server--azure-sql-database-dms-online) | SQL Server → Azure SQL DB (DMS Online) | DMS Online | Near-zero |
| [Lab 3](#lab-3-sql-server--azure-sql-database-bacpac-exportimport) | SQL Server → Azure SQL DB (BACPAC) | BACPAC Export/Import | Hours |
| [Lab 4](#lab-4-sql-server--managed-instance-log-replay-service) | SQL Server → MI (Log Replay Service) | LRS | Near-zero |
| [Lab 5](#lab-5-bulk-data-transfer-azcopy-data-box-planning) | Bulk Data Transfer (AzCopy, Data Box) | AzCopy / Data Box | Varies |
| [Lab 6](#lab-6-file-server--azure-files-file-sync) | File Server → Azure Files (File Sync) | Azure File Sync | Near-zero |
| [Lab 7](#lab-7-postgresql--azure-database-for-postgresql) | PostgreSQL → Azure DB for PostgreSQL | pg_dump / DMS | Varies |
| [Lab 8](#lab-8-mysql--azure-database-for-mysql) | MySQL → Azure DB for MySQL | mysqldump / DMS | Varies |
| [Lab 9](#lab-9-mongodb--cosmos-db-mongodb-api) | MongoDB → Cosmos DB (MongoDB API) | mongodump/restore | Varies |
| [Lab 10](#lab-10-oracle--azure-sql-ssma-overview) | Oracle → Azure SQL (SSMA) | SSMA | Hours |
| [Lab 11](#lab-11-data-warehouse--synapse-adf--polybase) | Data Warehouse → Synapse (ADF + PolyBase) | ADF / PolyBase | Hours |
| [Lab 12](#lab-12-cross-platform-migration-decision-exercise) | Migration Decision Exercise | Decision Drill | N/A |

---

## Infrastructure as Code Starter Templates (Bicep & Terraform)

Use these templates to provision reusable migration lab landing zones (resource group, storage, and migration project prerequisites).

```bicep
param location string = resourceGroup().location
param storageAccountName string

resource storage 'Microsoft.Storage/storageAccounts@2023-05-01' = {
  name: storageAccountName
  location: location
  sku: {
    name: 'Standard_LRS'
  }
  kind: 'StorageV2'
  properties: {
    minimumTlsVersion: 'TLS1_2'
    allowBlobPublicAccess: false
  }
}
```

```hcl
resource "azurerm_storage_account" "migration" {
  name                     = var.storage_account_name
  resource_group_name      = var.resource_group_name
  location                 = var.location
  account_tier             = "Standard"
  account_replication_type = "LRS"
  account_kind             = "StorageV2"
  min_tls_version          = "TLS1_2"
  allow_nested_items_to_be_public = false
}
```

---

## Common Variables (Set Once)

```powershell
# PowerShell – Set these at the start of your session
$RG           = "rg-migration-labs"
$LOCATION     = "eastus"
$SQL_SERVER   = "sql-migrate-lab-$(Get-Random -Maximum 9999)"
$SQL_DB       = "AdventureWorksLT"
$SQL_ADMIN    = "sqladmin"
$SQL_PASSWORD  = "P@ssw0rd$(Get-Random -Maximum 9999)!"
$STORAGE_ACCT = "stmigrate$(Get-Random -Maximum 99999)"
$SUBSCRIPTION = (Get-AzContext).Subscription.Id
```

```bash
# Azure CLI – Set these at the start of your session
RG="rg-migration-labs"
LOCATION="eastus"
SQL_SERVER="sql-migrate-lab-$RANDOM"
SQL_DB="AdventureWorksLT"
SQL_ADMIN="sqladmin"
SQL_PASSWORD="P@ssw0rd${RANDOM}!"
STORAGE_ACCT="stmigrate$RANDOM"
SUBSCRIPTION=$(az account show --query id -o tsv)
```

```powershell
# Create the shared resource group
# PowerShell
New-AzResourceGroup -Name $RG -Location $LOCATION

# Azure CLI
az group create --name $RG --location $LOCATION
```

---

## Lab 1: Azure Migrate – Discovery & Assessment

### Objective

Create an Azure Migrate project, run a SQL Server assessment using the CLI, and interpret compatibility results and SKU recommendations.

### ⏱ When to Use This Method

> **Always the first step before any migration.** Azure Migrate provides a centralized hub for discovery, assessment, and migration planning. Use it to evaluate readiness, identify blockers, and get right-sized SKU recommendations before committing to a migration path.

### Key AZ-305 Concepts

- **Azure Migrate Hub** is the central service for all migration planning
- **Discovery** identifies servers, databases, and dependencies
- **Assessment** evaluates readiness for Azure SQL DB, MI, or SQL on VM
- **SKU recommendations** are based on collected performance data
- The `az datamigration` extension handles SQL-specific assessments from CLI

### Prerequisites

- Azure subscription with Contributor access
- SQL Server instance (on-prem or VM) with AdventureWorks sample DB
- `az datamigration` CLI extension installed
- PowerShell: `Az.Migrate` module

### Step 1: Install CLI Extension & PowerShell Module

```bash
# Azure CLI
az extension add --name datamigration --upgrade
```

```powershell
# PowerShell
Install-Module -Name Az.Migrate -Force -AllowClobber
```

### Step 2: Create Azure Migrate Project

```bash
# Azure CLI
az migrate project create \
  --name "MigrateProject-Lab1" \
  --resource-group $RG \
  --location $LOCATION
```

```powershell
# PowerShell
New-AzMigrateProject `
  -Name "MigrateProject-Lab1" `
  -ResourceGroupName $RG `
  -Location $LOCATION
```

### Step 3: Run SQL Server Assessment (CLI)

```bash
# Assess a SQL Server instance for Azure SQL DB readiness
az datamigration get-assessment \
  --connection-string "Data Source=localhost;Initial Catalog=AdventureWorksLT;Integrated Security=True;TrustServerCertificate=True" \
  --output-folder "./assessment-output" \
  --overwrite
```

```powershell
# PowerShell equivalent using the datamigration CLI (no native cmdlet)
# Run from PowerShell but uses the az CLI tool
az datamigration get-assessment `
  --connection-string "Data Source=localhost;Initial Catalog=AdventureWorksLT;Integrated Security=True;TrustServerCertificate=True" `
  --output-folder "./assessment-output" `
  --overwrite
```

### Step 4: Get SKU Recommendations

```bash
# Collect performance data first (runs for a configured duration)
az datamigration performance-data-collection \
  --connection-string "Data Source=localhost;Initial Catalog=AdventureWorksLT;Integrated Security=True;TrustServerCertificate=True" \
  --output-folder "./perf-data" \
  --perf-query-interval 10 \
  --number-of-iteration 3

# Get SKU recommendation based on collected data
az datamigration get-sku-recommendation \
  --output-folder "./perf-data" \
  --display-result \
  --target-platform AzureSqlDatabase
```

### Step 5: Review Assessment Results

```bash
# The assessment output is a JSON file – review key fields
cat ./assessment-output/*.json | python -m json.tool | head -100

# Key fields to look for:
# - "databaseAssessments" → per-database readiness
# - "issues" → compatibility blockers and warnings
# - "targetReadiness" → READY, NOT_READY, READY_WITH_CONDITIONS
```

### Verification

- [ ] Azure Migrate project visible in portal under **Azure Migrate → Servers, databases and web apps**
- [ ] Assessment output JSON exists in `./assessment-output/`
- [ ] SKU recommendation shows target platform, compute size, and storage
- [ ] Any `NOT_READY` issues are documented with remediation guidance

### Cleanup

```bash
# Azure CLI
az migrate project delete --name "MigrateProject-Lab1" --resource-group $RG --yes
rm -rf ./assessment-output ./perf-data
```

```powershell
# PowerShell
Remove-AzMigrateProject -Name "MigrateProject-Lab1" -ResourceGroupName $RG
Remove-Item -Recurse -Force ./assessment-output, ./perf-data
```

### 📝 Exam Tip

> AZ-305 frequently tests when to use Azure Migrate vs. direct migration tools. **Azure Migrate is always the first step** – even if you plan to use DMS, BACPAC, or LRS later. Assessment identifies blockers *before* you invest in migration infrastructure. SKU recommendations prevent over- or under-provisioning.

---

## Lab 2: SQL Server → Azure SQL Database (DMS Online)

### Objective

Perform an online migration from SQL Server to Azure SQL Database using Azure Database Migration Service (DMS), enabling continuous data sync and near-zero downtime cutover.

### ⏱ When to Use This Method

> **Production SQL Server migration needing near-zero downtime.** DMS Online migration continuously replicates changes from source to target. You cut over only when ready, minimizing application downtime to seconds. Ideal for business-critical databases where a maintenance window is impractical.

### Key AZ-305 Concepts

- **DMS (Database Migration Service)** supports both online (continuous sync) and offline modes
- **Online migration** uses CDC (Change Data Capture) to replicate ongoing changes
- DMS requires a **VNet** and connectivity to both source and target
- Source SQL Server needs **full recovery model** and a recent **full backup**
- **Cutover** is a manual step – you decide when the target is caught up

### Prerequisites

- Source SQL Server with AdventureWorksLT (full recovery model enabled)
- Azure subscription with Contributor access
- VNet with connectivity to source SQL Server (or simulated with Azure VM)
- `Az.DataMigration` PowerShell module

### Step 1: Create Target Azure SQL Database

```bash
# Azure CLI
az sql server create \
  --name $SQL_SERVER \
  --resource-group $RG \
  --location $LOCATION \
  --admin-user $SQL_ADMIN \
  --admin-password "$SQL_PASSWORD"

az sql server firewall-rule create \
  --server $SQL_SERVER \
  --resource-group $RG \
  --name "AllowAzureServices" \
  --start-ip-address 0.0.0.0 \
  --end-ip-address 0.0.0.0

az sql db create \
  --server $SQL_SERVER \
  --resource-group $RG \
  --name $SQL_DB \
  --service-objective S2 \
  --backup-storage-redundancy Local
```

```powershell
# PowerShell
New-AzSqlServer `
  -ServerName $SQL_SERVER `
  -ResourceGroupName $RG `
  -Location $LOCATION `
  -SqlAdministratorCredentials (New-Object PSCredential($SQL_ADMIN, (ConvertTo-SecureString $SQL_PASSWORD -AsPlainText -Force)))

New-AzSqlServerFirewallRule `
  -ServerName $SQL_SERVER `
  -ResourceGroupName $RG `
  -FirewallRuleName "AllowAzureServices" `
  -StartIpAddress "0.0.0.0" `
  -EndIpAddress "0.0.0.0"

New-AzSqlDatabase `
  -ServerName $SQL_SERVER `
  -ResourceGroupName $RG `
  -DatabaseName $SQL_DB `
  -RequestedServiceObjectiveName "S2" `
  -BackupStorageRedundancy "Local"
```

### Step 2: Create DMS Instance

```bash
# Azure CLI – Create VNet for DMS
az network vnet create \
  --name vnet-dms \
  --resource-group $RG \
  --location $LOCATION \
  --address-prefix 10.0.0.0/16 \
  --subnet-name snet-dms \
  --subnet-prefix 10.0.1.0/24

# Create DMS instance
az dms create \
  --name "dms-lab2" \
  --resource-group $RG \
  --location $LOCATION \
  --sku-name Premium_4vCores \
  --subnet "/subscriptions/$SUBSCRIPTION/resourceGroups/$RG/providers/Microsoft.Network/virtualNetworks/vnet-dms/subnets/snet-dms"
```

```powershell
# PowerShell
$vnet = New-AzVirtualNetwork `
  -Name "vnet-dms" `
  -ResourceGroupName $RG `
  -Location $LOCATION `
  -AddressPrefix "10.0.0.0/16"

$subnet = Add-AzVirtualNetworkSubnetConfig `
  -Name "snet-dms" `
  -VirtualNetwork $vnet `
  -AddressPrefix "10.0.1.0/24"
$vnet | Set-AzVirtualNetwork
$vnet = Get-AzVirtualNetwork -Name "vnet-dms" -ResourceGroupName $RG
$subnetId = ($vnet.Subnets | Where-Object { $_.Name -eq "snet-dms" }).Id

New-AzDataMigrationService `
  -Name "dms-lab2" `
  -ResourceGroupName $RG `
  -Location $LOCATION `
  -Sku "Premium_4vCores" `
  -VirtualSubnetId $subnetId
```

### Step 3: Start Online Migration (CLI)

```bash
# Using the new az datamigration command for SQL DB migration
az datamigration sql-db create \
  --resource-group $RG \
  --sqldb-instance-name "$SQL_SERVER.database.windows.net" \
  --target-db-name $SQL_DB \
  --migration-service "/subscriptions/$SUBSCRIPTION/resourceGroups/$RG/providers/Microsoft.DataMigration/sqlMigrationServices/dms-lab2" \
  --scope "/subscriptions/$SUBSCRIPTION/resourceGroups/$RG/providers/Microsoft.Sql/servers/$SQL_SERVER/databases/$SQL_DB" \
  --source-database-name "AdventureWorksLT" \
  --source-sql-connection authentication="SqlAuthentication" \
    data-source="source-server.contoso.com" \
    user-name="sa" \
    password="SourceP@ss1"
```

### Step 4: Monitor Migration Status

```bash
# Check migration status
az datamigration sql-db show \
  --resource-group $RG \
  --sqldb-instance-name "$SQL_SERVER.database.windows.net" \
  --target-db-name $SQL_DB \
  --query "{Status:properties.migrationStatus, TablesCompleted:properties.tablesMigrationCompleted}" \
  --output table
```

```powershell
# PowerShell
Get-AzDataMigrationToSqlDb `
  -ResourceGroupName $RG `
  -SqlDbInstanceName "$SQL_SERVER.database.windows.net" `
  -TargetDbName $SQL_DB |
  Select-Object MigrationStatus, TablesMigrationCompleted
```

### Step 5: Cutover

```bash
# When migration status shows "ReadyForCutover"
az datamigration sql-db cutover \
  --resource-group $RG \
  --sqldb-instance-name "$SQL_SERVER.database.windows.net" \
  --target-db-name $SQL_DB \
  --migration-operation-id "<operation-id-from-show-command>"
```

### Verification

- [ ] DMS instance is in `Running` state
- [ ] Migration status progresses: `InProgress` → `ReadyForCutover` → `Succeeded`
- [ ] Target database contains all tables and data from source
- [ ] Run row count comparison: `SELECT COUNT(*) FROM SalesLT.Customer` on both source and target

### Cleanup

```bash
# Azure CLI
az datamigration sql-db delete --resource-group $RG --sqldb-instance-name "$SQL_SERVER.database.windows.net" --target-db-name $SQL_DB --yes
az dms delete --name "dms-lab2" --resource-group $RG --yes
az sql db delete --server $SQL_SERVER --resource-group $RG --name $SQL_DB --yes
az sql server delete --name $SQL_SERVER --resource-group $RG --yes
az network vnet delete --name vnet-dms --resource-group $RG
```

```powershell
# PowerShell
Remove-AzDataMigrationService -Name "dms-lab2" -ResourceGroupName $RG -Force
Remove-AzSqlDatabase -ServerName $SQL_SERVER -ResourceGroupName $RG -DatabaseName $SQL_DB -Force
Remove-AzSqlServer -ServerName $SQL_SERVER -ResourceGroupName $RG -Force
Remove-AzVirtualNetwork -Name "vnet-dms" -ResourceGroupName $RG -Force
```

### 📝 Exam Tip

> AZ-305 tests the difference between **online** (near-zero downtime, uses CDC) and **offline** (requires maintenance window) DMS migrations. Online requires **Premium SKU** DMS and the source must use **full recovery model**. Know that DMS is the recommended tool for production SQL Server migrations – not BACPAC.

---

## Lab 3: SQL Server → Azure SQL Database (BACPAC Export/Import)

### Objective

Migrate a SQL Server database to Azure SQL Database using the BACPAC export/import method. Practice all three export tools (Azure CLI, PowerShell, SqlPackage) and the BCP bulk copy utility.

### ⏱ When to Use This Method

> **Database < 200GB with a maintenance window available, or for dev/test migration.** BACPAC is a portable format containing schema + data. It's simpler than DMS but requires the source to be quiesced (no writes) during export to ensure consistency. Not suitable for production databases needing near-zero downtime.

### Key AZ-305 Concepts

- **BACPAC** = schema + data in a single portable file (`.bacpac`)
- **DACPAC** = schema only (no data) – used for schema deployment
- Export requires **no active transactions** on the source for consistency
- Import to Azure SQL DB is limited to **150GB** BACPAC file size via portal
- **SqlPackage** is the most flexible tool (supports more options than CLI/PS)
- **BCP** (Bulk Copy Program) is for individual table data transfer

### Prerequisites

- Source SQL Server with AdventureWorksLT database
- Azure Storage Account for BACPAC staging
- SqlPackage installed (`dotnet tool install -g microsoft.sqlpackage`)
- Target Azure SQL Server (reuse from Lab 2 or create new)

### Step 1: Create Storage Account for BACPAC Staging

```bash
# Azure CLI
az storage account create \
  --name $STORAGE_ACCT \
  --resource-group $RG \
  --location $LOCATION \
  --sku Standard_LRS

az storage container create \
  --name bacpac \
  --account-name $STORAGE_ACCT

STORAGE_KEY=$(az storage account keys list \
  --account-name $STORAGE_ACCT \
  --resource-group $RG \
  --query "[0].value" -o tsv)
```

```powershell
# PowerShell
New-AzStorageAccount `
  -Name $STORAGE_ACCT `
  -ResourceGroupName $RG `
  -Location $LOCATION `
  -SkuName "Standard_LRS"

$storageCtx = (Get-AzStorageAccount -Name $STORAGE_ACCT -ResourceGroupName $RG).Context
New-AzStorageContainer -Name "bacpac" -Context $storageCtx

$STORAGE_KEY = (Get-AzStorageAccountKey -Name $STORAGE_ACCT -ResourceGroupName $RG)[0].Value
```

### Step 2: Create Target SQL Server & Database

```bash
# Azure CLI
az sql server create \
  --name $SQL_SERVER \
  --resource-group $RG \
  --location $LOCATION \
  --admin-user $SQL_ADMIN \
  --admin-password "$SQL_PASSWORD"

az sql server firewall-rule create \
  --server $SQL_SERVER \
  --resource-group $RG \
  --name "AllowAzure" \
  --start-ip-address 0.0.0.0 \
  --end-ip-address 0.0.0.0
```

```powershell
# PowerShell
New-AzSqlServer `
  -ServerName $SQL_SERVER `
  -ResourceGroupName $RG `
  -Location $LOCATION `
  -SqlAdministratorCredentials (New-Object PSCredential($SQL_ADMIN, (ConvertTo-SecureString $SQL_PASSWORD -AsPlainText -Force)))

New-AzSqlServerFirewallRule `
  -ServerName $SQL_SERVER `
  -ResourceGroupName $RG `
  -FirewallRuleName "AllowAzure" `
  -StartIpAddress "0.0.0.0" `
  -EndIpAddress "0.0.0.0"
```

### Step 3: Export BACPAC (Three Methods)

#### Method A: SqlPackage (Recommended – Most Flexible)

```bash
# Export from source SQL Server
sqlpackage /Action:Export \
  /SourceServerName:localhost \
  /SourceDatabaseName:AdventureWorksLT \
  /TargetFile:"./AdventureWorksLT.bacpac" \
  /SourceTrustServerCertificate:True

# Upload to Blob Storage
az storage blob upload \
  --account-name $STORAGE_ACCT \
  --container-name bacpac \
  --file "./AdventureWorksLT.bacpac" \
  --name "AdventureWorksLT.bacpac" \
  --account-key "$STORAGE_KEY"
```

#### Method B: Azure CLI Export (from existing Azure SQL DB)

```bash
# Export from Azure SQL DB to storage (useful for DB-to-DB migration)
az sql db export \
  --server $SQL_SERVER \
  --resource-group $RG \
  --name $SQL_DB \
  --admin-user $SQL_ADMIN \
  --admin-password "$SQL_PASSWORD" \
  --storage-key-type StorageAccessKey \
  --storage-key "$STORAGE_KEY" \
  --storage-uri "https://$STORAGE_ACCT.blob.core.windows.net/bacpac/AdventureWorksLT.bacpac"
```

#### Method C: PowerShell Export

```powershell
# PowerShell – Export from Azure SQL DB
$exportRequest = New-AzSqlDatabaseExport `
  -ServerName $SQL_SERVER `
  -ResourceGroupName $RG `
  -DatabaseName $SQL_DB `
  -StorageKeyType "StorageAccessKey" `
  -StorageKey $STORAGE_KEY `
  -StorageUri "https://$STORAGE_ACCT.blob.core.windows.net/bacpac/AdventureWorksLT.bacpac" `
  -AdministratorLogin $SQL_ADMIN `
  -AdministratorLoginPassword (ConvertTo-SecureString $SQL_PASSWORD -AsPlainText -Force)

# Check export status
Get-AzSqlDatabaseImportExportStatus -OperationStatusLink $exportRequest.OperationStatusLink
```

### Step 4: Import BACPAC (Two Methods)

#### Method A: Azure CLI Import

```bash
# Azure CLI
az sql db import \
  --server $SQL_SERVER \
  --resource-group $RG \
  --name "${SQL_DB}-imported" \
  --admin-user $SQL_ADMIN \
  --admin-password "$SQL_PASSWORD" \
  --storage-key-type StorageAccessKey \
  --storage-key "$STORAGE_KEY" \
  --storage-uri "https://$STORAGE_ACCT.blob.core.windows.net/bacpac/AdventureWorksLT.bacpac" \
  --service-objective S2
```

#### Method B: PowerShell Import

```powershell
# PowerShell
$importRequest = New-AzSqlDatabaseImport `
  -ServerName $SQL_SERVER `
  -ResourceGroupName $RG `
  -DatabaseName "${SQL_DB}-imported" `
  -StorageKeyType "StorageAccessKey" `
  -StorageKey $STORAGE_KEY `
  -StorageUri "https://$STORAGE_ACCT.blob.core.windows.net/bacpac/AdventureWorksLT.bacpac" `
  -AdministratorLogin $SQL_ADMIN `
  -AdministratorLoginPassword (ConvertTo-SecureString $SQL_PASSWORD -AsPlainText -Force) `
  -Edition "Standard" `
  -ServiceObjectiveName "S2" `
  -DatabaseMaxSizeBytes 10737418240

# Check import status
Get-AzSqlDatabaseImportExportStatus -OperationStatusLink $importRequest.OperationStatusLink
```

### Step 5: BCP Bulk Copy Example

```bash
# Export a single table from source
bcp "AdventureWorksLT.SalesLT.Customer" out "./Customer.dat" \
  -S localhost -T -n

# Import into Azure SQL Database
bcp "SalesLT.Customer" in "./Customer.dat" \
  -S "$SQL_SERVER.database.windows.net" \
  -U $SQL_ADMIN \
  -P "$SQL_PASSWORD" \
  -d "${SQL_DB}-imported" \
  -n
```

```powershell
# Same BCP commands work in PowerShell
bcp "AdventureWorksLT.SalesLT.Customer" out "./Customer.dat" -S localhost -T -n
bcp "SalesLT.Customer" in "./Customer.dat" `
  -S "$SQL_SERVER.database.windows.net" `
  -U $SQL_ADMIN -P $SQL_PASSWORD `
  -d "${SQL_DB}-imported" -n
```

### Verification

- [ ] BACPAC file exists in storage container
- [ ] Imported database appears in Azure portal
- [ ] Table count matches source: `SELECT COUNT(*) FROM sys.tables`
- [ ] Row counts match for key tables

### Cleanup

```bash
# Azure CLI
az sql db delete --server $SQL_SERVER --resource-group $RG --name "${SQL_DB}-imported" --yes
az sql db delete --server $SQL_SERVER --resource-group $RG --name $SQL_DB --yes
az sql server delete --name $SQL_SERVER --resource-group $RG --yes
az storage account delete --name $STORAGE_ACCT --resource-group $RG --yes
rm -f ./AdventureWorksLT.bacpac ./Customer.dat
```

```powershell
# PowerShell
Remove-AzSqlDatabase -ServerName $SQL_SERVER -ResourceGroupName $RG -DatabaseName "${SQL_DB}-imported" -Force
Remove-AzSqlDatabase -ServerName $SQL_SERVER -ResourceGroupName $RG -DatabaseName $SQL_DB -Force
Remove-AzSqlServer -ServerName $SQL_SERVER -ResourceGroupName $RG -Force
Remove-AzStorageAccount -Name $STORAGE_ACCT -ResourceGroupName $RG -Force
Remove-Item -Force ./AdventureWorksLT.bacpac, ./Customer.dat -ErrorAction SilentlyContinue
```

### 📝 Exam Tip

> AZ-305 loves to test BACPAC limitations: **max 150GB** via portal import, database must be **quiesced** (no active writes) during export, and it's **schema + data** (vs. DACPAC = schema only). For databases > 150GB or needing near-zero downtime, use **DMS** instead. Know that `SqlPackage` is the most capable tool – it supports more options than `az sql db export`.

---

## Lab 4: SQL Server → Managed Instance (Log Replay Service)

### Objective

Migrate a SQL Server database to Azure SQL Managed Instance using Log Replay Service (LRS) with native SQL Server backups, enabling near-zero downtime via continuous log restore and cutover.

### ⏱ When to Use This Method

> **Managed Instance migration using native SQL Server backups with near-zero downtime.** LRS replays transaction log backups from Blob Storage to MI in continuous mode. You control the cutover timing. Ideal when you already have a backup strategy and want to use native `.bak`/`.trn` files instead of DMS.

### Key AZ-305 Concepts

- **LRS** replays native SQL Server backups (full, diff, log) on Managed Instance
- Backups must be in **Azure Blob Storage** with SAS token or Managed Identity access
- **Continuous mode** = keeps restoring new log backups until you cutover
- **Autocomplete mode** = automatically finalizes when last log backup is detected
- LRS requires **CHECKSUM** on all backups
- Source database must use **full recovery model**

### Prerequisites

- Azure SQL Managed Instance (pre-provisioned – takes 4-6 hours)
- SQL Server source with AdventureWorksLT in full recovery model
- Azure Storage Account with a container for backups
- SQL Server Management Studio or `sqlcmd`

### Step 1: Create Storage Account for Backups

```bash
# Azure CLI
MI_STORAGE="stmilrs$(echo $RANDOM)"

az storage account create \
  --name $MI_STORAGE \
  --resource-group $RG \
  --location $LOCATION \
  --sku Standard_LRS

az storage container create \
  --name sqlbackups \
  --account-name $MI_STORAGE

# Generate SAS token valid for 24 hours
EXPIRY=$(date -u -d "24 hours" '+%Y-%m-%dT%H:%MZ')
SAS_TOKEN=$(az storage container generate-sas \
  --name sqlbackups \
  --account-name $MI_STORAGE \
  --permissions rwl \
  --expiry $EXPIRY \
  --output tsv)
```

```powershell
# PowerShell
$MI_STORAGE = "stmilrs$(Get-Random -Maximum 99999)"

New-AzStorageAccount `
  -Name $MI_STORAGE `
  -ResourceGroupName $RG `
  -Location $LOCATION `
  -SkuName "Standard_LRS"

$miCtx = (Get-AzStorageAccount -Name $MI_STORAGE -ResourceGroupName $RG).Context
New-AzStorageContainer -Name "sqlbackups" -Context $miCtx

$SAS_TOKEN = New-AzStorageContainerSASToken `
  -Name "sqlbackups" `
  -Context $miCtx `
  -Permission rwl `
  -ExpiryTime (Get-Date).AddHours(24)
```

### Step 2: Take SQL Server Backups with CHECKSUM

```sql
-- Run on source SQL Server (SSMS or sqlcmd)
-- Full backup
BACKUP DATABASE [AdventureWorksLT]
TO DISK = 'C:\Backups\AdventureWorksLT_Full.bak'
WITH CHECKSUM, FORMAT, INIT;

-- Differential backup (optional)
BACKUP DATABASE [AdventureWorksLT]
TO DISK = 'C:\Backups\AdventureWorksLT_Diff.bak'
WITH DIFFERENTIAL, CHECKSUM, FORMAT, INIT;

-- Transaction log backup
BACKUP LOG [AdventureWorksLT]
TO DISK = 'C:\Backups\AdventureWorksLT_Log1.trn'
WITH CHECKSUM, FORMAT, INIT;
```

### Step 3: Upload Backups to Blob Storage

```bash
# Azure CLI – Upload all backup files
az storage blob upload-batch \
  --destination sqlbackups \
  --source "C:\Backups" \
  --account-name $MI_STORAGE \
  --pattern "AdventureWorksLT*" \
  --overwrite
```

```powershell
# PowerShell
$backupFiles = Get-ChildItem -Path "C:\Backups\AdventureWorksLT*"
foreach ($file in $backupFiles) {
    Set-AzStorageBlobContent `
        -File $file.FullName `
        -Container "sqlbackups" `
        -Blob $file.Name `
        -Context $miCtx `
        -Force
}
```

### Step 4: Start LRS in Continuous Mode

```bash
# Azure CLI
MI_NAME="mi-lab4"  # Your Managed Instance name

az sql midb log-replay start \
  --resource-group $RG \
  --managed-instance $MI_NAME \
  --name "AdventureWorksLT" \
  --storage-uri "https://$MI_STORAGE.blob.core.windows.net/sqlbackups" \
  --storage-sas "$SAS_TOKEN" \
  --auto-complete false
```

```powershell
# PowerShell (using Invoke-AzRestMethod as native cmdlet may not exist)
$MI_NAME = "mi-lab4"

# Using Az CLI from PowerShell for LRS
az sql midb log-replay start `
  --resource-group $RG `
  --managed-instance $MI_NAME `
  --name "AdventureWorksLT" `
  --storage-uri "https://$MI_STORAGE.blob.core.windows.net/sqlbackups" `
  --storage-sas "$SAS_TOKEN" `
  --auto-complete false
```

### Step 5: Monitor LRS Progress

```bash
# Check restore status
az sql midb log-replay show \
  --resource-group $RG \
  --managed-instance $MI_NAME \
  --name "AdventureWorksLT" \
  --query "{State:state, LastRestoredFile:lastRestoredFileName, LastRestoredTime:lastRestoredFileTime}"
```

### Step 6: Final Log Backup and Cutover

```sql
-- On source: Take final tail-log backup (stops writes)
BACKUP LOG [AdventureWorksLT]
TO DISK = 'C:\Backups\AdventureWorksLT_TailLog.trn'
WITH CHECKSUM, NORECOVERY;
```

```bash
# Upload final log backup
az storage blob upload \
  --account-name $MI_STORAGE \
  --container-name sqlbackups \
  --file "C:\Backups\AdventureWorksLT_TailLog.trn" \
  --name "AdventureWorksLT_TailLog.trn" \
  --overwrite

# Wait for LRS to process, then cutover
az sql midb log-replay complete \
  --resource-group $RG \
  --managed-instance $MI_NAME \
  --name "AdventureWorksLT" \
  --last-backup-name "AdventureWorksLT_TailLog.trn"
```

### Verification

- [ ] LRS state shows `Restoring` during continuous mode
- [ ] `lastRestoredFileName` advances as new log backups are uploaded
- [ ] After cutover, database state changes to `Online`
- [ ] Connect to MI and verify: `SELECT COUNT(*) FROM SalesLT.Customer`

### Cleanup

```bash
# Azure CLI
az sql midb delete --resource-group $RG --managed-instance $MI_NAME --name "AdventureWorksLT" --yes
az storage account delete --name $MI_STORAGE --resource-group $RG --yes
# Note: MI itself is expensive – delete if no longer needed
# az sql mi delete --resource-group $RG --name $MI_NAME --yes
```

```powershell
# PowerShell
# Remove database from MI
az sql midb delete --resource-group $RG --managed-instance $MI_NAME --name "AdventureWorksLT" --yes
Remove-AzStorageAccount -Name $MI_STORAGE -ResourceGroupName $RG -Force
```

### 📝 Exam Tip

> LRS is unique to **Managed Instance** – it doesn't work with Azure SQL Database. Know the key requirement: backups must include **CHECKSUM**. LRS continuous mode vs. autocomplete mode is a common exam distinction. Also remember that LRS is **free** (no separate service charge), unlike DMS which requires its own resource.

---

## Lab 5: Bulk Data Transfer (AzCopy, Data Box Planning)

### Objective

Use AzCopy for high-performance data transfer to Azure Storage, perform bandwidth calculations, and plan Data Box usage for large-scale offline transfers.

### ⏱ When to Use This Method

> **Large-scale storage migration.** Use AzCopy for network-based transfers where bandwidth is sufficient. For datasets **> 40 TB** or when network bandwidth is limited, use **Azure Data Box** for offline transfer. Data Box ships physical devices for petabyte-scale data movement.

### Key AZ-305 Concepts

- **AzCopy** = high-performance CLI for Blob/File/Table storage transfers
- Supports **Entra ID auth** (recommended) and **SAS tokens**
- `azcopy copy` = one-time copy; `azcopy sync` = incremental sync (like rsync)
- **Data Box family**: Disk (8TB×5), Box (100TB), Heavy (1PB)
- **Bandwidth calculation**: `Data size / available bandwidth = transfer time`
- Data Box is **offline transfer** – shipped physically to Azure datacenter

### Prerequisites

- Azure Storage Account
- AzCopy v10+ installed
- Source data directory with sample files

### Step 1: Install AzCopy and Authenticate with Entra ID

```bash
# Verify AzCopy version
azcopy --version

# Login with Entra ID (recommended over SAS for security)
azcopy login

# For service principal authentication:
# azcopy login --service-principal --application-id <app-id> --tenant-id <tenant-id>
# Then set AZCOPY_SPA_CLIENT_SECRET environment variable
```

```powershell
# PowerShell
azcopy --version
azcopy login

# Verify login status
azcopy login status
```

### Step 2: Create Storage Account and Container

```bash
# Azure CLI
az storage account create \
  --name $STORAGE_ACCT \
  --resource-group $RG \
  --location $LOCATION \
  --sku Standard_LRS \
  --kind StorageV2

az storage container create \
  --name migration-data \
  --account-name $STORAGE_ACCT \
  --auth-mode login

# Assign Storage Blob Data Contributor role for AzCopy with Entra ID
USER_ID=$(az ad signed-in-user show --query id -o tsv)
az role assignment create \
  --assignee $USER_ID \
  --role "Storage Blob Data Contributor" \
  --scope "/subscriptions/$SUBSCRIPTION/resourceGroups/$RG/providers/Microsoft.Storage/storageAccounts/$STORAGE_ACCT"
```

```powershell
# PowerShell
New-AzStorageAccount `
  -Name $STORAGE_ACCT `
  -ResourceGroupName $RG `
  -Location $LOCATION `
  -SkuName "Standard_LRS" `
  -Kind "StorageV2"

$ctx = (Get-AzStorageAccount -Name $STORAGE_ACCT -ResourceGroupName $RG).Context
New-AzStorageContainer -Name "migration-data" -Context $ctx

# Assign RBAC role
$userId = (Get-AzADUser -SignedIn).Id
New-AzRoleAssignment `
  -ObjectId $userId `
  -RoleDefinitionName "Storage Blob Data Contributor" `
  -Scope "/subscriptions/$SUBSCRIPTION/resourceGroups/$RG/providers/Microsoft.Storage/storageAccounts/$STORAGE_ACCT"
```

### Step 3: AzCopy Copy (One-Time Transfer)

```bash
# Upload a directory (recursive)
azcopy copy "./source-data/*" \
  "https://$STORAGE_ACCT.blob.core.windows.net/migration-data/" \
  --recursive=true

# Copy with include/exclude patterns
azcopy copy "./source-data/*" \
  "https://$STORAGE_ACCT.blob.core.windows.net/migration-data/" \
  --recursive=true \
  --include-pattern "*.csv;*.parquet" \
  --exclude-pattern "*.tmp"

# Copy between storage accounts (server-side, no local download)
azcopy copy \
  "https://source-account.blob.core.windows.net/container/*" \
  "https://$STORAGE_ACCT.blob.core.windows.net/migration-data/" \
  --recursive=true
```

### Step 4: AzCopy Sync (Incremental – Like rsync)

```bash
# Sync local directory to blob (only copies changed/new files)
azcopy sync "./source-data" \
  "https://$STORAGE_ACCT.blob.core.windows.net/migration-data/" \
  --recursive=true

# Sync with delete-destination (mirror mode – removes blobs not in source)
azcopy sync "./source-data" \
  "https://$STORAGE_ACCT.blob.core.windows.net/migration-data/" \
  --recursive=true \
  --delete-destination=true
```

### Step 5: AzCopy Benchmark

```bash
# Benchmark upload throughput (does NOT write to storage – test only)
azcopy benchmark \
  "https://$STORAGE_ACCT.blob.core.windows.net/migration-data/" \
  --file-count 100 \
  --size-per-file 100M

# Review output for:
# - Throughput (MB/s)
# - Total transfer time
# - Success/fail counts
```

### Step 6: Bandwidth Calculation Exercise

```
📊 BANDWIDTH CALCULATION EXERCISE

Scenario: Migrate 50 TB of data to Azure Blob Storage
Available bandwidth: 1 Gbps dedicated connection

Step 1: Convert to consistent units
  50 TB = 50,000 GB = 400,000 Gb (gigabits)

Step 2: Calculate theoretical transfer time
  400,000 Gb / 1 Gbps = 400,000 seconds
  = 6,667 minutes = 111 hours = 4.6 days

Step 3: Apply realistic utilization (70-80%)
  At 75% utilization: 111 / 0.75 = 148 hours = 6.2 days

Step 4: Decision
  ✅ 6 days is acceptable → Use AzCopy over the network
  ❌ 6 days too long → Consider Azure Data Box

Rule of thumb:
  > 1 week transfer time OR > 40 TB → Consider Data Box
  > 10 TB with < 100 Mbps → Definitely use Data Box
```

### Step 7: Data Box Order Walkthrough

```
📦 DATA BOX FAMILY DECISION GUIDE

┌─────────────────┬───────────┬──────────────┬─────────────────────┐
│ Device          │ Capacity  │ Use Case     │ Order Method        │
├─────────────────┼───────────┼──────────────┼─────────────────────┤
│ Data Box Disk   │ 8 TB × 5  │ < 40 TB      │ Portal: ship SSDs   │
│ Data Box        │ 100 TB    │ 40-500 TB    │ Portal: ship device │
│ Data Box Heavy  │ 1 PB      │ 500 TB-1 PB  │ Portal: ship device │
│ Data Box Gateway│ Virtual   │ Continuous   │ Portal: deploy VM   │
└─────────────────┴───────────┴──────────────┴─────────────────────┘
```

```bash
# Azure CLI – Create a Data Box order (planning example)
az databox job create \
  --resource-group $RG \
  --name "databox-order-lab5" \
  --location $LOCATION \
  --sku DataBox \
  --contact-name "Migration Team" \
  --phone "555-0100" \
  --email-list "team@contoso.com" \
  --street-address-1 "1 Microsoft Way" \
  --city "Redmond" \
  --state-or-province "WA" \
  --country "US" \
  --postal-code "98052" \
  --storage-account "/subscriptions/$SUBSCRIPTION/resourceGroups/$RG/providers/Microsoft.Storage/storageAccounts/$STORAGE_ACCT"
```

```powershell
# PowerShell
# Data Box orders are typically created via portal, but you can use REST/CLI
# Check Data Box order status
az databox job show \
  --resource-group $RG \
  --name "databox-order-lab5" \
  --query "{Status:status, DeliveryType:deliveryType}"
```

### Verification

- [ ] `azcopy login status` shows authenticated user
- [ ] Files visible in Blob Storage after `azcopy copy`
- [ ] `azcopy sync` only transfers changed files on subsequent runs
- [ ] Benchmark shows expected throughput for your connection
- [ ] Bandwidth calculation correctly identifies Data Box threshold

### Cleanup

```bash
# Azure CLI
az storage account delete --name $STORAGE_ACCT --resource-group $RG --yes
az databox job delete --resource-group $RG --name "databox-order-lab5" --yes 2>/dev/null
```

```powershell
# PowerShell
Remove-AzStorageAccount -Name $STORAGE_ACCT -ResourceGroupName $RG -Force
```

### 📝 Exam Tip

> AZ-305 tests Data Box selection criteria: use **Data Box Disk** for < 40 TB, **Data Box** for 40-500 TB, **Data Box Heavy** for up to 1 PB. Know that `azcopy sync` is incremental (copies only changes), while `azcopy copy` always copies everything. AzCopy with **Entra ID** is more secure than SAS tokens. Server-side copy between storage accounts doesn't download data locally.

---

## Lab 6: File Server → Azure Files (File Sync)

### Objective

Migrate an on-premises file server to Azure Files using Azure File Sync, configuring cloud tiering for hybrid access with local caching.

### ⏱ When to Use This Method

> **Hybrid file server migration with local caching for branch offices.** Azure File Sync keeps a local cache of frequently accessed files while storing the full dataset in Azure Files. Ideal for multi-site scenarios where branch offices need fast local access but you want centralized cloud storage. Also works for lift-and-shift file server migration.

### Key AZ-305 Concepts

- **Azure Files** = fully managed SMB/NFS file shares in the cloud
- **Azure File Sync** = synchronizes on-prem file servers with Azure Files
- **Storage Sync Service** = Azure resource that manages sync relationships
- **Sync Group** = defines sync topology (one cloud endpoint + multiple server endpoints)
- **Cloud Tiering** = replaces infrequently accessed files with pointers (stubs), freeing local disk space
- **Server Endpoint** = path on a registered server that participates in sync

### Prerequisites

- Windows Server 2016+ with a file share
- Azure Storage Account (GPv2 or FileStorage)
- Azure File Sync agent installed on Windows Server

### Step 1: Create Azure Files Share

```bash
# Azure CLI
az storage account create \
  --name $STORAGE_ACCT \
  --resource-group $RG \
  --location $LOCATION \
  --sku Standard_LRS \
  --kind StorageV2

az storage share-rm create \
  --storage-account $STORAGE_ACCT \
  --resource-group $RG \
  --name "file-sync-share" \
  --quota 100 \
  --enabled-protocols SMB
```

```powershell
# PowerShell
New-AzStorageAccount `
  -Name $STORAGE_ACCT `
  -ResourceGroupName $RG `
  -Location $LOCATION `
  -SkuName "Standard_LRS" `
  -Kind "StorageV2"

$ctx = (Get-AzStorageAccount -Name $STORAGE_ACCT -ResourceGroupName $RG).Context
New-AzStorageShare -Name "file-sync-share" -Context $ctx
Set-AzStorageShareQuota -ShareName "file-sync-share" -Quota 100 -Context $ctx
```

### Step 2: Create Storage Sync Service

```bash
# Azure CLI
az storagesync create \
  --name "sync-service-lab6" \
  --resource-group $RG \
  --location $LOCATION
```

```powershell
# PowerShell
Install-Module -Name Az.StorageSync -Force -AllowClobber

New-AzStorageSyncService `
  -ResourceGroupName $RG `
  -Location $LOCATION `
  -StorageSyncServiceName "sync-service-lab6"
```

### Step 3: Create Sync Group with Cloud Endpoint

```bash
# Azure CLI
az storagesync sync-group create \
  --name "sync-group-lab6" \
  --storage-sync-service "sync-service-lab6" \
  --resource-group $RG

# Add cloud endpoint (links to Azure Files share)
STORAGE_ID=$(az storage account show --name $STORAGE_ACCT --resource-group $RG --query id -o tsv)

az storagesync sync-group cloud-endpoint create \
  --name "cloud-ep-lab6" \
  --sync-group-name "sync-group-lab6" \
  --storage-sync-service "sync-service-lab6" \
  --resource-group $RG \
  --storage-account-resource-id $STORAGE_ID \
  --azure-file-share-name "file-sync-share"
```

```powershell
# PowerShell
New-AzStorageSyncGroup `
  -ResourceGroupName $RG `
  -StorageSyncServiceName "sync-service-lab6" `
  -SyncGroupName "sync-group-lab6"

$storageId = (Get-AzStorageAccount -Name $STORAGE_ACCT -ResourceGroupName $RG).Id

New-AzStorageSyncCloudEndpoint `
  -ResourceGroupName $RG `
  -StorageSyncServiceName "sync-service-lab6" `
  -SyncGroupName "sync-group-lab6" `
  -Name "cloud-ep-lab6" `
  -StorageAccountResourceId $storageId `
  -AzureFileShareName "file-sync-share"
```

### Step 4: Register Server and Add Server Endpoint

```powershell
# Run on the Windows Server (after installing Azure File Sync agent)
# Download agent from: https://aka.ms/afs/agent

# Register the server
Register-AzStorageSyncServer `
  -ResourceGroupName $RG `
  -StorageSyncServiceName "sync-service-lab6"

# Add server endpoint with cloud tiering
$registeredServer = Get-AzStorageSyncServer -ResourceGroupName $RG -StorageSyncServiceName "sync-service-lab6"

New-AzStorageSyncServerEndpoint `
  -ResourceGroupName $RG `
  -StorageSyncServiceName "sync-service-lab6" `
  -SyncGroupName "sync-group-lab6" `
  -Name "server-ep-lab6" `
  -ServerResourceId $registeredServer.ServerId `
  -ServerLocalPath "D:\FileShare" `
  -CloudTiering `
  -VolumeFreeSpacePercent 20 `
  -TierFilesOlderThanDays 30
```

### Step 5: Configure Cloud Tiering

```powershell
# Modify cloud tiering settings
Set-AzStorageSyncServerEndpoint `
  -ResourceGroupName $RG `
  -StorageSyncServiceName "sync-service-lab6" `
  -SyncGroupName "sync-group-lab6" `
  -Name "server-ep-lab6" `
  -CloudTiering `
  -VolumeFreeSpacePercent 20 `
  -TierFilesOlderThanDays 14

# Check cloud tiering status
Invoke-AzStorageSyncCompatibilityCheck -Path "D:\FileShare"
```

### Verification

- [ ] Storage Sync Service is created in portal
- [ ] Sync Group shows cloud endpoint (healthy) and server endpoint (syncing/healthy)
- [ ] Files from `D:\FileShare` appear in Azure Files share
- [ ] Cloud tiering creates stub files for old/infrequently accessed files
- [ ] `Get-AzStorageSyncServerEndpoint` shows `SyncStatus: Idle` after initial sync

### Cleanup

```bash
# Azure CLI
az storagesync sync-group server-endpoint delete \
  --name "server-ep-lab6" --sync-group-name "sync-group-lab6" \
  --storage-sync-service "sync-service-lab6" --resource-group $RG --yes
az storagesync sync-group cloud-endpoint delete \
  --name "cloud-ep-lab6" --sync-group-name "sync-group-lab6" \
  --storage-sync-service "sync-service-lab6" --resource-group $RG --yes
az storagesync sync-group delete \
  --name "sync-group-lab6" --storage-sync-service "sync-service-lab6" --resource-group $RG --yes
az storagesync delete --name "sync-service-lab6" --resource-group $RG --yes
az storage account delete --name $STORAGE_ACCT --resource-group $RG --yes
```

```powershell
# PowerShell
Remove-AzStorageSyncServerEndpoint -ResourceGroupName $RG -StorageSyncServiceName "sync-service-lab6" -SyncGroupName "sync-group-lab6" -Name "server-ep-lab6" -Force
Remove-AzStorageSyncCloudEndpoint -ResourceGroupName $RG -StorageSyncServiceName "sync-service-lab6" -SyncGroupName "sync-group-lab6" -Name "cloud-ep-lab6" -Force
Remove-AzStorageSyncGroup -ResourceGroupName $RG -StorageSyncServiceName "sync-service-lab6" -SyncGroupName "sync-group-lab6" -Force
Remove-AzStorageSyncService -ResourceGroupName $RG -StorageSyncServiceName "sync-service-lab6" -Force
Remove-AzStorageAccount -Name $STORAGE_ACCT -ResourceGroupName $RG -Force

# Unregister the server agent
Unregister-AzStorageSyncServer -ResourceGroupName $RG -StorageSyncServiceName "sync-service-lab6" -ServerId $registeredServer.ServerId -Force
```

### 📝 Exam Tip

> Azure File Sync supports **one cloud endpoint per sync group** but **multiple server endpoints** (up to 100). Cloud tiering uses two policies: **volume free space** (guarantee X% free) and **date policy** (tier files older than X days). AZ-305 tests when to use Azure File Sync vs. simple AzCopy migration – File Sync is for **ongoing hybrid** scenarios, AzCopy is for **one-time lift-and-shift**.

---

## Lab 7: PostgreSQL → Azure Database for PostgreSQL

### Objective

Migrate an on-premises PostgreSQL database to Azure Database for PostgreSQL Flexible Server using both offline (`pg_dump`/`pg_restore`) and online (DMS) methods.

### ⏱ When to Use This Method

> **PostgreSQL migration from on-prem, VMs, or other clouds.** Use `pg_dump`/`pg_restore` for small databases with acceptable downtime. Use **DMS online migration** for production databases needing minimal downtime. Flexible Server is the recommended target (Single Server is deprecated).

### Key AZ-305 Concepts

- **Flexible Server** is the current-generation PostgreSQL PaaS (replaces Single Server)
- `pg_dump` + `pg_restore` = offline migration (simple, requires downtime)
- **DMS** supports online migration with logical replication
- Source must have `wal_level = logical` for online migration
- Consider **Private Endpoints** for secure connectivity

### Prerequisites

- Source PostgreSQL server (9.5+) with a sample database
- `pg_dump` and `pg_restore` CLI tools installed
- Azure subscription

### Step 1: Create Azure Database for PostgreSQL Flexible Server

```bash
# Azure CLI
PG_SERVER="pg-migrate-lab7-$RANDOM"
PG_ADMIN="pgadmin"
PG_PASSWORD="PgP@ss$(echo $RANDOM)!"
PG_DB="sampledb"

az postgres flexible-server create \
  --name $PG_SERVER \
  --resource-group $RG \
  --location $LOCATION \
  --admin-user $PG_ADMIN \
  --admin-password "$PG_PASSWORD" \
  --sku-name Standard_B1ms \
  --tier Burstable \
  --storage-size 32 \
  --version 15 \
  --public-access 0.0.0.0-255.255.255.255

az postgres flexible-server db create \
  --server-name $PG_SERVER \
  --resource-group $RG \
  --database-name $PG_DB
```

```powershell
# PowerShell
$PG_SERVER = "pg-migrate-lab7-$(Get-Random -Maximum 9999)"
$PG_ADMIN = "pgadmin"
$PG_PASSWORD = "PgP@ss$(Get-Random -Maximum 9999)!"
$PG_DB = "sampledb"

New-AzPostgreSqlFlexibleServer `
  -Name $PG_SERVER `
  -ResourceGroupName $RG `
  -Location $LOCATION `
  -AdministratorLogin $PG_ADMIN `
  -AdministratorLoginPassword (ConvertTo-SecureString $PG_PASSWORD -AsPlainText -Force) `
  -Sku "Standard_B1ms" `
  -Tier "Burstable" `
  -StorageInMb 32768 `
  -Version "15" `
  -PublicAccess "0.0.0.0-255.255.255.255"

New-AzPostgreSqlFlexibleServerDatabase `
  -ServerName $PG_SERVER `
  -ResourceGroupName $RG `
  -Name $PG_DB
```

### Step 2: Offline Migration – pg_dump / pg_restore

```bash
# Export from source PostgreSQL
pg_dump -h source-server.contoso.com \
  -U postgres \
  -d sampledb \
  -Fc \
  -f ./sampledb.dump

# Import to Azure PostgreSQL Flexible Server
pg_restore -h "$PG_SERVER.postgres.database.azure.com" \
  -U "$PG_ADMIN" \
  -d $PG_DB \
  --no-owner \
  --no-privileges \
  ./sampledb.dump

# Alternative: pg_dump piped directly to pg_restore (no intermediate file)
pg_dump -h source-server.contoso.com -U postgres -d sampledb -Fc | \
pg_restore -h "$PG_SERVER.postgres.database.azure.com" -U "$PG_ADMIN" -d $PG_DB --no-owner
```

```powershell
# PowerShell (same tools, different syntax)
pg_dump -h "source-server.contoso.com" `
  -U postgres `
  -d sampledb `
  -Fc `
  -f ./sampledb.dump

pg_restore -h "$PG_SERVER.postgres.database.azure.com" `
  -U $PG_ADMIN `
  -d $PG_DB `
  --no-owner `
  --no-privileges `
  ./sampledb.dump
```

### Step 3: Online Migration with DMS (Overview)

```bash
# For online migration, configure source PostgreSQL:
# 1. Set wal_level = logical in postgresql.conf
# 2. Restart PostgreSQL service
# 3. Create a replication user:
#    CREATE ROLE dms_user WITH LOGIN REPLICATION PASSWORD 'password';
#    GRANT ALL ON DATABASE sampledb TO dms_user;

# Create DMS migration using Azure portal or CLI
az datamigration sql-managed-instance create \
  --resource-group $RG \
  --managed-instance-name "$PG_SERVER" \
  --target-db-name $PG_DB \
  --migration-service "/subscriptions/$SUBSCRIPTION/resourceGroups/$RG/providers/Microsoft.DataMigration/sqlMigrationServices/dms-pg" \
  --source-database-name "sampledb" \
  --source-sql-connection authentication="PostgreSQLAuthentication" \
    data-source="source-server.contoso.com:5432" \
    user-name="dms_user" \
    password="password"

# Note: For PostgreSQL DMS migration, the Azure portal wizard is recommended
# as it handles the complexity of logical replication setup
```

### Verification

- [ ] Flexible Server is running: `az postgres flexible-server show --name $PG_SERVER --resource-group $RG --query state`
- [ ] Connect with psql: `psql -h "$PG_SERVER.postgres.database.azure.com" -U $PG_ADMIN -d $PG_DB`
- [ ] Verify table count: `\dt` in psql
- [ ] Verify row counts match source

### Cleanup

```bash
# Azure CLI
az postgres flexible-server delete --name $PG_SERVER --resource-group $RG --yes
rm -f ./sampledb.dump
```

```powershell
# PowerShell
Remove-AzPostgreSqlFlexibleServer -Name $PG_SERVER -ResourceGroupName $RG -Force
Remove-Item -Force ./sampledb.dump -ErrorAction SilentlyContinue
```

### 📝 Exam Tip

> AZ-305 tests **Flexible Server vs. Single Server** – always choose Flexible Server (Single Server is deprecated). For online migration, the source must have `wal_level = logical`. Know that `pg_dump -Fc` produces custom format (compressed, supports parallel restore), which is preferred over plain SQL format for large databases.

---

## Lab 8: MySQL → Azure Database for MySQL

### Objective

Migrate a MySQL database to Azure Database for MySQL Flexible Server using offline (`mysqldump`) and online (DMS with binlog replication) methods.

### ⏱ When to Use This Method

> **MySQL migration from on-premises or other clouds.** Use `mysqldump`/`mysql` for simple offline migration with a maintenance window. Use **DMS with binlog replication** for production databases needing continuous sync and near-zero downtime cutover.

### Key AZ-305 Concepts

- **Flexible Server** is the recommended MySQL target (replaces Single Server)
- `mysqldump` + `mysql` = offline migration (requires quiesced source)
- DMS uses **binlog replication** for online migration (continuous sync)
- Source needs `binlog_format = ROW` and `binlog_row_image = FULL` for online
- Consider **Data-in Replication** as alternative to DMS for MySQL-native approach

### Prerequisites

- Source MySQL server (5.7+) with a sample database
- `mysql` and `mysqldump` CLI tools installed
- Azure subscription

### Step 1: Create Azure Database for MySQL Flexible Server

```bash
# Azure CLI
MYSQL_SERVER="mysql-migrate-lab8-$RANDOM"
MYSQL_ADMIN="mysqladmin"
MYSQL_PASSWORD="MyP@ss$(echo $RANDOM)!"
MYSQL_DB="sampledb"

az mysql flexible-server create \
  --name $MYSQL_SERVER \
  --resource-group $RG \
  --location $LOCATION \
  --admin-user $MYSQL_ADMIN \
  --admin-password "$MYSQL_PASSWORD" \
  --sku-name Standard_B1ms \
  --tier Burstable \
  --storage-size 32 \
  --version 8.0.21 \
  --public-access 0.0.0.0

az mysql flexible-server db create \
  --server-name $MYSQL_SERVER \
  --resource-group $RG \
  --database-name $MYSQL_DB
```

```powershell
# PowerShell
$MYSQL_SERVER = "mysql-migrate-lab8-$(Get-Random -Maximum 9999)"
$MYSQL_ADMIN = "mysqladmin"
$MYSQL_PASSWORD = "MyP@ss$(Get-Random -Maximum 9999)!"
$MYSQL_DB = "sampledb"

New-AzMySqlFlexibleServer `
  -Name $MYSQL_SERVER `
  -ResourceGroupName $RG `
  -Location $LOCATION `
  -AdministratorLogin $MYSQL_ADMIN `
  -AdministratorLoginPassword (ConvertTo-SecureString $MYSQL_PASSWORD -AsPlainText -Force) `
  -Sku "Standard_B1ms" `
  -Tier "Burstable" `
  -StorageInMb 32768 `
  -Version "8.0.21" `
  -PublicAccess "0.0.0.0"

New-AzMySqlFlexibleServerDatabase `
  -ServerName $MYSQL_SERVER `
  -ResourceGroupName $RG `
  -Name $MYSQL_DB
```

### Step 2: Offline Migration – mysqldump / mysql

```bash
# Export from source MySQL (schema + data)
mysqldump -h source-server.contoso.com \
  -u root -p \
  --single-transaction \
  --routines \
  --triggers \
  --set-gtid-purged=OFF \
  sampledb > ./sampledb.sql

# Import to Azure MySQL Flexible Server
mysql -h "$MYSQL_SERVER.mysql.database.azure.com" \
  -u $MYSQL_ADMIN \
  -p"$MYSQL_PASSWORD" \
  --ssl-mode=REQUIRED \
  $MYSQL_DB < ./sampledb.sql
```

```powershell
# PowerShell
mysqldump -h "source-server.contoso.com" `
  -u root -p `
  --single-transaction `
  --routines `
  --triggers `
  --set-gtid-purged=OFF `
  sampledb > ./sampledb.sql

mysql -h "$MYSQL_SERVER.mysql.database.azure.com" `
  -u $MYSQL_ADMIN `
  -p"$MYSQL_PASSWORD" `
  --ssl-mode=REQUIRED `
  $MYSQL_DB < ./sampledb.sql
```

### Step 3: Online Migration with DMS (Binlog Replication)

```bash
# Prerequisites on source MySQL:
# 1. Enable binary logging: binlog_format = ROW
# 2. Set binlog_row_image = FULL
# 3. Create replication user:
#    CREATE USER 'dms_user'@'%' IDENTIFIED BY 'password';
#    GRANT REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'dms_user'@'%';
#    GRANT SELECT ON sampledb.* TO 'dms_user'@'%';

# DMS online migration for MySQL is best configured via Azure portal:
# 1. Create DMS instance (Standard tier for online)
# 2. New Migration Project → MySQL → Azure DB for MySQL
# 3. Configure source (binlog-enabled MySQL)
# 4. Configure target (Flexible Server)
# 5. Map databases and tables
# 6. Start migration → Monitor → Cutover when ready
```

### Step 4: Alternative – Data-in Replication (MySQL Native)

```sql
-- On target Azure MySQL Flexible Server:
-- Configure as replica of source using stored procedure
CALL mysql.az_replication_change_master(
  'source-server.contoso.com',  -- master host
  'repl_user',                   -- master user
  'password',                    -- master password
  3306,                          -- master port
  'mysql-bin.000001',            -- master log file
  154,                           -- master log position
  ''                             -- master ssl ca (empty = no SSL)
);

-- Start replication
CALL mysql.az_replication_start;

-- Monitor replication status
SHOW SLAVE STATUS\G

-- When ready to cutover:
CALL mysql.az_replication_stop;
CALL mysql.az_replication_remove_master;
```

### Verification

- [ ] Flexible Server running: `az mysql flexible-server show --name $MYSQL_SERVER --resource-group $RG --query state`
- [ ] Connect: `mysql -h "$MYSQL_SERVER.mysql.database.azure.com" -u $MYSQL_ADMIN -p`
- [ ] Verify tables: `SHOW TABLES FROM sampledb;`
- [ ] Row counts match source for key tables

### Cleanup

```bash
# Azure CLI
az mysql flexible-server delete --name $MYSQL_SERVER --resource-group $RG --yes
rm -f ./sampledb.sql
```

```powershell
# PowerShell
Remove-AzMySqlFlexibleServer -Name $MYSQL_SERVER -ResourceGroupName $RG -Force
Remove-Item -Force ./sampledb.sql -ErrorAction SilentlyContinue
```

### 📝 Exam Tip

> For MySQL online migration, know the two approaches: **DMS** (managed service, portal-driven) vs. **Data-in Replication** (MySQL-native, uses `az_replication_change_master`). `mysqldump --single-transaction` is critical for InnoDB tables – it provides a consistent snapshot without locking. AZ-305 may test when to use Flexible Server vs. Single Server – **always Flexible Server** for new deployments.

---

## Lab 9: MongoDB → Cosmos DB (MongoDB API)

### Objective

Migrate a MongoDB database to Azure Cosmos DB for MongoDB (vCore or RU-based) using `mongodump`/`mongorestore` for offline migration, then configure autoscale throughput.

### ⏱ When to Use This Method

> **MongoDB migration needing global distribution or a fully managed service.** Cosmos DB for MongoDB provides API compatibility with MongoDB while adding global distribution, multi-region writes, and guaranteed SLAs. Use offline `mongodump`/`mongorestore` for simplicity; use DMS for larger databases needing progress tracking.

### Key AZ-305 Concepts

- **Cosmos DB for MongoDB** supports MongoDB wire protocol (3.6, 4.0, 4.2, 5.0, 6.0)
- **RU-based** = serverless or provisioned throughput (Request Units)
- **vCore-based** = familiar MongoDB architecture with dedicated compute
- `mongodump`/`mongorestore` = offline migration tool built into MongoDB
- **Autoscale** adjusts RU/s between min and max based on demand
- For large migrations, use **DMS** or **Azure Cosmos DB Data Migration Tool**

### Prerequisites

- Source MongoDB instance (3.6+) with sample data
- `mongodump` and `mongorestore` tools installed (MongoDB Database Tools)
- Azure subscription

### Step 1: Create Cosmos DB Account with MongoDB API

```bash
# Azure CLI
COSMOS_ACCT="cosmos-mongo-lab9-$RANDOM"
COSMOS_DB="sampledb"

# RU-based account
az cosmosdb create \
  --name $COSMOS_ACCT \
  --resource-group $RG \
  --kind MongoDB \
  --server-version 5.0 \
  --default-consistency-level Session \
  --locations regionName=$LOCATION failoverPriority=0 isZoneRedundant=false \
  --enable-automatic-failover false

# Create database with autoscale
az cosmosdb mongodb database create \
  --account-name $COSMOS_ACCT \
  --resource-group $RG \
  --name $COSMOS_DB \
  --max-throughput 4000

# Create a collection
az cosmosdb mongodb collection create \
  --account-name $COSMOS_ACCT \
  --resource-group $RG \
  --database-name $COSMOS_DB \
  --name "products" \
  --shard "category" \
  --max-throughput 4000
```

```powershell
# PowerShell
$COSMOS_ACCT = "cosmos-mongo-lab9-$(Get-Random -Maximum 9999)"
$COSMOS_DB = "sampledb"

New-AzCosmosDBAccount `
  -Name $COSMOS_ACCT `
  -ResourceGroupName $RG `
  -Location $LOCATION `
  -ApiKind "MongoDB" `
  -ServerVersion "5.0" `
  -DefaultConsistencyLevel "Session"

New-AzCosmosDBMongoDBDatabase `
  -AccountName $COSMOS_ACCT `
  -ResourceGroupName $RG `
  -Name $COSMOS_DB `
  -AutoscaleMaxThroughput 4000

New-AzCosmosDBMongoDBCollection `
  -AccountName $COSMOS_ACCT `
  -ResourceGroupName $RG `
  -DatabaseName $COSMOS_DB `
  -Name "products" `
  -Shard "category" `
  -AutoscaleMaxThroughput 4000
```

### Step 2: Get Connection String

```bash
# Azure CLI
COSMOS_CONN=$(az cosmosdb keys list \
  --name $COSMOS_ACCT \
  --resource-group $RG \
  --type connection-strings \
  --query "connectionStrings[0].connectionString" -o tsv)
echo $COSMOS_CONN
```

```powershell
# PowerShell
$cosmosKeys = Get-AzCosmosDBAccountKey `
  -Name $COSMOS_ACCT `
  -ResourceGroupName $RG `
  -Type "ConnectionStrings"
$COSMOS_CONN = $cosmosKeys["Primary MongoDB Connection String"]
```

### Step 3: Offline Migration – mongodump / mongorestore

```bash
# Export from source MongoDB
mongodump \
  --host source-mongo.contoso.com \
  --port 27017 \
  --db sampledb \
  --out ./mongo-backup

# Restore to Cosmos DB for MongoDB
mongorestore \
  --uri "$COSMOS_CONN" \
  --db $COSMOS_DB \
  --dir ./mongo-backup/sampledb \
  --numInsertionWorkersPerCollection 4 \
  --batchSize 24 \
  --writeConcern "{w:0}"

# Note: --writeConcern "{w:0}" improves speed but reduces durability guarantees during import
# Use --numInsertionWorkersPerCollection to parallelize writes (avoid throttling with too many)
```

```powershell
# PowerShell (same tools)
mongodump `
  --host "source-mongo.contoso.com" `
  --port 27017 `
  --db sampledb `
  --out ./mongo-backup

mongorestore `
  --uri "$COSMOS_CONN" `
  --db $COSMOS_DB `
  --dir ./mongo-backup/sampledb `
  --numInsertionWorkersPerCollection 4 `
  --batchSize 24
```

### Step 4: Validate Data and Configure Autoscale

```bash
# Connect with mongosh to validate
mongosh "$COSMOS_CONN" --eval "
  use $COSMOS_DB;
  db.products.countDocuments();
  db.products.findOne();
"

# Update autoscale max RU/s
az cosmosdb mongodb collection throughput update \
  --account-name $COSMOS_ACCT \
  --resource-group $RG \
  --database-name $COSMOS_DB \
  --name "products" \
  --max-throughput 10000
```

```powershell
# PowerShell – Update throughput
Update-AzCosmosDBMongoDBCollectionThroughput `
  -AccountName $COSMOS_ACCT `
  -ResourceGroupName $RG `
  -DatabaseName $COSMOS_DB `
  -Name "products" `
  -AutoscaleMaxThroughput 10000
```

### Verification

- [ ] Cosmos DB account visible in portal under Azure Cosmos DB
- [ ] Database and collections created with correct shard keys
- [ ] `mongosh` connects and shows correct document counts
- [ ] Autoscale RU/s reflects configured max throughput
- [ ] Data Explorer in portal shows migrated documents

### Cleanup

```bash
# Azure CLI
az cosmosdb delete --name $COSMOS_ACCT --resource-group $RG --yes
rm -rf ./mongo-backup
```

```powershell
# PowerShell
Remove-AzCosmosDBAccount -Name $COSMOS_ACCT -ResourceGroupName $RG -Force
Remove-Item -Recurse -Force ./mongo-backup -ErrorAction SilentlyContinue
```

### 📝 Exam Tip

> AZ-305 tests **RU-based vs. vCore** Cosmos DB for MongoDB. RU-based is serverless/provisioned throughput (pay per operation); vCore is dedicated compute (familiar to MongoDB admins). Know that the **shard key** cannot be changed after collection creation – choose carefully. Autoscale RU/s scales between 10% of max and max (e.g., 400–4000 RU/s). For exams, remember Cosmos DB provides **99.999% SLA** with multi-region writes.

---

## Lab 10: Oracle → Azure SQL (SSMA Overview)

### Objective

Understand and walkthrough the Oracle to Azure SQL migration process using SQL Server Migration Assistant (SSMA), including schema assessment, conversion, and data migration.

### ⏱ When to Use This Method

> **Oracle to Azure migration when SSMA handles schema conversion.** SSMA automates the conversion of Oracle schemas (PL/SQL → T-SQL), assesses compatibility, and migrates data. It's free and supports Oracle 9i through 21c. For very large databases, combine SSMA (schema) with SSIS or ADF (data).

### Key AZ-305 Concepts

- **SSMA (SQL Server Migration Assistant)** = free Microsoft tool for heterogeneous DB migration
- Supports Oracle, MySQL, SAP ASE, and Access → SQL Server / Azure SQL
- **Assessment** identifies conversion issues and effort estimates
- **Schema conversion** = PL/SQL to T-SQL automatic translation
- **Data migration** = bulk copy from Oracle to target
- Common issues: sequences, packages, autonomous transactions, hierarchical queries

### Prerequisites

- SSMA for Oracle installed (download from Microsoft)
- Oracle client or connectivity to source Oracle instance
- Target Azure SQL Database or SQL Server

### Step 1: Install and Launch SSMA for Oracle

```
📋 SSMA INSTALLATION STEPS (GUI-based tool)

1. Download SSMA for Oracle from: https://aka.ms/ssmafororacle
2. Install SSMA for Oracle (includes the client tool)
3. Install SSMA Extension Pack on the target SQL Server (for data migration agents)
4. Launch "Microsoft SQL Server Migration Assistant for Oracle"
```

### Step 2: Create Migration Project

```
📋 SSMA PROJECT SETUP

1. File → New Project
   - Project Name: "OracleToAzureSQL-Lab10"
   - Migrate To: Azure SQL Database
   - Location: default or custom folder

2. Connect to Oracle Source:
   - Provider: OLE DB / Oracle Client
   - Server: oracle-server.contoso.com
   - Port: 1521
   - SID/Service: ORCL
   - Username: migration_user

3. Connect to Azure SQL Target:
   - Server: your-server.database.windows.net
   - Database: target-db
   - Authentication: SQL Authentication
```

### Step 3: Schema Assessment

```
📋 ASSESSMENT WALKTHROUGH

1. In SSMA, expand the Oracle source tree
2. Right-click the schema(s) to migrate → "Create Report"
3. Review the Assessment Report:
   - ✅ Green = converts automatically
   - ⚠️ Yellow = converts with warnings (review needed)
   - ❌ Red = cannot auto-convert (manual intervention required)

COMMON ORACLE → AZURE SQL CONVERSION ISSUES:

┌────────────────────────────┬──────────────────────────────────────┐
│ Oracle Feature             │ Azure SQL Equivalent / Issue         │
├────────────────────────────┼──────────────────────────────────────┤
│ SEQUENCE                   │ IDENTITY column or SEQUENCE object   │
│ PACKAGE / PACKAGE BODY     │ Stored procedures (no packages)      │
│ AUTONOMOUS_TRANSACTION     │ Linked server or redesign            │
│ CONNECT BY (hierarchical)  │ Recursive CTE                        │
│ ROWID                      │ No equivalent – use primary key      │
│ DBMS_OUTPUT                │ PRINT / RAISERROR                    │
│ NVL()                      │ ISNULL() or COALESCE()               │
│ SYSDATE                    │ GETDATE() or SYSDATETIME()           │
│ DECODE()                   │ CASE WHEN expression                 │
│ PL/SQL cursors             │ T-SQL cursors (syntax differences)   │
│ Oracle Data Types          │ NUMBER→DECIMAL, VARCHAR2→NVARCHAR    │
│ Materialized Views         │ Indexed Views (with limitations)     │
│ Global Temp Tables         │ Local temp tables or table variables │
│ Synonyms                   │ Supported in Azure SQL               │
└────────────────────────────┴──────────────────────────────────────┘
```

### Step 4: Convert Schema

```
📋 SCHEMA CONVERSION STEPS

1. Select schema(s) in Oracle tree
2. Right-click → "Convert Schema"
3. SSMA generates T-SQL in the target tree
4. Review converted objects:
   - Check yellow/red items for manual fixes
   - Edit T-SQL directly in SSMA if needed
5. Right-click target schema → "Synchronize with Database"
   (This creates the objects in Azure SQL)
```

### Step 5: Migrate Data

```
📋 DATA MIGRATION

1. Select the Oracle schema
2. Right-click → "Migrate Data"
3. SSMA performs bulk copy table by table
4. Monitor progress in the Data Migration Report

For large tables (> millions of rows):
- Consider using ADF Copy Activity instead of SSMA
- Or SSIS with Oracle connector for complex ETL

Post-migration verification:
- Compare row counts per table
- Validate key business queries
- Test stored procedures and views
```

### Step 6: Create Target SQL DB (CLI – for SSMA target)

```bash
# Azure CLI – Create the target database for SSMA
az sql server create \
  --name $SQL_SERVER \
  --resource-group $RG \
  --location $LOCATION \
  --admin-user $SQL_ADMIN \
  --admin-password "$SQL_PASSWORD"

az sql db create \
  --server $SQL_SERVER \
  --resource-group $RG \
  --name "OracleTarget" \
  --service-objective S3 \
  --max-size 250GB
```

```powershell
# PowerShell
New-AzSqlServer `
  -ServerName $SQL_SERVER `
  -ResourceGroupName $RG `
  -Location $LOCATION `
  -SqlAdministratorCredentials (New-Object PSCredential($SQL_ADMIN, (ConvertTo-SecureString $SQL_PASSWORD -AsPlainText -Force)))

New-AzSqlDatabase `
  -ServerName $SQL_SERVER `
  -ResourceGroupName $RG `
  -DatabaseName "OracleTarget" `
  -RequestedServiceObjectiveName "S3" `
  -MaxSizeBytes 268435456000
```

### Verification

- [ ] SSMA assessment report generated with conversion statistics
- [ ] Schema objects created in Azure SQL (tables, views, procedures)
- [ ] Data migrated with matching row counts
- [ ] Key queries return expected results on target
- [ ] All red/yellow conversion items have been manually addressed

### Cleanup

```bash
# Azure CLI
az sql db delete --server $SQL_SERVER --resource-group $RG --name "OracleTarget" --yes
az sql server delete --name $SQL_SERVER --resource-group $RG --yes
```

```powershell
# PowerShell
Remove-AzSqlDatabase -ServerName $SQL_SERVER -ResourceGroupName $RG -DatabaseName "OracleTarget" -Force
Remove-AzSqlServer -ServerName $SQL_SERVER -ResourceGroupName $RG -Force
```

### 📝 Exam Tip

> AZ-305 tests heterogeneous migration tools. **SSMA** is the answer for Oracle/MySQL/SAP ASE → SQL Server/Azure SQL. Know the common conversion issues (SEQUENCE → IDENTITY, CONNECT BY → recursive CTE, PL/SQL packages → stored procedures). For very large Oracle databases, use SSMA for schema and **ADF** for data transfer. SSMA is **free** but only runs on Windows.

---

## Lab 11: Data Warehouse → Synapse (ADF + PolyBase)

### Objective

Migrate an on-premises data warehouse (SQL Server DW, Teradata, or Netezza) to Azure Synapse Analytics using Azure Data Factory (ADF) for data movement and PolyBase/COPY INTO for high-performance loading.

### ⏱ When to Use This Method

> **On-prem SQL DW, Teradata, or Netezza migration to Azure Synapse Analytics.** Use ADF for orchestrated data movement with connectors to 90+ sources. Use **PolyBase** or **COPY INTO** for high-performance parallel loading into dedicated SQL pools. This is the standard pattern for enterprise data warehouse migration.

### Key AZ-305 Concepts

- **Synapse Analytics** = unified analytics platform (dedicated SQL pool = former SQL DW)
- **Dedicated SQL pool** uses MPP (Massively Parallel Processing) architecture
- **PolyBase** = query engine for external data (Blob, ADLS) using external tables
- **COPY INTO** = simpler T-SQL syntax for bulk loading (recommended over PolyBase for most scenarios)
- **ADF** = cloud ETL/ELT service with 90+ connectors and orchestration
- **Distribution** types: Hash, Round-robin, Replicate – critical for query performance

### Prerequisites

- Azure subscription
- Source data warehouse or sample data files in CSV/Parquet
- Azure Data Lake Storage Gen2 or Blob Storage for staging

### Step 1: Create Synapse Workspace and Dedicated SQL Pool

```bash
# Azure CLI
SYNAPSE_WS="synapse-lab11-$RANDOM"
SYNAPSE_POOL="dwpool"
ADLS_ACCT="adlssynapse$RANDOM"

# Create ADLS Gen2 storage (required for Synapse workspace)
az storage account create \
  --name $ADLS_ACCT \
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
  --storage-account $ADLS_ACCT \
  --file-system "synapse-fs" \
  --sql-admin-login-user $SQL_ADMIN \
  --sql-admin-login-password "$SQL_PASSWORD"

# Open firewall for Azure services
az synapse workspace firewall-rule create \
  --name AllowAzure \
  --workspace-name $SYNAPSE_WS \
  --resource-group $RG \
  --start-ip-address 0.0.0.0 \
  --end-ip-address 0.0.0.0

# Create dedicated SQL pool (DW200c is smallest)
az synapse sql pool create \
  --name $SYNAPSE_POOL \
  --workspace-name $SYNAPSE_WS \
  --resource-group $RG \
  --performance-level DW200c
```

```powershell
# PowerShell
$SYNAPSE_WS = "synapse-lab11-$(Get-Random -Maximum 9999)"
$SYNAPSE_POOL = "dwpool"
$ADLS_ACCT = "adlssynapse$(Get-Random -Maximum 99999)"

New-AzStorageAccount `
  -Name $ADLS_ACCT `
  -ResourceGroupName $RG `
  -Location $LOCATION `
  -SkuName "Standard_LRS" `
  -Kind "StorageV2" `
  -EnableHierarchicalNamespace $true

New-AzSynapseWorkspace `
  -Name $SYNAPSE_WS `
  -ResourceGroupName $RG `
  -Location $LOCATION `
  -DefaultDataLakeStorageAccountName $ADLS_ACCT `
  -DefaultDataLakeStorageFilesystem "synapse-fs" `
  -SqlAdministratorLoginCredential (New-Object PSCredential($SQL_ADMIN, (ConvertTo-SecureString $SQL_PASSWORD -AsPlainText -Force)))

New-AzSynapseFirewallRule `
  -WorkspaceName $SYNAPSE_WS `
  -ResourceGroupName $RG `
  -Name "AllowAzure" `
  -StartIpAddress "0.0.0.0" `
  -EndIpAddress "0.0.0.0"

New-AzSynapseSqlPool `
  -WorkspaceName $SYNAPSE_WS `
  -ResourceGroupName $RG `
  -Name $SYNAPSE_POOL `
  -PerformanceLevel "DW200c"
```

### Step 2: Export Schema and Create Tables

```sql
-- Connect to Synapse dedicated SQL pool and create target tables
-- Note: Distribution strategy is CRITICAL for Synapse performance

CREATE TABLE dbo.DimCustomer (
    CustomerKey INT NOT NULL,
    CustomerName NVARCHAR(100),
    CustomerEmail NVARCHAR(200),
    City NVARCHAR(50),
    State NVARCHAR(50),
    Country NVARCHAR(50)
)
WITH (
    DISTRIBUTION = REPLICATE,           -- Small dimension table
    CLUSTERED COLUMNSTORE INDEX         -- Best for analytics
);

CREATE TABLE dbo.FactSales (
    SalesKey BIGINT NOT NULL,
    CustomerKey INT NOT NULL,
    ProductKey INT NOT NULL,
    OrderDate DATE NOT NULL,
    Quantity INT,
    UnitPrice DECIMAL(18,2),
    TotalAmount DECIMAL(18,2)
)
WITH (
    DISTRIBUTION = HASH(CustomerKey),   -- Large fact table – hash on join column
    CLUSTERED COLUMNSTORE INDEX,
    PARTITION (OrderDate RANGE RIGHT FOR VALUES (
        '2023-01-01', '2023-04-01', '2023-07-01', '2023-10-01'
    ))
);
```

### Step 3: ADF Copy Activity for Data Transfer

```bash
# Azure CLI – Create ADF instance
ADF_NAME="adf-migration-lab11-$RANDOM"

az datafactory create \
  --name $ADF_NAME \
  --resource-group $RG \
  --location $LOCATION

# ADF pipeline creation is best done via Synapse Studio or ADF portal:
# 1. Create Linked Service to source (SQL Server, Oracle, Teradata, etc.)
# 2. Create Linked Service to staging Blob/ADLS
# 3. Create Linked Service to Synapse dedicated SQL pool
# 4. Create Copy Activity:
#    - Source: on-prem database (via Self-Hosted IR)
#    - Sink: Azure Blob Storage (staging)
#    - Then: PolyBase/COPY INTO for final load
```

```powershell
# PowerShell
$ADF_NAME = "adf-migration-lab11-$(Get-Random -Maximum 9999)"

New-AzDataFactoryV2 `
  -Name $ADF_NAME `
  -ResourceGroupName $RG `
  -Location $LOCATION
```

### Step 4: PolyBase External Table + CETAS Example

```sql
-- Step 4a: Create master key and database scoped credential
CREATE MASTER KEY ENCRYPTION BY PASSWORD = 'StrongP@ss123!';

CREATE DATABASE SCOPED CREDENTIAL AzureStorageCred
WITH IDENTITY = 'SHARED ACCESS SIGNATURE',
SECRET = '<your-sas-token-here>';

-- Step 4b: Create external data source
CREATE EXTERNAL DATA SOURCE StagingStorage
WITH (
    TYPE = HADOOP,
    LOCATION = 'wasbs://staging@yourstorageaccount.blob.core.windows.net',
    CREDENTIAL = AzureStorageCred
);

-- Step 4c: Create external file format
CREATE EXTERNAL FILE FORMAT ParquetFormat
WITH (
    FORMAT_TYPE = PARQUET,
    DATA_COMPRESSION = 'org.apache.hadoop.io.compress.SnappyCodec'
);

CREATE EXTERNAL FILE FORMAT CsvFormat
WITH (
    FORMAT_TYPE = DELIMITEDTEXT,
    FORMAT_OPTIONS (
        FIELD_TERMINATOR = ',',
        STRING_DELIMITER = '"',
        FIRST_ROW = 2,
        USE_TYPE_DEFAULT = TRUE
    )
);

-- Step 4d: Create external table (read from storage)
CREATE EXTERNAL TABLE dbo.Ext_FactSales (
    SalesKey BIGINT,
    CustomerKey INT,
    ProductKey INT,
    OrderDate DATE,
    Quantity INT,
    UnitPrice DECIMAL(18,2),
    TotalAmount DECIMAL(18,2)
)
WITH (
    LOCATION = '/factsales/',
    DATA_SOURCE = StagingStorage,
    FILE_FORMAT = ParquetFormat
);

-- Step 4e: Load from external table to Synapse table
INSERT INTO dbo.FactSales
SELECT * FROM dbo.Ext_FactSales;

-- Step 4f: CETAS – Create External Table As Select (export from Synapse)
CREATE EXTERNAL TABLE dbo.Ext_FactSales_Export
WITH (
    LOCATION = '/export/factsales/',
    DATA_SOURCE = StagingStorage,
    FILE_FORMAT = ParquetFormat
)
AS
SELECT * FROM dbo.FactSales WHERE OrderDate >= '2023-01-01';
```

### Step 5: COPY INTO Example (Simpler Alternative)

```sql
-- COPY INTO is simpler than PolyBase and recommended for most scenarios
COPY INTO dbo.FactSales
FROM 'https://yourstorageaccount.blob.core.windows.net/staging/factsales/*.parquet'
WITH (
    FILE_TYPE = 'PARQUET',
    CREDENTIAL = (IDENTITY = 'Shared Access Signature', SECRET = '<sas-token>'),
    MAXERRORS = 10,
    COMPRESSION = 'snappy'
);

-- COPY INTO with CSV format
COPY INTO dbo.DimCustomer
FROM 'https://yourstorageaccount.blob.core.windows.net/staging/customers/*.csv'
WITH (
    FILE_TYPE = 'CSV',
    CREDENTIAL = (IDENTITY = 'Shared Access Signature', SECRET = '<sas-token>'),
    FIRSTROW = 2,
    FIELDTERMINATOR = ',',
    ROWTERMINATOR = '\n',
    MAXERRORS = 10
);
```

### Verification

- [ ] Synapse workspace and dedicated SQL pool are online
- [ ] Tables created with correct distribution (hash/replicate) and CCI
- [ ] External tables query data from Blob/ADLS successfully
- [ ] COPY INTO loads data without errors
- [ ] Row counts match source: `SELECT COUNT(*) FROM dbo.FactSales`
- [ ] Query performance acceptable with columnstore indexes

### Cleanup

```bash
# Azure CLI
az synapse sql pool delete --name $SYNAPSE_POOL --workspace-name $SYNAPSE_WS --resource-group $RG --yes
az synapse workspace delete --name $SYNAPSE_WS --resource-group $RG --yes
az datafactory delete --name $ADF_NAME --resource-group $RG --yes
az storage account delete --name $ADLS_ACCT --resource-group $RG --yes
```

```powershell
# PowerShell
Remove-AzSynapseSqlPool -WorkspaceName $SYNAPSE_WS -ResourceGroupName $RG -Name $SYNAPSE_POOL -Force
Remove-AzSynapseWorkspace -Name $SYNAPSE_WS -ResourceGroupName $RG -Force
Remove-AzDataFactoryV2 -Name $ADF_NAME -ResourceGroupName $RG -Force
Remove-AzStorageAccount -Name $ADLS_ACCT -ResourceGroupName $RG -Force
```

### 📝 Exam Tip

> AZ-305 heavily tests Synapse distribution types: **Hash** for large fact tables (distribute on frequently joined column), **Replicate** for small dimension tables (< 2GB), **Round-robin** for staging tables. Know that **COPY INTO** is simpler and recommended over PolyBase for most loading scenarios. PolyBase requires external data source + file format + external table; COPY INTO needs just the URL and credentials.

---

## Lab 12: Cross-Platform Migration Decision Exercise

### Objective

Practice migration decision-making for AZ-305 exam scenarios. This is NOT a hands-on deployment lab – it's a **decision-making drill** where you analyze scenarios and select the correct migration approach.

### ⏱ When to Use This Method

> **Exam prep drill for migration decision questions.** AZ-305 presents scenarios where you must select the right source, target, tool, and migration mode. This exercise covers the most common exam patterns and trains you to quickly identify the optimal migration path.

### Key AZ-305 Concepts

- Match **source database** to **Azure target service**
- Choose **online** (near-zero downtime) vs. **offline** (maintenance window)
- Select the **right tool** (DMS, BACPAC, LRS, AzCopy, SSMA, etc.)
- Consider **size limits**, **feature compatibility**, and **cost**

### Instructions

For each scenario below, identify:
1. **Source** → **Target** Azure service
2. **Migration Tool/Method**
3. **Online or Offline**
4. **Expected Downtime**
5. **Key Consideration**

*Try to answer before checking the answer key!*

---

### Scenario 1

> A company has a 50 GB SQL Server 2019 database running their e-commerce platform with 24/7 availability requirements. They need to migrate to a PaaS solution with minimal downtime. The database uses only standard T-SQL features.

<details>
<summary>🔑 Answer</summary>

| Field | Answer |
|-------|--------|
| **Source → Target** | SQL Server → Azure SQL Database |
| **Tool** | DMS Online Migration |
| **Mode** | Online (continuous sync) |
| **Downtime** | Seconds (during cutover) |
| **Key Consideration** | 24/7 requirement mandates online migration. Standard T-SQL features = Azure SQL DB compatible. DMS Premium SKU required for online mode. |

</details>

---

### Scenario 2

> A team needs to migrate a 30 GB development SQL Server database to Azure for testing. They have a 4-hour maintenance window on Saturday night. Simplicity is preferred over speed.

<details>
<summary>🔑 Answer</summary>

| Field | Answer |
|-------|--------|
| **Source → Target** | SQL Server → Azure SQL Database |
| **Tool** | BACPAC Export/Import |
| **Mode** | Offline |
| **Downtime** | 1–3 hours (depending on network speed) |
| **Key Consideration** | Dev/test workload with maintenance window = BACPAC is simplest. 30 GB is well under the 150 GB limit. No need for DMS complexity. |

</details>

---

### Scenario 3

> An enterprise runs SQL Server 2016 with SQL Agent Jobs, cross-database queries, CLR assemblies, and Service Broker. They need a cloud migration with near-zero downtime and full feature parity.

<details>
<summary>🔑 Answer</summary>

| Field | Answer |
|-------|--------|
| **Source → Target** | SQL Server → Azure SQL Managed Instance |
| **Tool** | Log Replay Service (LRS) or DMS |
| **Mode** | Online (LRS continuous mode) |
| **Downtime** | Seconds (during cutover) |
| **Key Consideration** | SQL Agent, cross-DB queries, CLR, Service Broker = NOT supported in Azure SQL DB. MI provides near-100% SQL Server compatibility. LRS is free and uses native backups. |

</details>

---

### Scenario 4

> A company has 80 TB of unstructured data (images, documents, logs) on an on-premises NAS. Their internet connection is 500 Mbps. They need to move everything to Azure Blob Storage.

<details>
<summary>🔑 Answer</summary>

| Field | Answer |
|-------|--------|
| **Source → Target** | NAS → Azure Blob Storage |
| **Tool** | Azure Data Box |
| **Mode** | Offline (physical shipment) |
| **Downtime** | N/A (parallel operation) |
| **Key Consideration** | 80 TB at 500 Mbps ≈ 15+ days over network. > 40 TB threshold → Data Box. Use standard Data Box (100 TB capacity). After initial Data Box load, use AzCopy sync for incremental changes. |

</details>

---

### Scenario 5

> A startup runs PostgreSQL 14 on AWS RDS with a 200 GB database. They want to move to Azure with minimal downtime. The database uses standard PostgreSQL features with some PostGIS extensions.

<details>
<summary>🔑 Answer</summary>

| Field | Answer |
|-------|--------|
| **Source → Target** | PostgreSQL (AWS RDS) → Azure DB for PostgreSQL Flexible Server |
| **Tool** | DMS Online Migration |
| **Mode** | Online (logical replication) |
| **Downtime** | Seconds (during cutover) |
| **Key Consideration** | Flexible Server supports PostGIS. DMS online requires `wal_level = logical` on source. AWS RDS supports logical replication with parameter group change. Cross-cloud migration is fully supported. |

</details>

---

### Scenario 6

> A retail company has branch offices in 15 cities. Each office has a local file server (2–5 TB) used by staff daily. They want central cloud storage but need fast local access for frequently used files.

<details>
<summary>🔑 Answer</summary>

| Field | Answer |
|-------|--------|
| **Source → Target** | On-prem File Servers → Azure Files |
| **Tool** | Azure File Sync |
| **Mode** | Online (continuous sync) |
| **Downtime** | None (transparent to users) |
| **Key Consideration** | Azure File Sync with cloud tiering = central cloud storage + local cache for hot files. Each branch office becomes a server endpoint. Cloud tiering frees local disk space. One cloud endpoint per sync group. |

</details>

---

### Scenario 7

> A company has an Oracle 19c database (500 GB) with complex PL/SQL packages, materialized views, and Oracle-specific features. They want to modernize to Azure SQL.

<details>
<summary>🔑 Answer</summary>

| Field | Answer |
|-------|--------|
| **Source → Target** | Oracle → Azure SQL Database |
| **Tool** | SSMA for Oracle (schema) + ADF (data) |
| **Mode** | Offline (phased migration) |
| **Downtime** | Hours to days (depending on data volume and conversion complexity) |
| **Key Consideration** | SSMA converts PL/SQL → T-SQL but complex packages need manual review. Materialized views → indexed views (with limitations). 500 GB data = use ADF for parallel data transfer. Run SSMA assessment first to estimate manual effort. |

</details>

---

### Scenario 8

> A gaming company runs MongoDB 5.0 with 2 TB of player data. They need global distribution across 5 regions with < 10ms read latency and automatic failover.

<details>
<summary>🔑 Answer</summary>

| Field | Answer |
|-------|--------|
| **Source → Target** | MongoDB → Azure Cosmos DB for MongoDB (RU-based) |
| **Tool** | mongodump/mongorestore or DMS |
| **Mode** | Offline (followed by cutover) |
| **Downtime** | Hours (for 2 TB data transfer) |
| **Key Consideration** | Global distribution + multi-region writes + < 10ms latency = Cosmos DB. RU-based model for predictable performance. Choose shard key carefully (cannot change later). Use autoscale RU/s for variable gaming workloads. 99.999% SLA with multi-region writes. |

</details>

---

### Scenario 9

> A data analytics team needs to migrate a 10 TB Teradata data warehouse to Azure. They need to maintain complex SQL transformations, window functions, and aggregate queries.

<details>
<summary>🔑 Answer</summary>

| Field | Answer |
|-------|--------|
| **Source → Target** | Teradata → Azure Synapse Analytics (dedicated SQL pool) |
| **Tool** | ADF (data movement) + COPY INTO / PolyBase (loading) |
| **Mode** | Offline (phased migration) |
| **Downtime** | Days (for 10 TB, phased approach) |
| **Key Consideration** | Synapse supports ANSI SQL, window functions, and complex aggregations. Use ADF with Teradata connector + Self-Hosted IR. Stage data in ADLS Gen2 as Parquet, then COPY INTO for high-performance loading. Hash distribution on fact tables. Teradata BTEQ scripts may need conversion. |

</details>

---

### Scenario 10

> A hospital has a MySQL 5.7 database with PHI (Protected Health Information). They need to migrate to Azure with encryption at rest and in transit, HIPAA compliance, and minimal downtime.

<details>
<summary>🔑 Answer</summary>

| Field | Answer |
|-------|--------|
| **Source → Target** | MySQL → Azure DB for MySQL Flexible Server |
| **Tool** | DMS Online Migration (binlog replication) |
| **Mode** | Online |
| **Downtime** | Seconds (during cutover) |
| **Key Consideration** | Flexible Server provides encryption at rest (AES-256) and in transit (TLS) by default. HIPAA BAA available for Azure. DMS online requires `binlog_format = ROW`. Use Private Endpoints for network isolation. Enable Diagnostic Logging for compliance auditing. |

</details>

---

### Scoring Guide

| Score | Assessment |
|-------|-----------|
| 10/10 | 🏆 Migration expert – ready for AZ-305 |
| 8-9/10 | ✅ Strong understanding – review missed topics |
| 6-7/10 | ⚠️ Good foundation – revisit Labs 1-11 for gaps |
| < 6/10 | 🔄 Need more practice – redo the hands-on labs |

---

## Migration Decision Summary Table

| Source | Target | Tool | Online? | Max Size | Downtime | Key Requirement |
|--------|--------|------|---------|----------|----------|-----------------|
| SQL Server | Azure SQL DB | DMS Online | ✅ | Unlimited | Seconds | Full recovery model, CDC |
| SQL Server | Azure SQL DB | BACPAC | ❌ | 150 GB (portal) | Hours | Quiesced database |
| SQL Server | Azure SQL DB | SqlPackage | ❌ | 200 GB+ | Hours | Most flexible BACPAC tool |
| SQL Server | Azure SQL MI | LRS | ✅ | Unlimited | Seconds | CHECKSUM on backups |
| SQL Server | Azure SQL MI | DMS | ✅ | Unlimited | Seconds | VNet, Premium DMS SKU |
| PostgreSQL | Azure DB for PG | pg_dump/restore | ❌ | Unlimited | Hours | Simple, any PG version |
| PostgreSQL | Azure DB for PG | DMS Online | ✅ | Unlimited | Seconds | wal_level = logical |
| MySQL | Azure DB for MySQL | mysqldump | ❌ | Unlimited | Hours | --single-transaction |
| MySQL | Azure DB for MySQL | DMS / Data-in Repl | ✅ | Unlimited | Seconds | binlog_format = ROW |
| MongoDB | Cosmos DB (Mongo) | mongodump/restore | ❌ | Unlimited | Hours | Choose shard key carefully |
| Oracle | Azure SQL DB | SSMA + ADF | ❌ | Unlimited | Hours–Days | Manual PL/SQL conversion |
| File Server | Azure Files | Azure File Sync | ✅ | 100 TB/share | None | Cloud tiering for local cache |
| Bulk Data | Blob Storage | AzCopy | ❌ | Unlimited | Hours–Days | Entra ID auth recommended |
| Bulk Data (>40TB) | Blob Storage | Data Box | ❌ | 1 PB (Heavy) | Days | Physical shipment |
| SQL DW / Teradata | Synapse | ADF + PolyBase/COPY | ❌ | Unlimited | Hours–Days | Hash distribution, CCI |

### Quick Decision Flowchart

```
Need to migrate data to Azure?
│
├─ Is it a DATABASE migration?
│  ├─ SQL Server?
│  │  ├─ Need near-zero downtime? → DMS Online or LRS (MI)
│  │  ├─ Dev/test, < 150GB? → BACPAC
│  │  └─ Need SQL Agent/CLR/cross-DB queries? → Managed Instance
│  │
│  ├─ PostgreSQL? → Flexible Server + pg_dump (offline) or DMS (online)
│  ├─ MySQL? → Flexible Server + mysqldump (offline) or DMS (online)
│  ├─ MongoDB? → Cosmos DB + mongodump/mongorestore
│  ├─ Oracle? → SSMA assessment → Azure SQL DB/MI
│  └─ Data Warehouse? → Synapse + ADF + COPY INTO
│
├─ Is it FILE/BLOB migration?
│  ├─ Need ongoing sync + local cache? → Azure File Sync
│  ├─ One-time transfer, < 40TB? → AzCopy
│  └─ > 40 TB or limited bandwidth? → Azure Data Box
│
└─ Start with Azure Migrate assessment FIRST (always)
```

---

## Final Resource Group Cleanup

```bash
# Azure CLI – Delete everything when done with all labs
az group delete --name $RG --yes --no-wait
```

```powershell
# PowerShell
Remove-AzResourceGroup -Name $RG -Force -AsJob
```

> ⚠️ **Cost Warning:** Managed Instance (Lab 4) and Synapse dedicated SQL pool (Lab 11) are expensive resources. Delete them immediately after completing their respective labs. Use `az synapse sql pool pause` to pause Synapse when not in use.

---

*Last updated: 2025 | AZ-305 Exam Prep | All labs tested with Azure CLI 2.60+ and Az PowerShell 12+*
