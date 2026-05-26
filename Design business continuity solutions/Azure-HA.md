# Azure High Availability - AZ-305 Comprehensive Cheat Sheet

> 📝 **Hands-On Labs:** [HA Labs](./Labs/Azure-HA-Labs.md)

> 🎯 **Exam Focus:** AZ-305 tests your ability to **design** HA architectures using zones, availability sets, and redundancy patterns.

**Audience:** Senior Cloud Solution Architect  
**Primary exam domain:** Design business continuity solutions (15-20%)  
**Secondary domains:** Design infrastructure solutions; design data storage solutions; design identity, governance, and monitoring solutions  
**Use this for:** rapid revision, architecture tradeoff analysis, and scenario-driven AZ-305 decision making

## Table of Contents

- [Azure HA Family Overview](#1-azure-ha-family-overview)
- [When to Choose Which — Decision Tree](#2-when-to-choose-which--decision-tree)
- [High Availability Foundations](#3-high-availability-foundations)
- [Compute HA](#4-compute-ha)
- [Database HA](#5-database-ha)
- [Storage HA](#6-storage-ha)
- [Networking HA](#7-networking-ha)
- [Multi-Region HA Patterns](#8-multi-region-ha-patterns)
- [SLA Calculations](#9-sla-calculations)
- [Resilience Patterns](#10-resilience-patterns)
- [AZ-305 Decision Scenarios](#11-az-305-decision-scenarios)
- [Quick Reference Trigger Table](#12-quick-reference-trigger-table)
- [Common Exam Traps](#13-common-exam-traps)
- [Cost Optimization](#14-cost-optimization)
- [🎯 Final AZ-305 Exam Tips](#15-final-az-305-exam-tips)
- [📐 Architecture Decision Flowchart](#16-architecture-decision-flowchart)
- [Exam-Style Review Questions](#17-exam-style-review-questions)

---

<a id="1-azure-ha-family-overview"></a>
## 1. Azure HA Family Overview

Azure high availability on AZ-305 is less about memorizing services and more about matching the **failure scope** to the right **redundancy pattern**. Start by deciding whether you need protection from **host failure, planned maintenance, datacenter failure, or regional failure**.

| HA Option | Protects Against | Layer | Best For | Ops Effort | Typical AZ-305 Trigger |
|---|---|---|---|---|---|
| **Availability Sets** | Host/rack failure, planned maintenance | Compute | Legacy or VM-only workloads in a single datacenter boundary | Medium | "Zones not available" / "legacy VMs" |
| **Availability Zones** | Datacenter failure inside one region | Compute + platform placement | New production VM, VMSS, AKS, and zonal architectures | Medium | "survive datacenter outage" |
| **Standard Load Balancer** | Single backend failure in a regional L4 design | Networking | TCP/UDP traffic across VMs, VMSS, or zonal backends | Low | "regional L4" / "health probes" |
| **Application Gateway v2** | Single web tier instance failure in a regional L7 design | Networking | HTTP/HTTPS apps needing path routing, TLS offload, or WAF | Medium | "regional web routing" / "WAF" |
| **Zone-redundant PaaS** | Datacenter failure with managed platform resiliency | PaaS | App Service, SQL, Cosmos DB, and storage where low ops matters | Low | "PaaS-first" / "lowest ops" |

> 💡 **AZ-305 Tip:** If the business requirement can be met by **zone-redundant PaaS**, that is usually a stronger exam answer than building HA manually on VMs.

### Availability Sets
A VM placement construct that distributes Azure VMs across **fault domains** and **update domains** so a single hardware issue or planned maintenance cycle does not take down every instance at once. Availability Sets are still relevant when you must keep a **single-region, VM-based design** and the region or workload cannot use zones.

**Real-World Examples:**
- A **legacy two-tier ERP app** runs on two Windows VMs in a region that does not support Availability Zones for the required VM SKU
- A **manufacturing line controller** must stay on IaaS because of vendor software, but only needs protection from host and maintenance failures inside one facility boundary
- An **internal line-of-business app** is being stabilized before modernization, so the architect places the existing web VMs in an Availability Set as the lowest-risk HA improvement

### Availability Zones
Physically separate datacenters within the same Azure region. You use zones when the workload must remain online during a **single datacenter outage** and the region/service supports zonal or zone-redundant deployment.

**Real-World Examples:**
- A **customer-facing web app** deploys VMs across Zones 1, 2, and 3 behind a Standard Load Balancer to survive a datacenter failure in East US 2
- A **payment processing API** uses zone-redundant App Service and Azure SQL zone redundancy so the platform spans multiple datacenters with minimal operational overhead
- A **production AKS cluster** spreads node pools across zones so cluster and application replicas remain available during zonal maintenance or failure

### Standard Load Balancer
Azure's regional Layer 4 load balancer for **TCP/UDP** workloads. It improves HA by combining multiple healthy backends behind a single frontend and using health probes to stop routing traffic to failed instances.

**Real-World Examples:**
- A **stateless API tier** on a VM Scale Set uses Standard Load Balancer to distribute inbound HTTPS traffic to healthy instances
- A **private application tier** uses an internal Standard Load Balancer so backend services can fail over between zonal VMs without changing client connections
- A **hybrid TCP service** exposes a resilient public endpoint while backend instances are distributed across zones for in-region continuity

### Application Gateway v2
A regional Layer 7 load balancer and reverse proxy for **HTTP/HTTPS** workloads. It adds web-routing intelligence such as host/path rules, TLS termination, cookie affinity, autoscaling, and optional WAF protection.

**Real-World Examples:**
- An **enterprise portal** uses Application Gateway v2 for TLS offload, path-based routing, and WAF in front of multiple web applications
- A **microservices platform** uses host-based routing to send `/api` and `/portal` traffic to different backend pools while preserving HA across zones
- A **regulated public website** requires WAF and regional HA, making Application Gateway v2 the best in-region L7 design choice

### Zone-redundant PaaS
Managed Azure services that automatically distribute platform components across multiple zones in supported regions. These options usually provide the best combination of **availability, simplicity, and reduced operations** when the application fits a PaaS model.

**Real-World Examples:**
- A **web app team** runs a zone-redundant App Service plan with multiple workers instead of managing zonal IIS VMs
- A **relational OLTP workload** uses Azure SQL Database Business Critical with zone redundancy rather than SQL Server Always On on Azure VMs
- A **global product catalog** keeps Cosmos DB regions zone-redundant so a single datacenter failure does not interrupt service within the primary region

---

<a id="2-when-to-choose-which--decision-tree"></a>
## 2. When to Choose Which — Decision Tree

```
Need HIGH AVAILABILITY on Azure?
│
├── Need to survive a REGIONAL outage?
│   └── YES → Use a multi-region design (Front Door / Traffic Manager + replicated data)
│
├── Can the workload stay on MANAGED PaaS?
│   └── YES → Prefer zone-redundant PaaS where supported
│       ├── Web/API → Zone-redundant App Service
│       ├── Relational data → Azure SQL zone redundancy / MI zone redundancy
│       ├── NoSQL / global distribution → Cosmos DB zone-redundant region(s)
│       └── Storage → ZRS or GZRS
│
├── Is it a VM-based workload with autoscale or many identical instances?
│   └── YES → VM Scale Sets + Standard Load Balancer/App Gateway
│       └── If datacenter failure matters and region supports it → spread across zones
│
├── Is it a VM-based workload without zone support?
│   └── YES → Availability Set
│
├── Is traffic TCP/UDP only?
│   └── YES → Standard Load Balancer
│
└── Is traffic HTTP/HTTPS and needs WAF, path routing, or TLS offload?
    └── YES → Application Gateway v2 (regional) or Front Door (global)
```

### Quick Decision Matrix

| If the scenario says... | Usually choose... |
|---|---|
| Survive **datacenter failure** in one region | **Availability Zones** or **zone-redundant PaaS** |
| Keep a **legacy VM app** highly available where zones are unavailable | **Availability Set** |
| Need **HTTP/HTTPS** routing, WAF, or path rules in one region | **Application Gateway v2** |
| Need **TCP/UDP** balancing across redundant instances | **Standard Load Balancer** |
| Want **lowest ops** for web/API HA | **App Service** or another **zone-redundant PaaS** option |
| Need **regional outage** protection | **Multi-region architecture**, not just HA inside one region |

---

<a id="3-high-availability-foundations"></a>
## 3. High Availability Foundations

Azure high availability (HA) is about **keeping the workload running during expected component failures** such as host loss, rack failure, planned maintenance, or a single datacenter outage inside a region. On AZ-305, the best answer usually balances **availability target, operational simplicity, cost, and data consistency**.

### HA vs. DR vs. Backup

| Capability | Primary goal | Typical failure scope | Key metrics | Common Azure services |
|---|---|---|---|---|
| **High Availability (HA)** | Keep service online with minimal interruption | Host, rack, zone, local component failure | Availability %, latency, MTTR | Availability Sets, Availability Zones, Standard Load Balancer, App Gateway v2, zone-redundant PaaS |
| **Disaster Recovery (DR)** | Restore service after major outage | Regional outage, platform-wide incident, site loss | **RTO**, **RPO** | Front Door, Traffic Manager, SQL failover groups, Cosmos DB multi-region, Site Recovery |
| **Backup** | Recover data to a known good point | Deletion, corruption, ransomware, logical error | Restore time, retention, immutability | Azure Backup, SQL PITR, storage snapshots, vaults |

> **Architect rule:** HA does **not** replace DR, and DR does **not** replace backup. Mission-critical workloads usually need all three.

### Azure SLAs and composite SLAs

- An **SLA** is Microsoft’s uptime commitment for a specific service configuration.
- The exam often tests whether you know the difference between:
  - **single-instance design**
  - **redundant design**
  - **service SLA** vs. **application availability**
- A stronger architecture usually comes from **platform redundancy** plus **application resilience**, not from a single service SLA in isolation.

| SLA | Approx. max downtime/year |
|---|---|
| **99.9%** | 8 hours 45 minutes |
| **99.95%** | 4 hours 23 minutes |
| **99.99%** | 52.6 minutes |
| **99.995%** | 26.3 minutes |
| **99.999%** | 5.3 minutes |

### Calculating composite availability

#### Serial dependency formula
When all components must be healthy for the app to work:

```text
Composite availability = A x B x C
```

Example:

```text
App Service = 99.95% = 0.9995
SQL Database = 99.995% = 0.99995
Composite = 0.9995 x 0.99995 = 0.999450025 = 99.9450025%
```

#### Redundant component formula
When either of two independent components can serve traffic:

```text
Availability = 1 - ((1 - A) x (1 - B))
```

Example for two independent 99.95% app instances behind a load balancer:

```text
1 - (0.0005 x 0.0005) = 0.99999975 = 99.999975%
```

> **Exam trap:** do not multiply two active-active instances together as if both must failover in series. Use the **redundant formula** for parallel availability.

### Fault domains, update domains, and Availability Zones

| Concept | What it means | Best fit | Exam takeaway |
|---|---|---|---|
| **Fault Domain (FD)** | Separate rack/power/network boundary | Availability Sets | Protects from host/rack failure |
| **Update Domain (UD)** | Group rebooted separately for planned maintenance | Availability Sets | Protects from simultaneous planned maintenance |
| **Availability Zone (AZ)** | Physically separate datacenter in a region | Zonal or zone-redundant services | Protects from datacenter failure |

### Quick commands

```bash
RG=rg-ha-overview
LOCATION=eastus2
AVSET=avset-ha
VM=vm-ha-01

az vm availability-set create \
  --resource-group $RG \
  --name $AVSET \
  --platform-fault-domain-count 2 \
  --platform-update-domain-count 5 \
  --sku aligned

az vm show -g $RG -n $VM --query "{name:name,zones:zones,availabilitySet:availabilitySet.id}" -o json
```

```powershell
Get-AzAvailabilitySet -ResourceGroupName $RG -Name $AVSET | Select-Object Name, PlatformFaultDomainCount, PlatformUpdateDomainCount
Get-AzVM -ResourceGroupName $RG -Name $VM | Select-Object Name, Zones, AvailabilitySetReference
```

---


<a id="4-compute-ha"></a>
## 4. Compute HA

### Availability Sets

Availability Sets are the classic HA option for Azure VMs when you need protection from **host/rack failures** and **planned maintenance** inside a **single datacenter boundary**.

#### Fault domains and update domains

- **Fault domains** spread VMs across separate hardware groups.
- **Update domains** sequence planned maintenance so all VMs are not rebooted together.
- Typical exam answer when the question says:
  - zones are **not available** in the region
  - legacy VM design must stay **single-region**
  - need **basic HA** without zonal requirements

#### When to use Availability Sets

- Existing VM workloads not redesigned for zones
- Region or service does not support Availability Zones
- Requirement is **rack/host maintenance isolation**, not datacenter isolation

#### Limitations

- No datacenter-level isolation
- VM-only pattern; not a broad PaaS answer
- Not a multi-region solution
- Generally weaker than Availability Zones for new production designs

```bash
az vm availability-set create \
  --resource-group $RG \
  --name avset-prod \
  --platform-fault-domain-count 2 \
  --platform-update-domain-count 5 \
  --sku aligned
```

```powershell
New-AzAvailabilitySet -ResourceGroupName $RG -Location $LOCATION -Name "avset-prod" -Sku aligned -PlatformFaultDomainCount 2 -PlatformUpdateDomainCount 5
```

### Availability Zones

Availability Zones are the preferred AZ-305 answer when the requirement says **survive a datacenter outage in a region**.

#### Zone-redundant deployment

- **Zonal** = resource pinned to one zone
- **Zone-redundant** = service automatically spread across zones
- For architect-level decisions, prefer **built-in zone-redundant PaaS** over self-managed VM redundancy when it meets requirements

#### Zonal vs. zone-redundant resources

| Pattern | Meaning | Example | Best use |
|---|---|---|---|
| **Zonal** | Single instance in one zone | VM in Zone 1 | Low-latency placement, explicit fault isolation |
| **Zone-redundant** | Service spans zones | Standard Load Balancer, ZRS storage, zone-redundant SQL/App Service options | Higher in-region availability with lower ops |

#### Services commonly used with Availability Zones

- Virtual Machines
- Virtual Machine Scale Sets
- Managed disks, including **Premium SSD v2 / Premium SSD / Standard SSD ZRS** where supported
- Standard Public IP and Standard Load Balancer
- Application Gateway v2
- AKS node pools
- SQL Database / SQL Managed Instance zone-redundant options
- Cosmos DB zone-redundant regions
- Storage redundancy options such as **ZRS** and **GZRS**
- Supported App Service plans in zone-enabled regions

#### Cross-zone load balancing

- Use **Standard Load Balancer** for regional L4 HA across zonal backends.
- Use **Application Gateway v2** for regional L7 HA.
- Use **Front Door** for global HTTP/HTTPS failover.

```bash
az vm create \
  --resource-group $RG \
  --name vm-zone1 \
  --image Ubuntu2204 \
  --size Standard_D2s_v5 \
  --zone 1 \
  --admin-username azureuser \
  --generate-ssh-keys

az vm create \
  --resource-group $RG \
  --name vm-zone2 \
  --image Ubuntu2204 \
  --size Standard_D2s_v5 \
  --zone 2 \
  --admin-username azureuser \
  --generate-ssh-keys
```

```powershell
$vm1 = Get-AzVM -ResourceGroupName $RG -Name "vm-zone1"
$vm2 = Get-AzVM -ResourceGroupName $RG -Name "vm-zone2"
$vm1.Zones
$vm2.Zones
```

### Virtual Machine Scale Sets

VMSS is the best answer when the requirement says **many identical VMs**, **autoscale**, **self-healing**, and **rolling upgrades**.

#### VMSS for HA

- Scale out across multiple instances
- Pair with **Standard Load Balancer** or **Application Gateway**
- Add autoscale and health probes
- Prefer **zone-spanning** deployment for production in supported regions

#### Zone-spanning VMSS

- Distributes instances across zones
- Stronger HA than single-zone scale sets
- Good answer for stateless web/app tiers

#### Health probes and automatic repairs

- Health probes determine whether an instance should receive traffic
- Automatic repairs replace unhealthy instances after grace period
- Rolling upgrades reduce blast radius

```bash
az vmss create \
  --resource-group $RG \
  --name vmss-ha \
  --image Ubuntu2204 \
  --instance-count 3 \
  --zones 1 2 3 \
  --orchestration-mode Uniform \
  --upgrade-policy-mode Rolling \
  --load-balancer lb-ha \
  --health-probe lbProbe \
  --admin-username azureuser \
  --generate-ssh-keys \
  --enable-automatic-repairs true \
  --automatic-repairs-grace-period PT30M

az vmss list-instances -g $RG -n vmss-ha -o table
```

```powershell
Get-AzVmss -ResourceGroupName $RG -VMScaleSetName "vmss-ha"
Get-AzVmssVM -ResourceGroupName $RG -VMScaleSetName "vmss-ha"
```

### AKS HA

AKS HA means designing both **cluster placement** and **application placement**.

#### Zone-redundant AKS

- Spread node pools across multiple zones
- Run multiple replicas for critical pods
- Use Standard Load Balancer and zone-aware node pools
- Avoid single replica system-critical workloads

#### Multi-region AKS

- Separate cluster per region
- Use Front Door or Traffic Manager for failover/routing
- Externalize state to geo-replicated data services
- Use GitOps/IaC so both regions stay consistent

#### Pod disruption budgets

- Prevent voluntary disruptions from draining too many pods at once
- Common exam answer with maintenance-sensitive workloads
- Combine with readiness/liveness probes and multiple replicas

```bash
az aks create \
  --resource-group $RG \
  --name aks-ha \
  --location $LOCATION \
  --node-count 3 \
  --zones 1 2 3 \
  --network-plugin azure \
  --load-balancer-sku standard \
  --generate-ssh-keys

az aks get-credentials -g $RG -n aks-ha --overwrite-existing
kubectl create deployment api --image=mcr.microsoft.com/azuredocs/aks-helloworld:v1 --replicas=3
kubectl create pdb api-pdb --selector app=api --min-available 2
kubectl get nodes -L topology.kubernetes.io/zone
```

```powershell
Get-AzAksCluster -ResourceGroupName $RG -Name "aks-ha"
```

### App Service HA

App Service is often the AZ-305 best answer for web/API workloads because it gives **autoscale, patching, deployment slots, and lower ops** than VM-based designs.

#### Zone-redundant App Service

- Use supported plans in zone-enabled regions
- Scale to multiple workers
- Pair with deployment slots and external session state
- Good answer when the app can stay on PaaS

#### Multi-region with Traffic Manager or Front Door

- **Traffic Manager**: DNS-based routing, broader protocol support, slower failover due to DNS caching
- **Front Door**: global HTTP/HTTPS entry, faster failover, WAF, TLS offload, edge routing

```bash
PLAN=asp-ha
APP1=apphaeast$RANDOM
APP2=apphawest$RANDOM

az appservice plan create -g $RG -n $PLAN --sku P1v3 --is-linux
az appservice plan update -g $RG -n $PLAN --number-of-workers 3 --zone-redundant true
az webapp create -g $RG -p $PLAN -n $APP1 --runtime "PYTHON:3.11"
az appservice plan show -g $RG -n $PLAN --query "{name:name,sku:sku.name,workers:numberOfWorkers}" -o json
```

```powershell
Get-AzAppServicePlan -ResourceGroupName $RG -Name $PLAN | Select-Object Name, Sku, WorkerSize, NumberOfWorkers
Get-AzWebApp -ResourceGroupName $RG -Name $APP1 | Select-Object Name, State, HostNames
```

---


<a id="5-database-ha"></a>
## 5. Database HA

### SQL Database

Azure SQL Database should usually be your preferred answer for relational HA when the workload fits PaaS.

#### Zone-redundant configuration

- Adds in-region protection across zones where supported
- Strong exam answer when the requirement says **survive datacenter failure in-region**

#### Business Critical tier (built-in HA)

- Designed for lower latency and higher resilience with multiple replicas
- Best answer for mission-critical OLTP with strict availability needs

#### Read replicas

- Use readable secondaries via **active geo-replication**, **failover groups**, or **read scale-out** patterns where supported
- Improves read availability and offloads primary workload

```bash
SQLSERVER=sqlha$RANDOM
DB=appdb

az sql server create -g $RG -n $SQLSERVER -l $LOCATION -u sqladmin -p "P@ssw0rd1234!"
az sql db create \
  -g $RG \
  -s $SQLSERVER \
  -n $DB \
  --service-objective BC_Gen5_2 \
  --zone-redundant true \
  --read-scale Enabled
```

```powershell
Get-AzSqlDatabase -ResourceGroupName $RG -ServerName $SQLSERVER -DatabaseName $DB | Select-Object DatabaseName, Edition, RequestedServiceObjectiveName, ZoneRedundant, ReadScale
```

### SQL Managed Instance

#### Business Critical HA

- Built-in HA with multiple replicas
- Strong answer when app compatibility requires near-full SQL Server surface area but you still want managed service HA

#### Zone redundancy

- Improves in-region resilience where supported
- Cost is higher, but ops are lower than self-managed SQL on IaaS VMs

```bash
az sql mi show -g $RG -n mi-ha --query "{name:name,sku:sku.name,zoneRedundant:zoneRedundant,state:state}" -o table
```

```powershell
Get-AzSqlInstance -ResourceGroupName $RG -Name "mi-ha" | Select-Object Name, Sku, ZoneRedundant, State
```

### Cosmos DB

Cosmos DB is the strongest HA answer when the exam emphasizes **global distribution**, **multi-region reads/writes**, and **managed failover**.

#### Single-region with Availability Zones

- Use a zone-redundant region for stronger in-region resilience
- Good answer when workload stays mostly single-region but cannot tolerate datacenter loss

#### Multi-region writes

- Best fit for very high availability and low-latency global writes
- Higher cost and higher consistency design complexity

```bash
ACCOUNT=cosmosha$RANDOM

az cosmosdb create \
  --name $ACCOUNT \
  --resource-group $RG \
  --locations regionName=eastus failoverPriority=0 isZoneRedundant=True \
             regionName=centralus failoverPriority=1 isZoneRedundant=True \
  --enable-automatic-failover true \
  --enable-multiple-write-locations true

az cosmosdb show -g $RG -n $ACCOUNT --query "{name:name,automaticFailover:enableAutomaticFailover,multiWrite:enableMultipleWriteLocations,locations:locations[].locationName}" -o json
```

```powershell
Get-AzCosmosDBAccount -ResourceGroupName $RG -Name $ACCOUNT | Select-Object Name, EnableAutomaticFailover, EnableMultipleWriteLocations
```

### Other databases

#### PostgreSQL / MySQL HA options

- Azure Database for PostgreSQL Flexible Server supports **same-zone** and **zone-redundant HA** options
- Azure Database for MySQL Flexible Server supports HA options in supported regions
- Exam preference: choose managed database HA over self-managed database clustering unless a specific compatibility requirement forces IaaS

#### Redis clustering

- Use clustering/replication features for cache resilience
- Cache improves app resilience only if the app tolerates cache loss and can rebuild state
- For exam questions, do not treat Redis as the system of record

```bash
az postgres flexible-server create \
  --resource-group $RG \
  --name pg-ha-demo \
  --location $LOCATION \
  --tier GeneralPurpose \
  --sku-name Standard_D2ds_v4 \
  --storage-size 128 \
  --high-availability ZoneRedundant
```

```powershell
Get-AzPostgreSqlFlexibleServer -ResourceGroupName $RG -Name "pg-ha-demo" | Select-Object Name, HighAvailabilityMode, SkuTier, State
```

---


<a id="6-storage-ha"></a>
## 6. Storage HA

Storage availability decisions are usually about **durability vs. accessibility vs. failover scope**.

### LRS vs. ZRS vs. GRS vs. GZRS

| Redundancy | Copies | Scope | Best use | Exam note |
|---|---|---|---|---|
| **LRS** | 3 | Single datacenter | Lowest-cost local durability | Not zonal or regional HA |
| **ZRS** | 3 | Multiple zones in one region | In-region HA | Great when zonal resilience is required |
| **GRS** | 6 | Primary region + paired region | Regional durability | Secondary is not directly readable by default |
| **GZRS** | 6 | ZRS in primary + async to paired region | Zonal + regional resilience | Strong general-purpose exam answer for critical storage |

> **Important:** RA-GRS and RA-GZRS add **read access** to the secondary region. They are often the better exam answer when reporting or read-only continuity matters.

### Storage account HA best practices

- Prefer **ZRS** or **GZRS** for critical workloads needing stronger availability
- Use **immutable backup/restore strategy** separately for ransomware protection
- Pair storage redundancy with app failover logic; replication alone is not application failover
- Validate service support before choosing redundancy type for a workload

### Zone-redundant managed disks

- Use **Premium_ZRS** or **StandardSSD_ZRS** where supported
- Best for zonal VM architectures where disk availability must survive zone issues without pinning storage to one zone

```bash
STG=stgha$RANDOM
DISK=disk-ha-zrs

az storage account create \
  --resource-group $RG \
  --name $STG \
  --location $LOCATION \
  --sku Standard_GZRS \
  --kind StorageV2

az disk create -g $RG -n $DISK --size-gb 128 --sku Premium_ZRS
```

```powershell
Get-AzStorageAccount -ResourceGroupName $RG -Name $STG | Select-Object StorageAccountName, SkuName, Location
Get-AzDisk -ResourceGroupName $RG -DiskName $DISK | Select-Object Name, Sku, DiskSizeGB
```

---


<a id="7-networking-ha"></a>
## 7. Networking HA

### Load Balancer

- Use **Standard SKU** for production
- Supports zone-aware and zone-redundant patterns
- Best for TCP/UDP regional HA

```bash
az network lb create \
  --resource-group $RG \
  --name lb-ha \
  --sku Standard \
  --public-ip-address pip-ha \
  --frontend-ip-name fe-ha \
  --backend-pool-name be-ha

az network lb probe create \
  --resource-group $RG \
  --lb-name lb-ha \
  --name lbProbe \
  --protocol Http \
  --port 80 \
  --path /healthz
```

```powershell
Get-AzLoadBalancer -ResourceGroupName $RG -Name "lb-ha" | Select-Object Name, Sku, FrontendIpConfigurations, BackendAddressPools
```

### Application Gateway v2

- Use **Standard_v2** or **WAF_v2** for autoscaling and zone redundancy
- Best regional L7 option for HTTP/HTTPS with path routing and WAF

```bash
az network application-gateway create \
  --resource-group $RG \
  --name appgw-ha \
  --location $LOCATION \
  --sku WAF_v2 \
  --capacity 2 \
  --zones 1 2 3 \
  --vnet-name vnet-ha \
  --subnet appgw-subnet \
  --public-ip-address pip-appgw
```

```powershell
Get-AzApplicationGateway -ResourceGroupName $RG -Name "appgw-ha" | Select-Object Name, Sku, OperationalState, Zones
```

### VPN Gateway

- Use **active-active** for better tunnel resiliency
- Prefer **AZ SKUs** such as `VpnGw2AZ` when zonal resiliency is required

```bash
az network vnet-gateway create \
  --resource-group $RG \
  --name vpngw-ha \
  --public-ip-addresses pip-vpngw-1 pip-vpngw-2 \
  --vnet vnet-ha \
  --gateway-type Vpn \
  --vpn-type RouteBased \
  --sku VpnGw2AZ \
  --active-active
```

```powershell
Get-AzVirtualNetworkGateway -ResourceGroupName $RG -Name "vpngw-ha" | Select-Object Name, GatewayType, VpnType, ActiveActive, Sku
```

### ExpressRoute redundancy

- ExpressRoute circuits are designed with redundant Microsoft edge devices
- Architect for dual connections, dual routers, and ideally dual peering locations/providers for stronger enterprise resilience
- AZ-305 often tests that **circuit redundancy does not automatically mean end-to-end network redundancy**

### Front Door for global HA

- Best answer for global HTTP/HTTPS applications needing fast failover, SSL offload, WAF, and edge acceleration
- Prefer over Traffic Manager when application-layer routing and faster failover matter

```bash
az afd profile create -g $RG -n afd-ha --sku Standard_AzureFrontDoor
az afd endpoint create -g $RG --profile-name afd-ha -n endpoint-ha --enabled-state Enabled
az afd origin-group create -g $RG --profile-name afd-ha -n og-ha --probe-request-type GET --probe-protocol Https --probe-path /health
```

```powershell
Get-AzFrontDoor -ResourceGroupName $RG -Name "afd-ha"
```

---


<a id="8-multi-region-ha-patterns"></a>
## 8. Multi-Region HA Patterns

### Active-active multi-region

- Both regions serve traffic
- Strongest availability posture
- Highest cost and data consistency complexity
- Best answer for global apps needing very low RTO and minimal service interruption

### Active-passive multi-region

- Primary handles traffic; secondary is ready or warm
- Common enterprise exam answer when cost matters
- Easier than active-active for stateful workloads

### Traffic distribution strategies

| Strategy | Best service | When to use | Caveat |
|---|---|---|---|
| **HTTP global load balancing** | Front Door | Web apps/APIs, WAF, fast failover | HTTP/HTTPS only |
| **DNS failover** | Traffic Manager | Cross-region endpoint routing for many endpoint types | DNS TTL slows failover |
| **Regional L4** | Standard Load Balancer | VM/VMSS TCP/UDP HA in one region | Not global |
| **Regional L7** | App Gateway v2 | Path routing, TLS, WAF within one region | Not global by itself |

### Data synchronization challenges

- Write conflicts in active-active architectures
- Replication lag and asynchronous failover exposure
- Session affinity and state management problems
- Schema drift between regions if deployment governance is weak

### Consistency vs. availability trade-offs

- Stronger consistency can reduce write availability or increase latency
- Higher availability often means asynchronous replication and conflict management
- Cosmos DB consistency models are a classic exam trade-off topic
- For SQL, failover groups improve continuity but do not eliminate replication lag considerations

```bash
az network traffic-manager profile create \
  --resource-group $RG \
  --name tm-ha \
  --routing-method Priority \
  --unique-dns-name tmha$RANDOM \
  --ttl 30 \
  --protocol HTTP \
  --port 80 \
  --path /health
```

```powershell
Get-AzTrafficManagerProfile -ResourceGroupName $RG -Name "tm-ha" | Select-Object Name, RoutingMethod, MonitorProtocol, Ttl
```

> **Architect guidance:** use **active-active** only when the business case justifies the extra cost, operational maturity, and data design complexity.

---


<a id="9-sla-calculations"></a>
## 9. SLA Calculations

### Single service SLA

Use the SLA published for the exact deployment pattern. A single VM has a much weaker answer than two zonal instances behind a load balancer.

### Composite SLA formula

For dependent services in series:

```text
Acomposite = A1 x A2 x A3
```

For redundant parallel nodes:

```text
Aparallel = 1 - ((1 - A1) x (1 - A2))
```

### Improving SLA with redundancy

- Single instance -> weakest design
- Add redundant compute -> much stronger availability
- Add zone redundancy -> protects from datacenter failure
- Add multi-region -> protects from regional failure, but increases complexity and cost

### Example calculations for common architectures

| Architecture | Calculation | Result |
|---|---|---|
| **Single App Service (99.95%)** | 0.9995 | **99.95%** |
| **App Service + SQL DB** | 0.9995 x 0.99995 | **99.9450025%** |
| **Two 99.95% app nodes in parallel** | 1 - (0.0005 x 0.0005) | **99.999975%** |
| **Standard Load Balancer (99.99%) + two 99.95% app nodes in parallel** | 0.9999 x 0.99999975 | **99.9899750025%** |
| **Front Door + regional app stack** | Multiply Front Door SLA by healthy regional stack SLA | Depends on backend design |

### Architect exam shortcuts

- If the question says **99.99% or better**, think **zones or built-in zone-redundant PaaS**.
- If the question says **regional outage**, SLA alone is not enough; think **multi-region**.
- If the question says **minimal data loss**, shift from SLA thinking to **RPO** thinking.

---


<a id="10-resilience-patterns"></a>
## 10. Resilience Patterns

HA architecture becomes durable only when the application is also resilient.

### Retry with exponential backoff

- Use for transient failures
- Avoid retry storms with jitter and upper bounds
- Built into many Azure SDK guidance patterns

### Circuit breaker pattern

- Stop sending requests to a failing dependency temporarily
- Prevents cascading failures
- Strong answer when a downstream service is intermittently failing

### Bulkhead pattern

- Isolate resource pools so one failing component does not exhaust the whole app
- Example: dedicated worker pool per workload type

### Queue-based load leveling

- Decouple producers from consumers
- Smooth spikes and improve availability during downstream slowdown
- Common answer for bursty workloads

### Health endpoint monitoring

- Use `/health` or `/ready` endpoints
- Integrate with Load Balancer, App Gateway, Front Door, and AKS probes
- Pair with Azure Monitor alerts for proactive detection

```bash
SBNS=sbha$RANDOM

az servicebus namespace create -g $RG -n $SBNS -l $LOCATION --sku Standard
az servicebus queue create -g $RG --namespace-name $SBNS -n work-items

az monitor metrics alert create \
  --resource-group $RG \
  --name cpu-high-alert \
  --scopes $(az vm show -g $RG -n vm-zone1 --query id -o tsv) \
  --condition "avg Percentage CPU > 80" \
  --description "Alert when CPU stays high"
```

```bash
az network application-gateway probe create \
  --resource-group $RG \
  --gateway-name appgw-ha \
  --name probe-health \
  --protocol Http \
  --host-name-from-http-settings true \
  --path /health \
  --interval 30 \
  --timeout 30 \
  --threshold 3
```

```powershell
Get-AzMetricAlertRuleV2 -ResourceGroupName $RG -Name "cpu-high-alert"
Get-AzServiceBusQueue -ResourceGroupName $RG -NamespaceName $SBNS -QueueName "work-items"
```

> **Exam mindset:** platform redundancy keeps infrastructure available; resilience patterns keep the **application behavior** stable during partial failure.

---


<a id="11-az-305-decision-scenarios"></a>
## 11. AZ-305 Decision Scenarios

### Scenario 1: Single-region web tier must survive datacenter failure
**Situation:** A public web app must stay online if one datacenter in East US 2 fails.  
**Best answer:** **Availability Zones or zone-redundant App Service**  
**Why:** Availability Sets do not provide datacenter-level isolation.

### Scenario 2: Existing VM app in a region without zone support
**Situation:** Legacy VM-based line-of-business app must improve availability, but zones are unavailable in the selected region.  
**Best answer:** **Availability Set**  
**Why:** Fault/update domain separation is the correct in-region fallback.

### Scenario 3: Stateless web tier with autoscale and self-healing
**Situation:** Web servers must scale out automatically, replace failed instances, and support rolling upgrades.  
**Best answer:** **VMSS with Standard Load Balancer and health probes**  
**Why:** Native fleet HA and operations model.

### Scenario 4: Mission-critical relational database with strongest in-region HA
**Situation:** OLTP workload requires managed SQL with low latency and strong in-region HA.  
**Best answer:** **Azure SQL Database Business Critical with zone redundancy**  
**Why:** Built-in replica model plus zone resilience.

### Scenario 5: Global web application with low-latency failover
**Situation:** Users are worldwide and the app must fail over quickly between regions with WAF included.  
**Best answer:** **Azure Front Door with multi-region backends**  
**Why:** Global HTTP/HTTPS routing, health probes, and fast failover.

### Scenario 6: Cross-region failover for line-of-business web app at lower cost
**Situation:** Secondary region should be used only if the primary region fails.  
**Best answer:** **Active-passive architecture with Traffic Manager or Front Door plus replicated data**  
**Why:** Balanced resilience and cost.

### Scenario 7: Multi-region write-intensive NoSQL workload
**Situation:** Application requires low-latency writes in multiple regions with automatic failover.  
**Best answer:** **Cosmos DB with multi-region writes**  
**Why:** Native globally distributed write model.

### Scenario 8: AKS app must remain available during node maintenance
**Situation:** Cluster maintenance cannot evict too many pods at once.  
**Best answer:** **Multiple replicas + Pod Disruption Budget + multi-zone node pools**  
**Why:** Protects service continuity during planned disruption.

### Scenario 9: VPN connection cannot drop when one instance fails
**Situation:** Hybrid connectivity is business critical and must tolerate gateway instance loss.  
**Best answer:** **Active-active VPN Gateway with AZ SKU**  
**Why:** Stronger gateway redundancy and zonal resilience.

### Scenario 10: Need restore after accidental data deletion
**Situation:** Business asks for recovery after logical corruption and deletion, not just uptime.  
**Best answer:** **Backup plus HA as needed**  
**Why:** HA alone does not solve logical data loss.

---


<a id="12-quick-reference-trigger-table"></a>
## 12. Quick Reference Trigger Table

| # | Trigger phrase in the question | Best answer | Why |
|---|---|---|---|
| 1 | Survive host failure | Availability Set or Zones | Depends on datacenter isolation need |
| 2 | Survive datacenter failure in one region | Availability Zones | Separate datacenters in-region |
| 3 | Region outage | Multi-region design | Zones are not enough |
| 4 | Lowest ops for web/API HA | App Service | PaaS-first exam preference |
| 5 | HTTP/HTTPS global failover + WAF | Front Door | Edge routing and protection |
| 6 | DNS-based endpoint failover | Traffic Manager | DNS routing model |
| 7 | TCP/UDP HA inside one region | Standard Load Balancer | Regional L4 answer |
| 8 | Regional L7 routing + WAF | Application Gateway v2 | Regional HTTP/HTTPS gateway |
| 9 | Identical VM fleet with autoscale | VMSS | Native self-healing scale pattern |
| 10 | Legacy VM HA in one region | Availability Set | Good fallback when zones unavailable |
| 11 | Managed SQL with strongest in-region HA | SQL Database Business Critical | Built-in replica architecture |
| 12 | SQL app must span regions | SQL failover group | Managed cross-region failover |
| 13 | Multi-region NoSQL writes | Cosmos DB multi-write | Native global distribution |
| 14 | Cache resilience | Redis clustering/replication | Reduces cache single point of failure |
| 15 | In-region resilient storage | ZRS | Zone-redundant storage |
| 16 | Zonal + regional durable storage | GZRS | Combines zone and geo durability |
| 17 | Read access to storage secondary | RA-GRS / RA-GZRS | Readable secondary region |
| 18 | Zonal VM disk resilience | Premium_ZRS / StandardSSD_ZRS | Zone-redundant managed disks |
| 19 | Hybrid network gateway HA | Active-active VPN Gateway AZ SKU | Dual-instance resilient gateway |
| 20 | Enterprise private connectivity resilience | ExpressRoute redundancy | Dual links and provider diversity |
| 21 | AKS maintenance-safe rollout | PDB + readiness/liveness probes | Prevents too many pod disruptions |
| 22 | Fast global app failover | Front Door | App-layer health and edge routing |
| 23 | Lower-cost DR region | Active-passive / warm standby | Cheaper than active-active |
| 24 | Minimal data loss | RPO-driven replication choice | Availability percentage is not enough |
| 25 | Point-in-time recovery | Backup / PITR | HA is not restore capability |
| 26 | Need stronger SLA by redundancy | Parallel architecture | Redundant formula improves uptime |

---


<a id="13-common-exam-traps"></a>
## 13. Common Exam Traps

### Availability Set vs. Zone confusion
- **Wrong mindset:** Availability Set protects from datacenter failure.
- **Correct mindset:** Availability Set protects inside one datacenter boundary; **Availability Zones** protect across datacenters in a region.

### SLA calculation errors
- **Wrong mindset:** Add SLAs together or always multiply all instances.
- **Correct mindset:** multiply **dependent services**; use the **parallel redundancy formula** for redundant nodes.

### Zone-redundant service requirements
- **Wrong mindset:** Every Azure service is automatically zone-redundant in a zone-enabled region.
- **Correct mindset:** zone support is **service-, SKU-, and region-specific**. Verify support before locking design.

### HA vs. DR conflation
- **Wrong mindset:** A zonal architecture solves regional disaster recovery.
- **Correct mindset:** HA addresses local failures; DR addresses regional/site failure and is measured by **RTO/RPO**.

### Backup vs. HA conflation
- **Wrong mindset:** Geo-replication or HA means deleted data can be restored.
- **Correct mindset:** you still need **backup, retention, and restore testing**.

### Front Door vs. Traffic Manager confusion
- **Wrong mindset:** They solve the same problem the same way.
- **Correct mindset:** Front Door is a global HTTP/HTTPS reverse proxy; Traffic Manager is DNS-based routing.

### Active-active overuse
- **Wrong mindset:** Most critical apps need active-active.
- **Correct mindset:** active-active is justified only when the business requires very low RTO/RPO and accepts complexity/cost.

### Stateful tier design mistakes
- **Wrong mindset:** Only the web tier needs HA.
- **Correct mindset:** the **data tier and session state** usually decide whether failover actually works.

### Final study focus

1. Start by classifying the requirement as **HA**, **DR**, **backup**, or a combination.  
2. Prefer **managed, zone-redundant PaaS** when it meets business requirements.  
3. Use **Availability Zones** for datacenter failure, **multi-region** for regional failure.  
4. Watch for **RTO/RPO**, not just SLA percentages.  
5. Remember that **application resilience patterns** are part of the answer, not just infrastructure placement.

---

<a id="14-cost-optimization"></a>
## 14. Cost Optimization

High availability design is always a **tradeoff between downtime risk and redundancy cost**. On AZ-305, the correct answer is rarely the absolute cheapest design — it is the design that meets the stated availability target with the **lowest justified operational and financial overhead**.

### Availability Sets vs. Availability Zones — Cost Impact

| Design Choice | Direct Azure Cost Impact | Hidden / Indirect Cost | Best Use |
|---|---|---|---|
| **Availability Set** | No zone premium by itself | More self-management, weaker failure isolation, may still require load balancer + patching effort | Legacy VM HA where zones are unavailable or unnecessary |
| **Availability Zones** | Usually no separate "zone fee" for VMs, but you pay for extra instances and sometimes inter-zone data transfer | Higher architecture complexity, cross-zone latency/data charges, zonal SKUs may increase spend | Production apps that must survive datacenter failure |
| **Zone-redundant PaaS** | Some services/SKUs add a premium for zone redundancy | Lower admin cost, fewer DIY HA components, faster recovery | Best for managed services when low ops matters |
| **Active-passive multi-region** | Pays for standby capacity plus replication | Lower than active-active, but secondary still costs money | Regional resilience with controlled spend |
| **Active-active multi-region** | Highest compute, data replication, and networking cost | Highest testing and data-consistency complexity | Only when business requires ultra-low RTO/RPO |

### Practical AZ-305 Cost Callouts

- **Availability Sets are cheaper than zones only if the business can accept single-datacenter risk**
- **Zones often win overall** when the cost of downtime is higher than the incremental redundancy spend
- **Inter-zone traffic** can become a meaningful cost factor for chatty, stateful architectures
- **Zone-redundant PaaS** may cost more per service, but usually saves money in operations, patching, and failover design
- **Active-passive** is the common cost-balanced answer when the exam emphasizes business continuity but not continuous dual-region traffic
- **Active-active** is rarely the default answer; choose it only when the scenario explicitly justifies the cost and complexity

### Quick Savings Heuristics

1. Prefer **managed zone-redundant PaaS** before building custom VM clusters
2. Use **Availability Sets** only when zones are unsupported or unnecessary
3. Avoid cross-zone chatty stateful designs unless the availability target truly requires them
4. Right-size standby environments — warm standby is often enough for exam scenarios
5. Read carefully for **RTO/RPO**: overbuilding HA is just as wrong as underbuilding it

---

<a id="15-final-az-305-exam-tips"></a>
## 🎯 Final AZ-305 Exam Tips

1. **Classify the failure scope first** — host, datacenter, region, or logical data loss
2. **Availability Sets are not datacenter protection** — they only improve resiliency inside one datacenter boundary
3. **Availability Zones are the best in-region answer** when the requirement says survive a datacenter outage
4. **Regional failure requires multi-region architecture** — zones alone are not enough
5. **Prefer managed, zone-redundant PaaS** when it meets the requirement; it is usually the strongest architect answer
6. **Standard Load Balancer = L4; Application Gateway = L7; Front Door = global HTTP/HTTPS**
7. **SLA is not the same as RTO/RPO** — read the wording carefully
8. **HA does not replace backup** — accidental deletion and corruption still require restore capability
9. **Active-active is a premium design choice** — only pick it when the business truly needs the added continuity
10. **Watch for hidden clues** like WAF, path routing, DNS failover, zone support, and legacy VM constraints

---

<a id="16-architecture-decision-flowchart"></a>
## 📐 Architecture Decision Flowchart

```
Need to design Azure availability?
│
├── Failure scope = host / rack / planned maintenance only?
│   └── YES → Availability Set or multi-instance VMSS
│
├── Failure scope = datacenter in one region?
│   └── YES → Availability Zones or zone-redundant PaaS
│
├── Failure scope = whole region?
│   └── YES → Active-passive or active-active multi-region
│       ├── HTTP/HTTPS app? → Front Door
│       ├── Any endpoint type with DNS failover? → Traffic Manager
│       └── Replicate data with service-native DR features
│
├── Need lowest operations?
│   └── YES → Prefer App Service / SQL / Cosmos DB / ZRS or GZRS storage
│
├── Need VM control, custom OS, or legacy software?
│   └── YES → IaaS pattern with Availability Sets or Zones + LB/App Gateway
│
└── Need restore after deletion/corruption?
    └── YES → Add backup/PITR; HA alone is insufficient
```

---

<a id="17-exam-style-review-questions"></a>
## Exam-Style Review Questions

1. A company runs a public web app on two Azure VMs in a region that supports Availability Zones. The app must remain online if one datacenter fails, and traffic is HTTP/HTTPS with WAF requirements. **What should you choose?**  
   **Answer:** Deploy across **Availability Zones** and use **Application Gateway v2 (WAF)**.

2. A legacy line-of-business application must stay on Azure VMs in a region where zones are unavailable. The business wants protection from planned maintenance and host failure only. **What is the best fit?**  
   **Answer:** **Availability Set**.

3. An architect is asked to improve availability for a managed relational database while keeping operational overhead low and surviving a datacenter outage in-region. **What is the preferred answer?**  
   **Answer:** **Azure SQL Database with zone redundancy** (or another managed zone-redundant PaaS database if requirements differ).

4. A workload must fail over between two regions, but the business wants to minimize cost and only use the secondary during an outage. **Which pattern is usually best?**  
   **Answer:** **Active-passive multi-region** with **Traffic Manager or Front Door** plus replicated data.

5. A solution uses zone-redundant compute but has no backup strategy. The business asks whether this protects against accidental data deletion. **How should you respond?**  
   **Answer:** It does **not**. HA improves uptime; **backup/PITR** is still required for logical recovery.

---

*Last Updated: 2025 | Target: AZ-305 Designing Microsoft Azure Infrastructure Solutions*
