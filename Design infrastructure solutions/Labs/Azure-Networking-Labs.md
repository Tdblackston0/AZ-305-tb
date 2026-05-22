# Azure Networking Hands-On Labs (AZ-305)

> 📖 **Cheat Sheet:** [Azure Networking](../Azure-Networking.md)

> **Primary exam domain:** Design infrastructure solutions (30-35%)  
> **Secondary domains:** Design identity, governance, and monitoring solutions (25-30%)  
> **Tools used:** Azure CLI, Azure PowerShell, Azure Network Watcher, Azure Monitor, Log Analytics, Bicep, Terraform  
> **Important:** Use a non-production subscription. Networking labs can create billable resources such as VPN gateways, Bastion, Azure Firewall, Application Gateway, and Front Door.

---

## Common prerequisites for all labs

- Azure subscription with **Contributor** rights on a sandbox resource group or subscription
- Azure CLI logged in with `az login`
- Azure PowerShell logged in with `Connect-AzAccount`
- Latest modules/extensions where needed:
  - `az extension add --name front-door`
  - `Install-Module Az -Scope CurrentUser`
- Required providers registered:
  - `Microsoft.Network`
  - `Microsoft.Compute`
  - `Microsoft.OperationalInsights`
  - `Microsoft.Insights`
  - `Microsoft.Cdn`
- A naming convention such as `rg-net-lab01`, `vnet-hub-lab01`, and `eastus`

### One-time provider registration

#### Azure CLI
```bash
az provider register --namespace Microsoft.Network
az provider register --namespace Microsoft.Compute
az provider register --namespace Microsoft.OperationalInsights
az provider register --namespace Microsoft.Insights
az provider register --namespace Microsoft.Cdn
```

#### PowerShell
```powershell
Register-AzResourceProvider -ProviderNamespace Microsoft.Network
Register-AzResourceProvider -ProviderNamespace Microsoft.Compute
Register-AzResourceProvider -ProviderNamespace Microsoft.OperationalInsights
Register-AzResourceProvider -ProviderNamespace Microsoft.Insights
Register-AzResourceProvider -ProviderNamespace Microsoft.Cdn
```

---

## Lab 1: Hub-Spoke Network Topology

### Objective
Create a hub virtual network with shared-services subnets, deploy two spoke VNets, configure hub-spoke peering, enable gateway transit settings, and prepare spoke-to-spoke routing through the hub.

### When to Use This
Use hub-spoke when you need enterprise segmentation, centralized inspection, shared connectivity, and delegated workload VNets without giving every application team a flat network.

### Exam domain mapping
- **Primary:** Design infrastructure solutions (30-35%)
- **Secondary:** Design identity, governance, and monitoring solutions (25-30%)

### Key AZ-305 Concepts
- Hub-spoke versus mesh topology
- VNet peering is **non-transitive**
- Gateway transit for shared VPN/ExpressRoute gateways
- User-defined routes (UDRs) for centralized inspection
- Shared services in the hub: Bastion, Firewall, VPN/ER gateway

### Prerequisites
- An Azure region that supports Azure Bastion, VPN Gateway, and Azure Firewall
- Optional: an Azure Firewall or NVA private IP if you want to test spoke-to-spoke inspection immediately

### Architecture and design rationale
The hub contains shared connectivity and security services. Spokes host application workloads. For AZ-305, remember that **peering alone does not create transitive spoke-to-spoke connectivity**. If workloads must communicate through centralized inspection, use a route table that sends inter-spoke traffic to the Azure Firewall or NVA in the hub. Enable gateway transit when one hub gateway should be reused by multiple spokes.

### Implementation steps
1. Create a resource group.
2. Create the hub VNet with `GatewaySubnet`, `AzureFirewallSubnet`, `AzureBastionSubnet`, and `SharedServices`.
3. Create two spoke VNets.
4. Peer hub to each spoke and each spoke back to the hub.
5. Enable gateway transit on hub peerings and remote gateway use on spoke peerings.
6. Create route tables that send inter-spoke traffic to a central next hop in the hub.
7. Validate peering state and effective routes.

### Full CLI + PowerShell commands

#### Azure CLI
```bash
RG="rg-net-lab01"
LOCATION="eastus"
HUB_VNET="vnet-hub-lab01"
SPOKE1_VNET="vnet-spoke1-lab01"
SPOKE2_VNET="vnet-spoke2-lab01"
HUB_NVA_IP="10.0.1.4"   # Replace with Azure Firewall or NVA private IP

az group create --name $RG --location $LOCATION

az network vnet create -g $RG -n $HUB_VNET --location $LOCATION --address-prefix 10.0.0.0/16 \
  --subnet-name SharedServices --subnet-prefix 10.0.1.0/24
az network vnet subnet create -g $RG --vnet-name $HUB_VNET -n GatewaySubnet --address-prefixes 10.0.255.0/27
az network vnet subnet create -g $RG --vnet-name $HUB_VNET -n AzureFirewallSubnet --address-prefixes 10.0.254.0/26
az network vnet subnet create -g $RG --vnet-name $HUB_VNET -n AzureBastionSubnet --address-prefixes 10.0.253.0/26

az network vnet create -g $RG -n $SPOKE1_VNET --location $LOCATION --address-prefix 10.1.0.0/16 \
  --subnet-name AppSubnet --subnet-prefix 10.1.1.0/24
az network vnet create -g $RG -n $SPOKE2_VNET --location $LOCATION --address-prefix 10.2.0.0/16 \
  --subnet-name AppSubnet --subnet-prefix 10.2.1.0/24

az network vnet peering create -g $RG -n hub-to-spoke1 --vnet-name $HUB_VNET \
  --remote-vnet $SPOKE1_VNET --allow-vnet-access --allow-forwarded-traffic --allow-gateway-transit
az network vnet peering create -g $RG -n spoke1-to-hub --vnet-name $SPOKE1_VNET \
  --remote-vnet $HUB_VNET --allow-vnet-access --allow-forwarded-traffic --use-remote-gateways
az network vnet peering create -g $RG -n hub-to-spoke2 --vnet-name $HUB_VNET \
  --remote-vnet $SPOKE2_VNET --allow-vnet-access --allow-forwarded-traffic --allow-gateway-transit
az network vnet peering create -g $RG -n spoke2-to-hub --vnet-name $SPOKE2_VNET \
  --remote-vnet $HUB_VNET --allow-vnet-access --allow-forwarded-traffic --use-remote-gateways

az network route-table create -g $RG -n rt-spoke1
az network route-table route create -g $RG --route-table-name rt-spoke1 -n to-spoke2 \
  --address-prefix 10.2.0.0/16 --next-hop-type VirtualAppliance --next-hop-ip-address $HUB_NVA_IP
az network route-table create -g $RG -n rt-spoke2
az network route-table route create -g $RG --route-table-name rt-spoke2 -n to-spoke1 \
  --address-prefix 10.1.0.0/16 --next-hop-type VirtualAppliance --next-hop-ip-address $HUB_NVA_IP

az network vnet subnet update -g $RG --vnet-name $SPOKE1_VNET -n AppSubnet --route-table rt-spoke1
az network vnet subnet update -g $RG --vnet-name $SPOKE2_VNET -n AppSubnet --route-table rt-spoke2

az network vnet peering list -g $RG --vnet-name $HUB_VNET -o table
az network nic show-effective-route-table -g $RG -n <spoke-vm-nic-name>
```

#### PowerShell
```powershell
$RG = "rg-net-lab01"
$Location = "eastus"
$HubVnetName = "vnet-hub-lab01"
$Spoke1VnetName = "vnet-spoke1-lab01"
$Spoke2VnetName = "vnet-spoke2-lab01"
$HubNvaIp = "10.0.1.4"   # Replace with Azure Firewall or NVA private IP

New-AzResourceGroup -Name $RG -Location $Location

$hubSubnets = @(
  New-AzVirtualNetworkSubnetConfig -Name "SharedServices" -AddressPrefix "10.0.1.0/24"
  New-AzVirtualNetworkSubnetConfig -Name "GatewaySubnet" -AddressPrefix "10.0.255.0/27"
  New-AzVirtualNetworkSubnetConfig -Name "AzureFirewallSubnet" -AddressPrefix "10.0.254.0/26"
  New-AzVirtualNetworkSubnetConfig -Name "AzureBastionSubnet" -AddressPrefix "10.0.253.0/26"
)
$hubVnet = New-AzVirtualNetwork -ResourceGroupName $RG -Location $Location -Name $HubVnetName -AddressPrefix "10.0.0.0/16" -Subnet $hubSubnets
$spoke1Vnet = New-AzVirtualNetwork -ResourceGroupName $RG -Location $Location -Name $Spoke1VnetName -AddressPrefix "10.1.0.0/16" -Subnet (New-AzVirtualNetworkSubnetConfig -Name "AppSubnet" -AddressPrefix "10.1.1.0/24")
$spoke2Vnet = New-AzVirtualNetwork -ResourceGroupName $RG -Location $Location -Name $Spoke2VnetName -AddressPrefix "10.2.0.0/16" -Subnet (New-AzVirtualNetworkSubnetConfig -Name "AppSubnet" -AddressPrefix "10.2.1.0/24")

Add-AzVirtualNetworkPeering -Name "hub-to-spoke1" -VirtualNetwork $hubVnet -RemoteVirtualNetworkId $spoke1Vnet.Id -AllowVirtualNetworkAccess -AllowForwardedTraffic -AllowGatewayTransit
Add-AzVirtualNetworkPeering -Name "spoke1-to-hub" -VirtualNetwork $spoke1Vnet -RemoteVirtualNetworkId $hubVnet.Id -AllowVirtualNetworkAccess -AllowForwardedTraffic -UseRemoteGateways
Add-AzVirtualNetworkPeering -Name "hub-to-spoke2" -VirtualNetwork $hubVnet -RemoteVirtualNetworkId $spoke2Vnet.Id -AllowVirtualNetworkAccess -AllowForwardedTraffic -AllowGatewayTransit
Add-AzVirtualNetworkPeering -Name "spoke2-to-hub" -VirtualNetwork $spoke2Vnet -RemoteVirtualNetworkId $hubVnet.Id -AllowVirtualNetworkAccess -AllowForwardedTraffic -UseRemoteGateways

$route1 = New-AzRouteConfig -Name "to-spoke2" -AddressPrefix "10.2.0.0/16" -NextHopType VirtualAppliance -NextHopIpAddress $HubNvaIp
$route2 = New-AzRouteConfig -Name "to-spoke1" -AddressPrefix "10.1.0.0/16" -NextHopType VirtualAppliance -NextHopIpAddress $HubNvaIp
$rt1 = New-AzRouteTable -Name "rt-spoke1" -ResourceGroupName $RG -Location $Location -Route $route1
$rt2 = New-AzRouteTable -Name "rt-spoke2" -ResourceGroupName $RG -Location $Location -Route $route2

Set-AzVirtualNetworkSubnetConfig -VirtualNetwork $spoke1Vnet -Name "AppSubnet" -AddressPrefix "10.1.1.0/24" -RouteTable $rt1 | Set-AzVirtualNetwork
Set-AzVirtualNetworkSubnetConfig -VirtualNetwork $spoke2Vnet -Name "AppSubnet" -AddressPrefix "10.2.1.0/24" -RouteTable $rt2 | Set-AzVirtualNetwork

Get-AzVirtualNetworkPeering -VirtualNetworkName $HubVnetName -ResourceGroupName $RG | Format-Table Name, PeeringState, AllowGatewayTransit
```

### IaC implementation

#### Bicep
```bicep
param location string = resourceGroup().location
param hubName string = 'vnet-hub-lab01'
param spoke1Name string = 'vnet-spoke1-lab01'
param spoke2Name string = 'vnet-spoke2-lab01'

resource hub 'Microsoft.Network/virtualNetworks@2023-11-01' = {
  name: hubName
  location: location
  properties: {
    addressSpace: { addressPrefixes: ['10.0.0.0/16'] }
    subnets: [
      { name: 'SharedServices' properties: { addressPrefix: '10.0.1.0/24' } }
      { name: 'GatewaySubnet' properties: { addressPrefix: '10.0.255.0/27' } }
      { name: 'AzureFirewallSubnet' properties: { addressPrefix: '10.0.254.0/26' } }
      { name: 'AzureBastionSubnet' properties: { addressPrefix: '10.0.253.0/26' } }
    ]
  }
}

resource spoke1 'Microsoft.Network/virtualNetworks@2023-11-01' = {
  name: spoke1Name
  location: location
  properties: {
    addressSpace: { addressPrefixes: ['10.1.0.0/16'] }
    subnets: [{ name: 'AppSubnet' properties: { addressPrefix: '10.1.1.0/24' } }]
  }
}

resource spoke2 'Microsoft.Network/virtualNetworks@2023-11-01' = {
  name: spoke2Name
  location: location
  properties: {
    addressSpace: { addressPrefixes: ['10.2.0.0/16'] }
    subnets: [{ name: 'AppSubnet' properties: { addressPrefix: '10.2.1.0/24' } }]
  }
}
```

#### Terraform
```hcl
resource "azurerm_virtual_network" "hub" {
  name                = "vnet-hub-lab01"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  address_space       = ["10.0.0.0/16"]
}

resource "azurerm_subnet" "hub_gateway" {
  name                 = "GatewaySubnet"
  resource_group_name  = azurerm_resource_group.rg.name
  virtual_network_name = azurerm_virtual_network.hub.name
  address_prefixes     = ["10.0.255.0/27"]
}

resource "azurerm_virtual_network" "spoke1" {
  name                = "vnet-spoke1-lab01"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  address_space       = ["10.1.0.0/16"]
}

resource "azurerm_virtual_network_peering" "hub_to_spoke1" {
  name                         = "hub-to-spoke1"
  resource_group_name          = azurerm_resource_group.rg.name
  virtual_network_name         = azurerm_virtual_network.hub.name
  remote_virtual_network_id    = azurerm_virtual_network.spoke1.id
  allow_virtual_network_access = true
  allow_forwarded_traffic      = true
  allow_gateway_transit        = true
}
```

### Validation and success criteria
- Hub and both spokes show **Connected** peering state.
- Spoke peerings show **UseRemoteGateways = True** when configured.
- Effective routes on a spoke VM include the UDR pointing to the hub appliance IP for the opposite spoke CIDR.
- You can explain why direct spoke-to-spoke communication does **not** happen automatically.

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
If the question says **centralized inspection**, **shared connectivity**, or **enterprise landing zone**, choose **hub-spoke**. If it says **spoke-to-spoke via peering only**, that is an exam trap because VNet peering is not transitive.

---

## Lab 2: Network Security Groups & Application Security Groups

### Objective
Create NSGs and ASGs, associate controls at subnet and NIC scope, validate traffic decisions, and enable NSG flow logs with Traffic Analytics.

### When to Use This
Use NSGs and ASGs when you need micro-segmentation inside a VNet without managing large numbers of IP-based rules.

### Exam domain mapping
- **Primary:** Design infrastructure solutions (30-35%)
- **Secondary:** Design identity, governance, and monitoring solutions (25-30%)

### Key AZ-305 Concepts
- NSG rule priority and default rules
- ASG-based rule targeting for dynamic workloads
- Subnet versus NIC associations
- Flow logging and Traffic Analytics for visibility
- East-west traffic control within a VNet

### Prerequisites
- A region with Network Watcher support
- Optional test VMs or NICs to validate rule behavior
- Log Analytics workspace and storage account for Traffic Analytics

### Architecture and design rationale
ASGs reduce operational overhead because rules target application roles instead of specific IPs. Associate the NSG to the subnet for broad policy and to the NIC only for exceptions. For the exam, remember that NSG evaluation is stateful and based on the **five-tuple**. Also note that Traffic Analytics adds operational insight, but production teams should stay aware of Azure guidance as NSG flow log capabilities evolve over time.

### Implementation steps
1. Create a VNet and subnet.
2. Create two NICs and ASGs for web and app tiers.
3. Create an NSG with allow and deny rules.
4. Associate the NSG to the subnet and one NIC.
5. Validate traffic with Network Watcher IP flow verify.
6. Enable flow logs and Traffic Analytics.

### Full CLI + PowerShell commands

#### Azure CLI
```bash
RG="rg-net-lab02"
LOCATION="eastus"
VNET="vnet-sec-lab02"
SUBNET="workload-subnet"
NSG="nsg-app-lab02"
WEB_ASG="asg-web-lab02"
APP_ASG="asg-app-lab02"
LAW="law-netlab02$RANDOM"
STG="stnetlab02$RANDOM"
NIC1="nic-web-lab02"
NIC2="nic-app-lab02"

az group create -n $RG -l $LOCATION
az network watcher configure -g NetworkWatcherRG -l $LOCATION --enabled true
az network vnet create -g $RG -n $VNET -l $LOCATION --address-prefix 10.20.0.0/16 --subnet-name $SUBNET --subnet-prefix 10.20.1.0/24
az network asg create -g $RG -n $WEB_ASG -l $LOCATION
az network asg create -g $RG -n $APP_ASG -l $LOCATION
az network nsg create -g $RG -n $NSG -l $LOCATION

az network nic create -g $RG -n $NIC1 --vnet-name $VNET --subnet $SUBNET --application-security-groups $WEB_ASG
az network nic create -g $RG -n $NIC2 --vnet-name $VNET --subnet $SUBNET --application-security-groups $APP_ASG

az network nsg rule create -g $RG --nsg-name $NSG -n Allow-Web-To-App-443 \
  --priority 100 --direction Inbound --access Allow --protocol Tcp \
  --source-asgs $WEB_ASG --destination-asgs $APP_ASG --destination-port-ranges 443
az network nsg rule create -g $RG --nsg-name $NSG -n Deny-Internet-To-App \
  --priority 110 --direction Inbound --access Deny --protocol Tcp \
  --source-address-prefixes Internet --destination-asgs $APP_ASG --destination-port-ranges 443

az network vnet subnet update -g $RG --vnet-name $VNET -n $SUBNET --network-security-group $NSG
az network nic update -g $RG -n $NIC2 --network-security-group $NSG

az network watcher test-ip-flow -g $RG --direction Inbound --protocol TCP \
  --local 10.20.1.5:443 --remote 10.20.1.4:50000 --nic $NIC2

az monitor log-analytics workspace create -g $RG -n $LAW -l $LOCATION
az storage account create -g $RG -n $STG -l $LOCATION --sku Standard_LRS
LAW_ID=$(az monitor log-analytics workspace show -g $RG -n $LAW --query id -o tsv)
NSG_ID=$(az network nsg show -g $RG -n $NSG --query id -o tsv)
STG_ID=$(az storage account show -g $RG -n $STG --query id -o tsv)

az network watcher flow-log create -g NetworkWatcherRG -l $LOCATION -n fl-nsg-lab02 \
  --nsg $NSG_ID --storage-account $STG_ID --enabled true --traffic-analytics true \
  --workspace $LAW_ID --interval 10
```

#### PowerShell
```powershell
$RG = "rg-net-lab02"
$Location = "eastus"
$VnetName = "vnet-sec-lab02"
$SubnetName = "workload-subnet"
$NsgName = "nsg-app-lab02"
$WebAsgName = "asg-web-lab02"
$AppAsgName = "asg-app-lab02"
$LawName = "law-netlab02$(Get-Random)"
$StorageName = "stnetlab02$(Get-Random -Maximum 9999)"

New-AzResourceGroup -Name $RG -Location $Location
New-AzNetworkWatcher -Name "NetworkWatcher_$Location" -ResourceGroupName "NetworkWatcherRG" -Location $Location -ErrorAction SilentlyContinue

$subnet = New-AzVirtualNetworkSubnetConfig -Name $SubnetName -AddressPrefix "10.20.1.0/24"
$vnet = New-AzVirtualNetwork -ResourceGroupName $RG -Location $Location -Name $VnetName -AddressPrefix "10.20.0.0/16" -Subnet $subnet
$webAsg = New-AzApplicationSecurityGroup -ResourceGroupName $RG -Location $Location -Name $WebAsgName
$appAsg = New-AzApplicationSecurityGroup -ResourceGroupName $RG -Location $Location -Name $AppAsgName

$allowRule = New-AzNetworkSecurityRuleConfig -Name "Allow-Web-To-App-443" -Priority 100 -Direction Inbound -Access Allow -Protocol Tcp -SourceApplicationSecurityGroup $webAsg -DestinationApplicationSecurityGroup $appAsg -SourcePortRange * -DestinationPortRange 443
$denyRule = New-AzNetworkSecurityRuleConfig -Name "Deny-Internet-To-App" -Priority 110 -Direction Inbound -Access Deny -Protocol Tcp -SourceAddressPrefix Internet -DestinationApplicationSecurityGroup $appAsg -SourcePortRange * -DestinationPortRange 443
$nsg = New-AzNetworkSecurityGroup -ResourceGroupName $RG -Location $Location -Name $NsgName -SecurityRules $allowRule,$denyRule

$subnetRef = Get-AzVirtualNetworkSubnetConfig -VirtualNetwork $vnet -Name $SubnetName
$nic1 = New-AzNetworkInterface -Name "nic-web-lab02" -ResourceGroupName $RG -Location $Location -Subnet $subnetRef -ApplicationSecurityGroup $webAsg
$nic2 = New-AzNetworkInterface -Name "nic-app-lab02" -ResourceGroupName $RG -Location $Location -Subnet $subnetRef -ApplicationSecurityGroup $appAsg -NetworkSecurityGroup $nsg

Set-AzVirtualNetworkSubnetConfig -VirtualNetwork $vnet -Name $SubnetName -AddressPrefix "10.20.1.0/24" -NetworkSecurityGroup $nsg | Set-AzVirtualNetwork

$law = New-AzOperationalInsightsWorkspace -ResourceGroupName $RG -Location $Location -Name $LawName -Sku PerGB2018
$storage = New-AzStorageAccount -ResourceGroupName $RG -Name $StorageName -Location $Location -SkuName Standard_LRS
$nw = Get-AzNetworkWatcher -Location $Location
Set-AzNetworkWatcherConfigFlowLog -NetworkWatcher $nw -TargetResourceId $nsg.Id -StorageId $storage.Id -EnableFlowLog $true -FormatType JSON -FormatVersion 2 -EnableTrafficAnalytics -WorkspaceResourceId $law.ResourceId -TrafficAnalyticsInterval 10
```

### IaC implementation

#### Bicep
```bicep
param location string = resourceGroup().location

resource asgWeb 'Microsoft.Network/applicationSecurityGroups@2023-11-01' = {
  name: 'asg-web-lab02'
  location: location
}

resource asgApp 'Microsoft.Network/applicationSecurityGroups@2023-11-01' = {
  name: 'asg-app-lab02'
  location: location
}

resource nsg 'Microsoft.Network/networkSecurityGroups@2023-11-01' = {
  name: 'nsg-app-lab02'
  location: location
  properties: {
    securityRules: [
      {
        name: 'Allow-Web-To-App-443'
        properties: {
          priority: 100
          direction: 'Inbound'
          access: 'Allow'
          protocol: 'Tcp'
          sourcePortRange: '*'
          destinationPortRange: '443'
          sourceApplicationSecurityGroups: [{ id: asgWeb.id }]
          destinationApplicationSecurityGroups: [{ id: asgApp.id }]
        }
      }
    ]
  }
}
```

#### Terraform
```hcl
resource "azurerm_application_security_group" "web" {
  name                = "asg-web-lab02"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
}

resource "azurerm_application_security_group" "app" {
  name                = "asg-app-lab02"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
}

resource "azurerm_network_security_group" "nsg" {
  name                = "nsg-app-lab02"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name

  security_rule {
    name                                       = "Allow-Web-To-App-443"
    priority                                   = 100
    direction                                  = "Inbound"
    access                                     = "Allow"
    protocol                                   = "Tcp"
    source_port_range                          = "*"
    destination_port_range                     = "443"
    source_application_security_group_ids      = [azurerm_application_security_group.web.id]
    destination_application_security_group_ids = [azurerm_application_security_group.app.id]
  }
}
```

### Validation and success criteria
- `Allow-Web-To-App-443` is hit before broader deny rules.
- IP flow verify shows **Allow** for web-to-app 443 and **Deny** for internet-to-app 443.
- Flow logs are enabled and Traffic Analytics starts populating after traffic is generated.

### Cleanup

#### Azure CLI
```bash
az group delete -n $RG --yes --no-wait
```

#### PowerShell
```powershell
Remove-AzResourceGroup -Name $RG -Force -AsJob
```

### Exam Tip
Use **ASGs** when the question says workloads scale dynamically and rules should follow the app tier, not the IP address.

---

## Lab 3: Azure Load Balancer (L4)

### Objective
Deploy a Standard Load Balancer, attach backend workloads, configure health probes and rules, add outbound rules, and compare public versus internal load balancing.

### When to Use This
Use Azure Load Balancer when you need high-performance TCP/UDP distribution with low latency and no Layer 7 inspection.

### Exam domain mapping
- **Primary:** Design infrastructure solutions (30-35%)
- **Secondary:** Design business continuity solutions (15-20%)

### Key AZ-305 Concepts
- Standard versus Basic SKU
- Public versus internal load balancer
- Health probes and backend health
- Inbound NAT versus load balancing rules
- Outbound SNAT behavior and outbound rules

### Prerequisites
- Two backend VMs or VM scale set instances in the same VNet
- NSG rules that allow health probe and application traffic

### Architecture and design rationale
Azure Load Balancer operates at Layer 4 only. It is ideal for non-HTTP protocols or when TLS termination should stay on the VM. For exam scenarios, use **Application Gateway** instead when you need cookie affinity, URL routing, or WAF. Choose **Standard Load Balancer** for production workloads because it is zone-aware and secure by default.

### Implementation steps
1. Create a VNet and backend subnet.
2. Deploy or identify two backend VMs.
3. Create a Standard public load balancer.
4. Create a backend pool, health probe, and load-balancing rule.
5. Add an outbound rule.
6. Create an internal load balancer for private consumers.

### Full CLI + PowerShell commands

#### Azure CLI
```bash
RG="rg-net-lab03"
LOCATION="eastus"
VNET="vnet-lb-lab03"
SUBNET="backend-subnet"
PIP="pip-lb-lab03"
SLB="slb-public-lab03"
ILB="slb-internal-lab03"
BACKEND1_NIC="nic-web01"
BACKEND2_NIC="nic-web02"

az group create -n $RG -l $LOCATION
az network vnet create -g $RG -n $VNET -l $LOCATION --address-prefix 10.30.0.0/16 --subnet-name $SUBNET --subnet-prefix 10.30.1.0/24
az network public-ip create -g $RG -n $PIP --sku Standard --allocation-method Static

az network lb create -g $RG -n $SLB --sku Standard --public-ip-address $PIP \
  --frontend-ip-name fe-public --backend-pool-name bepool-public
az network lb probe create -g $RG --lb-name $SLB -n http-probe --protocol Tcp --port 80
az network lb rule create -g $RG --lb-name $SLB -n web-rule --protocol Tcp \
  --frontend-port 80 --backend-port 80 --frontend-ip-name fe-public --backend-pool-name bepool-public \
  --probe-name http-probe --idle-timeout 15 --enable-tcp-reset true
az network lb outbound-rule create -g $RG --lb-name $SLB -n outbound-web \
  --frontend-ip-configs fe-public --backend-pool-name bepool-public --protocol All --allocated-outbound-ports 1024

az network nic ip-config address-pool add -g $RG --nic-name $BACKEND1_NIC --address-pool bepool-public --ip-config-name ipconfig1 --lb-name $SLB
az network nic ip-config address-pool add -g $RG --nic-name $BACKEND2_NIC --address-pool bepool-public --ip-config-name ipconfig1 --lb-name $SLB

az network lb create -g $RG -n $ILB --sku Standard --vnet-name $VNET --subnet $SUBNET \
  --private-ip-address 10.30.1.100 --frontend-ip-name fe-internal --backend-pool-name bepool-internal
az network lb probe create -g $RG --lb-name $ILB -n tcp-8080-probe --protocol Tcp --port 8080
az network lb rule create -g $RG --lb-name $ILB -n internal-app-rule --protocol Tcp \
  --frontend-port 8080 --backend-port 8080 --frontend-ip-name fe-internal --backend-pool-name bepool-internal \
  --probe-name tcp-8080-probe
```

#### PowerShell
```powershell
$RG = "rg-net-lab03"
$Location = "eastus"
$VnetName = "vnet-lb-lab03"
$SubnetName = "backend-subnet"

New-AzResourceGroup -Name $RG -Location $Location
$subnet = New-AzVirtualNetworkSubnetConfig -Name $SubnetName -AddressPrefix "10.30.1.0/24"
$vnet = New-AzVirtualNetwork -ResourceGroupName $RG -Location $Location -Name $VnetName -AddressPrefix "10.30.0.0/16" -Subnet $subnet
$pip = New-AzPublicIpAddress -ResourceGroupName $RG -Location $Location -Name "pip-lb-lab03" -AllocationMethod Static -Sku Standard

$fePublic = New-AzLoadBalancerFrontendIpConfig -Name "fe-public" -PublicIpAddress $pip
$bePoolPublic = New-AzLoadBalancerBackendAddressPoolConfig -Name "bepool-public"
$probe80 = New-AzLoadBalancerProbeConfig -Name "http-probe" -Protocol Tcp -Port 80 -IntervalInSeconds 15 -ProbeCount 2
$rule80 = New-AzLoadBalancerRuleConfig -Name "web-rule" -FrontendIpConfiguration $fePublic -BackendAddressPool $bePoolPublic -Probe $probe80 -Protocol Tcp -FrontendPort 80 -BackendPort 80 -IdleTimeoutInMinutes 15 -EnableTcpReset $true
$outboundRule = New-AzLoadBalancerOutboundRuleConfig -Name "outbound-web" -Protocol All -FrontendIPConfiguration $fePublic -BackendAddressPool $bePoolPublic -AllocatedOutboundPort 1024
$slb = New-AzLoadBalancer -ResourceGroupName $RG -Location $Location -Name "slb-public-lab03" -Sku Standard -FrontendIpConfiguration $fePublic -BackendAddressPool $bePoolPublic -Probe $probe80 -LoadBalancingRule $rule80 -OutboundRule $outboundRule

$subnetRef = Get-AzVirtualNetworkSubnetConfig -VirtualNetwork $vnet -Name $SubnetName
$feInternal = New-AzLoadBalancerFrontendIpConfig -Name "fe-internal" -Subnet $subnetRef -PrivateIpAddress "10.30.1.100"
$bePoolInternal = New-AzLoadBalancerBackendAddressPoolConfig -Name "bepool-internal"
$probe8080 = New-AzLoadBalancerProbeConfig -Name "tcp-8080-probe" -Protocol Tcp -Port 8080
$rule8080 = New-AzLoadBalancerRuleConfig -Name "internal-app-rule" -FrontendIpConfiguration $feInternal -BackendAddressPool $bePoolInternal -Probe $probe8080 -Protocol Tcp -FrontendPort 8080 -BackendPort 8080
New-AzLoadBalancer -ResourceGroupName $RG -Location $Location -Name "slb-internal-lab03" -Sku Standard -FrontendIpConfiguration $feInternal -BackendAddressPool $bePoolInternal -Probe $probe8080 -LoadBalancingRule $rule8080
```

### IaC implementation

#### Bicep
```bicep
param location string = resourceGroup().location
param publicIpName string = 'pip-lb-lab03'
param lbName string = 'slb-public-lab03'

resource pip 'Microsoft.Network/publicIPAddresses@2023-11-01' = {
  name: publicIpName
  location: location
  sku: { name: 'Standard' }
  properties: { publicIPAllocationMethod: 'Static' }
}

resource lb 'Microsoft.Network/loadBalancers@2023-11-01' = {
  name: lbName
  location: location
  sku: { name: 'Standard' }
  properties: {
    frontendIPConfigurations: [
      {
        name: 'fe-public'
        properties: { publicIPAddress: { id: pip.id } }
      }
    ]
    backendAddressPools: [{ name: 'bepool-public' }]
    probes: [{ name: 'http-probe' properties: { protocol: 'Tcp' port: 80 intervalInSeconds: 15 numberOfProbes: 2 } }]
  }
}
```

#### Terraform
```hcl
resource "azurerm_public_ip" "lb_pip" {
  name                = "pip-lb-lab03"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  allocation_method   = "Static"
  sku                 = "Standard"
}

resource "azurerm_lb" "public" {
  name                = "slb-public-lab03"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  sku                 = "Standard"

  frontend_ip_configuration {
    name                 = "fe-public"
    public_ip_address_id = azurerm_public_ip.lb_pip.id
  }
}
```

### Validation and success criteria
- Health probes show healthy backend instances.
- Public LB distributes TCP traffic on port 80 across both backends.
- Internal LB responds only from inside the VNet or connected networks.
- Outbound connectivity uses the configured outbound rule instead of implicit outbound behavior.

### Cleanup

#### Azure CLI
```bash
az group delete -n $RG --yes --no-wait
```

#### PowerShell
```powershell
Remove-AzResourceGroup -Name $RG -Force -AsJob
```

### Exam Tip
**Load Balancer = L4**. If the question requires **WAF**, **host/path routing**, or **TLS offload**, Azure Load Balancer is the wrong answer.

---

## Lab 4: Application Gateway with WAF

### Objective
Deploy Application Gateway v2, configure backend pools, HTTP settings, custom probes, URL-based routing, SSL termination, and WAF protection.

### When to Use This
Use Application Gateway when you need Layer 7 web load balancing, TLS termination, cookie affinity, path-based routing, or regional WAF.

### Exam domain mapping
- **Primary:** Design infrastructure solutions (30-35%)
- **Secondary:** Design identity, governance, and monitoring solutions (25-30%)

### Key AZ-305 Concepts
- Layer 7 versus Layer 4 load balancing
- Autoscaling and zone redundancy in v2 SKU
- WAF versus reverse proxy only
- End-to-end TLS versus SSL offload
- URL-based routing for multi-site applications

### Prerequisites
- Backend web servers or App Service endpoints
- A PFX certificate for listener TLS termination
- Separate subnet for Application Gateway

### Architecture and design rationale
Application Gateway is best for regional web applications. Use WAF_v2 for production because it supports autoscaling, zone redundancy, and current WAF capabilities. For AZ-305, the trap is choosing Front Door when the requirement is **regional only** or choosing Load Balancer when path-based routing or WAF is required.

### Implementation steps
1. Create a VNet with a dedicated Application Gateway subnet.
2. Create a public IP.
3. Create backend pools and HTTP settings.
4. Create health probes.
5. Configure basic and URL-path routing.
6. Import an SSL certificate and enable HTTPS listener.
7. Enable WAF in prevention mode.

### Full CLI + PowerShell commands

#### Azure CLI
```bash
RG="rg-net-lab04"
LOCATION="eastus"
VNET="vnet-appgw-lab04"
APPGW_SUBNET="appgw-subnet"
APP_SUBNET="app-subnet"
PIP="pip-appgw-lab04"
APPGW="appgw-lab04"
WAFPOL="waf-appgw-lab04"
PFX_PATH="<path-to-cert.pfx>"
PFX_PASSWORD="<pfx-password>"
BACKEND1="10.40.2.4"
BACKEND2="10.40.2.5"

az group create -n $RG -l $LOCATION
az network vnet create -g $RG -n $VNET -l $LOCATION --address-prefix 10.40.0.0/16 --subnet-name $APPGW_SUBNET --subnet-prefix 10.40.1.0/24
az network vnet subnet create -g $RG --vnet-name $VNET -n $APP_SUBNET --address-prefixes 10.40.2.0/24
az network public-ip create -g $RG -n $PIP --sku Standard --allocation-method Static

az network application-gateway waf-policy create -g $RG -n $WAFPOL --location $LOCATION --mode Prevention --type OWASP --version 3.2
az network application-gateway create -g $RG -n $APPGW -l $LOCATION \
  --sku WAF_v2 --capacity 2 --vnet-name $VNET --subnet $APPGW_SUBNET --public-ip-address $PIP \
  --frontend-port 80 --http-settings-cookie-based-affinity Disabled --http-settings-port 80 \
  --http-settings-protocol Http --servers $BACKEND1 $BACKEND2 --waf-policy $WAFPOL

az network application-gateway probe create -g $RG --gateway-name $APPGW -n web-probe \
  --protocol Http --host-name-from-http-settings true --path /health --interval 30 --timeout 30 --threshold 3
az network application-gateway http-settings update -g $RG --gateway-name $APPGW -n appGatewayBackendHttpSettings \
  --probe web-probe --port 80 --protocol Http --timeout 30
az network application-gateway ssl-cert create -g $RG --gateway-name $APPGW -n sitecert \
  --cert-file $PFX_PATH --cert-password $PFX_PASSWORD
az network application-gateway frontend-port create -g $RG --gateway-name $APPGW -n port-443 --port 443
az network application-gateway http-listener create -g $RG --gateway-name $APPGW -n https-listener \
  --frontend-ip appGatewayFrontendIP --frontend-port port-443 --ssl-cert sitecert
az network application-gateway url-path-map create -g $RG --gateway-name $APPGW -n apps-pathmap \
  --default-address-pool appGatewayBackendPool --default-http-settings appGatewayBackendHttpSettings \
  --path-rules Name=imagesRule Paths=/images/* BackendAddressPool=appGatewayBackendPool BackendHttpSettings=appGatewayBackendHttpSettings
```

#### PowerShell
```powershell
$RG = "rg-net-lab04"
$Location = "eastus"
$VnetName = "vnet-appgw-lab04"
$PfxPath = "<path-to-cert.pfx>"
$PfxPassword = ConvertTo-SecureString "<pfx-password>" -AsPlainText -Force

New-AzResourceGroup -Name $RG -Location $Location
$appGwSubnet = New-AzVirtualNetworkSubnetConfig -Name "appgw-subnet" -AddressPrefix "10.40.1.0/24"
$appSubnet = New-AzVirtualNetworkSubnetConfig -Name "app-subnet" -AddressPrefix "10.40.2.0/24"
$vnet = New-AzVirtualNetwork -ResourceGroupName $RG -Location $Location -Name $VnetName -AddressPrefix "10.40.0.0/16" -Subnet $appGwSubnet,$appSubnet
$pip = New-AzPublicIpAddress -ResourceGroupName $RG -Name "pip-appgw-lab04" -Location $Location -AllocationMethod Static -Sku Standard

$gip = New-AzApplicationGatewayIPConfiguration -Name "gw-ipcfg" -Subnet (Get-AzVirtualNetworkSubnetConfig -VirtualNetwork $vnet -Name "appgw-subnet")
$fip = New-AzApplicationGatewayFrontendIPConfig -Name "frontend-ip" -PublicIPAddress $pip
$fp80 = New-AzApplicationGatewayFrontendPort -Name "port-80" -Port 80
$fp443 = New-AzApplicationGatewayFrontendPort -Name "port-443" -Port 443
$pool = New-AzApplicationGatewayBackendAddressPool -Name "backend-pool" -BackendIPAddresses "10.40.2.4","10.40.2.5"
$probe = New-AzApplicationGatewayProbeConfig -Name "web-probe" -Protocol Http -Path "/health" -Interval 30 -Timeout 30 -UnhealthyThreshold 3 -HostNameFromBackendHttpSettings
$setting = New-AzApplicationGatewayBackendHttpSetting -Name "http-setting" -Port 80 -Protocol Http -CookieBasedAffinity Disabled -Probe $probe
$cert = New-AzApplicationGatewaySslCertificate -Name "sitecert" -CertificateFile $PfxPath -Password $PfxPassword
$listener80 = New-AzApplicationGatewayHttpListener -Name "listener-80" -Protocol Http -FrontendIPConfiguration $fip -FrontendPort $fp80
$listener443 = New-AzApplicationGatewayHttpListener -Name "listener-443" -Protocol Https -FrontendIPConfiguration $fip -FrontendPort $fp443 -SslCertificate $cert
$rule = New-AzApplicationGatewayRequestRoutingRule -Name "basic-rule" -RuleType Basic -HttpListener $listener80 -BackendHttpSettings $setting -BackendAddressPool $pool
$sslRule = New-AzApplicationGatewayRequestRoutingRule -Name "https-rule" -RuleType Basic -HttpListener $listener443 -BackendHttpSettings $setting -BackendAddressPool $pool
$sku = New-AzApplicationGatewaySku -Name WAF_v2 -Tier WAF_v2 -Capacity 2
$wafConfig = New-AzApplicationGatewayWebApplicationFirewallConfiguration -Enabled $true -FirewallMode Prevention -RuleSetType OWASP -RuleSetVersion 3.2
New-AzApplicationGateway -Name "appgw-lab04" -ResourceGroupName $RG -Location $Location -BackendAddressPools $pool -BackendHttpSettingsCollection $setting -FrontendIpConfigurations $fip -GatewayIpConfigurations $gip -FrontendPorts $fp80,$fp443 -HttpListeners $listener80,$listener443 -RequestRoutingRules $rule,$sslRule -Sku $sku -Probes $probe -SslCertificates $cert -WebApplicationFirewallConfig $wafConfig
```

### IaC implementation

#### Bicep
```bicep
param location string = resourceGroup().location
param appGwName string = 'appgw-lab04'
param publicIpName string = 'pip-appgw-lab04'

resource pip 'Microsoft.Network/publicIPAddresses@2023-11-01' = {
  name: publicIpName
  location: location
  sku: { name: 'Standard' }
  properties: { publicIPAllocationMethod: 'Static' }
}

resource appgw 'Microsoft.Network/applicationGateways@2023-11-01' = {
  name: appGwName
  location: location
  sku: { name: 'WAF_v2' tier: 'WAF_v2' capacity: 2 }
  properties: {
    autoscaleConfiguration: { minCapacity: 2 maxCapacity: 4 }
  }
}
```

#### Terraform
```hcl
resource "azurerm_public_ip" "appgw" {
  name                = "pip-appgw-lab04"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  allocation_method   = "Static"
  sku                 = "Standard"
}

resource "azurerm_web_application_firewall_policy" "appgw" {
  name                = "waf-appgw-lab04"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name

  policy_settings {
    enabled = true
    mode    = "Prevention"
  }

  managed_rules {
    managed_rule_set {
      type    = "OWASP"
      version = "3.2"
    }
  }
}
```

### Validation and success criteria
- Backend health shows healthy targets.
- HTTP or HTTPS requests reach the correct backend.
- WAF policy is attached and in **Prevention** mode.
- You can explain when Application Gateway is preferred over Front Door.

### Cleanup

#### Azure CLI
```bash
az group delete -n $RG --yes --no-wait
```

#### PowerShell
```powershell
Remove-AzResourceGroup -Name $RG -Force -AsJob
```

### Exam Tip
Use **Application Gateway** for **regional** Layer 7 load balancing and WAF. Use **Front Door** when the requirement is **global** HTTP(S) entry point and acceleration.

---

## Lab 5: Azure Front Door

### Objective
Deploy an Azure Front Door Standard profile, configure origins and origin groups, route traffic, enable caching, attach WAF, and plan custom-domain onboarding.

### When to Use This
Use Front Door when you need global HTTP(S) load balancing, edge acceleration, anycast entry points, and WAF close to users.

### Exam domain mapping
- **Primary:** Design infrastructure solutions (30-35%)
- **Secondary:** Design identity, governance, and monitoring solutions (25-30%)

### Key AZ-305 Concepts
- Global anycast entry point
- Origin groups and health probes
- Caching and CDN-like acceleration
- WAF at the edge
- Front Door versus Traffic Manager versus Application Gateway

### Prerequisites
- Two internet-reachable origins such as App Service, Application Gateway, or NGINX VMs
- Custom domain DNS zone if you want full domain onboarding
- `front-door` Azure CLI extension installed

### Architecture and design rationale
Front Door is the preferred service for global web entry because it provides fast failover, TLS at the edge, and optional caching. It is not a private east-west traffic service. For the exam, the common trap is choosing Traffic Manager when the scenario requires **WAF**, **TLS offload**, or **caching**.

### Implementation steps
1. Create a Front Door profile and endpoint.
2. Create an origin group with health probe settings.
3. Add origins.
4. Create a route for the endpoint.
5. Enable caching.
6. Create and attach a WAF policy.
7. Add a custom domain and validate DNS.

### Full CLI + PowerShell commands

#### Azure CLI
```bash
RG="rg-net-lab05"
LOCATION="eastus"
PROFILE="afdprof-lab05"
ENDPOINT="afd-endpoint-lab05"
ORIGIN_GROUP="og-apps-lab05"
ROUTE="route-default"
ORIGIN1_HOST="<origin1-hostname>"
ORIGIN2_HOST="<origin2-hostname>"
CUSTOM_DOMAIN="<www.contoso.com>"

az group create -n $RG -l $LOCATION
az afd profile create -g $RG --profile-name $PROFILE --sku Standard_AzureFrontDoor
az afd endpoint create -g $RG --profile-name $PROFILE --endpoint-name $ENDPOINT --enabled-state Enabled
az afd origin-group create -g $RG --profile-name $PROFILE --origin-group-name $ORIGIN_GROUP \
  --probe-request-type GET --probe-protocol Https --probe-interval-in-seconds 120 \
  --sample-size 4 --successful-samples-required 3 --additional-latency-in-milliseconds 50
az afd origin create -g $RG --profile-name $PROFILE --origin-group-name $ORIGIN_GROUP --origin-name origin1 \
  --host-name $ORIGIN1_HOST --origin-host-header $ORIGIN1_HOST --priority 1 --weight 1000 --enabled-state Enabled
az afd origin create -g $RG --profile-name $PROFILE --origin-group-name $ORIGIN_GROUP --origin-name origin2 \
  --host-name $ORIGIN2_HOST --origin-host-header $ORIGIN2_HOST --priority 1 --weight 1000 --enabled-state Enabled
az afd route create -g $RG --profile-name $PROFILE --endpoint-name $ENDPOINT --route-name $ROUTE \
  --origin-group $ORIGIN_GROUP --supported-protocols Http Https --patterns-to-match '/*' \
  --forwarding-protocol MatchRequest --https-redirect Enabled --link-to-default-domain Enabled
az afd route update -g $RG --profile-name $PROFILE --endpoint-name $ENDPOINT --route-name $ROUTE \
  --cache-behavior OverrideAlways --query-string-caching-behavior IgnoreQueryString
az network front-door waf-policy create -g $RG -n afd-waf-lab05 --mode Prevention --disabled false
az network front-door waf-policy managed-rules add -g $RG --policy-name afd-waf-lab05 --type Microsoft_DefaultRuleSet --version 2.1
az afd custom-domain create -g $RG --profile-name $PROFILE --custom-domain-name customdomain1 --host-name $CUSTOM_DOMAIN
```

#### PowerShell
```powershell
$RG = "rg-net-lab05"
$Location = "eastus"
$ProfileName = "afdprof-lab05"
$EndpointName = "afd-endpoint-lab05"
$OriginGroup = "og-apps-lab05"
$Origin1 = "<origin1-hostname>"
$Origin2 = "<origin2-hostname>"
$CustomDomain = "<www.contoso.com>"

New-AzResourceGroup -Name $RG -Location $Location
$profile = New-AzFrontDoorCdnProfile -ResourceGroupName $RG -Name $ProfileName -Location Global -SkuName Standard_AzureFrontDoor
$endpoint = New-AzFrontDoorCdnEndpoint -ProfileName $ProfileName -ResourceGroupName $RG -Name $EndpointName -EnabledState Enabled
$ogHealth = New-AzFrontDoorCdnOriginGroupHealthProbeSettingObject -ProbeRequestType GET -ProbeProtocol Https -ProbeIntervalInSecond 120
$ogLb = New-AzFrontDoorCdnOriginGroupLoadBalancingSettingObject -SampleSize 4 -SuccessfulSamplesRequired 3 -AdditionalLatencyInMillisecond 50
New-AzFrontDoorCdnOriginGroup -ProfileName $ProfileName -ResourceGroupName $RG -Name $OriginGroup -HealthProbeSetting $ogHealth -LoadBalancingSetting $ogLb
New-AzFrontDoorCdnOrigin -ProfileName $ProfileName -ResourceGroupName $RG -OriginGroupName $OriginGroup -Name origin1 -HostName $Origin1 -OriginHostHeader $Origin1 -Priority 1 -Weight 1000
New-AzFrontDoorCdnOrigin -ProfileName $ProfileName -ResourceGroupName $RG -OriginGroupName $OriginGroup -Name origin2 -HostName $Origin2 -OriginHostHeader $Origin2 -Priority 1 -Weight 1000
New-AzFrontDoorCdnRoute -EndpointName $EndpointName -ProfileName $ProfileName -ResourceGroupName $RG -Name route-default -OriginGroupId (Get-AzFrontDoorCdnOriginGroup -ProfileName $ProfileName -ResourceGroupName $RG -Name $OriginGroup).Id -SupportedProtocol Http,Https -PatternsToMatch '/*' -ForwardingProtocol MatchRequest -HttpsRedirect Enabled -LinkToDefaultDomain Enabled

# If your Az.Cdn version lacks newer WAF cmdlets, use REST or Azure CLI from PowerShell.
az network front-door waf-policy create -g $RG -n afd-waf-lab05 --mode Prevention --disabled false
az afd custom-domain create -g $RG --profile-name $ProfileName --custom-domain-name customdomain1 --host-name $CustomDomain
```

### IaC implementation

#### Bicep
```bicep
param profileName string = 'afdprof-lab05'
param location string = 'global'

resource profile 'Microsoft.Cdn/profiles@2024-02-01' = {
  name: profileName
  location: location
  sku: { name: 'Standard_AzureFrontDoor' }
}

resource endpoint 'Microsoft.Cdn/profiles/afdEndpoints@2024-02-01' = {
  name: 'afd-endpoint-lab05'
  parent: profile
  location: location
  properties: { enabledState: 'Enabled' }
}
```

#### Terraform
```hcl
resource "azurerm_cdn_frontdoor_profile" "afd" {
  name                = "afdprof-lab05"
  resource_group_name = azurerm_resource_group.rg.name
  sku_name            = "Standard_AzureFrontDoor"
}

resource "azurerm_cdn_frontdoor_endpoint" "endpoint" {
  name                     = "afd-endpoint-lab05"
  cdn_frontdoor_profile_id = azurerm_cdn_frontdoor_profile.afd.id
}
```

### Validation and success criteria
- Front Door endpoint is reachable over HTTPS.
- Health probes report healthy origins.
- Cached content shows lower latency on repeated requests.
- WAF policy is associated with the Front Door route or domain.
- Custom-domain DNS validation completes successfully.

### Cleanup

#### Azure CLI
```bash
az group delete -n $RG --yes --no-wait
```

#### PowerShell
```powershell
Remove-AzResourceGroup -Name $RG -Force -AsJob
```

### Exam Tip
If the requirement says **global**, **edge**, **WAF**, **acceleration**, or **single anycast endpoint**, think **Azure Front Door** first.

---

## Lab 6: Azure Firewall

### Objective
Deploy Azure Firewall Standard, configure DNAT, network, and application rules, prepare forced-tunneling design elements, and enable threat intelligence.

### When to Use This
Use Azure Firewall when you need centralized, managed network security, outbound FQDN control, and scalable policy-based inspection in Azure.

### Exam domain mapping
- **Primary:** Design infrastructure solutions (30-35%)
- **Secondary:** Design identity, governance, and monitoring solutions (25-30%)

### Key AZ-305 Concepts
- Centralized egress and ingress control
- DNAT versus network versus application rules
- Firewall Policy and rule collection groups
- Threat intelligence alert or deny modes
- Forced tunneling for on-prem egress inspection

### Prerequisites
- Dedicated `AzureFirewallSubnet`
- Optional `AzureFirewallManagementSubnet` for forced tunneling patterns
- Public IPs for data plane and management plane if using forced tunneling

### Architecture and design rationale
Azure Firewall should usually sit in the hub VNet. Use Firewall Policy so rules can be reused and managed centrally. Application rules filter by FQDN and are better than broad outbound internet network rules when the protocol is HTTP/S. For the exam, DNAT is for inbound publishing, not general outbound access control.

### Implementation steps
1. Create a hub VNet with firewall and workload subnets.
2. Deploy Azure Firewall Standard and Firewall Policy.
3. Add DNAT, network, and application rule collections.
4. Set threat intelligence mode.
5. Add default route tables that force workload egress through the firewall.
6. If required, add management subnet and management public IP for forced tunneling design.

### Full CLI + PowerShell commands

#### Azure CLI
```bash
RG="rg-net-lab06"
LOCATION="eastus"
VNET="vnet-fw-lab06"
FW="azfw-lab06"
POLICY="fwpol-lab06"
PIP="pip-fw-lab06"
MGMTPIP="pip-fwmgmt-lab06"

az group create -n $RG -l $LOCATION
az network vnet create -g $RG -n $VNET -l $LOCATION --address-prefix 10.60.0.0/16 --subnet-name AzureFirewallSubnet --subnet-prefix 10.60.1.0/26
az network vnet subnet create -g $RG --vnet-name $VNET -n AzureFirewallManagementSubnet --address-prefixes 10.60.2.0/26
az network vnet subnet create -g $RG --vnet-name $VNET -n WorkloadSubnet --address-prefixes 10.60.10.0/24
az network public-ip create -g $RG -n $PIP --sku Standard --allocation-method Static
az network public-ip create -g $RG -n $MGMTPIP --sku Standard --allocation-method Static
az network firewall policy create -g $RG -n $POLICY -l $LOCATION --threat-intel-mode Alert
az network firewall create -g $RG -n $FW -l $LOCATION --sku AZFW_VNet --tier Standard --firewall-policy $POLICY
az network firewall ip-config create -g $RG -f $FW -n fw-data --public-ip-address $PIP --vnet-name $VNET
az network firewall management-ip-config create -g $RG -f $FW -n fw-mgmt --public-ip-address $MGMTPIP --vnet-name $VNET

az network firewall policy rule-collection-group create -g $RG --policy-name $POLICY -n rcg-default --priority 100
az network firewall policy rule-collection-group collection add-filter-collection -g $RG --policy-name $POLICY --rule-collection-group-name rcg-default \
  -n app-rc --action Allow --collection-priority 200
az network firewall policy rule-collection-group collection rule add -g $RG --policy-name $POLICY --rule-collection-group-name rcg-default \
  --collection-name app-rc -n allow-msupdate --rule-type ApplicationRule --source-addresses 10.60.10.0/24 \
  --protocols Http=80 Https=443 --target-fqdns '*.windowsupdate.com' '*.microsoft.com'
az network firewall policy rule-collection-group collection add-network-collection -g $RG --policy-name $POLICY --rule-collection-group-name rcg-default \
  -n net-rc --action Allow --collection-priority 300
az network firewall policy rule-collection-group collection rule add -g $RG --policy-name $POLICY --rule-collection-group-name rcg-default \
  --collection-name net-rc -n allow-dns --rule-type NetworkRule --ip-protocols UDP TCP \
  --source-addresses 10.60.10.0/24 --destination-addresses 168.63.129.16 --destination-ports 53
az network firewall policy rule-collection-group collection add-nat-collection -g $RG --policy-name $POLICY --rule-collection-group-name rcg-default \
  -n nat-rc --action Dnat --collection-priority 100
az network firewall policy rule-collection-group collection rule add -g $RG --policy-name $POLICY --rule-collection-group-name rcg-default \
  --collection-name nat-rc -n dnat-rdp --rule-type NatRule --source-addresses '*' --destination-addresses <firewall-public-ip> \
  --destination-ports 3389 --translated-address 10.60.10.4 --translated-port 3389 --ip-protocols TCP

FW_PRIVATE_IP=$(az network firewall ip-config list -g $RG -f $FW --query "[0].privateIpAddress" -o tsv)
az network route-table create -g $RG -n rt-fw-egress
az network route-table route create -g $RG --route-table-name rt-fw-egress -n default-via-fw \
  --address-prefix 0.0.0.0/0 --next-hop-type VirtualAppliance --next-hop-ip-address $FW_PRIVATE_IP
az network vnet subnet update -g $RG --vnet-name $VNET -n WorkloadSubnet --route-table rt-fw-egress
```

#### PowerShell
```powershell
$RG = "rg-net-lab06"
$Location = "eastus"
$VnetName = "vnet-fw-lab06"

New-AzResourceGroup -Name $RG -Location $Location
$subnets = @(
  New-AzVirtualNetworkSubnetConfig -Name "AzureFirewallSubnet" -AddressPrefix "10.60.1.0/26"
  New-AzVirtualNetworkSubnetConfig -Name "AzureFirewallManagementSubnet" -AddressPrefix "10.60.2.0/26"
  New-AzVirtualNetworkSubnetConfig -Name "WorkloadSubnet" -AddressPrefix "10.60.10.0/24"
)
$vnet = New-AzVirtualNetwork -ResourceGroupName $RG -Location $Location -Name $VnetName -AddressPrefix "10.60.0.0/16" -Subnet $subnets
$pip = New-AzPublicIpAddress -ResourceGroupName $RG -Name "pip-fw-lab06" -Location $Location -AllocationMethod Static -Sku Standard
$mgmtPip = New-AzPublicIpAddress -ResourceGroupName $RG -Name "pip-fwmgmt-lab06" -Location $Location -AllocationMethod Static -Sku Standard
$policy = New-AzFirewallPolicy -Name "fwpol-lab06" -ResourceGroupName $RG -Location $Location -ThreatIntelMode Alert
$fw = New-AzFirewall -Name "azfw-lab06" -ResourceGroupName $RG -Location $Location -VirtualNetworkName $VnetName -PublicIpName "pip-fw-lab06" -FirewallPolicyId $policy.Id -SkuName AZFW_VNet -SkuTier Standard

$fw = Get-AzFirewall -Name "azfw-lab06" -ResourceGroupName $RG
$fw.ManagementIpConfiguration = New-AzFirewallIPConfiguration -Name "fw-mgmt" -SubnetId ((Get-AzVirtualNetworkSubnetConfig -VirtualNetwork $vnet -Name "AzureFirewallManagementSubnet").Id) -PublicIpAddress $mgmtPip
Set-AzFirewall -AzureFirewall $fw

$appRule = New-AzFirewallPolicyApplicationRule -Name "allow-msupdate" -SourceAddress "10.60.10.0/24" -Protocol "http:80","https:443" -TargetFqdn "*.windowsupdate.com","*.microsoft.com"
$appCollection = New-AzFirewallPolicyFilterRuleCollection -Name "app-rc" -Priority 200 -ActionType Allow -Rule $appRule
$netRule = New-AzFirewallPolicyNetworkRule -Name "allow-dns" -Protocol "UDP","TCP" -SourceAddress "10.60.10.0/24" -DestinationAddress "168.63.129.16" -DestinationPort 53
$netCollection = New-AzFirewallPolicyFilterRuleCollection -Name "net-rc" -Priority 300 -ActionType Allow -Rule $netRule
$natRule = New-AzFirewallPolicyNatRule -Name "dnat-rdp" -Protocol TCP -SourceAddress * -DestinationAddress "<firewall-public-ip>" -DestinationPort 3389 -TranslatedAddress "10.60.10.4" -TranslatedPort 3389
$natCollection = New-AzFirewallPolicyNatRuleCollection -Name "nat-rc" -Priority 100 -ActionType Dnat -Rule $natRule
New-AzFirewallPolicyRuleCollectionGroup -Name "rcg-default" -FirewallPolicyObject $policy -Priority 100 -RuleCollection $natCollection,$appCollection,$netCollection
```

### IaC implementation

#### Bicep
```bicep
param location string = resourceGroup().location
param firewallName string = 'azfw-lab06'

resource pip 'Microsoft.Network/publicIPAddresses@2023-11-01' = {
  name: 'pip-fw-lab06'
  location: location
  sku: { name: 'Standard' }
  properties: { publicIPAllocationMethod: 'Static' }
}

resource policy 'Microsoft.Network/firewallPolicies@2023-11-01' = {
  name: 'fwpol-lab06'
  location: location
  properties: {
    threatIntelMode: 'Alert'
    sku: { tier: 'Standard' }
  }
}
```

#### Terraform
```hcl
resource "azurerm_firewall_policy" "policy" {
  name                = "fwpol-lab06"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  sku                 = "Standard"
  threat_intelligence_mode = "Alert"
}

resource "azurerm_firewall" "fw" {
  name                = "azfw-lab06"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  sku_name            = "AZFW_VNet"
  sku_tier            = "Standard"
  firewall_policy_id  = azurerm_firewall_policy.policy.id
}
```

### Validation and success criteria
- Azure Firewall is provisioned with a private IP in `AzureFirewallSubnet`.
- DNAT, network, and application rule collections exist in the firewall policy.
- Threat intelligence mode is enabled.
- Workload subnet routes `0.0.0.0/0` to the firewall private IP.

### Cleanup

#### Azure CLI
```bash
az group delete -n $RG --yes --no-wait
```

#### PowerShell
```powershell
Remove-AzResourceGroup -Name $RG -Force -AsJob
```

### Exam Tip
Use **application rules** for HTTP/S FQDN filtering. Use **network rules** for non-web protocols. Use **DNAT** only when publishing inbound services.

---

## Lab 7: Private Endpoints

### Objective
Create private endpoints for Storage Account and Azure SQL Database, integrate Private DNS zones, validate private name resolution, and disable public access.

### When to Use This
Use private endpoints when PaaS services must be consumed privately over Azure networking rather than from public internet endpoints.

### Exam domain mapping
- **Primary:** Design infrastructure solutions (30-35%)
- **Secondary:** Design data storage solutions (20-25%)

### Key AZ-305 Concepts
- Private Link versus service endpoints
- Private DNS zone integration
- Public network access disablement
- Data exfiltration reduction
- Name resolution requirements for private access

### Prerequisites
- A VNet and subnet with private endpoint network policies disabled
- SQL admin credentials for the logical server
- A test VM in the same VNet if you want to run DNS and connectivity validation

### Architecture and design rationale
Private Endpoint maps a PaaS resource to a private IP inside your VNet. This is stronger isolation than service endpoints because the resource is reached through Private Link and can have public access disabled. The exam trap is assuming private endpoints work without proper **Private DNS zone** configuration.

### Implementation steps
1. Create a VNet and subnet for private endpoints.
2. Create a Storage Account and SQL server/database.
3. Create private DNS zones and link them to the VNet.
4. Create private endpoints and zone groups.
5. Disable public network access on both services.
6. Validate DNS and connectivity.

### Full CLI + PowerShell commands

#### Azure CLI
```bash
RG="rg-net-lab07"
LOCATION="eastus"
VNET="vnet-pe-lab07"
SUBNET="pe-subnet"
STG="stpeaz305lab07$RANDOM"
SQLSERVER="sqlpeaz305lab07$RANDOM"
SQLDB="sqldb-lab07"
SQLADMIN="sqladminuser"
SQLPASSWORD="<StrongPassword123!>"
BLOB_ZONE="privatelink.blob.core.windows.net"
SQL_ZONE="privatelink.database.windows.net"

az group create -n $RG -l $LOCATION
az network vnet create -g $RG -n $VNET -l $LOCATION --address-prefix 10.70.0.0/16 --subnet-name $SUBNET --subnet-prefix 10.70.1.0/24
az network vnet subnet update -g $RG --vnet-name $VNET -n $SUBNET --disable-private-endpoint-network-policies true

az storage account create -g $RG -n $STG -l $LOCATION --sku Standard_LRS --kind StorageV2
az sql server create -g $RG -n $SQLSERVER -l $LOCATION -u $SQLADMIN -p $SQLPASSWORD
az sql db create -g $RG -s $SQLSERVER -n $SQLDB --service-objective S0

az network private-dns zone create -g $RG -n $BLOB_ZONE
az network private-dns zone create -g $RG -n $SQL_ZONE
az network private-dns link vnet create -g $RG -n blob-link -z $BLOB_ZONE -v $VNET -e false
az network private-dns link vnet create -g $RG -n sql-link -z $SQL_ZONE -v $VNET -e false

STG_ID=$(az storage account show -g $RG -n $STG --query id -o tsv)
SQL_ID=$(az sql server show -g $RG -n $SQLSERVER --query id -o tsv)

az network private-endpoint create -g $RG -n pe-storage-lab07 -l $LOCATION --vnet-name $VNET --subnet $SUBNET \
  --private-connection-resource-id $STG_ID --group-id blob --connection-name peconn-storage
az network private-endpoint dns-zone-group create -g $RG --endpoint-name pe-storage-lab07 -n blob-zg \
  --private-dns-zone $BLOB_ZONE --zone-name $BLOB_ZONE

az network private-endpoint create -g $RG -n pe-sql-lab07 -l $LOCATION --vnet-name $VNET --subnet $SUBNET \
  --private-connection-resource-id $SQL_ID --group-id sqlServer --connection-name peconn-sql
az network private-endpoint dns-zone-group create -g $RG --endpoint-name pe-sql-lab07 -n sql-zg \
  --private-dns-zone $SQL_ZONE --zone-name $SQL_ZONE

az storage account update -g $RG -n $STG --public-network-access Disabled
az sql server update -g $RG -n $SQLSERVER --set publicNetworkAccess=Disabled
```

#### PowerShell
```powershell
$RG = "rg-net-lab07"
$Location = "eastus"
$VnetName = "vnet-pe-lab07"
$SubnetName = "pe-subnet"
$StorageName = "stpeaz305lab07$(Get-Random)"
$SqlServerName = "sqlpeaz305lab07$(Get-Random)"
$SqlDbName = "sqldb-lab07"
$SqlAdmin = "sqladminuser"
$SqlPassword = ConvertTo-SecureString "<StrongPassword123!>" -AsPlainText -Force
$Cred = New-Object System.Management.Automation.PSCredential($SqlAdmin,$SqlPassword)

New-AzResourceGroup -Name $RG -Location $Location
$subnet = New-AzVirtualNetworkSubnetConfig -Name $SubnetName -AddressPrefix "10.70.1.0/24" -PrivateEndpointNetworkPoliciesFlag Disabled
$vnet = New-AzVirtualNetwork -ResourceGroupName $RG -Location $Location -Name $VnetName -AddressPrefix "10.70.0.0/16" -Subnet $subnet
$storage = New-AzStorageAccount -ResourceGroupName $RG -Name $StorageName -Location $Location -SkuName Standard_LRS -Kind StorageV2
$sqlServer = New-AzSqlServer -ResourceGroupName $RG -ServerName $SqlServerName -Location $Location -SqlAdministratorCredentials $Cred
New-AzSqlDatabase -ResourceGroupName $RG -ServerName $SqlServerName -DatabaseName $SqlDbName -RequestedServiceObjectiveName "S0"

$blobZone = New-AzPrivateDnsZone -ResourceGroupName $RG -Name "privatelink.blob.core.windows.net"
$sqlZone = New-AzPrivateDnsZone -ResourceGroupName $RG -Name "privatelink.database.windows.net"
New-AzPrivateDnsVirtualNetworkLink -ResourceGroupName $RG -ZoneName $blobZone.Name -Name "blob-link" -VirtualNetworkId $vnet.Id
New-AzPrivateDnsVirtualNetworkLink -ResourceGroupName $RG -ZoneName $sqlZone.Name -Name "sql-link" -VirtualNetworkId $vnet.Id

$blobPe = New-AzPrivateEndpoint -ResourceGroupName $RG -Name "pe-storage-lab07" -Location $Location -Subnet (Get-AzVirtualNetworkSubnetConfig -VirtualNetwork $vnet -Name $SubnetName) -PrivateLinkServiceConnection (New-AzPrivateLinkServiceConnection -Name "peconn-storage" -PrivateLinkServiceId $storage.Id -GroupId "blob")
New-AzPrivateDnsZoneGroup -ResourceGroupName $RG -PrivateEndpointName $blobPe.Name -Name "blob-zg" -PrivateDnsZoneConfig (New-AzPrivateDnsZoneConfig -Name "blob-config" -PrivateDnsZoneId $blobZone.ResourceId)

$sqlPe = New-AzPrivateEndpoint -ResourceGroupName $RG -Name "pe-sql-lab07" -Location $Location -Subnet (Get-AzVirtualNetworkSubnetConfig -VirtualNetwork $vnet -Name $SubnetName) -PrivateLinkServiceConnection (New-AzPrivateLinkServiceConnection -Name "peconn-sql" -PrivateLinkServiceId $sqlServer.ResourceId -GroupId "sqlServer")
New-AzPrivateDnsZoneGroup -ResourceGroupName $RG -PrivateEndpointName $sqlPe.Name -Name "sql-zg" -PrivateDnsZoneConfig (New-AzPrivateDnsZoneConfig -Name "sql-config" -PrivateDnsZoneId $sqlZone.ResourceId)

Update-AzStorageAccountNetworkRuleSet -ResourceGroupName $RG -Name $StorageName -DefaultAction Deny
Set-AzSqlServer -ResourceGroupName $RG -ServerName $SqlServerName -PublicNetworkAccess Disabled
```

### IaC implementation

#### Bicep
```bicep
param location string = resourceGroup().location
param storageName string

resource storage 'Microsoft.Storage/storageAccounts@2023-05-01' = {
  name: storageName
  location: location
  kind: 'StorageV2'
  sku: { name: 'Standard_LRS' }
  properties: {
    publicNetworkAccess: 'Disabled'
  }
}
```

#### Terraform
```hcl
resource "azurerm_private_dns_zone" "blob" {
  name                = "privatelink.blob.core.windows.net"
  resource_group_name = azurerm_resource_group.rg.name
}

resource "azurerm_storage_account" "st" {
  name                     = var.storage_name
  resource_group_name      = azurerm_resource_group.rg.name
  location                 = azurerm_resource_group.rg.location
  account_tier             = "Standard"
  account_replication_type = "LRS"
  public_network_access_enabled = false
}
```

### Validation and success criteria
- The storage and SQL FQDNs resolve to private IP addresses from inside the VNet.
- Public network access is disabled.
- Connectivity works from a VM inside the VNet.
- You can explain why Private DNS is required.

### Cleanup

#### Azure CLI
```bash
az group delete -n $RG --yes --no-wait
```

#### PowerShell
```powershell
Remove-AzResourceGroup -Name $RG -Force -AsJob
```

### Exam Tip
If the requirement says **PaaS privately reachable and public endpoint disabled**, pick **Private Endpoint** instead of service endpoints.

---

## Lab 8: Site-to-Site VPN

### Objective
Deploy a VPN Gateway, create a Local Network Gateway, configure a site-to-site connection, validate connection health, and review optional BGP settings.

### When to Use This
Use site-to-site VPN when you need hybrid connectivity over the public internet with encryption and lower cost than dedicated private circuits.

### Exam domain mapping
- **Primary:** Design infrastructure solutions (30-35%)
- **Secondary:** Design business continuity solutions (15-20%)

### Key AZ-305 Concepts
- VPN Gateway SKUs and tunnel throughput
- Local Network Gateway representation of on-premises
- Shared key and IPSec tunnel setup
- Optional BGP for route exchange
- Gateway transit reuse from a hub

### Prerequisites
- Public IP address of the on-premises VPN device
- On-premises address prefixes
- Shared key agreed on both ends

### Architecture and design rationale
Site-to-site VPN is useful for hybrid proof of concepts, branch offices, and cost-sensitive hybrid connectivity. For the exam, if the requirement emphasizes **predictable high bandwidth**, **private connectivity**, or **compliance isolation**, ExpressRoute is usually the better answer.

### Implementation steps
1. Create a VNet with `GatewaySubnet`.
2. Create a public IP and VPN Gateway.
3. Create a Local Network Gateway.
4. Create the site-to-site connection.
5. Verify tunnel status.
6. Optionally enable BGP.

### Full CLI + PowerShell commands

#### Azure CLI
```bash
RG="rg-net-lab08"
LOCATION="eastus"
VNET="vnet-vpn-lab08"
VNG="vng-lab08"
LNG="lng-onprem-lab08"
PIP="pip-vng-lab08"
ONPREM_IP="<onprem-vpn-public-ip>"
ONPREM_PREFIX="192.168.0.0/16"
SHARED_KEY="Az305S2SVpnKey!"

az group create -n $RG -l $LOCATION
az network vnet create -g $RG -n $VNET -l $LOCATION --address-prefix 10.80.0.0/16 --subnet-name GatewaySubnet --subnet-prefix 10.80.255.0/27
az network public-ip create -g $RG -n $PIP --sku Standard --allocation-method Static
az network vnet-gateway create -g $RG -n $VNG -l $LOCATION --public-ip-addresses $PIP --vnet $VNET \
  --gateway-type Vpn --vpn-type RouteBased --sku VpnGw1 --asn 65515 --no-wait
az network local-gateway create -g $RG -n $LNG -l $LOCATION --gateway-ip-address $ONPREM_IP --local-address-prefixes $ONPREM_PREFIX
az network vpn-connection create -g $RG -n s2s-conn-lab08 --vnet-gateway1 $VNG --local-gateway2 $LNG --shared-key $SHARED_KEY --enable-bgp false
az network vpn-connection show -g $RG -n s2s-conn-lab08 --query "{status:connectionStatus,ingress:ingressBytesTransferred,egress:egressBytesTransferred}"
```

#### PowerShell
```powershell
$RG = "rg-net-lab08"
$Location = "eastus"
$OnPremIp = "<onprem-vpn-public-ip>"
$SharedKey = ConvertTo-SecureString "Az305S2SVpnKey!" -AsPlainText -Force

New-AzResourceGroup -Name $RG -Location $Location
$gwSubnet = New-AzVirtualNetworkSubnetConfig -Name "GatewaySubnet" -AddressPrefix "10.80.255.0/27"
$vnet = New-AzVirtualNetwork -ResourceGroupName $RG -Location $Location -Name "vnet-vpn-lab08" -AddressPrefix "10.80.0.0/16" -Subnet $gwSubnet
$pip = New-AzPublicIpAddress -ResourceGroupName $RG -Name "pip-vng-lab08" -Location $Location -AllocationMethod Static -Sku Standard
$gwIp = New-AzVirtualNetworkGatewayIpConfig -Name "gwipcfg" -SubnetId (Get-AzVirtualNetworkSubnetConfig -VirtualNetwork $vnet -Name "GatewaySubnet").Id -PublicIpAddressId $pip.Id
$lng = New-AzLocalNetworkGateway -Name "lng-onprem-lab08" -ResourceGroupName $RG -Location $Location -GatewayIpAddress $OnPremIp -AddressPrefix @("192.168.0.0/16")
$vng = New-AzVirtualNetworkGateway -Name "vng-lab08" -ResourceGroupName $RG -Location $Location -IpConfigurations $gwIp -GatewayType Vpn -VpnType RouteBased -GatewaySku VpnGw1 -Asn 65515
New-AzVirtualNetworkGatewayConnection -Name "s2s-conn-lab08" -ResourceGroupName $RG -Location $Location -VirtualNetworkGateway1 $vng -LocalNetworkGateway2 $lng -ConnectionType IPsec -SharedKey (ConvertFrom-SecureString $SharedKey -AsPlainText)
Get-AzVirtualNetworkGatewayConnection -Name "s2s-conn-lab08" -ResourceGroupName $RG | Format-List ConnectionStatus, EgressBytesTransferred, IngressBytesTransferred
```

### IaC implementation

#### Bicep
```bicep
param location string = resourceGroup().location
param gatewayPipName string = 'pip-vng-lab08'

resource pip 'Microsoft.Network/publicIPAddresses@2023-11-01' = {
  name: gatewayPipName
  location: location
  sku: { name: 'Standard' }
  properties: { publicIPAllocationMethod: 'Static' }
}
```

#### Terraform
```hcl
resource "azurerm_virtual_network_gateway" "vpn" {
  name                = "vng-lab08"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  type                = "Vpn"
  vpn_type            = "RouteBased"
  sku                 = "VpnGw1"
}
```

### Validation and success criteria
- VPN gateway reaches **Succeeded** provisioning state.
- Connection status shows **Connected** after on-prem configuration matches.
- Optional BGP settings are documented if dynamic route exchange is required.

### Cleanup

#### Azure CLI
```bash
az group delete -n $RG --yes --no-wait
```

#### PowerShell
```powershell
Remove-AzResourceGroup -Name $RG -Force -AsJob
```

### Exam Tip
Use **VPN Gateway** for encrypted internet-based hybrid connectivity. Use **ExpressRoute** when the scenario prioritizes private connectivity and deterministic performance.

---

## Lab 9: ExpressRoute Configuration (Conceptual + Monitoring)

### Objective
Review ExpressRoute circuit provisioning, deploy an ExpressRoute gateway, connect a VNet, monitor metrics, and design redundancy.

### When to Use This
Use ExpressRoute when the business requires private, dedicated connectivity between Azure and on-premises with higher predictability than internet VPN.

### Exam domain mapping
- **Primary:** Design infrastructure solutions (30-35%)
- **Secondary:** Design business continuity solutions (15-20%)

### Key AZ-305 Concepts
- Provider-provisioned circuit model
- Private peering and route advertisement
- ExpressRoute gateway and FastPath considerations
- Circuit authorization for cross-subscription scenarios
- Dual circuit and dual provider redundancy

### Prerequisites
- Connectivity provider or ExpressRoute Direct option
- Supported peering location and bandwidth plan
- A VNet with `GatewaySubnet`

### Architecture and design rationale
ExpressRoute is not a simple self-service VPN equivalent. The circuit involves provider-side provisioning and peering configuration. For the exam, choose ExpressRoute when requirements emphasize **private path**, **regulatory isolation**, **high throughput**, or **predictable latency**. Design redundancy with dual circuits, dual MSEEs, and optionally geo-diverse peering locations.

### Implementation steps
1. Create the ExpressRoute circuit resource.
2. Share authorization if needed.
3. Deploy an ExpressRoute gateway into the target VNet.
4. Link the circuit to the gateway.
5. Monitor metrics and alerts.
6. Document redundant design choices.

### Full CLI + PowerShell commands

#### Azure CLI
```bash
RG="rg-net-lab09"
LOCATION="eastus"
VNET="vnet-er-lab09"
PIP="pip-ergw-lab09"
ER_CIRCUIT="ercircuit-lab09"
ER_GW="ergw-lab09"
SERVICE_PROVIDER="Equinix"
PEERING_LOCATION="Washington DC"
BANDWIDTH=200

az group create -n $RG -l $LOCATION
az network vnet create -g $RG -n $VNET -l $LOCATION --address-prefix 10.90.0.0/16 --subnet-name GatewaySubnet --subnet-prefix 10.90.255.0/27
az network express-route create -g $RG -n $ER_CIRCUIT --location $LOCATION \
  --bandwidth $BANDWIDTH --peering-location "$PEERING_LOCATION" --provider "$SERVICE_PROVIDER" --sku-family MeteredData --sku-tier Standard --allow-global-reach false
az network express-route auth create -g $RG --circuit-name $ER_CIRCUIT -n auth-lab09
az network public-ip create -g $RG -n $PIP --sku Standard --allocation-method Static
az network vnet-gateway create -g $RG -n $ER_GW -l $LOCATION --public-ip-addresses $PIP --vnet $VNET \
  --gateway-type ExpressRoute --sku ErGw1AZ
az network vpn-connection create -g $RG -n er-conn-lab09 --vnet-gateway1 $ER_GW --express-route-circuit2 $ER_CIRCUIT
az monitor metrics list --resource $(az network express-route show -g $RG -n $ER_CIRCUIT --query id -o tsv) \
  --metric BitsInPerSecond BitsOutPerSecond ArpAvailability
```

#### PowerShell
```powershell
$RG = "rg-net-lab09"
$Location = "eastus"
$Provider = "Equinix"
$PeeringLocation = "Washington DC"

New-AzResourceGroup -Name $RG -Location $Location
$gwSubnet = New-AzVirtualNetworkSubnetConfig -Name "GatewaySubnet" -AddressPrefix "10.90.255.0/27"
$vnet = New-AzVirtualNetwork -ResourceGroupName $RG -Location $Location -Name "vnet-er-lab09" -AddressPrefix "10.90.0.0/16" -Subnet $gwSubnet
$circuit = New-AzExpressRouteCircuit -Name "ercircuit-lab09" -ResourceGroupName $RG -Location $Location -SkuTier Standard -SkuFamily MeteredData -ServiceProviderName $Provider -PeeringLocation $PeeringLocation -BandwidthInMbps 200
New-AzExpressRouteCircuitAuthorization -ExpressRouteCircuit $circuit -Name "auth-lab09"
$pip = New-AzPublicIpAddress -ResourceGroupName $RG -Name "pip-ergw-lab09" -Location $Location -AllocationMethod Static -Sku Standard
$gwIp = New-AzVirtualNetworkGatewayIpConfig -Name "gwipcfg" -SubnetId (Get-AzVirtualNetworkSubnetConfig -VirtualNetwork $vnet -Name "GatewaySubnet").Id -PublicIpAddressId $pip.Id
$ergw = New-AzVirtualNetworkGateway -Name "ergw-lab09" -ResourceGroupName $RG -Location $Location -IpConfigurations $gwIp -GatewayType ExpressRoute -GatewaySku ErGw1AZ
New-AzVirtualNetworkGatewayConnection -Name "er-conn-lab09" -ResourceGroupName $RG -Location $Location -VirtualNetworkGateway1 $ergw -PeerId $circuit.Id -ConnectionType ExpressRoute
Get-AzMetric -ResourceId $circuit.Id -MetricName BitsInPerSecond,BitsOutPerSecond,ArpAvailability -DetailedOutput
```

### IaC implementation

#### Bicep
```bicep
param location string = resourceGroup().location

resource circuit 'Microsoft.Network/expressRouteCircuits@2023-11-01' = {
  name: 'ercircuit-lab09'
  location: location
  sku: {
    tier: 'Standard'
    family: 'MeteredData'
    name: 'Standard_MeteredData'
  }
  properties: {
    serviceProviderProperties: {
      serviceProviderName: 'Equinix'
      peeringLocation: 'Washington DC'
      bandwidthInMbps: 200
    }
    allowClassicOperations: false
  }
}
```

#### Terraform
```hcl
resource "azurerm_express_route_circuit" "er" {
  name                  = "ercircuit-lab09"
  resource_group_name   = azurerm_resource_group.rg.name
  location              = azurerm_resource_group.rg.location
  service_provider_name = "Equinix"
  peering_location      = "Washington DC"
  bandwidth_in_mbps     = 200

  sku {
    tier   = "Standard"
    family = "MeteredData"
  }
}
```

### Validation and success criteria
- ExpressRoute circuit exists and shows provider status progression.
- ExpressRoute gateway is deployed successfully.
- Metrics can be queried for circuit health and throughput.
- Redundant design includes dual circuits or geo-diverse peering locations.

### Cleanup

#### Azure CLI
```bash
az group delete -n $RG --yes --no-wait
```

#### PowerShell
```powershell
Remove-AzResourceGroup -Name $RG -Force -AsJob
```

### Exam Tip
If the question says **private connectivity**, **large sustained throughput**, or **compliance over non-internet path**, prefer **ExpressRoute** over VPN.

---

## Lab 10: Azure Bastion

### Objective
Deploy Azure Bastion Standard, connect to a VM without a public IP, use native client connectivity, and review IP-based connection capability.

### When to Use This
Use Azure Bastion when administrators need secure RDP/SSH access to VMs without exposing public IP addresses.

### Exam domain mapping
- **Primary:** Design infrastructure solutions (30-35%)
- **Secondary:** Design identity, governance, and monitoring solutions (25-30%)

### Key AZ-305 Concepts
- Bastion Standard versus Basic
- Browser-based access versus native client
- No public IP on target VM
- Dedicated `AzureBastionSubnet`
- Security improvement over jump boxes with public IPs

### Prerequisites
- A VNet with `AzureBastionSubnet` sized appropriately
- A test VM in a workload subnet with no public IP
- NSG rules that allow RDP or SSH from Bastion inside the VNet

### Architecture and design rationale
Bastion reduces exposure by removing direct public management ports from VMs. For the exam, Bastion is usually the best answer when the requirement is **secure administrative access without public IPs**. Do not confuse it with a VPN gateway or firewall.

### Implementation steps
1. Create a VNet with workload and Bastion subnets.
2. Deploy a VM without public IP.
3. Deploy Bastion Standard.
4. Connect through portal or native client.
5. Review IP-based connection for approved internal addresses.

### Full CLI + PowerShell commands

#### Azure CLI
```bash
RG="rg-net-lab10"
LOCATION="eastus"
VNET="vnet-bastion-lab10"
VM="vm-bastion-target"
BASTION="bas-lab10"
PIP="pip-bastion-lab10"
ADMINUSER="azureuser"

az group create -n $RG -l $LOCATION
az network vnet create -g $RG -n $VNET -l $LOCATION --address-prefix 10.100.0.0/16 --subnet-name AzureBastionSubnet --subnet-prefix 10.100.1.0/26
az network vnet subnet create -g $RG --vnet-name $VNET -n WorkloadSubnet --address-prefixes 10.100.2.0/24
az vm create -g $RG -n $VM --image Ubuntu2204 --admin-username $ADMINUSER --generate-ssh-keys \
  --public-ip-address "" --vnet-name $VNET --subnet WorkloadSubnet
az network public-ip create -g $RG -n $PIP --sku Standard --allocation-method Static
az network bastion create -g $RG -n $BASTION -l $LOCATION --public-ip-address $PIP --vnet-name $VNET --sku Standard
az network bastion ssh -g $RG -n $BASTION --target-resource-id $(az vm show -g $RG -n $VM --query id -o tsv) --auth-type ssh-key --username $ADMINUSER --ssh-key ~/.ssh/id_rsa
# IP-based connection example
az network bastion tunnel -g $RG -n $BASTION --target-ip-address 10.100.2.4 --resource-port 22 --port 50022
```

#### PowerShell
```powershell
$RG = "rg-net-lab10"
$Location = "eastus"
$AdminUser = "azureuser"

New-AzResourceGroup -Name $RG -Location $Location
$bastionSubnet = New-AzVirtualNetworkSubnetConfig -Name "AzureBastionSubnet" -AddressPrefix "10.100.1.0/26"
$workloadSubnet = New-AzVirtualNetworkSubnetConfig -Name "WorkloadSubnet" -AddressPrefix "10.100.2.0/24"
$vnet = New-AzVirtualNetwork -ResourceGroupName $RG -Location $Location -Name "vnet-bastion-lab10" -AddressPrefix "10.100.0.0/16" -Subnet $bastionSubnet,$workloadSubnet
$pip = New-AzPublicIpAddress -ResourceGroupName $RG -Name "pip-bastion-lab10" -Location $Location -AllocationMethod Static -Sku Standard
$vmCred = Get-Credential -UserName $AdminUser -Message "Enter the VM credentials for the Bastion target VM"
New-AzVM -ResourceGroupName $RG -Name "vm-bastion-target" -Location $Location -VirtualNetworkName $vnet.Name -SubnetName "WorkloadSubnet" -Credential $vmCred -Image Ubuntu2204 -PublicIpAddressName "" 
New-AzBastion -ResourceGroupName $RG -Name "bas-lab10" -Location $Location -PublicIpAddressRgName $RG -PublicIpAddressName "pip-bastion-lab10" -VirtualNetworkRgName $RG -VirtualNetworkName $vnet.Name -Sku Standard

# Native client examples from PowerShell
az network bastion ssh -g $RG -n bas-lab10 --target-resource-id (Get-AzVM -ResourceGroupName $RG -Name "vm-bastion-target").Id --auth-type password --username $AdminUser
az network bastion tunnel -g $RG -n bas-lab10 --target-ip-address 10.100.2.4 --resource-port 22 --port 50022
```

### IaC implementation

#### Bicep
```bicep
param location string = resourceGroup().location

resource bastionPip 'Microsoft.Network/publicIPAddresses@2023-11-01' = {
  name: 'pip-bastion-lab10'
  location: location
  sku: { name: 'Standard' }
  properties: { publicIPAllocationMethod: 'Static' }
}

resource bastion 'Microsoft.Network/bastionHosts@2023-11-01' = {
  name: 'bas-lab10'
  location: location
  sku: { name: 'Standard' }
}
```

#### Terraform
```hcl
resource "azurerm_bastion_host" "bastion" {
  name                = "bas-lab10"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  sku                 = "Standard"
}
```

### Validation and success criteria
- Target VM has no public IP.
- Bastion connects successfully over browser or native client.
- Standard SKU capabilities such as native client or tunneling are available.

### Cleanup

#### Azure CLI
```bash
az group delete -n $RG --yes --no-wait
```

#### PowerShell
```powershell
Remove-AzResourceGroup -Name $RG -Force -AsJob
```

### Exam Tip
If the requirement is **RDP/SSH to Azure VMs without public IPs**, Bastion is usually the cleanest managed answer.

---

## Lab 11: Virtual WAN

### Objective
Deploy a Standard Virtual WAN, create a virtual hub, connect spokes, onboard a VPN site, and review hub-to-hub connectivity.

### When to Use This
Use Virtual WAN for large-scale branch connectivity, simplified global transit, and centralized connectivity across many sites and VNets.

### Exam domain mapping
- **Primary:** Design infrastructure solutions (30-35%)
- **Secondary:** Design business continuity solutions (15-20%)

### Key AZ-305 Concepts
- Virtual WAN versus custom hub-spoke
- Virtual hub as managed transit router
- Branch, VNet, and user VPN connectivity
- Hub-to-hub transit across regions
- Operational simplification at scale

### Prerequisites
- At least one branch site definition with public IP and prefixes
- One or more spoke VNets to attach
- Virtual WAN Standard SKU for advanced connectivity features

### Architecture and design rationale
Virtual WAN is best when the environment grows beyond a few VNets and branches. It reduces operational overhead compared with building every transit construct manually. For the exam, choose Virtual WAN when the requirement emphasizes **many branches**, **multiple regions**, and **managed transit**.

### Implementation steps
1. Create a Standard Virtual WAN.
2. Create a virtual hub.
3. Connect spoke VNets.
4. Create a VPN site and VPN gateway.
5. Create site connections.
6. Add a second hub if you need hub-to-hub connectivity.

### Full CLI + PowerShell commands

#### Azure CLI
```bash
RG="rg-net-lab11"
LOCATION="eastus"
VWAN="vwan-lab11"
VHUB="vhub-eastus-lab11"
SPOKE="vnet-spoke-lab11"
SITE="vpnsite-branch1-lab11"
VPNGW="vpngw-vhub-lab11"
BRANCH_IP="<branch-public-ip>"
BRANCH_PREFIX="172.16.0.0/16"

az group create -n $RG -l $LOCATION
az network vwan create -g $RG -n $VWAN -l $LOCATION --type Standard
az network vhub create -g $RG -n $VHUB -l $LOCATION --address-prefix 10.110.0.0/24 --vwan $VWAN
az network vnet create -g $RG -n $SPOKE -l $LOCATION --address-prefix 10.111.0.0/16 --subnet-name AppSubnet --subnet-prefix 10.111.1.0/24
az network vhub connection create -g $RG --vhub-name $VHUB -n spoke-conn-lab11 --remote-vnet $SPOKE --internet-security false
az network vpn-site create -g $RG -n $SITE -l $LOCATION --virtual-wan $VWAN --ip-address $BRANCH_IP --address-prefixes $BRANCH_PREFIX
az network vpn-gateway create -g $RG -n $VPNGW -l $LOCATION --vhub $VHUB --scale-unit 1
az network vpn-connection create -g $RG --gateway-name $VPNGW -n branch1-conn --remote-vpn-site $SITE --shared-key Az305VwanKey!
# Optional: create a second hub in another region and connect both hubs through the same Virtual WAN
```

#### PowerShell
```powershell
$RG = "rg-net-lab11"
$Location = "eastus"

New-AzResourceGroup -Name $RG -Location $Location
$vwan = New-AzVirtualWan -Name "vwan-lab11" -ResourceGroupName $RG -Location $Location -Type Standard
$vhub = New-AzVirtualHub -Name "vhub-eastus-lab11" -ResourceGroupName $RG -Location $Location -AddressPrefix "10.110.0.0/24" -VirtualWan $vwan
$spokeSubnet = New-AzVirtualNetworkSubnetConfig -Name "AppSubnet" -AddressPrefix "10.111.1.0/24"
$spoke = New-AzVirtualNetwork -ResourceGroupName $RG -Location $Location -Name "vnet-spoke-lab11" -AddressPrefix "10.111.0.0/16" -Subnet $spokeSubnet
New-AzVirtualHubVnetConnection -ResourceGroupName $RG -VirtualHubName $vhub.Name -Name "spoke-conn-lab11" -RemoteVirtualNetwork $spoke
$site = New-AzVpnSite -ResourceGroupName $RG -Name "vpnsite-branch1-lab11" -Location $Location -VirtualWan $vwan -IpAddress "<branch-public-ip>" -AddressSpace @("172.16.0.0/16")
$vpnGw = New-AzVpnGateway -ResourceGroupName $RG -Name "vpngw-vhub-lab11" -Location $Location -VirtualHub $vhub -VpnGatewayScaleUnit 1
New-AzVpnConnection -ResourceGroupName $RG -ParentResourceName $vpnGw.Name -Name "branch1-conn" -RemoteVpnSiteId $site.Id -SharedKey "Az305VwanKey!"
```

### IaC implementation

#### Bicep
```bicep
param location string = resourceGroup().location

resource vwan 'Microsoft.Network/virtualWans@2023-11-01' = {
  name: 'vwan-lab11'
  location: location
  properties: {
    type: 'Standard'
  }
}

resource vhub 'Microsoft.Network/virtualHubs@2023-11-01' = {
  name: 'vhub-eastus-lab11'
  location: location
  properties: {
    addressPrefix: '10.110.0.0/24'
    virtualWan: { id: vwan.id }
  }
}
```

#### Terraform
```hcl
resource "azurerm_virtual_wan" "vwan" {
  name                = "vwan-lab11"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  type                = "Standard"
}

resource "azurerm_virtual_hub" "hub" {
  name                = "vhub-eastus-lab11"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  virtual_wan_id      = azurerm_virtual_wan.vwan.id
  address_prefix      = "10.110.0.0/24"
}
```

### Validation and success criteria
- Virtual WAN is Standard SKU.
- Virtual hub exists and spoke VNet connection is in connected state.
- VPN site and VPN gateway are deployed.
- You can describe why Virtual WAN simplifies large multi-site environments.

### Cleanup

#### Azure CLI
```bash
az group delete -n $RG --yes --no-wait
```

#### PowerShell
```powershell
Remove-AzResourceGroup -Name $RG -Force -AsJob
```

### Exam Tip
If the scenario includes **many branches**, **multiple regions**, and **centralized managed transit**, consider **Virtual WAN** before building a custom hub-spoke fabric yourself.

---

## Lab 12: Networking Decision Exercise

### Objective
Practice choosing the correct Azure networking service or topology for common AZ-305 scenarios.

### When to Use This
Use this lab before the exam to sharpen service-selection speed and eliminate common distractors.

### Exam domain mapping
- **Primary:** Design infrastructure solutions (30-35%)
- **Secondary:** Design business continuity solutions (15-20%)

### Key AZ-305 Concepts
- Matching service capability to requirement
- Global versus regional traffic management
- Private versus public connectivity
- East-west segmentation versus north-south publishing
- Security, cost, and operations tradeoffs

### Prerequisites
- Familiarity with Labs 1-11
- Optional access to an Azure subscription for reviewing deployed network resources

### Architecture and design rationale
AZ-305 networking questions often test whether you can separate similar services. The fastest path is to classify the scenario first: **private versus public**, **global versus regional**, **L4 versus L7**, **branch/hybrid versus Azure-only**, and **inspection versus acceleration**.

### Implementation steps
1. Read each scenario.
2. Identify the core requirement.
3. Pick the service or topology.
4. Compare your answer to the key and reasoning.

### Full CLI + PowerShell commands

#### Azure CLI
```bash
# Optional discovery commands to inventory a live subscription while practicing design choices
az network vnet list -o table
az network lb list -o table
az network application-gateway list -o table
az network front-door list -o table
az network bastion list -o table
az network firewall list -o table
```

#### PowerShell
```powershell
Get-AzVirtualNetwork | Format-Table Name, ResourceGroupName, Location
Get-AzLoadBalancer | Format-Table Name, ResourceGroupName, Sku
Get-AzApplicationGateway | Format-Table Name, ResourceGroupName, Sku
Get-AzBastion | Format-Table Name, ResourceGroupName, Location
Get-AzFirewall | Format-Table Name, ResourceGroupName, Sku
```

### IaC implementation

#### Bicep
```bicep
// No deployment required for this decision lab.
```

#### Terraform
```hcl
# No deployment required for this decision lab.
```

### Validation and success criteria
- You can justify the chosen service in one sentence.
- You can identify at least one wrong-but-plausible distractor for each scenario.
- You can explain cost, security, or operational tradeoffs.

### Cleanup
No resources are required for this exercise. If you created a temporary resource group for practice, delete it afterward.

### Exam Tip
On AZ-305, the right answer is usually the service that meets the requirement with the **least operational complexity** and **strongest native security alignment**.

### Scenarios

| # | Scenario | Your task |
|---|---|---|
| 1 | A company wants to centralize Azure Firewall, VPN Gateway, and Bastion while keeping app teams isolated by workload. | Identify the topology. |
| 2 | A web app must route `/images` and `/api` to different backends and block common web exploits. | Identify the service. |
| 3 | A multinational company needs a single global web entry point with acceleration and edge WAF. | Identify the service. |
| 4 | A banking app must access Azure SQL privately and disable the public endpoint. | Identify the service. |
| 5 | A branch office needs encrypted hybrid connectivity over the internet at lower cost than private circuits. | Identify the service. |
| 6 | A manufacturing customer needs dedicated private connectivity with predictable latency and compliance requirements. | Identify the service. |
| 7 | Administrators need SSH and RDP access to VMs without assigning public IP addresses. | Identify the service. |
| 8 | An application needs TCP load balancing for custom protocol traffic on port 9000. | Identify the service. |
| 9 | An enterprise has hundreds of branches and wants managed transit across regions with less custom networking overhead. | Identify the service. |
| 10 | Security needs workload rules based on application tier membership rather than static IPs. | Identify the service. |

### Answer key with reasoning

| # | Best answer | Reasoning | Common distractor |
|---|---|---|---|
| 1 | Hub-spoke topology | Centralized shared services plus isolated spokes is the classic enterprise design. | Full mesh peering adds complexity and does not scale well. |
| 2 | Application Gateway with WAF | Path-based routing and WAF are Layer 7 requirements. | Load Balancer cannot do URL routing or WAF. |
| 3 | Azure Front Door | Global anycast, acceleration, and edge WAF point to Front Door. | Traffic Manager has DNS load balancing but no WAF or TLS edge offload. |
| 4 | Private Endpoint + Private DNS | It provides private IP access to PaaS and supports disabling public access. | Service endpoints do not privately expose the resource itself. |
| 5 | Site-to-Site VPN | Hybrid encrypted internet path at moderate cost fits S2S VPN. | ExpressRoute is more expensive and used for dedicated private circuits. |
| 6 | ExpressRoute | Dedicated private connectivity and deterministic performance match ExpressRoute. | VPN Gateway uses internet transport. |
| 7 | Azure Bastion | Secure admin access without public IPs is Bastion's core use case. | Jump boxes increase attack surface and operations. |
| 8 | Azure Load Balancer | TCP/UDP distribution is an L4 requirement. | Application Gateway is optimized for HTTP(S), not arbitrary TCP. |
| 9 | Virtual WAN | It is designed for large branch connectivity with managed hubs and transit. | Custom hub-spoke is possible but higher-ops at large scale. |
| 10 | Application Security Groups with NSGs | ASGs let rules follow role membership instead of hard-coded IPs. | Plain NSGs with IP rules do not scale as cleanly. |

---

## Decision summary table

| Requirement pattern | Best-fit Azure service/pattern | Why |
|---|---|---|
| Enterprise segmentation with centralized security and shared services | Hub-spoke | Separates workloads while centralizing connectivity and inspection |
| East-west subnet or tier control | NSG + ASG | Lightweight micro-segmentation inside VNets |
| L4 TCP/UDP load balancing | Azure Load Balancer | High-performance transport-layer distribution |
| Regional L7 routing + WAF | Application Gateway WAF_v2 | Path/host routing, SSL offload, WAF |
| Global web entry with acceleration | Azure Front Door | Edge POPs, WAF, caching, anycast |
| Centralized outbound and inbound network security | Azure Firewall | Managed firewall with FQDN filtering and DNAT |
| Private PaaS connectivity | Private Endpoint | Private Link with public access disablement |
| Hybrid connectivity over internet | Site-to-Site VPN | Encrypted, lower-cost hybrid option |
| Dedicated private hybrid connectivity | ExpressRoute | Private path, predictable performance |
| Admin access without public IPs | Azure Bastion | Secure management access |
| Large-scale branch transit | Virtual WAN | Managed hubs and simplified global branch connectivity |

## Load balancer comparison matrix

| Service | Layer | Scope | Protocols | WAF | Path-based routing | Private backends | Best for |
|---|---|---|---|---|---|---|---|
| Azure Load Balancer | L4 | Regional | TCP/UDP | No | No | Yes | Non-HTTP workloads, internal or public transport balancing |
| Application Gateway | L7 | Regional | HTTP/HTTPS/WebSocket | Yes | Yes | Yes | Regional web apps needing WAF and SSL offload |
| Azure Front Door | L7 | Global | HTTP/HTTPS | Yes | Yes | Public origins or internet-reachable regional entry points | Global web applications and edge acceleration |
| Traffic Manager | DNS | Global | Any (DNS-based) | No | No | Indirect | DNS failover and simple global distribution |

## Exam-style review questions

1. A company needs a global web entry point with WAF and caching but still wants regional application gateways behind it. Which service should be internet-facing first, and why?
2. When should you choose Private Endpoint instead of service endpoints for Azure Storage or SQL?
3. Why is Azure Load Balancer the wrong answer for URL-based routing with SSL termination?
4. What is the primary reason hub-spoke designs use UDRs with Azure Firewall for spoke-to-spoke traffic?
5. In which scenario would Virtual WAN be preferred over a custom hub-spoke topology?
