> 📝 **Hands-On Labs:** [Networking Labs](./Labs/Azure-Networking-Labs.md)

# Azure Networking Cheat Sheet + AZ-305 Exam Prep

**Audience:** Senior Cloud Solution Architect  
**Primary exam domain:** Design infrastructure solutions (30-35%)  
**Secondary domains:** Identity, governance, and monitoring; business continuity  
**Use this for:** fast revision, architecture decisions, command recall, and scenario-based exam thinking

---

## 1. Azure Networking Overview

Azure networking for AZ-305 is about choosing the **right traffic path, trust boundary, and control plane** for each workload. The exam is less about memorizing every SKU and more about understanding **when to isolate, inspect, publish, encrypt, and scale**.

### Network topology patterns

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

## 2. Virtual Networks (VNet)

### VNet design: address space planning and CIDR

A VNet is the logical isolation boundary for Azure networking. Good VNet design starts with **future growth and hybrid overlap avoidance**.

### Address planning rules

- Use **RFC1918** space unless there is a specific reason not to.
- Avoid overlapping ranges across:
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

Common subnet patterns:
- `GatewaySubnet` - required for VPN/ExpressRoute gateway.
- `AzureBastionSubnet` - required for Bastion.
- `AzureFirewallSubnet` - required for Azure Firewall.
- `AzureFirewallManagementSubnet` - needed for forced tunneling scenarios.
- `AppGatewaySubnet` - recommended dedicated subnet for Application Gateway.
- `Web`, `App`, `Data`, `Mgmt` - standard tiered application segmentation.

**Subnet delegation** allows a subnet to be dedicated to a specific PaaS service, such as:
- Azure App Service Environment
- Azure Container Apps environment scenarios
- Azure NetApp Files
- Azure Database services in delegated VNet modes

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

### VNet-to-VNet connectivity options

| Option | Best for | Pros | Cons |
|---|---|---|---|
| **VNet peering** | Azure-to-Azure private connectivity | Fast, simple, high bandwidth, low latency | Non-transitive |
| **VNet-to-VNet VPN** | When peering is not suitable or encryption over gateway is needed | Encrypted tunnel | Higher latency, gateway cost |
| **ExpressRoute + transit/gateway** | Enterprise hybrid and controlled private routing | Predictable private path | Costlier and more complex |
| **Virtual WAN** | Large transit design | Managed scale | Less custom than self-built hub |

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

## 3. Network Security

### Network Security Groups (NSGs)

NSGs are **stateful packet filters** for subnets and NICs.

#### NSG rules

- Rules have **priority**: lower number wins.
- Rules specify:
  - source
  - source port
  - destination
  - destination port
  - protocol
  - access (allow/deny)
  - direction (inbound/outbound)
- NSGs are **stateful**: return traffic is automatically allowed for established flows.

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

#### When to use ASGs

- Dynamic app tiers where IP tracking is painful.
- VM-based workloads in the same VNet.
- Environments with frequent scale changes.

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

## 4. Load Balancing Decision Tree

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

## 5. Hybrid Connectivity

### ExpressRoute

ExpressRoute provides **private connectivity** between on-premises and Microsoft cloud services.

#### Peering types

| Peering | Use for |
|---|---|
| **Private peering** | VNets in Azure |
| **Microsoft peering** | Microsoft public services such as Microsoft 365/Azure PaaS services where supported/configured |

> ⚠️ **Exam trap:** Do not confuse **private peering** with Private Link. Private peering is for ExpressRoute circuit connectivity; Private Link is a service-level private access model.

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

Use BGP for dynamic route exchange in larger or changing hybrid networks.

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

Virtual WAN is a managed networking service for **branch, user, and Azure connectivity at scale**.

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

| Scenario | Choose |
|---|---|
| Small number of VNets, custom control | Traditional hub-spoke |
| Global enterprise with many sites and managed transit need | Virtual WAN |

### Hybrid design shortcuts

- **Enterprise private WAN replacement** → ExpressRoute.
- **Fast, lower-cost branch connectivity** → VPN Gateway.
- **Global branch at scale** → Virtual WAN.
- **Need resilient hybrid core** → ExpressRoute with redundant circuits, optionally plus VPN backup.

---

## 6. Private Connectivity

### Private Link & Private Endpoints

A **Private Endpoint** assigns a private IP from your VNet to an Azure PaaS service via Private Link.

#### Use cases

- Storage account private access
- SQL Database private access
- Key Vault private access
- App Service private access
- Private access from peered VNets or on-premises over ExpressRoute/VPN

#### Private Link Service

Use **Private Link Service** to publish your own service privately to consumers.

#### DNS integration

Private Endpoints usually require:
- Azure Private DNS zone
- zone linking to VNets
- conditional forwarding/resolver integration for on-premises

**Exam rule:** If Private Endpoint is present, **DNS is usually part of the answer**.

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

## 7. DNS

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

### DNS forwarding and resolver

Use **Azure DNS Private Resolver** or DNS forwarders when you need:
- Azure-to-on-prem name resolution
- on-prem-to-Azure private zone lookup
- hybrid split-horizon designs

### Split-horizon DNS

Use split-horizon when the same hostname resolves differently:
- public clients → public IP
- internal clients → private IP

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

## 8. Network Monitoring

### Network Watcher

Network Watcher is the core Azure networking diagnostics service.

### Key tools

| Tool | Use for |
|---|---|
| **Connection troubleshoot** | Test connectivity between endpoints |
| **IP flow verify** | Check whether NSG allows or denies traffic |
| **Next hop** | See effective routing decision |
| **NSG flow logs** | Record allowed/denied flow metadata |
| **Traffic Analytics** | Analyze NSG flow logs at scale |
| **Connection Monitor** | Ongoing end-to-end connectivity monitoring |

### Troubleshooting workflow

1. Check **effective NSGs**.
2. Check **IP flow verify**.
3. Check **effective routes / next hop**.
4. Check **DNS resolution**.
5. Check **firewall/WAF/load balancer health probe** behavior.
6. Check **flow logs / Connection Monitor**.

### Commands

```bash
az network watcher test-connectivity \
  -g rg-network-prod \
  -n conn-test-web-to-sql \
  --source-resource vm-web01 \
  --dest-address sqlprod.database.windows.net \
  --dest-port 1433

az network watcher show-next-hop \
  -g rg-network-prod \
  -n next-hop-web01 \
  --source-ip 10.11.1.4 \
  --dest-ip 10.20.1.4 \
  --nic nic-web01
```

```powershell
Test-AzNetworkWatcherConnectivity -NetworkWatcher $nw -SourceId $vm.Id -DestinationAddress "sqlprod.database.windows.net" -DestinationPort 1433
Get-AzEffectiveNetworkSecurityGroup -NetworkInterfaceName "nic-web01" -ResourceGroupName "rg-network-prod"
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

---

## 9. AZ-305 Decision Scenarios

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

## 10. Quick Reference Trigger Table

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
| inspect effective network path | Next hop / effective routes |
| determine NSG allow/deny result | IP flow verify |
| ongoing connectivity checks | Connection Monitor |
| inspect allowed/denied traffic history | NSG flow logs |
| private DNS for PaaS | Azure Private DNS zone |
| same name internal and external different answers | Split-horizon DNS |
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

---

## 11. Common Exam Traps

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

---

## Final Architecture Checklist

Use this mental checklist in the exam:

1. **What traffic type is this?** TCP/UDP, HTTP/S, private PaaS, hybrid, admin, east-west?
2. **What scope is needed?** Subnet, VNet, region, global, hybrid?
3. **What security level is required?** NSG, Firewall, WAF, DDoS, Private Link?
4. **What availability target is implied?** zone redundancy, active-active, dual circuits, multi-region?
5. **What DNS/routing dependency exists?** private DNS, split-horizon, UDRs, BGP?
6. **What is the simplest managed service that meets the need?**

---

## Senior Architect Summary

- Default to **hub-spoke**, **private-by-default**, **Standard SKUs**, and **centralized inspection where justified**.
- For web workloads: separate **global edge**, **regional L7**, and **regional L4** decisions clearly.
- For PaaS: if security matters, think **Private Endpoint + Private DNS**.
- For hybrid: choose **ExpressRoute** for enterprise private connectivity and **VPN** for cost/speed/flexibility.
- For exams: map the requirement to **scope + protocol + trust boundary** before picking the service.
