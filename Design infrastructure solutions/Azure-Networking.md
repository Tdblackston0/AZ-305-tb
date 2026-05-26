<a id="top"></a>
# Azure Networking - AZ-305 Comprehensive Cheat Sheet

> 📝 **Hands-On Labs:** [Networking Labs](./Labs/Azure-Networking-Labs.md)

> 🎯 **Exam Focus:** AZ-305 tests your ability to **design** network topologies and select the right connectivity, security, and traffic distribution services.

## Table of Contents

- [Azure Networking Family Overview](#1-azure-networking-family-overview)
- [When to Choose Which — Decision Tree](#2-when-to-choose-which-decision-tree)
- [Virtual Networks (VNet)](#3-virtual-networks-vnet)
- [Network Security](#4-network-security)
- [Load Balancing Services](#5-load-balancing-services)
- [Hybrid Connectivity](#6-hybrid-connectivity)
- [Private Connectivity](#7-private-connectivity)
- [DNS](#8-dns)
- [Network Monitoring](#9-network-monitoring)
- [Availability & Resilience](#10-availability-resilience)
- [Cost Optimization](#11-cost-optimization)
- [AZ-305 Decision Scenarios](#12-az-305-decision-scenarios)
- [Quick Reference Trigger Table](#13-quick-reference-trigger-table)
- [Common Exam Traps](#14-common-exam-traps)
- [Final AZ-305 Exam Tips](#15-final-az-305-exam-tips)
- [Architecture Decision Flowchart](#16-architecture-decision-flowchart)
- [Exam-Style Review Questions](#17-exam-style-review-questions)

---

<a id="1-azure-networking-family-overview"></a>
## 1. Azure Networking Family Overview

Azure networking decisions on AZ-305 usually come down to five design questions: **scope** (subnet, VNet, region, global), **protocol** (TCP/UDP vs HTTP/S), **trust boundary** (public, private, inspected), **connectivity path** (Azure-only, hybrid, branch), and **resilience target** (zone, region, global). Start with the service family, then refine the exact SKU and topology.

### Comparison Table

| Service | Primary role | Scope | Best when | Watch out for |
|---|---|---|---|---|
| **VNet** | Private network boundary | Regional | You need isolation, subnets, routing, peering | Overlapping CIDR blocks break future connectivity |
| **NSG** | Stateful L3/L4 filtering | Subnet/NIC | You need low-cost segmentation and deny/allow rules | No advanced L7 inspection or TLS awareness |
| **Azure Firewall** | Central managed firewall | Regional hub / Virtual WAN | You need centralized egress, DNAT, threat intel, TLS inspection | Higher cost than NSGs; route design matters |
| **Application Gateway** | Regional L7 reverse proxy | Regional | You need path-based routing, SSL offload, WAF | HTTP/S only; not for generic TCP/UDP |
| **Load Balancer** | Regional L4 load balancing | Regional | You need TCP/UDP balancing for public or internal endpoints | No URL routing or WAF features |
| **Front Door** | Global edge HTTP/S delivery | Global | You need anycast entry, acceleration, global failover, edge WAF | HTTP/S only; not a private east-west service |
| **Traffic Manager** | DNS-based endpoint selection | Global | You need simple global failover for any protocol | DNS caching slows failover visibility |
| **Private Link** | Private PaaS/service access | Regional with hybrid reach | You must use private IPs and disable public exposure | DNS is part of the design, not optional |
| **ExpressRoute** | Private dedicated hybrid connectivity | Hybrid / global enterprise | You need predictable private connectivity and compliance | More expensive and slower to provision than VPN |
| **VPN Gateway** | Encrypted internet-based connectivity | Hybrid / branch / user | You need fast, flexible hybrid or remote-user access | Throughput and SLA lower than ExpressRoute |
| **Virtual WAN** | Managed transit networking | Global / multi-branch | You need many sites, regions, and centralized routing at scale | Less customizable than self-built hub-spoke |

### Virtual Network (VNet)
A VNet is Azure's logical private network boundary for IP planning, segmentation, routing, peering, and service placement. It is the starting point when the design requires subnet isolation, private east-west communication, or a landing zone foundation.

**Real-World Examples:**
- A **global retailer** allocates separate hub and spoke VNets per region so stores, e-commerce apps, and shared security services stay isolated but connected.
- A **healthcare provider** places web, app, data, and management tiers in separate subnets inside one VNet to enforce least-privilege east-west access.
- A **merging enterprise** reserves large non-overlapping VNet ranges in Azure to avoid address conflicts with future acquired networks.

### Network Security Group (NSG)
An NSG is a stateful packet-filtering policy applied at the subnet or NIC level to allow or deny traffic based on source, destination, protocol, and port. Use it for baseline micro-segmentation and deny-by-default controls close to the workload.

**Real-World Examples:**
- A **three-tier application** allows HTTPS from the web subnet to the app subnet and SQL only from the app subnet to the data subnet.
- A **regulated finance workload** denies outbound internet access from database subnets while still allowing backup traffic to approved service tags.
- A **shared services platform** uses NSGs on spoke subnets so only the Bastion and management subnet can reach VM admin ports.

### Azure Firewall
Azure Firewall is a fully managed, centralized, stateful firewall for hub-based or secured virtual hub architectures. It is best when you must control outbound internet access, centralize DNAT, or apply threat-intelligence and TLS-inspection policies across many spokes.

**Real-World Examples:**
- An **enterprise landing zone** forces all spoke outbound internet traffic through a hub firewall for FQDN filtering and audit logging.
- A **payment-processing environment** uses Azure Firewall Premium for TLS inspection and IDPS before traffic reaches card-processing APIs.
- A **manufacturing company** centralizes branch-to-cloud inspection through a secured Virtual WAN hub with Azure Firewall policies.

### Application Gateway
Application Gateway is a regional Layer 7 load balancer and reverse proxy for HTTP/S applications. Choose it when you need URL/path-based routing, host-based routing, SSL termination, cookie affinity, or integrated WAF in a single Azure region.

**Real-World Examples:**
- An **insurance portal** routes `/claims` and `/billing` to different backend pools while publishing a single public endpoint.
- A **B2B SaaS provider** hosts multiple customer domains on one gateway using host-header routing and separate TLS certificates.
- A **government web app** uses WAF mode to block OWASP attacks before traffic reaches private application servers.

### Azure Load Balancer
Azure Load Balancer is a Layer 4 service for distributing TCP/UDP traffic across healthy instances. It is the right answer for non-HTTP workloads, internal tier balancing, or high-performance regional front ends where application-aware routing is unnecessary.

**Real-World Examples:**
- A **SQL Server Always On listener** uses an internal load balancer to present a single private endpoint to application servers.
- A **legacy ERP farm** publishes TCP-based application servers behind a public Standard Load Balancer.
- A **high-scale network virtual appliance pair** uses HA Ports on an internal load balancer to process east-west flows.

### Azure Front Door
Azure Front Door is Microsoft's global edge application delivery platform for HTTP/S workloads. It provides an anycast entry point, global load balancing, acceleration, WAF, caching, and failover across regions.

**Real-World Examples:**
- A **multi-region e-commerce platform** sends shoppers to the closest healthy region and fails over automatically during a regional outage.
- A **media site** uses edge caching and WAF at Front Door to reduce origin load and improve page performance worldwide.
- A **private App Service deployment** exposes global web traffic through Front Door Premium with Private Link origins.

### Traffic Manager
Traffic Manager is a DNS-based global traffic distribution service that returns the best endpoint in DNS responses based on routing rules. It works for almost any endpoint type and protocol, but it does not proxy traffic.

**Real-World Examples:**
- A **global API provider** uses priority routing so a secondary region is returned only if the primary region fails health checks.
- A **gaming company** uses performance routing to send players to the lowest-latency regional service endpoint.
- A **hybrid application** returns either an Azure endpoint or an on-premises endpoint because the workload is not HTTP/S and cannot use Front Door.

### Private Link
Private Link exposes Azure PaaS services or customer services through private IP addresses in your VNet using Private Endpoints. It is the preferred choice when the design requires private-only access, data exfiltration control, and hybrid reach without public endpoints.

**Real-World Examples:**
- A **bank** disables public access to Azure SQL and forces all app traffic through a Private Endpoint with private DNS resolution.
- A **research lab** publishes an internal SaaS service to partner subscriptions through Private Link Service instead of over the public internet.
- An **enterprise data platform** allows on-prem users to access a Storage account privately over ExpressRoute using Private Endpoints and DNS forwarding.

### ExpressRoute
ExpressRoute provides private, dedicated connectivity between on-premises environments and Microsoft cloud services. It is the premium hybrid option when the scenario emphasizes predictable latency, compliance, or keeping traffic off the public internet.

**Real-World Examples:**
- A **large bank** connects its datacenters to Azure over redundant ExpressRoute circuits for core payment workloads.
- A **pharmaceutical company** uses ExpressRoute Direct for very high-throughput data ingestion from lab systems into Azure analytics platforms.
- A **multinational enterprise** uses Global Reach so geographically separate offices can exchange traffic over Microsoft's backbone.

### VPN Gateway
VPN Gateway provides encrypted connectivity over the internet for site-to-site, point-to-site, and VNet-to-VNet scenarios. It is ideal when you need faster deployment or lower cost than ExpressRoute and can tolerate internet-based characteristics.

**Real-World Examples:**
- A **small branch office** connects to Azure over a site-to-site VPN while waiting for long-term WAN upgrades.
- A **remote workforce** connects securely to private Azure apps over point-to-site OpenVPN.
- A **disaster recovery design** uses VPN Gateway as backup connectivity if the primary ExpressRoute path fails.

### Virtual WAN
Virtual WAN is a managed Microsoft service for large-scale branch, user, and Azure transit networking. It simplifies multi-region and multi-branch connectivity by integrating hubs, routing, VPN, ExpressRoute, and security services under one control plane.

**Real-World Examples:**
- A **retail chain with hundreds of stores** connects branches globally through managed Virtual WAN hubs instead of building many custom hub VNets.
- A **professional services firm** centralizes remote-user VPN, branch VPN, and Azure VNet connectivity in one managed routing architecture.
- A **global manufacturer** adds Azure Firewall to secured virtual hubs so branch internet egress is inspected consistently worldwide.

### Preserved networking overview guidance
### Original Azure Networking Overview (preserved)

Azure networking for AZ-305 is about choosing the **right traffic path, trust boundary, and control plane** for each workload. The exam is less about memorizing every SKU and more about understanding **when to isolate, inspect, publish, encrypt, and scale**.

### Network topology patterns

> 🎯 **Exam Focus:** For the exam, focus primarily on **Hub-and-Spoke** topology. It's the default enterprise pattern and will appear on the test. Outside of Virtual WAN, it's realistically the only pattern most customers consider for production workloads.

| Pattern | Best for | Strengths | Tradeoffs | Architect guidance |
|---|---|---|---|---|
| **Hub-spoke** | Enterprise landing zones, shared services, centralized security | Centralized routing, inspection, DNS, egress control | More routing complexity, possible hub bottlenecks | Default enterprise choice for most AZ-305 scenarios |
| **Mesh** | Small set of VNets with heavy east-west communication | Simple direct paths, lower latency between specific VNets | Hard to scale and govern; peering sprawl | Use sparingly; avoid at large scale |
| **Flat** | Small dev/test environments | Fast to deploy, low complexity | Weak segmentation, larger blast radius | Rarely best for production |
| **Virtual WAN** | Global branch connectivity and large-scale hub-spoke | Managed transit architecture, simplified branch onboarding | Less granular than self-managed hubs | Strong choice for many sites/regions |

### Defense in depth for networking

Use layered controls, not a single control:

1. **Segmentation** - separate apps, data, management, and shared services into subnets/VNets.
2. **Perimeter protection** - Azure Firewall, WAF, DDoS Protection Standard.
3. **Identity-aware access** - Private Link, Entra ID, JIT/JEA, Conditional Access for admins.
4. **Routing control** - user-defined routes (UDRs), forced tunneling, hub inspection.
5. **Monitoring** - Network Watcher, NSG flow logs, Connection Monitor, Firewall logs.
6. **Least privilege** - deny by default, narrow ports, narrow sources, narrow destinations.

### Zero Trust network principles

- **Assume breach** - do not trust traffic only because it is on the corporate network.
- **Verify explicitly** - validate identity, device, application, and context.
- **Use least privilege** - minimal network exposure, minimal inbound paths, minimal lateral movement.
- **Prefer private access** - Private Endpoints, Bastion, private DNS, no public IP unless required.
- **Inspect critical flows** - use WAF/Firewall for north-south and segmentation for east-west.

### Core design heuristics

- Prefer **private-by-default** PaaS access.
- Prefer **hub-spoke** over flat networks for production.
- Prefer **Standard** SKUs for security and HA features.
- Prefer **regional service first**, then add **global service** when the workload is multi-region.
- Keep **management traffic** separate from application traffic where possible.

### Quick commands

```bash
az network vnet list -o table
az network watcher list -o table
az network public-ip list -o table
```

```powershell
Get-AzVirtualNetwork
Get-AzNetworkWatcher
Get-AzPublicIpAddress
```

---

<a id="2-when-to-choose-which-decision-tree"></a>
## 2. When to Choose Which — Decision Tree

### Load Balancing Decision Flow

```text
┌────────────────────────────────────────────────────────────┐
│                  What traffic are you routing?            │
└───────────────────────────────┬────────────────────────────┘
                                │
                    ┌───────────┴───────────┐
                    │ HTTP/S                │ Non-HTTP (TCP/UDP)
                    ▼                       ▼
        ┌──────────────────────────┐   ┌──────────────────────────┐
        │ Need a global edge, WAF, │   │ Need regional balancing? │
        │ acceleration, or caching?│   └──────────────┬───────────┘
        └──────────────┬───────────┘                  │
                       │                              │
             ┌─────────┴─────────┐                    │
             │ YES               │ NO                 │
             ▼                   ▼                    ▼
    ┌──────────────────┐  ┌──────────────────────┐  ┌────────────────────┐
    │ Azure Front Door │  │ Need path/host-based │  │ Azure Load Balancer│
    │   (Global L7)    │  │ routing or regional  │  │   (Regional L4)    │
    └──────────────────┘  │ WAF inside one region?│  └────────────────────┘
                          └───────────┬───────────┘
                                      │
                            ┌─────────┴─────────┐
                            │ YES               │ NO
                            ▼                   ▼
                   ┌────────────────────┐  ┌─────────────────────┐
                   │ Application Gateway│  │ Traffic Manager if  │
                   │   (Regional L7)    │  │ DNS-only steering   │
                   └────────────────────┘  │ is acceptable       │
                                           └─────────────────────┘
```

### Connectivity Decision Flow

> ⚠️ **Important:** Service Endpoints and ExpressRoute solve **different problems**. ExpressRoute is for hybrid connectivity (on-prem to Azure). Service Endpoints enable firewall/access control over PaaS resources from specific subnets — they do NOT provide hybrid connectivity.

```text
┌────────────────────────────────────────────────────────────┐
│                  What kind of connectivity?               │
└───────────────────────────────┬────────────────────────────┘
                                │
               ┌────────────────┼────────────────┐
               │                │                │
               ▼                ▼                ▼
      Azure-to-Azure      Private access to   Azure-to-on-prem /
      network design      Azure PaaS/service  branch / users
               │                │                │
               ▼                ▼                ▼
   ┌────────────────────┐  ┌──────────────────┐  ┌─────────────────────────┐
   │ Need direct, low-  │  │ Must disable     │  │ Need hybrid connectivity│
   │ latency private    │  │ public access or │  │ (on-prem to Azure)?     │
   │ connectivity?      │  │ use a private IP?│  └──────────────┬──────────┘
   └──────────┬─────────┘  └──────────┬───────┘                 │
              │                       │                ┌────────┴────────┐
     ┌────────┴────────┐      ┌───────┴────────┐       │ YES             │ NO
     │ YES             │ NO   │ YES            │ NO    ▼                 ▼
     ▼                 ▼      ▼                ▼  ┌─────────────────┐  (No hybrid
┌──────────────┐  ┌──────────┐ ┌──────────────┐ ┌──────────────┐ │ Need dedicated │   needed)
│ VNet Peering │  │ Virtual  │ │ Private Link │ │ Service      │ │ private path   │
│ (or Global   │  │ WAN / Hub│ │ / Private    │ │ Endpoint     │ │ w/ compliance? │
│ Peering)     │  │ design   │ │ Endpoint     │ │ (subnet-level│ └────────┬──────┘
└──────────────┘  └──────────┘ └──────────────┘ │ PaaS access) │          │
                                                └──────────────┘ ┌────────┴────────┐
                                                                 │ YES             │ NO
                                                                 ▼                 ▼
                                                         ┌──────────────┐  ┌──────────────┐
                                                         │ ExpressRoute │  │ VPN Gateway  │
                                                         └──────┬───────┘  │ (S2S/P2S)    │
                                                                │          └──────────────┘
                                                                ▼
                                                   ┌────────────────────────┐
                                                   │ Need many branches or  │
                                                   │ managed global transit?│
                                                   └───────────┬────────────┘
                                                               │
                                                     ┌─────────┴─────────┐
                                                     │ YES               │ NO
                                                     ▼                   ▼
                                            ┌────────────────┐  ┌────────────────┐
                                            │  Virtual WAN   │  │  VPN Gateway   │
                                            │  (managed)     │  │ (self-managed) │
                                            └────────────────┘  └────────────────┘
```

**Service Endpoints vs Private Endpoints — Key Distinction:**
- **Service Endpoint**: Extends your VNet identity to PaaS services, enabling firewall rules that restrict PaaS access to specific subnets. The service **still uses its public endpoint**. Good for simple subnet-level restriction when public access can remain enabled.
- **Private Endpoint**: Creates a private IP in your VNet for the PaaS service via Private Link. Enables you to **disable the public endpoint entirely**. Required for true private-only access.

**Services that support Service Endpoints:**
Azure Storage, Azure SQL Database, Azure Cosmos DB, Azure Key Vault, Azure Service Bus, Azure Event Hubs, Azure App Service, Azure Container Registry, Azure Cognitive Services, and others. Check Microsoft documentation for the current list.

### Quick Decision Matrix

| If the requirement says... | Start with... |
|---|---|
| Global web entry point, WAF, acceleration | **Azure Front Door** |
| Regional web reverse proxy, path routing, WAF | **Application Gateway** |
| Regional TCP/UDP distribution | **Azure Load Balancer** |
| DNS-based global failover for any protocol | **Traffic Manager** |
| Private PaaS access with no public endpoint | **Private Link / Private Endpoint** |
| Dedicated private hybrid connectivity | **ExpressRoute** |
| Fast, lower-cost encrypted hybrid connectivity | **VPN Gateway** |
| Many branches and managed transit at scale | **Virtual WAN** |

---

<a id="3-virtual-networks-vnet"></a>
## 3. Virtual Networks (VNet)


### VNet design: address space planning and CIDR

A VNet is the logical isolation boundary for Azure networking. Good VNet design starts with **future growth and hybrid overlap avoidance**.

### Address planning rules

- Use **RFC1918** space unless there is a specific reason not to.
- Avoid overlapping ranges across:

> 🔴 **HIGH-YIELD EXAM TOPIC:** Know the RFC1918 private address ranges by heart. Expect 1-2 questions where you must identify valid private IP ranges or spot when an address is NOT RFC1918 compliant.
>
> **RFC1918 Private Address Ranges:**
> | Range | CIDR | Total addresses | Common use |
> |---|---|---|---|
> | `10.0.0.0` – `10.255.255.255` | `10.0.0.0/8` | ~16.7 million | Large enterprise networks |
> | `172.16.0.0` – `172.31.255.255` | `172.16.0.0/12` | ~1 million | Mid-size networks |
> | `192.168.0.0` – `192.168.255.255` | `192.168.0.0/16` | ~65,000 | Small networks / home |
>
> **Exam Tip:** Any IP address starting with `172.32.x.x` through `172.255.x.x` is **NOT** private RFC1918 space — this is a common exam trap!
  - on-premises networks
  - other Azure VNets
  - future DR regions
  - acquired/business-partner networks if mergers are realistic
- Reserve space for:
  - shared services
  - firewalls/NVAs
  - gateway subnets
  - Bastion
  - AKS/App Gateway growth
- Document CIDR ownership centrally.

### Common CIDR examples

| Scope | Example | Notes |
|---|---|---|
| Enterprise Azure supernet | `10.0.0.0/8` | Large planning pool |
| Region hub | `10.10.0.0/16` | Shared services, firewall, gateways |
| Spoke app VNet | `10.11.0.0/16` | App-specific address space |
| Subnet | `10.11.1.0/24` | Typical app subnet |
| Small subnet | `10.11.10.0/27` | Niche platform service needs |

> 💡 **AZ-305 tip:** Overlapping address spaces break VNet peering and hybrid routing designs. If the scenario mentions acquisitions, future regions, or hybrid expansion, choose a design with larger non-overlapping allocation blocks.

### Subnets and subnet delegation

Use subnets for **policy boundaries, route boundaries, and workload grouping**.

> 🔴 **HIGH-YIELD EXAM TOPIC:** Subnet delegation is a frequent exam topic, especially for App Service and Azure Container Apps (ACA). When a subnet is delegated, it becomes **dedicated** to that service — meaning **only one service instance can use the delegated subnet**. This is a key design constraint.

Common subnet patterns:
- `GatewaySubnet` - required for VPN/ExpressRoute gateway.
- `AzureBastionSubnet` - required for Bastion.
- `AzureFirewallSubnet` - required for Azure Firewall.
- `AzureFirewallManagementSubnet` - needed for forced tunneling scenarios.
- `AppGatewaySubnet` - recommended dedicated subnet for Application Gateway.
- `Web`, `App`, `Data`, `Mgmt` - standard tiered application segmentation.

**Subnet delegation** allows a subnet to be dedicated to a specific PaaS service. Key services requiring delegation:

| Service | Delegation type | Key requirement |
|---|---|---|
| **App Service Environment** | `Microsoft.Web/hostingEnvironments` | Requires dedicated subnet; minimum /24 recommended |
| **App Service VNet Integration** | `Microsoft.Web/serverFarms` | **Dedicated subnet required** — one App Service Plan per delegated subnet |
| **Azure Container Apps (ACA)** | `Microsoft.App/environments` | **Dedicated subnet required** — minimum /23 recommended; one ACA environment per subnet |
| **Azure NetApp Files** | `Microsoft.NetApp/volumes` | Dedicated subnet required |
| **Azure SQL Managed Instance** | `Microsoft.Sql/managedInstances` | Dedicated subnet required |

> ⚠️ **Exam trap:** If a question mentions App Service VNet integration or ACA environment, remember that delegation **almost always means dedicated subnet** — you cannot share the delegated subnet with other resources or services.

Use delegation when the platform service needs direct control of subnet operations.

### VNet peering

| Type | Use case | Key facts |
|---|---|---|
| **Regional peering** | Same-region VNets | Low latency, private backbone |
| **Global peering** | Cross-region VNets | Supported across regions; still not transitive |

Key facts:
- Peering is **not transitive**.
- Peered VNets communicate over the **Microsoft backbone**.
- Peering does **not** require gateways.
- Use gateway transit carefully for shared hybrid connectivity.

> 🔴 **HIGH-YIELD EXAM TOPIC:** Understand the **traffic control options** available with VNet peering:
>
> | Setting | What it controls | Default |
> |---|---|---|
> | **Allow virtual network access** | Controls whether resources in the peered VNets can communicate | Enabled |
> | **Allow forwarded traffic** | Allows traffic that originated outside the peered VNet to flow through peering | Disabled |
> | **Allow gateway transit** | Allows the peered VNet to use this VNet's gateway | Disabled |
> | **Use remote gateway** | Allows this VNet to use the peered VNet's gateway | Disabled |
>
> **Exam scenarios:**
> - If spoke VNets need to reach on-prem through a hub's VPN/ExpressRoute gateway → enable **Allow gateway transit** on hub, **Use remote gateway** on spokes
> - If traffic must flow through an NVA or firewall in a hub → enable **Allow forwarded traffic**
> - To completely block traffic between peered VNets → disable **Allow virtual network access**

### VNet-to-VNet connectivity options

> 🎯 **Exam Focus:** For VNet-to-VNet connectivity, realistically you only need to know two options: **VNet Peering** for direct Azure-to-Azure connectivity, and **Virtual WAN** for managed transit at scale. The other options exist but are rarely the exam's preferred answer.

| Option | Best for | Pros | Cons |
|---|---|---|---|
| **VNet peering** | Azure-to-Azure private connectivity | Fast, simple, high bandwidth, low latency | Non-transitive |
| **Virtual WAN** | Large transit design | Managed scale | Less custom than self-built hub |
| **VNet-to-VNet VPN** | When peering is not suitable or encryption over gateway is needed | Encrypted tunnel | Higher latency, gateway cost |
| **ExpressRoute + transit/gateway** | Enterprise hybrid and controlled private routing | Predictable private path | Costlier and more complex |

### Service endpoints vs Private endpoints

| Decision point | Service Endpoint | Private Endpoint |
|---|---|---|
| Service reachable over | Public endpoint of PaaS service | Private IP in your VNet |
| Traffic path | Azure backbone from allowed subnet | Private Link NIC in subnet |
| Public exposure still exists? | **Yes** | Can be disabled entirely |
| DNS changes needed | Usually no | Usually yes (private DNS) |
| Data exfiltration control | Moderate | Stronger |
| On-prem access over private path | Not directly the same way | Yes, with DNS/routing integration |
| Best use | Simple subnet-based restriction | Strong isolation and private-only access |

> ⚠️ **Exam trap — NSGs and Private Endpoints:** By default, NSG rules do **not** apply to private endpoints. To use NSG rules to control traffic to private endpoints, you must explicitly enable **Network Policies for Private Endpoints** on the subnet. Without this configuration, NSG rules will be bypassed for private endpoint traffic.
>
> Enable using CLI: `az network vnet subnet update --disable-private-endpoint-network-policies false`

**Architect rule:** If the exam says **disable public access**, **use private IP**, **private connectivity from on-prem**, or **reduce exfiltration risk**, choose **Private Endpoint**.

### CLI and PowerShell examples

```bash
az group create -n rg-network-prod -l eastus
az network vnet create \
  -g rg-network-prod \
  -n vnet-hub-eastus \
  --address-prefix 10.10.0.0/16 \
  --subnet-name AzureFirewallSubnet \
  --subnet-prefix 10.10.1.0/24

az network vnet subnet create \
  -g rg-network-prod \
  --vnet-name vnet-hub-eastus \
  -n GatewaySubnet \
  --address-prefixes 10.10.2.0/27

az network vnet peering create \
  -g rg-network-prod \
  -n hub-to-spoke1 \
  --vnet-name vnet-hub-eastus \
  --remote-vnet /subscriptions/<subId>/resourceGroups/rg-app/providers/Microsoft.Network/virtualNetworks/vnet-spoke1 \
  --allow-vnet-access
```

```powershell
$rg = "rg-network-prod"
$vnet = New-AzVirtualNetwork -Name "vnet-spoke-app" -ResourceGroupName $rg -Location "EastUS" -AddressPrefix "10.11.0.0/16"
Add-AzVirtualNetworkSubnetConfig -Name "Web" -VirtualNetwork $vnet -AddressPrefix "10.11.1.0/24"
Add-AzVirtualNetworkSubnetConfig -Name "App" -VirtualNetwork $vnet -AddressPrefix "10.11.2.0/24"
$vnet | Set-AzVirtualNetwork

Add-AzVirtualNetworkPeering -Name "spoke-to-hub" `
  -VirtualNetwork $vnet `
  -RemoteVirtualNetworkId "/subscriptions/<subId>/resourceGroups/rg-network-prod/providers/Microsoft.Network/virtualNetworks/vnet-hub-eastus" `
  -AllowVirtualNetworkAccess
```

### Exam-ready design takeaways

- **Peering** for Azure-to-Azure connectivity.
- **Gateway transit** if spokes should use a hub VPN/ExpressRoute gateway.
- **Private Endpoint** for high-security PaaS access.
- **Service Endpoint** for simple subnet-restricted PaaS access when public endpoint can remain enabled.

---

<a id="4-network-security"></a>
## 4. Network Security

### Network Security Groups (NSGs)

NSGs are **stateful packet filters** for subnets and NICs.

> 🎯 **Exam Focus:** Understand NSG rule ordering and Service Tags. Expect troubleshooting questions where you must identify why traffic is blocked or allowed based on rule priority and evaluation order.

#### NSG rules

- Rules have **priority**: lower number wins (lower priority number = higher precedence).
- **Rule ordering matters**: Rules are evaluated in priority order. The first matching rule (lowest number) determines the action. Once matched, no further rules are evaluated.
- Rules specify:
  - source
  - source port
  - destination
  - destination port
  - protocol
  - access (allow/deny)
  - direction (inbound/outbound)
- NSGs are **stateful**: return traffic is automatically allowed for established flows.

#### Service Tags

**Service Tags** are Microsoft-managed prefixes that simplify NSG rules by representing groups of IP addresses for Azure services.

Common service tags to know:
- **Internet** - all public internet IP addresses
- **VirtualNetwork** - VNet address space, peered VNets, on-prem (via gateway)
- **AzureLoadBalancer** - Azure infrastructure load balancer health probes
- **Storage** - Azure Storage IP ranges (can be region-specific: `Storage.EastUS`)
- **Sql** - Azure SQL Database IP ranges
- **AzureActiveDirectory** - Entra ID service endpoints
- **AzureMonitor** - Azure Monitor services

> ⚠️ **Exam trap:** When troubleshooting NSG rules, always check: (1) rule priority ordering, (2) whether the correct service tag is used, (3) whether both subnet and NIC NSGs are evaluated, and (4) direction (inbound vs outbound).

#### Default rules

Inbound defaults include:
- Allow VNet inbound
- Allow Azure Load Balancer inbound
- Deny all inbound

Outbound defaults include:
- Allow VNet outbound
- Allow Internet outbound
- Deny all outbound to VNet? No — this is a common confusion. Default outbound allows VNet and Internet, then denies other unmatched traffic.

#### NSG association: subnet vs NIC

| Association point | Best for | Guidance |
|---|---|---|
| **Subnet** | Broad workload segmentation | Preferred for consistency |
| **NIC** | Exception handling on specific VMs | Use sparingly; complexity rises fast |

> 🔴 **HIGH-YIELD EXAM TOPIC:** Expect multi-subnet VNet scenarios with NSGs attached at both **subnet level and NIC level**. You may see a series of questions on the same configuration asking whether specific traffic would pass.
>
> **NSG Evaluation Order:**
> 1. **Inbound traffic**: Subnet NSG evaluated first → then NIC NSG
> 2. **Outbound traffic**: NIC NSG evaluated first → then Subnet NSG
>
> **Key rules:**
> - Traffic must be allowed by **ALL applicable NSGs** — if either denies, traffic is blocked
> - If no NSG is associated at a level, traffic passes to the next evaluation point
> - Use **Effective Security Rules** in the portal to see the combined result
>
> **Example scenario:** VM has NIC NSG allowing port 443. Subnet NSG denies port 443. **Result: Traffic blocked** (both must allow).

#### NSG flow logs

- Historical traffic logging for NSG-evaluated flows.
- Stored in a Storage Account and can be analyzed with Traffic Analytics.
- Useful for baselining, troubleshooting, and policy refinement.

#### Effective security rules troubleshooting

Use **effective security rules** to understand the combined impact of:
- subnet NSGs
- NIC NSGs
- default rules
- ASGs/service tags where relevant

#### NSG commands

```bash
az network nsg create -g rg-network-prod -n nsg-web-eastus
az network nsg rule create \
  -g rg-network-prod \
  --nsg-name nsg-web-eastus \
  -n allow-https-internet \
  --priority 100 \
  --direction Inbound \
  --access Allow \
  --protocol Tcp \
  --source-address-prefixes Internet \
  --source-port-ranges '*' \
  --destination-address-prefixes '*' \
  --destination-port-ranges 443
```

```powershell
$nsg = New-AzNetworkSecurityGroup -Name "nsg-app-eastus" -ResourceGroupName "rg-network-prod" -Location "EastUS"
Add-AzNetworkSecurityRuleConfig -Name "Allow-App-To-Sql" `
  -NetworkSecurityGroup $nsg `
  -Priority 200 -Direction Outbound -Access Allow -Protocol Tcp `
  -SourceAddressPrefix "10.11.2.0/24" -SourcePortRange * `
  -DestinationAddressPrefix "10.11.3.0/24" -DestinationPortRange 1433 | Set-AzNetworkSecurityGroup
```

### Application Security Groups (ASGs)

ASGs let you group NICs logically and reference the group in NSG rules instead of IP ranges.

> 🎯 **Exam Focus:** ASGs don't come up frequently in practice, but watch for this specific scenario: **if NSG rules target specific IPs but those IPs might change** (auto-scaling, VM replacement, dynamic workloads), **ASGs are the answer**. They provide a stable reference point regardless of IP changes.

#### When to use ASGs

- Dynamic app tiers where IP tracking is painful.
- VM-based workloads in the same VNet.
- Environments with frequent scale changes.
- **Key trigger:** Question mentions "IPs change frequently" or "dynamic scaling" + NSG rules needed.

**Good fit:** `asg-web`, `asg-app`, `asg-db`.  
**Poor fit:** broad cross-VNet abstraction or replacing all network design.

#### ASG example

```bash
az network asg create -g rg-network-prod -n asg-web
az network asg create -g rg-network-prod -n asg-app
```

### Azure Firewall

Azure Firewall is a **managed, stateful, centralized network security service** for hub-based designs.

#### Azure Firewall tiers

| Tier | Best for | Core capabilities |
|---|---|---|
| **Basic** | SMB/simple branch or small workloads | Network rules, application rules, DNAT, basic security |
| **Standard** | Most enterprise workloads | L3-L7 filtering, threat intelligence, FQDN filtering |
| **Premium** | High-security/regulated workloads | IDPS, TLS inspection, URL filtering, web categories |

#### Rule types

| Rule type | Use for |
|---|---|
| **DNAT** | Publish internal service behind public IP |
| **Network rules** | Any protocol/port filtering (L3/L4) |
| **Application rules** | Outbound HTTP/S + FQDN filtering |

#### Premium features

- **IDPS** - signature-based intrusion detection and prevention.
- **TLS inspection** - decrypt/inspect/re-encrypt outbound/inbound TLS flows where designed appropriately.
- **URL filtering/web categories** - more granular web control.

#### Threat intelligence

Threat intelligence-based filtering can alert or deny traffic to known malicious IPs/domains.

#### Firewall Manager and policies

Use **Azure Firewall Manager** and **Firewall Policy** for:
- central rule management
- multiple firewalls
- secure virtual hub integration
- consistent enterprise governance

#### Forced tunneling

Use forced tunneling when Azure workloads must send outbound traffic to:
- on-premises inspection stack
- centralized egress controls
- regulated corporate egress path

#### Azure Firewall vs NVA vs NSG

| Service | Best for | Not a replacement for |
|---|---|---|
| **NSG** | Distributed subnet/NIC filtering | Central L7 inspection |
| **Azure Firewall** | Centralized managed filtering and egress control | Fine-grained app host hardening |
| **NVA** | Specialized partner features or migration constraints | Simplicity/low ops overhead |

**Exam rule:** If you need a **managed central firewall**, prefer **Azure Firewall**. If the scenario says **third-party feature requirement**, **existing vendor standard**, or **special protocol handling**, an **NVA** may be correct.

#### Azure Firewall commands

```bash
az network public-ip create -g rg-network-prod -n pip-afw-eastus --sku Standard
az network firewall create -g rg-network-prod -n afw-hub-eastus -l eastus
az network firewall ip-config create \
  -g rg-network-prod \
  -f afw-hub-eastus \
  -n afw-config \
  --public-ip-address pip-afw-eastus \
  --vnet-name vnet-hub-eastus
```

```powershell
$publicIp = New-AzPublicIpAddress -Name "pip-afw-eastus" -ResourceGroupName "rg-network-prod" -Location "EastUS" -AllocationMethod Static -Sku Standard
$firewall = New-AzFirewall -Name "afw-hub-eastus" -ResourceGroupName "rg-network-prod" -Location "EastUS"
```

### Web Application Firewall (WAF)

WAF protects **HTTP/S applications** from common web exploits.

#### WAF service placements

| WAF option | Best for |
|---|---|
| **Application Gateway WAF** | Regional L7 load balancing + WAF |
| **Front Door WAF** | Global edge protection + global HTTP routing |
| **CDN WAF** | Content delivery edge protection |

#### Key capabilities

- OWASP managed rule sets
- Custom rules
- Bot/rate controls depending on service/features
- **Detection** mode for learning
- **Prevention** mode for blocking

**Architect guidance:** Use **Detection** briefly during tuning, then shift to **Prevention** for production.

### DDoS Protection

| Tier | Scope | Guidance |
|---|---|---|
| **Basic** | Included platform-wide | Baseline only |
| **Standard** | Enhanced protection for VNets | Use for business-critical public-facing services |

Enable **DDoS Protection Standard** when the scenario includes:
- public-facing workloads with revenue impact
- regulated workloads
- strong resiliency requirements
- many public IP resources in a VNet

### Security decision summary

- **NSG** = subnet/NIC traffic control.
- **ASG** = simplify NSG management.
- **Azure Firewall** = central managed filtering.
- **WAF** = HTTP/S attack protection.
- **DDoS Standard** = enhanced volumetric attack protection for public exposure.

---

<a id="5-load-balancing-services"></a>
## 5. Load Balancing Services

### Fast decision tree

```text
Is traffic HTTP or HTTPS?
├── No
│   ├── Need regional L4 balancing? → Azure Load Balancer
│   └── Need global failover/distribution? → Cross-region Load Balancer or combine regional LB with Traffic Manager/Front Door depending on app pattern
└── Yes
    ├── Need global edge routing/acceleration? → Azure Front Door
    ├── Need regional L7 routing into a VNet/app tier? → Application Gateway
    └── Need only DNS-based endpoint selection? → Traffic Manager
```

### Azure Load Balancer (L4)

Use for **TCP/UDP**, not web application intelligence.

#### Key choices

- **Public** vs **Internal**
- **Standard** vs Basic: **always prefer Standard**
- Health probes
- Load balancing rules
- Outbound rules
- HA Ports for all ports load balancing
- Cross-region Load Balancer for global resiliency over regional Standard LBs

#### Why Standard only?

- Better security model
- Zone-aware options
- Production feature set
- Basic SKU retirement/avoidance guidance in modern designs

#### Commands

```bash
az network lb create \
  -g rg-network-prod \
  -n slb-web \
  --sku Standard \
  --public-ip-address pip-lb-web \
  --frontend-ip-name web-fe \
  --backend-pool-name web-be

az network lb probe create \
  -g rg-network-prod \
  --lb-name slb-web \
  -n httpsProbe \
  --protocol Tcp \
  --port 443
```

```powershell
$fe = New-AzLoadBalancerFrontendIpConfig -Name "web-fe" -PublicIpAddress $publicIp
$be = New-AzLoadBalancerBackendAddressPoolConfig -Name "web-be"
$probe = New-AzLoadBalancerProbeConfig -Name "tcp443" -Protocol Tcp -Port 443 -IntervalInSeconds 5 -ProbeCount 2
New-AzLoadBalancer -ResourceGroupName "rg-network-prod" -Name "slb-web" -Location "EastUS" -Sku Standard -FrontendIpConfiguration $fe -BackendAddressPool $be -Probe $probe
```

### Application Gateway (L7)

Use for **regional web traffic** where you need application-aware routing.

#### Key capabilities

- SSL termination and re-encryption
- End-to-end SSL
- URL/path-based routing
- Multi-site hosting
- Cookie affinity if needed
- Autoscaling
- WAF integration
- Private or public frontend

#### When Application Gateway vs Load Balancer

| Need | Choose |
|---|---|
| TCP/UDP only | Load Balancer |
| URL-based routing, host headers, WAF | Application Gateway |
| Web app in-region with layer 7 logic | Application Gateway |

### Azure Front Door

Use for **global HTTP/S load balancing**, edge acceleration, and application delivery.

#### Key capabilities

- Anycast global edge entry point
- Split TCP for performance optimization
- Global WAF integration
- Caching/static acceleration
- Health probing and failover
- Private Link origins for private backend access

#### When Front Door vs Traffic Manager vs App Gateway

| Need | Choose |
|---|---|
| Global HTTP/S acceleration and failover | Front Door |
| DNS-only routing for any protocol | Traffic Manager |
| Regional L7 reverse proxy inside Azure region | Application Gateway |

### Traffic Manager

Traffic Manager is **DNS-based** routing, not a proxy.

> ⚠️ **Exam trap:** Traffic Manager **only works for publicly reachable endpoints**. Applications that are fully internal (private endpoints only) cannot use Traffic Manager because it cannot health-probe private endpoints. For internal-only global routing, consider other patterns like cross-region load balancing or application-level failover.

#### Routing methods

- **Priority** - active/passive failover
- **Weighted** - distribute based on weights
- **Performance** - lowest-latency DNS response
- **Geographic** - user geography-based
- **Multivalue** - return multiple healthy endpoints
- **Subnet** - map client IP subnet to endpoint

#### Health monitoring

Traffic Manager checks endpoint health and changes DNS answers. Existing client DNS caching still applies.

#### Nested profiles

Use nested profiles for complex global architectures, such as regional grouping with local routing logic.

### Load Balancer decision matrix

| Scenario | Service |
|---|---|
| Regional TCP/UDP balancing | **Azure Load Balancer** |
| Regional HTTP/S with path routing and WAF | **Application Gateway** |
| Global HTTP/S acceleration and failover | **Azure Front Door** |
| DNS-based global endpoint selection for any protocol | **Traffic Manager** |

### Architect exam shortcuts

- **L4 regional** → Load Balancer.
- **L7 regional** → Application Gateway.
- **L7 global** → Front Door.
- **DNS-based global** → Traffic Manager.

---

<a id="6-hybrid-connectivity"></a>
## 6. Hybrid Connectivity

### ExpressRoute

ExpressRoute provides **private connectivity** between on-premises environments and Microsoft cloud services via a direct carrier-provided connection. It is the hybrid networking solution for linking on-premises data centers and colocation facilities to Azure.

> 🎯 **Exam Focus:** Remember that ExpressRoute is specifically for hybrid connectivity (on-prem to Azure). It requires a **virtual network gateway** (ExpressRoute type) deployed in Azure in the GatewaySubnet. Don't confuse ExpressRoute with Private Link — they solve different problems.

#### Peering types

> 🔴 **HIGH-YIELD EXAM TOPIC:** Understand the difference between Private peering and Microsoft peering — expect at least one question on this.

| Peering | Use for | Traffic type |
|---|---|---|
| **Private peering** | Access to Azure VNets | Traffic stays entirely on private network; used for VM, database, and internal app access |
| **Microsoft peering** | Access to Microsoft public services (M365, Azure PaaS public endpoints, Dynamics 365) | Traffic goes to Microsoft public services but uses the dedicated ExpressRoute circuit instead of internet |

**Key distinctions:**
- **Private peering** = Your on-prem network connects to your Azure VNets over the ExpressRoute circuit. This is the most common peering type for Azure workloads.
- **Microsoft peering** = Your on-prem network can reach Microsoft 365 and Azure PaaS services (Storage, SQL, etc.) over the ExpressRoute circuit instead of over the public internet. Requires route filters to specify which services to advertise.

> ⚠️ **Exam trap:** Do not confuse **private peering** with Private Link. Private peering is for ExpressRoute circuit connectivity; Private Link is a service-level private access model that provides a private IP in your VNet.

#### ExpressRoute SKUs and features

- Higher SKUs support more scale and add-ons.
- **ExpressRoute Global Reach** connects on-premises sites through Microsoft's network.
- **ExpressRoute Direct** gives direct dual 10/100 Gbps connectivity options for very high throughput needs.
- **FastPath** bypasses gateway data-path processing for improved performance in supported designs.

#### Redundancy design

Best practice:
- Dual provider where possible
- Dual circuits
- Diverse peering locations
- Active-active enterprise design

#### When ExpressRoute vs VPN

| Requirement | Choose |
|---|---|
| High throughput, predictable private path, compliance | **ExpressRoute** |
| Lower cost, faster setup, internet-based encrypted path | **VPN Gateway** |
| Mission-critical hybrid core connectivity | **ExpressRoute** |
| Small branch or temporary connectivity | **VPN** |

### VPN Gateway

#### Common SKUs

Range from **Basic** to **VpnGw5**. Higher SKUs provide more throughput, tunnels, and scale.

#### Connectivity types

- **Site-to-Site (S2S)** - branch/datacenter to Azure.
- **Point-to-Site (P2S)** - user device to Azure.
- **VNet-to-VNet** - encrypted Azure VNet connectivity using VPN gateways.

#### Active-active vs active-passive

| Mode | Benefit | Requirement |
|---|---|---|
| **Active-active** | Higher availability and throughput | Route-based VPN gateway + 2 public IPs |
| **Active-passive** | Simpler | Less resilient |

#### BGP support

> 🎯 **Exam Focus:** BGP's typical use cases are:
> - **Routing in large networks** with many routes or complex topologies
> - **Routing on the internet** (ExpressRoute Microsoft peering, multi-homed connections)
> - **Dynamic route exchange** when routes change frequently
> - **Active-active VPN** configurations require BGP for automatic failover

Use BGP for dynamic route exchange in larger or changing hybrid networks. If a scenario mentions "hundreds of routes," "dynamic failover," or "multi-homed connectivity," BGP is likely part of the answer.

#### IKEv2 vs OpenVPN

| Protocol | Best for |
|---|---|
| **IKEv2** | Device-native VPN where supported |
| **OpenVPN** | Broad client compatibility, internet-restricted environments |

#### Commands

```bash
az network vnet-gateway create \
  -g rg-network-prod \
  -n vpngw-hub-eastus \
  --public-ip-addresses pip-vpngw1 pip-vpngw2 \
  --vnet vnet-hub-eastus \
  --gateway-type Vpn \
  --vpn-type RouteBased \
  --sku VpnGw2 \
  --active-active true
```

```powershell
New-AzVirtualNetworkGateway -Name "vpngw-hub-eastus" `
  -ResourceGroupName "rg-network-prod" `
  -Location "EastUS" `
  -IpConfigurations $gwIpConfigs `
  -GatewayType Vpn -VpnType RouteBased -GatewaySku VpnGw2
```

### Virtual WAN

Virtual WAN is a **managed global hub-and-spoke service** for branch, user, and Azure connectivity at scale. It simplifies multi-region and multi-branch networking by providing Microsoft-managed hubs with integrated routing.

> 🎯 **Exam Focus:** Know that Virtual WAN is the **only "managed" option** for scenarios requiring many branches or managed global transit. Using VPN Gateway alone requires the customer to manage all configuration and routing manually.

#### Best fit

- Many branches
- Many regions
- Need managed transit
- Need integrated VPN, ExpressRoute, Firewall, and routing policy at scale

#### SKUs

| SKU | Best for |
|---|---|
| **Basic** | Limited scenarios, basic branch connectivity |
| **Standard** | Enterprise-scale managed transit with richer features |

#### When Virtual WAN vs traditional hub-spoke

| Scenario | Choose | Notes |
|---|---|---|
| Small number of VNets, custom control | Traditional hub-spoke | Customer manages all configuration and routing |
| Global enterprise with many sites and managed transit need | Virtual WAN | Microsoft manages the hub infrastructure |
| Need many branches with simplified operations | Virtual WAN | **Only "managed" option** — VPN Gateway alone requires manual config |

### Hybrid design shortcuts

- **Enterprise private WAN replacement** → ExpressRoute.
- **Fast, lower-cost branch connectivity** → VPN Gateway.
- **Global branch at scale** → Virtual WAN.
- **Need resilient hybrid core** → ExpressRoute with redundant circuits, optionally plus VPN backup.

---

<a id="7-private-connectivity"></a>
## 7. Private Connectivity

### Private Link & Private Endpoints

**Private Link** is the Azure PaaS service that enables **Private Endpoints** on supported Azure services. Think of Private Link as the underlying platform capability, and Private Endpoints as the specific instances you create in your VNet.

A **Private Endpoint** assigns a private IP from your VNet to an Azure PaaS service via Private Link.

> 🔴 **HIGH-YIELD EXAM TOPIC:** Private Endpoints can **span regions, subscriptions, and even tenants**. This is a powerful capability for enterprise architectures:
> - **Cross-region:** Create a private endpoint in East US for a Storage account in West US
> - **Cross-subscription:** Create a private endpoint in Subscription A for a SQL Database in Subscription B
> - **Cross-tenant:** Create a private endpoint for a service in a partner's tenant (requires approval)
>
> This flexibility enables hub-spoke architectures where a central Private Endpoint serves multiple consuming applications.

#### Use cases

- Storage account private access
- SQL Database private access
- Key Vault private access
- App Service private access
- Private access from peered VNets or on-premises over ExpressRoute/VPN

#### Private Link Service

Use **Private Link Service** to publish your own service privately to consumers.

#### DNS integration

> 🔴 **HIGH-YIELD EXAM TOPIC:** Private Endpoint DNS resolution is a critical topic. Make sure you understand the complete resolution path, including **conditional forwarding** for hybrid scenarios.

Private Endpoints require proper DNS configuration:
- Azure Private DNS zone with the appropriate `privatelink.*` name
- Zone linking to VNets that need resolution
- **Conditional forwarding** for on-premises clients

**DNS Resolution Flow:**
```text
Client query: storageaccount.blob.core.windows.net
      │
      ▼
Public DNS returns CNAME: storageaccount.privatelink.blob.core.windows.net
      │
      ▼
If using Azure Private DNS → Returns private IP (10.x.x.x)
If using on-premises DNS → Must forward to Azure (168.63.129.16) or Azure DNS Private Resolver
```

**Conditional Forwarding Setup for Hybrid:**
1. Create Private DNS zone (e.g., `privatelink.blob.core.windows.net`)
2. Link zone to Azure VNets
3. Configure on-prem DNS to forward `privatelink.*` queries to Azure DNS resolver (inbound endpoint)
4. Deploy **Azure DNS Private Resolver** for bi-directional resolution

> 🎯 **Exam Focus:** Expect questions on DNS for Private Endpoints. You'll use **Azure Private DNS zones** to host the appropriate `privatelink.*` zone name for the service. You need to identify the correct zone name for each service type.

**Common Private DNS zones to know:**
| Service | Private DNS Zone |
|---|---|
| Azure SQL Database | `privatelink.database.windows.net` |
| Azure Blob Storage | `privatelink.blob.core.windows.net` |
| Azure File Storage | `privatelink.file.core.windows.net` |
| Azure Key Vault | `privatelink.vaultcore.azure.net` |
| Azure Cosmos DB (SQL API) | `privatelink.documents.azure.com` |
| Azure App Service / Functions | `privatelink.azurewebsites.net` |
| Azure Event Hubs | `privatelink.servicebus.windows.net` |

**Exam rule:** If Private Endpoint is present, **DNS is usually part of the answer**. Be ready to select the correct `privatelink.*` zone.

#### When Private Endpoint vs Service Endpoint

| Requirement | Choose |
|---|---|
| Disable public endpoint | **Private Endpoint** |
| Need private IP in VNet | **Private Endpoint** |
| Quick subnet-level restriction only | **Service Endpoint** |
| Strongest data exfiltration reduction | **Private Endpoint** |

#### Commands

```bash
az network private-endpoint create \
  -g rg-network-prod \
  -n pe-storage-prod \
  --vnet-name vnet-spoke-app \
  --subnet Data \
  --private-connection-resource-id /subscriptions/<subId>/resourceGroups/rg-data/providers/Microsoft.Storage/storageAccounts/stprod001 \
  --group-id blob \
  --connection-name pe-storage-prod-conn
```

```powershell
$pls = Get-AzPrivateLinkResource -PrivateLinkResourceId "/subscriptions/<subId>/resourceGroups/rg-data/providers/Microsoft.Storage/storageAccounts/stprod001"
New-AzPrivateEndpoint -Name "pe-storage-prod" -ResourceGroupName "rg-network-prod" -Location "EastUS" -Subnet $subnet -PrivateLinkServiceConnection $connection
```

### Service Endpoints

Service Endpoints extend VNet identity to Azure PaaS public services.

#### Best use

- Low-complexity subnet restriction
- No need for private IP or private DNS
- Public endpoint can remain enabled

#### Limitations vs Private Endpoints

- Service still uses public endpoint.
- More limited exfiltration protection.
- Not the best answer when strict private-only access is required.

### Azure Bastion

Azure Bastion provides secure RDP/SSH access **without exposing VM public IPs**.

#### SKU view

| SKU | Best for |
|---|---|
| **Basic** | Core browser-based admin access |
| **Standard** | Larger scale and richer capabilities such as native client support |

#### Why architects choose Bastion

- Removes public IPs from VMs.
- Reduces jump box overhead.
- Supports Zero Trust admin posture.

#### Commands

```bash
az network bastion create \
  -g rg-network-prod \
  -n bastion-hub-eastus \
  --public-ip-address pip-bastion \
  --vnet-name vnet-hub-eastus \
  -l eastus
```

```powershell
New-AzBastion -ResourceGroupName "rg-network-prod" -Name "bastion-hub-eastus" -PublicIpAddressRgName "rg-network-prod" -PublicIpAddressName "pip-bastion" -VirtualNetworkRgName "rg-network-prod" -VirtualNetworkName "vnet-hub-eastus"
```

---

<a id="8-dns"></a>
## 8. DNS

### Azure DNS zones (public)

Use Azure DNS for authoritative public DNS hosting.

Best for:
- internet-facing domains
- highly available managed DNS
- ARM/RBAC-managed DNS lifecycle

### Azure Private DNS zones

Use Azure Private DNS for name resolution inside VNets.

Best for:
- Private Endpoints
- private service discovery
- hub-spoke shared name resolution

### Azure DNS vs Custom DNS on VNets

> 🔴 **HIGH-YIELD EXAM TOPIC:** Understand the implications of using Azure-provided DNS vs custom DNS servers on your VNets.

| DNS Configuration | Implications |
|---|---|
| **Azure-provided DNS (default)** | Automatically resolves Azure Private DNS zones linked to the VNet; no additional configuration needed for Private Endpoints; limited to Azure services |
| **Custom DNS servers** | Must configure forwarders to Azure DNS (168.63.129.16) for Private Endpoint resolution; provides integration with on-prem DNS; more complex setup |

**Key decision points:**
- If you use **custom DNS servers** on your VNet, you **must** configure them to forward Azure private zones to `168.63.129.16` or deploy **Azure DNS Private Resolver**
- Azure-provided DNS cannot resolve on-premises names — you need custom DNS or Azure DNS Private Resolver for hybrid scenarios
- VMs using custom DNS won't automatically resolve Private Endpoint names without proper forwarding

> ⚠️ **Exam trap:** If a question mentions Private Endpoints not resolving, and the VNet uses custom DNS servers, the answer is likely "configure conditional forwarders" or "use Azure DNS Private Resolver."

### DNS forwarding and resolver

Use **Azure DNS Private Resolver** or DNS forwarders when you need:
- Azure-to-on-prem name resolution
- on-prem-to-Azure private zone lookup
- hybrid split-horizon designs

### Split-horizon DNS

> 🔴 **HIGH-YIELD EXAM TOPIC:** Split-horizon (also called split-brain DNS) is a common exam scenario.

Use split-horizon when the same hostname resolves differently:
- public clients → public IP
- internal clients → private IP

**Common scenario:** Your web app `app.contoso.com` should resolve to:
- Public IP (e.g., 20.1.2.3) when accessed from the internet
- Private Endpoint IP (e.g., 10.0.1.5) when accessed from the corporate network

**Implementation:**
1. Public DNS zone: `app.contoso.com` → public IP
2. Azure Private DNS zone linked to VNet: `app.contoso.com` → private IP
3. On-prem DNS forwards to Azure for internal resolution

### DNS-based authorization for managed certificates

> 🔴 **HIGH-YIELD EXAM TOPIC:** Understand how Azure PaaS services use DNS for certificate validation when issuing managed certificates.

**How it works:**
1. You request a managed certificate for your custom domain (e.g., `api.contoso.com`)
2. Azure generates a unique TXT record value
3. You add this TXT record to your DNS zone (e.g., `_dnsauth.api.contoso.com`)
4. Azure validates the record exists with the correct value
5. Only then does Azure issue the certificate

**Services that use DNS-based authorization:**
| Service | DNS Record Required |
|---|---|
| **Azure API Management (APIM)** | TXT record for custom domain certificate |
| **Azure App Service** | TXT record for custom domain verification and managed certificate |
| **Azure Front Door (AFD)** | TXT record for custom domain validation |
| **Azure CDN** | CNAME or TXT record for domain verification |

> ⚠️ **Exam scenario:** If a question asks about custom domain certificates not being issued or failing validation, the answer is likely related to missing or incorrect DNS TXT records.

### Private Endpoint DNS integration

Typical pattern:
1. Create private endpoint.
2. Create or use recommended private DNS zone (for example, `privatelink.blob.core.windows.net`).
3. Link the zone to required VNets.
4. Configure hybrid DNS forwarding if on-prem clients must resolve the private name.

### Commands

```bash
az network private-dns zone create -g rg-network-prod -n privatelink.blob.core.windows.net
az network private-dns link vnet create \
  -g rg-network-prod \
  -n link-spoke-app \
  -z privatelink.blob.core.windows.net \
  -v /subscriptions/<subId>/resourceGroups/rg-network-prod/providers/Microsoft.Network/virtualNetworks/vnet-spoke-app \
  -e false
```

```powershell
New-AzPrivateDnsZone -ResourceGroupName "rg-network-prod" -Name "privatelink.database.windows.net"
New-AzPrivateDnsVirtualNetworkLink -ResourceGroupName "rg-network-prod" -ZoneName "privatelink.database.windows.net" -Name "link-hub" -VirtualNetworkId $hubVnet.Id
```

### Exam-ready DNS heuristics

- Private Endpoint almost always implies **Private DNS**.
- Hybrid name resolution often implies **DNS forwarding/private resolver**.
- Split-horizon DNS is the clean answer when internal and external clients need different answers for the same name.

---

<a id="9-network-monitoring"></a>
## 9. Network Monitoring

> 🔴 **HIGH-YIELD EXAM TOPIC:** Network Watcher is **heavily tested** on the AZ-305 exam. Expect multiple questions on Network Watcher capabilities and which tool to use for specific troubleshooting scenarios. Study this section thoroughly and practice if possible.

### Network Watcher

Network Watcher is the core Azure networking diagnostics service. It provides a suite of tools for monitoring, diagnosing, and gaining insights into your Azure network.

### Key tools and when to use them

| Tool | What it does | When to use | Exam scenario trigger |
|---|---|---|---|
| **Connection troubleshoot** | Tests TCP connectivity between source and destination | Diagnose why VM can't reach a service | "VM cannot connect to..." |
| **IP flow verify** | Checks if a packet is allowed/denied by NSG rules | Identify NSG blocking traffic | "Traffic blocked", "NSG issue" |
| **Next hop** | Shows effective routing decision | Identify UDR or routing issues | "Packets going wrong direction", "routing problem" |
| **NSG flow logs** | Records all allowed/denied flows through NSG | Historical traffic analysis, compliance | "Audit network traffic", "what traffic passed" |
| **Traffic Analytics** | AI-powered analysis of NSG flow logs | Identify traffic patterns, threats, bandwidth hogs | "Analyze traffic patterns", "identify anomalies" |
| **Connection Monitor** | Ongoing monitoring of connectivity health | Continuous monitoring and alerting | "Ongoing monitoring", "alert on connectivity loss" |
| **Packet capture** | Captures packets on VM NIC | Deep troubleshooting, security investigation | "Capture traffic for analysis" |
| **VPN troubleshoot** | Diagnoses VPN Gateway issues | VPN connectivity problems | "VPN tunnel down" |
| **NSG diagnostics** | Detailed NSG rule evaluation | Complex NSG troubleshooting | "Which rule is blocking" |
| **Effective security rules** | Shows combined NSG rules affecting a NIC | Understand aggregate NSG impact | "Multiple NSGs, what's effective" |

### Detailed tool capabilities

#### IP Flow Verify
- Tests a specific 5-tuple (source IP, dest IP, source port, dest port, protocol)
- Returns: Allow or Deny + the specific NSG rule causing the decision
- **Use when:** You need to know if NSG is the problem and which rule

#### Next Hop
- Shows where traffic will be routed for a given destination
- Returns: Next hop type (VNet, Internet, VirtualAppliance, None) and IP
- **Use when:** Traffic is going somewhere unexpected or not arriving

#### Connection Troubleshoot
- Performs an actual connectivity test (not just rule evaluation)
- Can test TCP connectivity, latency, and packet loss
- **Use when:** You need end-to-end connectivity verification

#### Connection Monitor
- Creates persistent monitoring tests
- Supports Azure VMs, on-prem machines (with agent), and Azure endpoints
- Provides continuous health metrics and alerting
- **Use when:** You need ongoing monitoring, not just point-in-time troubleshooting

#### NSG Flow Logs
- Logs metadata about flows through NSGs (not packet contents)
- Version 2 includes throughput information
- Stored in Storage Account, analyzed with Traffic Analytics
- **Use when:** You need historical traffic records or compliance auditing

### Troubleshooting decision tree

```text
Problem: Network connectivity issue
          │
          ├─► Is it NSG-related?
          │   └─► IP Flow Verify → Shows allow/deny and rule name
          │
          ├─► Is traffic going to wrong destination?
          │   └─► Next Hop → Shows routing decision
          │
          ├─► Need to test actual connectivity?
          │   └─► Connection Troubleshoot → TCP test with results
          │
          ├─► Need ongoing monitoring?
          │   └─► Connection Monitor → Continuous health checks
          │
          ├─► Need historical traffic analysis?
          │   └─► NSG Flow Logs + Traffic Analytics
          │
          └─► Need packet-level inspection?
              └─► Packet Capture → Full packet data
```

### Troubleshooting workflow

1. Check **effective NSGs** — see combined rules from subnet and NIC NSGs.
2. Check **IP flow verify** — test if specific traffic is allowed/denied.
3. Check **effective routes / next hop** — verify routing is correct.
4. Check **DNS resolution** — ensure names resolve to expected IPs.
5. Check **firewall/WAF/load balancer health probe** behavior.
6. Check **flow logs / Connection Monitor** — historical or ongoing analysis.

### Exam-ready Network Watcher scenarios

| Scenario | Tool to use |
|---|---|
| "VM in subnet A cannot reach VM in subnet B" | IP Flow Verify → Connection Troubleshoot |
| "Web app traffic not reaching backend through firewall" | Next Hop → verify UDR routes to firewall |
| "Identify which NSG rule is blocking RDP" | IP Flow Verify |
| "Monitor connectivity health between regions" | Connection Monitor |
| "Audit all traffic that passed through subnet NSG last week" | NSG Flow Logs |
| "Identify unusual traffic patterns or potential threats" | Traffic Analytics |
| "VPN tunnel keeps dropping" | VPN Troubleshoot |
| "Traffic to storage account timing out" | Connection Troubleshoot (test TCP 443) |

### Commands

```bash
# IP Flow Verify - Check if traffic is allowed
az network watcher test-ip-flow \
  --direction Inbound \
  --protocol TCP \
  --local 10.11.1.4:* \
  --remote 10.20.1.4:443 \
  --vm vm-web01 \
  --nic nic-web01

# Connection Troubleshoot - Test actual connectivity
az network watcher test-connectivity \
  -g rg-network-prod \
  --source-resource vm-web01 \
  --dest-address sqlprod.database.windows.net \
  --dest-port 1433

# Next Hop - Check routing decision
az network watcher show-next-hop \
  --source-ip 10.11.1.4 \
  --dest-ip 10.20.1.4 \
  --vm vm-web01 \
  --nic nic-web01

# Enable NSG Flow Logs
az network watcher flow-log create \
  --location eastus \
  --nsg nsg-web-eastus \
  --storage-account stflowlogs \
  --enabled true \
  --format JSON \
  --log-version 2
```

```powershell
# IP Flow Verify
Test-AzNetworkWatcherIPFlow -NetworkWatcher $nw -TargetVirtualMachineId $vm.Id `
  -Direction Inbound -Protocol TCP -LocalIPAddress 10.11.1.4 -LocalPort 443 `
  -RemoteIPAddress 10.20.1.4 -RemotePort 0

# Connection Troubleshoot
Test-AzNetworkWatcherConnectivity -NetworkWatcher $nw -SourceId $vm.Id `
  -DestinationAddress "sqlprod.database.windows.net" -DestinationPort 1433

# Get Effective Security Rules
Get-AzEffectiveNetworkSecurityGroup -NetworkInterfaceName "nic-web01" -ResourceGroupName "rg-network-prod"

# Get Effective Route Table
Get-AzEffectiveRouteTable -NetworkInterfaceName "nic-web01" -ResourceGroupName "rg-network-prod"
```

### Monitoring design advice

- Enable diagnostics on **Firewall, Application Gateway, Front Door, VPN Gateway, and Bastion**.
- Send logs to **Log Analytics** for correlation.
- Use alerts for:
  - probe failures
  - tunnel status changes
  - firewall threat intel hits
  - DDoS events
  - Front Door/App Gateway backend health degradation

### Azure Files Networking

> 🔴 **HIGH-YIELD EXAM TOPIC:** Azure Files networking, supported protocols, and required ports appeared on the exam. Make sure you know the protocols and their port requirements.

#### Supported protocols and ports

| Protocol | Port | Use case | Notes |
|---|---|---|---|
| **SMB 3.x** | **TCP 445** | Windows file shares, AD integration | Most common; requires port 445 open |
| **NFS 4.1** | **TCP 2049** | Linux file shares | Premium tier only; no AD integration |
| **SFTP** | **TCP 22** | Secure file transfer (SSH-based) | Requires hierarchical namespace (ADLS Gen2) |
| **REST API** | **TCP 443** | Programmatic access via HTTPS | Standard Azure Storage REST |

#### SMB over QUIC

> 🔴 **HIGH-YIELD EXAM TOPIC:** SMB over QUIC is a newer feature that may appear on the exam.

**What is SMB over QUIC?**
- Tunnels SMB traffic inside **HTTPS (port 443)** using the QUIC protocol
- Enables SMB file access **without requiring port 445** to be open
- Designed for **remote/mobile workers** who need to access Azure Files without VPN

**Why it matters:**
- Many ISPs and corporate networks block port 445 (security risk for older SMB versions)
- SMB over QUIC allows secure file share access from anywhere without VPN
- Uses TLS 1.3 encryption built into QUIC

**Requirements:**
- Azure Files Premium or Standard
- Windows 11 or Windows Server 2022 Datacenter: Azure Edition clients
- Azure Active Directory Kerberos authentication configured

#### Azure Files connectivity options

| Access method | Description | When to use |
|---|---|---|
| **Public endpoint** | Access via `<storage>.file.core.windows.net` | Simple scenarios, firewall rules for security |
| **Private Endpoint** | Private IP in your VNet | Secure access, hybrid via ExpressRoute/VPN |
| **Service Endpoint** | Subnet-level restriction | Restrict to specific VNets, public endpoint remains |
| **SMB over QUIC** | SMB tunneled in HTTPS | Remote users without VPN, blocked port 445 |

#### Port requirements summary

**Know these ports for the exam:**
- **SMB**: TCP 445 (or TCP 443 with SMB over QUIC)
- **NFS**: TCP 2049
- **SFTP**: TCP 22
- **REST/HTTPS**: TCP 443

---

<a id="10-availability-resilience"></a>
## 10. Availability & Resilience

### Zone Redundancy Design Patterns

| Service / pattern | Resilience choice | AZ-305 design takeaway |
|---|---|---|
| **Standard Load Balancer** | Zone-redundant frontend or zonal frontend | Use zone-redundant for shared public/internal entry points when one zone loss must not break access. |
| **Application Gateway v2** | Multi-instance across zones | Strong regional web tier choice when you need WAF and zone-aware scaling. |
| **Azure Firewall** | Availability zones + Firewall Policy | Pair with hub-spoke routing and zone-aware deployment for critical egress paths. |
| **VPN Gateway** | Active-active gateway instances | Default HA answer when hybrid uptime matters. |
| **ExpressRoute** | Dual circuits, dual providers, diverse peering locations | Think circuit redundancy, not just gateway redundancy. |
| **Front Door / Traffic Manager** | Multi-region endpoints | Use when one whole region can fail and the app must remain reachable. |

### Multi-Region Networking Strategies

| Scenario | Recommended pattern | Why it works |
|---|---|---|
| Global web app with active-active regions | **Front Door + regional App Gateway / app backends** | Gives edge routing, health-based failover, and optional WAF at both global and regional layers. |
| Any-protocol service with primary/secondary regions | **Traffic Manager + regional endpoints** | DNS steering works even when the backend is not HTTP/S. |
| Mission-critical hybrid app | **ExpressRoute primary + VPN backup** | Balances private predictable connectivity with a lower-cost failover path. |
| Enterprise hub-spoke per region | **Regional hubs + global peering / Virtual WAN** | Contains blast radius while keeping regional independence. |
| Private PaaS dependency across regions | **Private Endpoints per region + DNS failover design** | Avoids a single-region private endpoint dependency. |

### Resilience Checklist

- Design for **zone failure first**, then decide whether the scenario also requires **regional failover**.
- Prefer **active-active** when the workload has low RTO and can tolerate data replication complexity.
- Remember that **DNS-based failover is not instant** because client caching still applies.
- Keep **DNS, routing, and identity dependencies** aligned with failover plans; a private endpoint without private DNS failover is incomplete.
- For hybrid, the exam often rewards **redundant circuits/gateways/providers** more than a single oversized connection.

---

<a id="11-cost-optimization"></a>
## 11. Cost Optimization

### Bandwidth and Data Transfer Cost Considerations

| Cost driver | Why it matters | Optimization move |
|---|---|---|
| **Internet egress** | Outbound traffic to the internet can dominate cost for public apps and data-heavy workloads. | Use caching, CDN/Front Door, compression, and keep chatty east-west traffic private where possible. |
| **Inter-region traffic** | Cross-region replication, peering, and app chatter can add significant charges. | Minimize unnecessary cross-region sync, colocate chatty tiers, and use async patterns when latency allows. |
| **VNet peering egress/ingress** | Peering is fast but not free at scale, especially across regions. | Centralize shared services carefully and avoid sending high-volume data flows through unnecessary peering hops. |
| **Firewall data processing** | Azure Firewall charges include deployment plus processed traffic. | Send only traffic that truly requires inspection; keep benign east-west flows local when policy allows. |
| **ExpressRoute pricing model** | Metered vs Unlimited data plans affect long-term hybrid cost. | Choose **Unlimited** when egress is consistently high; choose **Metered** for lighter or bursty use. |

### Reserved Capacity and Commitment-Based Savings

| Service | Savings lever | When to use it |
|---|---|---|
| **Azure Firewall** | Reserved capacity | Long-running production firewalls with predictable usage where lower unit cost matters more than short-term flexibility. |
| **ExpressRoute Direct / circuit commitments** | Port capacity commitment and plan selection | High-throughput enterprise connectivity where steady usage justifies committed spend. |
| **Front Door / edge acceleration patterns** | Offload origin traffic with caching | Not reserved capacity in the same way, but a major design-time cost reducer for high-read global apps. |

### Cost-Aware Networking Rules of Thumb

- The cheapest secure design is usually **not** the cheapest raw service; balance **security controls, egress volume, and operational overhead**.
- **Private Endpoint** can reduce exfiltration risk, but it may add DNS and per-endpoint cost — justify it with security and compliance value.
- **Virtual WAN** simplifies operations at scale, but a small environment may be cheaper with a traditional hub-spoke design.
- Right-size premium services: use **Front Door, Firewall Premium, ExpressRoute, and WAF** where the scenario truly requires their differentiated capabilities.

---

<a id="12-az-305-decision-scenarios"></a>
## 12. AZ-305 Decision Scenarios


| Scenario | Best answer | Why | Common wrong answer |
|---|---|---|---|
| **1. Multi-region web application needing global failover and WAF** | **Azure Front Door + regional Application Gateway or app backends** | Global HTTP/S routing, WAF, acceleration, failover | Traffic Manager when app-layer acceleration/inspection is required |
| **2. Hybrid connectivity for mission-critical ERP with predictable latency** | **ExpressRoute with redundant circuits** | Private, resilient, enterprise-grade | VPN only |
| **3. Secure PaaS access with no public exposure** | **Private Endpoint + Private DNS** | Private IP and private-only resolution path | Service Endpoint |
| **4. Enterprise hub-spoke landing zone** | **Hub-spoke with centralized Azure Firewall, shared DNS, optional Bastion** | Centralized control and scalable governance | Flat VNet |
| **5. Zero Trust admin access to Azure VMs** | **Azure Bastion + no public VM IPs + JIT/least privilege** | Removes direct inbound admin ports | Jump box with broad exposure |
| **6. Need to publish multiple websites with path-based routing in one region** | **Application Gateway** | L7 routing and optional WAF | Load Balancer |
| **7. Need to balance non-HTTP SAP or database traffic in-region** | **Standard Load Balancer** | L4 balancing | Application Gateway |
| **8. Need DNS-based failover across endpoints for non-HTTP service** | **Traffic Manager** | Works at DNS layer for any endpoint type | Front Door |
| **9. Need centralized outbound FQDN filtering and threat intelligence** | **Azure Firewall Standard/Premium** | Managed central egress control | NSG alone |
| **10. Need branch connectivity for many global offices with simplified operations** | **Virtual WAN** | Managed transit at scale | Hand-built mesh of gateways |
| **11. Need east-west segmentation between app tiers in a VNet** | **Subnets + NSGs + optionally ASGs** | Right scope for tier segmentation | Azure Firewall only |
| **12. Need protection from volumetric attacks on public workloads** | **DDoS Protection Standard** | Enhanced DDoS mitigation for critical workloads | NSG or WAF alone |
| **13. Need encrypted Azure-to-Azure connectivity when peering is not viable** | **VNet-to-VNet VPN** | Gateway-based encrypted connection | Service Endpoint |
| **14. Need private access from on-prem to Azure SQL over hybrid link** | **Private Endpoint + ExpressRoute/VPN + DNS integration** | Private path plus correct name resolution | Public endpoint with firewall rules |

### Scenario memory hooks

- **Global web** → Front Door.
- **Regional web** → Application Gateway.
- **Regional TCP/UDP** → Load Balancer.
- **Any-protocol DNS steering** → Traffic Manager.
- **Private PaaS** → Private Endpoint.
- **Hybrid enterprise core** → ExpressRoute.
- **Branch/user connectivity at scale** → Virtual WAN.

---

<a id="13-quick-reference-trigger-table"></a>
## 13. Quick Reference Trigger Table

| Trigger phrase in question | Best answer / thought |
|---|---|
| private access to PaaS | Private Endpoint |
| disable public access | Private Endpoint |
| subnet-restricted PaaS access only | Service Endpoint |
| no public IP for VM admin | Azure Bastion |
| centralized outbound filtering | Azure Firewall |
| URL/path-based routing | Application Gateway |
| global HTTP(S) acceleration | Azure Front Door |
| DNS-based failover | Traffic Manager |
| TCP/UDP load balancing | Azure Load Balancer |
| internal-only balancing | Internal Load Balancer |
| public web protection from OWASP threats | WAF |
| volumetric attack mitigation | DDoS Protection Standard |
| many branches worldwide | Virtual WAN |
| private dedicated hybrid circuit | ExpressRoute |
| encrypted internet-based hybrid connection | VPN Gateway |
| active-active VPN | route-based gateway + two public IPs |
| many spoke VNets with shared security | Hub-spoke |
| direct VNet-to-VNet Azure connectivity | VNet peering |
| cross-region VNet connection | Global VNet peering |
| transitive routing required | Hub with gateway/Firewall/Virtual WAN, not peering alone |
| future acquisitions / avoid overlap | Large CIDR planning |
| app tier micro-segmentation | NSG + ASG |
| NIC-level exception | NIC NSG association |
| IPs change frequently / dynamic scaling | ASG for NSG rules |
| inspect effective network path | Next hop / effective routes |
| determine NSG allow/deny result | IP flow verify |
| ongoing connectivity checks | Connection Monitor |
| inspect allowed/denied traffic history | NSG flow logs |
| analyze traffic patterns | Traffic Analytics |
| troubleshoot VM connectivity | Connection Troubleshoot |
| private DNS for PaaS | Azure Private DNS zone |
| same name internal and external different answers | Split-horizon DNS |
| custom DNS on VNet + Private Endpoints not resolving | Conditional forwarder to Azure DNS |
| managed certificate for custom domain | DNS TXT record validation |
| shared hybrid transit for spokes | Gateway transit |
| branch-to-branch via Microsoft backbone | ExpressRoute Global Reach |
| very high-throughput dedicated connectivity | ExpressRoute Direct |
| bypass gateway datapath for ER | FastPath |
| managed enterprise transit networking | Virtual WAN |
| publish your own private service | Private Link Service |
| managed central firewall with IDPS/TLS inspection | Azure Firewall Premium |
| standard SKU only recommendation | Load Balancer / public IP / Firewall best practice |
| HTTP/S global + WAF + caching | Front Door |
| HTTP/S regional + WAF | Application Gateway WAF |
| compare malicious traffic intelligence filtering | Azure Firewall threat intelligence |
| secure remote admin without jump box | Bastion |
| web app private origin behind edge | Front Door + Private Link origins |
| Windows file share access | SMB (port 445) or SMB over QUIC (443) |
| Linux file share | NFS (port 2049) |
| file transfer without VPN / blocked port 445 | SMB over QUIC |
| NSG not applying to Private Endpoint | Enable Network Policies for Private Endpoints |
| App Service or ACA subnet requirement | Dedicated delegated subnet |

---

<a id="14-common-exam-traps"></a>
## 14. Common Exam Traps

### 1. Service Endpoint vs Private Endpoint confusion

- **Service Endpoint** does **not** give the service a private IP in your VNet.
- **Private Endpoint** does.
- If the question says **disable public access**, choose **Private Endpoint**.

### 2. Load Balancer SKU selection

- For modern designs, choose **Standard Load Balancer**, not Basic.
- The same mindset applies to public IPs and production networking design: **Standard-first**.

### 3. ExpressRoute peering types

- **Private peering** = Azure VNets.
- **Microsoft peering** = Microsoft public services.
- Do not mix this up with Private Link.

### 4. Azure Firewall vs NSG scope

- **NSG** = distributed subnet/NIC filter.
- **Azure Firewall** = centralized managed firewall.
- Often the best design uses **both**, not one or the other.

### 5. Traffic Manager vs Front Door

- **Traffic Manager** = DNS-based, any protocol, no proxy.
- **Front Door** = global HTTP/S reverse proxy at the edge.
- If the scenario mentions **SSL offload, WAF, acceleration, caching, anycast**, choose **Front Door**.

### 6. VPN Gateway active-active requirements

- Requires **route-based VPN gateway**.
- Requires **two public IP addresses**.
- Do not choose active-active for a policy-based gateway scenario.

### 7. Peering transitivity assumption

- VNet peering is **not transitive**.
- Hub-spoke transit needs the right architecture: gateway transit, firewall/NVA routing, or Virtual WAN.

### 8. WAF vs DDoS roles

- **WAF** protects against web application attacks.
- **DDoS Standard** protects against volumetric/protocol attacks.
- They solve different problems and can be used together.

### 9. Bastion vs jump box

- If the requirement is secure admin access without public exposure, **Bastion** is usually the cleanest managed answer.

### 10. DNS omitted from Private Endpoint designs

- Private Endpoint without correct DNS is an incomplete design.
- If clients cannot resolve the private name, connectivity fails.

### 11. Subnet delegation means dedicated

- **Delegated subnets are dedicated** — only one service instance per delegated subnet.
- App Service VNet integration and ACA both require dedicated delegated subnets.
- Don't try to share a delegated subnet with other resources.

### 12. NSG doesn't apply to Private Endpoints by default

- By default, NSG rules **do not apply** to Private Endpoint traffic.
- You must enable **Network Policies for Private Endpoints** on the subnet to use NSGs.

### 13. Azure Files port requirements

- **SMB uses port 445** — often blocked by ISPs.
- For blocked port 445 scenarios, consider **SMB over QUIC** (port 443).
- **NFS uses port 2049**; **SFTP uses port 22**.

### 14. Custom DNS breaks Private Endpoint resolution

- If VNet uses custom DNS servers, you **must** configure conditional forwarders.
- Forward `privatelink.*` zones to Azure DNS (168.63.129.16) or use Azure DNS Private Resolver.

---

<a id="15-final-az-305-exam-tips"></a>
## 🎯 Final AZ-305 Exam Tips

1. Start every networking question by identifying the **protocol**: TCP/UDP usually points to Load Balancer, while HTTP/S opens the door to App Gateway or Front Door.
2. Separate **regional** and **global** decisions. App Gateway is regional; Front Door and Traffic Manager are global.
3. If the requirement says **private IP**, **disable public access**, or **minimize exfiltration**, think **Private Endpoint + Private DNS**.
4. Use **NSG** for baseline segmentation and **Azure Firewall** for centralized inspection; many strong designs intentionally use both.
5. Remember that **VNet peering is not transitive**. If transit is required, introduce a hub, gateway transit, firewall routing, or Virtual WAN.
6. When hybrid connectivity must be private and predictable, choose **ExpressRoute**; when speed and cost matter more, choose **VPN Gateway**.
7. **Traffic Manager** answers with DNS; **Front Door** proxies traffic at the edge. That distinction resolves many exam distractors.
8. For resilience, think in layers: **zone redundancy**, **active-active gateways**, **dual circuits/providers**, then **multi-region failover**.
9. Standard SKUs are usually the architecturally correct answer for production networking because they include stronger security and HA capabilities.
10. Cost questions often hide in architecture questions — watch for **egress**, **cross-region traffic**, **inspection charges**, and **reserved capacity opportunities**.

### Preserved Final Architecture Checklist

Use this mental checklist in the exam:

1. **What traffic type is this?** TCP/UDP, HTTP/S, private PaaS, hybrid, admin, east-west?
2. **What scope is needed?** Subnet, VNet, region, global, hybrid?
3. **What security level is required?** NSG, Firewall, WAF, DDoS, Private Link?
4. **What availability target is implied?** zone redundancy, active-active, dual circuits, multi-region?
5. **What DNS/routing dependency exists?** private DNS, split-horizon, UDRs, BGP?
6. **What is the simplest managed service that meets the need?**

---

<a id="16-architecture-decision-flowchart"></a>
## 📐 Architecture Decision Flowchart

```text
Start
  │
  ├─► 1. What is the traffic type?
  │      ├─ HTTP/S ─► Global? ─► Front Door
  │      │              └─ Regional? ─► Application Gateway
  │      ├─ TCP/UDP ─► Regional? ─► Load Balancer
  │      └─ Mixed / branch / private? ─► Continue
  │
  ├─► 2. Is the destination PaaS and should it stay private?
  │      ├─ Yes ─► Private Endpoint + Private DNS
  │      └─ No  ─► Continue
  │
  ├─► 3. Is connectivity hybrid?
  │      ├─ Dedicated/private/compliance ─► ExpressRoute
  │      ├─ Internet-based / fast / lower cost ─► VPN Gateway
  │      └─ Many branches / managed transit ─► Virtual WAN
  │
  ├─► 4. Is centralized inspection required?
  │      ├─ Web threats ─► WAF
  │      ├─ East-west / egress control ─► Azure Firewall + UDRs
  │      └─ Simple segmentation ─► NSG + ASG
  │
  └─► 5. Add resilience and cost choices
         ├─ Zone redundancy / active-active / multi-region
         └─ Optimize egress, peering, reserved capacity, and SKU fit
```

### Senior Architect Summary

- Default to **hub-spoke**, **private-by-default**, **Standard SKUs**, and **centralized inspection where justified**.
- For web workloads: separate **global edge**, **regional L7**, and **regional L4** decisions clearly.
- For PaaS: if security matters, think **Private Endpoint + Private DNS**.
- For hybrid: choose **ExpressRoute** for enterprise private connectivity and **VPN** for cost/speed/flexibility.
- For exams: map the requirement to **scope + protocol + trust boundary** before picking the service.


---

<a id="17-exam-style-review-questions"></a>
## Exam-Style Review Questions

1. A company runs a multi-region e-commerce site and needs a single global web endpoint, WAF protection, and fast failover. Which service should sit at the edge, and why is Traffic Manager alone insufficient?  
   **Answer:** Azure Front Door, because it provides global HTTP/S proxying, WAF, acceleration, and health-based failover; Traffic Manager is DNS-only.

2. A bank must let on-premises apps reach Azure SQL privately and disable all public access. What networking combination best fits the requirement?  
   **Answer:** Private Endpoint + Private DNS + ExpressRoute or VPN connectivity, because the service needs a private IP path and correct private name resolution.

3. A solution architect needs to distribute TCP traffic across VM appliances inside one Azure region. Why is Application Gateway the wrong choice?  
   **Answer:** Application Gateway is Layer 7 and HTTP/S-focused; Azure Load Balancer is the correct Layer 4 regional balancing service.

4. An enterprise with hundreds of branches wants managed transit networking and simplified branch onboarding across regions. Which service best fits, and what tradeoff comes with it?  
   **Answer:** Virtual WAN; the tradeoff is less granular customization than a fully self-managed hub-spoke network.

5. A design requires centralized outbound filtering, threat intelligence, and optional TLS inspection for spoke VNets. Which service tier should you consider first?  
   **Answer:** Azure Firewall Standard or Premium, with Premium preferred when TLS inspection and IDPS are explicit requirements.

---

## Footer

*Azure Networking for AZ-305: choose by protocol, scope, trust boundary, resilience target, and cost profile. Pair this sheet with the Networking Labs for hands-on reinforcement.*

[Back to top](#top)
