# Azure Hybrid Hands-On Labs (AZ-305)

> 📖 **Cheat Sheet:** [Azure Hybrid](../Azure-Hybrid.md)

> **Primary exam domain:** Design infrastructure solutions (30-35%)  
> **Secondary domains:** Design identity, governance, and monitoring solutions; Design business continuity solutions  
> **Tools used:** Azure CLI, Azure PowerShell, kubectl, Windows DNS, Azure Arc, Azure Site Recovery, Azure Migrate, Microsoft Entra Connect  
> **Important:** Use an isolated lab subscription or resource group. Several labs create billable resources and hybrid management agents.

---

## Table of Contents

| Lab | Title | Primary Focus |
|-----|-------|---------------|
| [Lab 1](#lab-1-azure-arc-enabled-servers) | Azure Arc-enabled Servers | Hybrid server governance |
| [Lab 2](#lab-2-azure-arc-enabled-kubernetes) | Azure Arc-enabled Kubernetes | Multi-cloud Kubernetes operations |
| [Lab 3](#lab-3-hybrid-dns-configuration) | Hybrid DNS Configuration | Private name resolution |
| [Lab 4](#lab-4-azure-automation-hybrid-runbook-worker) | Azure Automation Hybrid Runbook Worker | On-prem automation from Azure |
| [Lab 5](#lab-5-azure-site-recovery-for-hybrid-dr) | Azure Site Recovery for Hybrid DR | Disaster recovery |
| [Lab 6](#lab-6-azure-stack-hci-conceptual-overview) | Azure Stack HCI (Conceptual Overview) | Azure-consistent hybrid infrastructure |
| [Lab 7](#lab-7-azure-migrate-assessment) | Azure Migrate Assessment | Migration planning |
| [Lab 8](#lab-8-hybrid-identity-integration) | Hybrid Identity Integration | Identity extension to Azure |
| [Lab 9](#lab-9-hybrid-decision-exercise) | Hybrid Decision Exercise | Exam scenario practice |

---

## Lab 1: Azure Arc-enabled Servers

### Objective
Generate an Azure Arc onboarding flow, connect a Windows or Linux server to Azure, verify the Arc-enabled server resource, deploy Azure Monitor Agent, apply Azure Policy, and prepare the server for Azure Update Manager.

**When to Use This:** Use Azure Arc-enabled servers when you must manage non-Azure servers from Azure without fully migrating them.

**Key AZ-305 Concepts:** Consistent governance across on-prem and multi-cloud servers, Azure Policy on non-Azure resources, extension-based management, centralized monitoring, and update orchestration.

### Exam Domain Mapping
- **Primary:** Design infrastructure solutions (30-35%)
- **Secondary:** Design identity, governance, and monitoring solutions (25-30%)

### Prerequisites
- Azure subscription with Contributor access on the target resource group
- Azure CLI logged in with `az login`
- Azure PowerShell logged in with `Connect-AzAccount`
- A Windows Server or Linux server with outbound internet access to Azure, or an Azure VM used as a simulation target
- `connectedmachine` Azure CLI extension
- A Log Analytics workspace for Azure Monitor Agent

### Architecture and Design Rationale
Arc projects non-Azure servers into Azure Resource Manager so you can apply RBAC, tags, policy, monitoring, and update management consistently. For AZ-305, Arc is the right choice when workloads must stay outside Azure because of latency, sovereignty, licensing, or phased migration requirements.

### Implementation Steps
1. Register required resource providers and install the Arc CLI extension.
2. Create a lab resource group and Log Analytics workspace.
3. Create a service principal for Arc onboarding.
4. Install the Connected Machine agent on Windows or Linux.
5. Connect the server to Azure Arc.
6. Deploy Azure Monitor Agent and associate it to the workspace.
7. Assign an Azure Policy initiative to audit or deploy baseline controls.
8. Create a maintenance configuration for Azure Update Manager.

### Full CLI + PowerShell Commands

#### Azure CLI
```bash
RG="rg-hybrid-arc-servers"
LOCATION="eastus"
LAW="lawhybridarc$RANDOM"
MACHINE_NAME="arc-server-01"
SUB_ID=$(az account show --query id -o tsv)
TENANT_ID=$(az account show --query tenantId -o tsv)

az extension add --name connectedmachine --upgrade
az extension add --name maintenance --upgrade
az provider register --namespace Microsoft.HybridCompute
az provider register --namespace Microsoft.GuestConfiguration
az provider register --namespace Microsoft.Insights
az provider register --namespace Microsoft.Maintenance

az group create --name $RG --location $LOCATION
az monitor log-analytics workspace create \
  --resource-group $RG \
  --workspace-name $LAW \
  --location $LOCATION \
  --retention-time 30

WORKSPACE_ID=$(az monitor log-analytics workspace show -g $RG -n $LAW --query id -o tsv)

az ad sp create-for-rbac \
  --name "arc-onboard-sp-$RANDOM" \
  --role "Azure Connected Machine Onboarding" \
  --scopes "/subscriptions/$SUB_ID/resourceGroups/$RG"

# Windows simulation: run these locally on the server in an elevated prompt.
# Linux simulation: run the curl/bash block on the Linux host.
```

**Linux agent install and connect**
```bash
curl -fsSL https://aka.ms/azcmagent -o install_linux_azcmagent.sh
sudo bash install_linux_azcmagent.sh
sudo azcmagent connect \
  --resource-group "$RG" \
  --tenant-id "$TENANT_ID" \
  --subscription-id "$SUB_ID" \
  --location "$LOCATION" \
  --cloud "AzureCloud" \
  --service-principal-id "<appId-from-sp-output>" \
  --service-principal-secret "<password-from-sp-output>"

az connectedmachine list -g $RG -o table

# Install Azure Monitor Agent on the Arc server.
az connectedmachine extension create \
  --machine-name $MACHINE_NAME \
  --resource-group $RG \
  --location $LOCATION \
  --name AzureMonitorLinuxAgent \
  --publisher Microsoft.Azure.Monitor \
  --type AzureMonitorLinuxAgent \
  --settings '{}'

# Assign built-in Azure Policy initiative at resource-group scope.
POLICY_SET_ID=$(az policy set-definition list --query "[?contains(displayName, 'Azure Monitor')].id | [0]" -o tsv)
az policy assignment create \
  --name arc-monitoring-baseline \
  --policy-set-definition $POLICY_SET_ID \
  --scope /subscriptions/$SUB_ID/resourceGroups/$RG

# Create an Update Manager maintenance configuration.
az maintenance configuration create \
  --resource-group $RG \
  --resource-name mc-arc-monthly \
  --location $LOCATION \
  --maintenance-scope InGuestPatch \
  --namespace Microsoft.Maintenance \
  --extension-properties '{"inGuestPatchMode":"User"}' \
  --install-patches '{"linuxParameters":{"classificationsToInclude":"Critical,Security"},"rebootSetting":"IfRequired"}' \
  --maintenance-window '{"startDateTime":"2026-01-10 02:00","duration":"03:00","timeZone":"UTC","recurEvery":"Month Second Tuesday"}'
```

#### PowerShell
```powershell
$RG = "rg-hybrid-arc-servers"
$Location = "eastus"
$Law = "lawhybridarc$(Get-Random)"
$SubId = (Get-AzContext).Subscription.Id
$TenantId = (Get-AzContext).Tenant.Id
$MachineName = "arc-server-01"

Register-AzResourceProvider -ProviderNamespace Microsoft.HybridCompute
Register-AzResourceProvider -ProviderNamespace Microsoft.GuestConfiguration
Register-AzResourceProvider -ProviderNamespace Microsoft.Insights
Register-AzResourceProvider -ProviderNamespace Microsoft.Maintenance

New-AzResourceGroup -Name $RG -Location $Location
$workspace = New-AzOperationalInsightsWorkspace `
  -ResourceGroupName $RG `
  -Name $Law `
  -Location $Location `
  -Sku PerGB2018 `
  -RetentionInDays 30

$sp = New-AzADServicePrincipal `
  -DisplayName "arc-onboard-sp-$(Get-Random)" `
  -Role "Azure Connected Machine Onboarding" `
  -Scope "/subscriptions/$SubId/resourceGroups/$RG"

# Run locally on Windows Server to install the agent.
Invoke-WebRequest -Uri "https://aka.ms/AzureConnectedMachineAgent" -OutFile "$env:TEMP\AzureConnectedMachineAgent.msi"
Start-Process msiexec.exe -ArgumentList "/i $env:TEMP\AzureConnectedMachineAgent.msi /qn" -Wait
& "$env:ProgramFiles\AzureConnectedMachineAgent\azcmagent.exe" connect `
  --resource-group $RG `
  --tenant-id $TenantId `
  --subscription-id $SubId `
  --location $Location `
  --service-principal-id $sp.AppId `
  --service-principal-secret "<client-secret>"

New-AzConnectedMachineExtension `
  -MachineName $MachineName `
  -ResourceGroupName $RG `
  -Name "AzureMonitorWindowsAgent" `
  -Location $Location `
  -Publisher "Microsoft.Azure.Monitor" `
  -ExtensionType "AzureMonitorWindowsAgent" `
  -Setting '{}'

$policySet = Get-AzPolicySetDefinition | Where-Object { $_.Properties.DisplayName -like '*Azure Monitor*' } | Select-Object -First 1
New-AzPolicyAssignment -Name "arc-monitoring-baseline" -PolicySetDefinition $policySet -Scope "/subscriptions/$SubId/resourceGroups/$RG"

New-AzMaintenanceConfiguration `
  -ResourceGroupName $RG `
  -ResourceName "mc-arc-monthly" `
  -Location $Location `
  -MaintenanceScope InGuestPatch `
  -StartDateTime "2026-01-10 02:00" `
  -Duration "03:00" `
  -Timezone "UTC" `
  -RecurEvery "Month Second Tuesday"
```

### Validation and Success Criteria
- The server appears in **Azure Arc > Machines**.
- Azure Monitor Agent extension shows **Succeeded**.
- Policy assignment exists and returns compliance state.
- Update Manager maintenance configuration exists and can be associated to Arc resources.

### Verification Steps
- Run `az connectedmachine list -g $RG -o table`.
- Run `az connectedmachine extension list --machine-name $MACHINE_NAME -g $RG -o table`.
- In the portal, confirm the Arc machine status is **Connected**.
- Open **Azure Update Manager** and confirm the Arc machine is discoverable.

### Cleanup

#### Azure CLI
```bash
az policy assignment delete --name arc-monitoring-baseline --scope /subscriptions/$SUB_ID/resourceGroups/$RG
az maintenance configuration delete --resource-group $RG --resource-name mc-arc-monthly --yes
az group delete --name $RG --yes --no-wait
```

#### PowerShell
```powershell
Remove-AzPolicyAssignment -Name "arc-monitoring-baseline" -Scope "/subscriptions/$SubId/resourceGroups/$RG"
Remove-AzMaintenanceConfiguration -ResourceGroupName $RG -ResourceName "mc-arc-monthly"
Remove-AzResourceGroup -Name $RG -Force -AsJob
```

### Exam Tip
If a question says the servers must stay on-premises or in another cloud, but governance, monitoring, extensions, and patching must be centralized in Azure, Arc-enabled servers are usually the best answer.

### Exam-Style Review Questions
1. When is Azure Arc a better answer than rehosting the server into Azure?
2. Why do many Azure Policy effects on Arc servers depend on extensions or guest configuration?
3. What is the architectural benefit of combining Arc, AMA, Policy, and Update Manager?

---

## Lab 2: Azure Arc-enabled Kubernetes

### Objective
Connect an existing Kubernetes cluster to Azure Arc, deploy a Flux GitOps configuration, apply Azure Policy, enable Container Insights, and publish an application through GitOps.

**When to Use This:** Use Arc-enabled Kubernetes when you need centralized control for on-premises or multi-cloud Kubernetes clusters without moving them into AKS.

**Key AZ-305 Concepts:** Arc as a control plane, GitOps for drift control, policy-based governance, centralized observability, and platform consistency across clusters.

### Exam Domain Mapping
- **Primary:** Design infrastructure solutions (30-35%)
- **Secondary:** Design identity, governance, and monitoring solutions (25-30%)

### Prerequisites
- Existing Kubernetes cluster (on-prem, AKS used as a simulation, kind, k3s, EKS, or GKE)
- `kubectl` configured to access the cluster
- Azure CLI with `connectedk8s`, `k8s-extension`, and `k8s-configuration` extensions
- Log Analytics workspace for Container Insights
- A Git repository containing Kubernetes manifests or Helm charts

### Architecture and Design Rationale
Arc-enabled Kubernetes is ideal when the cluster location is fixed but governance and operations must be centralized. For AZ-305, know that Arc plus Flux is a strong answer for multi-cluster standardization, while AKS is the better answer when Azure-hosted managed Kubernetes is allowed and preferred.

### Implementation Steps
1. Create a resource group and Log Analytics workspace.
2. Connect the cluster to Azure Arc.
3. Install the Azure Policy extension.
4. Install Azure Monitor Container Insights.
5. Create a Flux configuration for platform configuration.
6. Deploy an application using a second GitOps configuration.
7. Validate compliance and workload deployment.

### Full CLI + PowerShell Commands

#### Azure CLI
```bash
RG="rg-hybrid-arc-k8s"
LOCATION="eastus"
CLUSTER_NAME="arc-k8s-lab"
LAW="lawhybridk8s$RANDOM"
GITOPS_NAME="platform-config"
APP_CONFIG_NAME="store-app-config"
GIT_URL="https://github.com/Azure-Samples/azure-voting-app-redis.git"
SUB_ID=$(az account show --query id -o tsv)

az extension add --name connectedk8s --upgrade
az extension add --name k8s-extension --upgrade
az extension add --name k8s-configuration --upgrade
az provider register --namespace Microsoft.Kubernetes
az provider register --namespace Microsoft.KubernetesConfiguration
az provider register --namespace Microsoft.PolicyInsights

az group create --name $RG --location $LOCATION
az monitor log-analytics workspace create \
  --resource-group $RG \
  --workspace-name $LAW \
  --location $LOCATION

LAW_ID=$(az monitor log-analytics workspace show -g $RG -n $LAW --query id -o tsv)
LAW_CUSTOMER_ID=$(az monitor log-analytics workspace show -g $RG -n $LAW --query customerId -o tsv)
LAW_SHARED_KEY=$(az monitor log-analytics workspace get-shared-keys -g $RG -n $LAW --query primarySharedKey -o tsv)

az connectedk8s connect \
  --name $CLUSTER_NAME \
  --resource-group $RG \
  --location $LOCATION

az k8s-extension create \
  --cluster-name $CLUSTER_NAME \
  --resource-group $RG \
  --cluster-type connectedClusters \
  --name azurepolicy \
  --extension-type microsoft.policyinsights

az k8s-extension create \
  --cluster-name $CLUSTER_NAME \
  --resource-group $RG \
  --cluster-type connectedClusters \
  --name azuremonitor-containers \
  --extension-type Microsoft.AzureMonitor.Containers \
  --configuration-settings logAnalyticsWorkspaceResourceID=$LAW_ID \
  --configuration-protected-settings logAnalyticsWorkspaceKey=$LAW_SHARED_KEY

az k8s-configuration flux create \
  --cluster-name $CLUSTER_NAME \
  --resource-group $RG \
  --cluster-type connectedClusters \
  --name $GITOPS_NAME \
  --namespace flux-system \
  --scope cluster \
  --url $GIT_URL \
  --branch main

az k8s-configuration flux create \
  --cluster-name $CLUSTER_NAME \
  --resource-group $RG \
  --cluster-type connectedClusters \
  --name $APP_CONFIG_NAME \
  --namespace apps \
  --scope cluster \
  --url https://github.com/Azure/arc-k8s-demo \
  --branch main \
  --kustomization name=store path=./store prune=true
```

#### PowerShell
```powershell
$RG = "rg-hybrid-arc-k8s"
$Location = "eastus"
$ClusterName = "arc-k8s-lab"
$Law = "lawhybridk8s$(Get-Random)"
$GitOpsName = "platform-config"
$AppConfigName = "store-app-config"

New-AzResourceGroup -Name $RG -Location $Location
$workspace = New-AzOperationalInsightsWorkspace `
  -ResourceGroupName $RG `
  -Name $Law `
  -Location $Location `
  -Sku PerGB2018

$workspaceKeys = Get-AzOperationalInsightsWorkspaceSharedKey -ResourceGroupName $RG -Name $Law

az connectedk8s connect `
  --name $ClusterName `
  --resource-group $RG `
  --location $Location

az k8s-extension create `
  --cluster-name $ClusterName `
  --resource-group $RG `
  --cluster-type connectedClusters `
  --name azurepolicy `
  --extension-type microsoft.policyinsights

az k8s-extension create `
  --cluster-name $ClusterName `
  --resource-group $RG `
  --cluster-type connectedClusters `
  --name azuremonitor-containers `
  --extension-type Microsoft.AzureMonitor.Containers `
  --configuration-settings logAnalyticsWorkspaceResourceID=$($workspace.ResourceId) `
  --configuration-protected-settings logAnalyticsWorkspaceKey=$($workspaceKeys.PrimarySharedKey)

az k8s-configuration flux create `
  --cluster-name $ClusterName `
  --resource-group $RG `
  --cluster-type connectedClusters `
  --name $GitOpsName `
  --namespace flux-system `
  --scope cluster `
  --url https://github.com/Azure/arc-k8s-demo `
  --branch main

kubectl get pods -A
```

### Validation and Success Criteria
- The Kubernetes cluster appears under **Azure Arc > Kubernetes clusters**.
- Policy and Azure Monitor extensions show **Installed**.
- Flux configuration status is **Ready**.
- Application pods are deployed successfully.

### Verification Steps
- Run `az connectedk8s list -g $RG -o table`.
- Run `az k8s-extension list --cluster-name $CLUSTER_NAME --resource-group $RG --cluster-type connectedClusters -o table`.
- Run `az k8s-configuration flux list --cluster-name $CLUSTER_NAME --resource-group $RG --cluster-type connectedClusters -o table`.
- Run `kubectl get pods -A` and confirm application pods are healthy.

### Cleanup

#### Azure CLI
```bash
az k8s-configuration flux delete --cluster-name $CLUSTER_NAME --resource-group $RG --cluster-type connectedClusters --name $APP_CONFIG_NAME --yes
az k8s-configuration flux delete --cluster-name $CLUSTER_NAME --resource-group $RG --cluster-type connectedClusters --name $GITOPS_NAME --yes
az connectedk8s delete --name $CLUSTER_NAME --resource-group $RG --yes
az group delete --name $RG --yes --no-wait
```

#### PowerShell
```powershell
az k8s-configuration flux delete --cluster-name $ClusterName --resource-group $RG --cluster-type connectedClusters --name $AppConfigName --yes
az k8s-configuration flux delete --cluster-name $ClusterName --resource-group $RG --cluster-type connectedClusters --name $GitOpsName --yes
az connectedk8s delete --name $ClusterName --resource-group $RG --yes
Remove-AzResourceGroup -Name $RG -Force -AsJob
```

### Exam Tip
If the cluster must remain outside Azure, Arc-enabled Kubernetes plus Flux plus Azure Policy is usually more appropriate than recommending AKS migration.

### Exam-Style Review Questions
1. Why is GitOps especially valuable for Arc-enabled Kubernetes?
2. When would AKS be a better recommendation than Arc-enabled Kubernetes?
3. What does Arc add to a non-Azure Kubernetes cluster from an AZ-305 perspective?

---

## Lab 3: Hybrid DNS Configuration

### Objective
Create hybrid name resolution for private workloads by using Azure Private DNS, a hub-based DNS forwarding pattern, conditional forwarding from on-premises DNS, private endpoint resolution, and split-horizon DNS.

**When to Use This:** Use this pattern when workloads span Azure and on-premises networks and must resolve private service names consistently.

**Key AZ-305 Concepts:** Hub-and-spoke DNS, Azure Private DNS, Azure DNS Private Resolver, private endpoint name resolution, split-horizon DNS, and hybrid connectivity dependencies.

### Exam Domain Mapping
- **Primary:** Design infrastructure solutions (30-35%)
- **Secondary:** Design identity, governance, and monitoring solutions (25-30%)

### Prerequisites
- Azure subscription with Contributor access
- Connectivity between Azure and on-premises DNS (VPN or ExpressRoute, or a simulated lab network)
- Windows DNS server on-premises or in a simulated VM
- Azure CLI and Azure PowerShell
- Resource providers for networking and private DNS

### Architecture and Design Rationale
In modern Azure designs, Azure DNS Private Resolver is usually preferable to self-managed DNS forwarder VMs because it reduces operational overhead. Split-horizon DNS is common when the same namespace must resolve to different IPs internally versus externally.

### Implementation Steps
1. Create a hub VNet and subnets for inbound and outbound DNS Private Resolver endpoints.
2. Create a spoke VNet or a test subnet for a private endpoint.
3. Create a private DNS zone and link it to the VNet.
4. Deploy Azure DNS Private Resolver.
5. Configure on-premises conditional forwarding to the inbound endpoint.
6. Create a private endpoint and validate DNS.
7. Create a public zone for split-horizon comparison.

### Full CLI + PowerShell Commands

#### Azure CLI
```bash
RG="rg-hybrid-dns"
LOCATION="eastus"
HUB_VNET="vnet-hub-dns"
SPOKE_VNET="vnet-spoke-app"
PE_SUBNET="snet-private-endpoint"
INBOUND_SUBNET="snet-dns-inbound"
OUTBOUND_SUBNET="snet-dns-outbound"
DNS_RESOLVER="dnspr-hybrid"
RULESET="dns-ruleset-hybrid"
PRIVATE_ZONE="privatelink.blob.core.windows.net"
PUBLIC_ZONE="apps.contoso.com"
STORAGE="sthyrbiddns$RANDOM"

az group create --name $RG --location $LOCATION

az network vnet create \
  --resource-group $RG \
  --name $HUB_VNET \
  --location $LOCATION \
  --address-prefix 10.20.0.0/16 \
  --subnet-name $INBOUND_SUBNET \
  --subnet-prefix 10.20.1.0/28

az network vnet subnet create -g $RG --vnet-name $HUB_VNET -n $OUTBOUND_SUBNET --address-prefixes 10.20.2.0/28
az network vnet create \
  --resource-group $RG \
  --name $SPOKE_VNET \
  --location $LOCATION \
  --address-prefix 10.21.0.0/16 \
  --subnet-name $PE_SUBNET \
  --subnet-prefix 10.21.1.0/24

az network private-dns zone create --resource-group $RG --name $PRIVATE_ZONE
az network private-dns link vnet create \
  --resource-group $RG \
  --zone-name $PRIVATE_ZONE \
  --name link-spoke \
  --virtual-network $SPOKE_VNET \
  --registration-enabled false

az network dns-resolver create --resource-group $RG --name $DNS_RESOLVER --location $LOCATION --virtual-network $HUB_VNET
az network dns-resolver inbound-endpoint create \
  --resource-group $RG \
  --dns-resolver-name $DNS_RESOLVER \
  --name inbound01 \
  --location $LOCATION \
  --ip-configurations private-ip-allocation-method Dynamic subnet=/subscriptions/$(az account show --query id -o tsv)/resourceGroups/$RG/providers/Microsoft.Network/virtualNetworks/$HUB_VNET/subnets/$INBOUND_SUBNET

az network dns-resolver outbound-endpoint create \
  --resource-group $RG \
  --dns-resolver-name $DNS_RESOLVER \
  --name outbound01 \
  --location $LOCATION \
  --subnet /subscriptions/$(az account show --query id -o tsv)/resourceGroups/$RG/providers/Microsoft.Network/virtualNetworks/$HUB_VNET/subnets/$OUTBOUND_SUBNET

az network dns-resolver forwarding-ruleset create --resource-group $RG --name $RULESET --location $LOCATION
az network dns-resolver forwarding-rule create \
  --resource-group $RG \
  --ruleset-name $RULESET \
  --name corp-contoso-local \
  --domain-name corp.contoso.local. \
  --target-dns-servers ip-address=10.1.0.4 port=53

az network dns-resolver forwarding-ruleset vnet-link create \
  --resource-group $RG \
  --ruleset-name $RULESET \
  --name link-hub \
  --virtual-network /subscriptions/$(az account show --query id -o tsv)/resourceGroups/$RG/providers/Microsoft.Network/virtualNetworks/$HUB_VNET

az storage account create --name $STORAGE --resource-group $RG --location $LOCATION --sku Standard_LRS --kind StorageV2
az network private-endpoint create \
  --resource-group $RG \
  --name pe-storage \
  --vnet-name $SPOKE_VNET \
  --subnet $PE_SUBNET \
  --private-connection-resource-id $(az storage account show -g $RG -n $STORAGE --query id -o tsv) \
  --group-id blob \
  --connection-name pe-storage-conn

az network private-endpoint dns-zone-group create \
  --resource-group $RG \
  --endpoint-name pe-storage \
  --name default \
  --private-dns-zone $PRIVATE_ZONE \
  --zone-name $PRIVATE_ZONE

az network dns zone create --resource-group $RG --name $PUBLIC_ZONE
az network dns record-set a add-record --resource-group $RG --zone-name $PUBLIC_ZONE --record-set-name app --ipv4-address 52.160.10.10
```

#### PowerShell
```powershell
$RG = "rg-hybrid-dns"
$Location = "eastus"
$HubVNet = "vnet-hub-dns"
$SpokeVNet = "vnet-spoke-app"
$PrivateZone = "privatelink.blob.core.windows.net"
$PublicZone = "apps.contoso.com"

New-AzResourceGroup -Name $RG -Location $Location

$hubInbound = New-AzVirtualNetworkSubnetConfig -Name "snet-dns-inbound" -AddressPrefix "10.20.1.0/28"
$hubOutbound = New-AzVirtualNetworkSubnetConfig -Name "snet-dns-outbound" -AddressPrefix "10.20.2.0/28"
$hub = New-AzVirtualNetwork -Name $HubVNet -ResourceGroupName $RG -Location $Location -AddressPrefix "10.20.0.0/16" -Subnet $hubInbound,$hubOutbound

$spokeSubnet = New-AzVirtualNetworkSubnetConfig -Name "snet-private-endpoint" -AddressPrefix "10.21.1.0/24"
$spoke = New-AzVirtualNetwork -Name $SpokeVNet -ResourceGroupName $RG -Location $Location -AddressPrefix "10.21.0.0/16" -Subnet $spokeSubnet

$zone = New-AzPrivateDnsZone -ResourceGroupName $RG -Name $PrivateZone
New-AzPrivateDnsVirtualNetworkLink -ResourceGroupName $RG -ZoneName $PrivateZone -Name "link-spoke" -VirtualNetworkId $spoke.Id

# On the on-prem Windows DNS server, point conditional forwarding to the Azure DNS Private Resolver inbound endpoint.
Add-DnsServerConditionalForwarderZone -Name "blob.core.windows.net" -MasterServers 10.20.1.4 -ReplicationScope Forest

# Split-horizon example: create a public zone and separate internal mapping.
New-AzDnsZone -Name $PublicZone -ResourceGroupName $RG
New-AzDnsRecordSet -Name "app" -RecordType A -ZoneName $PublicZone -ResourceGroupName $RG -Ttl 300 -DnsRecords (New-AzDnsRecordConfig -IPv4Address "52.160.10.10")
```

### Validation and Success Criteria
- Private endpoint FQDN resolves to a private IP from Azure.
- On-premises DNS can forward private Azure namespace queries to Azure.
- The same business name can resolve differently internally and externally.

### Verification Steps
- Run `nslookup <storageaccount>.blob.core.windows.net` from an Azure VM and from an on-prem server.
- Confirm the response is a private IP inside the spoke VNet range.
- Run `Resolve-DnsName app.apps.contoso.com` internally and compare it with public internet resolution.

### Cleanup

#### Azure CLI
```bash
az group delete --name $RG --yes --no-wait
```

#### PowerShell
```powershell
Remove-DnsServerConditionalForwarderZone -Name "blob.core.windows.net" -Force
Remove-AzResourceGroup -Name $RG -Force -AsJob
```

### Exam Tip
For hybrid name resolution, the best answer is rarely “just use hosts files” or “deploy custom DNS everywhere.” Prefer Azure Private DNS plus Azure DNS Private Resolver unless a scenario explicitly requires legacy DNS appliances.

### Exam-Style Review Questions
1. Why is Azure DNS Private Resolver often preferable to a DNS forwarder VM in Azure?
2. What DNS design is required for private endpoints across Azure and on-premises?
3. When is split-horizon DNS the correct design choice?

---

## Lab 4: Azure Automation Hybrid Runbook Worker

### Objective
Create an Automation Account, register a Hybrid Runbook Worker group, install the Hybrid Worker extension on an Arc-enabled server, run a runbook locally on that machine, and schedule recurring execution.

**When to Use This:** Use Hybrid Runbook Workers when automation must run close to on-premises resources that Azure cannot directly reach.

**Key AZ-305 Concepts:** Automation control plane in Azure, local execution on hybrid resources, Arc-based extension deployment, scheduling, and credential/run-as design.

### Exam Domain Mapping
- **Primary:** Design infrastructure solutions (30-35%)
- **Secondary:** Design identity, governance, and monitoring solutions (25-30%)

### Prerequisites
- An Arc-enabled Windows or Linux server from Lab 1
- Azure PowerShell `Az.Automation` module or Azure CLI
- Contributor access on the automation account resource group
- Outbound connectivity from the hybrid worker to Azure Automation endpoints

### Architecture and Design Rationale
Hybrid Runbook Workers allow centralized orchestration while keeping execution local to the target environment. This is useful for line-of-business systems, private networks, or domain-joined operations where cloud-only runbooks would not have reachability.

### Implementation Steps
1. Create an Automation Account.
2. Create a Hybrid Runbook Worker group.
3. Install the Hybrid Worker extension on the Arc-enabled server.
4. Create a test PowerShell runbook.
5. Start the runbook on the hybrid worker.
6. Schedule recurring execution.

### Full CLI + PowerShell Commands

#### Azure CLI
```bash
RG="rg-hybrid-automation"
LOCATION="eastus"
AA="aa-hybrid-$RANDOM"
HW_GROUP="hwg-branch01"
RUNBOOK="Get-HybridStatus"
ARC_MACHINE="arc-server-01"

az extension add --name automation --upgrade
az extension add --name connectedmachine --upgrade
az group create --name $RG --location $LOCATION
az automation account create --name $AA --resource-group $RG --location $LOCATION --sku Basic

SUB_ID=$(az account show --query id -o tsv)
AUTOMATION_ID=$(az automation account show -g $RG -n $AA --query id -o tsv)
AUTOMATION_URL="https://${AA}.azure-automation.net"

az rest --method put \
  --url "https://management.azure.com/subscriptions/$SUB_ID/resourceGroups/$RG/providers/Microsoft.Automation/automationAccounts/$AA/hybridRunbookWorkerGroups/$HW_GROUP?api-version=2023-11-01" \
  --body '{}'

az connectedmachine extension create \
  --machine-name $ARC_MACHINE \
  --resource-group rg-hybrid-arc-servers \
  --location $LOCATION \
  --name HybridWorkerExtension \
  --publisher Microsoft.Azure.Automation.HybridWorker \
  --type HybridWorkerForWindows \
  --settings "{\"AutomationAccountURL\":\"$AUTOMATION_URL\",\"HybridWorkerGroupName\":\"$HW_GROUP\"}"

cat > runbook.ps1 <<'EOF'
param([string]$Name = 'AZ-305')
Write-Output "Hybrid worker executed on $env:COMPUTERNAME for $Name"
Get-Date
EOF

az automation runbook create --automation-account-name $AA --resource-group $RG --name $RUNBOOK --type PowerShell
az automation runbook replace-content --automation-account-name $AA --resource-group $RG --name $RUNBOOK --content @runbook.ps1
az automation runbook publish --automation-account-name $AA --resource-group $RG --name $RUNBOOK
az automation schedule create --automation-account-name $AA --resource-group $RG --name daily-8am --start-time 2026-01-10T08:00:00+00:00 --frequency Day --interval 1
az automation job create --automation-account-name $AA --resource-group $RG --runbook-name $RUNBOOK --parameters Name=BranchOps
```

#### PowerShell
```powershell
$RG = "rg-hybrid-automation"
$Location = "eastus"
$AutomationAccount = "aa-hybrid-$(Get-Random)"
$HybridWorkerGroup = "hwg-branch01"
$RunbookName = "Get-HybridStatus"
$ArcMachine = "arc-server-01"

New-AzResourceGroup -Name $RG -Location $Location
$aa = New-AzAutomationAccount -Name $AutomationAccount -ResourceGroupName $RG -Location $Location -Plan Basic
New-AzAutomationHybridRunbookWorkerGroup -AutomationAccountName $AutomationAccount -Name $HybridWorkerGroup -ResourceGroupName $RG

New-AzConnectedMachineExtension `
  -MachineName $ArcMachine `
  -ResourceGroupName "rg-hybrid-arc-servers" `
  -Name "HybridWorkerExtension" `
  -Location $Location `
  -Publisher "Microsoft.Azure.Automation.HybridWorker" `
  -ExtensionType "HybridWorkerForWindows" `
  -Setting (@{ AutomationAccountURL = "https://$AutomationAccount.azure-automation.net"; HybridWorkerGroupName = $HybridWorkerGroup } | ConvertTo-Json)

$runbookContent = @"
param([string]`$Name = 'AZ-305')
Write-Output "Hybrid worker executed on `$env:COMPUTERNAME for `$Name"
Get-Date
"@
$runbookFile = Join-Path (Get-Location) "runbook.ps1"
Set-Content -Path $runbookFile -Value $runbookContent

Import-AzAutomationRunbook -AutomationAccountName $AutomationAccount -ResourceGroupName $RG -Name $RunbookName -Type PowerShell -Path $runbookFile -Force -Published
$job = Start-AzAutomationRunbook -AutomationAccountName $AutomationAccount -ResourceGroupName $RG -Name $RunbookName -Parameters @{ Name = 'BranchOps' } -RunOn $HybridWorkerGroup
New-AzAutomationSchedule -AutomationAccountName $AutomationAccount -ResourceGroupName $RG -Name "daily-8am" -StartTime "2026-01-10T08:00:00Z" -DayInterval 1
Register-AzAutomationScheduledRunbook -AutomationAccountName $AutomationAccount -ResourceGroupName $RG -RunbookName $RunbookName -ScheduleName "daily-8am"
```

### Validation and Success Criteria
- The Hybrid Runbook Worker group exists.
- The Arc machine has the Hybrid Worker extension installed.
- The runbook executes successfully on the hybrid worker.
- The schedule is associated to the runbook.

### Verification Steps
- Run `az automation job list --automation-account-name $AA --resource-group $RG -o table`.
- In the portal, open the runbook job output and confirm the local machine name appears.
- Verify the worker group shows the target server as **healthy**.

### Cleanup

#### Azure CLI
```bash
az automation schedule delete --automation-account-name $AA --resource-group $RG --name daily-8am --yes
az automation runbook delete --automation-account-name $AA --resource-group $RG --name $RUNBOOK --yes
az group delete --name $RG --yes --no-wait
```

#### PowerShell
```powershell
Unregister-AzAutomationScheduledRunbook -AutomationAccountName $AutomationAccount -ResourceGroupName $RG -RunbookName $RunbookName -ScheduleName "daily-8am" -Force
Remove-AzAutomationRunbook -AutomationAccountName $AutomationAccount -ResourceGroupName $RG -Name $RunbookName -Force
Remove-AzResourceGroup -Name $RG -Force -AsJob
Remove-Item -Path .\runbook.ps1 -Force -ErrorAction SilentlyContinue
```

### Exam Tip
If Azure Automation must reach domain controllers, file shares, or internal-only endpoints, use a Hybrid Runbook Worker instead of a cloud-only runbook.

### Exam-Style Review Questions
1. Why would you choose a Hybrid Runbook Worker over an Azure sandbox runbook?
2. What role does Arc play in the newer Hybrid Worker deployment model?
3. What design concern should you review before scheduling runbooks against on-prem resources?

---

## Lab 5: Azure Site Recovery for Hybrid DR

### Objective
Create a Recovery Services vault, define a replication policy, enable replication for a simulated Azure VM representing an on-premises workload, run a test failover, and monitor replication health.

**When to Use This:** Use Azure Site Recovery when workloads must remain recoverable during site outages and Azure acts as the secondary recovery location.

**Key AZ-305 Concepts:** RPO/RTO planning, replication policy selection, test failover versus planned failover, vault design, and health monitoring.

### Exam Domain Mapping
- **Primary:** Design business continuity solutions (15-20%)
- **Secondary:** Design infrastructure solutions (30-35%)

### Prerequisites
- Existing Azure VM to use as an Azure-to-Azure simulation, or an on-premises VM in a supported replication scenario
- Contributor or Site Recovery Contributor permissions
- Recovery Services provider registration
- Separate target virtual network for failover testing

### Architecture and Design Rationale
ASR is the right answer when business continuity requires orchestrated recovery rather than simple backup restoration. For AZ-305, always align the design to explicit RTO and RPO values, because backup and ASR solve different problems.

### Implementation Steps
1. Create a Recovery Services vault.
2. Configure vault context and replication policy.
3. Enable replication for the source VM.
4. Run a test failover into an isolated network.
5. Review replication health and recovery point status.

### Full CLI + PowerShell Commands

#### Azure CLI
```bash
RG="rg-hybrid-asr"
LOCATION="eastus"
VAULT="rsv-hybrid-$RANDOM"
TARGET_VNET="vnet-asr-dr"
SOURCE_VM="vm-app-01"
SUB_ID=$(az account show --query id -o tsv)

az group create --name $RG --location $LOCATION
az provider register --namespace Microsoft.RecoveryServices
az recoveryservices vault create --name $VAULT --resource-group $RG --location $LOCATION
az recoveryservices vault backup-properties set --name $VAULT --resource-group $RG --soft-delete-feature-state Enable
az network vnet create --resource-group $RG --name $TARGET_VNET --location $LOCATION --address-prefix 10.60.0.0/16 --subnet-name dr-subnet --subnet-prefix 10.60.1.0/24

# Native Azure CLI coverage for ASR replication is limited; use az rest for policy inspection and PowerShell for full protection workflows.
az rest --method get \
  --url "https://management.azure.com/subscriptions/$SUB_ID/resourceGroups/$RG/providers/Microsoft.RecoveryServices/vaults/$VAULT/replicationPolicies?api-version=2023-04-01"

az recoveryservices vault show --name $VAULT --resource-group $RG -o table
```

#### PowerShell
```powershell
$RG = "rg-hybrid-asr"
$Location = "eastus"
$VaultName = "rsv-hybrid-$(Get-Random)"
$SourceVmName = "vm-app-01"
$TargetVNetName = "vnet-asr-dr"

New-AzResourceGroup -Name $RG -Location $Location
$vault = New-AzRecoveryServicesVault -Name $VaultName -ResourceGroupName $RG -Location $Location
Set-AzRecoveryServicesAsrVaultContext -Vault $vault

New-AzVirtualNetwork -Name $TargetVNetName -ResourceGroupName $RG -Location $Location -AddressPrefix "10.60.0.0/16" -Subnet (New-AzVirtualNetworkSubnetConfig -Name "dr-subnet" -AddressPrefix "10.60.1.0/24")

$policy = New-AzRecoveryServicesAsrPolicy -Name "A2A-30MinPolicy" -ReplicationProvider "A2A" -RecoveryPointRetentionInHours 24 -ApplicationConsistentSnapshotFrequencyInHours 4 -ReplicationInterval 30
$sourceVm = Get-AzVM -Name $SourceVmName -ResourceGroupName "rg-source-workloads"

# Discovery and protectable item retrieval can take time in a real environment.
# Use this block after the source fabric and container mappings exist.
$vaultId = $vault.ID
Write-Host "Vault created: $vaultId"
Write-Host "Create or validate the A2A fabric mappings in portal if this is the first ASR deployment in the region."
Write-Host "Then enable replication for $SourceVmName using Recovery Services Vault > Site Recovery."

# Example health query
Get-AzRecoveryServicesAsrJob | Select-Object Name, State, TargetObjectName, StartTime | Format-Table
```

### Validation and Success Criteria
- The Recovery Services vault is deployed.
- Replication policy exists.
- The VM shows replication health and recovery points.
- Test failover completes into the isolated DR network.

### Verification Steps
- In the vault, open **Site Recovery Infrastructure** and confirm fabrics and policies.
- Confirm the protected item shows a healthy replication status.
- Run a **test failover** and verify the recovered VM boots in the target VNet.
- Review ASR jobs and alerts after the test.

### Cleanup

#### Azure CLI
```bash
az group delete --name $RG --yes --no-wait
```

#### PowerShell
```powershell
Remove-AzResourceGroup -Name $RG -Force -AsJob
```

### Exam Tip
Backup protects data. Site Recovery protects service continuity. If the requirement includes low RTO, orchestrated failover, or replica startup in Azure, think ASR rather than backup alone.

### Exam-Style Review Questions
1. Why is ASR a better answer than backup for low-RTO disaster recovery?
2. Why should test failover use an isolated network?
3. What design input must be known before choosing ASR policy settings?

---

## Lab 6: Azure Stack HCI (Conceptual Overview)

### Objective
Review Azure Stack HCI architecture, prerequisites, cluster registration, Azure benefits, and the tradeoffs compared with traditional hyperconverged infrastructure.

**When to Use This:** Use Azure Stack HCI when you need Azure-consistent infrastructure, management, and services while keeping compute local.

**Key AZ-305 Concepts:** Hyperconverged clusters, Azure registration, Arc-based management, AKS on Azure Stack HCI, Azure Virtual Desktop on HCI, and hybrid consistency.

### Exam Domain Mapping
- **Primary:** Design infrastructure solutions (30-35%)
- **Secondary:** Design identity, governance, and monitoring solutions (25-30%)

### Prerequisites
- Access to Windows Admin Center or a lab cluster
- Windows Server knowledge and Hyper-V familiarity
- Azure subscription for registration and Arc integration

### Architecture and Design Rationale
Azure Stack HCI is designed for customers that must keep virtualization and storage local but want Azure control-plane services, billing, and lifecycle benefits. For AZ-305, it is often the right answer for branch, edge, manufacturing, or regulated datacenter scenarios.

### Implementation Steps
1. Review hardware, network, AD DS, and witness prerequisites.
2. Validate cluster readiness.
3. Register the cluster with Azure.
4. Explore Arc-enabled resource projection and Azure benefits.
5. Compare Stack HCI to traditional HCI.

### Full CLI + PowerShell Commands

#### Azure CLI
```bash
RG="rg-stackhci-lab"
LOCATION="eastus"
CLUSTER_NAME="stackhci-cluster01"

az group create --name $RG --location $LOCATION
az provider register --namespace Microsoft.AzureStackHCI

# Use Azure CLI mainly for resource inspection after registration.
az resource list --resource-group $RG --resource-type Microsoft.AzureStackHCI/clusters -o table
az resource show --resource-group $RG --resource-type Microsoft.AzureStackHCI/clusters --name $CLUSTER_NAME
```

#### PowerShell
```powershell
# Run on a candidate Azure Stack HCI cluster.
Test-Cluster
Get-WindowsFeature -Name Failover-Clustering,Hyper-V,Data-Center-Bridging

# After deploying the cluster, register it with Azure.
Register-AzStackHCI -SubscriptionId (Get-AzContext).Subscription.Id -Region eastus -ComputerName stackhci-node01
Get-AzStackHciCluster | Format-Table Name, ProvisioningState, Location
```

### Validation and Success Criteria
- You can explain the control-plane relationship between Azure Stack HCI and Azure.
- You can identify prerequisites for networking, identity, and witness design.
- You can distinguish Stack HCI from a generic Hyper-V cluster.

### Verification Steps
- Confirm the cluster appears in Azure after registration.
- Review whether AKS on HCI, Arc VMs, and Azure Virtual Desktop are relevant to the scenario.
- Compare operational benefits such as centralized updates, Azure billing alignment, and Arc governance.

### Cleanup

#### Azure CLI
```bash
az group delete --name $RG --yes --no-wait
```

#### PowerShell
```powershell
Unregister-AzStackHCI -ComputerName stackhci-node01 -Confirm:$false
```

### Exam Tip
If the requirement says “run locally, keep virtualization on-premises, but use Azure-consistent management and services,” Azure Stack HCI is usually stronger than recommending a plain Hyper-V or VMware cluster.

### Exam-Style Review Questions
1. What makes Azure Stack HCI different from traditional hyperconverged infrastructure?
2. When would Azure Stack HCI be preferable to migrating directly to Azure IaaS?
3. Why is Arc important in the Azure Stack HCI story?

---

## Lab 7: Azure Migrate Assessment

### Objective
Create an Azure Migrate project, review appliance deployment requirements, run discovery, create a migration assessment, and evaluate right-sizing and TCO recommendations.

**When to Use This:** Use Azure Migrate when planning hybrid-to-cloud migration waves and you need evidence-based readiness and cost recommendations before moving workloads.

**Key AZ-305 Concepts:** Discovery, dependency mapping, readiness analysis, performance-based sizing, migration waves, and cost/TCO planning.

### Exam Domain Mapping
- **Primary:** Design infrastructure solutions (30-35%)
- **Secondary:** Design business continuity solutions (15-20%)

### Prerequisites
- Azure subscription with Contributor access
- VMware, Hyper-V, physical, or simulated server inventory
- Azure Migrate project permissions
- Appliance deployment media or plan

### Architecture and Design Rationale
Azure Migrate is typically the first step in a migration program because it reduces guesswork. For AZ-305, it is often the best answer when the question emphasizes discovery, dependency analysis, right-sizing, or phased migration planning.

### Implementation Steps
1. Create an Azure Migrate project.
2. Review appliance prerequisites and deployment model.
3. Start discovery.
4. Create a VM assessment.
5. Review readiness, sizing, and cost outputs.
6. Group workloads into migration waves.

### Full CLI + PowerShell Commands

#### Azure CLI
```bash
RG="rg-hybrid-migrate"
LOCATION="eastus"
PROJECT_NAME="migrate-hybrid-$RANDOM"

az extension add --name migrate --upgrade
az group create --name $RG --location $LOCATION
az migrate project create --name $PROJECT_NAME --resource-group $RG --location $LOCATION

# Discovery appliance deployment is typically prepared from the portal.
# Use the project to download the appliance key and register the appliance.

echo "Portal path: Azure Migrate > Servers, databases and web apps > Discover"

echo "After discovery completes, create a performance-based assessment and review readiness, cost, and dependency output."

az resource list --resource-group $RG --resource-type Microsoft.Migrate/assessmentProjects -o table
```

#### PowerShell
```powershell
$RG = "rg-hybrid-migrate"
$Location = "eastus"
$ProjectName = "migrate-hybrid-$(Get-Random)"

New-AzResourceGroup -Name $RG -Location $Location
New-AzMigrateProject -Name $ProjectName -ResourceGroupName $RG -Location $Location

Write-Host "Deploy the Azure Migrate appliance from the portal for VMware, Hyper-V, or physical servers."
Write-Host "After discovery, create an assessment and review: readiness, right-sizing, Azure Hybrid Benefit, and monthly cost."
```

### Validation and Success Criteria
- Azure Migrate project exists.
- Discovery appliance is registered.
- Assessment output shows readiness and SKU recommendations.
- Workloads are grouped into migration waves.

### Verification Steps
- Open the project in the portal and confirm the discovered server count.
- Review the assessment for readiness state: Ready, Conditionally Ready, Not Ready.
- Review dependency mapping before defining migration waves.
- Validate whether performance-based sizing reduces projected costs.

### Cleanup

#### Azure CLI
```bash
az migrate project delete --name $PROJECT_NAME --resource-group $RG --yes
az group delete --name $RG --yes --no-wait
```

#### PowerShell
```powershell
Remove-AzMigrateProject -Name $ProjectName -ResourceGroupName $RG
Remove-AzResourceGroup -Name $RG -Force -AsJob
```

### Exam Tip
If the question is about migration planning, the best first answer is often Azure Migrate, not a direct recommendation to rehost or refactor.

### Exam-Style Review Questions
1. Why should you perform discovery and assessment before choosing a migration tool?
2. When is performance-based sizing better than “as on-premises” sizing?
3. Why is dependency mapping important when planning migration waves?

---

## Lab 8: Hybrid Identity Integration

### Objective
Review a Microsoft Entra Connect deployment, configure Password Hash Sync and Seamless SSO, test hybrid authentication, and validate synchronization health.

**When to Use This:** Use hybrid identity integration when users and devices must access both on-premises and Azure resources with a unified identity.

**Key AZ-305 Concepts:** Microsoft Entra Connect, Password Hash Sync, Seamless SSO, synchronization health, hybrid identity operations, and identity as a core hybrid dependency.

### Exam Domain Mapping
- **Primary:** Design identity, governance, and monitoring solutions (25-30%)
- **Secondary:** Design infrastructure solutions (30-35%)

### Prerequisites
- On-premises Active Directory or a simulated domain controller VM
- Microsoft Entra tenant with Global Administrator access
- A dedicated Microsoft Entra Connect server
- Azure CLI and Azure PowerShell
- Domain admin access to configure AD DS

### Architecture and Design Rationale
For most hybrid identity scenarios on AZ-305, Password Hash Sync plus Seamless SSO is the simplest recommended pattern unless the scenario explicitly requires federation or passthrough authentication.

### Implementation Steps
1. Deploy or identify a domain controller and an Entra Connect server.
2. Configure AD DS and DNS.
3. Install Microsoft Entra Connect.
4. Enable Password Hash Sync.
5. Enable Seamless SSO.
6. Test sign-in and review sync health.

### Full CLI + PowerShell Commands

#### Azure CLI
```bash
RG="rg-hybrid-identity"
LOCATION="eastus"
VNET="vnet-hybrid-identity"
DC_VM="vm-dc01"
SYNC_VM="vm-sync01"

az group create --name $RG --location $LOCATION
az network vnet create \
  --resource-group $RG \
  --name $VNET \
  --location $LOCATION \
  --address-prefix 10.70.0.0/16 \
  --subnet-name identity-subnet \
  --subnet-prefix 10.70.1.0/24

az vm create \
  --resource-group $RG \
  --name $DC_VM \
  --image Win2022Datacenter \
  --size Standard_D2s_v5 \
  --admin-username azureadmin \
  --admin-password 'P@ssw0rd123!Complex' \
  --vnet-name $VNET \
  --subnet identity-subnet

az vm create \
  --resource-group $RG \
  --name $SYNC_VM \
  --image Win2022Datacenter \
  --size Standard_D2s_v5 \
  --admin-username azureadmin \
  --admin-password 'P@ssw0rd123!Complex' \
  --vnet-name $VNET \
  --subnet identity-subnet

DC_IP=$(az vm show -d -g $RG -n $DC_VM --query privateIps -o tsv)
az network vnet update --resource-group $RG --name $VNET --dns-servers $DC_IP

echo "Complete Microsoft Entra Connect setup on $SYNC_VM using the wizard: select Password Hash Sync and enable Seamless SSO."
```

#### PowerShell
```powershell
$RG = "rg-hybrid-identity"
$Location = "eastus"
$DomainName = "contoso.local"
$SafeMode = ConvertTo-SecureString "P@ssw0rd123!Safe" -AsPlainText -Force

Install-WindowsFeature AD-Domain-Services -IncludeManagementTools
Import-Module ADDSDeployment
Install-ADDSForest `
  -DomainName $DomainName `
  -DomainNetbiosName "CONTOSO" `
  -InstallDns `
  -SafeModeAdministratorPassword $SafeMode `
  -Force

# On the Entra Connect server after installation.
Import-Module ADSync
Get-ADSyncScheduler
Start-ADSyncSyncCycle -PolicyType Delta

# Sync health checks
Get-ADSyncConnectorRunStatus
Get-ADSyncScheduler | Format-List *
```

### Validation and Success Criteria
- Users from on-prem AD synchronize to Microsoft Entra ID.
- Password Hash Sync is enabled.
- Seamless SSO is enabled for domain-joined clients.
- Sync runs are healthy and monitored.

### Verification Steps
- Sign in to a cloud application with a synchronized user.
- Confirm the user object shows **Synced from on-premises AD** in Entra admin center.
- Review the Entra Connect Health dashboard or local scheduler output.
- Trigger a delta sync and confirm it succeeds.

### Cleanup

#### Azure CLI
```bash
az group delete --name $RG --yes --no-wait
```

#### PowerShell
```powershell
# If this is a disposable lab domain, decommission the VMs and remove the lab resource group.
Remove-AzResourceGroup -Name $RG -Force -AsJob
```

### Exam Tip
If the requirement is “simple hybrid identity with minimal infrastructure,” Password Hash Sync plus Seamless SSO is usually better than federation unless the scenario explicitly needs federation features.

### Exam-Style Review Questions
1. Why is Password Hash Sync often the recommended default for hybrid identity?
2. When would federation be required instead of Password Hash Sync?
3. What problem does Seamless SSO solve for domain-joined users?

---

## Lab 9: Hybrid Decision Exercise

### Objective
Practice exam-style hybrid architecture decisions by mapping scenarios to services, connectivity models, and operational approaches.

**When to Use This:** Use this lab for final AZ-305 review when you need to translate requirements into hybrid architecture decisions quickly.

**Key AZ-305 Concepts:** Service selection, connectivity choices, management plane consistency, DR tradeoffs, migration sequencing, and workload placement.

### Exam Domain Mapping
- **Primary:** Design infrastructure solutions (30-35%)
- **Secondary:** Identity, governance, monitoring; business continuity

### Prerequisites
- Familiarity with Arc, DNS, Automation, ASR, Azure Migrate, and hybrid identity
- Access to the cheat sheet and the earlier labs in this file

### Architecture and Design Rationale
AZ-305 hybrid questions are usually won by translating business constraints into the correct service combination. The goal is not to memorize product names only, but to map each scenario to connectivity, management, governance, and resilience requirements.

### Implementation Steps
1. Read each scenario.
2. Identify the service stack.
3. Choose connectivity.
4. Choose governance and monitoring approach.
5. Compare your answer with the key.

### Full CLI + PowerShell Commands

#### Azure CLI
```bash
# Quick command references you should recognize during exam prep.
az connectedmachine list -o table
az connectedk8s list -o table
az network private-dns zone list -o table
az automation account list -o table
az recoveryservices vault list -o table
az migrate project list -o table
```

#### PowerShell
```powershell
Get-AzConnectedMachine
Get-AzOperationalInsightsWorkspace
Get-AzAutomationAccount
Get-AzRecoveryServicesVault
Get-AzMigrateProject
```

### Scenario Set

| # | Scenario |
|---|----------|
| 1 | A factory keeps latency-sensitive Windows servers on-premises but wants Azure Policy, Defender, and Update Manager. |
| 2 | An enterprise runs Kubernetes in AWS and on-premises and wants one GitOps model and one compliance view. |
| 3 | A company uses private endpoints in Azure and must resolve them from branch offices through existing corporate DNS. |
| 4 | Security automation must run inside a private datacenter because the target systems are not internet reachable from Azure sandboxes. |
| 5 | A critical app runs on-premises and must fail over to Azure within 30 minutes during a site outage. |
| 6 | A retailer wants Azure-consistent virtualization and AKS at the edge without moving store workloads to Azure public regions. |
| 7 | A migration program needs readiness, dependency mapping, and right-sizing before moving 400 VMs. |
| 8 | Users must keep one identity across on-prem AD and Microsoft 365 with minimal sign-in friction. |

### Answer Key with Reasoning

| Scenario | Best Services | Connectivity | Management Approach | Reasoning |
|----------|---------------|--------------|---------------------|-----------|
| 1 | Azure Arc-enabled Servers + Azure Policy + Defender for Cloud + Update Manager | Internet or private egress to Azure | Azure control plane manages on-prem servers | Workloads stay local, but governance and operations move to Azure. |
| 2 | Arc-enabled Kubernetes + Flux GitOps + Azure Policy + Container Insights | Existing cluster connectivity to Azure | Centralized multi-cluster governance | Best fit for non-Azure Kubernetes requiring consistency without AKS migration. |
| 3 | Azure Private DNS + Azure DNS Private Resolver + conditional forwarding | VPN or ExpressRoute | Hub-based DNS resolution | Private endpoint DNS across hybrid networks is the key requirement. |
| 4 | Azure Automation + Hybrid Runbook Worker on Arc server | Outbound to Azure Automation | Azure orchestration, local execution | Runbooks need local reachability to private resources. |
| 5 | Azure Site Recovery + Recovery Services Vault | VPN or ExpressRoute preferred for steady state; Azure network for failover | Recovery orchestration from Azure | Low-RTO failover is a DR requirement, not just backup. |
| 6 | Azure Stack HCI + Arc + optional AKS on HCI/AVD | Local datacenter or edge | Azure-consistent local platform | The scenario requires local compute with Azure consistency. |
| 7 | Azure Migrate + appliance + assessment + dependency mapping | Appliance connectivity to source estate and Azure | Migration program planning from Azure | This is discovery and planning first, migration second. |
| 8 | Microsoft Entra Connect + Password Hash Sync + Seamless SSO | Standard hybrid identity connectivity | Centralized cloud identity with synced on-prem source | Simplest common hybrid identity pattern for most enterprises. |

### Validation and Success Criteria
- You can justify each design choice in one sentence.
- You can separate management, connectivity, and workload-placement decisions.
- You can explain why at least one alternative answer is weaker for each scenario.

### Verification Steps
- Cover the answer key and try to answer all eight scenarios from memory.
- For each scenario, identify the trigger words that point to the recommended service.
- Revisit any scenario where you confused backup with DR, Arc with migration, or AKS with Arc-enabled Kubernetes.

### Cleanup

#### Azure CLI
```bash
# No dedicated cleanup unless you deployed referenced services during practice.
```

#### PowerShell
```powershell
# No dedicated cleanup unless you deployed referenced services during practice.
```

### Exam Tip
Hybrid questions often test whether you can distinguish workload placement from management-plane design. Arc usually means “manage it where it is.” Migrate means “plan or move it.” ASR means “recover it.”

### Exam-Style Review Questions
1. What wording in a scenario usually indicates Arc instead of migration?
2. What wording points to ASR instead of backup?
3. What wording suggests Azure DNS Private Resolver instead of custom DNS VMs?

---

## Hybrid Decision Summary Table

| Requirement Pattern | Best-Fit Azure Service(s) | Why |
|---------------------|---------------------------|-----|
| Manage non-Azure Windows/Linux servers from Azure | Azure Arc-enabled Servers | Extends Azure governance, monitoring, and patching to servers anywhere |
| Govern Kubernetes clusters across on-prem and other clouds | Arc-enabled Kubernetes + Flux + Policy | Centralized GitOps and policy without relocating clusters |
| Resolve private Azure names from on-premises networks | Azure Private DNS + Azure DNS Private Resolver | Native hybrid DNS resolution for private endpoints and internal zones |
| Run automation near private resources | Azure Automation Hybrid Runbook Worker | Azure control plane with local execution |
| Recover hybrid workloads into Azure | Azure Site Recovery | Low-RTO failover and recovery orchestration |
| Keep compute local with Azure-consistent operations | Azure Stack HCI | Local infrastructure with Azure integration |
| Assess and wave-plan migrations | Azure Migrate | Discovery, dependency mapping, readiness, and right-sizing |
| Extend on-prem identity to Azure | Microsoft Entra Connect + PHS + Seamless SSO | Simple and common hybrid identity model |
| Need centralized policy, monitoring, and inventory but cannot move workloads | Azure Arc family | Consistent management is the main design goal |
| Need a phased hybrid modernization path | Arc + Migrate + ASR as needed | Combines management, migration planning, and DR |
