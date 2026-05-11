# Azure Storage Accounts – Hands-On Labs

> 📖 **Cheat Sheet:** [Azure Storage Accounts](../Design Data Storage Solutions/Azure-Storage-Accounts.md)

> **Exam Focus:** AZ-305 – Design Data Storage Solutions  
> **Prerequisite:** Azure CLI installed, active subscription, Owner/Contributor role

---

## Lab 1: Storage Account with Lifecycle Management

### Objective
Create a Standard GPv2 storage account with GRS redundancy, upload blobs, and configure an automated lifecycle management policy that transitions blobs through access tiers and eventually deletes them.

### Key AZ-305 Concepts Practiced
- Storage account types and redundancy options (LRS, GRS, RA-GRS, GZRS)
- Access tiers: Hot, Cool, Cold, Archive
- Lifecycle management policies for cost optimization
- GPv2 as the recommended general-purpose account type

### Steps

```bash
# Variables
RG="rg-storage-lab1"
LOCATION="eastus2"
SA="stlab1lifecycle$RANDOM"

# Create resource group
az group create --name $RG --location $LOCATION

# Create Standard GPv2 account with GRS redundancy
az storage account create \
  --name $SA \
  --resource-group $RG \
  --location $LOCATION \
  --sku Standard_GRS \
  --kind StorageV2 \
  --access-tier Hot

# Get connection string
CONN=$(az storage account show-connection-string \
  --name $SA --resource-group $RG --query connectionString -o tsv)

# Create containers for different data categories
az storage container create --name "active-data" --connection-string $CONN
az storage container create --name "reports" --connection-string $CONN
az storage container create --name "archives" --connection-string $CONN

# Upload sample blobs
echo "Current quarter financial data" > active.txt
echo "Last year annual report" > report.txt
echo "Historical records 2019" > archive.txt

az storage blob upload --container-name "active-data" \
  --file active.txt --name "q4-2024-financials.txt" --connection-string $CONN

az storage blob upload --container-name "reports" \
  --file report.txt --name "annual-report-2023.txt" --connection-string $CONN

az storage blob upload --container-name "archives" \
  --file archive.txt --name "records-2019.txt" --connection-string $CONN

# Create lifecycle management policy (Hot → Cool → Cold → Archive → Delete)
cat > lifecycle-policy.json << 'EOF'
{
  "rules": [
    {
      "enabled": true,
      "name": "tiering-rule",
      "type": "Lifecycle",
      "definition": {
        "actions": {
          "baseBlob": {
            "tierToCool": {
              "daysAfterModificationGreaterThan": 30
            },
            "tierToCold": {
              "daysAfterModificationGreaterThan": 90
            },
            "tierToArchive": {
              "daysAfterModificationGreaterThan": 180
            },
            "delete": {
              "daysAfterModificationGreaterThan": 365
            }
          },
          "snapshot": {
            "delete": {
              "daysAfterCreationGreaterThan": 90
            }
          }
        },
        "filters": {
          "blobTypes": ["blockBlob"],
          "prefixMatch": ["active-data/", "reports/", "archives/"]
        }
      }
    }
  ]
}
EOF

az storage account management-policy create \
  --account-name $SA \
  --resource-group $RG \
  --policy @lifecycle-policy.json
```

### Verification

```bash
# Verify lifecycle policy is applied
az storage account management-policy show \
  --account-name $SA --resource-group $RG

# Verify account redundancy
az storage account show --name $SA --resource-group $RG \
  --query "{Name:name, SKU:sku.name, Kind:kind, AccessTier:accessTier}"

# List blobs and their tiers
az storage blob list --container-name "active-data" \
  --connection-string $CONN \
  --query "[].{Name:name, Tier:properties.blobTier}" -o table
```

### Cleanup

```bash
rm -f active.txt report.txt archive.txt lifecycle-policy.json
az group delete --name $RG --yes --no-wait
```

---

## Lab 2: Private Endpoint & Network Security

### Objective
Secure a storage account by disabling public access and configuring a private endpoint with private DNS resolution, ensuring traffic stays on the Microsoft backbone network.

### Key AZ-305 Concepts Practiced
- Private endpoints for PaaS services
- Network isolation and the default deny model
- Private DNS zones for name resolution
- VNet integration patterns for storage
- Defense in depth: network + identity + encryption

### Steps

```bash
# Variables
RG="rg-storage-lab2"
LOCATION="eastus2"
SA="stlab2private$RANDOM"
VNET="vnet-storage"
SUBNET_PE="snet-privateendpoints"
SUBNET_WORKLOAD="snet-workload"

# Create resource group
az group create --name $RG --location $LOCATION

# Create VNet with subnets
az network vnet create \
  --name $VNET \
  --resource-group $RG \
  --location $LOCATION \
  --address-prefix 10.0.0.0/16

az network vnet subnet create \
  --name $SUBNET_PE \
  --resource-group $RG \
  --vnet-name $VNET \
  --address-prefix 10.0.1.0/24

az network vnet subnet create \
  --name $SUBNET_WORKLOAD \
  --resource-group $RG \
  --vnet-name $VNET \
  --address-prefix 10.0.2.0/24

# Create storage account with public access DISABLED
az storage account create \
  --name $SA \
  --resource-group $RG \
  --location $LOCATION \
  --sku Standard_LRS \
  --kind StorageV2 \
  --public-network-access Disabled \
  --allow-blob-public-access false

# Get storage account resource ID
SA_ID=$(az storage account show --name $SA --resource-group $RG --query id -o tsv)

# Create private endpoint for blob service
az network private-endpoint create \
  --name "pe-${SA}-blob" \
  --resource-group $RG \
  --vnet-name $VNET \
  --subnet $SUBNET_PE \
  --private-connection-resource-id $SA_ID \
  --group-id blob \
  --connection-name "pec-${SA}-blob" \
  --location $LOCATION

# Create private DNS zone for blob storage
az network private-dns zone create \
  --resource-group $RG \
  --name "privatelink.blob.core.windows.net"

# Link DNS zone to VNet
az network private-dns link vnet create \
  --resource-group $RG \
  --zone-name "privatelink.blob.core.windows.net" \
  --name "dnslink-${VNET}" \
  --virtual-network $VNET \
  --registration-enabled false

# Create DNS zone group (auto-registers DNS records)
az network private-endpoint dns-zone-group create \
  --resource-group $RG \
  --endpoint-name "pe-${SA}-blob" \
  --name "default" \
  --private-dns-zone "privatelink.blob.core.windows.net" \
  --zone-name "blob"
```

### Verification

```bash
# Verify private endpoint connection state
az network private-endpoint show \
  --name "pe-${SA}-blob" --resource-group $RG \
  --query "privateLinkServiceConnections[0].privateLinkServiceConnectionState.status" -o tsv

# Verify DNS records
az network private-dns record-set a list \
  --resource-group $RG \
  --zone-name "privatelink.blob.core.windows.net" -o table

# Verify storage account network rules (public access disabled)
az storage account show --name $SA --resource-group $RG \
  --query "{PublicAccess:publicNetworkAccess, DefaultAction:networkRuleSet.defaultAction}" -o table

# Attempt public access (should fail from outside VNet)
az storage container list --account-name $SA --auth-mode login 2>&1 || echo "Access denied as expected"
```

### Cleanup

```bash
az group delete --name $RG --yes --no-wait
```

---

## Lab 3: Immutable Storage (WORM) for Compliance

### Objective
Configure immutable blob storage with time-based retention policies and legal holds to meet regulatory requirements (SEC 17a-4, CFTC, FINRA compliance).

### Key AZ-305 Concepts Practiced
- WORM (Write Once, Read Many) storage for compliance
- Time-based retention policies (locked vs unlocked)
- Legal holds (tag-based, independent of retention)
- SEC 17a-4(f) compliance pattern
- Immutability at container level vs account level (version-level)

### Steps

```bash
# Variables
RG="rg-storage-lab3"
LOCATION="eastus2"
SA="stlab3worm$RANDOM"

# Create resource group
az group create --name $RG --location $LOCATION

# Create storage account
az storage account create \
  --name $SA \
  --resource-group $RG \
  --location $LOCATION \
  --sku Standard_LRS \
  --kind StorageV2

CONN=$(az storage account show-connection-string \
  --name $SA --resource-group $RG --query connectionString -o tsv)

# Create container for compliance data
az storage container create \
  --name "compliance-records" \
  --connection-string $CONN

# Set time-based retention policy (7 days for lab; production: years)
az storage container immutability-policy create \
  --resource-group $RG \
  --account-name $SA \
  --container-name "compliance-records" \
  --period 7 \
  --allow-protected-append-writes true

# Upload a compliance record
echo "Financial transaction record - DO NOT DELETE" > transaction.txt

az storage blob upload \
  --container-name "compliance-records" \
  --file transaction.txt \
  --name "trade-2024-001.txt" \
  --connection-string $CONN

# Attempt to delete (SHOULD FAIL due to retention policy)
echo "--- Attempting deletion (should fail) ---"
az storage blob delete \
  --container-name "compliance-records" \
  --name "trade-2024-001.txt" \
  --connection-string $CONN 2>&1 || echo "DELETE BLOCKED: Immutability policy prevents deletion"

# Add a legal hold (independent of retention policy)
az storage container legal-hold set \
  --account-name $SA \
  --resource-group $RG \
  --container-name "compliance-records" \
  --tags "investigation-2024-Q4" "audit-hold"

# Verify legal hold is active
az storage container legal-hold show \
  --account-name $SA \
  --resource-group $RG \
  --container-name "compliance-records"

# Clear one legal hold tag (data still protected by retention + remaining holds)
az storage container legal-hold clear \
  --account-name $SA \
  --resource-group $RG \
  --container-name "compliance-records" \
  --tags "investigation-2024-Q4"
```

### Verification

```bash
# Verify immutability policy
az storage container immutability-policy show \
  --resource-group $RG \
  --account-name $SA \
  --container-name "compliance-records"

# Verify remaining legal holds
az storage container legal-hold show \
  --account-name $SA \
  --resource-group $RG \
  --container-name "compliance-records" \
  --query "tags"

# Show container properties (hasImmutabilityPolicy, hasLegalHold)
az storage container show \
  --name "compliance-records" \
  --connection-string $CONN \
  --query "{Name:name, ImmutabilityPolicy:properties.hasImmutabilityPolicy, LegalHold:properties.hasLegalHold}"
```

> **SEC 17a-4 Pattern:** For full compliance, lock the retention policy after creation  
> (once locked, it CANNOT be shortened or deleted — only extended):
> ```bash
> # Get the ETag first
> ETAG=$(az storage container immutability-policy show \
>   --resource-group $RG --account-name $SA \
>   --container-name "compliance-records" --query etag -o tsv)
>
> # Lock the policy (IRREVERSIBLE)
> az storage container immutability-policy lock \
>   --resource-group $RG --account-name $SA \
>   --container-name "compliance-records" \
>   --if-match $ETAG
> ```

### Cleanup

```bash
# Note: Cannot delete container with locked immutability policy until retention expires
# For this lab (unlocked policy), clear legal holds first, then delete
az storage container legal-hold clear \
  --account-name $SA --resource-group $RG \
  --container-name "compliance-records" --tags "audit-hold"

rm -f transaction.txt
az group delete --name $RG --yes --no-wait
```

---

## Lab 4: Entra ID RBAC & User Delegation SAS

### Objective
Configure Entra ID (Azure AD) based access to blob storage using RBAC roles, and generate a User Delegation SAS — the most secure SAS type backed by Entra ID credentials rather than storage account keys.

### Key AZ-305 Concepts Practiced
- Entra ID authentication vs Shared Key vs SAS
- Storage RBAC roles (Data Reader, Data Contributor, Data Owner)
- User Delegation SAS (backed by Entra ID — most secure)
- Service SAS vs Account SAS vs User Delegation SAS comparison
- Principle of least privilege for data access

### Steps

```bash
# Variables
RG="rg-storage-lab4"
LOCATION="eastus2"
SA="stlab4rbac$RANDOM"

# Create resource group
az group create --name $RG --location $LOCATION

# Create storage account (allow shared key for comparison, but prefer Entra ID)
az storage account create \
  --name $SA \
  --resource-group $RG \
  --location $LOCATION \
  --sku Standard_LRS \
  --kind StorageV2

# Get current user's Object ID
USER_ID=$(az ad signed-in-user show --query id -o tsv)

# Assign Storage Blob Data Contributor role (read, write, delete blobs)
az role assignment create \
  --assignee $USER_ID \
  --role "Storage Blob Data Contributor" \
  --scope "/subscriptions/$(az account show --query id -o tsv)/resourceGroups/$RG/providers/Microsoft.Storage/storageAccounts/$SA"

echo "Waiting for RBAC propagation (30s)..."
sleep 30

# Create container using Entra ID auth (--auth-mode login)
az storage container create \
  --name "rbac-demo" \
  --account-name $SA \
  --auth-mode login

# Upload blob using Entra ID (no connection string or keys needed!)
echo "Uploaded via Entra ID RBAC" > rbac-upload.txt

az storage blob upload \
  --account-name $SA \
  --container-name "rbac-demo" \
  --file rbac-upload.txt \
  --name "entra-id-blob.txt" \
  --auth-mode login

# Download blob using Entra ID
az storage blob download \
  --account-name $SA \
  --container-name "rbac-demo" \
  --name "entra-id-blob.txt" \
  --file downloaded.txt \
  --auth-mode login

cat downloaded.txt

# Generate User Delegation SAS (MOST SECURE - backed by Entra ID)
# First, get the user delegation key
EXPIRY=$(date -u -d "+1 hour" '+%Y-%m-%dT%H:%MZ' 2>/dev/null || date -u -v+1H '+%Y-%m-%dT%H:%MZ')

USER_DELEGATION_SAS=$(az storage blob generate-sas \
  --account-name $SA \
  --container-name "rbac-demo" \
  --name "entra-id-blob.txt" \
  --permissions r \
  --expiry $EXPIRY \
  --auth-mode login \
  --as-user \
  -o tsv)

echo "User Delegation SAS: ?${USER_DELEGATION_SAS}"

# For comparison: Service SAS (less secure - uses account key)
CONN=$(az storage account show-connection-string \
  --name $SA --resource-group $RG --query connectionString -o tsv)

SERVICE_SAS=$(az storage blob generate-sas \
  --container-name "rbac-demo" \
  --name "entra-id-blob.txt" \
  --permissions r \
  --expiry $EXPIRY \
  --connection-string $CONN \
  -o tsv)

echo "Service SAS: ?${SERVICE_SAS}"
```

### SAS Comparison Table

| SAS Type | Backed By | Revocation | Security Level |
|----------|-----------|------------|----------------|
| **User Delegation** | Entra ID credentials | Revoke delegation key or RBAC | ⭐⭐⭐ Highest |
| **Service SAS** | Storage account key | Rotate key (breaks all SAS) | ⭐⭐ Medium |
| **Account SAS** | Storage account key | Rotate key (breaks all SAS) | ⭐ Lowest |

### Verification

```bash
# Verify role assignment
az role assignment list \
  --assignee $USER_ID \
  --scope "/subscriptions/$(az account show --query id -o tsv)/resourceGroups/$RG/providers/Microsoft.Storage/storageAccounts/$SA" \
  --query "[].{Role:roleDefinitionName, Scope:scope}" -o table

# Verify blob was uploaded via Entra ID
az storage blob list \
  --account-name $SA \
  --container-name "rbac-demo" \
  --auth-mode login \
  --query "[].{Name:name, Size:properties.contentLength}" -o table
```

### Cleanup

```bash
rm -f rbac-upload.txt downloaded.txt
az group delete --name $RG --yes --no-wait
```

---

## Lab 5: Azure File Sync (Hybrid)

### Objective
Set up Azure Files with a Storage Sync Service for hybrid file sharing scenarios. This enables on-premises servers to sync with Azure Files, providing cloud tiering and multi-site access.

### Key AZ-305 Concepts Practiced
- Azure Files SMB/NFS shares for lift-and-shift
- Storage Sync Service architecture (Sync Groups, Cloud/Server Endpoints)
- Cloud tiering for capacity optimization
- Hybrid storage patterns (on-prem + cloud)
- Premium vs Standard file shares

### Steps

```bash
# Variables
RG="rg-storage-lab5"
LOCATION="eastus2"
SA="stlab5files$RANDOM"
SYNC_SERVICE="sync-service-lab5"
SYNC_GROUP="sg-fileshare"

# Create resource group
az group create --name $RG --location $LOCATION

# Create storage account for Azure Files (LRS for lab; use ZRS/GRS for production)
az storage account create \
  --name $SA \
  --resource-group $RG \
  --location $LOCATION \
  --sku Standard_LRS \
  --kind StorageV2

CONN=$(az storage account show-connection-string \
  --name $SA --resource-group $RG --query connectionString -o tsv)

# Create Azure Files share (5 TB quota)
az storage share-rm create \
  --resource-group $RG \
  --storage-account $SA \
  --name "department-share" \
  --quota 5120 \
  --access-tier Hot

# Upload a sample file to the share
echo "Shared department document" > shared-doc.txt

az storage file upload \
  --share-name "department-share" \
  --source shared-doc.txt \
  --connection-string $CONN

# Create directory structure
az storage directory create \
  --share-name "department-share" \
  --name "finance" \
  --connection-string $CONN

az storage directory create \
  --share-name "department-share" \
  --name "engineering" \
  --connection-string $CONN

# Create Storage Sync Service
az resource create \
  --resource-group $RG \
  --resource-type "Microsoft.StorageSync/storageSyncServices" \
  --name $SYNC_SERVICE \
  --location $LOCATION \
  --properties '{}'

# Create Sync Group
az resource create \
  --resource-group $RG \
  --resource-type "Microsoft.StorageSync/storageSyncServices/syncGroups" \
  --name "${SYNC_SERVICE}/${SYNC_GROUP}" \
  --properties '{}'

# Register Cloud Endpoint (connects Azure Files share to sync group)
SA_ID=$(az storage account show --name $SA --resource-group $RG --query id -o tsv)

az resource create \
  --resource-group $RG \
  --resource-type "Microsoft.StorageSync/storageSyncServices/syncGroups/cloudEndpoints" \
  --name "${SYNC_SERVICE}/${SYNC_GROUP}/ce-department-share" \
  --properties "{
    \"storageAccountResourceId\": \"${SA_ID}\",
    \"azureFileShareName\": \"department-share\",
    \"storageAccountTenantId\": \"$(az account show --query tenantId -o tsv)\"
  }"
```

### Server Agent Installation (Discussion)

> **Note:** Server endpoints require installing the Azure File Sync agent on a Windows Server.  
> This cannot be done in Cloud Shell but is documented here for completeness.

```powershell
# On Windows Server (2016/2019/2022):
# 1. Download Azure File Sync agent from Microsoft Download Center
# 2. Install the agent (StorageSyncAgent.msi)
# 3. Register server with Storage Sync Service:
#    - Opens a wizard that authenticates to Azure
#    - Associates the server with your Storage Sync Service
# 4. Create Server Endpoint via portal or CLI:
#
# az storagesync sync-group server-endpoint create \
#   --resource-group $RG \
#   --storage-sync-service $SYNC_SERVICE \
#   --sync-group-name $SYNC_GROUP \
#   --name "server-endpoint-1" \
#   --server-id "<registered-server-id>" \
#   --server-local-path "D:\DepartmentShare" \
#   --cloud-tiering "on" \
#   --volume-free-space-percent 20
```

### Verification

```bash
# Verify file share exists
az storage share-rm list \
  --storage-account $SA --resource-group $RG \
  --query "[].{Name:name, Quota:shareQuota, Tier:accessTier}" -o table

# Verify Storage Sync Service
az resource show \
  --resource-group $RG \
  --resource-type "Microsoft.StorageSync/storageSyncServices" \
  --name $SYNC_SERVICE \
  --query "{Name:name, Location:location, ProvisioningState:properties.provisioningState}"

# List files in share
az storage file list \
  --share-name "department-share" \
  --connection-string $CONN \
  --query "[].{Name:name, Type:type}" -o table
```

### Cleanup

```bash
rm -f shared-doc.txt
az group delete --name $RG --yes --no-wait
```

---

## Lab 6: Data Lake Gen2 with ACLs

### Objective
Create a storage account with Hierarchical Namespace (HNS) enabled for Azure Data Lake Storage Gen2, set up directory structures, and configure POSIX-style ACLs for team-based access isolation.

### Key AZ-305 Concepts Practiced
- Data Lake Storage Gen2 (HNS on Blob Storage)
- Hierarchical namespace for big data analytics
- POSIX ACLs (access + default) for fine-grained control
- Directory-level permissions for team isolation
- Integration with analytics services (Synapse, Databricks, HDInsight)

### Steps

```bash
# Variables
RG="rg-storage-lab6"
LOCATION="eastus2"
SA="stlab6datalake$RANDOM"
FILESYSTEM="analytics"

# Create resource group
az group create --name $RG --location $LOCATION

# Create Data Lake Gen2 account (HNS enabled)
az storage account create \
  --name $SA \
  --resource-group $RG \
  --location $LOCATION \
  --sku Standard_LRS \
  --kind StorageV2 \
  --hns true

# Get current user for RBAC
USER_ID=$(az ad signed-in-user show --query id -o tsv)

# Assign Storage Blob Data Owner role (needed for ACL management)
az role assignment create \
  --assignee $USER_ID \
  --role "Storage Blob Data Owner" \
  --scope "/subscriptions/$(az account show --query id -o tsv)/resourceGroups/$RG/providers/Microsoft.Storage/storageAccounts/$SA"

echo "Waiting for RBAC propagation (30s)..."
sleep 30

# Create filesystem (container in ADLS Gen2 terms)
az storage fs create \
  --name $FILESYSTEM \
  --account-name $SA \
  --auth-mode login

# Create directory hierarchy for team isolation
az storage fs directory create \
  --name "raw" \
  --file-system $FILESYSTEM \
  --account-name $SA \
  --auth-mode login

az storage fs directory create \
  --name "raw/sales" \
  --file-system $FILESYSTEM \
  --account-name $SA \
  --auth-mode login

az storage fs directory create \
  --name "raw/engineering" \
  --file-system $FILESYSTEM \
  --account-name $SA \
  --auth-mode login

az storage fs directory create \
  --name "curated" \
  --file-system $FILESYSTEM \
  --account-name $SA \
  --auth-mode login

az storage fs directory create \
  --name "curated/reports" \
  --file-system $FILESYSTEM \
  --account-name $SA \
  --auth-mode login

# Set POSIX ACLs for team isolation
# Sales team gets rwx on raw/sales (using a sample Object ID)
# In production, replace with actual group Object IDs
SALES_GROUP_ID="00000000-0000-0000-0000-000000000001"  # Placeholder
ENG_GROUP_ID="00000000-0000-0000-0000-000000000002"    # Placeholder

# Set access ACL on raw/sales directory (rwx for sales group)
az storage fs access set \
  --path "raw/sales" \
  --file-system $FILESYSTEM \
  --account-name $SA \
  --auth-mode login \
  --acl "user::rwx,group::r-x,group:${SALES_GROUP_ID}:rwx,other::---"

# Set default ACL (inherited by new files/subdirectories)
az storage fs access set \
  --path "raw/sales" \
  --file-system $FILESYSTEM \
  --account-name $SA \
  --auth-mode login \
  --acl "default:user::rwx,default:group::r-x,default:group:${SALES_GROUP_ID}:rwx,default:other::---"

# Set access ACL on raw/engineering directory (rwx for engineering group)
az storage fs access set \
  --path "raw/engineering" \
  --file-system $FILESYSTEM \
  --account-name $SA \
  --auth-mode login \
  --acl "user::rwx,group::r-x,group:${ENG_GROUP_ID}:rwx,other::---"

# Upload sample data
echo "sales_region,amount\nnorth,50000\nsouth,42000" > sales-data.csv

az storage fs file upload \
  --file-system $FILESYSTEM \
  --path "raw/sales/q4-data.csv" \
  --source sales-data.csv \
  --account-name $SA \
  --auth-mode login
```

### Verification

```bash
# Verify HNS is enabled
az storage account show --name $SA --resource-group $RG \
  --query "{Name:name, HNS:isHnsEnabled, Kind:kind}" -o table

# List directory structure
az storage fs file list \
  --file-system $FILESYSTEM \
  --account-name $SA \
  --auth-mode login \
  --query "[].{Name:name, IsDirectory:isDirectory}" -o table

# Check ACLs on sales directory
az storage fs access show \
  --path "raw/sales" \
  --file-system $FILESYSTEM \
  --account-name $SA \
  --auth-mode login

# Check ACLs on engineering directory
az storage fs access show \
  --path "raw/engineering" \
  --file-system $FILESYSTEM \
  --account-name $SA \
  --auth-mode login
```

### Cleanup

```bash
rm -f sales-data.csv
az group delete --name $RG --yes --no-wait
```

---

## Lab 7: Customer-Managed Keys with Key Vault

### Objective
Configure storage account encryption using Customer-Managed Keys (CMK) stored in Azure Key Vault, with a system-assigned managed identity for key access. This provides full control over the encryption key lifecycle.

### Key AZ-305 Concepts Practiced
- Encryption at rest: Microsoft-managed vs Customer-managed keys
- Key Vault with purge protection (required for CMK)
- Managed identity for service-to-service authentication
- Key Vault Crypto Service Encryption User role
- Infrastructure encryption (double encryption)

### Steps

```bash
# Variables
RG="rg-storage-lab7"
LOCATION="eastus2"
SA="stlab7cmk$RANDOM"
KV="kv-lab7-$RANDOM"
KEY_NAME="storage-encryption-key"

# Create resource group
az group create --name $RG --location $LOCATION

# Create Key Vault with purge protection (REQUIRED for CMK)
az keyvault create \
  --name $KV \
  --resource-group $RG \
  --location $LOCATION \
  --enable-purge-protection true \
  --enable-rbac-authorization true \
  --retention-days 7

# Assign Key Vault Crypto Officer role to current user (to create keys)
USER_ID=$(az ad signed-in-user show --query id -o tsv)
KV_ID=$(az keyvault show --name $KV --resource-group $RG --query id -o tsv)

az role assignment create \
  --assignee $USER_ID \
  --role "Key Vault Crypto Officer" \
  --scope $KV_ID

echo "Waiting for RBAC propagation (30s)..."
sleep 30

# Create RSA 2048-bit key for storage encryption
az keyvault key create \
  --vault-name $KV \
  --name $KEY_NAME \
  --kty RSA \
  --size 2048

# Get key URI (without version for auto-rotation)
KEY_URI=$(az keyvault key show --vault-name $KV --name $KEY_NAME \
  --query "key.kid" -o tsv | sed 's|/[^/]*$||')

echo "Key URI (versionless): $KEY_URI"

# Create storage account with system-assigned managed identity
az storage account create \
  --name $SA \
  --resource-group $RG \
  --location $LOCATION \
  --sku Standard_LRS \
  --kind StorageV2 \
  --identity-type SystemAssigned

# Get the managed identity principal ID
SA_PRINCIPAL=$(az storage account show --name $SA --resource-group $RG \
  --query "identity.principalId" -o tsv)

echo "Storage Account Identity: $SA_PRINCIPAL"

# Assign Key Vault Crypto Service Encryption User role to storage identity
az role assignment create \
  --assignee $SA_PRINCIPAL \
  --role "Key Vault Crypto Service Encryption User" \
  --scope $KV_ID

echo "Waiting for RBAC propagation (30s)..."
sleep 30

# Configure CMK encryption on storage account
KEY_URI_FULL=$(az keyvault key show --vault-name $KV --name $KEY_NAME \
  --query "key.kid" -o tsv)

az storage account update \
  --name $SA \
  --resource-group $RG \
  --encryption-key-source Microsoft.Keyvault \
  --encryption-key-vault "https://${KV}.vault.azure.net" \
  --encryption-key-name $KEY_NAME \
  --key-vault-identity ""
```

### Verification

```bash
# Verify encryption configuration
az storage account show --name $SA --resource-group $RG \
  --query "{Name:name, KeySource:encryption.keySource, KeyVault:encryption.keyVaultProperties.keyVaultUri, KeyName:encryption.keyVaultProperties.keyName}" -o table

# Verify Key Vault key exists
az keyvault key show --vault-name $KV --name $KEY_NAME \
  --query "{Name:name, KeyType:key.kty, KeySize:key.n}" -o table

# Verify managed identity
az storage account show --name $SA --resource-group $RG \
  --query "identity.{Type:type, PrincipalId:principalId, TenantId:tenantId}" -o table

# Verify role assignment
az role assignment list \
  --assignee $SA_PRINCIPAL --scope $KV_ID \
  --query "[].{Role:roleDefinitionName}" -o table
```

### Cleanup

```bash
az group delete --name $RG --yes --no-wait
```

---

## Lab 8: Blob Versioning & Point-in-Time Restore

### Objective
Enable blob versioning, change feed, and point-in-time restore to protect against accidental modifications and deletions. Demonstrate version history and container-level restore capabilities.

### Key AZ-305 Concepts Practiced
- Blob versioning for automatic version history
- Change feed for auditing blob changes
- Point-in-time restore (requires versioning + change feed + soft delete)
- Data protection strategy layers
- RPO considerations for backup/restore

### Steps

```bash
# Variables
RG="rg-storage-lab8"
LOCATION="eastus2"
SA="stlab8versions$RANDOM"

# Create resource group
az group create --name $RG --location $LOCATION

# Create storage account
az storage account create \
  --name $SA \
  --resource-group $RG \
  --location $LOCATION \
  --sku Standard_LRS \
  --kind StorageV2

# Enable blob soft delete (required for point-in-time restore)
az storage account blob-service-properties update \
  --account-name $SA \
  --resource-group $RG \
  --delete-retention-days 14 \
  --enable-delete-retention true

# Enable container soft delete
az storage account blob-service-properties update \
  --account-name $SA \
  --resource-group $RG \
  --container-delete-retention-days 7 \
  --enable-container-delete-retention true

# Enable versioning
az storage account blob-service-properties update \
  --account-name $SA \
  --resource-group $RG \
  --enable-versioning true

# Enable change feed
az storage account blob-service-properties update \
  --account-name $SA \
  --resource-group $RG \
  --enable-change-feed true

# Enable point-in-time restore (30-day window)
az storage account blob-service-properties update \
  --account-name $SA \
  --resource-group $RG \
  --enable-restore-policy true \
  --restore-days 30

CONN=$(az storage account show-connection-string \
  --name $SA --resource-group $RG --query connectionString -o tsv)

# Create container
az storage container create --name "documents" --connection-string $CONN

# Upload initial version of a blob
echo "Version 1: Original document content" > doc.txt
az storage blob upload --container-name "documents" \
  --file doc.txt --name "important-doc.txt" \
  --connection-string $CONN --overwrite

echo "Sleeping 5s between versions..."
sleep 5

# Overwrite with version 2
echo "Version 2: Updated with corrections" > doc.txt
az storage blob upload --container-name "documents" \
  --file doc.txt --name "important-doc.txt" \
  --connection-string $CONN --overwrite

sleep 5

# Overwrite with version 3
echo "Version 3: Final approved version" > doc.txt
az storage blob upload --container-name "documents" \
  --file doc.txt --name "important-doc.txt" \
  --connection-string $CONN --overwrite

sleep 5

# Overwrite with version 4 (accidental bad data)
echo "Version 4: OOPS - this was an accidental overwrite with bad data!" > doc.txt
az storage blob upload --container-name "documents" \
  --file doc.txt --name "important-doc.txt" \
  --connection-string $CONN --overwrite

# Record timestamp for potential restore
RESTORE_TIME=$(date -u '+%Y-%m-%dT%H:%M:%SZ')
echo "Restore point: $RESTORE_TIME"

# List all versions of the blob
echo "--- All Versions ---"
az storage blob list --container-name "documents" \
  --connection-string $CONN \
  --include v \
  --query "[?name=='important-doc.txt'].{Name:name, VersionId:versionId, IsCurrentVersion:isCurrentVersion}" -o table

# Download a previous version (get version ID from list above)
PREV_VERSION=$(az storage blob list --container-name "documents" \
  --connection-string $CONN --include v \
  --query "[?name=='important-doc.txt' && isCurrentVersion==null].versionId | [0]" -o tsv)

if [ -n "$PREV_VERSION" ]; then
  az storage blob download --container-name "documents" \
    --name "important-doc.txt" \
    --version-id $PREV_VERSION \
    --file restored-version.txt \
    --connection-string $CONN
  echo "--- Previous version content ---"
  cat restored-version.txt
fi

# Point-in-time restore (restore entire container to before bad overwrite)
# Note: Requires a few minutes after enabling PITR before first restore works
echo "--- Attempting Point-in-Time Restore ---"
echo "Note: PITR needs a few minutes after initial enable before it can be used."
echo "In production, you would run:"
echo "az storage blob restore --account-name $SA --resource-group $RG \\"
echo "  --time-to-restore <timestamp-before-bad-write>"
```

### Verification

```bash
# Verify all data protection features are enabled
az storage account blob-service-properties show \
  --account-name $SA --resource-group $RG \
  --query "{Versioning:isVersioningEnabled, ChangeFeed:changeFeed.enabled, SoftDelete:deleteRetentionPolicy.enabled, SoftDeleteDays:deleteRetentionPolicy.days, PITR:restorePolicy.enabled, PITRDays:restorePolicy.days, ContainerSoftDelete:containerDeleteRetentionPolicy.enabled}" -o table

# Show current blob content
az storage blob download --container-name "documents" \
  --name "important-doc.txt" --file current.txt \
  --connection-string $CONN --no-progress
echo "Current content:"
cat current.txt

# Count versions
VERSION_COUNT=$(az storage blob list --container-name "documents" \
  --connection-string $CONN --include v \
  --query "[?name=='important-doc.txt'] | length(@)" -o tsv)
echo "Total versions (including current): $VERSION_COUNT"
```

### Cleanup

```bash
rm -f doc.txt restored-version.txt current.txt
az group delete --name $RG --yes --no-wait
```

---

## Summary: AZ-305 Storage Decision Framework

| Scenario | Solution | Key Feature |
|----------|----------|-------------|
| Cost optimization over time | Lifecycle Management | Auto-tier Hot→Cool→Archive |
| Network isolation | Private Endpoints | No public internet exposure |
| Regulatory compliance | Immutable Storage (WORM) | Time-based retention + Legal holds |
| Secure delegated access | User Delegation SAS | Entra ID-backed, revocable |
| Hybrid file sharing | Azure File Sync | Cloud tiering + multi-site sync |
| Big data analytics | ADLS Gen2 (HNS) | POSIX ACLs + hierarchical namespace |
| Key control & compliance | Customer-Managed Keys | Key Vault + managed identity |
| Data protection & recovery | Versioning + PITR | Point-in-time restore capability |
