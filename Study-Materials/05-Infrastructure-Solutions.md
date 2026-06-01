# Infrastructure Solutions - SECONDARY PRIORITY ⭐⭐

**Exam Weight:** 30-35% of exam (LARGEST SECTION)  
**Your Performance:** ⚠️ WEAK (short bar)  
**Potential Points:** +5-8

---

## Overview

**Infrastructure** is the largest exam domain (30-35%), but you performed weaker here. This includes:
- Compute selection (covered in 02-App-Architecture)
- Networking (VNets, load balancing)
- Storage (already covered in data storage domain)
- Hybrid connectivity (on-prem to cloud)
- High availability and disaster recovery patterns

This guide focuses on the areas not covered elsewhere.

---

## Networking Fundamentals

### Virtual Networks (VNets)

**What it is:** Isolated network environment for your Azure resources

**Key Components:**

#### Subnets
```
VNet: 10.0.0.0/16 (65,536 addresses)
  ├─ Subnet1: 10.0.1.0/24 (256 addresses, 251 usable)
  ├─ Subnet2: 10.0.2.0/24 (256 addresses)
  └─ Subnet3: 10.0.3.0/24 (256 addresses)
```

**Why subnets matter:**
- Organize resources logically
- Apply different security rules
- Control traffic between subnets
- Better network management

#### Network Security Groups (NSGs)

**What it is:** Stateful firewall for Azure resources

**Rules Structure:**
```
Inbound Rules (incoming traffic):
├─ Priority 100: Allow HTTPS (443) from Internet
├─ Priority 200: Allow RDP (3389) from Corp Network
└─ Priority 300: Deny everything else

Outbound Rules (outgoing traffic):
├─ Priority 100: Allow all to Internet
└─ Default: Deny all (unless allowed)
```

**Applied At:**
- Subnet level (all resources in subnet)
- NIC level (specific VM)

**Exam Scenario:** "Web servers should accept HTTPS from internet, app servers should accept traffic only from web servers, databases should accept only from app servers. Design the NSGs."

**Answer:**
```
Web Server NSG:
- Allow HTTPS (443) from 0.0.0.0/0 (internet)
- Deny everything else

App Server NSG:
- Allow specific port from Web Server subnet IP
- Deny everything else

Database NSG:
- Allow port 1433 from App Server subnet IP
- Deny everything else (including internet)
```

#### Network Interfaces (NICs)

**What it is:** Virtual network adapter for VMs

**Important:** Each VM has ≥1 NIC with:
- Private IP address (10.x.x.x)
- Public IP address (optional, for internet access)
- NSG association

---

### Load Balancing

#### Azure Load Balancer (Layer 4 - TCP/UDP)

**When to Use:**
- High performance, ultra-low latency needed
- Protocol is TCP/UDP
- Extreme scale (millions of connections)

**Use Cases:**
- Gaming servers
- IoT device communication
- Non-HTTP protocols

**Pricing:** Charged per LB + data processed

#### Application Gateway (Layer 7 - HTTP/HTTPS)

**When to Use:**
- HTTP/HTTPS traffic
- URL-based routing (/images → image service, /api → API service)
- Host-based routing (api.example.com vs web.example.com)
- SSL termination
- WAF (Web Application Firewall) needed

**Routing Rules:**
```
Request to /api/users
  ↓
Application Gateway (examines URL path)
  ↓
Routes to Backend Pool: "API Servers"
  ↓
Distribution: Round-robin
  ↓
Response back to client
```

**Common Pattern:**
```
Internet
  ↓
Application Gateway (public IP)
  ├─ WAF (blocks attacks)
  ├─ URL routing
  └─ SSL termination
  ↓
Backend Pools:
├─ Web Server Pool (VMSS)
└─ API Server Pool (VMSS)
```

#### Azure Front Door

**When to Use:**
- Global load balancing across regions
- Content delivery (CDN)
- Failover between regions
- DDoS protection
- SSL/TLS offloading

**Use Case:** E-commerce site needs fast response worldwide + regional failover

```
Global Users
  ↓
Front Door (analyzes location)
  ├─ Route US users to US datacenter
  ├─ Route EU users to EU datacenter
  └─ Route APAC users to APAC datacenter
  ↓
Regional Application Gateways
  ↓
VMs/VMSS in each region
```

---

### Connectivity Options

#### Site-to-Site VPN

**Connects:** On-premises network to Azure VNet

**How it works:**
```
On-premises Network
  ↓ (encrypted tunnel)
VPN Gateway (Azure)
  ↓
Azure VNet
```

**Use Cases:**
- Hybrid environment
- Secure communication with on-prem
- Small to medium bandwidth

**Limitations:**
- Internet-dependent (slower, less reliable)
- Bandwidth limited (~1 Gbps)

#### ExpressRoute

**Connects:** On-premises to Azure via private connection (not internet)

**Advantages over VPN:**
- Private circuit (not over internet)
- Predictable performance (bandwidth guaranteed)
- Lower latency
- Higher bandwidth (up to 100 Gbps)
- Better security

**Typical Setup:**
```
Data Center
  ↓ (direct connection)
ExpressRoute Circuit (private, dedicated)
  ↓
Microsoft Enterprise Edge
  ↓
Azure VNet
```

**Cost:** Higher than VPN, but worth it for large/critical workloads

**Exam Scenario:** "Your company has a 50 Mbps on-prem network and needs to replicate 10 TB of data to Azure daily. Should you use VPN or ExpressRoute?"
- **Answer:** ExpressRoute (higher bandwidth guarantee, better for large data transfers)

---

## High Availability (HA) Patterns

### Pattern 1: Single Region HA

```
VNet
  ├─ Availability Zone 1
  │   └─ VM1 (Web Server)
  ├─ Availability Zone 2
  │   └─ VM2 (Web Server)
  └─ Availability Zone 3
  │   └─ VM3 (Web Server)
      ↓
Load Balancer
  ↓
Distributes traffic across zones
  ↓
If 1 zone fails → 2 zones still serving
```

**SLA: 99.99% (4 nines)**

### Pattern 2: Multi-Region HA (Disaster Recovery)

```
Primary Region (East US)          Secondary Region (West US)
├─ VNet                           ├─ VNet
├─ VMSS (active)                  ├─ VMSS (warm standby)
├─ SQL DB (primary)               ├─ SQL DB (replica)
└─ Storage (LRS)                  └─ Storage (GRS)
        ↓ Traffic Manager          ↓
        └─────────────┬────────────┘
                      ↓
                  Global Users
```

**RTO:** Seconds (automatic failover)  
**RPO:** Near-zero (continuous replication)

---

## Disaster Recovery (DR) Concepts

### RTO vs RPO

| Concept | Meaning | Example |
|---------|---------|---------|
| **RTO** (Recovery Time Objective) | How long until system is back | 1 hour |
| **RPO** (Recovery Point Objective) | How much data loss is acceptable | 15 minutes |

**Implications:**
- If RPO = 15 min, must back up data every 15 min (costs more)
- If RTO = 1 hour, can take 1 hour to recover (slower recovery = cheaper)

### Backup Strategies

| Strategy | Time to Recover | Data Loss | Cost |
|----------|-----------------|-----------|------|
| **Snapshots** | Minutes | Minimal | Medium |
| **Backups (Local)** | Hours | Variable | Low |
| **Geo-Replicated Backups** | Hours | Variable | Medium |
| **Always-On Replica** | Seconds | None | High |

**Exam Scenario:** "Design DR for a business-critical database with RTO=1 hour, RPO=30 minutes"
- **Answer:**
  - Use SQL Always-On (active-passive replica)
  - Or Azure SQL failover groups (automatic failover)
  - Daily snapshots for point-in-time restore
  - Geo-replicate for regional disaster

---

## Common Infrastructure Scenarios

### Scenario 1: Secure 3-Tier Web Application

```
Internet
  ↓
Application Gateway (public subnet)
  ├─ WAF enabled
  ├─ SSL termination
  └─ URL routing
  ↓
NSG: Allow from App Gateway
  ↓
Web Tier (internal subnet)
├─ VMSS with auto-scale
├─ No public IPs
├─ NSG: Allow only from App Gateway
└─ NSG: Allow outbound to App Tier
  ↓
App Tier (internal subnet)
├─ VMSS with auto-scale
├─ No public IPs
├─ NSG: Allow only from Web Tier
└─ NSG: Allow outbound to DB Tier
  ↓
Database Tier (private subnet)
├─ SQL Database (PaaS, simpler)
├─ Backup to geo-replicated storage
├─ NSG: Allow only from App Tier
└─ No internet access
```

**Security Principles Applied:**
- DMZ pattern (App Gateway faces internet)
- Layered security (NSGs at each tier)
- Private IPs except entry point
- No direct internet access to backend

### Scenario 2: Hybrid Connectivity

```
On-Premises
├─ Data Center (10.0.0.0/8)
├─ Users
└─ On-Prem Systems
      ↓
ExpressRoute (private circuit)
      ↓
Azure
├─ VNet (10.100.0.0/16)
├─ Hybrid Runbook Workers
└─ Applications accessing on-prem resources
```

**Use Cases:**
- Extend on-prem network to Azure
- Migration with hybrid period
- Bursting (auto-scale to cloud)
- Disaster recovery (failover to cloud)

### Scenario 3: Global Application

```
Global Users
  ↓
Azure Front Door (global load balancer)
  ├─ Route by location
  ├─ CDN for static content
  └─ DDoS protection
  ↓
  ├─ East US Region
  │   ├─ App Service (active)
  │   ├─ SQL Failover Group (primary)
  │   └─ Redis Cache (session state)
  └─ West Europe Region
      ├─ App Service (standby)
      ├─ SQL Failover Group (replica)
      └─ Redis Cache (replica)
```

**Benefits:**
- Low latency (users connect to nearest region)
- Automatic failover (if region fails)
- Geographic redundancy

---

## Infrastructure Best Practices

### 1. Naming Convention
```
[resource-type]-[environment]-[region]-[number]

Examples:
- vm-prod-eastus-01
- vnet-dev-westus-network
- sql-prod-eastus-db
```

### 2. Tagging Strategy
```
Tags (applied to all resources):
- Environment: prod/staging/dev
- CostCenter: finance/engineering
- Owner: team name
- BackupPolicy: daily/weekly
- SecurityLevel: public/internal/restricted
```

**Why:** Enables cost tracking, compliance, automation

### 3. Capacity Planning
- Estimate growth (3-5 years)
- Choose subnet size accordingly (don't run out of IPs)
- Plan for peak loads (holidays, campaigns)

### 4. Network Segmentation
- Separate by tier (web/app/db)
- Separate by environment (prod/non-prod)
- Separate by security level (public/internal/restricted)

---

## Quick Reference: When to Use Each Service

| Need | Service |
|------|---------|
| Firewall rules | NSG |
| URL-based routing | Application Gateway |
| Global load balancing | Front Door or Traffic Manager |
| On-prem to Azure | ExpressRoute (preferred) or VPN |
| High availability zones | Availability Zones |
| Cross-region redundancy | Traffic Manager + geo-replicated storage |
| Backup with recovery | Azure Backup |
| Point-in-time restore | Snapshots/database restore |

---

## Practice Scenarios

### Question 1: HA Design
Design a web app that must withstand:
- Zone failure (one AZ down)
- Region failure (entire region down)

**Answer:** Availability Zones in primary region (99.99%) + secondary region with Traffic Manager failover (99.99% > 99.95% combined)

### Question 2: Hybrid Network
Connect on-prem office (100 Mbps, critical data) to Azure. What's best?

**Answer:** ExpressRoute (guaranteed bandwidth, private circuit)

### Question 3: Security Architecture
Design network for: public web servers, internal APIs, sensitive databases

**Answer:** DMZ pattern - Application Gateway → Web tier NSG → App tier NSG → Database NSG (each tier isolated)

---

## Study Strategy for Infrastructure

**Priority:** After 01-Governance, 02-App-Architecture, 03-Data-Integration  
**Time:** 1.5-2 hours  
**Hands-On:** 1-2 hours

### Focus Areas (in order):
1. ✅ **VNets, Subnets, NSGs** (foundational)
2. ✅ **Load Balancing** (Application Gateway, Traffic Manager)
3. ✅ **HA/DR Patterns** (zones, regions, failover)
4. ⚠️ **Connectivity** (VPN, ExpressRoute)
5. ⚠️ **Advanced networking** (nice to know)

---

## Key Microsoft Learn Resources

1. **[Design compute solutions](https://learn.microsoft.com/training/modules/compute-solutions/)** - 45 min
2. **[Design network solutions](https://learn.microsoft.com/training/modules/design-network-solutions/)** - 50 min
3. **[Design resilient solutions](https://learn.microsoft.com/training/modules/design-resilient-applications/)** - 45 min
4. **[Hybrid connectivity](https://learn.microsoft.com/training/modules/hybrid-connectivity-azure/)** - 40 min

---

## Next Steps

1. ✅ Complete priority domains first (01-03)
2. **→ Then tackle this Infrastructure domain**
3. **→ Focus on VNets, NSGs, load balancing labs**
4. **→ Practice designing 3-tier architecture**
5. **→ Study HA/DR patterns for scenarios**

**Remember:** You're close to passing! Focus on the top 3 priority domains first, then come back to strengthen infrastructure.** 🚀
