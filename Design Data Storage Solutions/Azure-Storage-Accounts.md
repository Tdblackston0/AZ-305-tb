# Azure Storage Accounts – AZ-305 Cheat Sheet

> **Perspective:** Senior Cloud Solution Architect preparing for AZ-305  
> **Focus:** Design decisions, trade-offs, and when to use what

---

## Table of Contents

1. [Storage Account Types](#storage-account-types)
2. [Storage Services Overview](#storage-services-overview)
3. [Managed Disks & Azure File Sync Design](#managed-disks--azure-file-sync-design)
4. [Performance Tiers](#performance-tiers)
5. [Access Tiers (Blob)](#access-tiers-blob)
6. [Redundancy Options](#redundancy-options)
7. [Security & Access Control](#security--access-control)
8. [Networking](#networking)
9. [Data Protection & Lifecycle](#data-protection--lifecycle)
10. [Decision Scenarios (AZ-305 Style)](#decision-scenarios-az-305-style)
11. [Pricing Considerations](#pricing-considerations)
12. [Hands-On Labs](#hands-on-labs)

---

## Storage Account Types

| Type | Use Case | Services Supported |
|------|----------|-------------------|
| **Standard general-purpose v2** | Most scenarios (default choice) | Blob, File, Queue, Table, Data Lake |
| **Premium Block Blobs** | Low-latency workloads, high transaction rates | Block blobs, Append blobs |
| **Premium File Shares** | Enterprise file shares needing high IOPS | Azure Files (SMB/NFS) |
| **Premium Page Blobs** | Unmanaged VM disks, databases | Page blobs only |

### AZ-305 Decision Rule
> **Always start with Standard GPv2** unless the scenario explicitly requires:
> - Sub-millisecond latency → Premium Block Blobs
> - High-performance file shares → Premium File Shares
> - Legacy unmanaged disks → Premium Page Blobs

---

## Storage Services Overview

### Blob Storage
- **Block Blobs** – Documents, images, video, backups (up to 190.7 TiB)
- **Append Blobs** – Log files, streaming data (optimized for append operations)
- **Page Blobs** – Random read/write, VHDs (up to 8 TiB)

### Azure Files
- Fully managed SMB (445) / NFS (2049) file shares
- Lift-and-shift of on-prem file servers
- Azure File Sync for hybrid scenarios

### Queue Storage
- Decoupling components, async messaging
- Messages up to 64 KB, queue unlimited size
- Simpler alternative to Service Bus for basic scenarios

### Table Storage
- NoSQL key-value store
- Semi-structured data at massive scale
- Consider Cosmos DB Table API for global distribution

### Data Lake Storage Gen2
- Hierarchical namespace on top of Blob storage
- Big data analytics (Spark, Databricks, Synapse)
- POSIX-compatible ACLs

---

<a id="managed-disks--azure-file-sync-design"></a>
## Managed Disks & Azure File Sync Design

### Azure Managed Disks (IaaS VM Storage)

| Disk Type | Performance Profile | Best For | AZ-305 Design Notes |
|-----------|---------------------|----------|---------------------|
| **Standard HDD** | Lowest cost, baseline performance | Dev/test, infrequent access workloads | Lowest cost option when performance is not critical |
| **Standard SSD** | Consistent baseline latency | Web apps, lightly used production workloads | Better latency than HDD at moderate cost |
| **Premium SSD** | High IOPS and low latency | Production OLTP, enterprise apps | Default for most production VM disks |
| **Premium SSD v2** | Independent scale of IOPS/throughput | Workloads with variable performance demand | Cost-optimize by sizing IOPS separately from capacity |
| **Ultra Disk** | Highest IOPS/throughput, sub-ms latency | SAP HANA, top-tier databases | Use only for extreme performance requirements |

### Managed Disk Roles
- **OS disk:** Boot/system volume for a VM
- **Data disk:** Persistent application and database data
- **Temporary disk:** Ephemeral local disk; do not store durable data

### Managed Disk Security & Availability
- Encryption at rest with **platform-managed keys (PMK)** by default
- **Customer-managed keys (CMK)** supported for stricter compliance requirements
- **Disk Encryption Sets (DES)** centralize CMK assignment for fleets
- Use **zone-redundant disks** where supported for higher intra-region resilience
- Pair disks with VM backup and snapshot strategy for restore requirements

### Azure File Sync Design Guidance
- Use Azure File Sync when you need **ongoing hybrid synchronization**, not one-time migration
- Keep **one cloud endpoint per sync group** and scale with multiple server endpoints
- Enable **cloud tiering** to preserve local cache for hot files while keeping full data set in Azure Files
- Use **volume free space policy** plus **date-based policy** to control local cache behavior
- Deploy Storage Sync Service close to primary users and validate bandwidth/latency for branch offices

### Azure File Sync IaC Starters

```bicep
param location string = resourceGroup().location
param syncServiceName string

resource syncService 'Microsoft.StorageSync/storageSyncServices@2022-09-01' = {
  name: syncServiceName
  location: location
}
```

```hcl
resource "azurerm_storage_sync" "sync" {
  name                = var.storage_sync_service_name
  resource_group_name = var.resource_group_name
  location            = var.location
}
```

> 🎯 **AZ-305 Tip:** If a scenario asks for branch office local performance + centralized cloud data + minimal user disruption, Azure File Sync is usually the best answer.

---

## Performance Tiers

| Tier | Backed By | Latency | Max IOPS (single blob) | Use Case |
|------|-----------|---------|------------------------|----------|
| **Standard** | HDD | Single-digit ms | 500 | General purpose |
| **Premium** | SSD | Sub-millisecond | 20,000 | IoT telemetry, AI/ML, gaming |

### Key Differences
- Premium does **not** support access tiers (Hot/Cool/Cold/Archive)
- Premium is **locally redundant (LRS)** or **zone-redundant (ZRS)** only
- Premium billing: provisioned capacity (not per-transaction like Standard)

---

## Access Tiers (Blob)

| Tier | Storage Cost | Access Cost | Min Retention | Use Case |
|------|-------------|-------------|---------------|----------|
| **Hot** | Highest | Lowest | None | Frequently accessed data |
| **Cool** | Lower | Higher | 30 days | Infrequent access (backups, older content) |
| **Cold** | Even Lower | Even Higher | 90 days | Rarely accessed, but needs availability |
| **Archive** | Lowest | Highest | 180 days | Compliance, long-term retention |

### Rehydration from Archive
| Priority | Time | Cost |
|----------|------|------|
| Standard | Up to 15 hours | Lower |
| High | < 1 hour (for blobs < 10 GB) | Higher |

### AZ-305 Tips
- **Account-level default tier:** Hot or Cool only (not Cold/Archive)
- **Blob-level tier:** Can be set individually on any blob
- **Lifecycle Management Policies** automate tier transitions
- Archive blobs are **offline** – must rehydrate before reading

---

## Redundancy Options

| Option | Copies | Scope | Durability (11 nines) | Use Case |
|--------|--------|-------|----------------------|----------|
| **LRS** | 3 | Single datacenter | 99.999999999% | Dev/test, non-critical data |
| **ZRS** | 3 | 3 Availability Zones | 99.9999999999% | High availability within a region |
| **GRS** | 6 | Primary + Secondary region | 99.99999999999999% | DR protection, data must survive region failure |
| **GZRS** | 6 | 3 AZs + Secondary region | 99.99999999999999% | **Maximum durability & availability** |
| **RA-GRS** | 6 | Same as GRS + read from secondary | Same as GRS | Read access during primary outage |
| **RA-GZRS** | 6 | Same as GZRS + read from secondary | Same as GZRS | Best RPO + read during failover |

### AZ-305 Decision Matrix

```
Is this production data?
├── No → LRS (cheapest)
├── Yes → Does it need to survive a datacenter failure?
│   ├── No → LRS
│   ├── Yes → Does it need to survive a region failure?
│       ├── No → ZRS
│       ├── Yes → Do you need read access during failover?
│           ├── No → GRS or GZRS
│           └── Yes → RA-GRS or RA-GZRS
```

### Failover Notes
- **RPO for geo-redundancy:** ~15 minutes (async replication)
- **Customer-managed failover:** You initiate, secondary becomes primary
- **Microsoft-managed failover:** Only in true regional disaster
- After failover, storage account becomes LRS in the new primary → must reconfigure

---

## Security & Access Control

### Authentication Methods (in order of preference)

| Method | When to Use | AZ-305 Priority |
|--------|-------------|-----------------|
| **Microsoft Entra ID + RBAC** | Applications, users, managed identities | ⭐ **Preferred** |
| **Shared Access Signatures (SAS)** | Time-limited delegated access to external parties | Second choice |
| **Storage Account Keys** | Legacy/admin scenarios only | Avoid in production |

### RBAC Roles (Most Common)

| Role | Scope |
|------|-------|
| Storage Blob Data Owner | Full access to blob data + ACL management |
| Storage Blob Data Contributor | Read/write/delete blobs |
| Storage Blob Data Reader | Read-only access to blobs |
| Storage Blob Delegator | Generate user delegation SAS |
| Storage File Data SMB Share Contributor | Read/write/delete on file shares |
| Storage Queue Data Contributor | Read/write/delete queue messages |
| Storage Table Data Contributor | Read/write/delete table entities |

### SAS Token Types

| Type | Signed With | Scope | Best For |
|------|-------------|-------|----------|
| **User Delegation SAS** | Entra ID credentials | Blob only | ⭐ Most secure SAS |
| **Service SAS** | Account key | Single service | Scoped access |
| **Account SAS** | Account key | Multiple services | Broader access |

### Stored Access Policies
- Attach to a container/queue/table/share
- Can revoke SAS tokens by modifying or deleting the policy
- **Cannot be used with User Delegation SAS**

### Encryption

| Layer | Default | Customizable |
|-------|---------|-------------|
| **At rest** | Microsoft-managed keys (SSE) | Customer-managed keys (CMK) in Key Vault |
| **In transit** | TLS 1.2 enforced | Can require minimum TLS version |
| **Infrastructure encryption** | Disabled | Double encryption (enable at creation) |

### AZ-305 Exam Scenario Tips
- "Comply with regulatory requirement for customer-controlled keys" → **CMK with Key Vault**
- "Ensure data is encrypted with two different algorithms" → **Infrastructure encryption**
- "Revoke access immediately" → **Stored Access Policy** (for Service SAS) or **rotate keys**
- "Least privilege access for an app" → **Managed Identity + RBAC**

---

## Networking

### Access Methods

| Method | Description | When to Use |
|--------|-------------|-------------|
| **Public endpoint** | Accessible from internet | Default; can restrict by IP/VNet |
| **Service Endpoint** | Optimal route from VNet to storage | Simple VNet integration |
| **Private Endpoint** | Private IP in your VNet | Zero internet exposure, compliance |
| **Azure Private Link** | Cross-tenant, cross-region private access | Multi-tenant SaaS scenarios |

### Firewall Rules
- Default: Allow all networks
- Can restrict to: specific VNets, IP ranges, resource instances, Azure services

### AZ-305 Decision
> **Private Endpoint** is almost always the correct answer when the scenario mentions:
> - "No data traverses the public internet"
> - "Comply with data residency / privacy requirements"
> - "Secure access from on-premises via ExpressRoute/VPN"

---

## Data Protection & Lifecycle

### Soft Delete
| Resource | Default Retention | Protects Against |
|----------|------------------|-----------------|
| Blobs | 7 days (configurable 1-365) | Accidental deletion/overwrite |
| Containers | 7 days | Accidental container deletion |
| File Shares | 7 days | Share deletion |

### Versioning & Snapshots
- **Blob Versioning:** Automatic previous version retention on every overwrite
- **Snapshots:** Point-in-time manual or automated captures
- **Point-in-time Restore:** Restore block blobs to a previous state (requires versioning + change feed)

### Immutable Storage (WORM)
| Policy Type | Can Be Deleted? | Use Case |
|-------------|----------------|----------|
| **Time-based retention** | Cannot delete/modify until retention expires | Compliance (SEC 17a-4, HIPAA) |
| **Legal Hold** | Cannot delete until all holds removed | Litigation, investigations |

### Lifecycle Management Policies
```json
{
  "rules": [
    {
      "name": "MoveToArchive",
      "type": "Lifecycle",
      "definition": {
        "filters": { "blobTypes": ["blockBlob"], "prefixMatch": ["logs/"] },
        "actions": {
          "baseBlob": {
            "tierToCool": { "daysAfterModificationGreaterThan": 30 },
            "tierToCold": { "daysAfterModificationGreaterThan": 90 },
            "tierToArchive": { "daysAfterModificationGreaterThan": 180 },
            "delete": { "daysAfterModificationGreaterThan": 365 }
          }
        }
      }
    }
  ]
}
```

---

## Decision Scenarios (AZ-305 Style)

### Scenario 1: Media Company Video Platform
**Requirement:** Store petabytes of video files. Recent content accessed frequently; older content rarely accessed; legal requirement to keep all content for 7 years.

**Solution:**
- Standard GPv2, Hot tier (default)
- Lifecycle policy: Cool after 30 days → Cold after 90 days → Archive after 1 year
- Immutable storage with 7-year time-based retention
- RA-GRS for read access during regional outage
- Private endpoint for internal app access
- CDN integration for public streaming

---

### Scenario 2: Financial Services Transaction Logs
**Requirement:** High-frequency writes, must be immutable, auditors need read access, comply with SEC 17a-4.

**Solution:**
- Premium Block Blobs (low-latency writes)
- Append Blobs (write-once log pattern)
- WORM with time-based retention (SEC 17a-4 compliance)
- Entra ID + RBAC (Storage Blob Data Reader for auditors)
- ZRS for zone redundancy
- Private endpoint + no public access
- CMK for encryption at rest

---

### Scenario 3: Hybrid File Server Migration
**Requirement:** 50 TB of files on-prem, branch offices need local caching, central management in Azure.

**Solution:**
- Azure Files (Standard, SMB)
- **Azure File Sync** agents on branch office servers
- Cloud tiering enabled (frequently accessed files cached locally)
- GRS for geo-redundancy
- Identity-based authentication (on-prem AD DS joined)
- Private endpoint for Azure connectivity

---

### Scenario 4: Data Lake for Analytics
**Requirement:** Multiple teams need to query raw data, enforce team-level access, integrate with Databricks and Synapse.

**Solution:**
- Storage account with **Hierarchical Namespace** enabled (ADLS Gen2)
- Hot tier for recent data, lifecycle policy for aging
- POSIX ACLs for folder-level team isolation
- Entra ID + RBAC for service principals
- ZRS or GZRS based on criticality
- Service endpoint from analytics VNet

---

### Scenario 5: IoT Telemetry Ingestion
**Requirement:** 100,000 messages/second from devices, real-time processing, retain raw data for 90 days.

**Solution:**
- Premium Block Blobs (high IOPS, low latency)
- LRS (data can be re-sent from devices if needed)
- Lifecycle policy: delete after 90 days
- Managed identity on processing service
- Event Grid for blob-created triggers
- Private endpoint from IoT processing VNet

---

### Scenario 6: Backup & Disaster Recovery
**Requirement:** Daily backups of databases, 30-day retention, must survive regional failure, RTO of 4 hours.

**Solution:**
- Standard GPv2
- Cool tier (infrequent access, cost-efficient)
- RA-GRS (read access from secondary for DR)
- Soft delete with 30-day retention
- Blob versioning enabled
- Lifecycle: delete after 30 days
- Customer-managed failover documented in DR runbook

---

## Pricing Considerations

### Cost Optimization Strategies
1. **Right-size redundancy** – Don't use GRS for data that's easily recreated
2. **Lifecycle policies** – Automate tier transitions aggressively
3. **Reserved capacity** – 1-year or 3-year reservations for predictable workloads (up to 72% savings)
4. **Choose correct tier** – Archive is 10x cheaper than Hot for storage
5. **Monitor transactions** – Cool/Cold/Archive have higher per-transaction costs
6. **Avoid early deletion penalties** – Respect minimum retention periods

### Cost Comparison (approx. per GB/month, East US)

| Tier | LRS Storage | GRS Storage |
|------|-------------|-------------|
| Hot | $0.018 | $0.036 |
| Cool | $0.01 | $0.02 |
| Cold | $0.0036 | $0.0072 |
| Archive | $0.00099 | $0.002 |

---

## Hands-On Labs

### Lab 1: Storage Account with Lifecycle Management
**Objective:** Create a storage account and configure automatic tiering.

```powershell
# Variables
$rg = "rg-az305-storage-lab1"
$location = "eastus2"
$storageAccount = "staz305lab1$(Get-Random -Maximum 9999)"

# Create Resource Group
az group create --name $rg --location $location

# Create Storage Account (Standard GPv2, Hot, GRS)
az storage account create `
  --name $storageAccount `
  --resource-group $rg `
  --location $location `
  --sku Standard_GRS `
  --kind StorageV2 `
  --access-tier Hot `
  --min-tls-version TLS1_2

# Create container
az storage container create `
  --name "documents" `
  --account-name $storageAccount `
  --auth-mode login

# Upload sample blobs
az storage blob upload `
  --account-name $storageAccount `
  --container-name "documents" `
  --name "reports/2024/annual-report.pdf" `
  --file "./sample.txt" `
  --auth-mode login

# Apply Lifecycle Policy
az storage account management-policy create `
  --account-name $storageAccount `
  --resource-group $rg `
  --policy '@lifecycle-policy.json'

# Verify
az storage account management-policy show `
  --account-name $storageAccount `
  --resource-group $rg
```

**Cleanup:**
```powershell
az group delete --name $rg --yes --no-wait
```

---

### Lab 2: Private Endpoint & Firewall Configuration
**Objective:** Secure a storage account with Private Endpoint and deny public access.

```powershell
$rg = "rg-az305-storage-lab2"
$location = "eastus2"
$storageAccount = "staz305lab2$(Get-Random -Maximum 9999)"
$vnetName = "vnet-lab2"
$subnetName = "snet-storage"
$peName = "pe-storage-lab2"

# Create RG and VNet
az group create --name $rg --location $location
az network vnet create `
  --resource-group $rg `
  --name $vnetName `
  --address-prefix 10.0.0.0/16 `
  --subnet-name $subnetName `
  --subnet-prefixes 10.0.1.0/24

# Disable subnet private endpoint policies
az network vnet subnet update `
  --resource-group $rg `
  --vnet-name $vnetName `
  --name $subnetName `
  --disable-private-endpoint-network-policies true

# Create Storage Account
az storage account create `
  --name $storageAccount `
  --resource-group $rg `
  --location $location `
  --sku Standard_LRS `
  --kind StorageV2 `
  --public-network-access Disabled

# Create Private Endpoint
az network private-endpoint create `
  --resource-group $rg `
  --name $peName `
  --vnet-name $vnetName `
  --subnet $subnetName `
  --private-connection-resource-id $(az storage account show --name $storageAccount --resource-group $rg --query id -o tsv) `
  --group-id blob `
  --connection-name "storage-blob-connection"

# Create Private DNS Zone
az network private-dns zone create `
  --resource-group $rg `
  --name "privatelink.blob.core.windows.net"

az network private-dns link vnet create `
  --resource-group $rg `
  --zone-name "privatelink.blob.core.windows.net" `
  --name "storage-dns-link" `
  --virtual-network $vnetName `
  --registration-enabled false

az network private-endpoint dns-zone-group create `
  --resource-group $rg `
  --endpoint-name $peName `
  --name "storage-dns-group" `
  --private-dns-zone "privatelink.blob.core.windows.net" `
  --zone-name "blob"

# Verify - should resolve to private IP
nslookup "$storageAccount.blob.core.windows.net"
```

**Cleanup:**
```powershell
az group delete --name $rg --yes --no-wait
```

---

### Lab 3: Immutable Storage (WORM) for Compliance
**Objective:** Configure immutable storage with time-based retention.

```powershell
$rg = "rg-az305-storage-lab3"
$location = "eastus2"
$storageAccount = "staz305lab3$(Get-Random -Maximum 9999)"

# Create Resources
az group create --name $rg --location $location
az storage account create `
  --name $storageAccount `
  --resource-group $rg `
  --location $location `
  --sku Standard_GRS `
  --kind StorageV2

# Create container with immutability
az storage container create `
  --name "compliance-data" `
  --account-name $storageAccount `
  --auth-mode login

# Set immutability policy (7-day retention for lab, use years in production)
az storage container immutability-policy create `
  --account-name $storageAccount `
  --container-name "compliance-data" `
  --period 7 `
  --resource-group $rg

# Upload a blob
az storage blob upload `
  --account-name $storageAccount `
  --container-name "compliance-data" `
  --name "audit-log-2024.csv" `
  --file "./sample.txt" `
  --auth-mode login

# Try to delete (should fail!)
az storage blob delete `
  --account-name $storageAccount `
  --container-name "compliance-data" `
  --name "audit-log-2024.csv" `
  --auth-mode login
# Expected: This operation is not permitted as the blob is immutable

# Add legal hold
az storage container legal-hold set `
  --account-name $storageAccount `
  --container-name "compliance-data" `
  --tags "case-2024-001" `
  --resource-group $rg
```

**Cleanup:**
```powershell
# Must clear legal hold and wait for retention to expire before deletion
az storage container legal-hold clear `
  --account-name $storageAccount `
  --container-name "compliance-data" `
  --tags "case-2024-001" `
  --resource-group $rg

# Wait for immutability policy to expire, then:
az group delete --name $rg --yes --no-wait
```

---

### Lab 4: Entra ID RBAC & User Delegation SAS
**Objective:** Configure Entra ID authentication and generate a User Delegation SAS.

```powershell
$rg = "rg-az305-storage-lab4"
$location = "eastus2"
$storageAccount = "staz305lab4$(Get-Random -Maximum 9999)"

# Create Resources
az group create --name $rg --location $location
az storage account create `
  --name $storageAccount `
  --resource-group $rg `
  --location $location `
  --sku Standard_LRS `
  --kind StorageV2 `
  --allow-blob-public-access false

# Assign RBAC role (Storage Blob Data Contributor to yourself)
$userId = az ad signed-in-user show --query id -o tsv
az role assignment create `
  --assignee $userId `
  --role "Storage Blob Data Contributor" `
  --scope "/subscriptions/$(az account show --query id -o tsv)/resourceGroups/$rg/providers/Microsoft.Storage/storageAccounts/$storageAccount"

# Wait for role propagation (~30 seconds)
Start-Sleep -Seconds 30

# Create container using Entra auth
az storage container create `
  --name "secure-data" `
  --account-name $storageAccount `
  --auth-mode login

# Upload blob using Entra auth
az storage blob upload `
  --account-name $storageAccount `
  --container-name "secure-data" `
  --name "confidential.txt" `
  --data "This is confidential data" `
  --auth-mode login

# Generate User Delegation SAS (most secure SAS type)
$expiry = (Get-Date).AddHours(1).ToUniversalTime().ToString("yyyy-MM-ddTHH:mm:ssZ")
az storage blob generate-sas `
  --account-name $storageAccount `
  --container-name "secure-data" `
  --name "confidential.txt" `
  --permissions r `
  --expiry $expiry `
  --auth-mode login `
  --as-user `
  --full-uri
```

**Cleanup:**
```powershell
az group delete --name $rg --yes --no-wait
```

---

### Lab 5: Azure File Sync (Hybrid Storage)
**Objective:** Set up Azure Files with File Sync for hybrid scenarios.

```powershell
$rg = "rg-az305-storage-lab5"
$location = "eastus2"
$storageAccount = "staz305lab5$(Get-Random -Maximum 9999)"
$syncServiceName = "afs-lab5"

# Create Resources
az group create --name $rg --location $location
az storage account create `
  --name $storageAccount `
  --resource-group $rg `
  --location $location `
  --sku Standard_GRS `
  --kind StorageV2

# Create File Share (Premium for production; Standard for lab)
az storage share-rm create `
  --storage-account $storageAccount `
  --resource-group $rg `
  --name "department-share" `
  --quota 100

# Create Storage Sync Service
az storagesync create `
  --resource-group $rg `
  --name $syncServiceName `
  --location $location

# Create Sync Group
az storagesync sync-group create `
  --resource-group $rg `
  --storage-sync-service $syncServiceName `
  --name "sg-department"

# Register Cloud Endpoint
az storagesync sync-group cloud-endpoint create `
  --resource-group $rg `
  --storage-sync-service $syncServiceName `
  --sync-group-name "sg-department" `
  --name "cloud-endpoint" `
  --storage-account-resource-id $(az storage account show --name $storageAccount --resource-group $rg --query id -o tsv) `
  --azure-file-share-name "department-share"

# NOTE: Server endpoint requires Azure File Sync agent installed on Windows Server
# Download agent from: https://aka.ms/afs/agent
# Register server, then create server endpoint:
# az storagesync sync-group server-endpoint create ...
```

**Cleanup:**
```powershell
az group delete --name $rg --yes --no-wait
```

---

### Lab 6: Data Lake Storage Gen2 with ACLs
**Objective:** Create ADLS Gen2 and configure directory-level access.

```powershell
$rg = "rg-az305-storage-lab6"
$location = "eastus2"
$storageAccount = "staz305lab6$(Get-Random -Maximum 9999)"

# Create Storage Account with HNS (Hierarchical Namespace)
az group create --name $rg --location $location
az storage account create `
  --name $storageAccount `
  --resource-group $rg `
  --location $location `
  --sku Standard_ZRS `
  --kind StorageV2 `
  --hns true

# Assign RBAC
$userId = az ad signed-in-user show --query id -o tsv
az role assignment create `
  --assignee $userId `
  --role "Storage Blob Data Owner" `
  --scope "/subscriptions/$(az account show --query id -o tsv)/resourceGroups/$rg/providers/Microsoft.Storage/storageAccounts/$storageAccount"

Start-Sleep -Seconds 30

# Create filesystem (container)
az storage fs create `
  --name "datalake" `
  --account-name $storageAccount `
  --auth-mode login

# Create directories for team isolation
az storage fs directory create `
  --name "raw/sales" `
  --file-system "datalake" `
  --account-name $storageAccount `
  --auth-mode login

az storage fs directory create `
  --name "raw/engineering" `
  --file-system "datalake" `
  --account-name $storageAccount `
  --auth-mode login

# Set ACLs (give a group read+execute on their directory)
# Replace <group-object-id> with actual Entra group ID
# az storage fs access set `
#   --acl "group:<group-object-id>:r-x" `
#   --path "raw/sales" `
#   --file-system "datalake" `
#   --account-name $storageAccount `
#   --auth-mode login

# Upload sample data
az storage fs file upload `
  --source "./sample.txt" `
  --path "raw/sales/transactions-2024.csv" `
  --file-system "datalake" `
  --account-name $storageAccount `
  --auth-mode login

# List directory
az storage fs file list `
  --file-system "datalake" `
  --path "raw" `
  --account-name $storageAccount `
  --auth-mode login `
  --recursive
```

**Cleanup:**
```powershell
az group delete --name $rg --yes --no-wait
```

---

### Lab 7: Customer-Managed Keys with Key Vault
**Objective:** Configure CMK encryption for regulatory compliance.

```powershell
$rg = "rg-az305-storage-lab7"
$location = "eastus2"
$storageAccount = "staz305lab7$(Get-Random -Maximum 9999)"
$kvName = "kv-az305-lab7-$(Get-Random -Maximum 9999)"

# Create Resources
az group create --name $rg --location $location

# Create Key Vault with purge protection (required for CMK)
az keyvault create `
  --name $kvName `
  --resource-group $rg `
  --location $location `
  --enable-purge-protection true `
  --enable-rbac-authorization true

# Assign Key Vault Crypto Officer to yourself
$userId = az ad signed-in-user show --query id -o tsv
az role assignment create `
  --assignee $userId `
  --role "Key Vault Crypto Officer" `
  --scope $(az keyvault show --name $kvName --resource-group $rg --query id -o tsv)

Start-Sleep -Seconds 30

# Create encryption key
az keyvault key create `
  --vault-name $kvName `
  --name "storage-cmk" `
  --kty RSA `
  --size 2048

# Create Storage Account with system-assigned identity
az storage account create `
  --name $storageAccount `
  --resource-group $rg `
  --location $location `
  --sku Standard_GRS `
  --kind StorageV2 `
  --identity-type SystemAssigned

# Get storage account identity
$identityPrincipalId = az storage account show `
  --name $storageAccount `
  --resource-group $rg `
  --query identity.principalId -o tsv

# Assign Key Vault Crypto Service Encryption User to storage identity
az role assignment create `
  --assignee $identityPrincipalId `
  --role "Key Vault Crypto Service Encryption User" `
  --scope $(az keyvault show --name $kvName --resource-group $rg --query id -o tsv)

Start-Sleep -Seconds 30

# Configure CMK
$keyUri = az keyvault key show --vault-name $kvName --name "storage-cmk" --query key.kid -o tsv
az storage account update `
  --name $storageAccount `
  --resource-group $rg `
  --encryption-key-source Microsoft.Keyvault `
  --encryption-key-vault $(az keyvault show --name $kvName --query properties.vaultUri -o tsv) `
  --encryption-key-name "storage-cmk"

# Verify
az storage account show `
  --name $storageAccount `
  --resource-group $rg `
  --query encryption
```

**Cleanup:**
```powershell
az group delete --name $rg --yes --no-wait
# Key Vault will be soft-deleted; purge after if needed:
# az keyvault purge --name $kvName --location $location
```

---

## Quick Reference: AZ-305 Exam Triggers

| If the scenario says... | Think... |
|------------------------|----------|
| "Minimize cost for infrequently accessed data" | Cool/Cold/Archive tier + Lifecycle policy |
| "Must survive regional failure" | GRS / GZRS |
| "Read access during outage" | RA-GRS / RA-GZRS |
| "No public internet exposure" | Private Endpoint |
| "Comply with SEC/HIPAA/retention" | Immutable storage (WORM) |
| "Customer-controlled encryption" | CMK with Key Vault |
| "High-performance file shares" | Premium Azure Files |
| "Big data analytics" | ADLS Gen2 (HNS enabled) |
| "Branch office file caching" | Azure File Sync |
| "Sub-millisecond latency" | Premium Block Blobs |
| "Decouple application components" | Queue Storage (or Service Bus for advanced) |
| "Least privilege access" | Managed Identity + RBAC |
| "Temporary external access" | User Delegation SAS |
| "Lift and shift file server" | Azure Files + SMB |
| "Multiple teams, folder-level access" | ADLS Gen2 + POSIX ACLs |

---

## Additional Study Resources

- [Microsoft Learn: AZ-305 Storage Design](https://learn.microsoft.com/en-us/training/paths/design-data-storage-solutions/)
- [Storage Account Overview](https://learn.microsoft.com/en-us/azure/storage/common/storage-account-overview)
- [Redundancy Documentation](https://learn.microsoft.com/en-us/azure/storage/common/storage-redundancy)
- [Security Recommendations](https://learn.microsoft.com/en-us/azure/storage/blobs/security-recommendations)

---

*Last Updated: May 2026 | AZ-305 Exam Prep*
