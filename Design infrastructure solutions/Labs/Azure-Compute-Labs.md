# Azure Compute Hands-On Labs (AZ-305)

> 📖 **Cheat Sheet:** [Azure Compute](../Azure-Compute.md)

> **Primary exam domain:** Design infrastructure solutions (30-35%)  
> **Tools used:** Azure CLI, Azure PowerShell, Azure Monitor, Azure Backup, AKS, App Service, Container Apps, Azure Functions, Azure Batch, Azure Advisor  
> **Important:** Use a non-production subscription or isolated resource groups. Several labs create billable compute resources, premium storage, backups, monitoring, and ingress components.

---

## How to Use These Labs

- Run each lab in its own resource group to simplify cleanup.
- Replace globally unique names where required.
- Register providers before starting if your subscription is new:

```bash
az provider register --namespace Microsoft.Compute
az provider register --namespace Microsoft.Network
az provider register --namespace Microsoft.Insights
az provider register --namespace Microsoft.RecoveryServices
az provider register --namespace Microsoft.ContainerService
az provider register --namespace Microsoft.Web
az provider register --namespace Microsoft.App
az provider register --namespace Microsoft.OperationalInsights
az provider register --namespace Microsoft.ContainerRegistry
az provider register --namespace Microsoft.Batch
```

```powershell
Register-AzResourceProvider -ProviderNamespace Microsoft.Compute
Register-AzResourceProvider -ProviderNamespace Microsoft.Network
Register-AzResourceProvider -ProviderNamespace Microsoft.Insights
Register-AzResourceProvider -ProviderNamespace Microsoft.RecoveryServices
Register-AzResourceProvider -ProviderNamespace Microsoft.ContainerService
Register-AzResourceProvider -ProviderNamespace Microsoft.Web
Register-AzResourceProvider -ProviderNamespace Microsoft.App
Register-AzResourceProvider -ProviderNamespace Microsoft.OperationalInsights
Register-AzResourceProvider -ProviderNamespace Microsoft.ContainerRegistry
Register-AzResourceProvider -ProviderNamespace Microsoft.Batch
```

---

## Lab 1: Virtual Machine Deployment with High Availability

### Objective
Deploy a production-style Windows or Linux VM architecture using Availability Zones, Premium SSD managed disks, Azure Backup, and diagnostics settings.

### When to Use This
Use Azure VMs when the workload needs full OS control, custom software, domain join, legacy dependencies, or specialized marketplace images. Use Availability Zones for production VMs that need higher availability within a region.

### Key AZ-305 Concepts
- Availability Set vs. Availability Zone vs. regional disaster recovery
- Premium SSD managed disks for predictable IOPS/latency
- Azure Backup vault design and policy assignment
- Boot diagnostics and platform metrics/log routing
- Bastion/private administration vs. public IP exposure

### Exam Domain Mapping
- **Primary:** Design infrastructure solutions (30-35%)
- **Secondary:** Design business continuity solutions (15-20%), Design identity, governance, and monitoring solutions (25-30%)

### Prerequisites
- Contributor on the subscription/resource group
- Azure CLI logged in with `az login`
- Azure PowerShell logged in with `Connect-AzAccount`
- SSH key pair or local admin credentials prepared
- Region that supports Availability Zones, such as `eastus2`

### Architecture and Design Rationale
Availability Zones protect against datacenter-level failure inside a region, while Availability Sets only protect against host and rack-level failures within a datacenter boundary. For AZ-305, prefer **Availability Zones** for new production VM workloads when zone support exists; choose **Availability Sets** mainly for legacy patterns or when zones are unavailable.

| Option | Best for | Strength | Limitation |
|---|---|---|---|
| Availability Set | Legacy HA in one datacenter stamp | Separates fault and update domains | No datacenter-level isolation |
| Availability Zone | Production HA in-zone-enabled region | Separate datacenters, stronger SLA story | Slightly higher complexity and possible zonal service constraints |
| Multi-region | Disaster recovery | Regional resilience | Higher cost and operational complexity |

### Implementation Steps
1. Create a zonal VNet, subnet, NSG, and public IP.
2. Deploy a VM into Zone 1 with Premium SSD OS disk.
3. Enable boot diagnostics and route diagnostics to a Log Analytics workspace.
4. Create a Recovery Services vault and protect the VM with Azure Backup.
5. Review how the design would differ if an Availability Set were chosen.

### Full CLI + PowerShell Commands

#### Azure CLI
```bash
RG="rg-az305-vm-ha"
LOCATION="eastus2"
VNET="vnet-vm-ha"
SUBNET="snet-app"
NSG="nsg-vm-ha"
PIP="pip-vm-ha"
NIC="nic-vm-ha"
VM="vmha$RANDOM"
ADMINUSER="azureuser"
VAULT="rsvmha$RANDOM"
LAW="lawvmha$RANDOM"

az group create --name $RG --location $LOCATION
az network vnet create --resource-group $RG --name $VNET --location $LOCATION --address-prefix 10.10.0.0/16 --subnet-name $SUBNET --subnet-prefix 10.10.1.0/24
az network nsg create --resource-group $RG --name $NSG --location $LOCATION
az network nsg rule create --resource-group $RG --nsg-name $NSG --name allow-ssh --priority 1000 --access Allow --direction Inbound --protocol Tcp --destination-port-ranges 22
az network public-ip create --resource-group $RG --name $PIP --location $LOCATION --sku Standard --zone 1
az network nic create --resource-group $RG --name $NIC --vnet-name $VNET --subnet $SUBNET --network-security-group $NSG --public-ip-address $PIP

az vm create \
  --resource-group $RG \
  --name $VM \
  --nics $NIC \
  --image Ubuntu2204 \
  --admin-username $ADMINUSER \
  --generate-ssh-keys \
  --zone 1 \
  --size Standard_D2s_v5 \
  --storage-sku Premium_LRS \
  --boot-diagnostics-storage ""

az monitor log-analytics workspace create --resource-group $RG --workspace-name $LAW --location $LOCATION
LAW_ID=$(az monitor log-analytics workspace show -g $RG -n $LAW --query id -o tsv)
VM_ID=$(az vm show -g $RG -n $VM --query id -o tsv)

az monitor diagnostic-settings create \
  --name vm-diag \
  --resource $VM_ID \
  --workspace $LAW_ID \
  --logs '[{"categoryGroup":"allLogs","enabled":true}]' \
  --metrics '[{"category":"AllMetrics","enabled":true}]'

az backup vault create --resource-group $RG --name $VAULT --location $LOCATION
az backup vault backup-properties set --name $VAULT --resource-group $RG --backup-storage-redundancy GeoRedundant
az backup policy get-default-for-vm --resource-group $RG --vault-name $VAULT
az backup protection enable-for-vm --resource-group $RG --vault-name $VAULT --vm $VM --policy-name DefaultPolicy

# Comparison aid: create an availability set for review only
az vm availability-set create --resource-group $RG --name avset-vm-ha --platform-fault-domain-count 2 --platform-update-domain-count 5 --sku aligned
```

#### PowerShell
```powershell
$RG = "rg-az305-vm-ha"
$Location = "eastus2"
$Vnet = "vnet-vm-ha"
$Subnet = "snet-app"
$Nsg = "nsg-vm-ha"
$Pip = "pip-vm-ha"
$Nic = "nic-vm-ha"
$VmName = "vmha$(Get-Random)"
$AdminUser = "azureuser"
$Vault = "rsvmha$(Get-Random)"
$Law = "lawvmha$(Get-Random)"
$Cred = Get-Credential -UserName $AdminUser -Message "Enter local admin or SSH-style credential for the VM"

New-AzResourceGroup -Name $RG -Location $Location
$subnetConfig = New-AzVirtualNetworkSubnetConfig -Name $Subnet -AddressPrefix "10.10.1.0/24"
$vnet = New-AzVirtualNetwork -ResourceGroupName $RG -Name $Vnet -Location $Location -AddressPrefix "10.10.0.0/16" -Subnet $subnetConfig
$nsgRule = New-AzNetworkSecurityRuleConfig -Name "allow-ssh" -Protocol Tcp -Direction Inbound -Priority 1000 -SourceAddressPrefix * -SourcePortRange * -DestinationAddressPrefix * -DestinationPortRange 22 -Access Allow
$nsgObj = New-AzNetworkSecurityGroup -ResourceGroupName $RG -Location $Location -Name $Nsg -SecurityRules $nsgRule
$pipObj = New-AzPublicIpAddress -ResourceGroupName $RG -Name $Pip -Location $Location -Sku Standard -AllocationMethod Static -Zone 1
$nicObj = New-AzNetworkInterface -ResourceGroupName $RG -Name $Nic -Location $Location -SubnetId $vnet.Subnets[0].Id -NetworkSecurityGroupId $nsgObj.Id -PublicIpAddressId $pipObj.Id

$vmConfig = New-AzVMConfig -VMName $VmName -VMSize "Standard_D2s_v5" -Zone "1"
$vmConfig = Set-AzVMOperatingSystem -VM $vmConfig -Linux -ComputerName $VmName -Credential $Cred -DisablePasswordAuthentication:$false
$vmConfig = Set-AzVMSourceImage -VM $vmConfig -PublisherName Canonical -Offer 0001-com-ubuntu-server-jammy -Skus 22_04-lts-gen2 -Version latest
$vmConfig = Set-AzVMOSDisk -VM $vmConfig -CreateOption FromImage -StorageAccountType Premium_LRS
$vmConfig = Add-AzVMNetworkInterface -VM $vmConfig -Id $nicObj.Id
New-AzVM -ResourceGroupName $RG -Location $Location -VM $vmConfig

$workspace = New-AzOperationalInsightsWorkspace -ResourceGroupName $RG -Name $Law -Location $Location -Sku PerGB2018
$vm = Get-AzVM -ResourceGroupName $RG -Name $VmName
Set-AzVMBootDiagnostic -ResourceGroupName $RG -VMName $VmName -Enable
Set-AzDiagnosticSetting -Name "vm-diag" -ResourceId $vm.Id -WorkspaceId $workspace.ResourceId -Enabled $true

$vaultObj = New-AzRecoveryServicesVault -Name $Vault -ResourceGroupName $RG -Location $Location
Set-AzRecoveryServicesVaultContext -Vault $vaultObj
$policy = Get-AzRecoveryServicesBackupProtectionPolicy -VaultId $vaultObj.ID -Name "DefaultPolicy"
Enable-AzRecoveryServicesBackupProtection -Policy $policy -Name $VmName -ResourceGroupName $RG

# Comparison aid
New-AzAvailabilitySet -ResourceGroupName $RG -Location $Location -Name "avset-vm-ha" -Sku aligned -PlatformFaultDomainCount 2 -PlatformUpdateDomainCount 5
```

### Verification Steps
```bash
az vm show -g $RG -n $VM --query "{name:name,zone:zones[0],size:hardwareProfile.vmSize,osDisk:storageProfile.osDisk.managedDisk.storageAccountType}" -o table
az backup item list --resource-group $RG --vault-name $VAULT --backup-management-type AzureIaasVM -o table
az monitor diagnostic-settings list --resource $VM_ID -o table
```

```powershell
Get-AzVM -ResourceGroupName $RG -Name $VmName | Select-Object Name, Zones, HardwareProfile
Get-AzRecoveryServicesBackupItem -BackupManagementType AzureVM -WorkloadType AzureVM -VaultId $vaultObj.ID
Get-AzDiagnosticSetting -ResourceId $vm.Id
```

### Cleanup
```bash
az group delete --name $RG --yes --no-wait
```

```powershell
Remove-AzResourceGroup -Name $RG -Force -AsJob
```

### Exam Tip
If the requirement says **protect against datacenter failure in a region**, choose **Availability Zones**. If the requirement is only host/update domain separation and zones are unavailable, **Availability Sets** are acceptable.

### Exam-Style Review Questions
1. A workload requires HA inside one Azure region and supports zonal deployment. Which option is the best fit: Availability Set or Availability Zone?
2. When would Premium SSD be preferred over Standard SSD for an application server?
3. Why is Azure Backup a better exam answer than ad hoc snapshots for operational recovery?

---

## Lab 2: Virtual Machine Scale Sets (VMSS)

### Objective
Deploy a Virtual Machine Scale Set using Flexible orchestration, CPU-based autoscale, health probes, automatic repairs, and rolling upgrades.

### When to Use This
Use VMSS when you need a fleet of similar VMs with elastic scale, controlled image rollout, and zonal or regional resiliency for stateless application tiers.

### Key AZ-305 Concepts
- Uniform vs. Flexible orchestration
- Autoscale rules and instance count boundaries
- Health probes and automatic instance repairs
- Rolling upgrade policy for controlled patching
- Stateless design assumptions for elastic compute

### Exam Domain Mapping
- **Primary:** Design infrastructure solutions (30-35%)
- **Secondary:** Design identity, governance, and monitoring solutions (25-30%)

### Prerequisites
- Contributor on the subscription/resource group
- Azure CLI and Azure PowerShell authenticated
- Region supporting Flexible orchestration
- Understanding that app state must be externalized to a database/cache/storage tier

### Architecture and Design Rationale
Flexible orchestration is preferred for many modern scenarios because it supports a more VM-like management model, mixed SKUs in some designs, and better alignment with zonal resilience. The exam tests whether you recognize that **VMSS is ideal for identical or near-identical stateless nodes**, not stateful legacy servers.

### Implementation Steps
1. Create network resources and a Standard Load Balancer.
2. Deploy a zonal Flexible VMSS.
3. Add a health probe and automatic repairs.
4. Configure autoscale for CPU > 70% scale-out and CPU < 30% scale-in.
5. Configure rolling upgrades and simulate demand.

### Full CLI + PowerShell Commands

#### Azure CLI
```bash
RG="rg-az305-vmss"
LOCATION="eastus2"
VNET="vnet-vmss"
SUBNET="snet-web"
LB="lb-vmss"
PIP="pip-vmss"
FE="fe-vmss"
BE="be-vmss"
PROBE="hp-http"
RULE="lbrule-http"
VMSS="vmssflex$RANDOM"
AUTOSCALE="autoscale-vmss"

az group create --name $RG --location $LOCATION
az network vnet create --resource-group $RG --name $VNET --location $LOCATION --address-prefix 10.20.0.0/16 --subnet-name $SUBNET --subnet-prefix 10.20.1.0/24
az network public-ip create --resource-group $RG --name $PIP --location $LOCATION --sku Standard
az network lb create --resource-group $RG --name $LB --location $LOCATION --sku Standard --public-ip-address $PIP --frontend-ip-name $FE --backend-pool-name $BE
az network lb probe create --resource-group $RG --lb-name $LB --name $PROBE --protocol Tcp --port 80
az network lb rule create --resource-group $RG --lb-name $LB --name $RULE --protocol Tcp --frontend-port 80 --backend-port 80 --frontend-ip-name $FE --backend-pool-name $BE --probe-name $PROBE

az vmss create \
  --resource-group $RG \
  --name $VMSS \
  --location $LOCATION \
  --orchestration-mode Flexible \
  --image Ubuntu2204 \
  --vm-sku Standard_D2s_v5 \
  --instance-count 2 \
  --zones 1 2 3 \
  --vnet-name $VNET \
  --subnet $SUBNET \
  --load-balancer $LB \
  --backend-pool-name $BE \
  --upgrade-policy-mode Rolling \
  --admin-username azureuser \
  --generate-ssh-keys

VMSS_ID=$(az vmss show -g $RG -n $VMSS --query id -o tsv)
az vmss update --resource-group $RG --name $VMSS --set automaticRepairsPolicy.enabled=true automaticRepairsPolicy.gracePeriod=PT10M upgradePolicy.mode=Rolling upgradePolicy.rollingUpgradePolicy.maxBatchInstancePercent=20 upgradePolicy.rollingUpgradePolicy.maxUnhealthyInstancePercent=20

az monitor autoscale create --resource-group $RG --name $AUTOSCALE --resource $VMSS_ID --min-count 2 --max-count 6 --count 2
az monitor autoscale rule create --resource-group $RG --autoscale-name $AUTOSCALE --condition "Percentage CPU > 70 avg 5m" --scale out 1
az monitor autoscale rule create --resource-group $RG --autoscale-name $AUTOSCALE --condition "Percentage CPU < 30 avg 10m" --scale in 1
```

#### PowerShell
```powershell
$RG = "rg-az305-vmss"
$Location = "eastus2"
$Vnet = "vnet-vmss"
$Subnet = "snet-web"
$Pip = "pip-vmss"
$Lb = "lb-vmss"
$Vmss = "vmssflex$(Get-Random)"
$AutoscaleName = "autoscale-vmss"
$Cred = Get-Credential -UserName "azureuser" -Message "Enter the local admin credential for VMSS instances"

New-AzResourceGroup -Name $RG -Location $Location
$subnetConfig = New-AzVirtualNetworkSubnetConfig -Name $Subnet -AddressPrefix "10.20.1.0/24"
$vnet = New-AzVirtualNetwork -ResourceGroupName $RG -Name $Vnet -Location $Location -AddressPrefix "10.20.0.0/16" -Subnet $subnetConfig
$pipObj = New-AzPublicIpAddress -ResourceGroupName $RG -Name $Pip -Location $Location -Sku Standard -AllocationMethod Static
$fe = New-AzLoadBalancerFrontendIpConfig -Name "fe-vmss" -PublicIpAddress $pipObj
$be = New-AzLoadBalancerBackendAddressPoolConfig -Name "be-vmss"
$probe = New-AzLoadBalancerProbeConfig -Name "hp-http" -Protocol Tcp -Port 80 -IntervalInSeconds 15 -ProbeCount 2
$rule = New-AzLoadBalancerRuleConfig -Name "lbrule-http" -Protocol Tcp -FrontendPort 80 -BackendPort 80 -FrontendIpConfiguration $fe -BackendAddressPool $be -Probe $probe
$lbObj = New-AzLoadBalancer -ResourceGroupName $RG -Name $Lb -Location $Location -Sku Standard -FrontendIpConfiguration $fe -BackendAddressPool $be -Probe $probe -LoadBalancingRule $rule

$ipConfig = New-AzVmssIpConfig -Name "ipconfig-vmss" -SubnetId $vnet.Subnets[0].Id -LoadBalancerBackendAddressPoolsId $lbObj.BackendAddressPools[0].Id
$vmProfile = New-AzVmssConfig -Location $Location -SkuCapacity 2 -SkuName "Standard_D2s_v5" -OrchestrationMode Flexible -UpgradePolicyMode Rolling
Set-AzVmssStorageProfile $vmProfile -ImageReferencePublisher Canonical -ImageReferenceOffer 0001-com-ubuntu-server-jammy -ImageReferenceSku 22_04-lts-gen2 -ImageReferenceVersion latest -OsDiskCreateOption FromImage -ManagedDisk Premium_LRS
Set-AzVmssOsProfile $vmProfile -ComputerNamePrefix "vmssflex" -AdminUsername $Cred.UserName -AdminPassword ($Cred.GetNetworkCredential().Password)
Add-AzVmssNetworkInterfaceConfiguration -VirtualMachineScaleSet $vmProfile -Name "nicconfig" -Primary $true -IPConfiguration $ipConfig
$vmssObj = New-AzVmss -ResourceGroupName $RG -Name $Vmss -Location $Location -VirtualMachineScaleSet $vmProfile -Zone "1","2","3"

Update-AzVmss -ResourceGroupName $RG -Name $Vmss -AutomaticRepairsPolicyEnabled $true -AutomaticRepairsPolicyGracePeriod "PT10M"
$target = "/subscriptions/$((Get-AzContext).Subscription.Id)/resourceGroups/$RG/providers/Microsoft.Compute/virtualMachineScaleSets/$Vmss"
$autoscale = New-AzAutoscaleSetting -ResourceGroup $RG -Name $AutoscaleName -TargetResourceId $target -MinCount 2 -MaxCount 6 -DefaultCount 2
Add-AzAutoscaleRule -AutoscaleSetting $autoscale -MetricName "Percentage CPU" -MetricResourceId $target -Operator GreaterThan -MetricStatistic Average -Threshold 70 -TimeGrain 00:01:00 -TimeWindow 00:05:00 -ScaleActionDirection Increase -ScaleActionScaleType ChangeCount -ScaleActionValue 1 -ScaleActionCooldown 00:05:00
Add-AzAutoscaleRule -AutoscaleSetting $autoscale -MetricName "Percentage CPU" -MetricResourceId $target -Operator LessThan -MetricStatistic Average -Threshold 30 -TimeGrain 00:01:00 -TimeWindow 00:10:00 -ScaleActionDirection Decrease -ScaleActionScaleType ChangeCount -ScaleActionValue 1 -ScaleActionCooldown 00:10:00
$autoscale | Update-AzAutoscaleSetting
```

### Verification Steps
```bash
az vmss show -g $RG -n $VMSS --query "{orchestration:orchestrationMode,sku:sku.name,capacity:sku.capacity,zones:zones}" -o json
az vmss list-instances -g $RG -n $VMSS -o table
az monitor autoscale show --resource-group $RG --name $AUTOSCALE
```

```powershell
Get-AzVmss -ResourceGroupName $RG -VMScaleSetName $Vmss | Select-Object Name, OrchestrationMode, Sku
Get-AzVmssVM -ResourceGroupName $RG -VMScaleSetName $Vmss
Get-AzAutoscaleSetting -ResourceGroupName $RG -Name $AutoscaleName
```

### Cleanup
```bash
az group delete --name $RG --yes --no-wait
```

```powershell
Remove-AzResourceGroup -Name $RG -Force -AsJob
```

### Exam Tip
If the scenario says **stateless tier**, **scale by metric**, and **standard VM image across many instances**, think **VMSS** before individual VMs.

### Exam-Style Review Questions
1. Why is VMSS a poor fit for a workload with strong server affinity and local state?
2. When is Flexible orchestration preferred over Uniform orchestration?
3. Why are health probes important for autoscale and automatic repairs?

---

## Lab 3: Azure Kubernetes Service (AKS) Cluster

### Objective
Deploy an AKS cluster using Azure CNI, add a user node pool with autoscaling, deploy a sample app, configure ingress, enable Container Insights, and attach Azure Container Registry.

### When to Use This
Use AKS when you need Kubernetes APIs, advanced orchestration, multi-container microservices, GitOps/platform engineering patterns, or portability for complex containerized apps.

### Key AZ-305 Concepts
- Azure CNI vs. kubenet networking tradeoffs
- System vs. user node pools
- Cluster autoscaler and ingress architecture
- ACR integration and image pull permissions
- Container Insights for cluster observability

### Exam Domain Mapping
- **Primary:** Design infrastructure solutions (30-35%)
- **Secondary:** Design identity, governance, and monitoring solutions (25-30%)

### Prerequisites
- Azure CLI with `aks-preview` extension updated if required
- `kubectl` installed locally
- Azure PowerShell authenticated
- Permissions to create managed identity, AKS, Log Analytics, and ACR resources

### Architecture and Design Rationale
AKS is the right answer when the workload needs the Kubernetes control plane: multiple services, ingress routing, pod-level autoscaling, or service mesh readiness. Azure CNI is preferred when you need direct VNet IP assignment and tighter network policy alignment, but you must size the subnet carefully to avoid IP exhaustion.

### Implementation Steps
1. Create a VNet and subnet sized for node and pod growth.
2. Create Log Analytics and ACR.
3. Deploy AKS using Azure CNI and managed identity.
4. Add a user node pool with autoscaling enabled.
5. Enable Container Insights and attach ACR.
6. Deploy a sample application and ingress controller.

### Full CLI + PowerShell Commands

#### Azure CLI
```bash
RG="rg-az305-aks"
LOCATION="eastus"
VNET="vnet-aks"
SUBNET="snet-aks"
AKS="aksaz305$RANDOM"
ACR="acr$RANDOM$RANDOM"
LAW="lawaks$RANDOM"
INGRESS_NS="ingress-nginx"

az group create --name $RG --location $LOCATION
az network vnet create --resource-group $RG --name $VNET --location $LOCATION --address-prefix 10.30.0.0/16 --subnet-name $SUBNET --subnet-prefix 10.30.0.0/22
SUBNET_ID=$(az network vnet subnet show -g $RG --vnet-name $VNET -n $SUBNET --query id -o tsv)

az monitor log-analytics workspace create --resource-group $RG --workspace-name $LAW --location $LOCATION
LAW_ID=$(az monitor log-analytics workspace show -g $RG -n $LAW --query id -o tsv)

az acr create --resource-group $RG --name $ACR --sku Standard --admin-enabled true
az aks create \
  --resource-group $RG \
  --name $AKS \
  --location $LOCATION \
  --network-plugin azure \
  --vnet-subnet-id $SUBNET_ID \
  --node-count 2 \
  --node-vm-size Standard_D4s_v5 \
  --enable-managed-identity \
  --enable-addons monitoring \
  --workspace-resource-id $LAW_ID \
  --generate-ssh-keys \
  --attach-acr $ACR

az aks nodepool add \
  --resource-group $RG \
  --cluster-name $AKS \
  --name usernp \
  --mode User \
  --node-count 1 \
  --enable-cluster-autoscaler \
  --min-count 1 \
  --max-count 4 \
  --node-vm-size Standard_D4s_v5

az aks get-credentials --resource-group $RG --name $AKS --overwrite-existing
kubectl create namespace sample
kubectl create deployment hello-aks --image=mcr.microsoft.com/azuredocs/aks-helloworld:v1 -n sample
kubectl expose deployment hello-aks --port 80 --target-port 80 --type ClusterIP -n sample
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/cloud/deploy.yaml
kubectl wait --namespace ingress-nginx --for=condition=ready pod --selector=app.kubernetes.io/component=controller --timeout=600s
cat <<'EOF' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: hello-aks-ingress
  namespace: sample
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: hello-aks
            port:
              number: 80
EOF
```

#### PowerShell
```powershell
$RG = "rg-az305-aks"
$Location = "eastus"
$Vnet = "vnet-aks"
$Subnet = "snet-aks"
$Aks = "aksaz305$(Get-Random)"
$Acr = ("acr{0}{1}" -f (Get-Random), (Get-Random)).ToLower().Substring(0,18)
$Law = "lawaks$(Get-Random)"

New-AzResourceGroup -Name $RG -Location $Location
$subnetConfig = New-AzVirtualNetworkSubnetConfig -Name $Subnet -AddressPrefix "10.30.0.0/22"
$vnet = New-AzVirtualNetwork -ResourceGroupName $RG -Name $Vnet -Location $Location -AddressPrefix "10.30.0.0/16" -Subnet $subnetConfig
$workspace = New-AzOperationalInsightsWorkspace -ResourceGroupName $RG -Name $Law -Location $Location -Sku PerGB2018
New-AzContainerRegistry -ResourceGroupName $RG -Name $Acr -Location $Location -Sku Standard -EnableAdminUser
$subnetId = (Get-AzVirtualNetworkSubnetConfig -Name $Subnet -VirtualNetwork $vnet).Id

New-AzAksCluster -ResourceGroupName $RG -Name $Aks -Location $Location -NodeCount 2 -NodeVmSize "Standard_D4s_v5" -NetworkPlugin Azure -VnetSubnetId $subnetId -EnableManagedServiceIdentity -EnableMonitoring -LogAnalyticsWorkspaceResourceId $workspace.ResourceId -GenerateSshKey
Import-AzAksCredential -ResourceGroupName $RG -Name $Aks -Force
Update-AzAksCluster -ResourceGroupName $RG -Name $Aks -AttachACR $Acr
New-AzAksNodePool -ClusterName $Aks -ResourceGroupName $RG -Name "usernp" -Mode User -Count 1 -VMSize "Standard_D4s_v5" -EnableAutoScaling -MinCount 1 -MaxCount 4

kubectl create namespace sample
kubectl create deployment hello-aks --image=mcr.microsoft.com/azuredocs/aks-helloworld:v1 -n sample
kubectl expose deployment hello-aks --port 80 --target-port 80 --type ClusterIP -n sample
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/cloud/deploy.yaml
@"
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: hello-aks-ingress
  namespace: sample
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: hello-aks
            port:
              number: 80
"@ | kubectl apply -f -
```

### Verification Steps
```bash
az aks show -g $RG -n $AKS --query "{name:name,networkPlugin:networkProfile.networkPlugin,kubernetesVersion:kubernetesVersion}" -o json
kubectl get nodes -o wide
kubectl get ingress -n sample
az acr show -g $RG -n $ACR --query loginServer -o tsv
```

```powershell
Get-AzAksCluster -ResourceGroupName $RG -Name $Aks | Select-Object Name, KubernetesVersion, ProvisioningState
kubectl get pods -A
kubectl get ingress -n sample
Get-AzContainerRegistry -ResourceGroupName $RG -Name $Acr
```

### Cleanup
```bash
az group delete --name $RG --yes --no-wait
```

```powershell
Remove-AzResourceGroup -Name $RG -Force -AsJob
```

### Exam Tip
Choose **AKS** only when you need Kubernetes-level control. If the requirement is just to run containers with less operational overhead, Container Apps is often the better answer.

### Exam-Style Review Questions
1. Why is Azure CNI often selected for enterprise AKS networking?
2. Why should system and user workloads be separated into different node pools?
3. When is AKS a better fit than App Service for a web-facing application?

---

## Lab 4: Azure App Service Deployment

### Objective
Deploy a web app to Azure App Service, configure a Standard App Service Plan, deployment slots, autoscale, VNet integration, and a safe slot swap.

### When to Use This
Use App Service when you need managed web hosting for web apps or APIs without managing servers or Kubernetes, while still supporting deployment slots, autoscale, and private network access.

### Key AZ-305 Concepts
- App Service Plan sizing and tier selection
- Deployment slots for near-zero-downtime releases
- Scale-up vs. scale-out decisions
- Regional VNet integration for outbound private access
- PaaS tradeoff: lower management overhead, less OS control

### Exam Domain Mapping
- **Primary:** Design infrastructure solutions (30-35%)
- **Secondary:** Design identity, governance, and monitoring solutions (25-30%)

### Prerequisites
- Contributor permissions
- Azure CLI and Azure PowerShell authenticated
- Region supporting the selected SKU
- Sample app code or willingness to use a built-in runtime sample

### Architecture and Design Rationale
App Service is a strong AZ-305 answer for standard web apps and APIs because it reduces patching, scaling, and platform maintenance. Deployment slots are a key exam feature because they reduce release risk, especially for apps that need staged validation before production cutover.

### Implementation Steps
1. Create a Standard App Service Plan.
2. Deploy a web app and staging slot.
3. Configure autoscale rules.
4. Create a delegated subnet and enable VNet integration.
5. Deploy content to staging and swap into production.

### Full CLI + PowerShell Commands

#### Azure CLI
```bash
RG="rg-az305-appsvc"
LOCATION="eastus"
PLAN="plan-appsvc-$RANDOM"
WEBAPP="webapp$RANDOM$RANDOM"
VNET="vnet-appsvc"
SUBNET="snet-appsvc-int"
AUTOSCALE="autoscale-appsvc"
ZIP_PATH="./sample-app.zip"

az group create --name $RG --location $LOCATION
az network vnet create --resource-group $RG --name $VNET --location $LOCATION --address-prefix 10.40.0.0/16 --subnet-name $SUBNET --subnet-prefix 10.40.1.0/24
az network vnet subnet update --resource-group $RG --vnet-name $VNET --name $SUBNET --delegations Microsoft.Web/serverFarms
az appservice plan create --resource-group $RG --name $PLAN --location $LOCATION --sku S1 --is-linux
az webapp create --resource-group $RG --plan $PLAN --name $WEBAPP --runtime "NODE|18-lts"
az webapp deployment slot create --resource-group $RG --name $WEBAPP --slot staging
az webapp config appsettings set --resource-group $RG --name $WEBAPP --slot staging --settings WEBSITE_RUN_FROM_PACKAGE=1 SLOT_NAME=staging
az webapp vnet-integration add --resource-group $RG --name $WEBAPP --vnet $VNET --subnet $SUBNET

PLAN_ID=$(az appservice plan show -g $RG -n $PLAN --query id -o tsv)
az monitor autoscale create --resource-group $RG --name $AUTOSCALE --resource $PLAN_ID --min-count 1 --max-count 4 --count 1
az monitor autoscale rule create --resource-group $RG --autoscale-name $AUTOSCALE --condition "CpuPercentage > 70 avg 10m" --scale out 1
az monitor autoscale rule create --resource-group $RG --autoscale-name $AUTOSCALE --condition "CpuPercentage < 30 avg 20m" --scale in 1

az webapp deploy --resource-group $RG --name $WEBAPP --slot staging --src-path $ZIP_PATH
az webapp deployment slot swap --resource-group $RG --name $WEBAPP --slot staging --target-slot production
```

#### PowerShell
```powershell
$RG = "rg-az305-appsvc"
$Location = "eastus"
$Plan = "plan-appsvc-$(Get-Random)"
$WebApp = ("webapp{0}{1}" -f (Get-Random), (Get-Random)).ToLower().Substring(0,18)
$Vnet = "vnet-appsvc"
$Subnet = "snet-appsvc-int"
$AutoscaleName = "autoscale-appsvc"
$ZipPath = ".\sample-app.zip"

New-AzResourceGroup -Name $RG -Location $Location
$subnetConfig = New-AzVirtualNetworkSubnetConfig -Name $Subnet -AddressPrefix "10.40.1.0/24" -Delegation (New-AzDelegation -Name "webfarmdelegation" -ServiceName "Microsoft.Web/serverFarms")
$vnet = New-AzVirtualNetwork -ResourceGroupName $RG -Name $Vnet -Location $Location -AddressPrefix "10.40.0.0/16" -Subnet $subnetConfig
New-AzAppServicePlan -ResourceGroupName $RG -Name $Plan -Location $Location -Tier Standard -NumberofWorkers 1 -WorkerSize Small -Linux
New-AzWebApp -ResourceGroupName $RG -Name $WebApp -Location $Location -AppServicePlan $Plan -RuntimeStack "NODE|18-lts"
New-AzWebAppSlot -ResourceGroupName $RG -Name $WebApp -Slot staging
Set-AzWebAppSlotConfigName -ResourceGroupName $RG -Name $WebApp -AppSettingNames "SLOT_NAME"
Set-AzWebApp -ResourceGroupName $RG -Name $WebApp -AppSettings @{ SLOT_NAME = "production" }
Set-AzWebApp -ResourceGroupName $RG -Name $WebApp -AppSettings @{ WEBSITE_RUN_FROM_PACKAGE = "1" }
Set-AzWebAppSlot -ResourceGroupName $RG -Name $WebApp -Slot staging -AppSettings @{ SLOT_NAME = "staging" }

$subnetResourceId = (Get-AzVirtualNetworkSubnetConfig -VirtualNetwork $vnet -Name $Subnet).Id
Set-AzResource -ResourceType Microsoft.Web/sites/config -ResourceGroupName $RG -ResourceName "$WebApp/virtualNetwork" -ApiVersion 2022-09-01 -PropertyObject @{ subnetResourceId = $subnetResourceId; swiftSupported = $true } -Force

$planResourceId = (Get-AzAppServicePlan -ResourceGroupName $RG -Name $Plan).Id
$autoscale = New-AzAutoscaleSetting -ResourceGroup $RG -Name $AutoscaleName -TargetResourceId $planResourceId -MinCount 1 -MaxCount 4 -DefaultCount 1
Add-AzAutoscaleRule -AutoscaleSetting $autoscale -MetricName "CpuPercentage" -MetricResourceId $planResourceId -Operator GreaterThan -MetricStatistic Average -Threshold 70 -TimeGrain 00:01:00 -TimeWindow 00:10:00 -ScaleActionDirection Increase -ScaleActionScaleType ChangeCount -ScaleActionValue 1 -ScaleActionCooldown 00:05:00
Add-AzAutoscaleRule -AutoscaleSetting $autoscale -MetricName "CpuPercentage" -MetricResourceId $planResourceId -Operator LessThan -MetricStatistic Average -Threshold 30 -TimeGrain 00:01:00 -TimeWindow 00:20:00 -ScaleActionDirection Decrease -ScaleActionScaleType ChangeCount -ScaleActionValue 1 -ScaleActionCooldown 00:10:00
$autoscale | Update-AzAutoscaleSetting

Publish-AzWebApp -ResourceGroupName $RG -Name $WebApp -ArchivePath $ZipPath -Slot staging
Switch-AzWebAppSlot -ResourceGroupName $RG -Name $WebApp -SourceSlotName staging -DestinationSlotName production
```

### Verification Steps
```bash
az webapp show -g $RG -n $WEBAPP --query "{name:name,state:state,defaultHostName:defaultHostName,plan:serverFarmId}" -o json
az webapp deployment slot list --resource-group $RG --name $WEBAPP -o table
az webapp vnet-integration list --resource-group $RG --name $WEBAPP -o table
```

```powershell
Get-AzWebApp -ResourceGroupName $RG -Name $WebApp
Get-AzWebAppSlot -ResourceGroupName $RG -Name $WebApp
(Get-AzResource -ResourceGroupName $RG -ResourceType Microsoft.Web/sites/config -ResourceName "$WebApp/virtualNetwork").Properties
```

### Cleanup
```bash
az group delete --name $RG --yes --no-wait
```

```powershell
Remove-AzResourceGroup -Name $RG -Force -AsJob
```

### Exam Tip
If the requirement is **web app/API**, **minimal ops**, **deployment slots**, and **autoscale**, App Service is often the best-fit answer over AKS or VMs.

### Exam-Style Review Questions
1. Why are deployment slots useful for production releases?
2. Why might App Service be chosen over AKS for a standard enterprise API?
3. What is the purpose of VNet integration for App Service?

---

## Lab 5: Azure Container Apps

### Objective
Create an Azure Container Apps environment, deploy a container from ACR, configure HTTP scaling and a queue-based KEDA scaler, add a custom domain, and enable the Dapr sidecar.

### When to Use This
Use Container Apps when you want containerized applications with serverless operations, KEDA-based scaling, revisions, and optional Dapr features without managing Kubernetes directly.

### Key AZ-305 Concepts
- Serverless containers and scale-to-zero
- Managed environment boundary and ingress settings
- KEDA HTTP and queue/event scalers
- Dapr sidecar for service invocation/pub-sub patterns
- Custom domain and certificate binding

### Exam Domain Mapping
- **Primary:** Design infrastructure solutions (30-35%)
- **Secondary:** Design identity, governance, and monitoring solutions (25-30%), Design data storage solutions (20-25%)

### Prerequisites
- Contributor permissions
- Azure CLI authenticated
- `containerapp` CLI extension available
- Azure PowerShell authenticated
- A public DNS zone you can update for custom domain validation

### Architecture and Design Rationale
Container Apps is the sweet spot between App Service and AKS when the application is already containerized and needs revision-based deployment, event-driven scale, or microservice patterns, but not full Kubernetes control. The exam often tests whether you can avoid AKS over-engineering.

### Implementation Steps
1. Create a Log Analytics workspace and Container Apps environment.
2. Create ACR and push or import a sample image.
3. Deploy a public HTTP container app.
4. Add a queue-based scaler and enable Dapr.
5. Bind a custom domain after DNS validation.

### Full CLI + PowerShell Commands

#### Azure CLI
```bash
RG="rg-az305-ca"
LOCATION="eastus"
ENV="cae-az305"
APP="ca-web-$RANDOM"
LAW="lawca$RANDOM"
ACR="acr$RANDOM$RANDOM"
STORAGE="stca$RANDOM"
QUEUE="orders"
CUSTOM_DOMAIN="api.contoso.example"

az extension add --name containerapp --upgrade
az group create --name $RG --location $LOCATION
az monitor log-analytics workspace create --resource-group $RG --workspace-name $LAW --location $LOCATION
LAW_ID=$(az monitor log-analytics workspace show -g $RG -n $LAW --query customerId -o tsv)
LAW_KEY=$(az monitor log-analytics workspace get-shared-keys -g $RG -n $LAW --query primarySharedKey -o tsv)
az containerapp env create --name $ENV --resource-group $RG --location $LOCATION --logs-workspace-id $LAW_ID --logs-workspace-key $LAW_KEY

az acr create --resource-group $RG --name $ACR --sku Basic --admin-enabled true
az acr import --name $ACR --source mcr.microsoft.com/azuredocs/containerapps-helloworld:latest --image apps/helloworld:latest
ACR_SERVER=$(az acr show -g $RG -n $ACR --query loginServer -o tsv)
ACR_USER=$(az acr credential show -g $RG -n $ACR --query username -o tsv)
ACR_PASS=$(az acr credential show -g $RG -n $ACR --query passwords[0].value -o tsv)

az storage account create --resource-group $RG --name $STORAGE --location $LOCATION --sku Standard_LRS
az storage queue create --account-name $STORAGE --name $QUEUE --auth-mode login
QUEUE_CONN=$(az storage account show-connection-string -g $RG -n $STORAGE --query connectionString -o tsv)

az containerapp create \
  --name $APP \
  --resource-group $RG \
  --environment $ENV \
  --image $ACR_SERVER/apps/helloworld:latest \
  --target-port 80 \
  --ingress external \
  --registry-server $ACR_SERVER \
  --registry-username $ACR_USER \
  --registry-password $ACR_PASS \
  --min-replicas 1 \
  --max-replicas 5 \
  --scale-rule-name http-scale \
  --scale-rule-http-concurrency 50 \
  --env-vars GREETING=az305

az containerapp secret set --name $APP --resource-group $RG --secrets queue-conn="$QUEUE_CONN"
az containerapp update \
  --name $APP \
  --resource-group $RG \
  --enable-dapr true \
  --dapr-app-id $APP \
  --dapr-app-port 80 \
  --scale-rule-name queue-scale \
  --scale-rule-type azure-queue \
  --scale-rule-metadata queueName=$QUEUE accountName=$STORAGE queueLength=5 \
  --scale-rule-auth connection=queue-conn

# Custom domain flow: create DNS TXT/CNAME records first, then bind
az containerapp hostname bind --name $APP --resource-group $RG --hostname $CUSTOM_DOMAIN --validation-method CNAME
```

#### PowerShell
```powershell
$RG = "rg-az305-ca"
$Location = "eastus"
$Env = "cae-az305"
$App = "ca-web-$(Get-Random)"
$Law = "lawca$(Get-Random)"
$Acr = ("acr{0}{1}" -f (Get-Random), (Get-Random)).ToLower().Substring(0,18)
$Storage = ("stca{0}" -f (Get-Random)).ToLower().Substring(0,12)
$Queue = "orders"
$CustomDomain = "api.contoso.example"

New-AzResourceGroup -Name $RG -Location $Location
$workspace = New-AzOperationalInsightsWorkspace -ResourceGroupName $RG -Name $Law -Location $Location -Sku PerGB2018
$keys = Get-AzOperationalInsightsWorkspaceSharedKey -ResourceGroupName $RG -Name $Law
az containerapp env create --name $Env --resource-group $RG --location $Location --logs-workspace-id $workspace.CustomerId --logs-workspace-key $keys.PrimarySharedKey

New-AzContainerRegistry -ResourceGroupName $RG -Name $Acr -Location $Location -Sku Basic -EnableAdminUser
az acr import --name $Acr --source mcr.microsoft.com/azuredocs/containerapps-helloworld:latest --image apps/helloworld:latest
$acrCred = Get-AzContainerRegistryCredential -ResourceGroupName $RG -Name $Acr
$acrServer = (Get-AzContainerRegistry -ResourceGroupName $RG -Name $Acr).LoginServer

New-AzStorageAccount -ResourceGroupName $RG -Name $Storage -Location $Location -SkuName Standard_LRS -Kind StorageV2
$ctx = (Get-AzStorageAccount -ResourceGroupName $RG -Name $Storage).Context
New-AzStorageQueue -Name $Queue -Context $ctx
$queueConn = (Get-AzStorageAccountKey -ResourceGroupName $RG -Name $Storage)[0].Value

az containerapp create --name $App --resource-group $RG --environment $Env --image "$acrServer/apps/helloworld:latest" --target-port 80 --ingress external --registry-server $acrServer --registry-username $acrCred.Username --registry-password $acrCred.Password --min-replicas 1 --max-replicas 5 --scale-rule-name http-scale --scale-rule-http-concurrency 50
az containerapp secret set --name $App --resource-group $RG --secrets queue-conn="DefaultEndpointsProtocol=https;AccountName=$Storage;AccountKey=$queueConn;EndpointSuffix=core.windows.net"
az containerapp update --name $App --resource-group $RG --enable-dapr true --dapr-app-id $App --dapr-app-port 80 --scale-rule-name queue-scale --scale-rule-type azure-queue --scale-rule-metadata queueName=$Queue accountName=$Storage queueLength=5 --scale-rule-auth connection=queue-conn
az containerapp hostname bind --name $App --resource-group $RG --hostname $CustomDomain --validation-method CNAME
```

### Verification Steps
```bash
az containerapp show -g $RG -n $APP --query "{fqdn:properties.configuration.ingress.fqdn,minReplicas:properties.template.scale.minReplicas,maxReplicas:properties.template.scale.maxReplicas,dapr:properties.configuration.dapr.enabled}" -o json
az containerapp revision list -g $RG -n $APP -o table
az storage message put --queue-name $QUEUE --content "order-1" --account-name $STORAGE --connection-string "$QUEUE_CONN"
```

```powershell
az containerapp show -g $RG -n $App
az containerapp revision list -g $RG -n $App -o table
$ctx = (Get-AzStorageAccount -ResourceGroupName $RG -Name $Storage).Context
Add-AzStorageQueueMessage -Queue $Queue -Message "order-1" -Context $ctx
```

### Cleanup
```bash
az group delete --name $RG --yes --no-wait
```

```powershell
Remove-AzResourceGroup -Name $RG -Force -AsJob
```

### Exam Tip
If the scenario says **containerized**, **event-driven scaling**, **HTTP + queue triggers**, and **minimal orchestration management**, Container Apps is often the best answer.

### Exam-Style Review Questions
1. Why is Container Apps usually a better fit than AKS for small event-driven container workloads?
2. What problem does Dapr solve in microservice-style architectures?
3. Why must you validate DNS ownership before binding a custom domain?

---

## Lab 6: Azure Functions (Premium Plan)

### Objective
Deploy an Azure Functions app on Premium plan, publish an HTTP trigger, configure VNet integration, enable Application Insights, compare cold vs. warm behavior, and configure deployment slots.

### When to Use This
Use Azure Functions Premium when the solution is event-driven or API-triggered but still needs VNet integration, predictable warm instances, long execution windows, or reduced cold-start impact.

### Key AZ-305 Concepts
- Consumption vs. Premium vs. Dedicated plans
- Pre-warmed instances and cold-start mitigation
- Function App networking and private dependencies
- Application Insights monitoring and tracing
- Deployment slots for premium apps

### Exam Domain Mapping
- **Primary:** Design infrastructure solutions (30-35%)
- **Secondary:** Design identity, governance, and monitoring solutions (25-30%)

### Prerequisites
- Azure Functions Core Tools installed for local publishing
- Azure CLI and Azure PowerShell authenticated
- A sample HTTP-trigger function project available locally
- Region supporting Elastic Premium plans

### Architecture and Design Rationale
Premium plan is the exam answer when Functions needs **always-ready instances**, **VNet integration**, or **higher performance guarantees**. Consumption is cheaper, but Premium is better for latency-sensitive enterprise integrations.

### Implementation Steps
1. Create storage, Application Insights, and an Elastic Premium plan.
2. Create the Function App and enable VNet integration.
3. Publish an HTTP-trigger project.
4. Add a staging slot.
5. Test warm behavior using repeated requests.

### Full CLI + PowerShell Commands

#### Azure CLI
```bash
RG="rg-az305-func"
LOCATION="eastus"
PLAN="plan-func-premium-$RANDOM"
FUNC="funcprem$RANDOM$RANDOM"
STORAGE="stfunc$RANDOM"
APPINSIGHTS="appi-func-$RANDOM"
VNET="vnet-func"
SUBNET="snet-func"

az group create --name $RG --location $LOCATION
az storage account create --resource-group $RG --name $STORAGE --location $LOCATION --sku Standard_LRS
az monitor app-insights component create --resource-group $RG --app $APPINSIGHTS --location $LOCATION --kind web --application-type web
az network vnet create --resource-group $RG --name $VNET --location $LOCATION --address-prefix 10.50.0.0/16 --subnet-name $SUBNET --subnet-prefix 10.50.1.0/24
az network vnet subnet update --resource-group $RG --vnet-name $VNET --name $SUBNET --delegations Microsoft.Web/serverFarms
az functionapp plan create --resource-group $RG --name $PLAN --location $LOCATION --sku EP1 --is-linux --min-instances 1 --max-burst 5
az functionapp create --resource-group $RG --plan $PLAN --name $FUNC --storage-account $STORAGE --runtime dotnet-isolated --functions-version 4 --os-type Linux --app-insights $APPINSIGHTS
az functionapp vnet-integration add --resource-group $RG --name $FUNC --vnet $VNET --subnet $SUBNET
az functionapp deployment slot create --resource-group $RG --name $FUNC --slot staging
func azure functionapp publish $FUNC

# Cold-start comparison: scale down staging slot to zero always-ready instances, compare first-request latency manually.
az functionapp config appsettings set --resource-group $RG --name $FUNC --settings WEBSITE_SWAP_WARMUP_PING_PATH=/api/HttpTrigger
```

#### PowerShell
```powershell
$RG = "rg-az305-func"
$Location = "eastus"
$Plan = "plan-func-premium-$(Get-Random)"
$Func = ("funcprem{0}{1}" -f (Get-Random), (Get-Random)).ToLower().Substring(0,18)
$Storage = ("stfunc{0}" -f (Get-Random)).ToLower().Substring(0,12)
$AppInsights = "appi-func-$(Get-Random)"
$Vnet = "vnet-func"
$Subnet = "snet-func"

New-AzResourceGroup -Name $RG -Location $Location
New-AzStorageAccount -ResourceGroupName $RG -Name $Storage -Location $Location -SkuName Standard_LRS -Kind StorageV2
New-AzApplicationInsights -ResourceGroupName $RG -Name $AppInsights -Location $Location -Kind web
$subnetConfig = New-AzVirtualNetworkSubnetConfig -Name $Subnet -AddressPrefix "10.50.1.0/24" -Delegation (New-AzDelegation -Name "funcdelegation" -ServiceName "Microsoft.Web/serverFarms")
$vnet = New-AzVirtualNetwork -ResourceGroupName $RG -Name $Vnet -Location $Location -AddressPrefix "10.50.0.0/16" -Subnet $subnetConfig
New-AzFunctionAppPlan -ResourceGroupName $RG -Name $Plan -Location $Location -Sku EP1 -MaximumWorkerCount 5 -MinimumElasticInstanceCount 1 -Linux
New-AzFunctionApp -ResourceGroupName $RG -Name $Func -Location $Location -StorageAccountName $Storage -Runtime DotNetIsolated -FunctionsVersion 4 -OSType Linux -AppServicePlan $Plan -ApplicationInsightsName $AppInsights
Set-AzResource -ResourceGroupName $RG -ResourceType Microsoft.Web/sites/config -ResourceName "$Func/virtualNetwork" -ApiVersion 2022-09-01 -PropertyObject @{ subnetResourceId = (Get-AzVirtualNetworkSubnetConfig -VirtualNetwork $vnet -Name $Subnet).Id; swiftSupported = $true } -Force
New-AzWebAppSlot -ResourceGroupName $RG -Name $Func -Slot staging
func azure functionapp publish $Func
Set-AzWebApp -ResourceGroupName $RG -Name $Func -AppSettings @{ WEBSITE_SWAP_WARMUP_PING_PATH = "/api/HttpTrigger" }
```

### Verification Steps
```bash
az functionapp show -g $RG -n $FUNC --query "{name:name,state:state,host:defaultHostName,plan:serverFarmId}" -o json
az functionapp deployment slot list --resource-group $RG --name $FUNC -o table
az monitor app-insights component show -g $RG -a $APPINSIGHTS --query "{name:name,connectionString:connectionString}" -o json
curl https://$FUNC.azurewebsites.net/api/HttpTrigger?name=AZ305
```

```powershell
Get-AzFunctionApp -ResourceGroupName $RG -Name $Func
Get-AzWebAppSlot -ResourceGroupName $RG -Name $Func
Get-AzApplicationInsights -ResourceGroupName $RG -Name $AppInsights
Invoke-WebRequest -Uri "https://$Func.azurewebsites.net/api/HttpTrigger?name=AZ305"
```

### Cleanup
```bash
az group delete --name $RG --yes --no-wait
```

```powershell
Remove-AzResourceGroup -Name $RG -Force -AsJob
```

### Exam Tip
If the requirement says **event-driven code** plus **private network access** or **predictable startup**, choose **Functions Premium** instead of Consumption.

### Exam-Style Review Questions
1. Why is Premium plan preferred over Consumption for latency-sensitive APIs?
2. What is the role of always-ready instances in Functions Premium?
3. Why would an architect use deployment slots for a Function App?

---

## Lab 7: Azure Container Instances (ACI)

### Objective
Deploy an Azure Container Instance group, define CPU and memory limits, mount an Azure Files share, run a batch-style job, and understand how ACI can back AKS virtual nodes.

### When to Use This
Use ACI for quick container execution, ad hoc jobs, isolated sandbox workloads, or burst scenarios where standing up AKS or VM infrastructure would be excessive.

### Key AZ-305 Concepts
- Fast-start single container or container group execution
- CPU and memory request sizing
- Persistent storage with Azure Files
- Virtual nodes for AKS burst capacity
- Tradeoff: simple runtime, limited orchestration features

### Exam Domain Mapping
- **Primary:** Design infrastructure solutions (30-35%)
- **Secondary:** Design data storage solutions (20-25%)

### Prerequisites
- Contributor permissions
- Azure CLI and Azure PowerShell authenticated
- A basic understanding of container images and command overrides
- Optional AKS cluster if you want to test virtual nodes end-to-end

### Architecture and Design Rationale
ACI is a good fit for short-lived or simple isolated containers, but not for complex microservice orchestration. On the exam, ACI often appears as the right answer for **quick execution** or **burst capacity**, especially when paired with AKS virtual nodes.

### Implementation Steps
1. Create storage for persistent file share mounting.
2. Deploy a public or private ACI container group.
3. Mount Azure Files.
4. Run a short batch command.
5. Review the virtual node pattern for AKS burst.

### Full CLI + PowerShell Commands

#### Azure CLI
```bash
RG="rg-az305-aci"
LOCATION="eastus"
ACI="aci-batch-$RANDOM"
STORAGE="staci$RANDOM"
SHARE="acishare"

az group create --name $RG --location $LOCATION
az storage account create --resource-group $RG --name $STORAGE --location $LOCATION --sku Standard_LRS
STORAGE_KEY=$(az storage account keys list -g $RG -n $STORAGE --query [0].value -o tsv)
az storage share-rm create --resource-group $RG --storage-account $STORAGE --name $SHARE --quota 50

az container create \
  --resource-group $RG \
  --name $ACI \
  --image mcr.microsoft.com/azuredocs/aci-helloworld \
  --cpu 2 \
  --memory 4 \
  --restart-policy OnFailure \
  --azure-file-volume-account-name $STORAGE \
  --azure-file-volume-account-key $STORAGE_KEY \
  --azure-file-volume-share-name $SHARE \
  --azure-file-volume-mount-path /mnt/aci \
  --ports 80 \
  --ip-address Public

# Batch-style run
az container create \
  --resource-group $RG \
  --name aci-job \
  --image mcr.microsoft.com/azuredocs/aci-wordcount:latest \
  --command-line "python wordcount.py https://aka.ms/az305-sample.txt" \
  --cpu 1 \
  --memory 2 \
  --restart-policy Never

# Virtual node concept (requires an existing AKS cluster)
az aks install-cli
az extension add --name virtual-node --upgrade
# az aks enable-addons -g <aks-rg> -n <aks-name> --addons virtual-node
```

#### PowerShell
```powershell
$RG = "rg-az305-aci"
$Location = "eastus"
$Aci = "aci-batch-$(Get-Random)"
$Storage = ("staci{0}" -f (Get-Random)).ToLower().Substring(0,12)
$Share = "acishare"

New-AzResourceGroup -Name $RG -Location $Location
$storageObj = New-AzStorageAccount -ResourceGroupName $RG -Name $Storage -Location $Location -SkuName Standard_LRS -Kind StorageV2
$ctx = $storageObj.Context
New-AzStorageShare -Name $Share -Context $ctx
$key = (Get-AzStorageAccountKey -ResourceGroupName $RG -Name $Storage)[0].Value

New-AzContainerGroup -ResourceGroupName $RG -Name $Aci -Location $Location -Image mcr.microsoft.com/azuredocs/aci-helloworld -OsType Linux -IpAddressType Public -Cpu 2 -MemoryInGB 4 -Port 80 -AzureFileVolumeShareName $Share -AzureFileVolumeStorageAccountName $Storage -AzureFileVolumeStorageAccountKey $key -AzureFileVolumeMountPath "/mnt/aci"
New-AzContainerGroup -ResourceGroupName $RG -Name "aci-job" -Location $Location -Image mcr.microsoft.com/azuredocs/aci-wordcount:latest -OsType Linux -RestartPolicy Never -Command "python wordcount.py https://aka.ms/az305-sample.txt" -Cpu 1 -MemoryInGB 2

# Virtual node concept for an existing AKS cluster
# Enable-AzAksAddon -ResourceGroupName <aks-rg> -Name <aks-name> -AddonName VirtualNode
```

### Verification Steps
```bash
az container show -g $RG -n $ACI --query "{state:instanceView.state,ip:ipAddress.ip,cpu:containers[0].resources.requests.cpu,memory:containers[0].resources.requests.memoryInGB}" -o json
az container logs -g $RG -n aci-job
az storage share-rm show --resource-group $RG --storage-account $STORAGE --name $SHARE
```

```powershell
Get-AzContainerGroup -ResourceGroupName $RG -Name $Aci
Get-AzContainerGroup -ResourceGroupName $RG -Name "aci-job"
Get-AzStorageShare -Context $ctx -Name $Share
```

### Cleanup
```bash
az group delete --name $RG --yes --no-wait
```

```powershell
Remove-AzResourceGroup -Name $RG -Force -AsJob
```

### Exam Tip
If the scenario says **run a container quickly** or **burst a simple container workload without full orchestration**, ACI is often the right answer.

### Exam-Style Review Questions
1. Why is ACI a better fit than AKS for a one-off batch container job?
2. When would Azure Files mounting be required in ACI?
3. Why is ACI not a strong choice for complex service-to-service routing patterns?

---

## Lab 8: VM Cost Optimization

### Objective
Deploy a Spot VM, review eviction behavior, check Azure Advisor recommendations, estimate Reserved Instance savings, and enable Azure Hybrid Benefit.

### When to Use This
Use these patterns when the business asks for lower compute cost without sacrificing critical requirements. The key AZ-305 skill is recognizing which workloads can tolerate interruption, commitment, or licensing reuse.

### Key AZ-305 Concepts
- Spot VM eviction policy and max price strategy
- Reserved Instance savings for predictable workloads
- Azure Hybrid Benefit for eligible Windows/SQL licensing
- Azure Advisor right-sizing guidance
- Matching commercial model to workload predictability

### Exam Domain Mapping
- **Primary:** Design infrastructure solutions (30-35%)
- **Secondary:** Design business continuity solutions (15-20%)

### Prerequisites
- Contributor permissions
- Azure CLI and PowerShell authenticated
- Access to Azure Advisor recommendations in the subscription
- A workload that can tolerate interruption for Spot testing

### Architecture and Design Rationale
Cost optimization in AZ-305 is about tradeoffs. **Spot VMs** fit interruptible jobs, **Reserved Instances** fit steady-state workloads, and **Azure Hybrid Benefit** fits organizations with eligible existing licenses. If the workload is mission-critical and cannot be interrupted, Spot is almost never the right answer.

### Implementation Steps
1. Deploy a Spot VM with capacity-only or price-based eviction.
2. Review Advisor right-sizing recommendations.
3. Estimate RI savings and discuss break-even logic.
4. Enable Azure Hybrid Benefit for Windows licensing reuse.

### Full CLI + PowerShell Commands

#### Azure CLI
```bash
RG="rg-az305-vm-cost"
LOCATION="eastus"
VM="vmspot$RANDOM"

az group create --name $RG --location $LOCATION
az vm create \
  --resource-group $RG \
  --name $VM \
  --image Win2022Datacenter \
  --admin-username azureuser \
  --admin-password "P@ssword1234!ChangeMe" \
  --size Standard_D2s_v5 \
  --priority Spot \
  --eviction-policy Deallocate \
  --max-price -1 \
  --license-type Windows_Server

az advisor recommendation list --category Cost --query "[?contains(resourceMetadata.resourceId, '$VM')]" -o table
az vm update --resource-group $RG --name $VM --set licenseType=Windows_Server

# Reserved Instance pricing research for exam prep
az consumption reservation summary list --grain monthly --filter "properties/usageDate ge 2024-01-01"
```

#### PowerShell
```powershell
$RG = "rg-az305-vm-cost"
$Location = "eastus"
$VmName = "vmspot$(Get-Random)"
$Cred = New-Object System.Management.Automation.PSCredential ("azureuser", (ConvertTo-SecureString "P@ssword1234!ChangeMe" -AsPlainText -Force))

New-AzResourceGroup -Name $RG -Location $Location
New-AzVM -ResourceGroupName $RG -Name $VmName -Location $Location -ImageName Win2022Datacenter -Credential $Cred -Size "Standard_D2s_v5" -Priority Spot -MaxPrice -1 -EvictionPolicy Deallocate -LicenseType Windows_Server
Get-AzAdvisorRecommendation | Where-Object { $_.Category -eq "Cost" -and $_.ResourceMetadata.ResourceId -match $VmName }
Update-AzVM -ResourceGroupName $RG -VM (Get-AzVM -ResourceGroupName $RG -Name $VmName)

# Reserved capacity review for cost planning
Get-AzConsumptionReservationSummary -Grain monthly
```

### Verification Steps
```bash
az vm show -g $RG -n $VM --query "{priority:priority,evictionPolicy:evictionPolicy,licenseType:licenseType}" -o json
az advisor recommendation list --category Cost -o table
```

```powershell
Get-AzVM -ResourceGroupName $RG -Name $VmName | Select-Object Name, Priority, EvictionPolicy, LicenseType
Get-AzAdvisorRecommendation | Where-Object Category -eq "Cost"
```

### Cleanup
```bash
az group delete --name $RG --yes --no-wait
```

```powershell
Remove-AzResourceGroup -Name $RG -Force -AsJob
```

### Exam Tip
- **Spot** = cheapest, but interruptible.  
- **Reserved** = best for predictable long-running workloads.  
- **Hybrid Benefit** = licensing discount when you already own eligible licenses.

### Exam-Style Review Questions
1. Why is a Spot VM a poor choice for a domain controller?
2. When is a Reserved Instance better than pay-as-you-go?
3. What business condition must exist before Azure Hybrid Benefit is valid?

---

## Lab 9: Azure Batch for HPC

### Objective
Create an Azure Batch account, build a pool with an autoscale formula, submit a job and tasks, monitor execution, and review how low-priority nodes reduce cost.

### When to Use This
Use Azure Batch for high-throughput computing, rendering, simulation, financial modeling, or other large-scale parallel processing where task scheduling matters more than interactive hosting.

### Key AZ-305 Concepts
- Batch account and pool separation
- Autoscale formulas based on queued tasks
- Job and task scheduling model
- Low-priority nodes for cost savings
- HPC workload fit vs. AKS/VMSS/Functions alternatives

### Exam Domain Mapping
- **Primary:** Design infrastructure solutions (30-35%)
- **Secondary:** Design cost optimization and resiliency tradeoffs within compute selection

### Prerequisites
- Contributor permissions
- Azure CLI and Azure PowerShell authenticated
- Azure Batch account access
- Basic familiarity with task commands and pool concepts

### Architecture and Design Rationale
Azure Batch is the right answer when you need a scheduler for many parallel jobs across a compute pool. It is not the best fit for user-facing APIs or always-on application hosting. For AZ-305, know that low-priority nodes reduce cost but can be preempted.

### Implementation Steps
1. Create a storage account and Batch account.
2. Create a pool using low-priority nodes.
3. Apply an autoscale formula.
4. Create a job and submit tasks.
5. Monitor job and task completion.

### Full CLI + PowerShell Commands

#### Azure CLI
```bash
RG="rg-az305-batch"
LOCATION="eastus"
BATCH="batchaz305$RANDOM"
STORAGE="stbatch$RANDOM"
POOL="pool-hpc"
JOB="job-hpc"

az group create --name $RG --location $LOCATION
az storage account create --resource-group $RG --name $STORAGE --location $LOCATION --sku Standard_LRS
az batch account create --resource-group $RG --name $BATCH --location $LOCATION --storage-account $STORAGE
az batch account login --resource-group $RG --name $BATCH --shared-key-auth

az batch pool create \
  --id $POOL \
  --vm-size Standard_HB120rs_v3 \
  --target-dedicated-nodes 0 \
  --target-low-priority-nodes 2 \
  --image canonical:0001-com-ubuntu-hpc:20_04-lts-hpc:latest \
  --node-agent-sku-id "batch.node.ubuntu 20.04"

az batch pool autoscale enable --pool-id $POOL --auto-scale-formula '$TargetLowPriorityNodes=max(0, min(10, $PendingTasks.GetSample(1))); $NodeDeallocationOption=taskcompletion;' --evaluation-interval PT5M
az batch job create --id $JOB --pool-id $POOL
az batch task create --job-id $JOB --task-id task1 --command-line "/bin/bash -c 'echo Hello from task1 && sleep 30'"
az batch task create --job-id $JOB --task-id task2 --command-line "/bin/bash -c 'echo Hello from task2 && sleep 30'"
```

#### PowerShell
```powershell
$RG = "rg-az305-batch"
$Location = "eastus"
$Batch = "batchaz305$(Get-Random)"
$Storage = ("stbatch{0}" -f (Get-Random)).ToLower().Substring(0,12)
$Pool = "pool-hpc"
$Job = "job-hpc"

New-AzResourceGroup -Name $RG -Location $Location
New-AzStorageAccount -ResourceGroupName $RG -Name $Storage -Location $Location -SkuName Standard_LRS -Kind StorageV2
New-AzBatchAccount -ResourceGroupName $RG -Name $Batch -Location $Location -AutoStorageAccountId (Get-AzStorageAccount -ResourceGroupName $RG -Name $Storage).Id
$batchAccount = Get-AzBatchAccount -ResourceGroupName $RG -Name $Batch
$keys = Get-AzBatchAccountKeys -AccountName $Batch -ResourceGroupName $RG
$cred = New-Object Microsoft.Azure.Commands.Batch.Models.PSBatchAccountKeys

# Use Azure CLI for data-plane task operations where it is simpler
az batch account login --resource-group $RG --name $Batch --shared-key-auth
az batch pool create --id $Pool --vm-size Standard_HB120rs_v3 --target-dedicated-nodes 0 --target-low-priority-nodes 2 --image canonical:0001-com-ubuntu-hpc:20_04-lts-hpc:latest --node-agent-sku-id "batch.node.ubuntu 20.04"
az batch pool autoscale enable --pool-id $Pool --auto-scale-formula '$TargetLowPriorityNodes=max(0, min(10, $PendingTasks.GetSample(1))); $NodeDeallocationOption=taskcompletion;' --evaluation-interval PT5M
az batch job create --id $Job --pool-id $Pool
az batch task create --job-id $Job --task-id task1 --command-line "/bin/bash -c 'echo Hello from task1 && sleep 30'"
az batch task create --job-id $Job --task-id task2 --command-line "/bin/bash -c 'echo Hello from task2 && sleep 30'"
```

### Verification Steps
```bash
az batch pool show --pool-id $POOL
az batch job show --job-id $JOB
az batch task list --job-id $JOB -o table
```

```powershell
Get-AzBatchAccount -ResourceGroupName $RG -Name $Batch
az batch pool show --pool-id $Pool
az batch task list --job-id $Job -o table
```

### Cleanup
```bash
az batch job delete --job-id $JOB --yes
az batch pool delete --pool-id $POOL --yes
az group delete --name $RG --yes --no-wait
```

```powershell
az batch job delete --job-id $Job --yes
az batch pool delete --pool-id $Pool --yes
Remove-AzResourceGroup -Name $RG -Force -AsJob
```

### Exam Tip
If the scenario describes **thousands of parallel tasks**, **rendering**, **simulation**, or **scheduler-managed compute pools**, think **Azure Batch**.

### Exam-Style Review Questions
1. Why is Azure Batch a better fit than Functions for HPC simulation jobs?
2. What is the tradeoff of low-priority nodes in a Batch pool?
3. Why does autoscale matter for queued task workloads?

---

## Lab 10: Compute Decision Exercise

### Objective
Practice service selection by mapping 10 common AZ-305 scenarios to the best Azure compute option and justifying the decision.

### When to Use This
Use this lab right before the exam or after completing the earlier hands-on labs to test whether you can convert business and technical requirements into the best compute recommendation.

### Key AZ-305 Concepts
- Compute service selection heuristics
- Cost vs. control vs. operations tradeoffs
- High availability, elasticity, and modernization paths
- Matching workload behavior to platform capabilities

### Exam Domain Mapping
- **Primary:** Design infrastructure solutions (30-35%)
- **Secondary:** All other domains, because compute choices affect monitoring, resiliency, security, and cost

### Prerequisites
- Familiarity with Labs 1-9
- Ability to identify whether a workload is stateful, event-driven, web-hosted, containerized, or parallel-processing oriented

### Architecture and Design Rationale
The AZ-305 exam usually rewards the **lowest-management service that still meets the requirement**. The architect must identify the constraint that dominates the scenario: OS control, Kubernetes APIs, serverless scale, steady-state cost, batch scheduling, VNet integration, or deployment safety.

### Scenario Exercise

| # | Scenario | Best Compute Service | Why |
|---|---|---|---|
| 1 | Legacy line-of-business app requires custom drivers and domain join. | Azure Virtual Machines | Full OS control, legacy compatibility, and AD integration. |
| 2 | Stateless web tier must scale from 2 to 20 instances by CPU. | VM Scale Sets | Native autoscale for identical VMs. |
| 3 | Microservices platform needs Kubernetes APIs, ingress, and node pools. | AKS | Full Kubernetes control plane and orchestration. |
| 4 | Enterprise API needs slots, autoscale, and minimal ops. | App Service | PaaS web hosting with staging slots. |
| 5 | Containerized API needs HTTP and queue scaling without Kubernetes management. | Container Apps | KEDA-based scaling and low ops overhead. |
| 6 | Event-driven code must stay warm and integrate with a VNet. | Functions Premium | Pre-warmed instances and VNet support. |
| 7 | Security team wants to run a one-time forensics container fast. | ACI | Fast isolated container execution. |
| 8 | Batch rendering farm must process thousands of jobs cheaply. | Azure Batch | Scheduler-managed compute pools and low-priority nodes. |
| 9 | Dev/test build agent can be interrupted anytime for lower cost. | Spot VM | Significant discount for interruptible compute. |
| 10 | Existing containerized web app needs scale-to-zero and revision-based releases. | Container Apps | Best balance of serverless operations and container hosting. |

### Answer Key with Reasoning
1. **Virtual Machines** - Best when full guest OS control, custom software installation, and legacy support are required.
2. **VM Scale Sets** - Best for a stateless, metric-driven scale scenario using consistent VM images.
3. **AKS** - Best when the platform explicitly needs Kubernetes constructs such as node pools, ingress, and pod orchestration.
4. **App Service** - Best for standard web/API hosting with deployment slots and lower platform management.
5. **Container Apps** - Best for event-driven containers and microservices without Kubernetes operational burden.
6. **Functions Premium** - Best when serverless code also needs warm instances and VNet integration.
7. **ACI** - Best for short-lived, isolated, quick-start containers.
8. **Azure Batch** - Best for large-scale scheduled compute tasks and HPC-style task distribution.
9. **Spot VM** - Best when interruption is acceptable and cost is the main driver.
10. **Container Apps** - Best when the app is already containerized and needs scale-to-zero, revisions, and simplified operations.

### Full CLI + PowerShell Commands

#### Azure CLI
```bash
# Use these inventory commands to test your reasoning against real resources in a lab subscription.
az vm list -d -o table
az vmss list -o table
az aks list -o table
az webapp list -o table
az containerapp list -o table
az functionapp list -o table
az container list -o table
az batch account list -o table
```

#### PowerShell
```powershell
Get-AzVM -Status
Get-AzVmss
Get-AzAksCluster
Get-AzWebApp
Get-AzContainerApp
Get-AzFunctionApp
Get-AzContainerGroup
Get-AzBatchAccount
```

### Verification Steps
- Re-answer the scenarios without looking at the answer column.
- For each scenario, identify the deciding factor first: **control**, **scale**, **ops**, **cost**, **Kubernetes**, or **event-driven**.
- Review the summary table below and explain why the second-best option is weaker.

### Cleanup
No Azure resources are required for this decision exercise. If you used the inventory commands against a live lab subscription, delete any temporary resource groups created during Labs 1-9.

### Exam Tip
On AZ-305, the wrong answers are often technically possible. Choose the service that meets the requirement with the **best architectural fit and least unnecessary operational overhead**.

### Exam-Style Review Questions
1. Why is AKS usually the wrong answer when Kubernetes is not explicitly required?
2. When is Container Apps a better fit than App Service?
3. Why are VMs still relevant in a PaaS-first cloud architecture exam?

---

## Compute Decision Summary Table

| Requirement Pattern | Best Service | Why It Wins | Common Wrong Answer |
|---|---|---|---|
| Full OS control, custom drivers, legacy software | Virtual Machines | Full guest OS and image control | App Service |
| Elastic stateless VM fleet | VM Scale Sets | Autoscale + consistent VM instances | Individual VMs |
| Kubernetes orchestration | AKS | Managed Kubernetes with node pools and ingress | Container Apps |
| Standard web app/API with minimal ops | App Service | PaaS hosting, slots, autoscale | AKS |
| Event-driven containers, scale-to-zero | Container Apps | KEDA + revisions + low ops | AKS |
| Event-driven code with premium networking/latency needs | Functions Premium | Always-ready instances + VNet support | Functions Consumption |
| One-off container or burst execution | ACI | Fast startup, simple execution model | AKS |
| Interruptible compute for lowest cost | Spot VM | Deep discount for tolerant workloads | Reserved Instance |
| Massive queued parallel compute | Azure Batch | Job/task scheduler + pool autoscale | Functions |
| Predictable long-running workload cost savings | Reserved VM Instance | Best steady-state savings | Spot VM |

## Final AZ-305 Compute Takeaways

- Prefer the **lowest-management platform** that still satisfies the workload.
- Prefer **Availability Zones** over Availability Sets for new production HA designs when region support exists.
- Use **VMSS** for stateless scale, **AKS** for Kubernetes, **App Service** for standard web apps, **Container Apps** for serverless containers, and **Functions Premium** for event-driven code with enterprise requirements.
- Use **ACI** for rapid isolated containers and **Batch** for scheduler-driven parallel jobs.
- For cost questions, match the pricing model to workload predictability and interruption tolerance.
