# Azure SQL Hands-On Labs (AZ-305)

> 📖 **Cheat Sheet:** [Azure SQL Cheat Sheet](../CheatSheets/Azure-SQL.md)

> **Prerequisites:** Azure CLI installed, active Azure subscription, appropriate RBAC permissions (Contributor+).  
> **Cost Warning:** Some labs (especially Lab 7) incur significant charges. Always run cleanup commands when finished.

---

## Lab 1: Deploy Azure SQL Database (vCore General Purpose)

### Objective
Deploy a single Azure SQL Database on the vCore General Purpose tier, configure network access, set up Entra ID authentication, and verify connectivity.

### Key AZ-305 Concepts Practiced
- Selecting appropriate service tiers (General Purpose vs. Business Critical vs. Hyperscale)
- vCore purchasing model vs. DTU model trade-offs
- Network security with firewall rules
- Microsoft Entra ID (Azure AD) authentication for centralized identity management

### Steps

#### 1.1 Set Variables
```bash
RG="rg-sql-lab01"
LOCATION="eastus2"
SQL_SERVER="sql-lab01-$(openssl rand -hex 4)"
DB_NAME="sqldb-lab01"
ADMIN_USER="sqladmin"
ADMIN_PASS="P@ssw0rd$(openssl rand -hex 4)!"
ENTRA_ADMIN_EMAIL="your-email@domain.com"  # Replace with your Entra ID email
ENTRA_ADMIN_OID="<your-object-id>"          # Replace with your Entra Object ID
```

#### 1.2 Create Resource Group
```bash
az group create --name $RG --location $LOCATION
```

#### 1.3 Create SQL Server (Logical Server)
```bash
az sql server create \
  --name $SQL_SERVER \
  --resource-group $RG \
  --location $LOCATION \
  --admin-user $ADMIN_USER \
  --admin-password $ADMIN_PASS \
  --enable-public-network true
```

#### 1.4 Configure Firewall Rules
```bash
# Allow Azure services
az sql server firewall-rule create \
  --resource-group $RG \
  --server $SQL_SERVER \
  --name "AllowAzureServices" \
  --start-ip-address 0.0.0.0 \
  --end-ip-address 0.0.0.0

# Allow your client IP
MY_IP=$(curl -s ifconfig.me)
az sql server firewall-rule create \
  --resource-group $RG \
  --server $SQL_SERVER \
  --name "AllowMyIP" \
  --start-ip-address $MY_IP \
  --end-ip-address $MY_IP
```

#### 1.5 Set Microsoft Entra ID Admin
```bash
az sql server ad-admin create \
  --resource-group $RG \
  --server $SQL_SERVER \
  --display-name "SQL Entra Admin" \
  --object-id $ENTRA_ADMIN_OID
```

#### 1.6 Create Database (vCore General Purpose)
```bash
az sql db create \
  --resource-group $RG \
  --server $SQL_SERVER \
  --name $DB_NAME \
  --edition GeneralPurpose \
  --family Gen5 \
  --capacity 2 \
  --compute-model Provisioned \
  --max-size 32GB \
  --zone-redundant false \
  --backup-storage-redundancy Local
```

#### 1.7 Verify Deployment
```bash
# Show database details
az sql db show \
  --resource-group $RG \
  --server $SQL_SERVER \
  --name $DB_NAME \
  --query "{Name:name, Edition:sku.tier, Capacity:sku.capacity, MaxSize:maxSizeBytes, Status:status}" \
  --output table

# Test connectivity with sqlcmd (if installed)
# sqlcmd -S "$SQL_SERVER.database.windows.net" -d $DB_NAME -U $ADMIN_USER -P "$ADMIN_PASS" -Q "SELECT @@VERSION"

# Alternatively, get connection string
az sql db show-connection-string \
  --server $SQL_SERVER \
  --name $DB_NAME \
  --client ado.net \
  --output tsv
```

### Cleanup
```bash
az group delete --name $RG --yes --no-wait
```

---

## Lab 2: Elastic Pool Management

### Objective
Create an Elastic Pool, add multiple databases, monitor resource utilization, and practice scaling operations.

### Key AZ-305 Concepts Practiced
- Elastic Pools for cost optimization with multiple databases having variable usage patterns
- eDTU/vCore sharing across databases
- Right-sizing pools based on aggregate workload
- Scaling strategies for multi-tenant architectures

### Steps

#### 2.1 Set Variables
```bash
RG="rg-sql-lab02"
LOCATION="eastus2"
SQL_SERVER="sql-lab02-$(openssl rand -hex 4)"
POOL_NAME="pool-lab02"
ADMIN_USER="sqladmin"
ADMIN_PASS="P@ssw0rd$(openssl rand -hex 4)!"
```

#### 2.2 Create Resource Group and Server
```bash
az group create --name $RG --location $LOCATION

az sql server create \
  --name $SQL_SERVER \
  --resource-group $RG \
  --location $LOCATION \
  --admin-user $ADMIN_USER \
  --admin-password $ADMIN_PASS
```

#### 2.3 Create Elastic Pool (vCore)
```bash
az sql elastic-pool create \
  --resource-group $RG \
  --server $SQL_SERVER \
  --name $POOL_NAME \
  --edition GeneralPurpose \
  --family Gen5 \
  --capacity 4 \
  --db-min-capacity 0 \
  --db-max-capacity 2 \
  --max-size 128GB \
  --zone-redundant false
```

#### 2.4 Create Databases in the Pool
```bash
for i in 1 2 3 4 5; do
  az sql db create \
    --resource-group $RG \
    --server $SQL_SERVER \
    --name "sqldb-tenant-$i" \
    --elastic-pool $POOL_NAME
done
```

#### 2.5 Add an Existing Standalone Database to the Pool
```bash
# Create a standalone database first
az sql db create \
  --resource-group $RG \
  --server $SQL_SERVER \
  --name "sqldb-standalone" \
  --edition GeneralPurpose \
  --family Gen5 \
  --capacity 2

# Move it into the pool
az sql db update \
  --resource-group $RG \
  --server $SQL_SERVER \
  --name "sqldb-standalone" \
  --elastic-pool $POOL_NAME
```

#### 2.6 Monitor Pool Usage
```bash
# List databases in the pool
az sql elastic-pool list-dbs \
  --resource-group $RG \
  --server $SQL_SERVER \
  --name $POOL_NAME \
  --query "[].{Name:name, Status:status}" \
  --output table

# Get pool metrics (resource utilization)
az monitor metrics list \
  --resource "/subscriptions/$(az account show --query id -o tsv)/resourceGroups/$RG/providers/Microsoft.Sql/servers/$SQL_SERVER/elasticPools/$POOL_NAME" \
  --metric "cpu_percent" "dtu_consumption_percent" "storage_percent" \
  --interval PT5M \
  --output table
```

#### 2.7 Scale the Pool Up
```bash
az sql elastic-pool update \
  --resource-group $RG \
  --server $SQL_SERVER \
  --name $POOL_NAME \
  --capacity 8 \
  --db-max-capacity 4
```

#### 2.8 Scale the Pool Down
```bash
az sql elastic-pool update \
  --resource-group $RG \
  --server $SQL_SERVER \
  --name $POOL_NAME \
  --capacity 4 \
  --db-max-capacity 2
```

#### 2.9 Verify Pool Configuration
```bash
az sql elastic-pool show \
  --resource-group $RG \
  --server $SQL_SERVER \
  --name $POOL_NAME \
  --query "{Name:name, Tier:sku.tier, Capacity:sku.capacity, DbMin:perDatabaseSettings.minCapacity, DbMax:perDatabaseSettings.maxCapacity}" \
  --output table
```

### Cleanup
```bash
az group delete --name $RG --yes --no-wait
```

---

## Lab 3: Hyperscale with Read Replicas

### Objective
Deploy a Hyperscale database, create named read replicas for scale-out, and demonstrate near-instant point-in-time restore capabilities.

### Key AZ-305 Concepts Practiced
- Hyperscale architecture (up to 100 TB, rapid scale-out)
- Named replicas for read scale-out (up to 30 replicas)
- Near-instant database snapshots regardless of size
- ApplicationIntent routing for read/write splitting
- When to choose Hyperscale vs. other tiers

### Steps

#### 3.1 Set Variables
```bash
RG="rg-sql-lab03"
LOCATION="eastus2"
SQL_SERVER="sql-lab03-$(openssl rand -hex 4)"
DB_NAME="sqldb-hyperscale"
ADMIN_USER="sqladmin"
ADMIN_PASS="P@ssw0rd$(openssl rand -hex 4)!"
```

#### 3.2 Create Resource Group and Server
```bash
az group create --name $RG --location $LOCATION

az sql server create \
  --name $SQL_SERVER \
  --resource-group $RG \
  --location $LOCATION \
  --admin-user $ADMIN_USER \
  --admin-password $ADMIN_PASS

# Add firewall rule
MY_IP=$(curl -s ifconfig.me)
az sql server firewall-rule create \
  --resource-group $RG \
  --server $SQL_SERVER \
  --name "AllowMyIP" \
  --start-ip-address $MY_IP \
  --end-ip-address $MY_IP
```

#### 3.3 Create Hyperscale Database
```bash
az sql db create \
  --resource-group $RG \
  --server $SQL_SERVER \
  --name $DB_NAME \
  --edition Hyperscale \
  --family Gen5 \
  --capacity 2 \
  --ha-replicas 1 \
  --backup-storage-redundancy Local
```

#### 3.4 Create Named Read Replicas
```bash
# Named replica 1 - for reporting workloads
az sql db replica create \
  --resource-group $RG \
  --server $SQL_SERVER \
  --name $DB_NAME \
  --partner-server $SQL_SERVER \
  --partner-database "sqldb-hyperscale-read1" \
  --secondary-type Named \
  --family Gen5 \
  --capacity 2

# Named replica 2 - for analytics
az sql db replica create \
  --resource-group $RG \
  --server $SQL_SERVER \
  --name $DB_NAME \
  --partner-server $SQL_SERVER \
  --partner-database "sqldb-hyperscale-read2" \
  --secondary-type Named \
  --family Gen5 \
  --capacity 2
```

#### 3.5 Get Connection Strings for Read Scale-Out
```bash
echo "=== Primary (Read-Write) Connection String ==="
echo "Server=$SQL_SERVER.database.windows.net;Database=$DB_NAME;ApplicationIntent=ReadWrite;"

echo ""
echo "=== HA Replica (Built-in Read Scale-Out) ==="
echo "Server=$SQL_SERVER.database.windows.net;Database=$DB_NAME;ApplicationIntent=ReadOnly;"

echo ""
echo "=== Named Replica 1 (Reporting) ==="
echo "Server=$SQL_SERVER.database.windows.net;Database=sqldb-hyperscale-read1;ApplicationIntent=ReadOnly;"

echo ""
echo "=== Named Replica 2 (Analytics) ==="
echo "Server=$SQL_SERVER.database.windows.net;Database=sqldb-hyperscale-read2;ApplicationIntent=ReadOnly;"
```

#### 3.6 Demonstrate Near-Instant Backup (Point-in-Time Restore)
```bash
# Hyperscale supports near-instant PITR regardless of database size
# Restore to a point 10 minutes ago
RESTORE_TIME=$(date -u -d '-10 minutes' '+%Y-%m-%dT%H:%M:%SZ')

az sql db restore \
  --resource-group $RG \
  --server $SQL_SERVER \
  --name "sqldb-hyperscale-restored" \
  --dest-name "sqldb-hyperscale-restored" \
  --time $RESTORE_TIME \
  --edition Hyperscale \
  --family Gen5 \
  --capacity 2
```

#### 3.7 List All Replicas
```bash
az sql db replica list-links \
  --resource-group $RG \
  --server $SQL_SERVER \
  --name $DB_NAME \
  --output table
```

#### 3.8 Verify Deployment
```bash
az sql db list \
  --resource-group $RG \
  --server $SQL_SERVER \
  --query "[].{Name:name, Edition:sku.tier, Capacity:sku.capacity, Status:status}" \
  --output table
```

### Cleanup
```bash
az group delete --name $RG --yes --no-wait
```

---

## Lab 4: Geo-Replication & Failover Groups

### Objective
Set up active geo-replication with an auto-failover group spanning two regions, perform a manual failover, and verify application reconnection through the failover group listener endpoint.

### Key AZ-305 Concepts Practiced
- RPO and RTO requirements driving replication choices
- Active geo-replication vs. failover groups
- Failover group listener endpoints for transparent failover
- Grace period configuration for automatic failover
- Read-only listener for geographic read distribution

### Steps

#### 4.1 Set Variables
```bash
RG="rg-sql-lab04"
PRIMARY_LOCATION="eastus2"
SECONDARY_LOCATION="westus2"
PRIMARY_SERVER="sql-lab04-primary-$(openssl rand -hex 4)"
SECONDARY_SERVER="sql-lab04-secondary-$(openssl rand -hex 4)"
DB_NAME="sqldb-geo"
FOG_NAME="fog-lab04-$(openssl rand -hex 4)"
ADMIN_USER="sqladmin"
ADMIN_PASS="P@ssw0rd$(openssl rand -hex 4)!"
```

#### 4.2 Create Resource Group and Primary Server
```bash
az group create --name $RG --location $PRIMARY_LOCATION

az sql server create \
  --name $PRIMARY_SERVER \
  --resource-group $RG \
  --location $PRIMARY_LOCATION \
  --admin-user $ADMIN_USER \
  --admin-password $ADMIN_PASS
```

#### 4.3 Create Primary Database
```bash
az sql db create \
  --resource-group $RG \
  --server $PRIMARY_SERVER \
  --name $DB_NAME \
  --edition GeneralPurpose \
  --family Gen5 \
  --capacity 2 \
  --backup-storage-redundancy Geo
```

#### 4.4 Create Secondary Server (Different Region)
```bash
az sql server create \
  --name $SECONDARY_SERVER \
  --resource-group $RG \
  --location $SECONDARY_LOCATION \
  --admin-user $ADMIN_USER \
  --admin-password $ADMIN_PASS
```

#### 4.5 Create Failover Group
```bash
az sql failover-group create \
  --resource-group $RG \
  --server $PRIMARY_SERVER \
  --partner-server $SECONDARY_SERVER \
  --name $FOG_NAME \
  --failover-policy Automatic \
  --grace-period 1 \
  --add-db $DB_NAME
```

#### 4.6 Show Failover Group Listener Endpoints
```bash
az sql failover-group show \
  --resource-group $RG \
  --server $PRIMARY_SERVER \
  --name $FOG_NAME \
  --query "{
    Name:name,
    PrimaryServer:partnerServers[0].id,
    ReadWriteEndpoint:readWriteEndpoint.failoverPolicy,
    GracePeriod:readWriteEndpoint.failoverWithDataLossGracePeriodMinutes,
    ReadWriteListener:'$FOG_NAME.database.windows.net',
    ReadOnlyListener:'$FOG_NAME.secondary.database.windows.net'
  }" \
  --output table

echo ""
echo "=== Application Connection Strings ==="
echo "Read-Write: Server=$FOG_NAME.database.windows.net;Database=$DB_NAME;"
echo "Read-Only:  Server=$FOG_NAME.secondary.database.windows.net;Database=$DB_NAME;"
```

#### 4.7 Check Replication Status
```bash
az sql failover-group show \
  --resource-group $RG \
  --server $PRIMARY_SERVER \
  --name $FOG_NAME \
  --query "{ReplicationRole:replicationRole, ReplicationState:replicationState}" \
  --output table
```

#### 4.8 Perform Manual Failover
```bash
echo "Initiating manual failover to secondary region ($SECONDARY_LOCATION)..."

# Failover is initiated from the SECONDARY server
az sql failover-group set-primary \
  --resource-group $RG \
  --server $SECONDARY_SERVER \
  --name $FOG_NAME

echo "Failover complete. Verifying new roles..."
```

#### 4.9 Verify Failover Completed
```bash
# Check from the new primary (was secondary)
az sql failover-group show \
  --resource-group $RG \
  --server $SECONDARY_SERVER \
  --name $FOG_NAME \
  --query "{Name:name, Role:replicationRole}" \
  --output table
```

#### 4.10 Fail Back to Original Primary
```bash
az sql failover-group set-primary \
  --resource-group $RG \
  --server $PRIMARY_SERVER \
  --name $FOG_NAME

echo "Failback complete."
```

### Cleanup
```bash
az group delete --name $RG --yes --no-wait
```

---

## Lab 5: Security Configuration

### Objective
Implement defense-in-depth security for Azure SQL: encryption (TDE with CMK, Always Encrypted), data masking, auditing, and threat protection.

### Key AZ-305 Concepts Practiced
- Transparent Data Encryption with customer-managed keys (regulatory compliance)
- Always Encrypted for client-side encryption of sensitive data
- Dynamic Data Masking for non-privileged user access
- SQL Auditing for compliance and forensics
- Microsoft Defender for SQL (threat detection, vulnerability assessment)

### Steps

#### 5.1 Set Variables
```bash
RG="rg-sql-lab05"
LOCATION="eastus2"
SQL_SERVER="sql-lab05-$(openssl rand -hex 4)"
DB_NAME="sqldb-secure"
KV_NAME="kv-sql-lab05-$(openssl rand -hex 4)"
STORAGE_ACCT="staudit$(openssl rand -hex 4)"
ADMIN_USER="sqladmin"
ADMIN_PASS="P@ssw0rd$(openssl rand -hex 4)!"
```

#### 5.2 Create Infrastructure
```bash
az group create --name $RG --location $LOCATION

az sql server create \
  --name $SQL_SERVER \
  --resource-group $RG \
  --location $LOCATION \
  --admin-user $ADMIN_USER \
  --admin-password $ADMIN_PASS \
  --assign-identity

# Get the server's managed identity
SERVER_IDENTITY=$(az sql server show \
  --resource-group $RG \
  --name $SQL_SERVER \
  --query identity.principalId -o tsv)

az sql db create \
  --resource-group $RG \
  --server $SQL_SERVER \
  --name $DB_NAME \
  --edition GeneralPurpose \
  --family Gen5 \
  --capacity 2
```

#### 5.3 Enable TDE with Customer-Managed Key (CMK)

```bash
# Create Key Vault with purge protection (required for TDE)
az keyvault create \
  --name $KV_NAME \
  --resource-group $RG \
  --location $LOCATION \
  --enable-purge-protection true \
  --enable-soft-delete true

# Grant SQL Server identity access to Key Vault
az keyvault set-policy \
  --name $KV_NAME \
  --object-id $SERVER_IDENTITY \
  --key-permissions get wrapKey unwrapKey list

# Create encryption key
az keyvault key create \
  --vault-name $KV_NAME \
  --name "sql-tde-key" \
  --kty RSA \
  --size 2048

# Get Key URI
KEY_URI=$(az keyvault key show \
  --vault-name $KV_NAME \
  --name "sql-tde-key" \
  --query key.kid -o tsv)

# Set TDE protector to customer-managed key
az sql server tde-key set \
  --resource-group $RG \
  --server $SQL_SERVER \
  --server-key-type AzureKeyVault \
  --kid $KEY_URI

# Verify TDE is using CMK
az sql server tde-key show \
  --resource-group $RG \
  --server $SQL_SERVER \
  --query "{KeyType:serverKeyType, KeyUri:uri}" \
  --output table
```

#### 5.4 Configure Always Encrypted

> **Note:** Always Encrypted is configured via SSMS or SqlServer PowerShell module in practice. Below shows the conceptual T-SQL and PowerShell approach.

```powershell
# PowerShell: Generate Column Master Key and Column Encryption Key
# Run this in a PowerShell session with the SqlServer module
Import-Module SqlServer

# Create a column master key in Azure Key Vault
$cmkSettings = New-SqlAzureKeyVaultColumnMasterKeySettings `
  -KeyURL "$KEY_URI"

# Connect and configure
$connStr = "Server=$SQL_SERVER.database.windows.net;Database=$DB_NAME;User ID=$ADMIN_USER;Password=$ADMIN_PASS;"
$database = Get-SqlDatabase -ConnectionString $connStr

# Create the CMK metadata in the database
New-SqlColumnMasterKey -Name "CMK_AKV" -InputObject $database -ColumnMasterKeySettings $cmkSettings

# Create a column encryption key
New-SqlColumnEncryptionKey -Name "CEK_1" -InputObject $database -ColumnMasterKey "CMK_AKV"
```

```sql
-- T-SQL: Create table with Always Encrypted columns
CREATE TABLE dbo.Patients (
    PatientId INT IDENTITY PRIMARY KEY,
    SSN CHAR(11) COLLATE Latin1_General_BIN2
        ENCRYPTED WITH (
            COLUMN_ENCRYPTION_KEY = CEK_1,
            ENCRYPTION_TYPE = Deterministic,
            ALGORITHM = 'AEAD_AES_256_CBC_HMAC_SHA_256'
        ),
    FirstName NVARCHAR(50),
    LastName NVARCHAR(50),
    BirthDate DATE
        ENCRYPTED WITH (
            COLUMN_ENCRYPTION_KEY = CEK_1,
            ENCRYPTION_TYPE = Randomized,
            ALGORITHM = 'AEAD_AES_256_CBC_HMAC_SHA_256'
        )
);
```

#### 5.5 Configure Dynamic Data Masking

```bash
# Mask email column with email masking function
az sql db ddm add \
  --resource-group $RG \
  --server $SQL_SERVER \
  --database-name $DB_NAME \
  --schema "dbo" \
  --table "Customers" \
  --column "Email" \
  --masking-function "Email"

# Mask credit card with partial masking (show last 4)
az sql db ddm add \
  --resource-group $RG \
  --server $SQL_SERVER \
  --database-name $DB_NAME \
  --schema "dbo" \
  --table "Customers" \
  --column "CreditCard" \
  --masking-function "Partial" \
  --prefix-size 0 \
  --suffix-size 4 \
  --replacement "XXXX-XXXX-XXXX-"

# Mask phone number with default masking
az sql db ddm add \
  --resource-group $RG \
  --server $SQL_SERVER \
  --database-name $DB_NAME \
  --schema "dbo" \
  --table "Customers" \
  --column "PhoneNumber" \
  --masking-function "Default"

# List all masking rules
az sql db ddm list \
  --resource-group $RG \
  --server $SQL_SERVER \
  --database-name $DB_NAME \
  --output table
```

#### 5.6 Enable Auditing to Storage Account

```bash
# Create storage account for audit logs
az storage account create \
  --name $STORAGE_ACCT \
  --resource-group $RG \
  --location $LOCATION \
  --sku Standard_LRS

# Enable server-level auditing
az sql server audit-policy update \
  --resource-group $RG \
  --name $SQL_SERVER \
  --state Enabled \
  --storage-account $STORAGE_ACCT \
  --retention-days 90

# Enable database-level auditing (optional, more granular)
az sql db audit-policy update \
  --resource-group $RG \
  --server $SQL_SERVER \
  --name $DB_NAME \
  --state Enabled \
  --storage-account $STORAGE_ACCT \
  --retention-days 90

# Verify auditing status
az sql server audit-policy show \
  --resource-group $RG \
  --name $SQL_SERVER \
  --query "{State:state, StorageAccount:storageAccountAccessKey, RetentionDays:retentionDays}" \
  --output table
```

#### 5.7 Enable Microsoft Defender for SQL

```bash
# Enable Advanced Threat Protection
az sql server advanced-threat-protection-setting update \
  --resource-group $RG \
  --name $SQL_SERVER \
  --state Enabled

# Enable Vulnerability Assessment
az sql server va-setting update \
  --resource-group $RG \
  --name $SQL_SERVER \
  --storage-account $STORAGE_ACCT \
  --email-admins true

# Verify Defender status
az sql server advanced-threat-protection-setting show \
  --resource-group $RG \
  --name $SQL_SERVER \
  --query "{State:state}" \
  --output table
```

### Cleanup
```bash
az group delete --name $RG --yes --no-wait
# Note: Key Vault with purge protection will be soft-deleted
# To permanently remove after retention period:
# az keyvault purge --name $KV_NAME
```

---

## Lab 6: Serverless Compute Tier

### Objective
Deploy a serverless Azure SQL Database, configure auto-pause and auto-scaling, observe the behavior, and understand billing implications.

### Key AZ-305 Concepts Practiced
- Serverless vs. Provisioned compute model trade-offs
- Auto-pause delay configuration (cost savings for intermittent workloads)
- Min/Max vCore settings for performance boundaries
- Per-second billing model
- Use cases: dev/test, intermittent workloads, unpredictable usage patterns

### Steps

#### 6.1 Set Variables
```bash
RG="rg-sql-lab06"
LOCATION="eastus2"
SQL_SERVER="sql-lab06-$(openssl rand -hex 4)"
DB_SERVERLESS="sqldb-serverless"
DB_PROVISIONED="sqldb-provisioned"
ADMIN_USER="sqladmin"
ADMIN_PASS="P@ssw0rd$(openssl rand -hex 4)!"
```

#### 6.2 Create Resource Group and Server
```bash
az group create --name $RG --location $LOCATION

az sql server create \
  --name $SQL_SERVER \
  --resource-group $RG \
  --location $LOCATION \
  --admin-user $ADMIN_USER \
  --admin-password $ADMIN_PASS

MY_IP=$(curl -s ifconfig.me)
az sql server firewall-rule create \
  --resource-group $RG \
  --server $SQL_SERVER \
  --name "AllowMyIP" \
  --start-ip-address $MY_IP \
  --end-ip-address $MY_IP
```

#### 6.3 Create Serverless Database
```bash
az sql db create \
  --resource-group $RG \
  --server $SQL_SERVER \
  --name $DB_SERVERLESS \
  --edition GeneralPurpose \
  --family Gen5 \
  --compute-model Serverless \
  --auto-pause-delay 60 \
  --min-capacity 0.5 \
  --capacity 4 \
  --max-size 32GB \
  --backup-storage-redundancy Local
```

#### 6.4 Create Provisioned Database for Comparison
```bash
az sql db create \
  --resource-group $RG \
  --server $SQL_SERVER \
  --name $DB_PROVISIONED \
  --edition GeneralPurpose \
  --family Gen5 \
  --compute-model Provisioned \
  --capacity 2 \
  --max-size 32GB \
  --backup-storage-redundancy Local
```

#### 6.5 Compare Configurations
```bash
az sql db list \
  --resource-group $RG \
  --server $SQL_SERVER \
  --query "[].{Name:name, Tier:sku.tier, ComputeModel:kind, MinCapacity:minCapacity, MaxCapacity:sku.capacity, AutoPause:autoPauseDelay}" \
  --output table
```

#### 6.6 Demonstrate Auto-Pause Behavior
```bash
echo "=== Serverless Auto-Pause Demonstration ==="
echo ""
echo "1. After 60 minutes of inactivity, the database will auto-pause"
echo "2. While paused, you only pay for storage (no compute charges)"
echo "3. First connection after pause causes ~1 minute resume delay"
echo ""

# Check current status
az sql db show \
  --resource-group $RG \
  --server $SQL_SERVER \
  --name $DB_SERVERLESS \
  --query "{Name:name, Status:status, AutoPauseDelay:autoPauseDelay, MinVCores:minCapacity, MaxVCores:sku.capacity}" \
  --output table

echo ""
echo "Status 'Paused' = auto-paused (no compute billing)"
echo "Status 'Online' = active (per-second compute billing)"
```

#### 6.7 Modify Auto-Pause and Scaling Settings
```bash
# Increase auto-pause delay to 120 minutes
az sql db update \
  --resource-group $RG \
  --server $SQL_SERVER \
  --name $DB_SERVERLESS \
  --auto-pause-delay 120 \
  --min-capacity 1 \
  --capacity 8

# Disable auto-pause (set to -1)
# az sql db update \
#   --resource-group $RG \
#   --server $SQL_SERVER \
#   --name $DB_SERVERLESS \
#   --auto-pause-delay -1
```

#### 6.8 Compare Billing Estimates
```bash
echo "=== Billing Comparison (East US 2, Gen5) ==="
echo ""
echo "Provisioned (2 vCores, always on):"
echo "  ~\$0.2052/vCore/hour × 2 vCores × 730 hours = ~\$299/month"
echo ""
echo "Serverless (0.5-4 vCores, 60 min auto-pause):"
echo "  Compute: ~\$0.000145/vCore/second (billed per second)"
echo "  If active 8 hours/day: ~\$0.000145 × avg 1 vCore × 28800 sec × 30 days = ~\$125/month"
echo "  Storage: same as provisioned"
echo ""
echo "Serverless is cost-effective when utilization < ~60% of time"
```

### Cleanup
```bash
az group delete --name $RG --yes --no-wait
```

---

## Lab 7: SQL Managed Instance Networking

### Objective
Design and deploy the network infrastructure for Azure SQL Managed Instance, understand subnet requirements, and configure secure connectivity.

### Key AZ-305 Concepts Practiced
- SQL MI subnet requirements (dedicated subnet, minimum /27 CIDR)
- VNet integration and private connectivity
- NSG rules required for MI operation
- Route table configuration
- Connectivity patterns (VNet peering, VPN, ExpressRoute)
- Deployment time considerations (4-6 hours for first instance in subnet)

### Steps

#### 7.1 Set Variables
```bash
RG="rg-sql-lab07"
LOCATION="eastus2"
VNET_NAME="vnet-mi"
MI_SUBNET="snet-mi"
VM_SUBNET="snet-vm"
MI_NAME="sqlmi-lab07-$(openssl rand -hex 4)"
NSG_MI="nsg-mi"
RT_MI="rt-mi"
ADMIN_USER="sqladmin"
ADMIN_PASS="P@ssw0rd$(openssl rand -hex 4)!"
```

#### 7.2 Create Resource Group
```bash
az group create --name $RG --location $LOCATION
```

#### 7.3 Create VNet with Subnets
```bash
# Create VNet
az network vnet create \
  --resource-group $RG \
  --name $VNET_NAME \
  --location $LOCATION \
  --address-prefixes 10.0.0.0/16

# Create MI subnet (minimum /27, recommended /26 or larger)
az network vnet subnet create \
  --resource-group $RG \
  --vnet-name $VNET_NAME \
  --name $MI_SUBNET \
  --address-prefixes 10.0.0.0/24 \
  --delegations "Microsoft.Sql/managedInstances"

# Create VM subnet for client connectivity testing
az network vnet subnet create \
  --resource-group $RG \
  --vnet-name $VNET_NAME \
  --name $VM_SUBNET \
  --address-prefixes 10.0.1.0/24
```

#### 7.4 Create and Configure NSG for MI Subnet
```bash
az network nsg create \
  --resource-group $RG \
  --name $NSG_MI \
  --location $LOCATION

# Required inbound rules for MI management
az network nsg rule create \
  --resource-group $RG \
  --nsg-name $NSG_MI \
  --name "allow-management-inbound" \
  --priority 100 \
  --direction Inbound \
  --access Allow \
  --protocol Tcp \
  --source-address-prefixes SqlManagement \
  --source-port-ranges "*" \
  --destination-address-prefixes 10.0.0.0/24 \
  --destination-port-ranges 9000 9003 1438 1440 1452

az network nsg rule create \
  --resource-group $RG \
  --nsg-name $NSG_MI \
  --name "allow-health-probe-inbound" \
  --priority 300 \
  --direction Inbound \
  --access Allow \
  --protocol "*" \
  --source-address-prefixes AzureLoadBalancer \
  --source-port-ranges "*" \
  --destination-address-prefixes 10.0.0.0/24 \
  --destination-port-ranges "*"

az network nsg rule create \
  --resource-group $RG \
  --nsg-name $NSG_MI \
  --name "allow-tds-inbound" \
  --priority 1000 \
  --direction Inbound \
  --access Allow \
  --protocol Tcp \
  --source-address-prefixes VirtualNetwork \
  --source-port-ranges "*" \
  --destination-address-prefixes 10.0.0.0/24 \
  --destination-port-ranges 1433

# Required outbound rules
az network nsg rule create \
  --resource-group $RG \
  --nsg-name $NSG_MI \
  --name "allow-management-outbound" \
  --priority 100 \
  --direction Outbound \
  --access Allow \
  --protocol Tcp \
  --source-address-prefixes 10.0.0.0/24 \
  --source-port-ranges "*" \
  --destination-address-prefixes AzureCloud \
  --destination-port-ranges 443 12000

az network nsg rule create \
  --resource-group $RG \
  --nsg-name $NSG_MI \
  --name "allow-mi-subnet-outbound" \
  --priority 200 \
  --direction Outbound \
  --access Allow \
  --protocol "*" \
  --source-address-prefixes 10.0.0.0/24 \
  --source-port-ranges "*" \
  --destination-address-prefixes 10.0.0.0/24 \
  --destination-port-ranges "*"

# Associate NSG with MI subnet
az network vnet subnet update \
  --resource-group $RG \
  --vnet-name $VNET_NAME \
  --name $MI_SUBNET \
  --network-security-group $NSG_MI
```

#### 7.5 Create Route Table for MI Subnet
```bash
az network route-table create \
  --resource-group $RG \
  --name $RT_MI \
  --location $LOCATION

# Associate route table with MI subnet
az network vnet subnet update \
  --resource-group $RG \
  --vnet-name $VNET_NAME \
  --name $MI_SUBNET \
  --route-table $RT_MI
```

#### 7.6 Deploy SQL Managed Instance

> ⚠️ **Warning:** MI deployment takes 4-6 hours for the first instance in a subnet. Consider using `--no-wait` and monitoring progress.

```bash
az sql mi create \
  --resource-group $RG \
  --name $MI_NAME \
  --location $LOCATION \
  --admin-user $ADMIN_USER \
  --admin-password $ADMIN_PASS \
  --subnet "/subscriptions/$(az account show --query id -o tsv)/resourceGroups/$RG/providers/Microsoft.Network/virtualNetworks/$VNET_NAME/subnets/$MI_SUBNET" \
  --vnet-name $VNET_NAME \
  --edition GeneralPurpose \
  --family Gen5 \
  --capacity 4 \
  --storage 32 \
  --license-type BasePrice \
  --backup-storage-redundancy Local \
  --public-data-endpoint-enabled false \
  --no-wait

echo "MI deployment started. Monitor with:"
echo "az sql mi show --resource-group $RG --name $MI_NAME --query provisioningState -o tsv"
```

#### 7.7 Monitor Deployment Progress
```bash
# Check provisioning state (run periodically)
az sql mi show \
  --resource-group $RG \
  --name $MI_NAME \
  --query "{Name:name, State:provisioningState, FQDN:fullyQualifiedDomainName}" \
  --output table
```

#### 7.8 Create Test VM for Connectivity (Optional)
```bash
# Create a VM in the same VNet to test connectivity
az vm create \
  --resource-group $RG \
  --name "vm-mi-client" \
  --vnet-name $VNET_NAME \
  --subnet $VM_SUBNET \
  --image Win2022Datacenter \
  --size Standard_B2s \
  --admin-username $ADMIN_USER \
  --admin-password $ADMIN_PASS \
  --no-wait

# After MI is provisioned, connect from the VM using:
# sqlcmd -S <MI_FQDN> -U sqladmin -P <password> -Q "SELECT @@VERSION"
```

#### 7.9 Verify MI Connectivity
```bash
# Get the MI FQDN (only available after provisioning completes)
MI_FQDN=$(az sql mi show \
  --resource-group $RG \
  --name $MI_NAME \
  --query fullyQualifiedDomainName -o tsv)

echo "Connect to MI at: $MI_FQDN"
echo "Port: 1433 (private endpoint within VNet)"
echo "Connection string: Server=$MI_FQDN;Database=master;User ID=$ADMIN_USER;Password=<password>;"
```

### Cleanup
```bash
# MI deletion also takes significant time
az sql mi delete --resource-group $RG --name $MI_NAME --yes --no-wait
# Wait for MI deletion, then delete resource group
az group delete --name $RG --yes --no-wait
```

---

## Lab 8: Database Migration with DMS

### Objective
Use Azure Database Migration Service to assess, plan, and execute a migration from an on-premises SQL Server (simulated) to Azure SQL Database.

### Key AZ-305 Concepts Practiced
- Migration assessment and compatibility checking
- Online vs. offline migration strategies
- Azure Database Migration Service (DMS) architecture
- Pre-migration validation and post-migration verification
- Minimizing downtime during migration (continuous sync)

### Steps

#### 8.1 Set Variables
```bash
RG="rg-sql-lab08"
LOCATION="eastus2"
SQL_SERVER="sql-lab08-$(openssl rand -hex 4)"
DB_NAME="sqldb-migrated"
DMS_NAME="dms-lab08"
VNET_NAME="vnet-dms"
DMS_SUBNET="snet-dms"
ADMIN_USER="sqladmin"
ADMIN_PASS="P@ssw0rd$(openssl rand -hex 4)!"
```

#### 8.2 Create Infrastructure
```bash
az group create --name $RG --location $LOCATION

# Create target SQL Server and database
az sql server create \
  --name $SQL_SERVER \
  --resource-group $RG \
  --location $LOCATION \
  --admin-user $ADMIN_USER \
  --admin-password $ADMIN_PASS

az sql db create \
  --resource-group $RG \
  --server $SQL_SERVER \
  --name $DB_NAME \
  --edition GeneralPurpose \
  --family Gen5 \
  --capacity 2

# Create VNet for DMS
az network vnet create \
  --resource-group $RG \
  --name $VNET_NAME \
  --location $LOCATION \
  --address-prefixes 10.0.0.0/16

az network vnet subnet create \
  --resource-group $RG \
  --vnet-name $VNET_NAME \
  --name $DMS_SUBNET \
  --address-prefixes 10.0.1.0/24
```

#### 8.3 Create Azure Database Migration Service
```bash
# Register the DMS resource provider (if not already registered)
az provider register --namespace Microsoft.DataMigration

# Create DMS instance
az dms create \
  --resource-group $RG \
  --name $DMS_NAME \
  --location $LOCATION \
  --sku-name Premium_4vCores \
  --subnet "/subscriptions/$(az account show --query id -o tsv)/resourceGroups/$RG/providers/Microsoft.Network/virtualNetworks/$VNET_NAME/subnets/$DMS_SUBNET"

# Verify DMS is provisioned
az dms show \
  --resource-group $RG \
  --name $DMS_NAME \
  --query "{Name:name, State:provisioningState, SKU:sku.name}" \
  --output table
```

#### 8.4 Assess Source Database Compatibility

> **Note:** The Data Migration Assistant (DMA) is a desktop tool. For CLI-based assessment, use the `az datamigration` commands or the newer Azure Migrate with DMS.

```bash
# Using the newer az datamigration extension for assessment
az extension add --name datamigration 2>/dev/null

# Perform SQL Server assessment (replace with actual source connection)
# This assesses compatibility issues for migration to Azure SQL DB
az datamigration get-assessment \
  --connection-string "Server=localhost;Database=AdventureWorks;Trusted_Connection=True;" \
  --output-folder "./assessment-output" \
  --overwrite

echo "=== Assessment Checklist ==="
echo "1. Feature parity issues (CLR, cross-database queries, Service Broker)"
echo "2. Breaking changes (deprecated syntax)"
echo "3. Migration blockers vs. warnings"
echo "4. Azure SQL DB vs. MI recommendation based on features used"
```

#### 8.5 Create Migration Project
```bash
# Create a migration project
az dms project create \
  --resource-group $RG \
  --service-name $DMS_NAME \
  --name "migration-project-adventureworks" \
  --source-platform SQL \
  --target-platform SQLDB \
  --location $LOCATION

# List projects
az dms project list \
  --resource-group $RG \
  --service-name $DMS_NAME \
  --output table
```

#### 8.6 Create and Start Migration Task

> **Note:** In production, you would configure source/target connection info and table mappings. Below shows the structure.

```bash
# Create migration task (online migration for minimal downtime)
# The task JSON defines source, target, and mapping configuration
cat << 'EOF'
{
  "taskType": "Migrate.SqlServer.SqlDb",
  "input": {
    "sourceConnectionInfo": {
      "type": "SqlConnectionInfo",
      "dataSource": "source-server.database.windows.net",
      "authentication": "SqlAuthentication",
      "userName": "sourceadmin",
      "password": "SourceP@ss",
      "encryptConnection": true,
      "trustServerCertificate": true
    },
    "targetConnectionInfo": {
      "type": "SqlConnectionInfo",
      "dataSource": "<SQL_SERVER>.database.windows.net",
      "authentication": "SqlAuthentication",
      "userName": "sqladmin",
      "password": "<ADMIN_PASS>",
      "encryptConnection": true
    },
    "selectedDatabases": [
      {
        "name": "AdventureWorks",
        "targetDatabaseName": "sqldb-migrated",
        "tableMap": {
          "dbo.Customers": "dbo.Customers",
          "dbo.Orders": "dbo.Orders"
        }
      }
    ]
  }
}
EOF

echo ""
echo "In practice, save the above as task-config.json and run:"
echo "az dms project task create \\"
echo "  --resource-group $RG \\"
echo "  --service-name $DMS_NAME \\"
echo "  --project-name migration-project-adventureworks \\"
echo "  --name migrate-adventureworks \\"
echo "  --task-type OfflineMigration \\"
echo "  --source-connection-json @source-conn.json \\"
echo "  --target-connection-json @target-conn.json \\"
echo "  --database-options-json @db-options.json"
```

#### 8.7 Monitor Migration Progress
```bash
# Check task status
az dms project task show \
  --resource-group $RG \
  --service-name $DMS_NAME \
  --project-name "migration-project-adventureworks" \
  --name "migrate-adventureworks" \
  --query "{State:properties.state, Errors:properties.errors}" \
  --output table
```

#### 8.8 Post-Migration Validation
```bash
echo "=== Post-Migration Validation Checklist ==="
echo ""
echo "1. Row counts match source and target:"
echo "   SELECT COUNT(*) FROM dbo.Customers -- compare with source"
echo ""
echo "2. Schema validation:"
echo "   SELECT TABLE_NAME, COLUMN_NAME, DATA_TYPE"
echo "   FROM INFORMATION_SCHEMA.COLUMNS ORDER BY TABLE_NAME"
echo ""
echo "3. Index and constraint verification:"
echo "   SELECT name, type_desc FROM sys.indexes"
echo "   SELECT name, type_desc FROM sys.objects WHERE type IN ('PK','FK','UQ')"
echo ""
echo "4. Application connectivity test"
echo "5. Performance baseline comparison"
echo "6. Security validation (users, permissions, encryption)"

# Verify target database is accessible
az sql db show \
  --resource-group $RG \
  --server $SQL_SERVER \
  --name $DB_NAME \
  --query "{Name:name, Status:status, MaxSize:maxSizeBytes, Tier:sku.tier}" \
  --output table
```

#### 8.9 Cutover (Online Migration)
```bash
echo "=== Online Migration Cutover Steps ==="
echo ""
echo "1. Stop application writes to source database"
echo "2. Wait for DMS to sync final transactions (CDC lag = 0)"
echo "3. Perform cutover:"
echo "   az dms project task cutover \\"
echo "     --resource-group $RG \\"
echo "     --service-name $DMS_NAME \\"
echo "     --project-name migration-project-adventureworks \\"
echo "     --name migrate-adventureworks \\"
echo "     --object-name AdventureWorks"
echo "4. Redirect application connection strings to Azure SQL"
echo "5. Validate application functionality"
echo "6. Decommission source (after validation period)"
```

### Cleanup
```bash
az group delete --name $RG --yes --no-wait
```

---

## Quick Reference: AZ-305 Decision Matrix

| Scenario | Recommended Service |
|----------|-------------------|
| Lift-and-shift with minimal changes | SQL Managed Instance |
| New cloud-native app | Azure SQL Database |
| Multiple small DBs, variable load | Elastic Pool |
| Database > 4 TB or need fast scale | Hyperscale |
| Dev/test, intermittent workloads | Serverless |
| RPO < 5 sec, multi-region | Failover Groups |
| Need SQL Agent, cross-DB queries | SQL Managed Instance |
| Cost-sensitive, predictable load | Provisioned General Purpose |
| Mission-critical, low latency | Business Critical |
| Read-heavy workloads | Hyperscale Named Replicas |

---

## Key Exam Tips

1. **Failover Groups vs. Geo-Replication**: Failover groups provide automatic failover with listener endpoints; geo-replication requires manual failover and app-level handling.
2. **Serverless limitations**: Not available for Elastic Pools, Hyperscale, or Business Critical tiers.
3. **MI subnet**: Must be delegated, minimum /27 (but /26+ recommended), no other resources allowed.
4. **TDE with CMK**: Requires Key Vault with purge protection enabled; server needs GET, WRAP, UNWRAP permissions.
5. **Hyperscale**: Cannot be moved to other tiers (one-way migration as of latest updates—check current docs).
6. **Always Encrypted**: Client-side encryption; SQL Server never sees plaintext. Deterministic for equality searches, Randomized for no query support.
7. **Backup storage redundancy**: Choose LRS/ZRS/GRS at creation; affects PITR and geo-restore capabilities.
