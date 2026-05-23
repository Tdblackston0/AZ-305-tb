# Azure Disaster Recovery Hands-On Labs (AZ-305)

> 📖 **Cheat Sheet:** [Azure DR](../Azure-DR.md)

> **Primary exam domain:** Design business continuity solutions (15-20%)  
> **Secondary domains:** Design infrastructure solutions (30-35%), Design data storage solutions (20-25%), Design identity, governance, and monitoring solutions (25-30%)  
> **Tools used:** Azure CLI, Azure PowerShell, Azure Site Recovery, Azure SQL Database, Azure Cosmos DB, Traffic Manager, Azure Front Door, Azure Storage  
> **Important:** Use a non-production subscription. Several labs create billable DR resources in two regions.

---

## How to Use These Labs

- Run each lab in its own resource group unless the lab explicitly builds on a previous lab.
- For Labs 1-4, keep the same Azure Site Recovery resources so you can progress from replication to failover to failback.
- Replace globally unique names before deployment.
- Validate regional support before starting, especially for Azure Site Recovery and SQL failover groups.
- Prefer least privilege: **Contributor** on the lab resource groups is usually sufficient.

### Shared Setup

#### Azure CLI
```bash
az login
az account show -o table
az extension add --name site-recovery
az provider register --namespace Microsoft.RecoveryServices
az provider register --namespace Microsoft.Network
az provider register --namespace Microsoft.Compute
az provider register --namespace Microsoft.Storage
az provider register --namespace Microsoft.Sql
az provider register --namespace Microsoft.DocumentDB
az provider register --namespace Microsoft.TrafficManager
az provider register --namespace Microsoft.Cdn
```

#### PowerShell
```powershell
Connect-AzAccount
Get-AzContext
Install-Module Az -Scope CurrentUser -Force -AllowClobber
Register-AzResourceProvider -ProviderNamespace Microsoft.RecoveryServices
Register-AzResourceProvider -ProviderNamespace Microsoft.Network
Register-AzResourceProvider -ProviderNamespace Microsoft.Compute
Register-AzResourceProvider -ProviderNamespace Microsoft.Storage
Register-AzResourceProvider -ProviderNamespace Microsoft.Sql
Register-AzResourceProvider -ProviderNamespace Microsoft.DocumentDB
Register-AzResourceProvider -ProviderNamespace Microsoft.TrafficManager
Register-AzResourceProvider -ProviderNamespace Microsoft.Cdn
```

---

## Lab 1: Azure Site Recovery - Azure to Azure

### Objective
Create a Recovery Services vault, prepare Azure Site Recovery, enable replication for an Azure VM to a secondary region, apply a replication policy, and monitor replication health.

### When to Use
Use Azure Site Recovery when you need **VM-level disaster recovery across Azure regions** for IaaS workloads that cannot be rebuilt quickly enough from code or backups alone.

### Key AZ-305 Concepts
- Recovery Services vault placement and regional DR boundaries
- Azure Site Recovery fabrics, protection containers, and container mappings
- Replication policy design and recovery point retention
- Cache storage accounts and target network mapping
- Warm standby vs. pilot light vs. backup-only recovery

### Exam Domain Mapping
- **Primary:** Design business continuity solutions (15-20%)
- **Secondary:** Design infrastructure solutions (30-35%)

### Prerequisites
- A source VM already running in the primary region (example: `eastus`)
- A recovery region selected and supported for A2A replication (example: `centralus`)
- Contributor rights on both source and recovery resource groups
- Azure CLI with the `site-recovery` extension installed
- Azure PowerShell `Az` modules installed

### Architecture and Design Rationale
Azure Site Recovery is best when the business wants to recover an entire VM with its OS, disks, and networking posture in another region. It is a good **active-passive / warm standby** pattern when the application is stateful or difficult to rebuild quickly. For AZ-305, remember that **ASR protects compute continuity**, but you must still think about **app dependencies and data consistency**.

### Implementation Steps
1. Create the Recovery Services vault in the recovery region.
2. Create or confirm the recovery VNet, subnet, and cache storage account.
3. Create source and recovery fabrics and protection containers.
4. Create the A2A replication policy.
5. Map source and recovery containers and networks in both directions.
6. Enable replication for the VM.
7. Monitor initial seeding until the item reaches a protected state.

### IaC Implementation
Use Bicep or Terraform to create the vault, recovery network, and cache storage account. The **replication enablement workflow itself is still operational** and is more practical through Azure CLI or Azure PowerShell.

#### Bicep
```bicep
param location string = 'centralus'
param vaultName string
param drVnetName string = 'vnet-dr'
param cacheStorageName string

resource vault 'Microsoft.RecoveryServices/vaults@2023-02-01' = {
  name: vaultName
  location: location
  sku: {
    name: 'Standard'
  }
  properties: {}
}

resource drVnet 'Microsoft.Network/virtualNetworks@2023-09-01' = {
  name: drVnetName
  location: location
  properties: {
    addressSpace: {
      addressPrefixes: [
        '10.50.0.0/16'
      ]
    }
    subnets: [
      {
        name: 'snet-dr'
        properties: {
          addressPrefix: '10.50.1.0/24'
        }
      }
    ]
  }
}

resource cache 'Microsoft.Storage/storageAccounts@2023-05-01' = {
  name: cacheStorageName
  location: 'eastus'
  sku: {
    name: 'Standard_LRS'
  }
  kind: 'StorageV2'
  properties: {
    supportsHttpsTrafficOnly: true
  }
}
```

#### Terraform
```hcl
resource "azurerm_recovery_services_vault" "vault" {
  name                = var.vault_name
  location            = var.recovery_region
  resource_group_name = azurerm_resource_group.dr.name
  sku                 = "Standard"
}

resource "azurerm_virtual_network" "dr" {
  name                = "vnet-dr"
  location            = var.recovery_region
  resource_group_name = azurerm_resource_group.dr.name
  address_space       = ["10.50.0.0/16"]
}

resource "azurerm_subnet" "dr" {
  name                 = "snet-dr"
  resource_group_name  = azurerm_resource_group.dr.name
  virtual_network_name = azurerm_virtual_network.dr.name
  address_prefixes     = ["10.50.1.0/24"]
}

resource "azurerm_storage_account" "cache" {
  name                     = var.cache_storage_name
  resource_group_name      = azurerm_resource_group.source.name
  location                 = var.primary_region
  account_tier             = "Standard"
  account_replication_type = "LRS"
}
```

### Azure CLI Commands
```bash
PRIMARY_RG="rg-dr-source"
RECOVERY_RG="rg-dr-recovery"
VAULT_RG="$RECOVERY_RG"
PRIMARY_REGION="eastus"
RECOVERY_REGION="centralus"
SOURCE_VM="vm-dr-source"
VAULT_NAME="rsvdr$RANDOM"
POLICY_NAME="a2a-policy"
PRIMARY_FABRIC="fabric-eastus"
RECOVERY_FABRIC="fabric-centralus"
PRIMARY_CONTAINER="pc-eastus"
RECOVERY_CONTAINER="pc-centralus"
DR_VNET="vnet-dr"
DR_SUBNET="snet-dr"
CACHE_SA="drcache$RANDOM"
PRIMARY_TO_RECOVERY_MAP="map-eastus-centralus"
RECOVERY_TO_PRIMARY_MAP="map-centralus-eastus"
NETMAP_PRIMARY="netmap-eastus-centralus"
NETMAP_RECOVERY="netmap-centralus-eastus"
RPI_NAME="rpi-$SOURCE_VM"

az group create --name $RECOVERY_RG --location $RECOVERY_REGION
az backup vault create --resource-group $VAULT_RG --name $VAULT_NAME --location $RECOVERY_REGION
az network vnet create --resource-group $RECOVERY_RG --name $DR_VNET --location $RECOVERY_REGION --address-prefix 10.50.0.0/16 --subnet-name $DR_SUBNET --subnet-prefix 10.50.1.0/24
az storage account create --name $CACHE_SA --resource-group $PRIMARY_RG --location $PRIMARY_REGION --sku Standard_LRS --kind StorageV2

az site-recovery fabric create -g $VAULT_RG -n $PRIMARY_FABRIC --vault-name $VAULT_NAME --custom-details "{azure:{location:$PRIMARY_REGION}}"
az site-recovery fabric create -g $VAULT_RG -n $RECOVERY_FABRIC --vault-name $VAULT_NAME --custom-details "{azure:{location:$RECOVERY_REGION}}"

az site-recovery protection-container create -g $VAULT_RG --fabric-name $PRIMARY_FABRIC -n $PRIMARY_CONTAINER --vault-name $VAULT_NAME --provider-input "[{instance-type:A2A}]"
az site-recovery protection-container create -g $VAULT_RG --fabric-name $RECOVERY_FABRIC -n $RECOVERY_CONTAINER --vault-name $VAULT_NAME --provider-input "[{instance-type:A2A}]"

az site-recovery policy create -g $VAULT_RG --vault-name $VAULT_NAME -n $POLICY_NAME --provider-specific-input "{a2a:{multi-vm-sync-status:Enable}}"

POLICY_ID=$(az site-recovery policy show -g $VAULT_RG --vault-name $VAULT_NAME -n $POLICY_NAME --query id -o tsv)
RECOVERY_CONTAINER_ID=$(az site-recovery protection-container show -g $VAULT_RG --vault-name $VAULT_NAME --fabric-name $RECOVERY_FABRIC -n $RECOVERY_CONTAINER --query id -o tsv)
PRIMARY_CONTAINER_ID=$(az site-recovery protection-container show -g $VAULT_RG --vault-name $VAULT_NAME --fabric-name $PRIMARY_FABRIC -n $PRIMARY_CONTAINER --query id -o tsv)
DR_VNET_ID=$(az network vnet show -g $RECOVERY_RG -n $DR_VNET --query id -o tsv)
CACHE_ID=$(az storage account show -g $PRIMARY_RG -n $CACHE_SA --query id -o tsv)
RECOVERY_RG_ID=$(az group show -n $RECOVERY_RG --query id -o tsv)
PRIMARY_RG_ID=$(az group show -n $PRIMARY_RG --query id -o tsv)

az site-recovery protection-container mapping create -g $VAULT_RG --fabric-name $PRIMARY_FABRIC -n $PRIMARY_TO_RECOVERY_MAP --protection-container $PRIMARY_CONTAINER --vault-name $VAULT_NAME --policy-id $POLICY_ID --provider-input "{a2a:{agent-auto-update-status:Disabled}}" --target-container $RECOVERY_CONTAINER_ID
az site-recovery protection-container mapping create -g $VAULT_RG --fabric-name $RECOVERY_FABRIC -n $RECOVERY_TO_PRIMARY_MAP --protection-container $RECOVERY_CONTAINER --vault-name $VAULT_NAME --policy-id $POLICY_ID --provider-input "{a2a:{agent-auto-update-status:Disabled}}" --target-container $PRIMARY_CONTAINER_ID

VM_ID=$(az vm show -g $PRIMARY_RG -n $SOURCE_VM --query id -o tsv)
SOURCE_NIC_ID=$(az vm show -g $PRIMARY_RG -n $SOURCE_VM --query "networkProfile.networkInterfaces[0].id" -o tsv)
SOURCE_SUBNET_ID=$(az network nic show --ids $SOURCE_NIC_ID --query "ipConfigurations[0].subnet.id" -o tsv)
SOURCE_VNET_ID=${SOURCE_SUBNET_ID%/subnets/*}
SOURCE_VNET_NAME=$(az resource show --ids $SOURCE_VNET_ID --query name -o tsv)
OS_DISK_ID=$(az vm show -g $PRIMARY_RG -n $SOURCE_VM --query storageProfile.osDisk.managedDisk.id -o tsv)
DATA_DISK_ID=$(az vm show -g $PRIMARY_RG -n $SOURCE_VM --query storageProfile.dataDisks[0].managedDisk.id -o tsv)

az site-recovery network mapping create -g $VAULT_RG --fabric-name $PRIMARY_FABRIC -n $NETMAP_PRIMARY --network-name $SOURCE_VNET_NAME --vault-name $VAULT_NAME --recovery-network-id $DR_VNET_ID --fabric-details "{azure-to-azure:{primary-network-id:$SOURCE_VNET_ID}}" --recovery-fabric-name $RECOVERY_FABRIC
az site-recovery network mapping create -g $VAULT_RG --fabric-name $RECOVERY_FABRIC -n $NETMAP_RECOVERY --network-name $DR_VNET --vault-name $VAULT_NAME --recovery-network-id $SOURCE_VNET_ID --fabric-details "{azure-to-azure:{primary-network-id:$DR_VNET_ID}}" --recovery-fabric-name $PRIMARY_FABRIC

az site-recovery protected-item create -g $VAULT_RG --fabric-name $PRIMARY_FABRIC -n $RPI_NAME --protection-container $PRIMARY_CONTAINER --vault-name $VAULT_NAME --policy-id $POLICY_ID --provider-details "{a2a:{fabric-object-id:$VM_ID,vm-managed-disks:[{disk-id:$OS_DISK_ID,primary-staging-azure-storage-account-id:$CACHE_ID,recovery-resource-group-id:$RECOVERY_RG_ID},{disk-id:$DATA_DISK_ID,primary-staging-azure-storage-account-id:$CACHE_ID,recovery-resource-group-id:$RECOVERY_RG_ID}],recovery-azure-network-id:$DR_VNET_ID,recovery-container-id:$RECOVERY_CONTAINER_ID,recovery-resource-group-id:$RECOVERY_RG_ID,recovery-subnet-name:$DR_SUBNET}}"

# If the VM has only an OS disk, remove the second vm-managed-disks object before running the protected-item create command.
```

### PowerShell Commands
```powershell
$PrimaryRG = "rg-dr-source"
$RecoveryRG = "rg-dr-recovery"
$PrimaryRegion = "East US"
$RecoveryRegion = "Central US"
$SourceVmName = "vm-dr-source"
$VaultName = "rsvdr$(Get-Random)"
$PolicyName = "a2a-policy"

$vault = New-AzRecoveryServicesVault -Name $VaultName -ResourceGroupName $RecoveryRG -Location $RecoveryRegion
Set-AzRecoveryServicesAsrVaultContext -Vault $vault

$vm = Get-AzVM -ResourceGroupName $PrimaryRG -Name $SourceVmName
$sourceNicId = $vm.NetworkProfile.NetworkInterfaces[0].Id
$sourceNicParts = $sourceNicId.Split('/')
$sourceNic = Get-AzNetworkInterface -ResourceGroupName $sourceNicParts[4] -Name $sourceNicParts[-1]
$primarySubnet = $sourceNic.IpConfigurations[0].Subnet
$primaryVnetId = (Split-Path (Split-Path $primarySubnet.Id)).Replace('\','/')

$drVnet = New-AzVirtualNetwork -Name "vnet-dr" -ResourceGroupName $RecoveryRG -Location $RecoveryRegion -AddressPrefix "10.50.0.0/16"
Add-AzVirtualNetworkSubnetConfig -Name "snet-dr" -VirtualNetwork $drVnet -AddressPrefix "10.50.1.0/24" | Set-AzVirtualNetwork

$cache = New-AzStorageAccount -Name ("drcache{0}" -f (Get-Random)) -ResourceGroupName $PrimaryRG -Location $PrimaryRegion -SkuName Standard_LRS -Kind StorageV2

$job = New-AzRecoveryServicesAsrFabric -Azure -Location $PrimaryRegion -Name "fabric-eastus"
$job = New-AzRecoveryServicesAsrFabric -Azure -Location $RecoveryRegion -Name "fabric-centralus"

$primaryFabric = Get-AzRecoveryServicesAsrFabric -Name "fabric-eastus"
$recoveryFabric = Get-AzRecoveryServicesAsrFabric -Name "fabric-centralus"

$job = New-AzRecoveryServicesAsrProtectionContainer -InputObject $primaryFabric -Name "pc-eastus"
$job = New-AzRecoveryServicesAsrProtectionContainer -InputObject $recoveryFabric -Name "pc-centralus"

$primaryContainer = Get-AzRecoveryServicesAsrProtectionContainer -Fabric $primaryFabric -Name "pc-eastus"
$recoveryContainer = Get-AzRecoveryServicesAsrProtectionContainer -Fabric $recoveryFabric -Name "pc-centralus"

$job = New-AzRecoveryServicesAsrPolicy -AzureToAzure -Name $PolicyName -RecoveryPointRetentionInHours 24 -ApplicationConsistentSnapshotFrequencyInHours 4
$policy = Get-AzRecoveryServicesAsrPolicy -Name $PolicyName

$job = New-AzRecoveryServicesAsrProtectionContainerMapping -Name "map-eastus-centralus" -Policy $policy -PrimaryProtectionContainer $primaryContainer -RecoveryProtectionContainer $recoveryContainer
$job = New-AzRecoveryServicesAsrProtectionContainerMapping -Name "map-centralus-eastus" -Policy $policy -PrimaryProtectionContainer $recoveryContainer -RecoveryProtectionContainer $primaryContainer

$primaryToRecoveryMap = Get-AzRecoveryServicesAsrProtectionContainerMapping -ProtectionContainer $primaryContainer -Name "map-eastus-centralus"

$job = New-AzRecoveryServicesAsrNetworkMapping -AzureToAzure -Name "netmap-eastus-centralus" -PrimaryFabric $primaryFabric -PrimaryAzureNetworkId $primaryVnetId -RecoveryFabric $recoveryFabric -RecoveryAzureNetworkId $drVnet.Id
$job = New-AzRecoveryServicesAsrNetworkMapping -AzureToAzure -Name "netmap-centralus-eastus" -PrimaryFabric $recoveryFabric -PrimaryAzureNetworkId $drVnet.Id -RecoveryFabric $primaryFabric -RecoveryAzureNetworkId $primaryVnetId

$recoveryRg = Get-AzResourceGroup -Name $RecoveryRG
$osDisk = New-AzRecoveryServicesAsrAzureToAzureDiskReplicationConfig -ManagedDisk -LogStorageAccountId $cache.Id -DiskId $vm.StorageProfile.OsDisk.ManagedDisk.Id -RecoveryResourceGroupId $recoveryRg.ResourceId -RecoveryReplicaDiskAccountType Premium_LRS -RecoveryTargetDiskAccountType Premium_LRS
$diskConfigs = @($osDisk)

if ($vm.StorageProfile.DataDisks.Count -gt 0) {
  foreach ($disk in $vm.StorageProfile.DataDisks) {
    $diskConfigs += New-AzRecoveryServicesAsrAzureToAzureDiskReplicationConfig -ManagedDisk -LogStorageAccountId $cache.Id -DiskId $disk.ManagedDisk.Id -RecoveryResourceGroupId $recoveryRg.ResourceId -RecoveryReplicaDiskAccountType Premium_LRS -RecoveryTargetDiskAccountType Premium_LRS
  }
}

$job = New-AzRecoveryServicesAsrReplicationProtectedItem -AzureToAzure -AzureVmId $vm.Id -Name (New-Guid).Guid -ProtectionContainerMapping $primaryToRecoveryMap -AzureToAzureDiskReplicationConfiguration $diskConfigs -RecoveryResourceGroupId $recoveryRg.ResourceId
```

### Verification and Success Criteria
```bash
az site-recovery protected-item show -g $VAULT_RG --vault-name $VAULT_NAME --fabric-name $PRIMARY_FABRIC -n $RPI_NAME --protection-container $PRIMARY_CONTAINER --query "{state:properties.protectionState,health:properties.replicationHealth,policy:properties.policyFriendlyName}" -o table
az site-recovery policy show -g $VAULT_RG --vault-name $VAULT_NAME -n $POLICY_NAME -o table
az storage account show -g $PRIMARY_RG -n $CACHE_SA --query "{name:name,location:primaryLocation,sku:sku.name}" -o table
```

```powershell
Get-AzRecoveryServicesAsrReplicationProtectedItem -ProtectionContainer $primaryContainer | Select-Object FriendlyName, ProtectionState, ReplicationHealth
Get-AzRecoveryServicesAsrPolicy -Name $PolicyName | Select-Object Name
Get-AzRecoveryServicesAsrNetworkMapping -PrimaryFabric $primaryFabric
```

**Success criteria**
- Replication health shows **Normal**.
- Protection state advances to **Protected** after initial replication completes.
- Reverse-direction mappings exist for future failback.

### Cleanup
> If you plan to continue with Labs 2-4, **do not run cleanup yet**.

```bash
az site-recovery protected-item remove -g $VAULT_RG --fabric-name $PRIMARY_FABRIC -n $RPI_NAME --protection-container $PRIMARY_CONTAINER --vault-name $VAULT_NAME --disable-protection-reason MigrationComplete
az site-recovery network mapping delete -g $VAULT_RG --fabric-name $PRIMARY_FABRIC -n $NETMAP_PRIMARY --network-name $SOURCE_VNET_NAME --vault-name $VAULT_NAME --yes
az site-recovery network mapping delete -g $VAULT_RG --fabric-name $RECOVERY_FABRIC -n $NETMAP_RECOVERY --network-name $DR_VNET --vault-name $VAULT_NAME --yes
az site-recovery protection-container mapping delete -g $VAULT_RG --fabric-name $PRIMARY_FABRIC -n $PRIMARY_TO_RECOVERY_MAP --protection-container $PRIMARY_CONTAINER --vault-name $VAULT_NAME --yes
az site-recovery protection-container mapping delete -g $VAULT_RG --fabric-name $RECOVERY_FABRIC -n $RECOVERY_TO_PRIMARY_MAP --protection-container $RECOVERY_CONTAINER --vault-name $VAULT_NAME --yes
az group delete --name $RECOVERY_RG --yes --no-wait
```

```powershell
$replicatedItem = Get-AzRecoveryServicesAsrReplicationProtectedItem -ProtectionContainer $primaryContainer | Select-Object -First 1
Remove-AzRecoveryServicesAsrReplicationProtectedItem -ReplicationProtectedItem $replicatedItem
Remove-AzResourceGroup -Name $RecoveryRG -Force -AsJob
```

### Exam Tip
If the question says **recover a full VM stack in another region with minutes-level RTO**, Azure Site Recovery is usually a better answer than backup restore alone.

### Exam-Style Review Questions
1. Why is Azure Site Recovery a better fit than Azure Backup when the business requires regional failover of a running VM-based application?
2. Why should you create **reverse mappings** before a real disaster happens?
3. What additional design question must you answer besides “Can the VM fail over?” before calling the DR design complete?

---

## Lab 2: ASR Test Failover

### Objective
Run a non-disruptive Azure Site Recovery test failover, validate the replicated VM in an isolated test network, clean up test artifacts, and document the outcome.

### When to Use
Use test failover when the business requires **regular DR validation without impacting production replication**.

### Key AZ-305 Concepts
- Non-disruptive DR drills
- Isolated test networks
- Recovery validation and evidence collection
- Runbook testing vs. production failover

### Exam Domain Mapping
- **Primary:** Design business continuity solutions (15-20%)
- **Secondary:** Design infrastructure solutions (30-35%)

### Prerequisites
- Lab 1 completed and replication state is **Protected**
- A separate test VNet that is not peered to production
- Access to the Recovery Services vault used in Lab 1

### Architecture and Design Rationale
Test failover is the safest way to prove that the DR design works without stopping replication or affecting the production VM. On AZ-305, Microsoft often rewards the answer that includes **regular DR testing with isolation**, because untested DR is not real DR.

### Implementation Steps
1. Create an isolated test VNet in the recovery region.
2. Start a test failover for the replicated item.
3. Validate the test VM boots, is attached to the test VNet, and the workload responds.
4. Document results, issues, and required remediation.
5. Clean up the test failover VM copy.

### IaC Implementation
Reuse the recovery VNet pattern from Lab 1. The **test failover operation itself is operational**, not declarative.

### Azure CLI Commands
> The Azure CLI `site-recovery` extension does not currently expose a first-class A2A `test-failover` verb. Use Azure CLI for setup and validation, then use PowerShell for the actual test failover operation.

```bash
RECOVERY_RG="rg-dr-recovery"
RECOVERY_REGION="centralus"
TEST_VNET="vnet-dr-test"
TEST_SUBNET="snet-test"
SOURCE_VM="vm-dr-source"

az network vnet create --resource-group $RECOVERY_RG --name $TEST_VNET --location $RECOVERY_REGION --address-prefix 10.60.0.0/16 --subnet-name $TEST_SUBNET --subnet-prefix 10.60.1.0/24
az network vnet show --resource-group $RECOVERY_RG --name $TEST_VNET --query "{name:name,subnet:subnets[0].name,addressSpace:addressSpace.addressPrefixes}" -o table

# After running the PowerShell failover below, use CLI for verification.
az vm list -g $RECOVERY_RG -d -o table
```

### PowerShell Commands
```powershell
# Reuse the vault context and ASR objects from Lab 1.
$testVnet = New-AzVirtualNetwork -Name "vnet-dr-test" -ResourceGroupName $RecoveryRG -Location $RecoveryRegion -AddressPrefix "10.60.0.0/16"
Add-AzVirtualNetworkSubnetConfig -Name "snet-test" -VirtualNetwork $testVnet -AddressPrefix "10.60.1.0/24" | Set-AzVirtualNetwork

$primaryContainer = Get-AzRecoveryServicesAsrProtectionContainer -Fabric $primaryFabric -Name "pc-eastus"
$replicatedItem = Get-AzRecoveryServicesAsrReplicationProtectedItem -ProtectionContainer $primaryContainer | Where-Object FriendlyName -eq $SourceVmName

$testFailoverJob = Start-AzRecoveryServicesAsrTestFailoverJob -ReplicationProtectedItem $replicatedItem -AzureVMNetworkId $testVnet.Id -Direction PrimaryToRecovery
Get-AzRecoveryServicesAsrJob -Job $testFailoverJob | Select-Object Name, JobType, State

# After validation, clean up the test failover copy.
$cleanupJob = Start-AzRecoveryServicesAsrTestFailoverCleanupJob -ReplicationProtectedItem $replicatedItem -Comment "DR test completed successfully"
Get-AzRecoveryServicesAsrJob -Job $cleanupJob | Select-Object Name, JobType, State
```

### Verification and Success Criteria
```bash
az vm list -g $RECOVERY_RG -d --query "[].{Name:name,PowerState:powerState,PrivateIPs:privateIps}" -o table
az network vnet show -g $RECOVERY_RG -n $TEST_VNET -o table
```

```powershell
Get-AzVM -ResourceGroupName $RecoveryRG | Select-Object Name, Location
Get-AzRecoveryServicesAsrJob -Job $testFailoverJob | Select-Object Name, State, JobType
```

Use this test record template:

```text
Date:
Protected item:
Recovery point used:
Boot successful (Y/N):
Application checks passed:
Network isolation confirmed:
Issues found:
Action owner:
```

**Success criteria**
- Test VM is created in the recovery region.
- Test VM lands on the isolated test VNet.
- Production replication remains intact.
- Cleanup removes the temporary test failover copy.

### Cleanup
```bash
az network vnet delete --resource-group $RECOVERY_RG --name $TEST_VNET
```

```powershell
$cleanupJob = Start-AzRecoveryServicesAsrTestFailoverCleanupJob -ReplicationProtectedItem $replicatedItem -Comment "TFO cleanup"
Remove-AzVirtualNetwork -Name "vnet-dr-test" -ResourceGroupName $RecoveryRG -Force
```

### Exam Tip
If the requirement says **prove DR readiness without disrupting production**, think **test failover to an isolated test network**.

### Exam-Style Review Questions
1. Why should a DR drill use an isolated test network instead of the actual recovery network?
2. What does a successful test failover prove, and what does it still not prove?
3. Why is documenting DR drill results an architectural best practice instead of just an operations task?

---

## Lab 3: ASR Planned Failover

### Objective
Prepare for a planned failover, execute it to the secondary region, verify the workload after failover, and decide whether to commit or roll back.

### When to Use
Use planned failover for **migration, maintenance windows, or controlled regional evacuation** where the source workload is still available.

### Key AZ-305 Concepts
- Planned failover vs. unplanned failover
- Commit vs. cancel before finalization
- Business-approved maintenance windows
- Recovery validation before commit

### Exam Domain Mapping
- **Primary:** Design business continuity solutions (15-20%)
- **Secondary:** Design infrastructure solutions (30-35%)

### Prerequisites
- Lab 1 completed
- Optional: Lab 2 completed to prove readiness first
- Business approval for a planned service interruption

### Architecture and Design Rationale
Planned failover is the right choice when the source environment is healthy enough to coordinate a controlled switchover. For AZ-305, planned failover is usually favored over unplanned failover when the scenario mentions **maintenance, migration, or intentional relocation**.

### Implementation Steps
1. Confirm replication health is normal.
2. Stop or drain application traffic if required.
3. Execute planned failover to the recovery region.
4. Validate workload startup and dependency connectivity in the secondary region.
5. Commit the failover if validation succeeds; cancel/rollback before commit if it fails.

### IaC Implementation
Not applicable for the failover action itself. Treat this as an **operational runbook** step layered on top of Lab 1 infrastructure.

### Azure CLI Commands
```bash
# Reuse variables from Lab 1.
# Validate health before failover.
az site-recovery protected-item show -g $VAULT_RG --vault-name $VAULT_NAME --fabric-name $PRIMARY_FABRIC -n $RPI_NAME --protection-container $PRIMARY_CONTAINER --query "{state:properties.protectionState,health:properties.replicationHealth}" -o table

# Alternative Azure CLI invocation for planned failover.
az site-recovery protected-item planned-failover --fabric-name $PRIMARY_FABRIC --protection-container $PRIMARY_CONTAINER -n $RPI_NAME -g $VAULT_RG --vault-name $VAULT_NAME --failover-direction PrimaryToRecovery

# If validation succeeds, commit the failover.
az site-recovery protected-item failover-commit --fabric-name $PRIMARY_FABRIC --protection-container $PRIMARY_CONTAINER -n $RPI_NAME -g $VAULT_RG --vault-name $VAULT_NAME
```

### PowerShell Commands
```powershell
$primaryContainer = Get-AzRecoveryServicesAsrProtectionContainer -Fabric $primaryFabric -Name "pc-eastus"
$replicatedItem = Get-AzRecoveryServicesAsrReplicationProtectedItem -ProtectionContainer $primaryContainer | Where-Object FriendlyName -eq $SourceVmName

Get-AzRecoveryServicesAsrReplicationProtectedItem -ProtectionContainer $primaryContainer | Select-Object FriendlyName, ProtectionState, ReplicationHealth

$plannedFailoverJob = Start-AzRecoveryServicesAsrPlannedFailoverJob -ReplicationProtectedItem $replicatedItem -Direction PrimaryToRecovery
Get-AzRecoveryServicesAsrJob -Job $plannedFailoverJob | Select-Object Name, JobType, State

# If validation succeeds, commit.
$commitJob = Start-AzRecoveryServicesAsrCommitFailoverJob -ReplicationProtectedItem $replicatedItem
Get-AzRecoveryServicesAsrJob -Job $commitJob | Select-Object Name, JobType, State

# If validation fails and you have not committed yet, cancel instead.
# $cancelJob = Start-AzRecoveryServicesAsrCancelFailoverJob -ReplicationProtectedItem $replicatedItem
```

### Verification and Success Criteria
```bash
az vm list -g $RECOVERY_RG -d -o table
az site-recovery protected-item show -g $VAULT_RG --vault-name $VAULT_NAME --fabric-name $PRIMARY_FABRIC -n $RPI_NAME --protection-container $PRIMARY_CONTAINER -o json
```

```powershell
Get-AzVM -ResourceGroupName $RecoveryRG | Select-Object Name, Location
Get-AzRecoveryServicesAsrJob -Job $plannedFailoverJob | Select-Object State, JobType
```

**Success criteria**
- Recovery VM starts in the secondary region.
- Application dependencies are reachable.
- Stakeholders confirm the service is healthy before commit.

### Cleanup
> If you plan to continue to Lab 4, keep the post-failover state.

```bash
# No cleanup yet if continuing to failback.
```

```powershell
# No cleanup yet if continuing to failback.
```

### Exam Tip
If Microsoft says **planned migration** or **maintenance**, choose **planned failover**, not unplanned failover.

### Exam-Style Review Questions
1. What is the key business condition that makes planned failover a better answer than unplanned failover?
2. Why should you avoid committing immediately before validation?
3. If the scenario emphasizes controlled maintenance with minimal surprise, what governance step should accompany the failover runbook?

---

## Lab 4: ASR Failback

### Objective
Reprotect the failed-over VM, reverse replication back to the primary region, perform failback, and complete migration back to the source region.

### When to Use
Use failback when the DR event or planned maintenance is over and you need to **return service to the original primary region**.

### Key AZ-305 Concepts
- Reverse replication / reprotect
- Bidirectional container and network mappings
- Recovery-to-primary failover sequence
- Operational readiness for return-to-normal

### Exam Domain Mapping
- **Primary:** Design business continuity solutions (15-20%)
- **Secondary:** Design infrastructure solutions (30-35%)

### Prerequisites
- Lab 3 completed and failover committed to the recovery region
- Reverse mappings from Lab 1 already created
- Source region is healthy and available again

### Architecture and Design Rationale
Failback is often ignored in shallow DR designs, but AZ-305 expects you to think beyond first recovery. A design is incomplete if it can fail over but cannot **re-establish protection and return to normal operations**.

### Implementation Steps
1. Create or confirm a cache storage account in the current active region.
2. Reprotect the workload from recovery back to primary.
3. Wait for replication to return to a healthy state.
4. Execute planned failover in the reverse direction.
5. Commit the failback and validate the original primary region workload.

### IaC Implementation
Not applicable for the protection-direction switch. This is an operational step that depends on the live protected item state.

### Azure CLI Commands
> For A2A reprotect, Azure PowerShell is currently easier to use than Azure CLI because reverse-replication provider details are verbose. Use CLI for monitoring and the final failover/commit steps.

```bash
# Validate current state before failback.
az site-recovery protected-item show -g $VAULT_RG --vault-name $VAULT_NAME --fabric-name $PRIMARY_FABRIC -n $RPI_NAME --protection-container $PRIMARY_CONTAINER -o json

# After PowerShell reprotect completes, perform reverse planned failover if desired from CLI.
az site-recovery protected-item planned-failover --fabric-name $RECOVERY_FABRIC --protection-container $RECOVERY_CONTAINER -n $RPI_NAME -g $VAULT_RG --vault-name $VAULT_NAME --failover-direction RecoveryToPrimary
az site-recovery protected-item failover-commit --fabric-name $RECOVERY_FABRIC --protection-container $RECOVERY_CONTAINER -n $RPI_NAME -g $VAULT_RG --vault-name $VAULT_NAME
```

### PowerShell Commands
```powershell
$recoveryContainer = Get-AzRecoveryServicesAsrProtectionContainer -Fabric $recoveryFabric -Name "pc-centralus"
$replicatedItem = Get-AzRecoveryServicesAsrReplicationProtectedItem -ProtectionContainer $recoveryContainer | Select-Object -First 1
$recoveryToPrimaryMap = Get-AzRecoveryServicesAsrProtectionContainerMapping -ProtectionContainer $recoveryContainer -Name "map-centralus-eastus"
$sourceRg = Get-AzResourceGroup -Name $PrimaryRG
$failbackCache = New-AzStorageAccount -Name ("drfailback{0}" -f (Get-Random)) -ResourceGroupName $RecoveryRG -Location $RecoveryRegion -SkuName Standard_LRS -Kind StorageV2

$reprotectJob = Update-AzRecoveryServicesAsrProtectionDirection -AzureToAzure -ProtectionContainerMapping $recoveryToPrimaryMap -LogStorageAccountId $failbackCache.Id -ReplicationProtectedItem $replicatedItem -RecoveryResourceGroupId $sourceRg.ResourceId
Get-AzRecoveryServicesAsrJob -Job $reprotectJob | Select-Object Name, JobType, State

$reverseFailoverJob = Start-AzRecoveryServicesAsrPlannedFailoverJob -ReplicationProtectedItem $replicatedItem -Direction RecoveryToPrimary
Get-AzRecoveryServicesAsrJob -Job $reverseFailoverJob | Select-Object Name, JobType, State

$commitFailbackJob = Start-AzRecoveryServicesAsrCommitFailoverJob -ReplicationProtectedItem $replicatedItem
Get-AzRecoveryServicesAsrJob -Job $commitFailbackJob | Select-Object Name, JobType, State
```

### Verification and Success Criteria
```bash
az vm list -g $PRIMARY_RG -d -o table
az site-recovery protected-item list -g $VAULT_RG --fabric-name $PRIMARY_FABRIC --protection-container $PRIMARY_CONTAINER --vault-name $VAULT_NAME -o table
```

```powershell
Get-AzVM -ResourceGroupName $PrimaryRG | Select-Object Name, Location
Get-AzRecoveryServicesAsrReplicationProtectedItem -ProtectionContainer $primaryContainer | Select-Object FriendlyName, ProtectionState, ReplicationHealth
```

**Success criteria**
- Replication direction is re-established before failback.
- The workload runs again in the original primary region.
- Protection health returns to normal after failback.

### Cleanup
```bash
az group delete --name $RECOVERY_RG --yes --no-wait
```

```powershell
Remove-AzResourceGroup -Name $RecoveryRG -Force -AsJob
```

### Exam Tip
The best DR answers include **failback**, not just failover. Microsoft often tests whether you remembered the **return path**.

### Exam-Style Review Questions
1. Why is reverse replication required before failback?
2. What design decision in Lab 1 made Lab 4 much easier to execute?
3. Why is a DR design incomplete if it only documents failover and not failback?

---

## Lab 5: SQL Database Geo-Replication

### Objective
Create an Azure SQL Database with a geo-secondary, add it to an auto-failover group, test failover, and monitor replication health.

### When to Use
Use SQL Database geo-replication or auto-failover groups when you need **cross-region database DR with low RPO and managed failover orchestration**.

### Key AZ-305 Concepts
- Active geo-replication vs. failover groups
- Readable secondaries
- Automatic vs. manual failover policy
- Database-layer DR vs. VM-layer DR

### Exam Domain Mapping
- **Primary:** Design business continuity solutions (15-20%)
- **Secondary:** Design data storage solutions (20-25%)

### Prerequisites
- Unique logical server names
- SQL admin login and password prepared
- Two supported regions, for example `eastus2` and `centralus`

### Architecture and Design Rationale
Use SQL-native DR when the exam scenario is primarily about **database continuity**, not whole-VM recovery. Failover groups are usually the strongest AZ-305 answer because they simplify listener endpoints and automatic failover behavior.

### Implementation Steps
1. Create a primary and secondary SQL logical server.
2. Create the primary database.
3. Create a geo-secondary.
4. Add the database to a failover group.
5. Test failover to the secondary server.
6. Monitor replication status and replication lag.

### IaC Implementation
#### Bicep
```bicep
param sqlAdmin string
@secure()
param sqlPassword string
param primaryServer string
param secondaryServer string
param location string = 'eastus2'
param secondaryLocation string = 'centralus'

resource primary 'Microsoft.Sql/servers@2022-05-01-preview' = {
  name: primaryServer
  location: location
  properties: {
    administratorLogin: sqlAdmin
    administratorLoginPassword: sqlPassword
  }
}

resource secondary 'Microsoft.Sql/servers@2022-05-01-preview' = {
  name: secondaryServer
  location: secondaryLocation
  properties: {
    administratorLogin: sqlAdmin
    administratorLoginPassword: sqlPassword
  }
}
```

#### Terraform
```hcl
resource "azurerm_mssql_server" "primary" {
  name                         = var.primary_server
  resource_group_name          = azurerm_resource_group.sql.name
  location                     = var.primary_region
  version                      = "12.0"
  administrator_login          = var.sql_admin
  administrator_login_password = var.sql_password
}

resource "azurerm_mssql_server" "secondary" {
  name                         = var.secondary_server
  resource_group_name          = azurerm_resource_group.sql.name
  location                     = var.secondary_region
  version                      = "12.0"
  administrator_login          = var.sql_admin
  administrator_login_password = var.sql_password
}
```

### Azure CLI Commands
```bash
RG="rg-sql-dr"
PRIMARY_REGION="eastus2"
SECONDARY_REGION="centralus"
PRIMARY_SERVER="sqlpri$RANDOM"
SECONDARY_SERVER="sqlsec$RANDOM"
DB_NAME="sqldrdb"
FAILOVER_GROUP="fog-sqldr"
ADMIN_USER="sqladminuser"
ADMIN_PASS="P@ssw0rd$RANDOM!"

az group create --name $RG --location $PRIMARY_REGION
az sql server create --name $PRIMARY_SERVER --resource-group $RG --location $PRIMARY_REGION --admin-user $ADMIN_USER --admin-password $ADMIN_PASS
az sql server create --name $SECONDARY_SERVER --resource-group $RG --location $SECONDARY_REGION --admin-user $ADMIN_USER --admin-password $ADMIN_PASS
az sql db create --resource-group $RG --server $PRIMARY_SERVER --name $DB_NAME --edition GeneralPurpose --family Gen5 --capacity 2 --compute-model Provisioned
az sql db replica create --resource-group $RG --server $PRIMARY_SERVER --name $DB_NAME --partner-server $SECONDARY_SERVER --secondary-type Geo
az sql failover-group create --name $FAILOVER_GROUP --resource-group $RG --server $PRIMARY_SERVER --partner-server $SECONDARY_SERVER --failover-policy Automatic --grace-period 1 --add-db $DB_NAME

# Test failover.
az sql failover-group set-primary --name $FAILOVER_GROUP --resource-group $RG --server $SECONDARY_SERVER
```

### PowerShell Commands
```powershell
$RG = "rg-sql-dr"
$PrimaryRegion = "East US 2"
$SecondaryRegion = "Central US"
$PrimaryServer = "sqlpri$(Get-Random)"
$SecondaryServer = "sqlsec$(Get-Random)"
$DbName = "sqldrdb"
$FailoverGroup = "fog-sqldr"
$AdminUser = "sqladminuser"
$SqlPassword = ConvertTo-SecureString "P@ssw0rd1234!" -AsPlainText -Force
$Cred = New-Object System.Management.Automation.PSCredential($AdminUser, $SqlPassword)

New-AzResourceGroup -Name $RG -Location $PrimaryRegion
New-AzSqlServer -ResourceGroupName $RG -ServerName $PrimaryServer -Location $PrimaryRegion -SqlAdministratorCredentials $Cred
New-AzSqlServer -ResourceGroupName $RG -ServerName $SecondaryServer -Location $SecondaryRegion -SqlAdministratorCredentials $Cred
New-AzSqlDatabase -ResourceGroupName $RG -ServerName $PrimaryServer -DatabaseName $DbName -Edition GeneralPurpose -ComputeGeneration Gen5 -VCore 2
New-AzSqlDatabaseSecondary -ResourceGroupName $RG -ServerName $PrimaryServer -DatabaseName $DbName -PartnerResourceGroupName $RG -PartnerServerName $SecondaryServer -AllowConnections All
New-AzSqlDatabaseFailoverGroup -ResourceGroupName $RG -ServerName $PrimaryServer -PartnerResourceGroupName $RG -PartnerServerName $SecondaryServer -FailoverGroupName $FailoverGroup -FailoverPolicy Automatic -GracePeriodWithDataLossHours 1 -Database $DbName

# Test failover.
Switch-AzSqlDatabaseFailoverGroup -ResourceGroupName $RG -ServerName $PrimaryServer -FailoverGroupName $FailoverGroup -AllowDataLoss
```

### Verification and Success Criteria
```bash
az sql db replica list-links --resource-group $RG --server $PRIMARY_SERVER --name $DB_NAME -o table
az sql failover-group show --name $FAILOVER_GROUP --resource-group $RG --server $SECONDARY_SERVER -o table
```

```powershell
Get-AzSqlDatabaseReplicationLink -ResourceGroupName $RG -ServerName $PrimaryServer -DatabaseName $DbName
Get-AzSqlDatabaseFailoverGroup -ResourceGroupName $RG -ServerName $SecondaryServer -FailoverGroupName $FailoverGroup
```

**Success criteria**
- Geo-secondary exists and shows a healthy replication link.
- Failover group listener is created.
- Failover successfully makes the partner server primary.

### Cleanup
```bash
az group delete --name $RG --yes --no-wait
```

```powershell
Remove-AzResourceGroup -Name $RG -Force -AsJob
```

### Exam Tip
If the requirement is **SQL PaaS DR with managed failover**, prefer **failover groups** over VM-based SQL DR.

### Exam-Style Review Questions
1. Why is a failover group usually a better answer than only creating a readable geo-secondary?
2. When would SQL Database geo-replication be a better fit than Azure Site Recovery?
3. What does the grace period in an automatic failover policy trade off?

---

## Lab 6: Cosmos DB Multi-Region DR

### Objective
Deploy a multi-region Cosmos DB account, set failover priorities, enable automatic failover, and test a manual failover.

### When to Use
Use Cosmos DB multi-region DR for **globally distributed applications that need low-latency access and region resilience at the database tier**.

### Key AZ-305 Concepts
- Multi-region reads and failover priorities
- Automatic failover
- Session consistency as a practical default
- Platform-managed global database continuity

### Exam Domain Mapping
- **Primary:** Design business continuity solutions (15-20%)
- **Secondary:** Design data storage solutions (20-25%)

### Prerequisites
- A globally unique Cosmos DB account name
- Two or more supported regions
- Basic understanding of consistency levels and write region placement

### Architecture and Design Rationale
Cosmos DB is a strong AZ-305 answer when the scenario calls for **globally distributed NoSQL with built-in regional failover**. It reduces the operational overhead compared with building custom replication on VMs or open-source databases.

### Implementation Steps
1. Create a Cosmos DB account with at least two regions.
2. Enable automatic failover.
3. Review current failover priorities.
4. Trigger a manual failover for testing.
5. Confirm the new write region and failover order.

### IaC Implementation
#### Bicep
```bicep
param accountName string
resource cosmos 'Microsoft.DocumentDB/databaseAccounts@2023-04-15' = {
  name: accountName
  location: 'eastus'
  kind: 'GlobalDocumentDB'
  properties: {
    databaseAccountOfferType: 'Standard'
    locations: [
      {
        locationName: 'eastus'
        failoverPriority: 0
      }
      {
        locationName: 'westus2'
        failoverPriority: 1
      }
    ]
    enableAutomaticFailover: true
    consistencyPolicy: {
      defaultConsistencyLevel: 'Session'
    }
  }
}
```

#### Terraform
```hcl
resource "azurerm_cosmosdb_account" "dr" {
  name                = var.account_name
  location            = var.primary_region
  resource_group_name = azurerm_resource_group.cosmos.name
  offer_type          = "Standard"
  kind                = "GlobalDocumentDB"

  consistency_policy {
    consistency_level = "Session"
  }

  geo_location {
    location          = var.primary_region
    failover_priority = 0
  }

  geo_location {
    location          = var.secondary_region
    failover_priority = 1
  }

  enable_automatic_failover = true
}
```

### Azure CLI Commands
```bash
RG="rg-cosmos-dr"
ACCOUNT="cosmosdr$RANDOM"
PRIMARY_REGION="eastus"
SECONDARY_REGION="westus2"

az group create --name $RG --location $PRIMARY_REGION
az cosmosdb create --name $ACCOUNT --resource-group $RG --default-consistency-level Session --enable-automatic-failover true --locations regionName=$PRIMARY_REGION failoverPriority=0 isZoneRedundant=False regionName=$SECONDARY_REGION failoverPriority=1 isZoneRedundant=False
az cosmosdb show --name $ACCOUNT --resource-group $RG --query "{writeLocations:writeLocations[].locationName,readLocations:readLocations[].locationName,automaticFailover:enableAutomaticFailover}" -o json

# Manual failover test.
az cosmosdb failover-priority-change --name $ACCOUNT --resource-group $RG --failover-policies $SECONDARY_REGION=0 $PRIMARY_REGION=1
```

### PowerShell Commands
```powershell
$RG = "rg-cosmos-dr"
$Account = "cosmosdr$(Get-Random)"
$PrimaryRegion = "East US"
$SecondaryRegion = "West US 2"

New-AzResourceGroup -Name $RG -Location $PrimaryRegion
$loc1 = New-AzCosmosDBLocationObject -LocationName $PrimaryRegion -FailoverPriority 0 -IsZoneRedundant $false
$loc2 = New-AzCosmosDBLocationObject -LocationName $SecondaryRegion -FailoverPriority 1 -IsZoneRedundant $false
New-AzCosmosDBAccount -ResourceGroupName $RG -Name $Account -LocationObject @($loc1,$loc2) -DefaultConsistencyLevel Session -EnableAutomaticFailover $true

# Manual failover test.
Update-AzCosmosDBAccountFailoverPriority -ResourceGroupName $RG -Name $Account -FailoverPolicy @{$SecondaryRegion=0; $PrimaryRegion=1}
```

### Verification and Success Criteria
```bash
az cosmosdb show --name $ACCOUNT --resource-group $RG --query "{writeLocations:writeLocations[].locationName,readLocations:readLocations[].locationName,failoverPolicies:failoverPolicies}" -o json
```

```powershell
Get-AzCosmosDBAccount -ResourceGroupName $RG -Name $Account | Select-Object Name, EnableAutomaticFailover, WriteLocations, ReadLocations
```

**Success criteria**
- Automatic failover is enabled.
- Region priorities are visible and correct.
- Manual failover changes the write region successfully.

### Cleanup
```bash
az group delete --name $RG --yes --no-wait
```

```powershell
Remove-AzResourceGroup -Name $RG -Force -AsJob
```

### Exam Tip
For globally distributed NoSQL applications, Cosmos DB multi-region capability is usually a better exam answer than building your own DR on VMs.

### Exam-Style Review Questions
1. Why is Cosmos DB often a stronger DR answer than self-managed NoSQL on IaaS VMs?
2. What does automatic failover solve, and what does it not solve by itself?
3. Why is session consistency commonly the balanced exam answer for many globally distributed workloads?

---

## Lab 7: Traffic Manager for DNS Failover

### Objective
Create a Traffic Manager profile with priority routing, add primary and secondary endpoints, configure health probes, and test DNS-based failover.

### When to Use
Use Traffic Manager when you need **DNS-based application failover across regions or across different endpoint types**.

### Key AZ-305 Concepts
- DNS-based failover
- Priority routing method
- Probe path and DNS TTL impact on recovery time
- App-layer failover vs. database failover

### Exam Domain Mapping
- **Primary:** Design business continuity solutions (15-20%)
- **Secondary:** Design infrastructure solutions (30-35%)

### Prerequisites
- Two publicly reachable app endpoints, such as two App Service default hostnames
- Health endpoint available on each app, for example `/health`
- Unique Traffic Manager DNS label

### Architecture and Design Rationale
Traffic Manager is best when the requirement is **global DNS routing** rather than a reverse-proxy layer. It is simple and cost-effective, but AZ-305 expects you to remember that **DNS caching affects failover time**.

### Implementation Steps
1. Create a Traffic Manager profile with priority routing.
2. Configure health probes.
3. Add the primary and secondary endpoints.
4. Resolve the Traffic Manager DNS name.
5. Disable the primary endpoint temporarily and observe failover.

### IaC Implementation
#### Bicep
```bicep
resource tm 'Microsoft.Network/trafficManagerProfiles@2022-04-01' = {
  name: 'tm-dr-profile'
  location: 'global'
  properties: {
    profileStatus: 'Enabled'
    trafficRoutingMethod: 'Priority'
    dnsConfig: {
      relativeName: 'tm-dr-profile-12345'
      ttl: 30
    }
    monitorConfig: {
      protocol: 'HTTPS'
      port: 443
      path: '/health'
    }
  }
}
```

#### Terraform
```hcl
resource "azurerm_traffic_manager_profile" "dr" {
  name                   = "tm-dr-profile"
  resource_group_name    = azurerm_resource_group.tm.name
  traffic_routing_method = "Priority"

  dns_config {
    relative_name = "tm-dr-profile-12345"
    ttl           = 30
  }

  monitor_config {
    protocol = "HTTPS"
    port     = 443
    path     = "/health"
  }
}
```

### Azure CLI Commands
```bash
RG="rg-tm-dr"
TM_PROFILE="tm-dr-profile"
TM_DNS="tm-dr-$RANDOM"
PRIMARY_FQDN="primary-app.azurewebsites.net"
SECONDARY_FQDN="secondary-app.azurewebsites.net"

az group create --name $RG --location eastus
az network traffic-manager profile create --resource-group $RG --name $TM_PROFILE --routing-method Priority --unique-dns-name $TM_DNS --ttl 30 --protocol HTTPS --port 443 --path /health
az network traffic-manager endpoint create --resource-group $RG --profile-name $TM_PROFILE --name primary-endpoint --type externalEndpoints --priority 1 --target $PRIMARY_FQDN --endpoint-status Enabled
az network traffic-manager endpoint create --resource-group $RG --profile-name $TM_PROFILE --name secondary-endpoint --type externalEndpoints --priority 2 --target $SECONDARY_FQDN --endpoint-status Enabled

# Simulate failure.
az network traffic-manager endpoint update --resource-group $RG --profile-name $TM_PROFILE --name primary-endpoint --type externalEndpoints --endpoint-status Disabled
```

### PowerShell Commands
```powershell
$RG = "rg-tm-dr"
$Profile = "tm-dr-profile"
$DnsName = "tm-dr-$(Get-Random)"

New-AzResourceGroup -Name $RG -Location "East US"
$tm = New-AzTrafficManagerProfile -Name $Profile -ResourceGroupName $RG -TrafficRoutingMethod Priority -RelativeDnsName $DnsName -Ttl 30 -MonitorProtocol HTTPS -MonitorPort 443 -MonitorPath "/health"
New-AzTrafficManagerEndpoint -Name "primary-endpoint" -ProfileName $Profile -ResourceGroupName $RG -Type ExternalEndpoints -Target "primary-app.azurewebsites.net" -EndpointStatus Enabled -Priority 1
New-AzTrafficManagerEndpoint -Name "secondary-endpoint" -ProfileName $Profile -ResourceGroupName $RG -Type ExternalEndpoints -Target "secondary-app.azurewebsites.net" -EndpointStatus Enabled -Priority 2

# Simulate failure.
Disable-AzTrafficManagerEndpoint -Name "primary-endpoint" -ProfileName $Profile -ResourceGroupName $RG -Type ExternalEndpoints -Force
```

### Verification and Success Criteria
```bash
az network traffic-manager profile show --resource-group $RG --name $TM_PROFILE -o table
az network traffic-manager endpoint list --resource-group $RG --profile-name $TM_PROFILE --type externalEndpoints -o table
```

```powershell
Resolve-DnsName "$DnsName.trafficmanager.net"
Get-AzTrafficManagerEndpoint -ProfileName $Profile -ResourceGroupName $RG -Type ExternalEndpoints | Select-Object Name, EndpointStatus, Priority, Target
```

**Success criteria**
- Both endpoints are registered with correct priorities.
- Health probe settings point to a working endpoint path.
- DNS eventually resolves toward the secondary endpoint when the primary is disabled.

### Cleanup
```bash
az group delete --name $RG --yes --no-wait
```

```powershell
Remove-AzResourceGroup -Name $RG -Force -AsJob
```

### Exam Tip
Traffic Manager is **DNS-based**, not a reverse proxy. If the scenario demands faster HTTP failover plus WAF and acceleration, consider **Front Door** instead.

### Exam-Style Review Questions
1. Why can Traffic Manager failover feel slower to users than Front Door failover?
2. When is Traffic Manager a better answer than Front Door?
3. What application design assumption must still be true even if DNS failover works?

---

## Lab 8: Front Door for Global DR

### Objective
Create Azure Front Door Standard/Premium with multiple origins, configure health probes and failover routing, and test origin failure.

### When to Use
Use Front Door when you need **global HTTP/HTTPS application failover with faster app-layer routing, health probes, and optional WAF support**.

### Key AZ-305 Concepts
- Global layer 7 failover
- Origin groups, priorities, and health probes
- HTTP acceleration and centralized ingress
- When Front Door is preferred over Traffic Manager

### Exam Domain Mapping
- **Primary:** Design business continuity solutions (15-20%)
- **Secondary:** Design infrastructure solutions (30-35%), Design identity, governance, and monitoring solutions (25-30%)

### Prerequisites
- Two public web origins, such as two App Services in separate regions
- Health endpoint exposed on both origins
- Microsoft.Cdn provider registered

### Architecture and Design Rationale
Front Door is a strong exam answer for **global web application DR** because it combines health probing, global routing, and optional security controls at the HTTP layer. It generally gives a better user experience than DNS-only failover.

### Implementation Steps
1. Create an Azure Front Door Standard profile.
2. Create an endpoint and origin group.
3. Add primary and secondary origins with different priorities.
4. Create a route that sends traffic to the origin group.
5. Disable the primary origin temporarily and verify failover.

### IaC Implementation
#### Bicep
```bicep
resource profile 'Microsoft.Cdn/profiles@2023-05-01' = {
  name: 'afd-dr-profile'
  location: 'Global'
  sku: {
    name: 'Standard_AzureFrontDoor'
  }
}
```

#### Terraform
```hcl
resource "azurerm_cdn_frontdoor_profile" "afd" {
  name                = "afd-dr-profile"
  resource_group_name = azurerm_resource_group.afd.name
  sku_name            = "Standard_AzureFrontDoor"
}
```

### Azure CLI Commands
```bash
RG="rg-afd-dr"
PROFILE="afd-dr-profile"
ENDPOINT="afd-endpoint"
ORIGIN_GROUP="dr-origin-group"
PRIMARY_ORIGIN="primary-origin"
SECONDARY_ORIGIN="secondary-origin"
ROUTE="route-default"
PRIMARY_HOST="primary-app.azurewebsites.net"
SECONDARY_HOST="secondary-app.azurewebsites.net"

az group create --name $RG --location eastus
az afd profile create --resource-group $RG --profile-name $PROFILE --sku Standard_AzureFrontDoor
az afd endpoint create --resource-group $RG --profile-name $PROFILE --endpoint-name $ENDPOINT --enabled-state Enabled
az afd origin-group create --resource-group $RG --profile-name $PROFILE --origin-group-name $ORIGIN_GROUP --probe-request-type GET --probe-protocol Https --probe-interval-in-seconds 30 --sample-size 4 --successful-samples-required 3 --additional-latency-in-milliseconds 50
az afd origin create --resource-group $RG --profile-name $PROFILE --origin-group-name $ORIGIN_GROUP --origin-name $PRIMARY_ORIGIN --host-name $PRIMARY_HOST --origin-host-header $PRIMARY_HOST --http-port 80 --https-port 443 --priority 1 --weight 1000 --enabled-state Enabled
az afd origin create --resource-group $RG --profile-name $PROFILE --origin-group-name $ORIGIN_GROUP --origin-name $SECONDARY_ORIGIN --host-name $SECONDARY_HOST --origin-host-header $SECONDARY_HOST --http-port 80 --https-port 443 --priority 2 --weight 1000 --enabled-state Enabled
az afd route create --resource-group $RG --profile-name $PROFILE --endpoint-name $ENDPOINT --route-name $ROUTE --origin-group $ORIGIN_GROUP --supported-protocols Http Https --patterns-to-match '/*' --forwarding-protocol MatchRequest --https-redirect Enabled --link-to-default-domain Enabled

# Simulate failure.
az afd origin update --resource-group $RG --profile-name $PROFILE --origin-group-name $ORIGIN_GROUP --origin-name $PRIMARY_ORIGIN --enabled-state Disabled
```

### PowerShell Commands
```powershell
$RG = "rg-afd-dr"
$Profile = "afd-dr-profile"
$Endpoint = "afd-endpoint"
$OriginGroup = "dr-origin-group"

New-AzResourceGroup -Name $RG -Location "East US"
New-AzFrontDoorCdnProfile -ResourceGroupName $RG -Name $Profile -SkuName Standard_AzureFrontDoor -Location Global
New-AzFrontDoorCdnEndpoint -ResourceGroupName $RG -ProfileName $Profile -EndpointName $Endpoint -EnabledState Enabled
$probe = New-AzFrontDoorCdnOriginGroupHealthProbeSettingObject -ProbeIntervalInSecond 30 -ProbePath "/health" -ProbeProtocol Https -ProbeRequestType GET
$lb = New-AzFrontDoorCdnOriginGroupLoadBalancingSettingObject -AdditionalLatencyInMillisecond 50 -SampleSize 4 -SuccessfulSamplesRequired 3
New-AzFrontDoorCdnOriginGroup -ResourceGroupName $RG -ProfileName $Profile -OriginGroupName $OriginGroup -HealthProbeSetting $probe -LoadBalancingSetting $lb
New-AzFrontDoorCdnOrigin -ResourceGroupName $RG -ProfileName $Profile -OriginGroupName $OriginGroup -OriginName "primary-origin" -HostName "primary-app.azurewebsites.net" -OriginHostHeader "primary-app.azurewebsites.net" -HttpPort 80 -HttpsPort 443 -Priority 1 -Weight 1000
New-AzFrontDoorCdnOrigin -ResourceGroupName $RG -ProfileName $Profile -OriginGroupName $OriginGroup -OriginName "secondary-origin" -HostName "secondary-app.azurewebsites.net" -OriginHostHeader "secondary-app.azurewebsites.net" -HttpPort 80 -HttpsPort 443 -Priority 2 -Weight 1000
$originGroupObj = Get-AzFrontDoorCdnOriginGroup -ResourceGroupName $RG -ProfileName $Profile -OriginGroupName $OriginGroup
New-AzFrontDoorCdnRoute -ResourceGroupName $RG -ProfileName $Profile -EndpointName $Endpoint -Name "route-default" -OriginGroupId $originGroupObj.Id -PatternsToMatch "/*" -SupportedProtocol Http,Https -LinkToDefaultDomain Enabled -HttpsRedirect Enabled -EnabledState Enabled -ForwardingProtocol MatchRequest

# Simulate failure.
Update-AzFrontDoorCdnOrigin -ResourceGroupName $RG -ProfileName $Profile -OriginGroupName $OriginGroup -OriginName "primary-origin" -EnabledState Disabled
```

### Verification and Success Criteria
```bash
az afd origin-group show --resource-group $RG --profile-name $PROFILE --origin-group-name $ORIGIN_GROUP -o json
az afd origin list --resource-group $RG --profile-name $PROFILE --origin-group-name $ORIGIN_GROUP -o table
az afd route show --resource-group $RG --profile-name $PROFILE --endpoint-name $ENDPOINT --route-name $ROUTE -o table
```

```powershell
Get-AzFrontDoorCdnOrigin -ResourceGroupName $RG -ProfileName $Profile -OriginGroupName $OriginGroup | Select-Object Name, HostName, Priority, EnabledState
Get-AzFrontDoorCdnRoute -ResourceGroupName $RG -ProfileName $Profile -EndpointName $Endpoint -RouteName "route-default"
```

**Success criteria**
- Front Door profile and endpoint are active.
- Primary and secondary origins are in the same origin group with the expected priorities.
- Disabling the primary origin causes traffic to move to the secondary origin.

### Cleanup
```bash
az group delete --name $RG --yes --no-wait
```

```powershell
Remove-AzResourceGroup -Name $RG -Force -AsJob
```

### Exam Tip
For **global HTTP apps**, Front Door is usually the strongest answer when the scenario mentions **fast failover, health probes, or WAF**.

### Exam-Style Review Questions
1. Why is Front Door usually a better answer than Traffic Manager for interactive web apps with strict user-experience requirements?
2. What does Front Door protect, and what must still be designed separately at the data tier?
3. If the app needs both global DR and web application firewall capabilities, what service should you think of first?

---

## Lab 9: Storage Account Failover

### Objective
Create a geo-redundant storage account, review last sync time, initiate customer-managed failover, and verify data availability after failover.

### When to Use
Use storage account failover when you need **storage-level DR for blob, file, queue, or table data** and the application can tolerate storage account endpoint changes during failover.

### Key AZ-305 Concepts
- GRS and RA-GRS
- Customer-managed failover
- Last sync time and potential data loss window
- Durability vs. application continuity

### Exam Domain Mapping
- **Primary:** Design business continuity solutions (15-20%)
- **Secondary:** Design data storage solutions (20-25%)

### Prerequisites
- Unique storage account name
- Understanding that failover promotes the secondary region and can involve data loss up to the last sync point
- Non-production subscription only

### Architecture and Design Rationale
Storage account failover is excellent when the exam is about **storage durability and regional recovery**, but remember it does **not automatically fail over your entire application**. On AZ-305, storage redundancy and application failover are related but different decisions.

### Implementation Steps
1. Create a GRS or RA-GRS storage account.
2. Upload a test blob.
3. Review `lastSyncTime`.
4. Trigger planned or unplanned customer-managed failover.
5. Verify the account now serves from the promoted region.

### IaC Implementation
#### Bicep
```bicep
resource sa 'Microsoft.Storage/storageAccounts@2023-05-01' = {
  name: 'drstorage12345'
  location: 'eastus2'
  sku: {
    name: 'Standard_RAGRS'
  }
  kind: 'StorageV2'
  properties: {
    supportsHttpsTrafficOnly: true
  }
}
```

#### Terraform
```hcl
resource "azurerm_storage_account" "dr" {
  name                     = var.storage_account_name
  resource_group_name      = azurerm_resource_group.storage.name
  location                 = var.primary_region
  account_tier             = "Standard"
  account_replication_type = "RAGRS"
}
```

### Azure CLI Commands
```bash
RG="rg-storage-dr"
LOCATION="eastus2"
SA="drstore$RANDOM"
CONTAINER="drtest"

az group create --name $RG --location $LOCATION
az storage account create --name $SA --resource-group $RG --location $LOCATION --sku Standard_RAGRS --kind StorageV2
az storage container create --name $CONTAINER --account-name $SA --auth-mode login
printf 'dr validation blob' > dr-test.txt
az storage blob upload --account-name $SA --container-name $CONTAINER --name dr-test.txt --file dr-test.txt --auth-mode login
az storage account show --name $SA --resource-group $RG --expand geoReplicationStats --query "{primary:primaryLocation,secondary:secondaryLocation,lastSync:geoReplicationStats.lastSyncTime,status:geoReplicationStats.status}" -o table

# Planned failover.
az storage account failover --name $SA --resource-group $RG --failover-type Planned --yes
```

### PowerShell Commands
```powershell
$RG = "rg-storage-dr"
$Location = "East US 2"
$StorageName = "drstore$(Get-Random)"
$Container = "drtest"

New-AzResourceGroup -Name $RG -Location $Location
$sa = New-AzStorageAccount -ResourceGroupName $RG -Name $StorageName -Location $Location -SkuName Standard_RAGRS -Kind StorageV2
$ctx = $sa.Context
New-AzStorageContainer -Name $Container -Context $ctx
"dr validation blob" | Set-Content -Path .\dr-test.txt
Set-AzStorageBlobContent -Context $ctx -Container $Container -File .\dr-test.txt -Blob dr-test.txt
Get-AzStorageAccount -ResourceGroupName $RG -Name $StorageName | Select-Object StorageAccountName, PrimaryLocation, SecondaryLocation

Invoke-AzStorageAccountFailover -ResourceGroupName $RG -Name $StorageName
```

### Verification and Success Criteria
```bash
az storage account show --name $SA --resource-group $RG --expand geoReplicationStats -o json
```

```powershell
Get-AzStorageAccount -ResourceGroupName $RG -Name $StorageName | Select-Object StorageAccountName, PrimaryLocation, SecondaryLocation, ProvisioningState
```

**Success criteria**
- Geo-replication stats show a recent last sync time.
- Failover completes successfully.
- Storage account primary region changes after failover.

### Cleanup
```bash
rm -f dr-test.txt
az group delete --name $RG --yes --no-wait
```

```powershell
Remove-Item -Path .\dr-test.txt -ErrorAction SilentlyContinue
Remove-AzResourceGroup -Name $RG -Force -AsJob
```

### Exam Tip
GRS/RA-GRS improve **storage durability**, but they are not a substitute for **application DR orchestration**.

### Exam-Style Review Questions
1. Why does `lastSyncTime` matter before triggering storage failover?
2. Why is storage failover not enough by itself for a multi-tier application?
3. When would RA-GRS be preferred over GRS?

---

## Lab 10: DR Decision Exercise

### Objective
Practice mapping business scenarios to the correct Azure DR pattern, service selection, and RTO/RPO tradeoff.

### When to Use
Use this exercise for **AZ-305 exam prep** when the question is mainly architectural rather than operational.

### Key AZ-305 Concepts
- RTO and RPO interpretation
- Backup vs. HA vs. DR
- Active-active vs. active-passive vs. warm standby vs. pilot light
- Platform-managed resiliency choices

### Exam Domain Mapping
- **Primary:** Design business continuity solutions (15-20%)
- **Secondary:** All other AZ-305 domains depending on workload type

### Prerequisites
- Review Labs 1-9 first
- Be comfortable identifying data-tier vs. app-tier DR requirements

### Architecture and Design Rationale
This lab is about the most important AZ-305 skill: picking the **lowest-cost, lowest-ops architecture that still meets the stated RTO and RPO**. Microsoft often rewards the answer that is sufficient and realistic, not the most complex design.

### Implementation Steps
1. Read each scenario.
2. Identify the likely DR pattern.
3. Choose Azure services.
4. Justify the answer in terms of RTO, RPO, cost, and operations.
5. Compare your answers with the key below.

### IaC Implementation
Not applicable. This is a design exercise.

### Azure CLI Commands
```bash
cat > dr-decision-answers.txt <<'EOF'
Scenario 1:
Pattern:
Services:
Why:

Scenario 2:
Pattern:
Services:
Why:

Scenario 3:
Pattern:
Services:
Why:

Scenario 4:
Pattern:
Services:
Why:

Scenario 5:
Pattern:
Services:
Why:

Scenario 6:
Pattern:
Services:
Why:

Scenario 7:
Pattern:
Services:
Why:

Scenario 8:
Pattern:
Services:
Why:
EOF
```

### PowerShell Commands
```powershell
@'
Scenario 1:
Pattern:
Services:
Why:

Scenario 2:
Pattern:
Services:
Why:

Scenario 3:
Pattern:
Services:
Why:

Scenario 4:
Pattern:
Services:
Why:

Scenario 5:
Pattern:
Services:
Why:

Scenario 6:
Pattern:
Services:
Why:

Scenario 7:
Pattern:
Services:
Why:

Scenario 8:
Pattern:
Services:
Why:
'@ | Set-Content -Path .\dr-decision-answers.txt
```

### Scenarios

| # | Scenario | RTO | RPO |
|---|---|---:|---:|
| 1 | Legacy 3-tier app on Azure VMs must survive a regional outage with recovery in under 60 minutes and less than 15 minutes of data loss. | < 1 hour | < 15 min |
| 2 | Global e-commerce web tier must stay online even during a regional outage, with near-instant user redirection. | Minutes | Minutes |
| 3 | Azure SQL Database backs an order system and must automatically fail over across regions with minimal operational overhead. | Minutes | Seconds |
| 4 | Global mobile app uses a NoSQL database and needs low-latency reads in multiple regions plus automatic regional failover. | Seconds to minutes | Near-zero to seconds |
| 5 | File archive data must remain durable if a region is lost, but the business can tolerate hours before access is restored. | Hours | Minutes to hours |
| 6 | A line-of-business app can be rebuilt from code, but core identity/configuration services must remain available in another region within 2 hours. | < 2 hours | < 1 hour |
| 7 | Critical customer portal must provide HTTP failover, TLS termination, health probes, and optional WAF at the global edge. | Minutes | App dependent |
| 8 | A DR compliance program requires regular non-disruptive proof that the VM recovery process works without affecting production. | N/A | N/A |

### Answer Key with Reasoning

| # | Best Pattern | Likely Services | Reasoning |
|---|---|---|---|
| 1 | Warm standby / active-passive DR | Azure Site Recovery, Recovery Services vault, Traffic Manager or Front Door, paired-region networking | VM-based app needs compute continuity, not only backups. ASR best matches VM recovery with low-ish RTO and minutes-level replication. |
| 2 | Active-active or hot standby at web tier | Azure Front Door, multi-region web apps, replicated data tier | Requirement emphasizes fast user redirection and global web continuity. DNS-only failover may be too slow. |
| 3 | Managed database DR | SQL Database failover group | Native database failover is lower-ops and better aligned than VM-based SQL DR for PaaS SQL. |
| 4 | Multi-region managed database | Cosmos DB multi-region with automatic failover | Cosmos DB directly solves global distribution and regional failover with managed replication. |
| 5 | Storage DR / durability pattern | GRS or RA-GRS storage account, optional customer-managed failover | Requirement is storage durability, not full app continuity. A storage redundancy answer is sufficient. |
| 6 | Pilot light | IaC, smaller DR footprint, replicated config/state, optional ASR for critical components | App can be rebuilt, so full warm standby is not required. Keep only critical services hot enough to meet the 2-hour objective. |
| 7 | Global HTTP DR | Azure Front Door Standard/Premium | Front Door adds layer 7 failover, probes, TLS, and optional WAF; this is stronger than Traffic Manager for the stated requirements. |
| 8 | Non-disruptive DR validation | Azure Site Recovery test failover | Requirement is proof and testing, not production cutover. Test failover is purpose-built for this. |

### Verification and Success Criteria
- You can explain **why** each chosen service matches the RTO/RPO.
- You can explain why at least one obvious alternative is worse.
- You do not confuse **backup** with **DR** or **HA**.

### Cleanup
```bash
rm -f dr-decision-answers.txt
```

```powershell
Remove-Item -Path .\dr-decision-answers.txt -ErrorAction SilentlyContinue
```

### Exam Tip
Always translate the scenario into three questions first: **What is the RTO? What is the RPO? Is this backup, HA, or DR?**

### Exam-Style Review Questions
1. Why is “most resilient” not always the best answer on AZ-305?
2. How do you decide between warm standby and pilot light?
3. Why is a platform-managed PaaS DR option often the better exam answer than a custom IaaS design?

---

## DR Pattern Decision Summary

| Requirement Pattern | Best-Fit Azure Approach | Why |
|---|---|---|
| VM-level regional DR | Azure Site Recovery | Replicates full VM stack and supports failover/failback |
| Managed relational database DR | SQL Database failover group | Lower ops, listener abstraction, automatic failover |
| Global NoSQL DR | Cosmos DB multi-region | Native global distribution and failover priorities |
| Global web failover with WAF/HTTP routing | Azure Front Door | Layer 7 routing, probes, TLS offload, optional WAF |
| DNS-based endpoint failover | Traffic Manager | Simple and cost-effective global DNS failover |
| Storage durability across regions | GRS / RA-GRS / GZRS | Storage-layer resiliency and optional customer-managed failover |
| Lowest-cost recovery | Backup + restore or pilot light | Lower steady-state cost, higher RTO |
| Fastest regional app recovery | Active-active or hot standby | Lowest RTO, highest complexity and cost |

## RTO/RPO Mapping Table

| Target RTO | Target RPO | Common Pattern | Typical Azure Services | AZ-305 Takeaway |
|---|---|---|---|---|
| Seconds | Near-zero | Active-active | Front Door, Cosmos DB multi-write, zone-redundant PaaS | Highest cost and complexity |
| Minutes | Seconds to minutes | Hot standby | Front Door, SQL failover group, Cosmos DB multi-region | Strong managed DR answer |
| < 1 hour | < 15 minutes | Warm standby | Azure Site Recovery, scaled-down secondary region, replicated data tier | Common enterprise DR pattern |
| 1-4 hours | < 1 hour | Pilot light | IaC + replicated core services + delayed app scale-out | Lower cost, slower recovery |
| Hours to day | Hours | Backup + restore | Azure Backup, storage redundancy, PITR | Good for recovery, not true app continuity |

> **Final AZ-305 memory hook:**  
> **Backup** protects data, **HA** protects against local failure, and **DR** protects against major or regional failure. Choose the **simplest architecture** that still satisfies the stated **RTO** and **RPO**.