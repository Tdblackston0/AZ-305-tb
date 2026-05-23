> 📝 **Hands-On Labs:** [DR Labs](./Labs/Azure-DR-Labs.md)

# Azure Disaster Recovery Cheat Sheet + AZ-305 Exam Prep

**Perspective:** Senior Cloud Solution Architect  
**Primary AZ-305 Domain:** Design business continuity solutions (15-20%)

Use this file to map business requirements to **RTO**, **RPO**, DR patterns, and Azure-native recovery services. The exam usually rewards the **simplest architecture that still meets recovery objectives**.

---

## 1. Disaster Recovery Overview

### DR vs Backup vs HA

| Capability | Primary goal | Typical scope | What it does well | What it does not solve by itself |
|---|---|---|---|---|
| **Backup** | Recover data | File, VM, DB, workload data | Point-in-time restore, long retention, ransomware recovery | Fast application failover |
| **High Availability (HA)** | Survive local failure | Host, rack, zone, service instance | Reduces downtime in a region | Regional disaster recovery |
| **Disaster Recovery (DR)** | Recover service after major outage | Region, datacenter, site, platform | Restores business operations after major failure | Fine-grained point-in-time recovery alone |

**Architect rule:**  
- Use **backup** for deletion, corruption, ransomware, and retention.  
- Use **HA** for local component/datacenter failures.  
- Use **DR** for site or regional failure.  
- Most enterprise workloads need **all three**.

### RTO and RPO definitions

- **RTO (Recovery Time Objective):** maximum acceptable downtime.
- **RPO (Recovery Point Objective):** maximum acceptable data loss measured in time.

**Examples**
- Payroll app: **RTO 8 hours / RPO 4 hours** -> backup/restore or pilot light may be enough.
- Online banking: **RTO < 15 minutes / RPO near 0** -> hot standby or active-active with synchronous/near-synchronous data replication.
- Internal wiki: **RTO 24 hours / RPO 12 hours** -> low-cost backup and restore.

### Business impact analysis (BIA)

A strong DR design starts with BIA inputs:
- Critical business process
- Financial/regulatory impact of outage
- Dependency map (identity, DNS, network, DB, secrets, monitoring)
- Acceptable downtime and data loss
- Peak load during failover
- Manual vs automated recovery requirements

### DR tiers and patterns

| Tier | Pattern | Cost | Complexity | Typical RTO | Typical RPO |
|---|---|---:|---:|---|---|
| Tier 0 | Backup and restore | Lowest | Low | Hours to days | Hours |
| Tier 1 | Pilot light | Low | Medium | Hours | Minutes to hours |
| Tier 2 | Warm standby | Medium | Medium | Minutes to hours | Minutes |
| Tier 3 | Hot standby | High | High | Minutes | Seconds to minutes |
| Tier 4 | Active-active | Highest | Highest | Seconds to minutes | Near 0 to seconds |

### Command snapshot

```bash
# Example: create a Recovery Services vault for backup/DR governance
az backup vault create \
  --name rsv-dr-prod-eus \
  --resource-group rg-dr-prod-eus \
  --location eastus \
  --storage-model-type GeoRedundant
```

```powershell
# Get business-critical recovery services inventory
Get-AzRecoveryServicesVault | Select-Object Name, ResourceGroupName, Location
```

---

## 2. RTO/RPO Design

### RTO/RPO requirements matrix

| Workload class | Example | RTO target | RPO target | Design implication |
|---|---|---:|---:|---|
| Mission critical | Trading, payments | < 15 min | Near 0 | Active-active or hot standby; native DB failover |
| Business critical | ERP, CRM | < 1 hour | < 15 min | Warm or hot standby; automated failover |
| Important | Intranet, analytics refresh | < 4 hours | < 1 hour | Warm standby or pilot light |
| Standard | Line-of-business app | < 24 hours | < 4 hours | Pilot light or backup/restore |
| Noncritical | Dev/test | 1-3 days | 24 hours+ | Backup/restore + IaC rebuild |

### Cost vs RTO/RPO trade-offs

| Design choice | Cost impact | Benefit | Common exam angle |
|---|---:|---|---|
| Multi-region always-on | High | Lowest RTO | Use only when justified by strict requirements |
| Scaled-down warm standby | Medium | Good balance | Common best answer |
| Backup only | Low | Cheapest | Only valid when downtime is acceptable |
| Synchronous replication | High | Lowest RPO | Needed only when data loss must approach zero |
| Asynchronous replication | Medium | Lower cost/latency | Common for cross-region DR |

### SLA mapping to architecture

**Remember:** SLA is not the same as DR.
- **SLA** = Microsoft uptime commitment for a service.
- **RTO/RPO** = your business recovery targets.
- A 99.99% SLA does **not** mean you have backup or regional failover.

| Need | Architecture cue |
|---|---|
| Survive zonal failure | Zone-redundant or multi-zone design |
| Survive regional failure | Cross-region DR |
| Recover deleted data | Backup/PITR/soft delete/immutability |
| Keep same app endpoint after DB failover | Failover group listener |

### Table: RTO/RPO targets -> recommended architecture

| RTO / RPO target | Recommended architecture | Typical Azure services |
|---|---|---|
| Seconds / near 0 | Active-active with global routing + multi-region data | Front Door, Cosmos DB multi-region writes, SQL sync patterns |
| < 15 min / < 5 min | Hot standby | ASR, SQL failover groups, Redis geo-replication |
| < 1 hour / < 15 min | Warm standby | ASR + recovery plans, SQL failover groups, GRS/GZRS |
| < 4 hours / < 1 hour | Pilot light | Replicated DB, IaC, automation runbooks |
| 8-24 hours / 4-24 hours | Backup and restore | Azure Backup, SQL PITR, Storage account redundancy |

### Command snapshot

```bash
# Example: create an Azure SQL failover group for business-critical databases
az sql failover-group create \
  --name fog-sales-prod \
  --partner-server sql-dr-westus \
  --resource-group rg-data-prod \
  --server sql-prod-eastus \
  --failover-policy Automatic \
  --grace-period 1 \
  --add-db salesdb inventorydb
```

```powershell
# Inspect storage account geo-replication lag when designing storage RPO
Get-AzStorageAccount -ResourceGroupName rg-data-prod -Name stproddata |
  Select-Object StorageAccountName, PrimaryLocation, SecondaryLocation, GeoReplicationStats
```

---

## 3. DR Patterns

### Backup and restore
- **Highest RTO, lowest cost**
- Rebuild compute from IaC, restore data from backup/replica
- Best for dev/test, low-criticality apps, long tolerated downtime

### Pilot light
- Core data and minimal services replicated
- Most app tiers deployed only during disaster
- Good when cost matters more than fast cutover

### Warm standby
- Full stack exists in secondary region at reduced scale
- Scale up during failover
- Common enterprise sweet spot for AZ-305

### Hot standby / Active-active
- **Hot standby:** secondary environment fully deployed and nearly ready
- **Active-active:** both regions actively serve traffic
- Lowest RTO, highest run cost, highest operational complexity

### Decision matrix for pattern selection

| Pattern | When to choose | Pros | Cons | Exam shortcut |
|---|---|---|---|---|
| Backup/restore | Cheap DR, noncritical app | Lowest cost | Slowest recovery | Good for dev/test or loose RTO |
| Pilot light | Need lower cost but some prepared core services | Better than backup-only | More automation required | Often chosen for moderate RTO |
| Warm standby | Need balanced cost and recovery speed | Practical, scalable | Ongoing standby cost | Most common balanced answer |
| Hot standby | Fast recovery required | Low RTO | Higher cost | Choose when minutes matter |
| Active-active | App must stay online globally | Best continuity | Highest complexity | Use only for strict RTO/RPO |

### Command snapshot

```bash
# Traffic Manager profile for active-passive DNS-based regional failover
az network traffic-manager profile create \
  --resource-group rg-global \
  --name tm-dr-prod \
  --routing-method Priority \
  --unique-dns-name contoso-dr-prod \
  --ttl 30 \
  --protocol HTTPS \
  --port 443 \
  --path /
```

```powershell
# Scale out a warm standby App Service plan during failover
Set-AzAppServicePlan -ResourceGroupName rg-dr-westus -Name asp-dr-westus -NumberofWorkers 3
```

---

## 4. Azure Site Recovery (ASR)

### ASR architecture and components

Core components:
- **Recovery Services vault**
- **Source fabric** (Azure, VMware, Hyper-V, physical)
- **Replication policies**
- **Process server / configuration server** for some non-Azure scenarios
- **Protected items** (VMs/servers)
- **Recovery plans** for orchestration
- **Automation runbooks / scripts** for post-failover tasks

### Supported scenarios

| Scenario | Supported with ASR? | Typical use |
|---|---|---|
| Azure VM to Azure region | Yes | Azure-to-Azure DR |
| VMware to Azure | Yes | Datacenter exit / hybrid DR |
| Hyper-V to Azure | Yes | On-prem DR to Azure |
| Physical servers to Azure | Yes | Legacy workload protection |
| Azure SQL Database PaaS | No | Use native SQL DR features instead |
| Cosmos DB PaaS | No | Use native multi-region features |

### Replication policies
- Recovery point retention
- App-consistent snapshot frequency
- Replication frequency/objective by scenario
- Multi-VM consistency where workloads must recover together

### Recovery plans and runbooks
Use recovery plans to:
- Fail over **multiple VMs in sequence**
- Group app, middleware, and database tiers
- Insert scripts/runbooks for DNS, validation, scaling, and notifications

### Test failover vs planned failover vs unplanned failover

| Action | Use when | Downtime expectation | Key note |
|---|---|---|---|
| **Test failover** | DR drill | None to production | Uses isolated network; non-disruptive |
| **Planned failover** | Controlled migration or expected outage | Minimal | Source typically still available; zero/low data loss goal |
| **Unplanned failover** | Real outage | Based on DR design | Used when source is unavailable |

### Failback process
1. Fail over to secondary region/site.
2. Stabilize and validate workload.
3. Reprotect back to original or new primary.
4. Perform planned failback.
5. Commit and resume normal replication.

### ASR networking considerations
- Map source and target VNets/subnets
- Reserve IP strategy for DR region
- Ensure NSGs, UDRs, firewalls, DNS, and private endpoints are addressed
- Test dependencies on ExpressRoute/VPN, domain controllers, Key Vault, and load balancers
- For isolated tests, use a dedicated **test failover VNet**

### Capacity planning
Plan for:
- Target region quotas
- VM SKU availability in DR region
- Premium SSD / managed disk availability
- Recovery Services vault scale
- Peak failover concurrency
- Network egress and application startup surge

### ASR with Availability Zones
- Availability Zones improve **HA in-region**.
- ASR provides **cross-region/site DR**.
- Together they form layered BCDR.
- Typical pattern: zone-redundant primary + ASR to paired/approved DR region.

### Command snapshot

```bash
# Azure CLI extension is commonly required for Site Recovery commands
az extension add --name site-recovery
```

```powershell
# Set ASR vault context
$vault = Get-AzRecoveryServicesVault -Name "rsv-dr-prod-eus"
Set-AzRecoveryServicesAsrVaultContext -Vault $vault

# View replication protected items
Get-AzRecoveryServicesAsrReplicationProtectedItem

# Start a test failover for a protected VM
$plan = Get-AzRecoveryServicesAsrRecoveryPlan -Name "rp-contoso-app"
Start-AzRecoveryServicesAsrTestFailoverJob -RecoveryPlan $plan -Direction PrimaryToRecovery

# Planned failover
Start-AzRecoveryServicesAsrPlannedFailoverJob -RecoveryPlan $plan -Direction PrimaryToRecovery

# Unplanned failover
Start-AzRecoveryServicesAsrUnplannedFailoverJob -RecoveryPlan $plan -Direction PrimaryToRecovery
```

---

## 5. Azure-to-Azure DR

### Cross-region replication
- ASR is the main DR service for **Azure VM to Azure VM** failover.
- Data replication is asynchronous across regions.
- Good for IaaS workloads where you need orchestrated recovery, boot order, and failback.

### Supported region pairs
- Prefer **Azure paired regions** for platform alignment and capacity planning.
- Exam answer: if asked for regional resilience, paired regions are usually the default unless compliance/data residency dictates otherwise.

### Network mapping
- Precreate target VNet/subnets in DR region
- Map NICs/subnets during replication configuration
- Plan public IP, private DNS, load balancer/Front Door/Traffic Manager updates
- Validate route tables and firewall paths after failover

### Automation with recovery plans
Use recovery plans for:
- Multi-tier boot order
- Pause points for approval
- Azure Automation runbooks
- Post-failover scale-out and validation steps

### Multi-VM consistency
- Use **multi-VM consistency** for workloads like app + DB tiers needing crash-consistent aligned recovery points.
- It improves coordinated recovery but can increase replication overhead.

### Command snapshot

```bash
# Check region availability for a VM size during DR planning
az vm list-skus \
  --location westus2 \
  --resource-type virtualMachines \
  --query "[?name=='Standard_D4s_v5'].{Name:name,Zones:locationInfo[0].zones}" \
  --output table
```

```powershell
# Review Azure-to-Azure replication protected items in the vault
Get-AzRecoveryServicesAsrReplicationProtectedItem |
  Select-Object FriendlyName, ProtectionState, ActiveLocation, RecoveryAzureNetworkId
```

---

## 6. Database DR Options

### SQL Database

#### Active geo-replication
- Up to four readable secondaries
- Manual failover
- Good when you need granular control at database level

#### Auto-failover groups
- Better for multiple databases and app continuity
- Provides **listener endpoints** so apps do not need connection string changes
- Supports automatic failover with grace period

#### RPO and RTO characteristics
- **RPO:** typically seconds
- **RTO:** typically seconds to minutes depending on workload and DNS propagation

**Senior architect guidance:** if the scenario says **multiple databases + automatic failover + stable endpoint**, choose **failover groups**.

```bash
# Manual SQL failover to DR region
az sql failover-group set-primary \
  --name fog-sales-prod \
  --resource-group rg-data-prod \
  --server sql-dr-westus
```

```powershell
# Review failover group configuration
Get-AzSqlDatabaseFailoverGroup -ResourceGroupName rg-data-prod -ServerName sql-prod-eastus |
  Select-Object FailoverGroupName, ReplicationRole, ReadWriteEndpoint, ReadOnlyEndpoint
```

### SQL Managed Instance

#### Failover groups
- Native cross-region DR option for managed instance
- Best fit when app needs managed SQL features plus DR orchestration
- Use listener endpoints to abstract region-specific instance names

```bash
az sql instance-failover-group create \
  --name mifog-prod \
  --location eastus \
  --managed-instance sqlmi-prod-eastus \
  --partner-managed-instance sqlmi-dr-westus \
  --resource-group rg-data-prod \
  --failover-policy Automatic \
  --grace-period 1
```

### Cosmos DB

#### Multi-region writes
- Best for global apps needing low latency and strong continuity
- Supports active-active style writes when enabled

#### Automatic failover
- Cosmos DB can automatically promote regions based on failover priorities
- Native service capability; do not use ASR for Cosmos DB

#### Consistency during failover
- **Strong consistency** -> lowest RPO, highest write latency and region constraints
- **Session consistency** -> common balance for app performance
- **Eventual** -> lowest latency, higher staleness risk

```bash
# Enable automatic failover and inspect region priorities
az cosmosdb update \
  --name cdb-global-prod \
  --resource-group rg-data-prod \
  --enable-automatic-failover true

az cosmosdb show \
  --name cdb-global-prod \
  --resource-group rg-data-prod \
  --query "failoverPolicies"
```

### Other databases

#### PostgreSQL/MySQL read replicas
- Good for read scale and regional resilience patterns
- Replica promotion may be manual; check service-specific capabilities
- Common exam nuance: read replica is not always equivalent to full DR orchestration

#### Redis geo-replication
- Use for session state or cache continuity
- Important for stateful web apps that externalize session data

```powershell
# Cosmos DB account continuity review
Get-AzCosmosDBAccount -ResourceGroupName rg-data-prod -Name cdb-global-prod |
  Select-Object Name, EnableAutomaticFailover, Locations, ConsistencyPolicy
```

### Database DR summary table

| Service | Best DR feature | RTO | RPO | Exam cue |
|---|---|---|---|---|
| Azure SQL Database | Auto-failover group | Seconds to minutes | Seconds | Multiple DBs + listener endpoint |
| SQL Managed Instance | Failover group | Minutes | Seconds to minutes | Managed SQL DR across regions |
| Cosmos DB | Multi-region + auto failover | Seconds | Near 0 to seconds | Global app, multi-region writes |
| PostgreSQL/MySQL | Read replicas / service-native replication | Minutes to hours | Seconds to minutes | Read scale + replica promotion |
| Azure Cache for Redis | Geo-replication | Minutes | Seconds to minutes | Session state continuity |

---

## 7. Storage DR

### GRS and RA-GRS
- **GRS:** asynchronously replicates to secondary paired region
- **RA-GRS:** same as GRS, plus read access to secondary endpoint

### GZRS and RA-GZRS
- **GZRS:** zone redundancy in primary region + geo replication to secondary region
- **RA-GZRS:** GZRS with read access to secondary

### Storage account failover
- Customer-initiated failover promotes secondary to primary
- Failover is **not** instant and is typically for severe regional outage scenarios
- **Exam trap:** after failover, the account typically runs as **LRS in the new primary** until geo-redundancy is re-enabled/reconfigured

### Blob replication options
- Object replication for block blobs across accounts/regions
- Snapshot/versioning + soft delete for operational recovery
- Change feed for downstream recovery workflows and audit patterns

### Storage redundancy table

| Option | Protects against | Read secondary? | Best for |
|---|---|---|---|
| LRS | Local hardware failure | No | Lowest cost, same region |
| ZRS | Zone failure in region | No | HA within region |
| GRS | Regional failure | No | Durable geo replication |
| RA-GRS | Regional failure | Yes | Read access during primary outage |
| GZRS | Zone + regional failure | No | Stronger primary region resiliency |
| RA-GZRS | Zone + regional failure | Yes | Best storage continuity mix |

### Command snapshot

```bash
# View geo-replication health and last sync time
az storage account show \
  --name stproddata \
  --resource-group rg-data-prod \
  --query "geoReplicationStats"

# Initiate storage account failover
az storage account failover \
  --name stproddata \
  --resource-group rg-data-prod
```

```powershell
# Validate storage redundancy configuration
Get-AzStorageAccount -ResourceGroupName rg-data-prod -Name stproddata |
  Select-Object StorageAccountName, SkuName, PrimaryLocation, SecondaryLocation
```

---

## 8. Application DR Patterns

### Stateless application DR
- Best DR posture for web/API tiers
- Rehydrate from IaC + CI/CD in secondary region
- Store session and state externally
- Front Door or Traffic Manager directs traffic to healthy region

### Stateful application considerations
- Externalize state to managed DB/cache/storage
- Coordinate app and data failover order
- Validate message durability, idempotency, and replay behavior

### Session management in DR
- In-memory session -> poor DR choice
- Better options:
  - Redis geo-replication
  - Database-backed session store
  - Token-based stateless auth

### DNS-based failover (Traffic Manager, Front Door)

| Service | Best use | Strength | Limitation |
|---|---|---|---|
| Traffic Manager | DNS-based regional failover | Simple, global, protocol-agnostic | Depends on DNS TTL/client cache |
| Front Door | HTTP/HTTPS app failover | Faster app-layer failover, WAF, acceleration | Web workloads only |

### Command snapshot

```bash
# Add a high-priority secondary endpoint to Traffic Manager
az network traffic-manager endpoint create \
  --resource-group rg-global \
  --profile-name tm-dr-prod \
  --name app-dr-westus \
  --type azureEndpoints \
  --target-resource-id /subscriptions/<subId>/resourceGroups/rg-dr-westus/providers/Microsoft.Web/sites/app-dr-westus \
  --priority 2
```

```powershell
# Review Front Door backend health (Standard/Premium cmdlets may vary by module version)
Get-AzFrontDoor -ResourceGroupName rg-global -Name fd-contoso-prod |
  Select-Object Name, FrontendEndpoints, BackendPools
```

---

## 9. DR Testing

### DR drill importance
- A DR plan not tested is a DR plan not trusted
- Test people, process, tooling, and communications
- Capture measured RTO/RPO after each drill

### Test failover in ASR
- Use isolated VNet
- Validate app, DB, identity, DNS, and monitoring dependencies
- Clean up test artifacts after drill

### Non-disruptive testing
- ASR **test failover** is the standard example
- Read-only secondary endpoints can validate some storage/database patterns
- Use staged DNS cutover or synthetic probes for application validation

### Documenting and automating DR
Document:
- Trigger conditions
- Decision authority
- Recovery steps
- Validation checklist
- Failback plan
- Contact and escalation paths

Automate with:
- Recovery plans
- Azure Automation runbooks
- Logic Apps / Functions for notifications
- IaC to rebuild secondary region

### Command snapshot

```powershell
# Example cleanup after ASR test failover
$plan = Get-AzRecoveryServicesAsrRecoveryPlan -Name "rp-contoso-app"
Stop-AzRecoveryServicesAsrTestFailoverCleanupJob -RecoveryPlan $plan
```

```bash
# Example: kick off an automation runbook used by a DR recovery plan
az automation runbook start \
  --automation-account-name aa-dr-prod \
  --resource-group rg-dr-prod \
  --name Invoke-PostFailoverValidation
```

### Success criteria checklist
- Application reachable through intended endpoint
- Data is current within target RPO
- Dependencies recovered in correct order
- Monitoring and alerts re-enabled
- Runbooks and rollback/failback steps validated

---

## 10. DR for Specific Workloads

### AKS multi-region DR
- Prefer GitOps/IaC to rebuild clusters consistently
- Replicate container images and secrets strategy
- Protect persistent volumes with backup/replication approach
- Use Front Door/Traffic Manager for global ingress
- Active-active is possible but data tier design is the hard part

```bash
# Check AKS node pools in secondary region before a drill
az aks nodepool list \
  --resource-group rg-aks-dr \
  --cluster-name aks-dr-westus \
  --output table
```

### App Service DR
- Deploy app to two regions
- Use Front Door or Traffic Manager for failover
- Externalize state and secrets
- Use deployment slots for safe release, not as a regional DR substitute

```bash
az webapp show \
  --name app-dr-westus \
  --resource-group rg-dr-westus \
  --query "{HostNames:defaultHostName,State:state,Plan:serverFarmId}"
```

### Azure Functions DR
- Re-deploy code to second region
- Pair with geo-resilient triggers and state stores (Storage, Service Bus, Cosmos DB)
- Consumption/Premium hosting decision affects cold start and failover readiness
- Durable Functions need storage/task hub continuity planning

```powershell
Get-AzFunctionApp -ResourceGroupName rg-dr-westus -Name func-dr-westus |
  Select-Object Name, State, DefaultHostName, Location
```

### Virtual Desktop DR
- Protect host pools, session hosts, FSLogix profiles, images, and identity dependencies
- Replicate profile storage using Azure Files/NetApp strategies
- Automate session host rebuild in secondary region

```bash
# Review Azure Virtual Desktop host pool metadata
az desktopvirtualization hostpool show \
  --resource-group rg-avd-prod \
  --name hp-avd-prod
```

### Workload decision table

| Workload | Common DR answer | Key design note |
|---|---|---|
| AKS | Multi-region cluster + replicated data + global routing | Data/state usually hardest part |
| App Service | Dual-region deployment + Front Door | Keep app stateless |
| Azure Functions | Dual deployment + durable backend continuity | Protect storage/task hubs |
| AVD | Secondary host pool + profile replication | User profile recovery is critical |
| SQL on Azure VM | ASR or SQL AG | Decide between infra-level and app-native DR |

---

## 11. AZ-305 Decision Scenarios (10+ scenarios)

### Scenario 1: Internal HR portal
- **Requirement:** RTO 24 hours, RPO 8 hours, cost-sensitive
- **Best fit:** Backup and restore
- **Why:** Downtime tolerance is high; warm standby would be unnecessary cost

### Scenario 2: Regional e-commerce platform
- **Requirement:** RTO 30 minutes, RPO 5 minutes
- **Best fit:** Warm standby app tier + SQL failover group + Front Door
- **Why:** Balanced cost and fast failover

### Scenario 3: Global SaaS API
- **Requirement:** RTO < 5 minutes, RPO near 0, worldwide users
- **Best fit:** Active-active + Front Door + Cosmos DB multi-region writes
- **Why:** Needs always-on global footprint and very low data loss

### Scenario 4: SQL Database with multiple dependent databases
- **Requirement:** Automatic failover and no app connection string changes
- **Best fit:** Auto-failover group
- **Why:** Listener endpoint is the exam keyword

### Scenario 5: Azure VM-based legacy app
- **Requirement:** Keep same OS/app stack, orchestrated cross-region recovery
- **Best fit:** Azure Site Recovery
- **Why:** ASR is built for VM-level DR, not PaaS databases

### Scenario 6: Dev/test environment
- **Requirement:** Lowest cost DR; rebuild acceptable within 1-2 days
- **Best fit:** Backup + IaC
- **Why:** Standby environments waste cost for noncritical workloads

### Scenario 7: Stateful web application
- **Requirement:** Session state must survive failover; RTO < 15 minutes
- **Best fit:** Stateless web tier + Redis geo-replication + global routing
- **Why:** In-memory sessions fail during regional cutover

### Scenario 8: Manufacturing app with on-prem VMware
- **Requirement:** DR to Azure with limited datacenter footprint
- **Best fit:** ASR VMware-to-Azure
- **Why:** Common hybrid DR modernization path

### Scenario 9: Compliance-heavy backup requirement
- **Requirement:** 7-year retention, immutable recovery points, ransomware resistance
- **Best fit:** Azure Backup + immutability + soft delete + GRS where required
- **Why:** DR alone does not satisfy retention/security needs

### Scenario 10: Storage-heavy application
- **Requirement:** Read access to secondary region during outage analysis
- **Best fit:** RA-GRS or RA-GZRS
- **Why:** Read access to secondary is the deciding phrase

### Scenario 11: SQL Managed Instance workload
- **Requirement:** Managed PaaS SQL with regional failover
- **Best fit:** SQL Managed Instance failover group
- **Why:** Native platform DR beats VM-based workaround

### Scenario 12: AKS business-critical platform
- **Requirement:** Region outage resilience, RTO < 1 hour
- **Best fit:** Warm standby AKS in second region + replicated data + Front Door
- **Why:** Full active-active may be unnecessary unless latency or uptime is stricter

### Scenario 13: App must survive zonal outage but not regional outage
- **Requirement:** Same-region resiliency only
- **Best fit:** Availability Zones / zone-redundant services
- **Why:** This is HA, not DR

### Scenario 14: Messaging platform using Service Bus
- **Requirement:** Namespace continuity across regions
- **Best fit:** Service Bus Geo-DR for namespace + app retry strategy
- **Why:** Messaging namespace alias is the continuity feature; messages in flight still require design consideration

---

## 12. Quick Reference Trigger Table (25+ entries)

| # | Trigger phrase | Think first | Recommended answer |
|---:|---|---|---|
| 1 | Minimal downtime | Low RTO | Warm standby, hot standby, or active-active |
| 2 | Minimal data loss | Low RPO | Native replication or synchronous design |
| 3 | Regional outage | Cross-region DR | ASR, failover groups, GRS/GZRS, multi-region services |
| 4 | Zone outage | In-region HA | Availability Zones / ZRS |
| 5 | Accidental deletion | Backup | PITR, soft delete, vault backup |
| 6 | Ransomware protection | Immutable backup | Vault immutability + soft delete |
| 7 | VM DR | ASR | Azure Site Recovery |
| 8 | VMware to Azure | Hybrid DR | ASR |
| 9 | Multiple databases | Coordinated DB DR | SQL failover group |
| 10 | Stable database endpoint | Listener endpoint | Failover group |
| 11 | Global NoSQL app | Native global database | Cosmos DB multi-region |
| 12 | Read from secondary storage | Read access geo replication | RA-GRS / RA-GZRS |
| 13 | Lowest-cost DR | Rebuild acceptable | Backup and restore |
| 14 | Balanced cost and speed | Mid-tier DR | Warm standby |
| 15 | Fastest application failover | App-layer global routing | Front Door |
| 16 | DNS-based failover | DNS TTL matters | Traffic Manager |
| 17 | Session continuity | Externalize state | Redis geo-replication / token-based auth |
| 18 | DB on Azure VM | Infra or DB-native DR | ASR or SQL AG |
| 19 | PaaS database DR | Native feature first | Failover groups / replicas / Cosmos DB failover |
| 20 | Test without production impact | Non-disruptive drill | ASR test failover |
| 21 | Recovery sequencing | Multi-tier orchestration | ASR recovery plans |
| 22 | Need approval before failover | Governance | Manual or scripted failover workflow |
| 23 | Secondary region capacity | Pre-stage quotas/SKUs | Capacity planning |
| 24 | Pair region by default | Platform-aligned DR | Azure paired regions |
| 25 | Compliance retention | Backup, not just DR | LTR + immutability |
| 26 | Cross-region blob replication | Data copy pattern | Object replication |
| 27 | App survives local failure only | HA not DR | Zones / load balancing |
| 28 | Database failover with seconds RPO | Native DB replication | SQL failover group / geo-replication |
| 29 | Active-active writes | Highest complexity | Cosmos DB multi-region writes |
| 30 | Storage failover | Manual severe-event action | Customer-initiated account failover |

---

## 13. Common Exam Traps

### Geo-replication vs ASR confusion
- **Wrong:** Use ASR for Azure SQL Database or Cosmos DB
- **Right:** Use **native PaaS DR features** for PaaS databases; use **ASR for VMs/servers**

### RPO vs RTO confusion
- **Wrong:** “Need recovery in 10 minutes” -> choose lowest RPO
- **Right:** 10 minutes recovery time is **RTO**, not RPO

### Failover group listener endpoints
- **Wrong:** App connects directly to regional SQL server hostname
- **Right:** App connects to **failover group listener** for automatic endpoint abstraction

### Storage failover limitations
- Storage failover is not app failover
- Secondary data may lag based on async replication
- Post-failover redundancy state changes must be reviewed and reconfigured

### Additional traps
- Availability Zones are **HA**, not cross-region DR
- Traffic Manager failover is **not instant** because DNS TTL and client caching matter
- Backup alone does not provide low RTO application recovery
- Read replica does not always mean automatic failover
- ASR improves DR for infrastructure, but application/data dependencies still need architecture

### Command snapshot

```bash
# Check SQL failover group listener endpoints
az sql failover-group show \
  --name fog-sales-prod \
  --resource-group rg-data-prod \
  --server sql-prod-eastus \
  --query "{ReadWrite:readWriteEndpoint,ReadOnly:readOnlyEndpoint,Role:replicationRole}"
```

```powershell
# Review storage account redundancy after planning a failover procedure
Get-AzStorageAccount -ResourceGroupName rg-data-prod -Name stproddata |
  Select-Object StorageAccountName, SkuName, SecondaryLocation
```

---

## Senior Architect Exam Notes

1. Start every scenario by classifying the requirement as **backup**, **HA**, **DR**, or **BCDR**.
2. Prefer **native managed service DR** before custom VM-based solutions when both satisfy requirements.
3. Match architecture to the **required** RTO/RPO, not to the most resilient design possible.
4. Multi-region design increases not just cost, but also **operations, testing, and data consistency complexity**.
5. The exam often rewards **warm standby** as the practical middle ground.
6. Always validate dependencies: identity, DNS, secrets, networking, quotas, monitoring, and automation.

## Final Memory Aids

- **Backup** = recover data  
- **HA** = survive local failure  
- **DR** = survive regional/site failure  
- **RTO** = time lost  
- **RPO** = data lost  
- **ASR** = DR for VMs/servers  
- **Failover group** = SQL endpoint abstraction  
- **RA-GRS** = readable storage secondary  
- **Front Door** = fast HTTP/HTTPS failover  
- **Traffic Manager** = DNS-based failover  

---

**Exam tip:** if the requirement says **lowest cost**, lean toward backup/restore or pilot light. If it says **minimal downtime and minimal data loss**, lean toward hot standby or active-active, but only if the business explicitly justifies the cost.