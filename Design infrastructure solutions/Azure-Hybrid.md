> 📝 **Hands-On Labs:** [Hybrid Labs](./Labs/Azure-Hybrid-Labs.md)

# Azure Hybrid & Multi-Cloud Cheat Sheet + AZ-305 Exam Prep

**Audience:** Senior Cloud Solution Architect  
**Primary AZ-305 Domain:** Design infrastructure solutions (30–35%)  
**Secondary Domains:** Identity, governance, monitoring; business continuity; data storage

Use this as a fast revision guide for **hybrid infrastructure**, **multi-cloud governance**, and **architectural tradeoffs**. For the exam, always optimize for **business constraints first**: compliance, latency, connectivity, operations, resilience, and modernization path.

---

## 1. Hybrid Cloud Overview

### Why hybrid
Hybrid exists because many organizations cannot move everything to Azure at once.

Common drivers:
- **Data sovereignty:** keep regulated data in-country or on-premises.
- **Latency-sensitive workloads:** factory systems, trading platforms, branch manufacturing, healthcare imaging.
- **Legacy systems:** tightly coupled apps, hardware dependencies, old licensing models, mainframe integration.
- **Regulations and security:** air-gapped, classified, export-controlled, or sovereign workloads.
- **Gradual modernization:** keep stable systems on-prem while moving web/API/analytics tiers to Azure.
- **Business continuity:** use Azure as a secondary site or burst capacity target.

### Azure hybrid strategy
Microsoft’s hybrid strategy is built around **consistent control plane and operations** across:
- Azure
- On-premises datacenters
- Edge locations
- Other clouds (AWS/GCP)

Core pillars:
- **Azure Arc** for management and governance anywhere
- **Azure Stack portfolio** for Azure-consistent capabilities outside Azure
- **Hybrid networking** with VPN, ExpressRoute, and Virtual WAN
- **Hybrid identity** with Microsoft Entra ID + directory sync/federation
- **Centralized operations** with Azure Monitor, Defender for Cloud, Policy, and Update Manager

### Consistent management across environments
For AZ-305, “consistent management” usually means:
- Single inventory of resources in Azure
- Common tagging and governance
- Policy-based compliance
- Common monitoring and alerting
- Security posture management across clouds and on-prem
- Standardized deployment and configuration using GitOps/IaC

**Architect takeaway:** Hybrid is not just connectivity. It is **common governance + operations + security** across distributed estates.

---

## 2. Azure Arc

Azure Arc extends Azure management and selected Azure services to infrastructure outside Azure.

### Core Azure Arc value proposition
- Project non-Azure/on-prem resources into Azure Resource Manager
- Apply Azure Policy and RBAC consistently
- Deploy Azure extensions
- Use Azure Monitor and Defender for Cloud
- Govern servers, Kubernetes, SQL Server, VMware, and selected PaaS-like services

> **Exam trigger:** If the requirement says “manage on-prem/AWS/GCP resources from Azure with consistent governance,” think **Azure Arc** first.

### Arc-enabled Servers

#### What it is
Arc-enabled servers onboard **Windows and Linux servers** running outside Azure:
- Physical servers
- VMware VMs
- Hyper-V VMs
- AWS EC2
- GCP VMs
- Edge devices

#### Onboarding Windows/Linux servers
Typical onboarding flow:
1. Create an Arc onboarding script in Azure.
2. Install the Connected Machine agent.
3. Register the machine into Azure.
4. Assign policy, monitoring, and extensions.

**Azure CLI**
```bash
az extension add --name connectedmachine
az connectedmachine list -g rg-hybrid-prod -o table
```

**PowerShell**
```powershell
Get-AzConnectedMachine -ResourceGroupName rg-hybrid-prod
```

**Sample onboarding pattern (Windows PowerShell)**
```powershell
$resourceGroup = "rg-hybrid-prod"
$tenantId = "<tenant-id>"
$subscriptionId = "<subscription-id>"
$location = "eastus"

# Agent installation typically uses the Connected Machine agent package
# followed by registration with the Azure service principal or onboarding script.
```

#### Azure extensions on Arc servers
Common extensions:
- Azure Monitor Agent (AMA)
- Custom Script Extension
- Dependency Agent
- Defender extensions
- Guest configuration / policy extensions

Use cases:
- Configure monitoring
- Push scripts/configuration
- Enable security tooling
- Standardize server baselines

**Azure CLI**
```bash
az connectedmachine extension create \
  --machine-name arc-sql-01 \
  --resource-group rg-hybrid-prod \
  --location eastus \
  --name AzureMonitorWindowsAgent \
  --publisher Microsoft.Azure.Monitor \
  --type AzureMonitorWindowsAgent \
  --settings '{}'
```

#### Azure Policy for Arc servers
Use Azure Policy to enforce or audit:
- Required extensions
- Allowed locations/tags
- Guest configuration
- Security baselines
- Log collection standards

**Exam point:** Arc lets you apply Azure governance to non-Azure servers, but policy effects may depend on installed agents/extensions.

#### Azure Monitor for Arc servers
With AMA + data collection rules, you can:
- Send logs/metrics to Log Analytics
- Build alerts/workbooks
- Correlate Azure + non-Azure server telemetry

**Azure CLI**
```bash
az monitor data-collection rule list -g rg-monitoring -o table
```

#### Microsoft Defender for Arc servers
Defender for Cloud can protect Arc-connected servers for:
- Vulnerability assessment
- Secure score improvement
- Threat detection
- Regulatory compliance visibility

**Best use:** centralized security posture across Azure and non-Azure machines.

#### Update Management
Use **Azure Update Manager** for update orchestration across:
- Azure VMs
- Arc-enabled servers

Architectural value:
- Single patching plane
- Maintenance windows
- Compliance visibility

#### When to use Arc-enabled servers
Use Arc-enabled servers when you need:
- Azure governance for non-Azure servers
- Unified security/monitoring/patching
- Incremental hybrid modernization
- Central inventory of distributed servers

Avoid using Arc as a replacement for full migration when the real goal is to **move** the workload to Azure-native services.

---

### Arc-enabled Kubernetes

#### What it is
Azure Arc can attach Kubernetes clusters running:
- On-premises
- Edge
- AWS EKS
- GCP GKE
- Other CNCF-conformant distributions

#### Onboarding Kubernetes clusters
Attach the cluster, then manage it from Azure.

**Azure CLI**
```bash
az extension add --name k8s-extension
az extension add --name connectedk8s
az connectedk8s connect \
  --name arc-aks-edge-01 \
  --resource-group rg-hybrid-k8s \
  --location eastus
```

#### GitOps with Flux
GitOps is a major Arc Kubernetes scenario.

Use GitOps for:
- Cluster configuration drift correction
- Namespace/app deployment
- Standardized platform configuration
- Multi-cluster consistency

**Azure CLI**
```bash
az k8s-configuration flux create \
  --cluster-name arc-aks-edge-01 \
  --resource-group rg-hybrid-k8s \
  --cluster-type connectedClusters \
  --name prod-platform-config \
  --namespace flux-system \
  --url https://github.com/contoso/platform-gitops \
  --branch main \
  --scope cluster
```

#### Azure Policy for Kubernetes
Use Azure Policy to audit/enforce:
- Pod security settings
- Allowed images/registries
- Required labels
- Resource limits
- Kubernetes standards

**Exam tip:** If governance across many Kubernetes clusters is needed, combine **Arc + Policy + GitOps**.

#### Azure Monitor Container Insights
Use Container Insights or Azure Monitor pipelines to:
- Collect node/pod/container metrics
- Centralize log analysis
- Build alerts for cluster health

#### When to use Arc-enabled Kubernetes
Choose Arc-enabled Kubernetes when:
- You already have Kubernetes outside Azure
- You need central governance and observability
- You want GitOps-based operations across multiple clusters/clouds
- Replatforming to AKS is not yet possible or not desired

Not the best answer if the question asks for a **fully managed Kubernetes service in Azure** and there is no hybrid requirement—then **AKS** is usually better.

---

### Arc-enabled Data Services

#### What it is
Azure Arc enables selected Azure data services to run on customer-managed infrastructure, usually Kubernetes.

Main examples:
- **Azure SQL Managed Instance enabled by Azure Arc**
- **Azure Database for PostgreSQL enabled by Azure Arc** (exam language may still reference Hyperscale/Citus in some materials)

#### Running Azure data services anywhere
Benefits:
- Azure-like data platform outside Azure
- Local processing for latency/compliance
- Consistent management for distributed locations
- Useful in disconnected or partially connected edge/hybrid sites

#### Azure SQL Managed Instance (Arc)
Use when you need near-Azure SQL MI experience on customer-managed infra for local/regulatory needs.

#### PostgreSQL Hyperscale / PostgreSQL enabled by Arc
Use for scale-out PostgreSQL scenarios in environments where full Azure residency is not possible.

#### When to use Arc data services
Best fit:
- Data must stay local
- Operations team wants Azure-style management model
- Edge/distributed sites require local data processing
- Temporary disconnected operations are possible

**Exam caution:** Arc data services are not identical to fully managed Azure PaaS. Operational responsibility is higher, and feature parity can differ.

---

### Arc-enabled App Service / Logic Apps / Functions

Azure Arc can bring selected app platform capabilities to Arc-enabled Kubernetes.

#### Running App Service on Arc-enabled Kubernetes
Potential scenarios:
- Standardized app hosting model across edge/on-prem/Azure-connected sites
- Platform abstraction on customer-managed Kubernetes
- Local processing with cloud governance

#### Use cases and limitations
Use when:
- Apps must run close to data/users
- You need consistent app operations in hybrid estates
- Central governance matters more than fully managed cloud elasticity

Limitations to remember:
- Not identical to multi-tenant Azure App Service in Azure
- Requires Kubernetes platform operations maturity
- Feature support can vary by service and release
- Suitability depends on connectivity, platform support, and operational model

**Exam mindset:** If the requirement is “run Azure-managed app platform features outside Azure,” Arc app services may fit. If the requirement is “lowest operational overhead,” native Azure PaaS usually wins.

---

### Azure Arc Resource Bridge

#### What it is
Azure Arc Resource Bridge connects environments such as:
- VMware vSphere
- System Center Virtual Machine Manager (SCVMM)

This enables projected resources and lifecycle operations from Azure.

#### Connecting VMware and SCVMM
Use Resource Bridge when customers want:
- Azure management for existing virtualization estates
- Inventory and lifecycle visibility in Azure
- Gradual modernization without immediate hypervisor replacement

#### Arc-enabled VMs
Arc-enabled VMs for VMware/SCVMM provide:
- ARM representation of VMs
- Azure governance constructs
- Easier transition path into broader Azure hybrid management

**Best exam trigger:** “Customer has major VMware investment and wants Azure governance without replatforming everything.” → **Azure Arc Resource Bridge + Arc-enabled VMware/SCVMM resources**.

---

## 3. Azure Stack Portfolio

### Azure Stack HCI

#### What it is
Azure Stack HCI is a **hyperconverged infrastructure** solution for running virtualized workloads on customer-managed hardware with Azure integration.

Key characteristics:
- Runs on validated hardware
- Hyper-V based virtualization
- Tight Azure integration
- Designed for hybrid operations

#### Hybrid by design
Azure Stack HCI integrates with:
- Azure Arc
- Azure Monitor
- Defender for Cloud
- Azure Backup/management capabilities (depending on scenario)

#### Azure services on Stack HCI
Common scenarios:
- **AKS on Azure Stack HCI**
- **Azure Virtual Desktop** session hosts
- **Arc-enabled VMs**
- Local virtualization with centralized Azure operations

#### Stretched clusters
Useful for:
- Site-level resilience
- Metro-distance fault tolerance
- Local HA without depending entirely on Azure region failover

#### When Azure Stack HCI vs on-prem virtualization
Choose **Azure Stack HCI** when:
- Customer wants modernized on-prem virtualization with Azure integration
- Existing workloads should remain local
- Branch/factory/retail/datacenter workloads need local compute
- Need to run VMs/containers locally with hybrid management

Choose traditional on-prem virtualization only when Azure integration is not needed or hardware/software standards dictate otherwise.

---

### Azure Stack Hub

#### What it is
Azure Stack Hub delivers **Azure-consistent services on-premises**, especially for:
- Disconnected
- Intermittently connected
- Air-gapped
- Sovereign/government scenarios

#### Disconnected/air-gapped scenarios
This is the classic exam fit.

Use cases:
- Military/defense
- Remote industrial sites with low/no connectivity
- Strictly regulated government workloads
- Classified environments

#### Azure-consistent APIs on-premises
Value:
- Similar Azure development/deployment experience
- Consistent application model in disconnected environments
- On-prem execution where Azure public cloud cannot be used directly

#### Government and regulated industries
Stack Hub is strong when compliance and isolation outweigh cloud elasticity.

#### When Stack Hub vs Stack HCI
Choose **Azure Stack Hub** when:
- Need Azure-consistent application services on-prem
- Need disconnected or air-gapped cloud-like platform
- App model consistency is more important than generic virtualization

Choose **Azure Stack HCI** when:
- Need modern hyperconverged virtualization platform with hybrid integration
- Focus is VM/container infrastructure, not Azure-consistent cloud platform services

> **Memory aid:** **Hub = cloud platform on-prem. HCI = virtualization platform with Azure integration.**

---

### Azure Stack Edge

#### What it is
Azure Stack Edge is an edge computing appliance/service for:
- Local compute at the edge
- AI/ML inferencing
- Data preprocessing
- Transfer of data back to Azure

#### Edge computing devices
Great for:
- Factories
- Retail branches
- Ships/oil platforms
- Remote sites
- Near-real-time analytics

#### AI at the edge
Use for:
- Video analytics
- Image recognition
- Sensor aggregation
- Local inferencing when cloud round-trip is too slow

#### Data transfer to Azure
Azure Stack Edge can preprocess/filter/compress data locally and move only needed data to Azure.

#### When Stack Edge vs IoT Edge
Choose **Azure Stack Edge** when:
- You need a managed edge appliance with compute/storage acceleration
- Large data movement and edge analytics are required
- AI inferencing is central

Choose **IoT Edge** when:
- Focus is IoT module/runtime deployment to devices
- You need software-centric edge orchestration, not a dedicated appliance

---

## 4. Hybrid Identity

> **Reference:** Core identity details belong in your Identity section. For hybrid infrastructure questions, focus on how identity affects operations and access.

### Hybrid identity patterns
Common patterns:
- On-prem AD + Microsoft Entra ID sync
- Federated identity for legacy requirements
- Password hash sync for simplicity/resilience
- Pass-through authentication for on-prem validation needs
- Cloud authentication with on-prem directory source

### Directory synchronization
For exam framing:
- **Microsoft Entra Connect Sync** (formerly Azure AD Connect) is used for directory synchronization.
- Hybrid identity supports SSO, centralized identity lifecycle, and cloud access control.
- Password hash sync is often the simplest and most resilient option unless policy requires otherwise.

**Architect considerations:**
- Authentication dependency on on-prem infrastructure
- Resilience during WAN outages
- Admin model and RBAC separation
- Service account modernization and managed identities where possible

---

## 5. Hybrid Networking

Hybrid networking questions usually test **bandwidth, latency, resiliency, routing, and DNS**.

### ExpressRoute for hybrid connectivity
Use ExpressRoute when you need:
- Private connectivity to Azure
- Predictable latency
- Higher throughput
- Better enterprise networking posture
- Regulatory preference for private connectivity

Best for:
- Mission-critical enterprise workloads
- Large data transfer volumes
- Consistent performance needs

**Azure CLI**
```bash
az network express-route list -o table
```

**PowerShell**
```powershell
Get-AzExpressRouteCircuit
```

### Site-to-Site VPN
Use S2S VPN when you need:
- Faster deployment
- Lower cost
- Internet-based encrypted connectivity
- Backup path to ExpressRoute
- Smaller branch or noncritical connectivity

**Azure CLI**
```bash
az network vpn-connection list -g rg-network-hybrid -o table
```

**PowerShell**
```powershell
Get-AzVirtualNetworkGatewayConnection -ResourceGroupName rg-network-hybrid
```

### Azure Virtual WAN for branch connectivity
Use Virtual WAN when there are:
- Many branches
- Need for managed transit architecture
- Centralized connectivity and routing
- SD-WAN integration
- Global branch connectivity patterns

**Azure CLI**
```bash
az network vwan create \
  --name vwan-global \
  --resource-group rg-network-hybrid \
  --location eastus \
  --type Standard
```

### DNS integration (on-prem ↔ Azure)
Hybrid DNS design must answer:
- Where are authoritative zones hosted?
- How do Azure workloads resolve on-prem names?
- How do on-prem workloads resolve Azure private endpoints/private zones?
- What happens during WAN impairment?

### Hybrid DNS resolution patterns
Common patterns:
- Azure DNS Private Resolver for inbound/outbound forwarding
- Conditional forwarders between on-prem DNS and Azure DNS services
- Split-horizon DNS for internal vs public resolution
- Private DNS zones for Azure PaaS private endpoints

**Azure CLI**
```bash
az network private-dns zone list -g rg-network-dns -o table
az network dns-resolver list -g rg-network-dns -o table
```

**Exam design guidance:**
- If Azure VMs must resolve on-prem names, use forwarders/resolvers toward on-prem DNS.
- If on-prem users must resolve private endpoints in Azure, plan conditional forwarding to Azure DNS Private Resolver.
- DNS failures often break “working connectivity,” even when VPN/ER is healthy.

---

## 6. Hybrid Management

### Azure Automation (hybrid runbook workers)
Use Hybrid Runbook Workers when automation must run:
- Inside on-prem environment
- Close to local resources
- Against systems inaccessible directly from Azure

Use cases:
- Patch orchestration
- Local service restarts
- AD/DNS scripts
- Local compliance remediation

**PowerShell**
```powershell
Get-AzAutomationHybridRunbookWorkerGroup -AutomationAccountName auto-hybrid-01 -ResourceGroupName rg-automation
```

### Azure Monitor (Arc integration)
Use Azure Monitor to centralize:
- VM/server logs
- Kubernetes telemetry
- Alerts and dashboards
- Dependency views
- Cross-environment operational visibility

### Azure Update Manager
Strong default for patch governance across Azure and Arc-connected machines.

Benefits:
- Unified update reporting
- Scheduling and maintenance windows
- Reduced tool sprawl

### Azure Policy (Arc extension)
Use Policy + Arc to enforce enterprise baselines for hybrid assets.

Common examples:
- Required AMA deployment
- Allowed locations/tags
- Security settings
- Kubernetes compliance controls

### Microsoft Defender for Cloud (hybrid)
Use Defender for Cloud for:
- Unified secure score
- Recommendations across Azure and Arc resources
- Threat protection for servers and containers
- Compliance dashboards

### Azure Migrate (assessment and migration)
Azure Migrate helps assess and plan movement of on-prem workloads to Azure.

Use when you need:
- Dependency mapping
- Right-sizing
- Cost estimation
- Migration waves
- App/service readiness assessment

**Azure CLI**
```bash
az migrate project list -o table
```

**Architect principle:** Arc manages what stays distributed; Azure Migrate assesses what should move.

---

## 7. Multi-Cloud Scenarios

### Azure Arc for multi-cloud management
Arc extends Azure management to:
- AWS EC2
- GCP VMs
- EKS/GKE clusters
- SQL Server outside Azure

This supports a control-plane strategy where Azure becomes the governance and operations hub.

### Connecting AWS/GCP resources to Azure Arc
Typical goals:
- Central inventory in Azure
- Standardized tagging/policy/compliance
- Common monitoring/security posture
- Gradual platform rationalization over time

### Consistent policy and monitoring
Best practice pattern:
- Arc onboard resources
- Apply RBAC and Policy
- Send telemetry to Azure Monitor / Log Analytics
- Use Defender for Cloud for posture/threat visibility
- Standardize deployment/configuration through GitOps or automation

### When multi-cloud makes sense
Valid reasons:
- Regulatory or contractual requirements
- M&A with inherited platforms
- Service-specific capabilities
- Risk diversification for certain workloads
- Geographic/provider constraints

Weak reasons:
- “Use every cloud for everything” without governance
- Unclear operating model
- Duplicate platform teams with no standardization

**Exam bias:** Multi-cloud is usually justified only when there is a clear business driver. If no driver exists, a simpler Azure-centric design is often preferred.

---

## 8. Hybrid Design Patterns

### Cloud bursting
Use when steady-state runs on-prem, but Azure handles temporary scale spikes.

Best for:
- Seasonal retail
- Batch processing peaks
- HPC/analytics overflow

Watch for:
- Data transfer latency
- Licensing portability
- Identity/network dependencies

### Tiered hybrid
Examples:
- Dev/test in Azure, prod on-prem
- Web/app tier in Azure, database on-prem
- Legacy core on-prem, digital front end in Azure

Best when modernization must be incremental.

### Disaster recovery (on-prem primary, Azure DR)
Common pattern:
- Production on-prem
- Azure Site Recovery / replicated workloads to Azure
- Azure as failover site

Best when:
- Customer wants DR without building second datacenter
- RTO/RPO can be met with Azure-based standby model

### Data gravity patterns
When data is huge, sensitive, or latency-critical, move apps closer to data instead of moving data.

Implication:
- Use edge/on-prem processing
- Use Arc or Stack offerings
- Replicate summarized results to Azure analytics platforms

### Edge-to-cloud patterns
Local processing at edge + central analytics in Azure.

Typical flow:
1. Collect/process locally
2. Filter/aggregate/infer at edge
3. Send events/results to Azure
4. Centralize monitoring and governance in Azure

---

## 9. AZ-305 Decision Scenarios

### 1) Government agency with air-gapped requirements
**Best fit:** Azure Stack Hub  
**Why:** Delivers Azure-consistent services in disconnected/air-gapped environments.  
**Do not choose:** Arc alone, because Arc assumes Azure-connected management scenarios.

### 2) Manufacturing company with edge processing and local AI inferencing
**Best fit:** Azure Stack Edge  
**Why:** Edge appliance, local processing, AI inferencing, and data transfer back to Azure.  
**Secondary services:** Arc, IoT, Monitor.

### 3) Enterprise with large VMware investment wanting Azure governance
**Best fit:** Azure Arc Resource Bridge + Arc-enabled VMware/SCVMM  
**Why:** Preserve VMware estate while projecting resources into Azure for management.

### 4) Organization wants one governance plane across Azure, AWS, GCP, and on-prem servers
**Best fit:** Azure Arc + Azure Policy + Defender for Cloud + Azure Monitor  
**Why:** Consistent governance and security across environments.

### 5) Hybrid Kubernetes strategy across datacenter, Azure, and other clouds
**Best fit:** Arc-enabled Kubernetes + GitOps with Flux + Azure Policy  
**Why:** Multi-cluster governance, configuration consistency, centralized visibility.

### 6) 500 branch offices need simplified connectivity into Azure
**Best fit:** Azure Virtual WAN  
**Why:** Managed branch/transit architecture, scalable branch connectivity, SD-WAN alignment.

### 7) Legacy application cannot leave on-prem database due to residency rules, but web tier needs rapid scaling
**Best fit:** Tiered hybrid architecture  
**Why:** Web/app in Azure, database remains local; plan for latency and secure connectivity.

### 8) Company needs private, predictable, high-throughput connectivity to Azure for business-critical workloads
**Best fit:** ExpressRoute  
**Why:** Private connectivity, predictable performance, enterprise-grade hybrid link.

### 9) Customer wants cheapest and fastest hybrid connection for a small site
**Best fit:** Site-to-Site VPN  
**Why:** Lower cost and faster deployment than ExpressRoute.

### 10) Customer wants Azure services on local infrastructure with modern virtualization and AKS support
**Best fit:** Azure Stack HCI  
**Why:** HCI platform with Azure integration, local VMs/containers, Arc management.

### 11) Customer needs SQL platform capabilities locally due to compliance but wants Azure-style management
**Best fit:** Azure Arc-enabled data services  
**Why:** Run selected Azure data services outside Azure.

### 12) Customer asks for DR without a second datacenter
**Best fit:** Azure as recovery site  
**Why:** Hybrid DR pattern using Azure replication/failover services.

---

## 10. Quick Reference Trigger Table

| Requirement / Trigger | Best Answer | Why it Fits |
|---|---|---|
| Air-gapped government cloud | Azure Stack Hub | Azure-consistent services in disconnected environment |
| Modernize on-prem virtualization with Azure integration | Azure Stack HCI | Hyperconverged platform with hybrid management |
| Edge AI inferencing | Azure Stack Edge | Local compute + AI at edge |
| Manage on-prem servers in Azure | Arc-enabled servers | Governance, monitoring, extensions |
| Manage AWS/GCP VMs from Azure | Arc-enabled servers | Multi-cloud server governance |
| Manage non-Azure Kubernetes centrally | Arc-enabled Kubernetes | Attach clusters for governance and GitOps |
| GitOps across many hybrid clusters | Arc + Flux | Declarative config and drift control |
| Azure policy on non-Azure K8s | Arc-enabled Kubernetes | Policy integration for attached clusters |
| VMware estate with Azure control plane | Arc Resource Bridge | Connect VMware/SCVMM into Azure |
| Need private high-throughput hybrid link | ExpressRoute | Private predictable enterprise connectivity |
| Need low-cost quick hybrid link | Site-to-Site VPN | Fast, encrypted internet-based option |
| Hundreds of branches need transit connectivity | Azure Virtual WAN | Scalable branch and routing architecture |
| Resolve on-prem names from Azure | DNS forwarders / Private Resolver | Hybrid name resolution |
| Resolve Azure private endpoint names from on-prem | Azure DNS Private Resolver + conditional forwarding | Required for private name resolution |
| Unified patching across Azure and on-prem | Azure Update Manager + Arc | Common update plane |
| Unified security posture for Azure and on-prem | Defender for Cloud + Arc | Central recommendations and protection |
| Unified monitoring across clouds/on-prem | Azure Monitor + Arc | Central telemetry and alerts |
| Assess on-prem workloads for migration | Azure Migrate | Readiness, sizing, dependency mapping |
| Keep data local but use Azure-style data platform | Arc-enabled data services | Local processing with Azure control model |
| App platform features on hybrid K8s | Arc-enabled App Service/Functions/Logic Apps | Azure app model outside Azure |
| Keep prod on-prem, dev/test in Azure | Tiered hybrid | Gradual modernization |
| Seasonal scale overflow to Azure | Cloud bursting | Extend capacity only during demand spikes |
| On-prem primary with Azure failover | Hybrid DR pattern | Azure as secondary site |
| Existing hardware dependency blocks migration | Arc / Stack solution | Keep local while modernizing control plane |
| Highly regulated data residency | Hybrid or Arc data services | Keep data local and govern centrally |
| Need Azure APIs on-prem, not just VM hosting | Azure Stack Hub | Cloud-consistent app platform |
| Need local VMs/containers, not disconnected cloud platform | Azure Stack HCI | Better fit for infrastructure workloads |
| Multi-cloud governance hub | Azure Arc | Azure as management plane |
| Local factory processing plus cloud analytics | Edge-to-cloud pattern | Process local, aggregate to Azure |
| Branch office SD-WAN integration | Virtual WAN | Managed connectivity hub |

---

## 11. Common Exam Traps

### Azure Stack Hub vs HCI confusion
- **Stack Hub:** Azure-consistent cloud platform on-prem, often disconnected/regulated.
- **Stack HCI:** Hyperconverged virtualization platform with Azure integration.
- If the question emphasizes **air-gapped cloud platform**, choose **Hub**.
- If it emphasizes **modern on-prem virtualization/AKS/VMs**, choose **HCI**.

### Arc licensing and requirements
- Arc is a management and governance capability, not magic migration.
- Some Arc scenarios/services have specific prerequisites, supported environments, or additional costs.
- Arc data services and app services can require Kubernetes/platform readiness.

### When Arc vs native Azure
- Choose **Arc** when workloads must remain outside Azure or across clouds.
- Choose **native Azure** when the goal is reduced operational overhead, elasticity, and managed PaaS.
- Exam trick: if nothing requires on-prem/multi-cloud, native Azure is often simpler and better.

### Hybrid connectivity bandwidth planning
- ExpressRoute is not just “better VPN”; it is chosen for throughput, predictability, and enterprise routing needs.
- VPN may be enough for small/medium sites or as backup.
- Always think about redundancy, routing design, and latency to dependent services.

### Arc data services limitations
- Do not assume full parity with Azure-native managed database services.
- Operational responsibility is higher.
- Validate support, scale model, and platform dependencies.

### DNS is often the hidden failure point
- Workloads may appear network-connected but still fail due to name resolution.
- Private endpoint and hybrid DNS design is a common architecture trap.

### Multi-cloud is not automatically best practice
- If the scenario lacks a business requirement, avoid unnecessary multi-cloud complexity.
- Governance, identity, cost, and operations become harder across providers.

---

## CLI & PowerShell Quick Commands

### Arc inventory
```bash
az connectedmachine list -o table
az connectedk8s list -o table
az resource list --resource-type Microsoft.HybridCompute/machines -o table
```

```powershell
Get-AzConnectedMachine
Get-AzResource -ResourceType Microsoft.Kubernetes/connectedClusters
```

### Policy and monitoring
```bash
az policy assignment list -o table
az monitor log-analytics workspace list -o table
az monitor data-collection rule list -o table
```

```powershell
Get-AzPolicyAssignment
Get-AzOperationalInsightsWorkspace
```

### Hybrid networking
```bash
az network vpn-gateway list -o table
az network express-route list -o table
az network vwan list -o table
az network private-dns zone list -o table
```

```powershell
Get-AzVirtualNetworkGateway
Get-AzExpressRouteCircuit
Get-AzDnsForwardingRuleset
```

### Migration and modernization
```bash
az migrate project list -o table
az resource list --tag environment=hybrid -o table
```

```powershell
Get-AzResource | Where-Object { $_.Tags['environment'] -eq 'hybrid' }
```

---

## Senior Architect Exam Notes

1. **Start with the business constraint.** If the question mentions sovereignty, air-gap, edge latency, or inherited platforms, hybrid services become likely.
2. **Separate control plane from execution plane.** Arc manages outside Azure; it does not make everything a native Azure service.
3. **Prefer the simplest architecture that satisfies requirements.** Hybrid is justified by constraints, not by fashion.
4. **Watch for hidden dependencies.** Identity, DNS, bandwidth, and operations often determine success more than compute.
5. **Use Azure Stack products carefully.** Hub, HCI, and Edge solve different problems.
6. **For multi-cloud, think governance first.** Arc + Policy + Monitor + Defender is the standard exam pattern.

---

## Rapid Memorization Summary

- **Arc = manage anywhere from Azure**
- **Arc Servers = govern Windows/Linux outside Azure**
- **Arc Kubernetes = govern clusters + GitOps**
- **Arc Data Services = Azure data capabilities on customer-managed infra**
- **Arc Resource Bridge = VMware/SCVMM connection into Azure**
- **Stack Hub = disconnected Azure-consistent cloud on-prem**
- **Stack HCI = modern hybrid virtualization platform**
- **Stack Edge = edge compute + AI + data transfer**
- **ExpressRoute = private enterprise connectivity**
- **VPN = low-cost fast setup connectivity**
- **Virtual WAN = branch/transit at scale**
- **Azure Migrate = assess and move**
- **Defender/Monitor/Policy/Update Manager = hybrid operations toolkit**

---

## Final Exam Checklist

Before choosing a hybrid answer, ask:
- Must the workload remain outside Azure?
- Is the site disconnected or air-gapped?
- Is low latency at edge/on-prem mandatory?
- Is there major VMware or legacy infrastructure to preserve?
- Is centralized governance across clouds required?
- Is private connectivity required, or is VPN enough?
- Is DNS/private resolution part of the requirement?
- Is the goal management, migration, app modernization, or all three?

If you can answer those eight questions, most AZ-305 hybrid questions become much easier.
