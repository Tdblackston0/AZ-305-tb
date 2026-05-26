# Azure Compute - AZ-305 Comprehensive Cheat Sheet

> 📝 **Hands-On Labs:** [Compute Labs](./Labs/Azure-Compute-Labs.md)

> 🎯 **Exam Focus:** AZ-305 tests your ability to **select** the right Azure compute platform based on business requirements — not just know what they are.

## Table of Contents

- [1. Azure Compute Family Overview](#1-azure-compute-family-overview)
- [2. When to Choose Which — Decision Tree](#2-when-to-choose-which--decision-tree)
- [3. Azure Virtual Machines](#3-azure-virtual-machines)
- [4. VM Series Deep Dive](#4-vm-series-deep-dive)
- [5. Virtual Machine Scale Sets (VMSS)](#5-virtual-machine-scale-sets-vmss)
- [6. Azure Kubernetes Service (AKS)](#6-azure-kubernetes-service-aks)
- [7. Azure App Service](#7-azure-app-service)
- [8. Azure Container Apps](#8-azure-container-apps)
- [9. Azure Container Instances (ACI)](#9-azure-container-instances-aci)
- [10. Azure Functions](#10-azure-functions)
- [11. Azure Batch](#11-azure-batch)
- [12. Availability & Resilience](#12-availability--resilience)
- [13. Cost Optimization](#13-cost-optimization)
- [14. AZ-305 Decision Scenarios](#14-az-305-decision-scenarios)
- [15. 🎯 Final AZ-305 Exam Tips](#15--final-az-305-exam-tips)
- [16. 📐 Architecture Decision Flowchart](#16--architecture-decision-flowchart)
- [17. Exam-Style Review Questions](#17-exam-style-review-questions)

---

<a id="1-azure-compute-family-overview"></a>
## 1. Azure Compute Family Overview

Azure compute questions on AZ-305 are rarely about memorizing definitions. The exam usually gives a workload with constraints around **control, scale, cost, latency, resiliency, and operations**, then asks for the **best-fit compute platform**.

### Compute Services Comparison Table

| Service | Model | Best fit | Ops overhead | Scaling model | Common exam trigger |
|---|---|---|---|---|---|
| **Virtual Machines** | IaaS | Full OS control, legacy apps, custom software, special drivers | Highest | Manual or scripted | “Lift-and-shift”, “full admin control”, “custom agent” |
| **VMSS** | IaaS fleet | Large sets of similar VMs with autoscale | High | Manual, scheduled, metric-based, predictive | “Identical web tier”, “autoscaling VM fleet” |
| **AKS** | Managed Kubernetes | Kubernetes orchestration, platform engineering, microservices | Medium-High | Pod autoscale + cluster autoscale + KEDA | “Kubernetes required”, “service mesh”, “operators” |
| **App Service** | PaaS | Web apps and APIs with minimal management | Low | Scale up/out, autoscale rules | “Deployment slots”, “Easy Auth”, “minimal ops” |
| **Container Apps** | Serverless containers | Containerized apps, APIs, event-driven microservices without Kubernetes management | Low-Medium | HTTP/event-driven KEDA scaling, scale to zero | “Scale to zero”, “Dapr”, “KEDA” |
| **ACI** | Serverless container runtime | One-off containers, short jobs, fast startup | Low | Instance-level only | “Quick container job”, “short-lived task” |
| **Functions** | FaaS | Event-driven code, timers, queues, orchestration | Lowest | Automatic event-driven scale | “Serverless”, “queue trigger”, “Durable Functions” |
| **Batch** | Managed batch platform | Large-scale parallel jobs, rendering, simulation, HPC farms | Medium | Pool/task driven autoscale | “Thousands of tasks”, “rendering”, “simulation” |

> 💡 **AZ-305 Tip:** If the scenario can be solved with a managed platform, the exam often prefers **PaaS/serverless** over VMs because of lower operational overhead.

### Azure Virtual Machines
Azure Virtual Machines remain the default answer when you need **full control of the operating system, custom software stacks, legacy runtime support, or specialized hardware**. They are the most flexible compute option, but they also create the largest operational burden because you manage the guest OS, patching strategy, configuration drift, backup design, monitoring agents, and much of the resilience model yourself.

**Real-World Examples:**
- A **15-year-old Windows application** must move to Azure with almost no code changes and still requires RDP access.
- A **third-party security appliance** needs kernel-level drivers and custom OS hardening.
- A **GPU-backed AI training server** must use a specific CUDA stack and custom image.

### Virtual Machine Scale Sets (VMSS)
VMSS is the right answer when you need a **fleet of compute instances** with a consistent image/configuration, **autoscaling**, and **high availability**. It keeps the VM model but adds fleet management features such as autoscale, rolling upgrades, automatic repairs, and integration with load balancing for stateless app tiers.

**Real-World Examples:**
- A **retail web tier** must scale from 4 to 40 identical VMs during holiday traffic.
- A **line-of-business API farm** needs rolling upgrades with health probes and automatic instance repair.
- A **VDI-style worker fleet** must scale on a schedule during business hours and shrink overnight.

### Azure Kubernetes Service (AKS)
AKS is the managed Kubernetes answer when the solution needs **container orchestration with Kubernetes APIs and advanced platform control**. It is the strongest fit when the team explicitly needs Kubernetes concepts such as node pools, custom ingress, service mesh, operators, Helm charts, or policy-driven pod governance.

**Real-World Examples:**
- A **fintech platform** needs microservices with service mesh, network policies, and kubectl-based operations.
- A **multi-team platform engineering group** standardizes deployments around Helm, GitOps, and Kubernetes CRDs.
- A **regulated workload** needs Windows and Linux node pools, private cluster networking, and workload identity.

### Azure App Service
App Service is the exam-favorite PaaS answer for **web apps and APIs** when you want **minimal infrastructure management**. It abstracts away servers and patching while giving built-in web hosting capabilities such as deployment slots, custom domains, TLS, autoscale, authentication/authorization, and strong integration with CI/CD workflows.

**Real-World Examples:**
- A **public .NET API** needs deployment slots, Entra ID authentication, autoscale, and a fast delivery model.
- A **corporate intranet site** needs custom domains, certificates, and minimal platform administration.
- A **customer portal** needs outbound VNet access to a private database while keeping the app itself fully managed.

### Azure Container Apps
Azure Container Apps is the best answer when you want **serverless containers** with **managed scaling** and **reduced operational complexity**. It provides container packaging, revisions, KEDA-based scaling, Dapr integration, and scale-to-zero without asking the team to run Kubernetes.

**Real-World Examples:**
- A **containerized order API** must scale to zero overnight and back up quickly during business spikes.
- A **microservices team** wants Dapr building blocks and event-driven scaling but does not want AKS overhead.
- A **queue-driven worker service** needs KEDA-based scale-out on Service Bus messages.

### Azure Container Instances (ACI)
ACI is the simplest Azure container runtime for **single containers or small container groups** that must start quickly without orchestration. It is ideal for bursty, disposable compute where per-second billing and fast provisioning matter more than platform-level deployment patterns.

**Real-World Examples:**
- A **CI pipeline** launches a short-lived test runner container for 20 minutes and then deletes it.
- A **data team** runs an ad hoc script in a container to preprocess files once per day.
- A **development team** needs a temporary environment to validate one container image quickly.

### Azure Functions
Azure Functions is the FaaS answer for **event-driven code execution** with minimal infrastructure management. It shines when code runs in response to events, timers, queues, or HTTP requests and when the organization wants to pay only when code executes or use Premium plans for low-latency serverless behavior.

**Real-World Examples:**
- An **IoT solution** triggers message processing whenever new telemetry lands in Event Hubs.
- A **back-office workflow** runs on a timer every night to reconcile orders.
- A **serverless integration app** uses Durable Functions to orchestrate human approvals and callbacks.

### Azure Batch
Azure Batch is purpose-built for **large-scale parallel processing** and **HPC-style job scheduling**. It coordinates pools, jobs, and tasks so architects can run thousands of parallel units of work, often using low-priority or Spot-style capacity for major cost savings.

**Real-World Examples:**
- A **media company** renders millions of frames overnight before the morning deadline.
- A **research lab** runs Monte Carlo simulations across hundreds of compute nodes.
- A **financial services firm** executes large parallel risk calculations every evening.

### When to use VMs vs containers vs serverless

| If the workload needs... | Prefer | Why |
|---|---|---|
| Custom OS settings, legacy middleware, domain join, RDP/SSH admin access | **VMs** | You control the guest OS and runtime |
| Container packaging, sidecars, portability, microservices, image-based deployment | **Containers** | Consistent packaging and easier app-level isolation |
| Event-driven execution, scale-to-zero, queue/timer/HTTP triggers, minimal ops | **Serverless** | Pay only when code runs |

### Decision heuristics

- Choose **VMs** if the requirement says: **legacy**, **lift-and-shift**, **custom agent**, **special kernel/driver**, **full admin control**.
- Choose **AKS** if the requirement says: **Kubernetes**, **service mesh**, **network policies**, **custom ingress**, **operators**, **Helm**, **kubectl**.
- Choose **App Service** if the requirement says: **standard web app/API**, **minimal operations**, **deployment slots**, **Easy Auth**.
- Choose **Container Apps** if the requirement says: **containerized**, **microservices**, **KEDA scaling**, **Dapr**, **no Kubernetes overhead**, **scale to zero**.
- Choose **Functions** if the requirement says: **events**, **timers**, **serverless**, **sporadic**, **queue processing**, **Durable Functions**.
- Choose **Batch** if the requirement says: **large-scale parallel jobs**, **rendering**, **simulation**, **HPC**, **scheduled compute farm**.

### Quick commands

```bash
az vm list -o table
az aks list -o table
az webapp list -o table
az functionapp list -o table
```

```powershell
Get-AzVM
Get-AzAksCluster
Get-AzWebApp
Get-AzFunctionApp
```

---

<a id="2-when-to-choose-which--decision-tree"></a>
## 2. When to Choose Which — Decision Tree

```text
Start
 │
 ├─ Need full OS control, legacy runtime support, domain join, custom drivers, or RDP/SSH?
 │  ├─ YES → Azure Virtual Machines
 │  │        └─ Need many similar VMs with autoscale, rolling upgrades, and health-based repairs?
 │  │           ├─ YES → VMSS
 │  │           └─ NO  → Individual VMs
 │  └─ NO
 │
 ├─ Is the workload containerized?
 │  ├─ NO
 │  │   ├─ Is it event-driven, timer-based, or queue-triggered code?
 │  │   │   ├─ YES → Azure Functions
 │  │   │   └─ NO  → Azure App Service (web app/API) or VMs if platform constraints exist
 │  └─ YES
 │      ├─ Need Kubernetes API, service mesh, operators, custom ingress, or advanced policies?
 │      │   ├─ YES → AKS
 │      │   └─ NO
 │      ├─ Need a long-running app/API with scale-to-zero, revisions, Dapr, or KEDA?
 │      │   ├─ YES → Azure Container Apps
 │      │   └─ NO
 │      ├─ Need a single short-lived container or ad hoc task?
 │      │   ├─ YES → ACI
 │      │   └─ NO  → Reassess AKS vs Container Apps
 │
 └─ Need thousands of parallel tasks, rendering, simulation, or HPC scheduling?
    ├─ YES → Azure Batch
    └─ NO  → Re-evaluate workload constraints
```

### Quick Decision Matrix

| Scenario | Best answer | Why |
|---|---|---|
| Lift-and-shift Windows/Linux application | **Virtual Machines** | Least refactoring and full OS control |
| Autoscaling stateless web/app tier on VMs | **VMSS** | Fleet management, autoscale, rolling upgrades |
| Kubernetes microservices platform | **AKS** | Full Kubernetes ecosystem and control |
| Public web app/API with minimal ops | **App Service** | PaaS simplicity, slots, Easy Auth |
| Containerized app without Kubernetes overhead | **Container Apps** | Revisions, KEDA, scale to zero |
| One-off container job or build runner | **ACI** | Fast startup and per-second billing |
| Event-driven code execution | **Functions** | Triggers, bindings, serverless scaling |
| Massive parallel job farm | **Batch** | Pools, jobs, tasks, HPC-style scheduling |

### Compute Decision Matrix

| Service | Typical scenario | Scaling model | Cost model | Best for |
|---|---|---|---|---|
| **Virtual Machines** | Lift-and-shift, legacy enterprise app, custom OS dependency | Manual or scripted / scale set assisted | Pay-as-you-go, RI, Savings Plan, Spot | Full control and legacy compatibility |
| **VMSS** | Stateless web/app tier with autoscaling | Manual, scheduled, metric-based, predictive | VM-based with RI/Spot opportunities | Large fleets of similar VMs |
| **AKS** | Kubernetes microservices, platform engineering | Pod autoscale + cluster autoscale + KEDA | Pay for nodes/storage/network, optional paid tier | Full K8s orchestration |
| **App Service** | Web apps and APIs | Scale up/out, autoscale rules | App Service Plan based | PaaS web hosting with minimal ops |
| **Container Apps** | Serverless containers, APIs, event-driven microservices | KEDA-based, HTTP/event, scale to zero | Consumption-style container billing | Managed container platform without AKS complexity |
| **ACI** | One-off containers, burst jobs, build/test runners | Instance-level only | Per-second container billing | Fast, simple container execution |
| **Functions** | Event-driven code, timers, queues, lightweight APIs | Automatic event-driven scale | Consumption, Premium, or App Service plan | FaaS and orchestration with Durable Functions |
| **Batch** | HPC, rendering, simulation, scheduled large job farms | Pool/task driven autoscale | Underlying VM/storage/network cost | Parallel processing at scale |

### Quick comparison summary

| Requirement | Best answer |
|---|---|
| Full OS control | **VMs** |
| Autoscaling identical VM fleet | **VMSS** |
| Full Kubernetes ecosystem | **AKS** |
| Simplest enterprise web app hosting | **App Service** |
| Serverless containers | **Container Apps** |
| Fast one-off containers | **ACI** |
| Event-driven code execution | **Functions** |
| Parallel compute farm | **Batch** |

---

<a id="3-azure-virtual-machines"></a>
## 3. Azure Virtual Machines

Azure Virtual Machines remain the default answer when you need **full control of the operating system, custom software stacks, legacy runtime support, or specialized hardware**.

### VM sizing methodology

1. **Profile the workload**: CPU, RAM, storage, IOPS, throughput, network, GPU, licensing.
2. **Classify workload shape**:
   - General purpose → D-series
   - Memory optimized → E/M-series
   - Compute optimized → F/H-series
   - Storage optimized → L-series
   - GPU → N-series
3. **Map non-functional requirements**: zones, encryption, backup, DR, region availability.
4. **Start with right family, then right-size** using baseline + peak metrics.
5. **Optimize cost** with Reserved Instances, Savings Plans, Azure Hybrid Benefit, Spot, or dev/test pricing where appropriate.
6. **Validate region support** because not every series is in every region.

### VM extensions

Common extensions to remember:
- **Custom Script Extension** - bootstrap configuration or run scripts after deployment
- **Azure Monitor Agent** - telemetry/monitoring
- **Desired State Configuration / Guest Configuration** - configuration enforcement
- **Dependency / security / antimalware related extensions** depending on workload needs

### Managed disks

| Disk type | Best for | Notes |
|---|---|---|
| **Standard HDD** | Backup, archive, low-cost dev/test | Lowest performance, cheapest |
| **Standard SSD** | Cost-sensitive production, light workloads | Better consistency than HDD |
| **Premium SSD** | Production OLTP, enterprise apps | Higher IOPS/throughput, common production choice |
| **Ultra Disk** | Extreme IOPS/low latency databases | Highest performance, separate configuration model |

### Disk encryption

| Option | What it does | When to choose |
|---|---|---|
| **Server-Side Encryption (SSE)** | Encrypts managed disks at rest by default | Default answer for most workloads |
| **CMK (customer-managed keys)** | Use your own key from Key Vault / Managed HSM | Compliance, key control, separation of duties |
| **Azure Disk Encryption (ADE)** | Encrypts OS/data disks inside guest using BitLocker/DM-Crypt | Use only when guest-level encryption is specifically required |

> 💡 **Exam mindset:** If the requirement only says “encrypt at rest,” **SSE** is usually enough. If it says “customer controls keys,” choose **CMK**.

### VM backup and disaster recovery

| Requirement | Service | Architect note |
|---|---|---|
| Point-in-time VM backup | **Azure Backup** | Protects VM and disks with policy-based backups |
| Region-to-region replication / failover | **Azure Site Recovery** | Best for DR and failover orchestration |
| Business continuity design | **Backup + ASR** | Backup is not the same as failover |

### CLI and PowerShell examples

```bash
# Create a Linux VM in a zone
az vm create \
  --resource-group rg-compute-prod \
  --name vm-web-01 \
  --image Ubuntu2204 \
  --size Standard_D4s_v5 \
  --zone 1 \
  --admin-username azureuser \
  --generate-ssh-keys

# List available sizes in a region
az vm list-sizes --location eastus --output table

# Resize a VM (typically after deallocation if required)
az vm deallocate --resource-group rg-compute-prod --name vm-web-01
az vm resize --resource-group rg-compute-prod --name vm-web-01 --size Standard_E4s_v5
az vm start --resource-group rg-compute-prod --name vm-web-01

# Add Custom Script Extension
az vm extension set \
  --resource-group rg-compute-prod \
  --vm-name vm-web-01 \
  --name CustomScript \
  --publisher Microsoft.Azure.Extensions \
  --settings '{"fileUris":["https://contoso.blob.core.windows.net/scripts/bootstrap.sh"],"commandToExecute":"bash bootstrap.sh"}'
```

```powershell
# Create a Windows VM
New-AzVM -ResourceGroupName "rg-compute-prod" -Name "vm-app-01" `
  -Location "EastUS" -VirtualNetworkName "vnet-app" -SubnetName "snet-app" `
  -SecurityGroupName "nsg-app" -PublicIpAddressName "pip-vm-app-01" `
  -ImageName "Win2022Datacenter" -Size "Standard_D4s_v5"

# Inspect VM sizes in a region
Get-AzVMSize -Location "EastUS" | Select-Object Name, NumberOfCores, MemoryInMB

# Resize a VM
$vm = Get-AzVM -ResourceGroupName "rg-compute-prod" -Name "vm-app-01"
$vm.HardwareProfile.VmSize = "Standard_E4s_v5"
Update-AzVM -VM $vm -ResourceGroupName "rg-compute-prod"

# Enable backup for a VM
$vault = Get-AzRecoveryServicesVault -Name "rsv-prod"
Set-AzRecoveryServicesVaultContext -Vault $vault
Enable-AzRecoveryServicesBackupProtection -Policy (Get-AzRecoveryServicesBackupProtectionPolicy -Name "DefaultPolicy") `
  -Name "vm-app-01" -ResourceGroupName "rg-compute-prod"
```

### AZ-305 takeaways

- **Availability Set** = rack/host resiliency in one datacenter.
- **Availability Zone** = datacenter-level isolation in one region.
- **PPG** = low latency, not high resiliency.
- **Dedicated Host** = compliance/licensing/isolation.
- **Spot** = deep savings, eviction risk.
- **Reserved Instances + Azure Hybrid Benefit** = strongest steady-state VM cost answer.

---

<a id="4-vm-series-deep-dive"></a>
## 4. VM Series Deep Dive

### Detailed VM Series Comparison

| Series | Best for | Key characteristics | Common exam trigger |
|---|---|---|---|
| **B-series** | Dev/test, low baseline CPU, burstable workloads | Credits-based CPU bursting, low cost | “Low average CPU with occasional spikes” |
| **D-series** | General-purpose app servers | Balanced CPU, memory, and temporary storage | “Standard enterprise app tier” |
| **E-series** | Memory-intensive apps | Higher memory-to-vCPU ratio | “In-memory cache/database” |
| **F-series** | Compute-heavy workloads | Higher CPU-to-memory ratio | “Build server / analytics / CPU bound” |
| **M-series** | Very large memory SAP/HANA workloads | Extreme memory capacity, SAP certified SKUs | “SAP HANA / huge memory footprint” |
| **N-series** | GPU workloads | NVIDIA GPUs for AI/ML, rendering, VDI | “GPU training / visualization” |
| **L-series** | Storage throughput / IOPS heavy workloads | High local NVMe / storage optimized | “NoSQL / big data / high disk throughput” |
| **H-series** | HPC and MPI workloads | High-performance CPU, RDMA/InfiniBand options | “Simulation / scientific compute / low latency MPI” |

### B-series
Use **B-series** when the workload runs at a low average CPU baseline and occasionally needs to burst. It is commonly the lowest-cost option for internal apps, development environments, and small services with idle periods.

**Use cases:** dev/test VMs, jump boxes, internal apps with month-end spikes.

### D-series
Use **D-series** for balanced enterprise workloads that need a reliable mix of compute, memory, and temporary storage. It is often the default starting point for application servers and web tiers.

**Use cases:** web servers, application servers, domain services, mid-tier business apps.

### E-series
Use **E-series** when memory is the bottleneck rather than CPU. These VMs are strong fits for in-memory databases, caching tiers, and analytics workloads that hold large datasets in RAM.

**Use cases:** SAP app servers, in-memory caches, relational databases, analytics engines.

### F-series
Use **F-series** for CPU-bound workloads that need more compute relative to memory. These are strong fits for build agents, rendering workers, and analytics processes where CPU is the primary constraint.

**Use cases:** build servers, batch compute workers, gaming backends, CPU-bound analytics.

### M-series
Use **M-series** when the workload needs extremely large memory footprints and enterprise certification. These are premium, specialized SKUs most commonly associated with SAP HANA and other huge in-memory workloads.

**Use cases:** SAP HANA, massive in-memory databases, large-scale ERP platforms.

### N-series
Use **N-series** when a workload needs GPU acceleration. This family is the exam answer for AI/ML training, inference at scale, rendering, CAD, and virtual desktop scenarios requiring GPU-backed graphics or compute.

**Use cases:** AI/ML training, video rendering, visualization, GPU-enabled VDI.

### L-series
Use **L-series** when local storage throughput and IOPS matter more than raw memory size. They are ideal for storage-optimized workloads, NoSQL engines, and data-intensive systems that benefit from very fast local disks.

**Use cases:** Cassandra-style workloads, Elasticsearch, big data processing, log analytics nodes.

### H-series
Use **H-series** when the scenario mentions HPC, scientific modeling, MPI, or extremely low-latency node-to-node communication. These are the classic exam choice for simulations and technical computing.

**Use cases:** engineering simulation, computational chemistry, weather modeling, MPI-based clusters.

> 💡 **AZ-305 Tip:** The exam often cares more about choosing the **right VM family** than memorizing exact SKUs.

---

<a id="5-virtual-machine-scale-sets-vmss"></a>
## 5. Virtual Machine Scale Sets (VMSS)

VMSS is the right answer when you need a **fleet of compute instances** with consistent image/configuration, **autoscaling**, and **high availability**.

### Uniform vs Flexible orchestration mode

| Mode | Best for | Key characteristics | Exam trigger |
|---|---|---|---|
| **Uniform** | Large identical stateless fleets | Same model/image/size, optimized for massive scale | “Identical web tier VMs” |
| **Flexible** | Mixed sizes, more VM-like management | More flexible placement, can support heterogeneous instance patterns | “Mix VM sizes / granular control / existing VMs style behavior” |

### Scaling policies

| Policy | Use case |
|---|---|
| **Manual** | Predictable or controlled admin scaling |
| **Scheduled** | Known peaks like business hours or month-end |
| **Metric-based autoscale** | CPU, memory via Monitor, queue depth, custom metrics |
| **Predictive autoscale** | Workloads with recurring usage patterns |

### Health probes and automatic repairs

- Use **Azure Load Balancer health probes** or app health signals to determine healthy instances.
- **Automatic instance repairs** replace unhealthy VMs when health criteria fail.
- Best for stateless tiers that should self-heal.

### Rolling upgrades

Rolling upgrades update instances in batches to reduce impact.

Use when the scenario says:
- minimize downtime during updates
- update fleet gradually
- maintain application availability during upgrade

### Overprovisioning

Overprovisioning creates extra instances temporarily to meet target healthy capacity faster.

**Exam note:** temporary extra instances used for overprovisioning are not a long-term cost trap; they help achieve target capacity faster.

### Scale-in policies

Know the scale-in options conceptually:
- default balancing behavior
- oldest/newest VM deletion patterns
- zone-aware and resiliency-aware decisions depending on deployment model

The exam usually tests whether you understand that **scale-in behavior matters** when preserving balanced capacity or protecting newer instances.

### Instance protection

Use instance protection to prevent selected VMs from:
- scale-in deletion
- model upgrades

Best for canary instances, session-heavy nodes, or temporary protected instances.

### When VMSS vs individual VMs vs AKS

| Requirement | Best answer |
|---|---|
| A few specialized servers with manual administration | **Individual VMs** |
| Many similar VMs with autoscaling | **VMSS** |
| Containerized microservices with orchestration | **AKS** |

### CLI and PowerShell examples

```bash
# Create a zone-aware VMSS
az vmss create \
  --resource-group rg-compute-prod \
  --name vmss-web-prod \
  --image Ubuntu2204 \
  --vm-sku Standard_D2s_v5 \
  --instance-count 3 \
  --zones 1 2 3 \
  --upgrade-policy-mode Rolling

# Create autoscale settings
az monitor autoscale create \
  --resource-group rg-compute-prod \
  --resource vmss-web-prod \
  --resource-type Microsoft.Compute/virtualMachineScaleSets \
  --name vmss-web-prod-autoscale \
  --min-count 2 \
  --max-count 10 \
  --count 3

az monitor autoscale rule create \
  --resource-group rg-compute-prod \
  --autoscale-name vmss-web-prod-autoscale \
  --condition "Percentage CPU > 70 avg 5m" \
  --scale out 2

# Protect one instance from scale-in
az vmss update \
  --resource-group rg-compute-prod \
  --name vmss-web-prod \
  --instance-id 0 \
  --protect-from-scale-in true
```

```powershell
# Create a VMSS
New-AzVmss -ResourceGroupName "rg-compute-prod" -VMScaleSetName "vmss-web-prod" `
  -Location "EastUS" -VirtualNetworkName "vnet-app" -SubnetName "snet-web" `
  -ImageName "Ubuntu2204" -InstanceCount 3 -SkuCapacity 3 -SkuName "Standard_D2s_v5"

# Review VMSS instances
Get-AzVmssVM -ResourceGroupName "rg-compute-prod" -VMScaleSetName "vmss-web-prod"

# Update capacity
Update-AzVmss -ResourceGroupName "rg-compute-prod" -VMScaleSetName "vmss-web-prod" -SkuCapacity 5
```

### AZ-305 takeaways

- **Uniform** = identical fleet.
- **Flexible** = more VM-like, more heterogeneous.
- VMSS is a strong answer for **autoscaling stateless app tiers**.
- Use **health probes + automatic repairs + rolling upgrades** for resilient fleet management.

---

<a id="6-azure-kubernetes-service-aks"></a>
## 6. Azure Kubernetes Service (AKS)

AKS is the managed Kubernetes answer when the solution needs **container orchestration with Kubernetes APIs and advanced platform control**.

### AKS architecture

| Component | Role |
|---|---|
| **Managed control plane** | Azure-managed API server and control services |
| **Node pools** | Groups of VMs hosting pods |
| **System node pool** | Runs critical cluster add-ons and system pods |
| **User node pool** | Runs application workloads |

### System vs user node pools

| Pool type | Purpose | Guidance |
|---|---|---|
| **System** | Core add-ons such as DNS, metrics, kube-system services | Keep stable and not overloaded |
| **User** | Business workloads | Separate by workload type, OS, cost, or scaling behavior |

### Node pool scaling

| Option | Use case |
|---|---|
| **Manual scaling** | Predictable environments |
| **Cluster autoscaler** | Scale nodes based on pending pods |
| **KEDA** | Event-driven scaling for workloads using external metrics like queues |

### AKS networking

| Model | Best for | Strengths | Tradeoffs |
|---|---|---|---|
| **kubenet** | Simpler/legacy Linux-only clusters with basic networking needs | Conserves VNet IPs | Less direct pod IP visibility; fewer advanced scenarios |
| **Azure CNI** | Full VNet integration, Windows node pools, direct IP visibility | Best compatibility and enterprise networking fit | Consumes more VNet IP addresses |
| **Azure CNI Overlay** | Large scale clusters needing IP efficiency with Azure networking benefits | Better IP utilization than traditional CNI | Validate feature fit and region support for design |

> 💡 **Exam tip:** If the scenario mentions **Windows containers**, **advanced networking**, **direct VNet visibility**, or **enterprise network policy**, prefer **Azure CNI**.

### AKS load balancing

- **External Load Balancer** for public services
- **Internal Load Balancer** for private services
- **Ingress controllers** for HTTP/HTTPS routing, TLS offload, and app routing
- Common design combinations:
  - AKS + internal ingress + Front Door/Application Gateway
  - AKS + private cluster + internal load balancer for high-security designs

### ACR integration

Use Azure Container Registry with AKS for:
- private image storage
- managed pull permissions
- simplified CI/CD image deployment

### AKS identity

| Identity option | Use case |
|---|---|
| **Managed identity for cluster/nodes** | Default Azure resource access pattern |
| **Workload identity** | Pod/application access to Azure resources without secrets |

### AKS monitoring

Recommended services:
- **Container Insights** for logs, node/pod visibility
- **Managed Prometheus** for metrics
- **Managed Grafana** for visualization
- Azure Monitor alerts for node pressure, pod restarts, cluster health

### Upgrades and maintenance windows

- Plan **control plane** and **node pool** upgrades separately.
- Use **maintenance windows** to control change timing.
- Upgrade non-production first; keep node images and Kubernetes versions supported.

### When to use AKS vs Container Apps vs App Service

| Requirement | Best answer |
|---|---|
| Full Kubernetes API, operators, service mesh, custom policies | **AKS** |
| Containerized apps without Kubernetes overhead | **Container Apps** |
| Traditional web app/API with simplest platform experience | **App Service** |

### CLI and PowerShell examples

```bash
# Create AKS with managed identity and zones
az aks create \
  --resource-group rg-platform-prod \
  --name aks-prod-eastus \
  --node-count 3 \
  --node-vm-size Standard_D4s_v5 \
  --zones 1 2 3 \
  --enable-managed-identity \
  --attach-acr acrplatformprod \
  --network-plugin azure \
  --tier standard

# Add a spot user node pool
az aks nodepool add \
  --resource-group rg-platform-prod \
  --cluster-name aks-prod-eastus \
  --name spotnp \
  --priority Spot \
  --eviction-policy Delete \
  --node-vm-size Standard_D2s_v5 \
  --min-count 0 \
  --max-count 10 \
  --enable-cluster-autoscaler

# Get cluster credentials
az aks get-credentials --resource-group rg-platform-prod --name aks-prod-eastus
```

```powershell
# Create AKS
New-AzAksCluster -ResourceGroupName "rg-platform-prod" -Name "aks-prod-eastus" `
  -Location "EastUS" -NodeCount 3 -NodeVmSize "Standard_D4s_v5" `
  -EnableManagedIdentity

# Get AKS details
Get-AzAksCluster -ResourceGroupName "rg-platform-prod" -Name "aks-prod-eastus"

# Import credentials for kubectl
Import-AzAksCredential -ResourceGroupName "rg-platform-prod" -Name "aks-prod-eastus" -Force
```

### AZ-305 takeaways

- AKS is not the default container answer; it is the answer when **Kubernetes itself is required**.
- Separate **system** and **user** node pools.
- Networking choice is a common exam differentiator: **kubenet vs Azure CNI vs Azure CNI Overlay**.
- For pod-to-Azure access, prefer **workload identity** over secrets.

---

<a id="7-azure-app-service"></a>
## 7. Azure App Service

App Service is the exam-favorite PaaS answer for **web apps and APIs** when you want **minimal infrastructure management**.

### App Service Plan tiers

| Tier | Best for | Key features | Exam guidance |
|---|---|---|---|
| **Free / Shared** | Demos and tiny non-prod apps | Shared resources | Rarely correct for production |
| **Basic** | Small dev/test or low-scale workloads | Dedicated compute, limited scale features | Still usually not best production answer |
| **Standard** | Production web apps | Autoscale, deployment slots | Common entry production tier |
| **Premium** | Higher scale/performance | Better CPU/RAM, networking options, more scale | Strong production choice |
| **Isolated** | Highly secure enterprise workloads | Dedicated environment in ASE | Use for maximum isolation |

### App Service Environment (ASE) v3

Use ASE v3 when the exam requires:
- dedicated isolated App Service environment
- heavy inbound/outbound control inside a VNet
- strict compliance or noisy-neighbor avoidance
- large-scale enterprise web hosting with isolation

### Deployment slots and swap

Use slots for:
- blue/green deployments
- validation before production cutover
- near-zero downtime releases

Common slots: **staging**, **production**.

### Auto-scaling rules

Scale based on:
- CPU or memory-related metrics
- HTTP queue length / requests
- schedule-based business hours

### Custom domains and SSL

- Map custom domains for branded endpoints.
- Use managed certificates or imported certificates.
- Exam often expects TLS everywhere.

### Authentication/authorization (Easy Auth)

Easy Auth is built-in authentication for App Service.

Best when the scenario says:
- integrate with Entra ID quickly
- reduce custom auth code
- protect app or API with managed identity providers

### VNet integration

**Important exam distinction:**
- **VNet Integration** = outbound app access into a VNet
- **Private Endpoint** = inbound private access to the app

### Hybrid Connections

Use Hybrid Connections when you need simple outbound connectivity from App Service to specific on-prem endpoints **without full VNet integration**.

### WebJobs vs Functions

| Requirement | Best answer |
|---|---|
| Background processing tied closely to a web app | **WebJobs** |
| Standalone event-driven serverless processing | **Functions** |

### When App Service vs Container Apps vs AKS

| Requirement | Best answer |
|---|---|
| Standard web app/API, low ops, slots, Easy Auth | **App Service** |
| Containerized apps, scale to zero, KEDA, Dapr | **Container Apps** |
| Kubernetes control, custom platform engineering | **AKS** |

### CLI and PowerShell examples

```bash
# Create plan and web app
az appservice plan create \
  --resource-group rg-app-prod \
  --name plan-web-prod \
  --sku P1v3 \
  --is-linux

az webapp create \
  --resource-group rg-app-prod \
  --plan plan-web-prod \
  --name webapp-contoso-prod \
  --runtime "DOTNET:8"

# Create and swap a deployment slot
az webapp deployment slot create \
  --resource-group rg-app-prod \
  --name webapp-contoso-prod \
  --slot staging

az webapp deployment slot swap \
  --resource-group rg-app-prod \
  --name webapp-contoso-prod \
  --slot staging \
  --target-slot production
```

```powershell
# Create App Service plan
New-AzAppServicePlan -ResourceGroupName "rg-app-prod" -Name "plan-web-prod" `
  -Location "EastUS" -Tier "PremiumV3" -NumberofWorkers 2 -Linux

# Create Web App
New-AzWebApp -ResourceGroupName "rg-app-prod" -Name "webapp-contoso-prod" `
  -Location "EastUS" -AppServicePlan "plan-web-prod"

# Create staging slot
New-AzWebAppSlot -ResourceGroupName "rg-app-prod" -Name "webapp-contoso-prod" -Slot "staging"
```

### AZ-305 takeaways

- App Service is often the best answer for **enterprise web apps and APIs**.
- **Slots** are a big differentiator versus other services.
- Remember the directionality: **VNet Integration outbound**, **Private Endpoint inbound**.
- Use **ASE v3** only when isolation/network control requirements justify it.

---

<a id="8-azure-container-apps"></a>
## 8. Azure Container Apps

Azure Container Apps is the best answer when you want **serverless containers** with **managed scaling** and **reduced operational complexity**.

### Serverless containers concept

Container Apps gives you:
- container packaging
- HTTP and event-driven scaling
- scale to zero
- simplified operations compared to AKS

### Container Apps environment

The environment is the shared boundary for:
- networking
- observability integration
- revision runtime context
- multiple container apps in a shared platform plane

### Revision management

Use revisions for:
- blue/green-style traffic splitting
- canary releases
- rollback to a prior revision

### KEDA-based scaling

| Trigger type | Example |
|---|---|
| **HTTP** | Scale on concurrent requests |
| **Queue / event** | Service Bus, Event Hubs, Kafka, Storage Queue |
| **Custom** | External metrics via KEDA scalers |

### Dapr integration

Dapr is useful for:
- service invocation
- pub/sub
- state abstraction
- secret and binding integration patterns

This is a strong answer when the scenario mentions **microservices building blocks without full Kubernetes management**.

### Ingress configuration

- **External ingress** for public endpoints
- **Internal ingress** for private/internal microservices

### Managed identity

Use managed identity for app access to:
- Key Vault
- Storage
- Service Bus
- SQL and other Azure services

### When Container Apps vs AKS vs App Service vs ACI

| Requirement | Best answer |
|---|---|
| Serverless containers, KEDA, Dapr, no Kubernetes admin | **Container Apps** |
| Need Kubernetes API and full cluster control | **AKS** |
| Standard web app/API without container platform focus | **App Service** |
| Single short-lived container/task | **ACI** |

### CLI and PowerShell examples

```bash
# Create Container Apps environment
az containerapp env create \
  --resource-group rg-containers-prod \
  --name cae-prod-eastus \
  --location eastus

# Create a Container App with HTTP ingress and scale range
az containerapp create \
  --resource-group rg-containers-prod \
  --name api-orders-prod \
  --environment cae-prod-eastus \
  --image mcr.microsoft.com/azuredocs/containerapps-helloworld:latest \
  --target-port 80 \
  --ingress external \
  --min-replicas 0 \
  --max-replicas 10

# Enable revision traffic split example
az containerapp revision list --resource-group rg-containers-prod --name api-orders-prod -o table
```

```powershell
# Create Container Apps environment
New-AzContainerAppManagedEnv -ResourceGroupName "rg-containers-prod" -Name "cae-prod-eastus" -Location "EastUS"

# Create a Container App
New-AzContainerApp -ResourceGroupName "rg-containers-prod" -Name "api-orders-prod" `
  -ManagedEnvironmentId "/subscriptions/<subId>/resourceGroups/rg-containers-prod/providers/Microsoft.App/managedEnvironments/cae-prod-eastus" `
  -Location "EastUS" -Image "mcr.microsoft.com/azuredocs/containerapps-helloworld:latest" `
  -TargetPort 80 -IngressExternal
```

### AZ-305 takeaways

- Container Apps is often the **middle ground** between App Service and AKS.
- Strong answer for **microservices without Kubernetes ops**.
- **Revisions + KEDA + Dapr** are key differentiators.

---

<a id="9-azure-container-instances-aci"></a>
## 9. Azure Container Instances (ACI)

ACI is the simplest Azure container runtime for **single containers or small container groups** that must start quickly without orchestration.

### Serverless container execution

ACI provides:
- per-second billing
- fast startup
- no cluster management
- ideal short-lived compute execution

### Container groups

A container group shares:
- lifecycle
- networking/IP
- storage context
- compute allocation boundary

Use when a sidecar pattern is needed but a full orchestrator is unnecessary.

### Common use cases

- burst compute
- build agents
- CI/CD steps
- short-lived API tests
- batch jobs or ad hoc scripts

### ACI as AKS virtual nodes

Exam concept: ACI can extend AKS for burst capacity through virtual-node style integration where appropriate.

### Limitations

- no full orchestrator
- no native rolling deployment model like AKS/App Service slots
- not ideal for long-running microservices platform designs
- scaling is not as rich as AKS or Container Apps

### When ACI vs Container Apps

| Requirement | Best answer |
|---|---|
| Quick one-off container or short job | **ACI** |
| Event-driven or HTTP app that should autoscale/scale to zero | **Container Apps** |

### CLI and PowerShell examples

```bash
# Create a simple ACI container
az container create \
  --resource-group rg-containers-prod \
  --name aci-test-runner \
  --image mcr.microsoft.com/azuredocs/aci-helloworld \
  --cpu 1 \
  --memory 1.5 \
  --ports 80 \
  --restart-policy Never

# Check status and logs
az container show --resource-group rg-containers-prod --name aci-test-runner --query instanceView.state
az container logs --resource-group rg-containers-prod --name aci-test-runner
```

```powershell
# Create ACI container group
New-AzContainerGroup -ResourceGroupName "rg-containers-prod" -Name "aci-test-runner" `
  -Location "EastUS" -Image "mcr.microsoft.com/azuredocs/aci-helloworld" `
  -OsType Linux -IpAddressType Public -Cpu 1 -MemoryInGB 1.5 -RestartPolicy Never

# Review ACI details
Get-AzContainerGroup -ResourceGroupName "rg-containers-prod" -Name "aci-test-runner"
```

### AZ-305 takeaways

- Use ACI for **fast, simple, short-lived container execution**.
- Do not choose ACI when the scenario clearly needs **orchestration, rich autoscale, or long-lived microservices governance**.

---

<a id="10-azure-functions"></a>
## 10. Azure Functions

Azure Functions is the FaaS answer for **event-driven code execution** with minimal infrastructure management.

### Hosting plans

| Plan | Best for | Scaling | Key exam note |
|---|---|---|---|
| **Consumption** | Sporadic events, cost-first serverless | Automatic scale on demand | Cold starts can matter |
| **Premium** | Low-latency serverless, VNet needs, prewarmed instances | Automatic with prewarmed capacity | Strong answer when cold start is unacceptable |
| **Dedicated (App Service plan)** | Functions sharing always-on App Service capacity | Manual/autoscale by App Service plan | Good when existing App Service capacity already exists |

### Durable Functions patterns

| Pattern | Use case |
|---|---|
| **Function chaining** | Ordered workflow steps |
| **Fan-out / fan-in** | Parallel work then aggregate |
| **Async HTTP APIs** | Long-running operation with status endpoint |
| **Monitoring** | Track workflow state |
| **Human interaction** | Approval steps and wait states |

### Triggers and bindings

Common triggers/bindings to memorize:
- HTTP
- Timer
- Storage Queue
- Service Bus
- Event Hubs
- Blob
- Cosmos DB

### Cold start mitigation

Use these when cold start matters:
- choose **Premium plan**
- keep prewarmed instances
- reduce package size/startup work
- use efficient runtimes and deployment approach

### VNet integration

For AZ-305, the safe answer is:
- choose **Premium** (or Dedicated/App Service plan) when the function app must reach private VNet resources

### Deployment options

- ZIP/package deployment
- CI/CD from GitHub Actions/Azure DevOps
- containerized Functions for specialized scenarios

### When Functions vs Logic Apps vs Container Apps

| Requirement | Best answer |
|---|---|
| Code-first event-driven processing | **Functions** |
| Workflow/connectors/low-code orchestration | **Logic Apps** |
| Containerized app or API with serverless scaling | **Container Apps** |

### CLI and PowerShell examples

```bash
# Create storage account for Functions
az storage account create \
  --resource-group rg-serverless-prod \
  --name stfuncprod001 \
  --location eastus \
  --sku Standard_LRS

# Create a Consumption plan function app
az functionapp create \
  --resource-group rg-serverless-prod \
  --consumption-plan-location eastus \
  --runtime dotnet-isolated \
  --functions-version 4 \
  --name func-orders-prod \
  --storage-account stfuncprod001

# Create a Premium plan and function app
az functionapp plan create \
  --resource-group rg-serverless-prod \
  --name plan-func-premium \
  --location eastus \
  --sku EP1 \
  --is-linux
```

```powershell
# Create Function App in a Consumption plan
New-AzFunctionApp -ResourceGroupName "rg-serverless-prod" -Name "func-orders-prod" `
  -StorageAccountName "stfuncprod001" -Runtime "dotnet-isolated" -FunctionsVersion 4 `
  -Location "EastUS"

# Create App Service plan for Premium/Dedicated scenarios
New-AzAppServicePlan -ResourceGroupName "rg-serverless-prod" -Name "plan-func-premium" `
  -Location "EastUS" -Tier "ElasticPremium" -WorkerSize "Small"
```

### AZ-305 takeaways

- **Consumption** is great for unpredictable event volume but can have cold start.
- **Premium** is the common answer for **no cold start + VNet integration**.
- **Durable Functions** is the answer when stateful orchestration is needed.

---

<a id="11-azure-batch"></a>
## 11. Azure Batch

Azure Batch is purpose-built for **large-scale parallel processing** and **HPC-style job scheduling**.

### Core concepts

| Object | Purpose |
|---|---|
| **Pool** | Group of compute nodes |
| **Job** | Logical batch workload |
| **Task** | Unit of work inside a job |
| **Node** | VM executing work |

### Large-scale parallel and HPC workloads

Best fit examples:
- media rendering
- Monte Carlo simulations
- genomics
- financial modeling
- engineering simulations
- large scheduled compute farms

### Low-priority VMs for cost savings

Batch can use low-priority/Spot-style capacity to reduce cost for interruptible jobs.

**Architect guidance:** great for checkpointable, retry-friendly workloads.

### When to use Batch

Choose Batch when the exam says:
- thousands of jobs/tasks
- parallel compute
- HPC scheduling
- rendering/simulation
- cost-sensitive batch execution

### CLI and PowerShell examples

```bash
# Create a Batch account
az batch account create \
  --resource-group rg-batch-prod \
  --name batchprod001 \
  --location eastus

# Create a pool with low-priority nodes
az batch pool create \
  --account-name batchprod001 \
  --id pool-render \
  --vm-size Standard_D4s_v5 \
  --target-dedicated-nodes 0 \
  --target-low-priority-nodes 4 \
  --image canonical:0001-com-ubuntu-server-jammy:22_04-lts \
  --node-agent-sku-id "batch.node.ubuntu 22.04"

# Create a job
az batch job create \
  --account-name batchprod001 \
  --id job-render-nightly \
  --pool-id pool-render
```

```powershell
# Get Batch account context
$batch = Get-AzBatchAccount -ResourceGroupName "rg-batch-prod" -AccountName "batchprod001"

# Review pools
Get-AzBatchPool -BatchContext $batch

# Review jobs
Get-AzBatchJob -BatchContext $batch
```

### AZ-305 takeaways

- Batch is a stronger answer than Functions/VMs for **large-scale parallel jobs**.
- Batch service itself is about orchestration; you still pay for compute/storage/networking.
- Use low-priority nodes when interruption is acceptable.

---

<a id="12-availability--resilience"></a>
## 12. Availability & Resilience

### Availability Sets

Availability Sets protect against host and maintenance failures **within a single datacenter**.

| Concept | Meaning | Exam note |
|---|---|---|
| **Fault domains (FDs)** | Separate physical racks/power/network boundaries | Protect from rack-level failure |
| **Update domains (UDs)** | Groups rebooted separately during planned maintenance | Protect from simultaneous maintenance reboot |

- Use when the scenario needs **resiliency in one datacenter** and zones are not required/available.
- Common AZ-305 distinction: **Availability Set ≠ datacenter isolation**.

### Availability Zones

Availability Zones place instances in **physically separate datacenters** within a region.

- Best for production HA requiring **datacenter-level isolation**.
- Use **zone-redundant architecture** when the question mentions failure of a full datacenter.
- Combine with **Load Balancer**, **Application Gateway**, **Front Door**, or zone-redundant platform services where relevant.

### Proximity Placement Groups (PPG)

Use PPG when the exam stresses:
- **ultra-low latency** between VMs
- tightly coupled app/database tiers
- trading some placement flexibility for co-location

Best fit: trading systems, HPC clusters, latency-sensitive app tiers.

### Dedicated Hosts

Use Azure Dedicated Hosts when you need:
- host-level isolation
- compliance or regulatory control
- software licensing tied to physical cores/sockets
- maintenance visibility over dedicated hardware

### Azure Site Recovery (ASR)

Use ASR when the scenario requires:
- region-to-region replication
- orchestrated failover and failback
- disaster recovery testing
- business continuity for VM-based workloads

> 🔑 **Critical distinction:** **Azure Backup** helps recover data. **Azure Site Recovery** helps recover service availability.

### Multi-region compute patterns

| Requirement | Common pattern |
|---|---|
| Global web app with regional failover | App Service or AKS in multiple regions behind **Azure Front Door** |
| Regional VM app with DR site | Primary VMs + **ASR** to secondary region |
| Highly available zone-aware VM fleet | **VMSS across zones** + load balancing |
| Mission-critical PaaS web app | Zone-redundant **App Service** or **Container Apps/AKS** design where supported |

### Resilience takeaways

- **Availability Set** = same datacenter resilience.
- **Availability Zone** = datacenter isolation inside a region.
- **Multi-region** = regional disaster scenario protection.
- **PPG** = latency optimization, not HA.
- **Dedicated Host** = isolation/compliance, not automatically the cheapest or simplest design.

---

<a id="13-cost-optimization"></a>
## 13. Cost Optimization

### Spot VMs

| Item | Key fact |
|---|---|
| Best for | Interruptible workloads, stateless jobs, batch, test farms |
| Not for | Mission-critical stateful workloads |
| Eviction policy | **Deallocate** or **Delete** |
| Eviction triggers | Capacity pressure or max price policy |

**Architect guidance:** Use Spot when workloads can checkpoint, retry, or tolerate interruption.

### Reserved Instances

| Term | Best for | Benefit | Tradeoff |
|---|---|---|---|
| **1-year RI** | Predictable medium-term workloads | Solid savings with shorter commitment | Less savings than 3-year |
| **3-year RI** | Stable long-running workloads | Highest savings | Longest commitment |

Key facts:
- Scope can be **shared** or **single subscription**.
- Instance size flexibility may apply within a VM family, improving utilization.
- Use RIs for **steady-state production**; avoid for uncertain or short-lived workloads.

### Savings Plans for Compute

Use **Azure Savings Plan for Compute** when the organization wants commitment-based savings with **more flexibility than Reserved Instances**. It applies to eligible compute usage across services and is often the better answer when workloads move between VM families, regions, or services over time.

| Option | Best for | Tradeoff |
|---|---|---|
| **Reserved Instances** | Stable, specific VM footprint | Highest fit when usage is predictable and fixed |
| **Savings Plan** | Variable but steady compute spend | More flexible, sometimes slightly less savings than the best RI fit |

### Azure Hybrid Benefit

Use Azure Hybrid Benefit to reuse eligible on-premises Windows Server or SQL Server licenses with Software Assurance / qualifying subscription benefits.

**Exam trigger words:** existing Windows licenses, reduce licensing cost, SQL Server on Azure VM, bring your own license.

### Service-specific cost notes

- **AKS**: use **Spot node pools**, right-size system/user pools separately, and use **cluster autoscaler** to avoid idle nodes.
- **App Service / Functions Dedicated / Premium**: commitments may benefit from **Savings Plans** for steady workloads.
- **Container Apps / Functions Consumption**: pay for active usage; strong fit for spiky or low-utilization workloads.
- **Batch**: use low-priority or Spot-style pool capacity where interruption is acceptable.

### Cost optimization takeaways

- **Spot** = biggest discount, highest interruption risk.
- **Reserved Instances** = strongest answer for predictable 24x7 workloads.
- **Savings Plans** = flexible commitment model across eligible compute.
- **Azure Hybrid Benefit** = licensing savings on top of compute optimization.
- Combining **RI + Azure Hybrid Benefit** is often the strongest steady-state Windows VM answer.

---

<a id="14-az-305-decision-scenarios"></a>
## 14. AZ-305 Decision Scenarios

### Scenario 1: Web app hosting decision
**Situation:** A company needs a public .NET web app with deployment slots, Entra ID auth, autoscale, and minimal admin overhead.  
**Best answer:** **Azure App Service**  
**Why:** Slots + Easy Auth + autoscale + PaaS operations.

### Scenario 2: Microservices architecture
**Situation:** A fintech team needs microservices, service mesh, custom ingress, network policies, and kubectl access.  
**Best answer:** **AKS**  
**Why:** Kubernetes control is explicitly required.

### Scenario 3: Batch processing workload
**Situation:** Nightly rendering workload must process millions of frames by morning at lowest possible cost.  
**Best answer:** **Azure Batch with low-priority nodes**  
**Why:** Native parallel scheduling plus interruptible cost optimization.

### Scenario 4: Machine learning training
**Situation:** Data scientists require NVIDIA GPU-backed compute for model training jobs that run for several hours.  
**Best answer:** **N-series VMs or Azure Batch with N-series pools**  
**Why:** GPU support and flexible compute scheduling.

### Scenario 5: Legacy app migration
**Situation:** A 15-year-old Windows application needs lift-and-shift migration with minimal code changes.  
**Best answer:** **Azure Virtual Machines**  
**Why:** Preserves OS/runtime compatibility and admin control.

### Scenario 6: Burstable workloads
**Situation:** An internal line-of-business app runs at low CPU most of the day but spikes heavily during month-end close.  
**Best answer:** **B-series VMs**  
**Why:** Burstable economics fit low-baseline, occasional-peak CPU patterns.

### Scenario 7: Multi-region deployment
**Situation:** Global web application requires low latency worldwide and resilient regional failover.  
**Best answer:** **App Service or AKS deployed in multiple regions behind Azure Front Door**  
**Why:** Regional compute + global traffic routing.

### Scenario 8: Cost optimization
**Situation:** A steady-state Windows server fleet runs 24x7 with existing licenses and predictable usage.  
**Best answer:** **Reserved Instances + Azure Hybrid Benefit**  
**Why:** Strongest combined VM cost savings pattern.

### Scenario 9: High-performance computing
**Situation:** Scientific modeling requires MPI-style low-latency node communication.  
**Best answer:** **H-series VMs or Azure Batch with HPC-capable nodes**  
**Why:** HPC-optimized compute and scheduling support.

### Scenario 10: IoT edge processing
**Situation:** Telemetry arrives in bursts from thousands of devices and must trigger event-driven processing.  
**Best answer:** **Azure Functions**  
**Why:** Event-driven scale and native trigger model.

### Scenario 11: Containerized APIs without Kubernetes complexity
**Situation:** Team packages APIs as containers but does not want to operate Kubernetes. Traffic is variable and should scale to zero.  
**Best answer:** **Azure Container Apps**  
**Why:** Serverless containers + KEDA + scale-to-zero.

### Scenario 12: Short-lived build agents
**Situation:** CI runners are needed for 20-minute jobs, then should disappear.  
**Best answer:** **ACI**  
**Why:** Fast startup and per-second billing without cluster overhead.

### Scenario 13: Private serverless integration workflow
**Situation:** Queue-triggered processing must access private databases in a VNet, and cold start is unacceptable.  
**Best answer:** **Azure Functions Premium**  
**Why:** Better latency profile and VNet integration support.

### Scenario 14: Autoscaling stateless app tier on VMs
**Situation:** Existing VM-based web tier must autoscale with health checks and rolling upgrades, but app is not containerized.  
**Best answer:** **VMSS**  
**Why:** Fleet management, autoscale, and upgrade controls without app refactoring.

### Quick Reference Trigger Table

| # | Trigger phrase in the question | Best answer | Why |
|---|---|---|---|
| 1 | Full OS control | VMs | IaaS control plane + guest OS access |
| 2 | Lift and shift | VMs | Least refactoring |
| 3 | Legacy Windows app | VMs or App Service (Windows) | Depends on compatibility and modernization scope |
| 4 | Burstable CPU | B-series VMs | Lowest-cost occasional burst pattern |
| 5 | General enterprise app server | D-series VMs | Balanced CPU/memory |
| 6 | Memory-heavy workload | E-series or M-series | High RAM ratio |
| 7 | Compute-heavy workload | F-series | More CPU per GB |
| 8 | SAP HANA | M-series | Huge memory, certified fit |
| 9 | GPU training/rendering | N-series | GPU-backed compute |
| 10 | High IOPS local storage | L-series | Storage-optimized |
| 11 | HPC / MPI | H-series | HPC-focused compute |
| 12 | Fault/update domains | Availability Set | Same-datacenter resilience |
| 13 | Datacenter isolation in region | Availability Zones | Separate datacenters |
| 14 | Ultra-low latency between VMs | Proximity Placement Group | Co-location benefit |
| 15 | Host isolation / licensing | Dedicated Host | Dedicated physical server |
| 16 | Interruptible cheap compute | Spot VMs | Accept eviction for savings |
| 17 | Predictable 24x7 VM workload | Reserved Instances | Commit for savings |
| 18 | Existing Windows/SQL licenses | Azure Hybrid Benefit | Lower licensing cost |
| 19 | Bootstrap post-deploy config | Custom Script Extension | Run initialization scripts |
| 20 | Identical autoscaling VM fleet | VMSS Uniform | Best for homogeneous scale |
| 21 | Mixed VM sizes in scale set | VMSS Flexible | Greater heterogeneity |
| 22 | Health-based instance replacement | VMSS automatic repairs | Self-healing fleet |
| 23 | Kubernetes required | AKS | Full K8s control |
| 24 | Windows node pools in AKS | Azure CNI | Stronger networking compatibility |
| 25 | Scale containers to zero | Container Apps | Serverless container model |
| 26 | Dapr microservices building blocks | Container Apps | Native Dapr option |
| 27 | Standard enterprise web app/API | App Service | PaaS simplicity |
| 28 | Blue/green swap | App Service slots | Native deployment slots |
| 29 | Inbound private web app access | App Service Private Endpoint | Private ingress |
| 30 | Outbound app access into VNet | App Service VNet Integration | Private outbound reach |
| 31 | One-off container job | ACI | Fast and simple |
| 32 | Event-driven code | Functions | FaaS execution model |
| 33 | No cold start serverless | Functions Premium | Prewarmed instances |
| 34 | Workflow orchestration | Durable Functions | Stateful serverless flow |
| 35 | Massive parallel job farm | Batch | Pools/jobs/tasks model |
| 36 | Cheapest batch compute | Batch + low-priority nodes | Cost-optimized parallelism |

---

<a id="15--final-az-305-exam-tips"></a>
## 15. 🎯 Final AZ-305 Exam Tips

1. **Default to managed services first.** If App Service, Container Apps, or Functions can meet the requirement, they often beat VMs on the exam because of lower operational overhead.
2. **Use VMs only when you truly need OS-level control.** Trigger phrases include lift-and-shift, custom drivers, special agents, domain join, or legacy software.
3. **Separate AKS from “just containers.”** Choose AKS only when Kubernetes itself is required; otherwise Container Apps is often the better answer.
4. **Remember the App Service networking directionality.** VNet Integration is outbound; Private Endpoint is inbound.
5. **Know the resilience ladder.** Availability Set = same datacenter, Availability Zone = datacenter isolation, multi-region = region failure protection.
6. **Match VM family to workload shape quickly.** B = burstable, D = general-purpose, E/M = memory, F/H = compute/HPC, N = GPU, L = storage.
7. **Don’t misuse Spot.** It is for interruptible, retry-friendly workloads only.
8. **For predictable 24x7 compute, think commitment savings.** Reserved Instances and Savings Plans are common cost optimization answers; add Azure Hybrid Benefit if licensing fits.
9. **Use Functions Premium when cold start or private networking matters.** Consumption is not always the right serverless answer.
10. **Batch beats generic compute for massive parallel jobs.** If the question mentions rendering, simulations, or thousands of parallel tasks, think Azure Batch.

### Common exam traps

#### 1. VMSS Uniform vs Flexible confusion
- **Wrong mindset:** VMSS is always only for identical instances.
- **Correct mindset:** **Uniform** is the classic identical fleet model; **Flexible** supports more heterogeneous and VM-like deployments.

#### 2. Availability Set vs Availability Zone confusion
- **Wrong mindset:** Availability Set protects against datacenter failure.
- **Correct mindset:** Availability Set protects from **host/rack/update** issues inside one datacenter; Zones protect across datacenters in a region.

#### 3. AKS networking model decision
- **Wrong mindset:** kubenet is always the cheapest/best answer.
- **Correct mindset:** if the scenario needs **Windows nodes**, **advanced networking**, or **direct VNet integration**, Azure CNI is usually the safer answer.

#### 4. App Service Plan tier selection
- **Wrong mindset:** Basic is enough for most production workloads.
- **Correct mindset:** production answers often land on **Standard** or **Premium** because of autoscale, slots, networking, and resilience needs.

#### 5. App Service VNet directionality
- **Wrong mindset:** VNet Integration gives private inbound access.
- **Correct mindset:** **VNet Integration = outbound**; **Private Endpoint = inbound**.

#### 6. Functions Consumption cold start issues
- **Wrong mindset:** Consumption is always the best serverless answer.
- **Correct mindset:** if the app needs **predictable latency** or **private networking**, prefer **Premium**.

#### 7. Container Apps vs AKS decision
- **Wrong mindset:** Any containerized app should go to AKS.
- **Correct mindset:** use AKS only when Kubernetes capability is truly required; otherwise Container Apps may be the better operational choice.

#### 8. ACI vs Container Apps decision
- **Wrong mindset:** ACI is a general microservices platform.
- **Correct mindset:** ACI is best for **simple, short-lived containers**; Container Apps is better for app-style scaling and traffic.

#### 9. Backup vs DR confusion
- **Wrong mindset:** Azure Backup alone solves disaster recovery.
- **Correct mindset:** Backup protects data recovery; **Site Recovery** addresses failover/orchestration.

#### 10. Spot VM misuse
- **Wrong mindset:** Spot is a universal cost optimization for production.
- **Correct mindset:** only use Spot where interruption is acceptable and the app can retry or recover.

#### 11. Reserved capacity misuse
- **Wrong mindset:** Reserved Instances are best for every workload.
- **Correct mindset:** use reservations for **predictable steady-state** workloads, not uncertain short-lived projects.

#### 12. PPG misuse
- **Wrong mindset:** Proximity Placement Groups improve resiliency.
- **Correct mindset:** PPG is primarily about **low latency**, not HA.

#### 13. Batch vs Functions confusion
- **Wrong mindset:** Functions is the default for any parallel workload.
- **Correct mindset:** Batch is the better answer for **large-scale scheduled parallel compute**.

#### 14. App Service Environment overuse
- **Wrong mindset:** ASE is the default secure App Service answer.
- **Correct mindset:** ASE is a premium isolation answer; choose it only when dedicated isolated hosting is explicitly needed.

#### 15. Serverless vs PaaS over-rotation
- **Wrong mindset:** Functions beats App Service for every web/API scenario.
- **Correct mindset:** App Service is often better for long-running web apps and APIs; Functions is for event-driven execution.

### Final revision checklist

- [ ] Can I explain when to choose **VMs, VMSS, AKS, App Service, Container Apps, ACI, Functions, and Batch**?
- [ ] Do I remember **Availability Set vs Availability Zone vs multi-region**?
- [ ] Can I match **VM family** to workload shape quickly?
- [ ] Do I know when **Spot, Reserved Instances, Savings Plans, and Hybrid Benefit** apply?
- [ ] Can I explain **AKS networking choices** and when Azure CNI is preferred?
- [ ] Do I remember **App Service slots**, **Easy Auth**, and **VNet directionality**?
- [ ] Can I explain **Functions Consumption vs Premium vs Dedicated**?
- [ ] Do I know when **Batch** is more appropriate than Functions or VMs?

---

<a id="16--architecture-decision-flowchart"></a>
## 16. 📐 Architecture Decision Flowchart

```text
                           ┌───────────────────────────────┐
                           │   Start with requirements     │
                           └───────────────┬───────────────┘
                                           │
                     ┌─────────────────────▼─────────────────────┐
                     │ Need full OS control or legacy support?  │
                     └───────────────┬───────────────────────────┘
                                     │
                          ┌──────────▼──────────┐
                          │ YES → VMs / VMSS    │
                          └──────────┬──────────┘
                                     │
                    ┌────────────────▼─────────────────┐
                    │ Need autoscaling VM fleet?       │
                    └───────────────┬──────────────────┘
                                    │
                         ┌──────────▼──────────┐
                         │ YES → VMSS          │
                         │ NO  → Individual VM │
                         └─────────────────────┘
                                     │
                                     NO
                                     │
              ┌──────────────────────▼──────────────────────┐
              │ Is the workload containerized?              │
              └───────────────┬─────────────────────────────┘
                              │
                ┌─────────────▼─────────────┐
                │ NO                        │ YES
                ▼                           ▼
   ┌───────────────────────────┐   ┌────────────────────────────────┐
   │ Event-driven / sporadic?  │   │ Need Kubernetes control plane? │
   └──────────────┬────────────┘   └──────────────┬─────────────────┘
                  │                               │
        ┌─────────▼─────────┐           ┌────────▼────────┐
        │ YES → Functions   │           │ YES → AKS       │
        └─────────┬─────────┘           └────────┬────────┘
                  │                               │
                  │                    ┌──────────▼──────────┐
                  │                    │ Need simple server-  │
                  │                    │ less containers?     │
                  │                    └──────────┬──────────┘
                  │                               │
        ┌─────────▼─────────┐           ┌─────────▼──────────┐
        │ NO → App Service  │           │ YES → Container    │
        │   (web/API)       │           │ Apps               │
        └───────────────────┘           └─────────┬──────────┘
                                                   │
                                      ┌────────────▼────────────┐
                                      │ One-off container job?  │
                                      └────────────┬────────────┘
                                                   │
                                         ┌─────────▼─────────┐
                                         │ YES → ACI         │
                                         └───────────────────┘

Extra decision: if the requirement says **thousands of parallel tasks / rendering / simulation**, branch directly to **Azure Batch**.
```

---

<a id="17-exam-style-review-questions"></a>
## 17. Exam-Style Review Questions

1. A company is migrating a legacy Windows application that requires OS-level customization, domain join, and a custom monitoring agent. Which Azure compute option is the best fit, and why would App Service be a poor choice?
2. A team has containerized microservices and needs Dapr, KEDA-based queue scaling, revision-based rollbacks, and scale-to-zero, but does not want to operate Kubernetes. Which service should they choose, and why is AKS not the best answer?
3. An organization needs a public API with deployment slots, built-in Entra ID authentication, autoscale, and minimal management overhead. Which service should they select, and what two features make it stand out on AZ-305?
4. A scientific workload must run thousands of parallel simulations overnight using low-cost interruptible compute, and failed work can be retried. Which Azure service and pricing approach should you recommend?
5. A queue-triggered serverless workflow must access private resources inside a VNet and avoid cold starts during peak business hours. Which Functions hosting plan is most appropriate, and why?

*Last Updated: 2025 | Target: AZ-305 Designing Microsoft Azure Infrastructure Solutions*
