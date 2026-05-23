# Azure High Availability Hands-On Labs (AZ-305)

> 📖 **Cheat Sheet:** [Azure HA](../Azure-HA.md)

> **Primary exam domain:** Design business continuity solutions (15-20%)  
> **Secondary domains:** Design infrastructure solutions (30-35%), Design data storage solutions (20-25%)  
> **Tools used:** Azure CLI, Azure PowerShell, kubectl, curl, Azure portal validation  
> **Important:** Use a non-production subscription. Several labs create billable compute, networking, database, and AKS resources.

---

## Lab 1: Availability Zones Deployment

### Objective
Deploy three web VMs across three Availability Zones, place them behind a zone-redundant Standard Load Balancer, and validate application availability during a simulated zonal failure.

### When to Use
Use this pattern for **regional HA for IaaS VMs** when the application must survive the loss of a datacenter within a supported Azure region.

### Key AZ-305 Concepts
- Availability Zones protect against datacenter-level failure inside a region.
- Standard Load Balancer supports zone-redundant frontend IPs.
- Stateless web tiers are easier to keep highly available.
- Zone-redundant design improves SLA compared to a single VM or Availability Set.

### Exam Domain Mapping
- **Primary:** Design business continuity solutions (15-20%)
- **Secondary:** Design infrastructure solutions (30-35%)

### Prerequisites
- Owner or Contributor on the subscription/resource group
- Zone-enabled region such as `eastus2`, `westeurope`, or `centralus`
- Azure CLI logged in with `az login`
- Azure PowerShell logged in with `Connect-AzAccount`
- SSH key available for Linux VM deployment

### Architecture and Design Rationale
This lab uses one VM per zone and a Standard Load Balancer with a zone-redundant public IP. This is the right choice when you need **in-region HA** without the complexity of multi-region deployment. For the exam, remember that **Availability Sets protect from rack/host/update issues**, while **Availability Zones protect from full datacenter failure**.

### Implementation Steps
1. Create the resource group, VNet, NSG, and subnet.
2. Create a Standard public IP and Standard Load Balancer.
3. Deploy one VM in each zone.
4. Install a lightweight web server on each VM.
5. Add NICs to the load balancer backend pool and create health probes/rules.
6. Simulate a zonal workload failure by stopping one zonal VM.
7. Verify the application still responds through the load balancer.

### Azure CLI Commands
```bash
RG="rg-ha-lab1"
LOCATION="eastus2"
VNET="vnet-ha-lab1"
SUBNET="snet-web"
NSG="nsg-ha-lab1"
PIP="pip-ha-lab1"
LB="lb-ha-lab1"
BEPOOL="bepool-ha"
PROBE="probe-http"
LBRULE="rule-http"
ADMINUSER="azureuser"
IMAGE="Ubuntu2204"

az group create --name $RG --location $LOCATION

az network vnet create \
  --resource-group $RG \
  --name $VNET \
  --location $LOCATION \
  --address-prefix 10.10.0.0/16 \
  --subnet-name $SUBNET \
  --subnet-prefix 10.10.1.0/24

az network nsg create --resource-group $RG --name $NSG --location $LOCATION
az network nsg rule create \
  --resource-group $RG \
  --nsg-name $NSG \
  --name allow-http \
  --priority 100 \
  --direction Inbound \
  --access Allow \
  --protocol Tcp \
  --destination-port-ranges 80

az network vnet subnet update \
  --resource-group $RG \
  --vnet-name $VNET \
  --name $SUBNET \
  --network-security-group $NSG

az network public-ip create \
  --resource-group $RG \
  --name $PIP \
  --location $LOCATION \
  --sku Standard \
  --version IPv4 \
  --zone 1 2 3

az network lb create \
  --resource-group $RG \
  --name $LB \
  --location $LOCATION \
  --sku Standard \
  --public-ip-address $PIP \
  --frontend-ip-name fe-ha \
  --backend-pool-name $BEPOOL

az network lb probe create \
  --resource-group $RG \
  --lb-name $LB \
  --name $PROBE \
  --protocol Tcp \
  --port 80

az network lb rule create \
  --resource-group $RG \
  --lb-name $LB \
  --name $LBRULE \
  --protocol Tcp \
  --frontend-port 80 \
  --backend-port 80 \
  --frontend-ip-name fe-ha \
  --backend-pool-name $BEPOOL \
  --probe-name $PROBE

for ZONE in 1 2 3; do
  VMNAME="vm-zone-$ZONE"
  NICNAME="nic-zone-$ZONE"

  az network nic create \
    --resource-group $RG \
    --name $NICNAME \
    --vnet-name $VNET \
    --subnet $SUBNET \
    --lb-name $LB \
    --lb-address-pools $BEPOOL

  az vm create \
    --resource-group $RG \
    --name $VMNAME \
    --nics $NICNAME \
    --image $IMAGE \
    --admin-username $ADMINUSER \
    --generate-ssh-keys \
    --size Standard_B2s \
    --zone $ZONE

  az vm extension set \
    --resource-group $RG \
    --vm-name $VMNAME \
    --name customScript \
    --publisher Microsoft.Azure.Extensions \
    --settings '{"commandToExecute":"bash -c \"apt-get update && apt-get install -y nginx && echo Zone '$ZONE' > /var/www/html/index.html && systemctl restart nginx\""}'
done
```

### PowerShell Commands
```powershell
$RG = "rg-ha-lab1"
$Location = "eastus2"
$VnetName = "vnet-ha-lab1"
$SubnetName = "snet-web"
$NsgName = "nsg-ha-lab1"
$PipName = "pip-ha-lab1"
$LbName = "lb-ha-lab1"
$AdminUser = "azureuser"
$Cred = Get-Credential -UserName $AdminUser -Message "Enter the local VM password or SSH-backed credential"

New-AzResourceGroup -Name $RG -Location $Location

$nsg = New-AzNetworkSecurityGroup -ResourceGroupName $RG -Location $Location -Name $NsgName
$nsg | Add-AzNetworkSecurityRuleConfig -Name allow-http -Description "Allow HTTP" -Access Allow -Protocol Tcp -Direction Inbound -Priority 100 -SourceAddressPrefix * -SourcePortRange * -DestinationAddressPrefix * -DestinationPortRange 80 | Set-AzNetworkSecurityGroup

$subnet = New-AzVirtualNetworkSubnetConfig -Name $SubnetName -AddressPrefix "10.10.1.0/24" -NetworkSecurityGroup $nsg
$vnet = New-AzVirtualNetwork -Name $VnetName -ResourceGroupName $RG -Location $Location -AddressPrefix "10.10.0.0/16" -Subnet $subnet

$pip = New-AzPublicIpAddress -ResourceGroupName $RG -Location $Location -Name $PipName -Sku Standard -AllocationMethod Static -Zone 1,2,3
$fe = New-AzLoadBalancerFrontendIpConfig -Name fe-ha -PublicIpAddress $pip
$be = New-AzLoadBalancerBackendAddressPoolConfig -Name bepool-ha
$probe = New-AzLoadBalancerProbeConfig -Name probe-http -Protocol Tcp -Port 80 -IntervalInSeconds 5 -ProbeCount 2
$rule = New-AzLoadBalancerRuleConfig -Name rule-http -Protocol Tcp -FrontendPort 80 -BackendPort 80 -FrontendIpConfiguration $fe -BackendAddressPool $be -Probe $probe
$lb = New-AzLoadBalancer -ResourceGroupName $RG -Name $LbName -Location $Location -Sku Standard -FrontendIpConfiguration $fe -BackendAddressPool $be -Probe $probe -LoadBalancingRule $rule

foreach ($Zone in 1..3) {
    $NicName = "nic-zone-$Zone"
    $VmName = "vm-zone-$Zone"
    $nic = New-AzNetworkInterface -Name $NicName -ResourceGroupName $RG -Location $Location -SubnetId $vnet.Subnets[0].Id -LoadBalancerBackendAddressPool $lb.BackendAddressPools[0]
    $vmConfig = New-AzVMConfig -VMName $VmName -VMSize "Standard_B2s" -Zone $Zone |
      Set-AzVMOperatingSystem -Linux -ComputerName $VmName -Credential $Cred -DisablePasswordAuthentication $false |
      Set-AzVMSourceImage -PublisherName Canonical -Offer 0001-com-ubuntu-server-jammy -Skus 22_04-lts-gen2 -Version latest |
      Add-AzVMNetworkInterface -Id $nic.Id
    New-AzVM -ResourceGroupName $RG -Location $Location -VM $vmConfig
    Set-AzVMCustomScriptExtension -ResourceGroupName $RG -VMName $VmName -Name webinit -Location $Location -FileUri @() -Run "bash -c 'apt-get update && apt-get install -y nginx && echo Zone $Zone > /var/www/html/index.html && systemctl restart nginx'"
}
```

### IaC Implementation
#### Bicep
```bicep
param location string = resourceGroup().location
param vmSku string = 'Standard_B2s'
param adminUsername string = 'azureuser'
param adminPassword string
var zones = [ '1' '2' '3' ]

resource pip 'Microsoft.Network/publicIPAddresses@2023-09-01' = {
  name: 'pip-ha-lab1'
  location: location
  sku: { name: 'Standard' }
  zones: zones
  properties: { publicIPAllocationMethod: 'Static' }
}
```

#### Terraform
```hcl
resource "azurerm_public_ip" "ha" {
  name                = "pip-ha-lab1"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  allocation_method   = "Static"
  sku                 = "Standard"
  zones               = ["1", "2", "3"]
}
```

### Validation and Success Criteria
- `az vm list -g $RG -d -o table` shows one VM in each zone.
- `az network lb show -g $RG -n $LB --query "frontendIpConfigurations[].zones" -o tsv` returns zones.
- Browse or `curl` the load balancer public IP and see responses.
- Stop one zonal VM and confirm the load balancer still serves traffic from remaining VMs.

### Verification
```bash
LBIP=$(az network public-ip show -g $RG -n $PIP --query ipAddress -o tsv)
for i in 1 2 3 4 5; do curl -s http://$LBIP; done
az vm deallocate --resource-group $RG --name vm-zone-1
for i in 1 2 3 4 5; do curl -s http://$LBIP; done
```

```powershell
$LbIp = (Get-AzPublicIpAddress -ResourceGroupName $RG -Name $PipName).IpAddress
1..5 | ForEach-Object { Invoke-WebRequest -Uri "http://$LbIp" -UseBasicParsing | Select-Object -ExpandProperty Content }
Stop-AzVM -ResourceGroupName $RG -Name "vm-zone-1" -Force
1..5 | ForEach-Object { Invoke-WebRequest -Uri "http://$LbIp" -UseBasicParsing | Select-Object -ExpandProperty Content }
```

### Cleanup
```bash
az group delete --name $RG --yes --no-wait
```

```powershell
Remove-AzResourceGroup -Name $RG -Force -AsJob
```

### Exam Tip
**Availability Zones** are the default AZ-305 answer for **regional HA against datacenter failure**. Use **multi-region** only when the requirement is to survive a **regional outage**, not just a zonal outage.

---

## Lab 2: Availability Sets

### Objective
Create an Availability Set, deploy two VMs into it, and compare the HA model to Availability Zones.

### When to Use
Use this pattern for **HA within a single datacenter-style deployment** when zones are unavailable or the workload cannot use zonal placement.

### Key AZ-305 Concepts
- Fault domains reduce impact from rack or power failures.
- Update domains reduce planned maintenance impact.
- Availability Sets do **not** protect against complete datacenter loss.
- Managed disks are required for modern Availability Set scenarios.

### Exam Domain Mapping
- **Primary:** Design business continuity solutions (15-20%)
- **Secondary:** Design infrastructure solutions (30-35%)

### Prerequisites
- Subscription permissions to create compute and network resources
- Azure CLI and Azure PowerShell authenticated
- Region that supports Availability Sets

### Architecture and Design Rationale
Availability Sets spread VMs across **fault domains** and **update domains** in the same datacenter scope. This improves planned and unplanned host resilience but does not deliver the same failure isolation as zones. For the exam, choose Availability Sets only when the problem stays **inside one datacenter** or when zones are unavailable.

### Implementation Steps
1. Create the resource group, VNet, NSG, subnet, and Availability Set.
2. Deploy two VMs into the Availability Set.
3. Install a web server on each VM.
4. Review fault domain and update domain placement.
5. Compare the design to Lab 1.

### Azure CLI Commands
```bash
RG="rg-ha-lab2"
LOCATION="eastus2"
AVSET="as-ha-lab2"
VNET="vnet-ha-lab2"
SUBNET="snet-web"
NSG="nsg-ha-lab2"
IMAGE="Ubuntu2204"
ADMINUSER="azureuser"

az group create --name $RG --location $LOCATION
az network vnet create --resource-group $RG --name $VNET --location $LOCATION --address-prefix 10.20.0.0/16 --subnet-name $SUBNET --subnet-prefix 10.20.1.0/24
az network nsg create --resource-group $RG --name $NSG --location $LOCATION
az network nsg rule create --resource-group $RG --nsg-name $NSG --name allow-http --priority 100 --direction Inbound --access Allow --protocol Tcp --destination-port-ranges 80
az network vnet subnet update --resource-group $RG --vnet-name $VNET --name $SUBNET --network-security-group $NSG

az vm availability-set create \
  --resource-group $RG \
  --name $AVSET \
  --location $LOCATION \
  --platform-fault-domain-count 2 \
  --platform-update-domain-count 5 \
  --managed true

for VM in 1 2; do
  az vm create \
    --resource-group $RG \
    --name vm-as-$VM \
    --availability-set $AVSET \
    --vnet-name $VNET \
    --subnet $SUBNET \
    --image $IMAGE \
    --size Standard_B2s \
    --admin-username $ADMINUSER \
    --generate-ssh-keys

  az vm extension set \
    --resource-group $RG \
    --vm-name vm-as-$VM \
    --name customScript \
    --publisher Microsoft.Azure.Extensions \
    --settings '{"commandToExecute":"bash -c \"apt-get update && apt-get install -y nginx && echo AvailabilitySet-VM-'$VM' > /var/www/html/index.html && systemctl restart nginx\""}'
done

az vm availability-set show \
  --resource-group $RG \
  --name $AVSET \
  --query "{FaultDomains:platformFaultDomainCount,UpdateDomains:platformUpdateDomainCount}" \
  -o table
```

### PowerShell Commands
```powershell
$RG = "rg-ha-lab2"
$Location = "eastus2"
$AvSet = "as-ha-lab2"
$VnetName = "vnet-ha-lab2"
$SubnetName = "snet-web"
$NsgName = "nsg-ha-lab2"
$AdminUser = "azureuser"
$Cred = Get-Credential -UserName $AdminUser

New-AzResourceGroup -Name $RG -Location $Location
$nsg = New-AzNetworkSecurityGroup -ResourceGroupName $RG -Name $NsgName -Location $Location
$nsg | Add-AzNetworkSecurityRuleConfig -Name allow-http -Access Allow -Protocol Tcp -Direction Inbound -Priority 100 -SourceAddressPrefix * -SourcePortRange * -DestinationAddressPrefix * -DestinationPortRange 80 | Set-AzNetworkSecurityGroup
$subnet = New-AzVirtualNetworkSubnetConfig -Name $SubnetName -AddressPrefix "10.20.1.0/24" -NetworkSecurityGroup $nsg
$vnet = New-AzVirtualNetwork -Name $VnetName -ResourceGroupName $RG -Location $Location -AddressPrefix "10.20.0.0/16" -Subnet $subnet
$avset = New-AzAvailabilitySet -ResourceGroupName $RG -Name $AvSet -Location $Location -Sku aligned -PlatformFaultDomainCount 2 -PlatformUpdateDomainCount 5

foreach ($Vm in 1..2) {
    $nic = New-AzNetworkInterface -Name "nic-as-$Vm" -ResourceGroupName $RG -Location $Location -SubnetId $vnet.Subnets[0].Id
    $vmConfig = New-AzVMConfig -VMName "vm-as-$Vm" -VMSize "Standard_B2s" -AvailabilitySetId $avset.Id |
      Set-AzVMOperatingSystem -Linux -ComputerName "vm-as-$Vm" -Credential $Cred -DisablePasswordAuthentication $false |
      Set-AzVMSourceImage -PublisherName Canonical -Offer 0001-com-ubuntu-server-jammy -Skus 22_04-lts-gen2 -Version latest |
      Add-AzVMNetworkInterface -Id $nic.Id
    New-AzVM -ResourceGroupName $RG -Location $Location -VM $vmConfig
}

Get-AzAvailabilitySet -ResourceGroupName $RG -Name $AvSet | Select-Object Name, PlatformFaultDomainCount, PlatformUpdateDomainCount
```

### IaC Implementation
#### Bicep
```bicep
resource avset 'Microsoft.Compute/availabilitySets@2023-09-01' = {
  name: 'as-ha-lab2'
  location: resourceGroup().location
  sku: { name: 'Aligned' }
  properties: {
    platformFaultDomainCount: 2
    platformUpdateDomainCount: 5
  }
}
```

#### Terraform
```hcl
resource "azurerm_availability_set" "ha" {
  name                         = "as-ha-lab2"
  location                     = azurerm_resource_group.rg.location
  resource_group_name          = azurerm_resource_group.rg.name
  managed                      = true
  platform_fault_domain_count  = 2
  platform_update_domain_count = 5
}
```

### Validation and Success Criteria
- Both VMs are deployed in the Availability Set.
- Fault domain and update domain counts are visible.
- You can explain why this design does **not** survive a full datacenter outage.

### Verification
```bash
az vm list -g $RG -d --query "[].{Name:name,Power:powerState,Zone:zones}" -o table
az vm availability-set show -g $RG -n $AVSET -o json
```

```powershell
Get-AzVM -ResourceGroupName $RG -Status | Select-Object Name, PowerState
Get-AzAvailabilitySet -ResourceGroupName $RG -Name $AvSet | Format-List *
```

### Cleanup
```bash
az group delete --name $RG --yes --no-wait
```

```powershell
Remove-AzResourceGroup -Name $RG -Force -AsJob
```

### Exam Tip
If the question says **single datacenter**, **fault domains**, or **update domains**, think **Availability Set**. If it says **survive datacenter failure within a region**, think **Availability Zones**.

---

## Lab 3: Zone-Redundant SQL Database

### Objective
Create an Azure SQL Database in the Business Critical tier with zone redundancy enabled, validate the configuration, and compare it with non-zone-redundant and regional failover options.

### When to Use
Use this pattern for **database HA within a region** when the application requires low downtime during datacenter-level failures without a full multi-region design.

### Key AZ-305 Concepts
- Zone redundancy is available for supported tiers and regions.
- Business Critical provides local HA and faster failover characteristics.
- Zone redundancy is **not** the same as geo-replication or failover groups.
- Monitoring availability requires metrics, alerts, and connection resiliency.

### Exam Domain Mapping
- **Primary:** Design business continuity solutions (15-20%)
- **Secondary:** Design data storage solutions (20-25%)

### Prerequisites
- Azure SQL providers registered
- SQL admin username/password prepared
- Azure CLI and PowerShell authenticated

### Architecture and Design Rationale
Zone-redundant Azure SQL Database replicates compute and storage across zones in the same region. This is best for **regional HA**, not regional DR. For AZ-305, compare this with **auto-failover groups**, which add **cross-region continuity** but introduce asynchronous replication considerations.

### Implementation Steps
1. Create the resource group and SQL logical server.
2. Configure firewall access for testing.
3. Create a Business Critical database with zone redundancy enabled.
4. Review the database SKU and zone-redundant property.
5. Monitor availability metrics and discuss geo-failover alternatives.

### Azure CLI Commands
```bash
RG="rg-ha-lab3"
LOCATION="eastus2"
SQLSERVER="sqlha$RANDOM"
DBNAME="sqldb-ha"
SQLADMIN="sqladminuser"
SQLPASSWORD="<ComplexPassword123!>"
MYIP=$(curl -s https://api.ipify.org)

az group create --name $RG --location $LOCATION
az sql server create \
  --resource-group $RG \
  --location $LOCATION \
  --name $SQLSERVER \
  --admin-user $SQLADMIN \
  --admin-password $SQLPASSWORD

az sql server firewall-rule create \
  --resource-group $RG \
  --server $SQLSERVER \
  --name allow-client \
  --start-ip-address $MYIP \
  --end-ip-address $MYIP

az sql db create \
  --resource-group $RG \
  --server $SQLSERVER \
  --name $DBNAME \
  --service-objective BC_Gen5_2 \
  --zone-redundant true \
  --backup-storage-redundancy Zone

az sql db show \
  --resource-group $RG \
  --server $SQLSERVER \
  --name $DBNAME \
  --query "{Edition:currentSku.tier,Name:name,ZoneRedundant:zoneRedundant,Status:status}" \
  -o table
```

### PowerShell Commands
```powershell
$RG = "rg-ha-lab3"
$Location = "eastus2"
$SqlServer = "sqlha$(Get-Random)"
$DbName = "sqldb-ha"
$SqlAdmin = "sqladminuser"
$SqlPassword = ConvertTo-SecureString "<ComplexPassword123!>" -AsPlainText -Force
$Cred = [System.Management.Automation.PSCredential]::new($SqlAdmin, $SqlPassword)
$MyIp = (Invoke-RestMethod -Uri "https://api.ipify.org")

New-AzResourceGroup -Name $RG -Location $Location
New-AzSqlServer -ResourceGroupName $RG -Location $Location -ServerName $SqlServer -SqlAdministratorCredentials $Cred
New-AzSqlServerFirewallRule -ResourceGroupName $RG -ServerName $SqlServer -FirewallRuleName allow-client -StartIpAddress $MyIp -EndIpAddress $MyIp
New-AzSqlDatabase -ResourceGroupName $RG -ServerName $SqlServer -DatabaseName $DbName -Edition BusinessCritical -ComputeGeneration Gen5 -VCore 2 -ZoneRedundant
Get-AzSqlDatabase -ResourceGroupName $RG -ServerName $SqlServer -DatabaseName $DbName | Select-Object DatabaseName, Edition, CurrentServiceObjectiveName, ZoneRedundant, Status
```

### IaC Implementation
#### Bicep
```bicep
resource sqlDb 'Microsoft.Sql/servers/databases@2023-08-01-preview' = {
  name: '${sqlServer.name}/sqldb-ha'
  location: resourceGroup().location
  sku: {
    name: 'BC_Gen5_2'
    tier: 'BusinessCritical'
  }
  properties: {
    zoneRedundant: true
  }
}
```

#### Terraform
```hcl
resource "azurerm_mssql_database" "ha" {
  name         = "sqldb-ha"
  server_id    = azurerm_mssql_server.sql.id
  sku_name     = "BC_Gen5_2"
  zone_redundant = true
}
```

### Validation and Success Criteria
- The database is created in the Business Critical tier.
- `zoneRedundant` returns `true`.
- You can explain the difference between **zone redundancy** and **auto-failover groups**.

### Verification
```bash
az sql db show -g $RG -s $SQLSERVER -n $DBNAME --query "{Tier:currentSku.tier,ZoneRedundant:zoneRedundant,BackupRedundancy:requestedBackupStorageRedundancy}" -o table
az monitor metrics list \
  --resource $(az sql db show -g $RG -s $SQLSERVER -n $DBNAME --query id -o tsv) \
  --metric cpu_percent dtu_consumption_percent \
  --interval PT5M
```

```powershell
$db = Get-AzSqlDatabase -ResourceGroupName $RG -ServerName $SqlServer -DatabaseName $DbName
$db | Select-Object DatabaseName, Edition, CurrentServiceObjectiveName, ZoneRedundant
Get-AzMetric -ResourceId $db.ResourceId -MetricName cpu_percent -TimeGrain 00:05:00
```

### Cleanup
```bash
az group delete --name $RG --yes --no-wait
```

```powershell
Remove-AzResourceGroup -Name $RG -Force -AsJob
```

### Exam Tip
**Zone-redundant SQL Database** solves **in-region HA**. If the requirement says **regional outage** or **secondary region failover**, move to **auto-failover groups** or **active geo-replication**.

---

## Lab 4: Zone-Redundant App Service

### Objective
Create a zone-redundant App Service Plan, deploy a simple web app, configure autoscale, and validate zonal distribution behavior.

### When to Use
Use this pattern for **PaaS web app HA** when you want zone-level protection with less operational overhead than IaaS.

### Key AZ-305 Concepts
- App Service can provide zone redundancy on supported Premium tiers and regions.
- Autoscale and zone redundancy are complementary.
- App Service reduces management overhead compared with zonal VMs.
- Zone redundancy is still **single-region**; combine with Front Door or Traffic Manager for multi-region HA.

### Exam Domain Mapping
- **Primary:** Design business continuity solutions (15-20%)
- **Secondary:** Design infrastructure solutions (30-35%)

### Prerequisites
- Supported region and App Service Premium tier
- Zip package or sample app content for deployment
- Azure CLI and Azure PowerShell authenticated

### Architecture and Design Rationale
This lab uses App Service because the workload is HTTP-based and benefits from managed patching, built-in scaling, and simplified HA. For AZ-305, know that App Service is often a better answer than VMs when the requirement is **managed platform + high availability + lower ops**.

### Implementation Steps
1. Create the resource group and zone-redundant Premium App Service Plan.
2. Create the web app and deploy a simple page.
3. Configure autoscale to scale out on CPU.
4. Validate `zoneRedundant` and worker count.
5. Discuss why this is easier to operate than zonal VMs.

### Azure CLI Commands
```bash
RG="rg-ha-lab4"
LOCATION="eastus2"
PLAN="asp-ha-lab4"
WEBAPP="webha$RANDOM"

az group create --name $RG --location $LOCATION
az appservice plan create \
  --resource-group $RG \
  --name $PLAN \
  --location $LOCATION \
  --sku P1V3 \
  --number-of-workers 3 \
  --zone-redundant true \
  --is-linux

az webapp create \
  --resource-group $RG \
  --plan $PLAN \
  --name $WEBAPP \
  --runtime "NODE|20-lts"

mkdir appservice-ha && cd appservice-ha
cat > index.html << 'EOF'
<html><body><h1>Zone-Redundant App Service</h1></body></html>
EOF
zip site.zip index.html
az webapp deploy \
  --resource-group $RG \
  --name $WEBAPP \
  --type zip \
  --src-path site.zip
cd ..

PLANID=$(az appservice plan show -g $RG -n $PLAN --query id -o tsv)
az monitor autoscale create \
  --resource-group $RG \
  --resource $PLANID \
  --resource-type Microsoft.Web/serverfarms \
  --name autoscale-ha \
  --min-count 3 \
  --max-count 6 \
  --count 3

az monitor autoscale rule create \
  --resource-group $RG \
  --autoscale-name autoscale-ha \
  --condition "Percentage CPU > 70 avg 10m" \
  --scale out 1

az appservice plan show \
  --resource-group $RG \
  --name $PLAN \
  --query "{Tier:sku.tier,Workers:sku.capacity,ZoneRedundant:zoneRedundant}" \
  -o table
```

### PowerShell Commands
```powershell
$RG = "rg-ha-lab4"
$Location = "eastus2"
$Plan = "asp-ha-lab4"
$WebApp = "webha$(Get-Random)"

New-AzResourceGroup -Name $RG -Location $Location
New-AzAppServicePlan -ResourceGroupName $RG -Name $Plan -Location $Location -Tier PremiumV3 -NumberofWorkers 3 -WorkerSize Small -Linux -ZoneRedundant
New-AzWebApp -ResourceGroupName $RG -Name $WebApp -Location $Location -AppServicePlan $Plan -Runtime "NODE|20-lts"
New-Item -ItemType Directory -Path .\appservice-ha -Force | Out-Null
"<html><body><h1>Zone-Redundant App Service</h1></body></html>" | Set-Content .\appservice-ha\index.html
Compress-Archive -Path .\appservice-ha\* -DestinationPath .\appservice-ha\site.zip -Force
Publish-AzWebApp -ResourceGroupName $RG -Name $WebApp -ArchivePath .\appservice-ha\site.zip
Get-AzAppServicePlan -ResourceGroupName $RG -Name $Plan | Select-Object Name, Tier, NumberofWorkers, ZoneRedundant
```

### IaC Implementation
#### Bicep
```bicep
resource plan 'Microsoft.Web/serverfarms@2023-12-01' = {
  name: 'asp-ha-lab4'
  location: resourceGroup().location
  sku: {
    name: 'P1v3'
    tier: 'PremiumV3'
    capacity: 3
  }
  kind: 'linux'
  properties: {
    reserved: true
    zoneRedundant: true
  }
}
```

#### Terraform
```hcl
resource "azurerm_service_plan" "ha" {
  name                = "asp-ha-lab4"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  os_type             = "Linux"
  sku_name            = "P1v3"
  worker_count        = 3
  zone_balancing_enabled = true
}
```

### Validation and Success Criteria
- The App Service Plan is Premium and zone redundant.
- Minimum worker count is at least three for better zone spread.
- Autoscale rules exist.
- The web app remains reachable during worker maintenance events.

### Verification
```bash
az appservice plan show -g $RG -n $PLAN --query "{Workers:sku.capacity,ZoneRedundant:zoneRedundant}" -o table
az webapp show -g $RG -n $WEBAPP --query "{State:state,Host:defaultHostName}" -o table
curl -I https://$(az webapp show -g $RG -n $WEBAPP --query defaultHostName -o tsv)
```

```powershell
$App = Get-AzWebApp -ResourceGroupName $RG -Name $WebApp
$PlanInfo = Get-AzAppServicePlan -ResourceGroupName $RG -Name $Plan
$PlanInfo | Select-Object Name, Tier, NumberofWorkers, ZoneRedundant
Invoke-WebRequest -Uri "https://$($App.DefaultHostName)" -UseBasicParsing | Select-Object StatusCode
```

### Cleanup
```bash
rm -rf appservice-ha
az group delete --name $RG --yes --no-wait
```

```powershell
Remove-Item .\appservice-ha -Recurse -Force -ErrorAction SilentlyContinue
Remove-AzResourceGroup -Name $RG -Force -AsJob
```

### Exam Tip
If the workload is a **web app** and the requirement emphasizes **managed platform, autoscale, and low operational overhead**, App Service usually beats VM-based designs.

---

## Lab 5: Multi-Region Web Application

### Objective
Deploy the application to two Azure regions, configure Traffic Manager for health-probe-based failover, test a regional failure scenario, and calculate composite SLA.

### When to Use
Use this pattern for **global application HA** when the app must survive a **regional outage** and continue serving users from another region.

### Key AZ-305 Concepts
- Multi-region HA is different from in-region HA.
- Traffic Manager is DNS-based and supports non-proxy failover.
- Front Door is often better for HTTP/HTTPS acceleration and WAF.
- Composite SLA depends on both the global routing tier and the backend design.

### Exam Domain Mapping
- **Primary:** Design business continuity solutions (15-20%)
- **Secondary:** Design infrastructure solutions (30-35%)

### Prerequisites
- Two Azure regions selected, for example `eastus2` and `centralus`
- Public DNS resolution to test Traffic Manager
- Azure CLI and Azure PowerShell authenticated

### Architecture and Design Rationale
This lab uses **Traffic Manager** because it is easy to understand and explicitly highlights **DNS-based global failover**. For HTTP/HTTPS apps requiring edge acceleration, SSL offload, and WAF, **Front Door** is usually the better exam answer. Here, the goal is to understand **regional failover flow** and **composite SLA math**.

### Implementation Steps
1. Create two regional resource groups.
2. Deploy one App Service app in each region.
3. Create a Traffic Manager profile with priority routing and health probes.
4. Add both regional apps as endpoints.
5. Stop or disable the primary endpoint.
6. Validate failover and compute composite SLA.

### Azure CLI Commands
```bash
RG1="rg-ha-lab5-east"
RG2="rg-ha-lab5-central"
LOC1="eastus2"
LOC2="centralus"
PLAN1="asp-ha-east"
PLAN2="asp-ha-central"
APP1="webhaeast$RANDOM"
APP2="webhacentral$RANDOM"
TM="tm-ha-$RANDOM"

az group create --name $RG1 --location $LOC1
az group create --name $RG2 --location $LOC2

az appservice plan create --resource-group $RG1 --name $PLAN1 --location $LOC1 --sku P1V3 --is-linux
az appservice plan create --resource-group $RG2 --name $PLAN2 --location $LOC2 --sku P1V3 --is-linux
az webapp create --resource-group $RG1 --plan $PLAN1 --name $APP1 --runtime "NODE|20-lts"
az webapp create --resource-group $RG2 --plan $PLAN2 --name $APP2 --runtime "NODE|20-lts"

mkdir tm-east && echo "<html><body><h1>Primary Region - East US 2</h1></body></html>" > tm-east/index.html
mkdir tm-central && echo "<html><body><h1>Secondary Region - Central US</h1></body></html>" > tm-central/index.html
(cd tm-east && zip site-east.zip index.html)
(cd tm-central && zip site-central.zip index.html)
az webapp deploy --resource-group $RG1 --name $APP1 --type zip --src-path tm-east/site-east.zip
az webapp deploy --resource-group $RG2 --name $APP2 --type zip --src-path tm-central/site-central.zip

az network traffic-manager profile create \
  --resource-group $RG1 \
  --name $TM \
  --routing-method Priority \
  --unique-dns-name $TM \
  --ttl 30 \
  --protocol HTTP \
  --port 80 \
  --path /

az network traffic-manager endpoint create \
  --resource-group $RG1 \
  --profile-name $TM \
  --name east-endpoint \
  --type azureEndpoints \
  --target-resource-id $(az webapp show -g $RG1 -n $APP1 --query id -o tsv) \
  --priority 1

az network traffic-manager endpoint create \
  --resource-group $RG1 \
  --profile-name $TM \
  --name central-endpoint \
  --type azureEndpoints \
  --target-resource-id $(az webapp show -g $RG2 -n $APP2 --query id -o tsv) \
  --priority 2

az network traffic-manager profile show -g $RG1 -n $TM --query "{DnsName:dnsConfig.fqdn,Routing:trafficRoutingMethod,MonitorPath:monitorConfig.path}" -o table
```

### PowerShell Commands
```powershell
$RG1 = "rg-ha-lab5-east"
$RG2 = "rg-ha-lab5-central"
$Loc1 = "eastus2"
$Loc2 = "centralus"
$Plan1 = "asp-ha-east"
$Plan2 = "asp-ha-central"
$App1 = "webhaeast$(Get-Random)"
$App2 = "webhacentral$(Get-Random)"
$Tm = "tm-ha-$(Get-Random)"

New-AzResourceGroup -Name $RG1 -Location $Loc1
New-AzResourceGroup -Name $RG2 -Location $Loc2
New-AzAppServicePlan -ResourceGroupName $RG1 -Name $Plan1 -Location $Loc1 -Tier PremiumV3 -NumberofWorkers 1 -WorkerSize Small -Linux
New-AzAppServicePlan -ResourceGroupName $RG2 -Name $Plan2 -Location $Loc2 -Tier PremiumV3 -NumberofWorkers 1 -WorkerSize Small -Linux
New-AzWebApp -ResourceGroupName $RG1 -Name $App1 -Location $Loc1 -AppServicePlan $Plan1
New-AzWebApp -ResourceGroupName $RG2 -Name $App2 -Location $Loc2 -AppServicePlan $Plan2
New-Item -ItemType Directory -Path .\tm-east -Force | Out-Null
New-Item -ItemType Directory -Path .\tm-central -Force | Out-Null
"<html><body><h1>Primary Region - East US 2</h1></body></html>" | Set-Content .\tm-east\index.html
"<html><body><h1>Secondary Region - Central US</h1></body></html>" | Set-Content .\tm-central\index.html
Compress-Archive -Path .\tm-east\* -DestinationPath .\tm-east\site-east.zip -Force
Compress-Archive -Path .\tm-central\* -DestinationPath .\tm-central\site-central.zip -Force
Publish-AzWebApp -ResourceGroupName $RG1 -Name $App1 -ArchivePath .\tm-east\site-east.zip
Publish-AzWebApp -ResourceGroupName $RG2 -Name $App2 -ArchivePath .\tm-central\site-central.zip

$tmProfile = New-AzTrafficManagerProfile -Name $Tm -ResourceGroupName $RG1 -TrafficRoutingMethod Priority -Ttl 30 -MonitorProtocol HTTP -MonitorPort 80 -MonitorPath "/" -RelativeDnsName $Tm
New-AzTrafficManagerEndpoint -Name east-endpoint -ProfileName $Tm -ResourceGroupName $RG1 -Type AzureEndpoints -TargetResourceId (Get-AzWebApp -ResourceGroupName $RG1 -Name $App1).Id -EndpointStatus Enabled -Priority 1
New-AzTrafficManagerEndpoint -Name central-endpoint -ProfileName $Tm -ResourceGroupName $RG1 -Type AzureEndpoints -TargetResourceId (Get-AzWebApp -ResourceGroupName $RG2 -Name $App2).Id -EndpointStatus Enabled -Priority 2
Get-AzTrafficManagerProfile -Name $Tm -ResourceGroupName $RG1 | Select-Object Name, TrafficRoutingMethod, Fqdn
```

### IaC Implementation
#### Bicep
```bicep
resource tm 'Microsoft.Network/trafficManagerProfiles@2022-04-01' = {
  name: 'tm-ha'
  location: 'global'
  properties: {
    profileStatus: 'Enabled'
    trafficRoutingMethod: 'Priority'
    dnsConfig: {
      relativeName: 'tm-ha'
      ttl: 30
    }
    monitorConfig: {
      protocol: 'HTTP'
      port: 80
      path: '/'
    }
  }
}
```

#### Terraform
```hcl
resource "azurerm_traffic_manager_profile" "ha" {
  name                   = "tm-ha"
  resource_group_name    = azurerm_resource_group.rg.name
  traffic_routing_method = "Priority"
  dns_config {
    relative_name = "tm-ha"
    ttl           = 30
  }
  monitor_config {
    protocol = "HTTP"
    port     = 80
    path     = "/"
  }
}
```

### Validation and Success Criteria
- Both regional apps are deployed and healthy.
- Traffic Manager shows the primary endpoint online.
- Disabling the primary endpoint shifts resolution to the secondary.
- You can explain the composite SLA formula for the solution.

### Verification
```bash
TMFQDN=$(az network traffic-manager profile show -g $RG1 -n $TM --query dnsConfig.fqdn -o tsv)
nslookup $TMFQDN
az network traffic-manager endpoint update --resource-group $RG1 --profile-name $TM --name east-endpoint --type azureEndpoints --endpoint-status Disabled
nslookup $TMFQDN
# Example SLA math: TM 99.99%, App Service 99.95% in each region
# Parallel app tier SLA = 1 - (1 - 0.9995)^2 = 0.99999975
# Composite SLA = 0.9999 * 0.99999975 = 0.99989975 (~99.99%)
```

```powershell
$TmFqdn = (Get-AzTrafficManagerProfile -Name $Tm -ResourceGroupName $RG1).Fqdn
Resolve-DnsName $TmFqdn
Disable-AzTrafficManagerEndpoint -Name east-endpoint -ProfileName $Tm -ResourceGroupName $RG1 -Type AzureEndpoints
Resolve-DnsName $TmFqdn
$parallelSla = 1 - ((1 - 0.9995) * (1 - 0.9995))
$compositeSla = 0.9999 * $parallelSla
"Composite SLA: {0:P6}" -f $compositeSla
```

### Cleanup
```bash
rm -rf tm-east tm-central
az group delete --name $RG1 --yes --no-wait
az group delete --name $RG2 --yes --no-wait
```

```powershell
Remove-Item .\tm-east -Recurse -Force -ErrorAction SilentlyContinue
Remove-Item .\tm-central -Recurse -Force -ErrorAction SilentlyContinue
Remove-AzResourceGroup -Name $RG1 -Force -AsJob
Remove-AzResourceGroup -Name $RG2 -Force -AsJob
```

### Exam Tip
Use **Traffic Manager** for **DNS-based global routing**. Use **Front Door** when the scenario says **HTTP/HTTPS**, **WAF**, **edge acceleration**, **TLS offload**, or **fast global failover**.

---

## Lab 6: AKS High Availability

### Objective
Create a zone-aware AKS cluster, deploy an application with pod anti-affinity and a Pod Disruption Budget, and validate resilience during node failure or maintenance.

### When to Use
Use this pattern for **container workload HA** when the application runs on Kubernetes and must tolerate node or zone-level disruption.

### Key AZ-305 Concepts
- AKS control plane SLA depends on selected tier/Uptime SLA.
- Availability Zones improve worker node resilience.
- Pod anti-affinity spreads replicas.
- Pod Disruption Budgets protect minimum pod availability during voluntary disruptions.

### Exam Domain Mapping
- **Primary:** Design business continuity solutions (15-20%)
- **Secondary:** Design infrastructure solutions (30-35%)

### Prerequisites
- `kubectl` installed
- Container image available, for example `mcr.microsoft.com/azuredocs/aks-helloworld:v1`
- Azure CLI and Azure PowerShell authenticated

### Architecture and Design Rationale
AKS high availability requires more than just a resilient control plane. The workload must also spread pods, tolerate node drain events, and avoid single-node placement. For AZ-305, know that **cluster HA** and **application HA** are related but separate design decisions.

### Implementation Steps
1. Create a zone-aware AKS cluster using VMSS-backed nodes.
2. Get cluster credentials.
3. Deploy a multi-replica application.
4. Apply pod anti-affinity and a Pod Disruption Budget.
5. Simulate a node failure or drain.
6. Verify pods stay available.

### Azure CLI Commands
```bash
RG="rg-ha-lab6"
LOCATION="eastus2"
AKSNAME="aks-ha-lab6"

az group create --name $RG --location $LOCATION
az aks create \
  --resource-group $RG \
  --name $AKSNAME \
  --location $LOCATION \
  --node-count 3 \
  --node-vm-size Standard_D4s_v5 \
  --nodepool-zones 1 2 3 \
  --vm-set-type VirtualMachineScaleSets \
  --load-balancer-sku standard \
  --tier standard \
  --generate-ssh-keys

az aks get-credentials --resource-group $RG --name $AKSNAME --overwrite-existing

kubectl create deployment aks-ha --image=mcr.microsoft.com/azuredocs/aks-helloworld:v1 --replicas=3
kubectl expose deployment aks-ha --type=LoadBalancer --port=80 --target-port=80
```

### PowerShell Commands
```powershell
$RG = "rg-ha-lab6"
$Location = "eastus2"
$AksName = "aks-ha-lab6"

New-AzResourceGroup -Name $RG -Location $Location
New-AzAksCluster -ResourceGroupName $RG -Name $AksName -Location $Location -NodeCount 3 -NodeVmSize "Standard_D4s_v5" -NodePoolAvailabilityZone 1,2,3 -NetworkPlugin azure -LoadBalancerSku standard -GenerateSshKey
Import-AzAksCredential -ResourceGroupName $RG -Name $AksName -Force
kubectl create deployment aks-ha --image=mcr.microsoft.com/azuredocs/aks-helloworld:v1 --replicas=3
kubectl expose deployment aks-ha --type=LoadBalancer --port=80 --target-port=80
```

### IaC Implementation
#### Bicep
```bicep
resource aks 'Microsoft.ContainerService/managedClusters@2024-02-01' = {
  name: 'aks-ha-lab6'
  location: resourceGroup().location
  sku: {
    tier: 'Standard'
    name: 'Base'
  }
  properties: {
    agentPoolProfiles: [
      {
        name: 'systempool'
        count: 3
        vmSize: 'Standard_D4s_v5'
        availabilityZones: [ '1' '2' '3' ]
        type: 'VirtualMachineScaleSets'
        mode: 'System'
      }
    ]
  }
}
```

#### Terraform
```hcl
resource "azurerm_kubernetes_cluster" "ha" {
  name                = "aks-ha-lab6"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  dns_prefix          = "aks-ha-lab6"
  sku_tier            = "Standard"
  default_node_pool {
    name                = "system"
    node_count          = 3
    vm_size             = "Standard_D4s_v5"
    zones               = ["1", "2", "3"]
    type                = "VirtualMachineScaleSets"
  }
  identity { type = "SystemAssigned" }
}
```

### Validation and Success Criteria
- AKS node pool spans three zones.
- Application has three replicas.
- Anti-affinity and a Pod Disruption Budget are applied.
- Draining or stopping one node does not make the application unavailable.

### Verification
Save the following workload manifest as `aks-ha.yaml`, then apply it:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: aks-ha
spec:
  replicas: 3
  selector:
    matchLabels:
      app: aks-ha
  template:
    metadata:
      labels:
        app: aks-ha
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            - labelSelector:
                matchExpressions:
                  - key: app
                    operator: In
                    values:
                      - aks-ha
              topologyKey: kubernetes.io/hostname
      containers:
        - name: web
          image: mcr.microsoft.com/azuredocs/aks-helloworld:v1
          ports:
            - containerPort: 80
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: aks-ha-pdb
spec:
  minAvailable: 2
  selector:
    matchLabels:
      app: aks-ha
```

```bash
kubectl apply -f aks-ha.yaml
kubectl get nodes -L topology.kubernetes.io/zone
kubectl get pods -o wide
NODE=$(kubectl get nodes -o jsonpath='{.items[0].metadata.name}')
kubectl drain $NODE --ignore-daemonsets --delete-emptydir-data
kubectl get pods -o wide
kubectl uncordon $NODE
```

```powershell
kubectl apply -f .\aks-ha.yaml
kubectl get nodes -L topology.kubernetes.io/zone
kubectl get pods -o wide
$Node = kubectl get nodes -o jsonpath='{.items[0].metadata.name}'
kubectl drain $Node --ignore-daemonsets --delete-emptydir-data
kubectl get pods -o wide
kubectl uncordon $Node
```

### Cleanup
```bash
rm -f aks-ha.yaml
az group delete --name $RG --yes --no-wait
```

```powershell
Remove-Item .\aks-ha.yaml -Force -ErrorAction SilentlyContinue
Remove-AzResourceGroup -Name $RG -Force -AsJob
```

### Exam Tip
For AKS, do not stop at **cluster creation**. The exam often expects you to combine **zones + replica count + anti-affinity + PDB + autoscaling** for true application-level availability.

---

## Lab 7: Zone-Redundant Storage

### Objective
Create a ZRS storage account, compare it with LRS and GRS-style redundancy choices, deploy a zone-redundant managed disk, and understand zone-failure behavior.

### When to Use
Use this pattern for **storage HA within a region** when the workload must survive zonal failure but does not require active multi-region read/write capability.

### Key AZ-305 Concepts
- LRS replicates in one datacenter scope.
- ZRS replicates synchronously across zones in one region.
- GRS/GZRS add cross-region durability.
- Zone-redundant managed disks help zonal VM architectures.

### Exam Domain Mapping
- **Primary:** Design business continuity solutions (15-20%)
- **Secondary:** Design data storage solutions (20-25%)

### Prerequisites
- ZRS-capable region
- Azure CLI and Azure PowerShell authenticated
- Familiarity with storage redundancy options

### Architecture and Design Rationale
ZRS is best when the requirement is **high availability inside one region**. If the requirement includes **regional disaster recovery**, then move to **GRS**, **GZRS**, or **RA-GZRS** depending read-access and zonal needs. For managed disks, zone-redundant disks are most valuable when compute architecture already spans zones.

### Implementation Steps
1. Create a ZRS storage account.
2. Review LRS vs ZRS vs GRS decision points.
3. Create a Premium SSD v2 or supported managed disk with zone-redundancy support.
4. Review behavior during zonal failure.
5. Discuss when GZRS or RA-GZRS is the better answer.

### Azure CLI Commands
```bash
RG="rg-ha-lab7"
LOCATION="eastus2"
SA="sthazrs$RANDOM"
DISK="disk-zrs-ha"

az group create --name $RG --location $LOCATION
az storage account create \
  --resource-group $RG \
  --name $SA \
  --location $LOCATION \
  --sku Standard_ZRS \
  --kind StorageV2

az storage account show \
  --resource-group $RG \
  --name $SA \
  --query "{Name:name,Redundancy:sku.name,PrimaryLocation:primaryLocation}" \
  -o table

az disk create \
  --resource-group $RG \
  --name $DISK \
  --location $LOCATION \
  --sku Premium_ZRS \
  --size-gb 128

az disk show \
  --resource-group $RG \
  --name $DISK \
  --query "{Name:name,Sku:sku.name,Location:location}" \
  -o table
```

### PowerShell Commands
```powershell
$RG = "rg-ha-lab7"
$Location = "eastus2"
$Sa = "sthazrs$(Get-Random)"
$Disk = "disk-zrs-ha"

New-AzResourceGroup -Name $RG -Location $Location
New-AzStorageAccount -ResourceGroupName $RG -Name $Sa -Location $Location -SkuName Standard_ZRS -Kind StorageV2
Get-AzStorageAccount -ResourceGroupName $RG -Name $Sa | Select-Object StorageAccountName, @{N='Sku';E={$_.Sku.Name}}, PrimaryLocation

$diskConfig = New-AzDiskConfig -Location $Location -SkuName Premium_ZRS -DiskSizeGB 128 -CreateOption Empty
New-AzDisk -ResourceGroupName $RG -DiskName $Disk -Disk $diskConfig
Get-AzDisk -ResourceGroupName $RG -DiskName $Disk | Select-Object Name, Sku, Location, DiskSizeGB
```

### IaC Implementation
#### Bicep
```bicep
resource sa 'Microsoft.Storage/storageAccounts@2023-05-01' = {
  name: 'sthazrs12345'
  location: resourceGroup().location
  sku: { name: 'Standard_ZRS' }
  kind: 'StorageV2'
}
```

#### Terraform
```hcl
resource "azurerm_storage_account" "ha" {
  name                     = "sthazrs12345"
  resource_group_name      = azurerm_resource_group.rg.name
  location                 = azurerm_resource_group.rg.location
  account_tier             = "Standard"
  account_replication_type = "ZRS"
}
```

### Validation and Success Criteria
- Storage account shows `Standard_ZRS`.
- Managed disk shows a zone-redundant SKU.
- You can explain when to choose **ZRS** over **GZRS** and **RA-GZRS**.

### Verification
```bash
az storage account show -g $RG -n $SA --query "{Sku:sku.name,Kind:kind}" -o table
az disk show -g $RG -n $DISK --query "{Sku:sku.name,Id:id}" -o table
```

```powershell
Get-AzStorageAccount -ResourceGroupName $RG -Name $Sa | Select-Object StorageAccountName, Kind, @{N='Sku';E={$_.Sku.Name}}
Get-AzDisk -ResourceGroupName $RG -DiskName $Disk | Select-Object Name, Sku, DiskSizeGB
```

### Cleanup
```bash
az group delete --name $RG --yes --no-wait
```

```powershell
Remove-AzResourceGroup -Name $RG -Force -AsJob
```

### Exam Tip
Use **ZRS** for **in-region availability**, **GRS/GZRS** for **regional durability**, and **RA-GRS/RA-GZRS** when the scenario explicitly requires **read access to the secondary region**.

---

## Lab 8: SLA Calculation Exercise

### Objective
Calculate composite SLA for sample HA architectures, identify single points of failure, and improve the design to meet a target availability requirement.

### When to Use
Use this exercise for **AZ-305 exam prep** whenever the question asks whether an architecture meets a required SLA.

### Key AZ-305 Concepts
- Serial dependency SLA multiplication
- Parallel redundancy formula
- Single points of failure reduce composite SLA
- SLA is not the same as RTO or RPO

### Exam Domain Mapping
- **Primary:** Design business continuity solutions (15-20%)
- **Secondary:** Design infrastructure solutions (30-35%)

### Prerequisites
- Basic calculator or spreadsheet
- Familiarity with Azure service SLAs
- Understanding of single-tier versus multi-tier architecture

### Architecture and Design Rationale
The exam often tests whether you can identify the **weakest link** in an architecture. If components depend on each other in series, the composite SLA is always **lower** than the lowest single service SLA. Redundancy in parallel can improve availability, but only if the design actually removes the single point of failure.

### Implementation Steps
1. Calculate the composite SLA for a three-tier app.
2. Identify the single points of failure.
3. Redesign the app for a higher target SLA.
4. Compare the result with a multi-region solution.

### Azure CLI Commands
```bash
# Use service configuration queries to determine which SLA tier applies
az vm show --resource-group <rg> --name <vm> --query "{Zones:zones,AvailabilitySet:availabilitySet.id}" -o table
az appservice plan show --resource-group <rg> --name <plan> --query "{Tier:sku.tier,ZoneRedundant:zoneRedundant}" -o table
az sql db show --resource-group <rg> --server <server> --name <db> --query "{Tier:currentSku.tier,ZoneRedundant:zoneRedundant}" -o table
az storage account show --resource-group <rg> --name <storage> --query "{Sku:sku.name}" -o table
```

### PowerShell Commands
```powershell
function Get-CompositeSla {
    param([double[]]$SerialSlas)
    $result = 1
    foreach ($sla in $SerialSlas) { $result *= $sla }
    return $result
}

function Get-ParallelSla {
    param([double[]]$ParallelSlas)
    $failure = 1
    foreach ($sla in $ParallelSlas) { $failure *= (1 - $sla) }
    return (1 - $failure)
}

$threeTier = Get-CompositeSla -SerialSlas @(0.9995, 0.9999, 0.999)
"Three-tier SLA: {0:P4}" -f $threeTier
$parallelWeb = Get-ParallelSla -ParallelSlas @(0.9995, 0.9995)
$composite = 0.9999 * $parallelWeb
"Multi-region web composite SLA: {0:P6}" -f $composite
```

### IaC Implementation
#### Bicep
```bicep
// No Azure resources are required. Use this lab to validate architecture choices before deployment.
```

#### Terraform
```hcl
# No deployment required. Document SLA assumptions in architecture reviews and landing zone standards.
```

### Validation and Success Criteria
Use these practice scenarios:

| Scenario | Formula | Result |
|---|---|---|
| App Service 99.95% + SQL DB 99.99% + Storage 99.9% | `0.9995 × 0.9999 × 0.999` | `99.84%` |
| Two App Services in parallel | `1 - (1 - 0.9995)^2` | `99.999975%` |
| Traffic Manager + parallel App Services | `0.9999 × 0.99999975` | `99.989975%` |

Success means you can:
- calculate the composite SLA correctly,
- identify the single point of failure,
- recommend the smallest design change that meets the target SLA.

### Verification
```bash
# Manual examples
# 99.95% web + 99.99% database + 99.9% storage = 99.84%
# If target is 99.99%, the single-region storage tier is a likely weak point.
```

```powershell
$monthlyMinutes = 43200
@(0.999,0.9995,0.9999,0.99999) | ForEach-Object {
    [PSCustomObject]@{
        SLA = "{0:P3}" -f $_
        MonthlyDowntimeMinutes = [math]::Round((1 - $_) * $monthlyMinutes, 2)
    }
} | Format-Table -AutoSize
```

### Cleanup
No Azure resources are created in this exercise.

### Exam Tip
If a solution contains even one **serial dependency** with a lower SLA, that dependency usually controls the answer. Always find the **single point of failure** before calculating.

---

## High Availability Decision Summary

| Requirement | Best-Fit Azure Pattern | Why | Common Trap |
|---|---|---|---|
| Survive host/rack maintenance in one datacenter | Availability Set | Protects across fault/update domains | Does not survive datacenter failure |
| Survive datacenter failure in one region | Availability Zones | Best regional HA pattern for zonal services | Still not regional DR |
| Highly available IaaS web tier | Zonal VMs + Standard Load Balancer | Zone isolation + regional load balancing | Keeping app state on local VM disk |
| Highly available PaaS web app | Zone-redundant App Service Plan | Lower ops than VMs | Forgetting that it remains single-region |
| Highly available relational database in-region | SQL Database zone redundancy | Automatic in-region resilience | Confusing it with geo-failover |
| Highly available storage in-region | ZRS | Synchronous cross-zone replication | Assuming it protects from region loss |
| Global web app failover | Front Door or Traffic Manager | Multi-region continuity | Using Traffic Manager when edge/WAF is required |
| Highly available Kubernetes workload | AKS zones + anti-affinity + PDB | Cluster and app-layer resilience | Ignoring workload scheduling rules |
| Meet a target SLA | Composite SLA calculation | Confirms architecture fit | Looking only at the highest-SLA component |

## SLA Reference Chart

### Common AZ-305 SLA Quick Reference

| Service / Pattern | Typical SLA | Notes |
|---|---|---|
| Single VM with Premium SSD | 99.9% | Lower than zonal or set-based deployment |
| Two or more VMs in an Availability Set | 99.95% | Intra-datacenter resilience |
| Two or more VMs across Availability Zones | 99.99% | In-region datacenter failure protection |
| Standard Load Balancer | 99.99% | Regional L4 balancing |
| App Service | 99.95% | Single-region managed web app |
| Traffic Manager | 99.99% | DNS-based global routing |
| Azure Front Door | 99.99% | Global HTTP/HTTPS routing and acceleration |
| Azure SQL Database | 99.99% to 99.995% | Depends on tier and design |
| AKS with Uptime SLA / Standard tier | 99.95% | Control-plane SLA; workload design still matters |
| Azure Storage (LRS/ZRS/GRS durability context) | 99.9% | Check service-specific SLA details |

### Monthly Downtime Reference

| SLA | Approx. Monthly Downtime | Approx. Annual Downtime |
|---|---|---|
| 99% | 7.2 hours | 3.65 days |
| 99.9% | 43.2 minutes | 8.76 hours |
| 99.95% | 21.6 minutes | 4.38 hours |
| 99.99% | 4.32 minutes | 52.56 minutes |
| 99.999% | 25.9 seconds | 5.26 minutes |
