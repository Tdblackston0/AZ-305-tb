# Azure Backup Hands-On Labs (AZ-305)

> 📖 **Cheat Sheet:** [Azure Backup](../Azure-Backup.md)

> **Primary exam domain:** Design business continuity solutions (**15-20%**)  
> **Secondary domains:** Design data storage solutions, design identity/governance/monitoring  
> **Lab focus:** Recovery Services Vault, VM backup/restore, SQL backup, Azure Files, Blob backup, monitoring, and backup security hardening.

## Exam Domain Mapping

- **Primary domain:** Design business continuity solutions (15-20%)
- **Secondary domains:** Design data storage solutions (20-25%), Design identity, governance, and monitoring solutions (25-30%)

| Lab | Primary Skill Tested |
|-----|---------------------|
| Lab 1: Recovery Services Vault | Vault setup, storage redundancy, soft delete, and private endpoint |
| Lab 2: Azure VM Backup | VM backup policy design and protection configuration |
| Lab 3: VM Restore Operations | Recovery point selection and restore type (original location vs. new VM) |
| Lab 4: Azure SQL Database Backup | Native PITR vs. LTR for compliance retention design |
| Lab 5: SQL Server in VM Backup | Workload-aware backup for SQL Server workloads on IaaS |
| Lab 6: Azure Files Backup | Share-level snapshot-based backup and granular file restore |
| Lab 7: Azure Blob Backup | Operational backup with PITR and change feed for blob data |
| Lab 8: Backup Center and Monitoring | Centralized backup visibility, alerting, and governance reporting |
| Lab 9: Backup Security Configuration | Ransomware-resistant backup with soft delete, immutability, and MUA |

## Infrastructure as Code Starter Templates (Bicep & Terraform)

Use these starter templates to deploy a Recovery Services Vault and baseline backup policy before running step-by-step lab commands.

```bicep
param location string = resourceGroup().location
param vaultName string

resource vault 'Microsoft.RecoveryServices/vaults@2023-06-01' = {
  name: vaultName
  location: location
  sku: {
    name: 'RS0'
    tier: 'Standard'
  }
  properties: {
    publicNetworkAccess: 'Disabled'
  }
}

resource backupConfig 'Microsoft.RecoveryServices/vaults/backupconfig@2023-06-01' = {
  parent: vault
  name: 'vaultconfig'
  properties: {
    storageModelType: 'GeoRedundant'
    softDeleteFeatureState: 'Enabled'
    crossRegionRestoreFlag: true
  }
}
```

```hcl
resource "azurerm_recovery_services_vault" "lab" {
  name                = var.vault_name
  location            = var.location
  resource_group_name = var.resource_group_name
  sku                 = "Standard"
  soft_delete_enabled = true

  identity {
    type = "SystemAssigned"
  }
}
```

> **Validation checklist after deployment:**
> - [ ] Vault appears in Recovery Services Vaults list
> - [ ] Storage redundancy shows GeoRedundant
> - [ ] Soft delete shows Enabled under Security Settings
> - [ ] Public network access shows Disabled (if private endpoint configured)

---

## Lab 1: Recovery Services Vault Setup

### Objective
Create a Recovery Services Vault, set backup storage redundancy to **GRS**, enable **soft delete**, and configure a **private endpoint**.

### When to Use
Foundation for all Azure Backup scenarios.

### Key AZ-305 Concepts
- Recovery Services Vault is the control plane for Azure VM, Azure Files, and workload backups.
- **GRS** improves regional resilience, but increases cost.
- **Soft delete** protects against accidental or malicious deletion.
- **Private endpoints** reduce exposure by keeping vault traffic on private IP space.

### Prerequisites
- Contributor or higher on the subscription/resource group
- Azure CLI and Az PowerShell installed
- `Microsoft.RecoveryServices` and `Microsoft.Network` resource providers registered
- A test VNet/subnet, or permission to create one

### Azure CLI Commands
```bash
RG="rg-backup-lab01"
LOCATION="eastus2"
VAULT_NAME="rsvaz305lab01$RANDOM"
VNET_NAME="vnet-backup-lab01"
SUBNET_NAME="snet-private-endpoints"
PE_NAME="pe-backup-vault"

az provider register --namespace Microsoft.RecoveryServices
az provider register --namespace Microsoft.Network

az group create --name $RG --location $LOCATION

az backup vault create \
  --resource-group $RG \
  --name $VAULT_NAME \
  --location $LOCATION

az backup vault backup-properties set \
  --resource-group $RG \
  --name $VAULT_NAME \
  --backup-storage-redundancy GeoRedundant

az backup vault backup-properties set \
  --resource-group $RG \
  --name $VAULT_NAME \
  --soft-delete-feature-state Enable

az network vnet create \
  --resource-group $RG \
  --name $VNET_NAME \
  --location $LOCATION \
  --address-prefixes 10.30.0.0/16 \
  --subnet-name $SUBNET_NAME \
  --subnet-prefixes 10.30.1.0/24

az network vnet subnet update \
  --resource-group $RG \
  --vnet-name $VNET_NAME \
  --name $SUBNET_NAME \
  --disable-private-endpoint-network-policies true

VAULT_ID=$(az backup vault show --resource-group $RG --name $VAULT_NAME --query id -o tsv)

az network private-endpoint create \
  --resource-group $RG \
  --name $PE_NAME \
  --location $LOCATION \
  --vnet-name $VNET_NAME \
  --subnet $SUBNET_NAME \
  --private-connection-resource-id $VAULT_ID \
  --group-id AzureBackup \
  --connection-name "${PE_NAME}-conn"

# Optional: add the recommended Private DNS zone shown in the portal for your region.
```

### PowerShell Commands
```powershell
$RG = "rg-backup-lab01-ps"
$Location = "EastUS2"
$VaultName = "rsvaz305lab01ps$((Get-Random -Maximum 9999))"
$VnetName = "vnet-backup-lab01-ps"
$SubnetName = "snet-private-endpoints"
$PeName = "pe-backup-vault-ps"

Register-AzResourceProvider -ProviderNamespace Microsoft.RecoveryServices
Register-AzResourceProvider -ProviderNamespace Microsoft.Network

New-AzResourceGroup -Name $RG -Location $Location

$Vault = New-AzRecoveryServicesVault `
  -Name $VaultName `
  -ResourceGroupName $RG `
  -Location $Location

Set-AzRecoveryServicesBackupProperty `
  -VaultId $Vault.ID `
  -BackupStorageRedundancy GeoRedundant `
  -SoftDeleteFeatureState Enable

$SubnetConfig = New-AzVirtualNetworkSubnetConfig `
  -Name $SubnetName `
  -AddressPrefix "10.31.1.0/24" `
  -PrivateEndpointNetworkPoliciesFlag Disabled

$Vnet = New-AzVirtualNetwork `
  -Name $VnetName `
  -ResourceGroupName $RG `
  -Location $Location `
  -AddressPrefix "10.31.0.0/16" `
  -Subnet $SubnetConfig

$Subnet = Get-AzVirtualNetworkSubnetConfig -Name $SubnetName -VirtualNetwork $Vnet
$PeConnection = New-AzPrivateLinkServiceConnection `
  -Name "$PeName-conn" `
  -PrivateLinkServiceId $Vault.ID `
  -GroupId "AzureBackup"

New-AzPrivateEndpoint `
  -Name $PeName `
  -ResourceGroupName $RG `
  -Location $Location `
  -Subnet $Subnet `
  -PrivateLinkServiceConnection $PeConnection
```

### Verification
- Confirm the vault shows **GeoRedundant** backup storage.
- Confirm **soft delete** is enabled.
- Confirm the private endpoint is **Approved** and has a private IP.

```bash
az backup vault show --resource-group $RG --name $VAULT_NAME \
  --query "{name:name,storage:properties.redundancySettings.standardTierStorageRedundancy,softDelete:properties.securitySettings.softDeleteSettings.softDeleteState}" -o yaml

az network private-endpoint show --resource-group $RG --name $PE_NAME \
  --query "{name:name,ip:customDnsConfigs[0].ipAddresses[0],state:privateLinkServiceConnections[0].privateLinkServiceConnectionState.status}" -o yaml
```

```powershell
Get-AzRecoveryServicesVault -Name $VaultName -ResourceGroupName $RG | Select-Object Name, Location
Get-AzPrivateEndpoint -Name $PeName -ResourceGroupName $RG | Select-Object Name, Location, ProvisioningState
```

### Cleanup
```bash
az group delete --name $RG --yes --no-wait
```

```powershell
Remove-AzResourceGroup -Name $RG -Force -AsJob
```

### Exam Tip
Use **Recovery Services Vault** for Azure VM, Azure Files, and SQL in Azure VM protection. If the question emphasizes **ransomware resilience**, prioritize **soft delete**, restricted access, and private connectivity before tuning retention.

---

## Lab 2: Azure VM Backup

### Objective
Create a daily VM backup policy with **30-day retention**, enable protection for a VM, trigger an on-demand backup, and monitor the resulting job.

### When to Use
Protecting Azure VMs.

### Key AZ-305 Concepts
- VM backup is policy-based and stored in a Recovery Services Vault.
- Daily backup with 30-day retention is a common baseline for non-mission-critical workloads.
- App-consistent backups matter for stateful workloads.
- Backup job monitoring is part of operational readiness.

### Prerequisites
- Lab 1 completed, or an existing Recovery Services Vault
- A test Azure VM (or permission to create one)
- Backup extension allowed on the VM

### Azure CLI Commands
```bash
RG="rg-backup-lab02"
LOCATION="eastus2"
VAULT_NAME="rsvaz305lab02$RANDOM"
VM_NAME="vmbackupaz305"
POLICY_NAME="vm-daily-30d"

az group create --name $RG --location $LOCATION
az backup vault create --resource-group $RG --name $VAULT_NAME --location $LOCATION

az vm create \
  --resource-group $RG \
  --name $VM_NAME \
  --image Ubuntu2204 \
  --size Standard_B2s \
  --admin-username azureuser \
  --generate-ssh-keys

az backup policy get-default-for-vm > vm-policy.json

python -c "import json; p=json.load(open('vm-policy.json')); p['properties']['retentionPolicy']['dailySchedule']['retentionDuration']['count']=30; json.dump(p, open('vm-policy-30d.json','w'))"

az backup policy create \
  --resource-group $RG \
  --vault-name $VAULT_NAME \
  --name $POLICY_NAME \
  --policy @vm-policy-30d.json

az backup protection enable-for-vm \
  --resource-group $RG \
  --vault-name $VAULT_NAME \
  --vm $VM_NAME \
  --policy-name $POLICY_NAME

ITEM_NAME=$(az backup item list \
  --resource-group $RG \
  --vault-name $VAULT_NAME \
  --backup-management-type AzureIaasVM \
  --query "[0].name" -o tsv)

CONTAINER_NAME=$(az backup item list \
  --resource-group $RG \
  --vault-name $VAULT_NAME \
  --backup-management-type AzureIaasVM \
  --query "[0].properties.containerName" -o tsv)

az backup protection backup-now \
  --resource-group $RG \
  --vault-name $VAULT_NAME \
  --backup-management-type AzureIaasVM \
  --container-name "$CONTAINER_NAME" \
  --item-name "$ITEM_NAME" \
  --retain-until 31-12-2026

az backup job list \
  --resource-group $RG \
  --vault-name $VAULT_NAME \
  --operation Backup \
  --status InProgress \
  -o table
```

### PowerShell Commands
```powershell
$RG = "rg-backup-lab02-ps"
$Location = "EastUS2"
$VaultName = "rsvaz305lab02ps$((Get-Random -Maximum 9999))"
$VmName = "vmbackupaz305ps"
$PolicyName = "vm-daily-30d-ps"

New-AzResourceGroup -Name $RG -Location $Location
$Vault = New-AzRecoveryServicesVault -Name $VaultName -ResourceGroupName $RG -Location $Location
Set-AzRecoveryServicesVaultContext -Vault $Vault

New-AzVM `
  -ResourceGroupName $RG `
  -Location $Location `
  -Name $VmName `
  -ImageName "Ubuntu2204" `
  -Size "Standard_B2s" `
  -Credential (Get-Credential -Message "Enter credentials for the lab VM")

$Schedule = Get-AzRecoveryServicesBackupSchedulePolicyObject -WorkloadType AzureVM
$Schedule.ScheduleRunTimes.Clear()
$Schedule.ScheduleRunTimes.Add((Get-Date "2025-01-01T23:00:00Z").ToUniversalTime())

$Retention = Get-AzRecoveryServicesBackupRetentionPolicyObject -WorkloadType AzureVM
$Retention.DailySchedule.DurationCountInDays = 30

$Policy = New-AzRecoveryServicesBackupProtectionPolicy `
  -Name $PolicyName `
  -WorkloadType AzureVM `
  -SchedulePolicy $Schedule `
  -RetentionPolicy $Retention `
  -VaultId $Vault.ID

Enable-AzRecoveryServicesBackupProtection `
  -Policy $Policy `
  -Name $VmName `
  -ResourceGroupName $RG `
  -VaultId $Vault.ID

$Container = Get-AzRecoveryServicesBackupContainer `
  -ContainerType AzureVM `
  -FriendlyName $VmName `
  -Status Registered `
  -VaultId $Vault.ID

$Item = Get-AzRecoveryServicesBackupItem `
  -Container $Container `
  -WorkloadType AzureVM `
  -VaultId $Vault.ID

Backup-AzRecoveryServicesBackupItem `
  -Item $Item `
  -ExpiryDateTimeUTC (Get-Date).ToUniversalTime().AddDays(30) `
  -VaultId $Vault.ID

Get-AzRecoveryServicesBackupJob -VaultId $Vault.ID |
  Sort-Object StartTime -Descending |
  Select-Object -First 5 WorkloadName, Operation, Status, StartTime
```

### Verification
```bash
az backup item list --resource-group $RG --vault-name $VAULT_NAME \
  --backup-management-type AzureIaasVM \
  --query "[].{name:properties.friendlyName,status:properties.protectionStatus,lastBackup:properties.lastBackupTime}" -o table

az backup job list --resource-group $RG --vault-name $VAULT_NAME \
  --query "[].{operation:properties.operation,status:properties.status,start:properties.startTime}" -o table
```

```powershell
Get-AzRecoveryServicesBackupItem -Container $Container -WorkloadType AzureVM -VaultId $Vault.ID |
  Select-Object Name, ProtectionStatus, LastBackupTime
```

### Cleanup
```bash
az backup protection disable \
  --resource-group $RG \
  --vault-name $VAULT_NAME \
  --container-name "$CONTAINER_NAME" \
  --item-name "$ITEM_NAME" \
  --backup-management-type AzureIaasVM \
  --delete-backup-data true \
  --yes

rm -f vm-policy.json vm-policy-30d.json
az group delete --name $RG --yes --no-wait
```

```powershell
Disable-AzRecoveryServicesBackupProtection -Item $Item -RemoveRecoveryPoints -VaultId $Vault.ID -Force
Remove-AzResourceGroup -Name $RG -Force -AsJob
```

### Exam Tip
If the requirement is **recover deleted or corrupted VM data**, Azure Backup is the answer. If the requirement is **rapid failover of a running VM to another region**, think **Azure Site Recovery**, not backup.

---

## Lab 3: VM Restore Operations

### Objective
Restore a protected VM by using **full VM restore to a new location**, **disk-only restore**, **file-level recovery**, and **cross-region restore**.

### When to Use
Recovery from VM failure or data loss.

### Key AZ-305 Concepts
- Recovery point selection drives RPO.
- Disk restore is faster to script; full VM restore adds compute/network recreation.
- File-level recovery is best for accidental file deletion.
- Cross-region restore requires a vault configuration that supports secondary-region recovery.

### Prerequisites
- Lab 2 completed and at least one successful backup exists
- A target resource group for restored disks/VMs
- For cross-region restore, use a vault configured for geo-redundant backup storage

### Azure CLI Commands
```bash
# Reuse RG, VAULT_NAME, VM_NAME, CONTAINER_NAME, and ITEM_NAME from Lab 2.
TARGET_RG="rg-backup-restore-target"
STAGING_SA="strestore$RANDOM"
LOCATION="eastus2"

az group create --name $TARGET_RG --location $LOCATION
az storage account create --name $STAGING_SA --resource-group $TARGET_RG --location $LOCATION --sku Standard_LRS

RP_NAME=$(az backup recoverypoint list \
  --resource-group $RG \
  --vault-name $VAULT_NAME \
  --container-name "$CONTAINER_NAME" \
  --item-name "$ITEM_NAME" \
  --backup-management-type AzureIaasVM \
  --query "[0].name" -o tsv)

# Disk restore
az backup restore restore-disks \
  --resource-group $RG \
  --vault-name $VAULT_NAME \
  --container-name "$CONTAINER_NAME" \
  --item-name "$ITEM_NAME" \
  --rp-name "$RP_NAME" \
  --storage-account $STAGING_SA \
  --target-resource-group $TARGET_RG

# After the restore completes, build a new VM from the restored disk.
RESTORED_OS_DISK=$(az disk list -g $TARGET_RG --query "[0].name" -o tsv)
az vm create \
  --resource-group $TARGET_RG \
  --name "${VM_NAME}-restored" \
  --attach-os-disk "$RESTORED_OS_DISK" \
  --os-type Linux

# File-level recovery note:
# Azure CLI does not provide the same first-class mount workflow as the portal/PowerShell.
# Use the portal to download the file recovery script, or use the PowerShell workflow below.

# Cross-region restore is supported only when GRS and vault settings allow secondary-region recovery.
# Use the portal if your CLI version does not expose the secondary-region restore switch.
```

### PowerShell Commands
```powershell
# Reuse $Vault, $Container, and $Item from Lab 2.
$TargetRG = "rg-backup-restore-target-ps"
$Location = "EastUS2"
New-AzResourceGroup -Name $TargetRG -Location $Location

$RecoveryPoint = Get-AzRecoveryServicesBackupRecoveryPoint `
  -Item $Item `
  -VaultId $Vault.ID |
  Sort-Object RecoveryPointTime -Descending |
  Select-Object -First 1

# Disk restore
Restore-AzRecoveryServicesBackupItem `
  -RecoveryPoint $RecoveryPoint `
  -StorageAccountName ("strestoreps" + (Get-Random -Maximum 9999)) `
  -StorageAccountResourceGroupName $TargetRG `
  -VaultId $Vault.ID

# File-level recovery (mount the recovery point and copy the required files)
$MountScriptPath = ".\vm-file-recovery.ps1"
Get-AzRecoveryServicesBackupRPMountScript `
  -RecoveryPoint $RecoveryPoint `
  -VaultId $Vault.ID `
  -Path $MountScriptPath

# Review the generated script, then run it in an elevated session.
# .\vm-file-recovery.ps1

# Cross-region restore: run only if the vault supports secondary-region recovery.
# Example pattern:
# $SecondaryRecoveryPoint = Get-AzRecoveryServicesBackupRecoveryPoint -Item $Item -UseSecondaryRegion -VaultId $Vault.ID |
#   Sort-Object RecoveryPointTime -Descending |
#   Select-Object -First 1
# Restore-AzRecoveryServicesBackupItem -RecoveryPoint $SecondaryRecoveryPoint -RestoreToSecondaryRegion -VaultId $Vault.ID
```

### Verification
```bash
az backup recoverypoint list --resource-group $RG --vault-name $VAULT_NAME \
  --container-name "$CONTAINER_NAME" --item-name "$ITEM_NAME" \
  --backup-management-type AzureIaasVM -o table

az vm show --resource-group $TARGET_RG --name "${VM_NAME}-restored" \
  --query "{name:name,provisioningState:provisioningState,powerState:powerState}" -o yaml
```

```powershell
Get-AzRecoveryServicesBackupRecoveryPoint -Item $Item -VaultId $Vault.ID |
  Select-Object RecoveryPointId, RecoveryPointTime, RecoveryPointType
```

### Cleanup
```bash
az group delete --name $TARGET_RG --yes --no-wait
```

```powershell
Remove-Item .\vm-file-recovery.ps1 -ErrorAction SilentlyContinue
Remove-AzResourceGroup -Name $TargetRG -Force -AsJob
```

### Exam Tip
Use **disk restore** when you want fast, scriptable recovery or need to inspect disks before bringing up a VM. Use **file-level recovery** when the issue is a deleted file, not a failed server.

---

## Lab 4: Azure SQL Database Backup

### Objective
Review built-in automated backup behavior for Azure SQL Database, configure **long-term retention (LTR)**, perform a **point-in-time restore**, and restore to a **different server**.

### When to Use
SQL Database protection and compliance.

### Key AZ-305 Concepts
- Azure SQL Database already includes automated backups.
- **PITR** meets operational recovery needs; **LTR** meets compliance or audit retention needs.
- Restoring to a different server is useful for forensics, testing, or isolated recovery.
- Backup storage redundancy affects durability and cost.

### Prerequisites
- An Azure SQL logical server and database
- SQL admin credentials or Entra admin access
- A second logical server for alternate restore testing

### Azure CLI Commands
```bash
RG="rg-sql-backup-lab04"
LOCATION="eastus2"
SQL_SERVER="sqlbackupaz305$RANDOM"
DR_SERVER="sqlbackupdraz305$RANDOM"
DB_NAME="appdb"
ADMIN_USER="sqladminuser"
ADMIN_PASS='P@ssw0rd12345!'

az group create --name $RG --location $LOCATION

az sql server create --resource-group $RG --name $SQL_SERVER --location $LOCATION \
  --admin-user $ADMIN_USER --admin-password "$ADMIN_PASS"
az sql server create --resource-group $RG --name $DR_SERVER --location $LOCATION \
  --admin-user $ADMIN_USER --admin-password "$ADMIN_PASS"

az sql db create --resource-group $RG --server $SQL_SERVER --name $DB_NAME \
  --edition GeneralPurpose --family Gen5 --capacity 2 --backup-storage-redundancy Geo

# Review automated backup settings
az sql db show --resource-group $RG --server $SQL_SERVER --name $DB_NAME \
  --query "{status:status,backupRedundancy:currentBackupStorageRedundancy,creationDate:creationDate}" -o yaml

# Configure long-term retention
az sql db ltr-policy set \
  --resource-group $RG \
  --server $SQL_SERVER \
  --name $DB_NAME \
  --weekly-retention P12W \
  --monthly-retention P12M \
  --yearly-retention P5Y \
  --week-of-year 1

# Point-in-time restore to same server
RESTORE_TIME=$(date -u +"%Y-%m-%dT%H:%M:%SZ")
az sql db restore \
  --resource-group $RG \
  --server $SQL_SERVER \
  --name $DB_NAME \
  --dest-name "${DB_NAME}-pitr" \
  --time "$RESTORE_TIME"

# Restore to different server
az sql db restore \
  --resource-group $RG \
  --server $SQL_SERVER \
  --name $DB_NAME \
  --dest-resource-group $RG \
  --dest-server $DR_SERVER \
  --dest-name "${DB_NAME}-drrestore" \
  --time "$RESTORE_TIME"
```

### PowerShell Commands
```powershell
$RG = "rg-sql-backup-lab04-ps"
$Location = "EastUS2"
$SqlServer = "sqlbackupaz305ps$((Get-Random -Maximum 9999))"
$DrServer = "sqlbackupdraz305ps$((Get-Random -Maximum 9999))"
$DbName = "appdb"
$Cred = Get-Credential -Message "Enter SQL admin credentials"

New-AzResourceGroup -Name $RG -Location $Location
New-AzSqlServer -ResourceGroupName $RG -ServerName $SqlServer -Location $Location -SqlAdministratorCredentials $Cred
New-AzSqlServer -ResourceGroupName $RG -ServerName $DrServer -Location $Location -SqlAdministratorCredentials $Cred

New-AzSqlDatabase `
  -ResourceGroupName $RG `
  -ServerName $SqlServer `
  -DatabaseName $DbName `
  -Edition GeneralPurpose `
  -Vcore 2 `
  -ComputeGeneration Gen5 `
  -BackupStorageRedundancy Geo

Get-AzSqlDatabase -ResourceGroupName $RG -ServerName $SqlServer -DatabaseName $DbName |
  Select-Object DatabaseName, Status, CurrentBackupStorageRedundancy

Set-AzSqlDatabaseBackupLongTermRetentionPolicy `
  -ResourceGroupName $RG `
  -ServerName $SqlServer `
  -DatabaseName $DbName `
  -WeeklyRetention P12W `
  -MonthlyRetention P12M `
  -YearlyRetention P5Y `
  -WeekOfYear 1

$RestoreTime = (Get-Date).ToUniversalTime()
Restore-AzSqlDatabase `
  -FromPointInTimeBackup `
  -PointInTime $RestoreTime `
  -ResourceGroupName $RG `
  -ServerName $SqlServer `
  -TargetDatabaseName "$DbName-pitr" `
  -ResourceId (Get-AzSqlDatabase -ResourceGroupName $RG -ServerName $SqlServer -DatabaseName $DbName).ResourceId

Restore-AzSqlDatabase `
  -FromPointInTimeBackup `
  -PointInTime $RestoreTime `
  -ResourceGroupName $RG `
  -ServerName $DrServer `
  -TargetDatabaseName "$DbName-drrestore" `
  -ResourceId (Get-AzSqlDatabase -ResourceGroupName $RG -ServerName $SqlServer -DatabaseName $DbName).ResourceId
```

### Verification
```bash
az sql db ltr-policy show --resource-group $RG --server $SQL_SERVER --name $DB_NAME -o yaml
az sql db list --resource-group $RG --server $SQL_SERVER --query "[].name" -o table
az sql db list --resource-group $RG --server $DR_SERVER --query "[].name" -o table
```

```powershell
Get-AzSqlDatabase -ResourceGroupName $RG -ServerName $SqlServer | Select-Object DatabaseName, Status
Get-AzSqlDatabase -ResourceGroupName $RG -ServerName $DrServer | Select-Object DatabaseName, Status
```

### Cleanup
```bash
az group delete --name $RG --yes --no-wait
```

```powershell
Remove-AzResourceGroup -Name $RG -Force -AsJob
```

### Exam Tip
For **Azure SQL Database**, you do **not** deploy a Recovery Services Vault for normal backups. The exam often tests whether you know to use **native automated backups + PITR/LTR** instead of VM-style backup tooling.

---

## Lab 5: SQL Server in VM Backup

### Objective
Register a SQL Server VM with a Recovery Services Vault, configure a **SQL workload backup policy**, back up an individual database, and restore it.

### When to Use
IaaS SQL Server protection.

### Key AZ-305 Concepts
- SQL in Azure VM uses **Azure Workload backup**, not Azure SQL Database native backup.
- Workload backup supports database-level protection and log backups for lower RPO.
- Auto-protection is valuable when new databases are frequently created.
- Granular restore is a major differentiator from VM-level backup.

### Prerequisites
- SQL Server running in an Azure VM
- SQL IaaS Agent extension installed and healthy
- Recovery Services Vault in the same region
- At least one user database on the SQL instance

### Azure CLI Commands
```bash
RG="rg-sqlvm-backup-lab05"
LOCATION="eastus2"
VAULT_NAME="rsvsqlvmlab05$RANDOM"
SQLVM_NAME="sqlvmaz305"
POLICY_NAME="sql-workload-policy"

az group create --name $RG --location $LOCATION
az backup vault create --resource-group $RG --name $VAULT_NAME --location $LOCATION

# Use an existing SQL VM if you already have one.
# If not, deploy one with the SQL image that matches your study scenario.

VM_ID=$(az vm show --resource-group $RG --name $SQLVM_NAME --query id -o tsv)

az backup container register \
  --resource-group $RG \
  --vault-name $VAULT_NAME \
  --resource-id $VM_ID \
  --backup-management-type AzureWorkload \
  --workload-type SQLDataBase

# Discover protectable SQL items after registration.
az backup protectable-item list \
  --resource-group $RG \
  --vault-name $VAULT_NAME \
  --backup-management-type AzureWorkload \
  --workload-type SQLDataBase \
  -o table

# For many tenants, create the policy in the portal or from a saved JSON policy file.
# Use the CLI primarily for discovery, job monitoring, and validation when studying SQL-in-VM backup flows.
```

### PowerShell Commands
```powershell
$RG = "rg-sqlvm-backup-lab05-ps"
$Location = "EastUS2"
$VaultName = "rsvsqlvmlab05ps$((Get-Random -Maximum 9999))"
$SqlVmName = "sqlvmaz305ps"
$PolicyName = "sql-workload-policy-ps"

New-AzResourceGroup -Name $RG -Location $Location
$Vault = New-AzRecoveryServicesVault -Name $VaultName -ResourceGroupName $RG -Location $Location
Set-AzRecoveryServicesVaultContext -Vault $Vault

$Vm = Get-AzVM -ResourceGroupName $RG -Name $SqlVmName
$Container = Register-AzRecoveryServicesBackupContainer `
  -ResourceId $Vm.Id `
  -BackupManagementType AzureWorkload `
  -WorkloadType MSSQL `
  -VaultId $Vault.ID `
  -Force

$ProtectableItems = Get-AzRecoveryServicesBackupProtectableItem `
  -VaultId $Vault.ID `
  -WorkloadType MSSQL `
  -Container $Container

$Schedule = Get-AzRecoveryServicesBackupSchedulePolicyObject -WorkloadType MSSQL
$Retention = Get-AzRecoveryServicesBackupRetentionPolicyObject -WorkloadType MSSQL
$Retention.FullBackupRetentionPolicy.DailySchedule.DurationCountInDays = 30

$Policy = New-AzRecoveryServicesBackupProtectionPolicy `
  -Name $PolicyName `
  -WorkloadType MSSQL `
  -BackupManagementType AzureWorkload `
  -SchedulePolicy $Schedule `
  -RetentionPolicy $Retention `
  -VaultId $Vault.ID

$Database = $ProtectableItems |
  Where-Object { $_.ProtectableItemType -eq 'SQLDataBase' } |
  Select-Object -First 1

Enable-AzRecoveryServicesBackupProtection `
  -Item $Database `
  -Policy $Policy `
  -VaultId $Vault.ID

$ProtectedItem = Get-AzRecoveryServicesBackupItem -VaultId $Vault.ID -Container $Container -WorkloadType MSSQL |
  Select-Object -First 1

Backup-AzRecoveryServicesBackupItem `
  -Item $ProtectedItem `
  -BackupType Full `
  -VaultId $Vault.ID

$RecoveryPoint = Get-AzRecoveryServicesBackupRecoveryPoint -Item $ProtectedItem -VaultId $Vault.ID |
  Sort-Object RecoveryPointTime -Descending |
  Select-Object -First 1

$RestoreConfig = Get-AzRecoveryServicesBackupWorkloadRecoveryConfig `
  -RecoveryPoint $RecoveryPoint `
  -TargetItem $ProtectedItem `
  -AlternateWorkloadRestore `
  -VaultId $Vault.ID `
  -TargetContainer $Container

$RestoreConfig.RestoredDBName = "restored-db"
Restore-AzRecoveryServicesBackupItem -WLRecoveryConfig $RestoreConfig -VaultId $Vault.ID
```

### Verification
```bash
az backup job list --resource-group $RG --vault-name $VAULT_NAME \
  --backup-management-type AzureWorkload \
  --query "[].{operation:properties.operation,status:properties.status,entity:properties.entityFriendlyName}" -o table
```

```powershell
Get-AzRecoveryServicesBackupItem -Container $Container -WorkloadType MSSQL -VaultId $Vault.ID |
  Select-Object Name, ProtectionStatus, LastBackupTime
```

### Cleanup
```bash
# Disable and unregister using the discovered item/container names.
az group delete --name $RG --yes --no-wait
```

```powershell
Disable-AzRecoveryServicesBackupProtection -Item $ProtectedItem -RemoveRecoveryPoints -VaultId $Vault.ID -Force
Unregister-AzRecoveryServicesBackupContainer -Container $Container -VaultId $Vault.ID -Force
Remove-AzResourceGroup -Name $RG -Force -AsJob
```

### Exam Tip
Choose **SQL workload backup** when the requirement is **database-level restore**, **log backup**, or a lower RPO for SQL Server running inside an Azure VM. Choose **VM backup** only when whole-VM recovery is sufficient.

---

## Lab 6: Azure Files Backup

### Objective
Protect an Azure Files share with a policy, trigger backup, perform an **item-level restore**, and test a **full share restore**.

### When to Use
File share data protection.

### Key AZ-305 Concepts
- Azure Files backup is snapshot-based and supports granular recovery.
- The vault and storage account must be in the same region.
- Share-level restore is useful for broad corruption; item restore is better for accidental file deletion.
- Retention design affects cost and operational recovery flexibility.

### Prerequisites
- Storage account with Azure Files enabled
- Recovery Services Vault in the same region
- At least one test file in the share

### Azure CLI Commands
```bash
RG="rg-files-backup-lab06"
LOCATION="eastus2"
VAULT_NAME="rsvfileslab06$RANDOM"
STORAGE_ACCOUNT="stfilesaz305$RANDOM"
SHARE_NAME="appshare"
POLICY_NAME="fileshare-policy"

az group create --name $RG --location $LOCATION
az backup vault create --resource-group $RG --name $VAULT_NAME --location $LOCATION
az storage account create --resource-group $RG --name $STORAGE_ACCOUNT --location $LOCATION --sku Standard_LRS --kind StorageV2

STORAGE_KEY=$(az storage account keys list --resource-group $RG --account-name $STORAGE_ACCOUNT --query "[0].value" -o tsv)

az storage share create --name $SHARE_NAME --account-name $STORAGE_ACCOUNT --account-key $STORAGE_KEY --quota 100
printf "version-1" > app.txt
az storage file upload --share-name $SHARE_NAME --source app.txt --path app.txt --account-name $STORAGE_ACCOUNT --account-key $STORAGE_KEY

SA_ID=$(az storage account show --resource-group $RG --name $STORAGE_ACCOUNT --query id -o tsv)
az backup container register --resource-group $RG --vault-name $VAULT_NAME --resource-id $SA_ID --backup-management-type AzureStorage

az backup protection enable-for-azurefileshare \
  --resource-group $RG \
  --vault-name $VAULT_NAME \
  --storage-account $STORAGE_ACCOUNT \
  --azure-file-share $SHARE_NAME

FILE_ITEM=$(az backup item list --resource-group $RG --vault-name $VAULT_NAME \
  --backup-management-type AzureStorage --query "[0].name" -o tsv)
FILE_CONTAINER=$(az backup item list --resource-group $RG --vault-name $VAULT_NAME \
  --backup-management-type AzureStorage --query "[0].properties.containerName" -o tsv)

az backup protection backup-now \
  --resource-group $RG \
  --vault-name $VAULT_NAME \
  --container-name "$FILE_CONTAINER" \
  --item-name "$FILE_ITEM" \
  --backup-management-type AzureStorage \
  --retain-until 31-12-2026

printf "corrupted-content" > app.txt
az storage file upload --share-name $SHARE_NAME --source app.txt --path app.txt --account-name $STORAGE_ACCOUNT --account-key $STORAGE_KEY

RP_NAME=$(az backup recoverypoint list --resource-group $RG --vault-name $VAULT_NAME \
  --container-name "$FILE_CONTAINER" --item-name "$FILE_ITEM" --backup-management-type AzureStorage \
  --query "[0].name" -o tsv)

# Item-level restore
az backup restore restore-azurefiles \
  --resource-group $RG \
  --vault-name $VAULT_NAME \
  --container-name "$FILE_CONTAINER" \
  --item-name "$FILE_ITEM" \
  --rp-name $RP_NAME \
  --restore-mode OriginalLocation \
  --resolve-conflict Overwrite \
  --source-file-type File \
  --source-file-path app.txt

# Full share restore to alternate location
az backup restore restore-azurefileshare \
  --resource-group $RG \
  --vault-name $VAULT_NAME \
  --container-name "$FILE_CONTAINER" \
  --item-name "$FILE_ITEM" \
  --rp-name $RP_NAME \
  --restore-mode AlternateLocation \
  --resolve-conflict Overwrite \
  --target-storage-account $STORAGE_ACCOUNT \
  --target-file-share "${SHARE_NAME}-restore"
```

### PowerShell Commands
```powershell
$RG = "rg-files-backup-lab06-ps"
$Location = "EastUS2"
$VaultName = "rsvfileslab06ps$((Get-Random -Maximum 9999))"
$StorageAccount = "stfilesaz305ps$((Get-Random -Maximum 9999))"
$ShareName = "appshare"

New-AzResourceGroup -Name $RG -Location $Location
$Vault = New-AzRecoveryServicesVault -Name $VaultName -ResourceGroupName $RG -Location $Location
$Storage = New-AzStorageAccount -ResourceGroupName $RG -Name $StorageAccount -Location $Location -SkuName Standard_LRS -Kind StorageV2
New-AzStorageShare -Context $Storage.Context -Name $ShareName
"version-1" | Out-File -FilePath .\app.txt -Encoding ascii
Set-AzStorageFileContent -Context $Storage.Context -ShareName $ShareName -Source .\app.txt -Path app.txt

$Container = Register-AzRecoveryServicesBackupContainer `
  -ResourceId $Storage.Id `
  -BackupManagementType AzureStorage `
  -WorkloadType AzureFiles `
  -VaultId $Vault.ID `
  -Force

$Item = Get-AzRecoveryServicesBackupProtectableItem -Container $Container -WorkloadType AzureFiles -VaultId $Vault.ID |
  Where-Object { $_.Name -like "*$ShareName*" }

$Policy = Get-AzRecoveryServicesBackupProtectionPolicy -VaultId $Vault.ID -WorkloadType AzureFiles | Select-Object -First 1
Enable-AzRecoveryServicesBackupProtection -Item $Item -Policy $Policy -VaultId $Vault.ID

$ProtectedItem = Get-AzRecoveryServicesBackupItem -Container $Container -WorkloadType AzureFiles -VaultId $Vault.ID
Backup-AzRecoveryServicesBackupItem -Item $ProtectedItem -ExpiryDateTimeUTC (Get-Date).ToUniversalTime().AddDays(30) -VaultId $Vault.ID

$RecoveryPoint = Get-AzRecoveryServicesBackupRecoveryPoint -Item $ProtectedItem -VaultId $Vault.ID |
  Sort-Object RecoveryPointTime -Descending |
  Select-Object -First 1

Restore-AzRecoveryServicesBackupItem `
  -RecoveryPoint $RecoveryPoint `
  -ResolveConflict Overwrite `
  -SourceFileType File `
  -SourceFilePath "app.txt" `
  -VaultId $Vault.ID `
  -VaultLocation $Location
```

### Verification
```bash
az backup item list --resource-group $RG --vault-name $VAULT_NAME --backup-management-type AzureStorage \
  --query "[].{name:name,status:properties.protectionStatus,lastBackup:properties.lastBackupTime}" -o table

az storage file download --share-name $SHARE_NAME --path app.txt --dest restored-app.txt \
  --account-name $STORAGE_ACCOUNT --account-key $STORAGE_KEY
cat restored-app.txt
```

```powershell
Get-AzRecoveryServicesBackupItem -Container $Container -WorkloadType AzureFiles -VaultId $Vault.ID |
  Select-Object Name, ProtectionStatus, LastBackupTime
```

### Cleanup
```bash
az backup protection disable \
  --resource-group $RG \
  --vault-name $VAULT_NAME \
  --container-name "$FILE_CONTAINER" \
  --item-name "$FILE_ITEM" \
  --backup-management-type AzureStorage \
  --delete-backup-data true \
  --yes

rm -f app.txt restored-app.txt
az group delete --name $RG --yes --no-wait
```

```powershell
Disable-AzRecoveryServicesBackupProtection -Item $ProtectedItem -RemoveRecoveryPoints -VaultId $Vault.ID -Force
Remove-Item .\app.txt -ErrorAction SilentlyContinue
Remove-AzResourceGroup -Name $RG -Force -AsJob
```

### Exam Tip
If the requirement is **restore a single file quickly**, Azure Files backup is usually the better answer than restoring an entire VM or copying a full storage account snapshot.

---

## Lab 7: Azure Blob Backup

### Objective
Enable **operational backup** for blobs, configure **point-in-time restore**, test blob recovery, and review **vaulted backup** prerequisites.

### When to Use
Blob storage protection.

### Key AZ-305 Concepts
- Operational backup for Blob Storage relies on **versioning**, **change feed**, and **delete retention**.
- Point-in-time restore is ideal for accidental overwrite or deletion.
- Vaulted backup provides an extra isolation layer for stronger protection.
- Backup Vault is used for Microsoft Data Protection scenarios, not Recovery Services Vault.

### Prerequisites
- General-purpose v2 storage account with Blob Storage
- Blob container with sample data
- Permissions to create a Backup Vault if testing vaulted backup

### Azure CLI Commands
```bash
RG="rg-blob-backup-lab07"
LOCATION="eastus2"
STORAGE_ACCOUNT="stblobaz305$RANDOM"
CONTAINER="appdata"
BACKUP_VAULT="bvblobaz305$RANDOM"

az group create --name $RG --location $LOCATION
az storage account create --resource-group $RG --name $STORAGE_ACCOUNT --location $LOCATION --sku Standard_GRS --kind StorageV2

az storage account blob-service-properties update \
  --resource-group $RG \
  --account-name $STORAGE_ACCOUNT \
  --enable-change-feed true \
  --enable-versioning true \
  --enable-delete-retention true \
  --delete-retention-days 14 \
  --enable-container-delete-retention true \
  --container-delete-retention-days 14 \
  --enable-restore-policy true \
  --restore-days 7

STORAGE_KEY=$(az storage account keys list --resource-group $RG --account-name $STORAGE_ACCOUNT --query "[0].value" -o tsv)
az storage container create --name $CONTAINER --account-name $STORAGE_ACCOUNT --account-key $STORAGE_KEY
printf "version-1" > blob.txt
az storage blob upload --container-name $CONTAINER --name blob.txt --file blob.txt --account-name $STORAGE_ACCOUNT --account-key $STORAGE_KEY

# Simulate corruption
printf "version-2-corrupted" > blob.txt
az storage blob upload --container-name $CONTAINER --name blob.txt --file blob.txt --overwrite true --account-name $STORAGE_ACCOUNT --account-key $STORAGE_KEY

# Review versions for recovery
az storage blob list --container-name $CONTAINER --include v --account-name $STORAGE_ACCOUNT --account-key $STORAGE_KEY -o table

# Create Backup Vault for vaulted backup evaluation
az dataprotection backup-vault create \
  --resource-group $RG \
  --vault-name $BACKUP_VAULT \
  --location $LOCATION \
  --storage-settings datastore-type=VaultStore type=GeoRedundant

# Use the portal or the Data Protection backup instance workflow to complete vaulted blob backup configuration.
# This is a good exam reminder: Blob operational backup and vaulted backup are different protection models.
```

### PowerShell Commands
```powershell
$RG = "rg-blob-backup-lab07-ps"
$Location = "EastUS2"
$StorageAccount = "stblobaz305ps$((Get-Random -Maximum 9999))"
$Container = "appdata"
$BackupVault = "bvblobaz305ps$((Get-Random -Maximum 9999))"

New-AzResourceGroup -Name $RG -Location $Location
$Storage = New-AzStorageAccount -ResourceGroupName $RG -Name $StorageAccount -Location $Location -SkuName Standard_GRS -Kind StorageV2

Update-AzStorageBlobServiceProperty `
  -ResourceGroupName $RG `
  -StorageAccountName $StorageAccount `
  -EnableChangeFeed $true `
  -EnableVersioning $true `
  -EnableDeleteRetentionPolicy $true `
  -DeleteRetentionDays 14 `
  -EnableRestorePolicy $true `
  -RestoreDays 7

New-AzStorageContainer -Name $Container -Context $Storage.Context
"version-1" | Out-File .\blob.txt -Encoding ascii
Set-AzStorageBlobContent -Context $Storage.Context -Container $Container -File .\blob.txt -Blob blob.txt

"version-2-corrupted" | Out-File .\blob.txt -Encoding ascii
Set-AzStorageBlobContent -Context $Storage.Context -Container $Container -File .\blob.txt -Blob blob.txt -Force

Get-AzStorageBlob -Context $Storage.Context -Container $Container -IncludeVersion | Select-Object Name, VersionId, IsLatestVersion

New-AzDataProtectionBackupVault `
  -ResourceGroupName $RG `
  -VaultName $BackupVault `
  -Location $Location `
  -StorageSetting @(@{Type='GeoRedundant'; DatastoreType='VaultStore'})
```

### Verification
```bash
az storage account blob-service-properties show --resource-group $RG --account-name $STORAGE_ACCOUNT \
  --query "{changeFeed:changeFeed.enabled,versioning:isVersioningEnabled,restorePolicy:restorePolicy.enabled}" -o yaml

az storage blob list --container-name $CONTAINER --include v --account-name $STORAGE_ACCOUNT --account-key $STORAGE_KEY -o table
```

```powershell
Get-AzStorageBlobServiceProperty -ResourceGroupName $RG -StorageAccountName $StorageAccount
```

### Cleanup
```bash
rm -f blob.txt
az group delete --name $RG --yes --no-wait
```

```powershell
Remove-Item .\blob.txt -ErrorAction SilentlyContinue
Remove-AzResourceGroup -Name $RG -Force -AsJob
```

### Exam Tip
If the prompt says **recover a blob to an earlier point after overwrite or deletion**, think **Blob versioning + point-in-time restore**. If it stresses **backup isolation**, look for **vaulted backup**.

---

## Lab 8: Backup Center and Monitoring

### Objective
Use **Backup Center**, enable **backup reports** with Log Analytics, create **alerts**, and review policy/compliance posture.

### When to Use
Centralized backup management.

### Key AZ-305 Concepts
- Backup Center gives a cross-vault, cross-subscription operational view.
- Log Analytics enables historical reporting and trend analysis.
- Alerts are critical for missed backups, failed jobs, and governance visibility.
- Compliance is not just retention; it also includes policy coverage and monitoring.

### Prerequisites
- At least one Recovery Services Vault or Backup Vault with protected items
- Log Analytics workspace permissions
- Access to Azure Monitor alerts

### Azure CLI Commands
```bash
RG="rg-backup-monitoring-lab08"
LOCATION="eastus2"
LAW_NAME="law-backup-monitoring-$RANDOM"
ACTION_GROUP="ag-backup-ops"

az group create --name $RG --location $LOCATION
az monitor log-analytics workspace create --resource-group $RG --workspace-name $LAW_NAME --location $LOCATION

LAW_ID=$(az monitor log-analytics workspace show --resource-group $RG --workspace-name $LAW_NAME --query id -o tsv)

# Example: send Recovery Services vault diagnostics to Log Analytics.
# Replace SOURCE_VAULT_ID with your existing vault resource ID.
SOURCE_VAULT_ID="/subscriptions/<sub-id>/resourceGroups/<rg>/providers/Microsoft.RecoveryServices/vaults/<vault-name>"

az monitor diagnostic-settings create \
  --name backup-diagnostics \
  --resource $SOURCE_VAULT_ID \
  --workspace $LAW_ID \
  --logs '[{"category":"AzureBackupReport","enabled":true},{"category":"CoreAzureBackup","enabled":true}]' \
  --metrics '[{"category":"AllMetrics","enabled":true}]'

az monitor action-group create \
  --resource-group $RG \
  --name $ACTION_GROUP \
  --short-name bkops

# Example metric alert for failed backup jobs count.
az monitor metrics alert create \
  --resource-group $RG \
  --name "backup-failures-alert" \
  --scopes $SOURCE_VAULT_ID \
  --condition "count 'Backup Health Event' > 0" \
  --description "Alert on backup health events" \
  --action $ACTION_GROUP

# Backup Center itself is explored in the Azure portal.
# Search for 'Backup Center' and review Overview, Backup Instances, Jobs, Alerts, and Policies.
```

### PowerShell Commands
```powershell
$RG = "rg-backup-monitoring-lab08-ps"
$Location = "EastUS2"
$WorkspaceName = "lawbackupmonitorps$((Get-Random -Maximum 9999))"
$ActionGroup = "ag-backup-ops-ps"

New-AzResourceGroup -Name $RG -Location $Location
$Workspace = New-AzOperationalInsightsWorkspace -ResourceGroupName $RG -Name $WorkspaceName -Location $Location -Sku PerGB2018

# Replace with an existing Recovery Services vault resource ID.
$SourceVaultId = "/subscriptions/<sub-id>/resourceGroups/<rg>/providers/Microsoft.RecoveryServices/vaults/<vault-name>"

Set-AzDiagnosticSetting `
  -Name "backup-diagnostics" `
  -ResourceId $SourceVaultId `
  -WorkspaceId $Workspace.ResourceId `
  -Enabled $true

$Receiver = New-AzActionGroupReceiver -Name "OpsMail" -EmailReceiver -EmailAddress "ops@example.com"
Set-AzActionGroup -ResourceGroupName $RG -Name $ActionGroup -ShortName "bkops" -Receiver $Receiver

# Review backup jobs through the vault and then correlate in Log Analytics.
```

### Verification
- In **Backup Center**, confirm vaults, backup items, jobs, and alerts are visible.
- In Log Analytics, run a simple query such as `AzureDiagnostics | take 10`.
- Confirm diagnostic settings are attached to the source vault.

```bash
az monitor diagnostic-settings list --resource $SOURCE_VAULT_ID -o table
az monitor log-analytics workspace show --resource-group $RG --workspace-name $LAW_NAME -o table
```

```powershell
Get-AzDiagnosticSetting -ResourceId $SourceVaultId
Get-AzOperationalInsightsWorkspace -ResourceGroupName $RG -Name $WorkspaceName
```

### Cleanup
```bash
az group delete --name $RG --yes --no-wait
```

```powershell
Remove-AzResourceGroup -Name $RG -Force -AsJob
```

### Exam Tip
If the question asks for **centralized backup visibility across multiple vaults/subscriptions**, the best answer is usually **Backup Center + Log Analytics + Azure Monitor alerts**, not a custom dashboard alone.

---

## Lab 9: Backup Security Configuration

### Objective
Review and harden backup security by using **multi-user authorization (MUA)** concepts, **immutable vault** planning, **soft delete recovery**, and vault security review.

### When to Use
Protecting backups from ransomware/deletion.

### Key AZ-305 Concepts
- Backup security is a major AZ-305 design theme when ransomware or malicious deletion appears in the scenario.
- **Soft delete** provides immediate safety; **immutable vault** adds stronger protection against tampering.
- **MUA** introduces an approval boundary for critical operations.
- Least privilege and separation of duties matter as much as retention.

### Prerequisites
- Existing Recovery Services Vault with at least one protected item
- RBAC rights to review vault settings
- For MUA, a design plan for Resource Guard / approval workflow

### Azure CLI Commands
```bash
# Reuse an existing vault.
RG="<existing-rg>"
VAULT_NAME="<existing-vault>"

az backup vault show --resource-group $RG --name $VAULT_NAME -o yaml

# Enable soft delete if it is not already enabled.
az backup vault backup-properties set \
  --resource-group $RG \
  --name $VAULT_NAME \
  --soft-delete-feature-state Enable

# Review vault properties and security-related settings.
az backup vault backup-properties show \
  --resource-group $RG \
  --name $VAULT_NAME -o yaml

# Multi-user authorization and immutable vault are commonly completed in the portal today.
# In the portal: Recovery Services vault -> Security Settings -> configure Resource Guard / MUA and Immutable vault.
# Then document who can approve destructive actions and test the approval flow.
```

### PowerShell Commands
```powershell
$RG = "<existing-rg>"
$VaultName = "<existing-vault>"
$Vault = Get-AzRecoveryServicesVault -ResourceGroupName $RG -Name $VaultName

Set-AzRecoveryServicesBackupProperty `
  -VaultId $Vault.ID `
  -SoftDeleteFeatureState Enable

Get-AzRecoveryServicesVault -ResourceGroupName $RG -Name $VaultName | Select-Object Name, Location

# For MUA and immutable vault, use the portal if the feature is not exposed in your Az module version.
# Validate the final configuration by reviewing Security Settings and attempting a protected destructive action.
```

### Verification
- Confirm **soft delete** is enabled.
- Review vault **Security Settings** in the portal.
- Attempt a non-production delete/disable-protection action and confirm the vault protection flow blocks or delays it as designed.
- Document which identities can approve destructive changes.

```bash
az backup vault backup-properties show --resource-group $RG --name $VAULT_NAME -o yaml
```

```powershell
Get-AzRecoveryServicesVault -ResourceGroupName $RG -Name $VaultName
```

### Cleanup
- If you enabled soft delete or immutable settings in a shared test vault, revert only if your governance policy allows it.
- Remove any temporary protected test items created for validation.
- Do **not** remove security protections from production vaults just to simplify cleanup.

### Exam Tip
When the exam mentions **ransomware**, **malicious insider**, or **backup deletion**, think in layers: **soft delete + immutable settings + MUA/approval + least privilege + monitoring**.

---

## Backup Decision Summary Table

| Scenario | Best Azure backup choice | Why | Key tradeoff | AZ-305 trigger phrase |
|---|---|---|---|---|
| Azure VM needs daily restore points | Recovery Services Vault + VM backup | Central policy, restore points, operational recovery | Longer restore than live DR | "Protect Azure VMs" |
| Azure VM needs near-live failover | Azure Site Recovery | Replication for low RTO failover | More cost/complexity than backup | "Regional VM failover" |
| Azure SQL Database needs operational restore | Native PITR | Built-in automated backups, no vault needed | Retention window varies by tier | "Point-in-time restore" |
| Azure SQL Database needs compliance retention | LTR | Long-term retention for audit/compliance | More storage cost | "7 years retention" |
| SQL Server in Azure VM needs DB-level restore | Azure Workload backup | Database/log-level backup, lower RPO | More setup than VM backup | "Restore a single database" |
| Azure Files needs single-file recovery | Azure Files backup | Snapshot-based, granular restore | Same-region design requirement | "Restore one file" |
| Blob data needs rollback after overwrite | Blob operational backup + PITR | Versioning/change feed/restore policy | Native feature setup required | "Recover overwritten blob" |
| Backup estate needs central oversight | Backup Center + Log Analytics | Cross-vault visibility and reporting | Requires monitoring design | "Centralized management" |
| Backup must resist deletion/ransomware | Soft delete + immutable vault + MUA | Defense-in-depth for recovery points | More governance/process overhead | "Protect backups from deletion" |

---

## Exam-Style Review Questions

1. When should you choose **Azure Backup** over **Azure Site Recovery** for a VM workload?
2. Why is **LTR** the correct answer for long-term Azure SQL Database retention, instead of a Recovery Services Vault?
3. What backup design would you choose if a company needs to recover a **single file** from a shared Azure Files workload?
4. Which security settings best protect backup data from a ransomware-style delete attempt?
5. What is the architectural difference between **Blob operational backup** and **vaulted backup**?
