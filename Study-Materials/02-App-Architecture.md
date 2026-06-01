# Design Application Architecture - CRITICAL PRIORITY ⭐⭐⭐

**Exam Weight:** Part of Infrastructure Solutions (30-35%)  
**Your Performance:** ⚠️⚠️ CRITICAL WEAKNESS  
**Potential Points:** +8-12

---

## Overview

**Application architecture** design is about choosing the right Azure compute services and patterns for your workload. You struggled here, which means exam questions tested your ability to match requirements to solutions.

Key challenge: **There's no one "right" answer** - it depends on requirements like:
- Performance requirements (latency, throughput)
- Scalability needs
- Cost constraints
- Operational complexity tolerance
- State management requirements

---

## Azure Compute Services Comparison

### Decision Matrix: Which Compute Service?

| Factor | VMs | VMSS | App Service | AKS | Functions | Logic Apps |
|--------|-----|------|-------------|-----|-----------|-----------|
| **Control** | Full | Full | Medium | Full | Low | Very Low |
| **Scaling** | Manual | Auto | Auto | Auto | Auto | Auto |
| **Cost** | High | High | Medium | High | Pay/use | Pay/use |
| **Ops Overhead** | High | Medium | Low | High | Very Low | Very Low |
| **Containerization** | No | No | Yes (optional) | Yes (required) | No | No |
| **State Management** | Local | Stateless only | Session state | Stateless preferred | Stateless | Stateless |

---

## 1. Virtual Machines (VMs)

**Use When:**
- Need full OS/application control
- Running legacy applications
- Custom middleware/drivers required
- Complex networking requirements

**Advantages:**
- Maximum control over OS and applications
- Support for any workload type
- Strong performance guarantees with Premium storage

**Disadvantages:**
- Highest operational overhead
- Manual scaling required (or VMSS)
- Higher cost per unit
- Patching and maintenance responsibility

**Exam Scenario:** "Your company has a legacy Windows application that requires specific registry entries and custom drivers. This app must scale to handle 10x traffic growth during peak season. What's the best approach?"
- **Answer:** Use VMSS with Windows VMs and auto-scale rules based on CPU/memory metrics

**Key Concepts:**
- **Availability Sets** - Protect against single points of failure (different fault/update domains)
- **Zones** - Distribute across geographic zones within a region for higher availability
- **VMSS (Virtual Machine Scale Sets)** - Auto-scaling group of identical VMs

**Hands-On Task:**
```
1. Create a Windows VM
2. Set up auto-scale with custom metrics
3. Deploy an app and test scaling
4. Configure availability zones
```

---

## 2. App Service

**Use When:**
- Web apps, APIs, mobile backends
- Need platform-managed scaling
- Want to minimize operations
- Building .NET, Node, Python, Java, PHP apps

**Advantages:**
- Simple deployment (zip/git push)
- Automatic scaling to handle load
- Built-in SSL/TLS
- Integrated monitoring and diagnostics
- Deployment slots for zero-downtime updates

**Disadvantages:**
- Less control than VMs
- Platform constraints (limited OS customization)
- Shared infrastructure (Standard tier and below)

**Pricing Tiers (Important!):**
| Tier | Scaling | Auto-scale | Use Case |
|------|---------|-----------|----------|
| **Free/Shared** | Manual | No | Dev/test only |
| **Basic** | Manual | No | Dev/test, small workloads |
| **Standard** | Manual | YES | Production web apps |
| **Premium** | Manual | YES | Enterprise, better performance |
| **Isolated** | Manual | YES | High security, compliance |

**Key Decision:** "My app needs auto-scaling" → Minimum **Standard** tier

**Exam Scenario:** "Your company has 5 internal ASP.NET applications. They need auto-scaling during business hours and low cost. What's the best approach?"
- **Answer:** App Service Plan with Standard tier, enable auto-scale rules based on schedule/CPU

**Advanced Features:**
- **Deployment Slots** - Run multiple versions simultaneously
- **Application Gateway integration** - Load balancing and routing
- **Managed Identity** - Secure authentication to Azure services

**Hands-On Task:**
```
1. Deploy web app from GitHub
2. Enable auto-scale rules
3. Set up staging slot
4. Configure custom domain
```

---

## 3. Azure Kubernetes Service (AKS)

**Use When:**
- Microservices architecture
- Container orchestration needed
- High scalability and resilience required
- Team has Kubernetes expertise

**Advantages:**
- Enterprise-grade container orchestration
- Auto-scaling of pods and nodes
- Multi-environment support (canary, blue-green deployment)
- Service mesh integration (Istio, Linkerd)
- Managed Kubernetes (Microsoft handles control plane)

**Disadvantages:**
- Complex to set up and operate
- Requires container knowledge (Docker, Kubernetes)
- Higher resource costs
- Steep learning curve

**Key Concepts:**
- **Pods** - Smallest deployable unit (usually one container)
- **Deployments** - Desired state (replicas, image version)
- **Services** - Network access to pods (ClusterIP, LoadBalancer, NodePort)
- **Ingress** - URL routing and SSL termination
- **Namespaces** - Logical isolation within cluster

**Scaling in AKS:**
- **Horizontal Pod Autoscaling (HPA)** - Scale pods based on metrics
- **Vertical Pod Autoscaling (VPA)** - Right-size container resources
- **Cluster Autoscaler** - Scale nodes up/down based on pod needs

**Exam Scenario:** "Your company has a microservices application with 15 services. Each service must scale independently. You need fast deployments with zero downtime. What's your architecture?"
- **Answer:** AKS cluster with horizontal pod autoscaling, use deployments for each microservice, implement service mesh for traffic management, use ingress for routing

**Hands-On Task:**
```
1. Create AKS cluster
2. Deploy multi-container application
3. Set up horizontal pod autoscaling
4. Implement service mesh (Istio)
5. Do blue-green deployment
```

---

## 4. Azure Functions

**Use When:**
- Event-driven workloads
- Short-running tasks (<15 minutes)
- Pay-per-execution pricing acceptable
- Minimal operational overhead needed

**Advantages:**
- Serverless (no infrastructure to manage)
- Pay only for execution time
- Auto-scaling is transparent
- Event-triggered (HTTP, timer, blob, queue, etc.)

**Disadvantages:**
- Limited execution time (15 minutes default)
- Stateless only
- Cold start latency
- Vendor lock-in

**Triggers Available:**
- HTTP
- Timer (schedule)
- Blob storage
- Queue storage
- Cosmos DB
- Event Grid
- Event Hub
- Service Bus

**Exam Scenario:** "You need to process images uploaded to blob storage: resize, convert to thumbnail, extract metadata, and store in database. This should scale automatically."
- **Answer:** Use Azure Functions with blob trigger, implement each processing step as a function, use queue for reliable messaging between steps

**Hands-On Task:**
```
1. Create HTTP-triggered function
2. Create blob-triggered function
3. Set up function chaining with queues
4. Monitor performance and costs
```

---

## 5. Logic Apps (Workflow Automation)

**Use When:**
- Workflow orchestration (not computation)
- Integrating SaaS/enterprise applications
- Visual workflow needed for non-developers
- Minimal custom code required

**Advantages:**
- Visual workflow design
- 500+ built-in connectors
- Serverless and auto-scaling
- Great for integration scenarios

**Disadvantages:**
- Not for compute-heavy workloads
- Limited custom code capability
- Can get expensive for complex workflows

---

## Architectural Patterns

### Pattern 1: N-Tier Architecture

```
┌─────────────────┐
│   Presentation  │ (App Service, VMs)
│     (Web UI)    │
└────────┬────────┘
         │
┌────────▼────────┐
│   Application   │ (App Service, VMs, AKS)
│     (API)       │
└────────┬────────┘
         │
┌────────▼────────┐
│   Data Layer    │ (SQL Database, Cosmos DB)
│   (Database)    │
└─────────────────┘
```

**When to use:** Traditional business applications  
**Scaling strategy:** Each tier scales independently  
**Database:** Usually relational (SQL)

### Pattern 2: Microservices Architecture

```
┌──────────┐  ┌──────────┐  ┌──────────┐
│ Service1 │  │ Service2 │  │ Service3 │
│  (AKS)   │  │  (AKS)   │  │ (AKS)    │
└────┬─────┘  └────┬─────┘  └────┬─────┘
     │             │             │
     └─────────────┴─────────────┘
              │
         ┌────▼─────┐
         │  API     │
         │ Gateway  │
         └──────────┘
```

**When to use:** Large, complex applications with independent teams  
**Scaling:** Each service scales based on its own load  
**Communication:** Usually async (queues) or gRPC  
**Data:** Database per service (not shared)  
**Deployment:** Independent, frequent deployments

### Pattern 3: Event-Driven Architecture

```
Event Source (IoT, blob, etc.)
         │
         ▼
    ┌─────────┐
    │Functions│ (Processes event)
    └────┬────┘
         │
    ┌────▼──────────┐
    │Queue/Topic    │ (Decouples processing)
    └────┬──────────┘
         │
    ┌────▼──────┐
    │Database   │ (Stores result)
    └───────────┘
```

**When to use:** Real-time data processing, notifications, async work  
**Tools:** Azure Functions, Event Grid, Service Bus, Event Hub

### Pattern 4: Lambda/Serverless

```
API Gateway (App Service, Functions)
         │
         ▼
    ┌─────────────┐
    │ Functions   │ (Auto-scales)
    │ (Stateless) │
    └────┬────────┘
         │
    ┌────▼─────────────┐
    │ Managed Services │
    │ (DB, Storage,    │
    │  Queues)         │
    └──────────────────┘
```

**When to use:** Variable workloads, rapid prototyping, cost sensitivity  
**Scaling:** Automatic, often very fast  
**Cost:** Pay-per-use  

---

## Making the Right Choice: Decision Flow

```
START
  │
  ├─ Needs full OS control?
  │  YES → Use VMs (or VMSS if scaling needed)
  │  NO → Continue
  │
  ├─ Containerized application?
  │  YES → AKS or App Service containers
  │  NO → Continue
  │
  ├─ Traditional web/API app?
  │  YES → App Service
  │  NO → Continue
  │
  ├─ Event-driven or short tasks?
  │  YES → Azure Functions
  │  NO → Continue
  │
  ├─ Workflow orchestration?
  │  YES → Logic Apps
  │  NO → Reconsider requirements
```

---

## Scaling Strategies

### Auto-Scaling Rules (VMs, App Service)

**Based on metrics:**
```
CPU > 70% for 5 minutes → Scale OUT (add instance)
CPU < 30% for 10 minutes → Scale IN (remove instance)
```

**Based on schedule:**
```
Monday-Friday 9 AM - 5 PM → 10 instances
Otherwise → 2 instances (saves cost)
```

**Important:** Set min/max limits to control cost

### Container Scaling (AKS)

**Horizontal Pod Autoscaling:**
```
If CPU > 80% → Scale pods horizontally
```

**Cluster Autoscaling:**
```
If pods can't be scheduled (no nodes) → Add nodes
If node utilization < 50% → Remove nodes
```

---

## Practice Scenarios

### Scenario 1: E-Commerce Platform
**Requirements:**
- High-traffic web frontend
- REST API backend with multiple services
- Real-time inventory updates
- Black Friday: 100x traffic surge

**Questions:**
1. Compute service for web frontend?
2. Compute service for backend?
3. How to handle inventory updates?
4. How to ensure 100x scaling?

**Sample Answer:**
- Frontend: App Service Standard tier with auto-scale
- Backend: AKS with microservices (inventory, orders, payments)
- Inventory: Event-driven using Service Bus + Functions
- Scaling: HPA in AKS, scheduled auto-scale for Black Friday, CDN for static content

### Scenario 2: Internal HR Management System
**Requirements:**
- 500 employees
- Predictable 9-5 usage
- Must be highly available
- Budget-conscious
- Legacy monolithic .NET app

**Questions:**
1. What compute service?
2. How to minimize cost?
3. How to ensure high availability?

**Sample Answer:**
- App Service Standard tier (managed platform)
- Enable auto-scale with schedule (ramp up before 9 AM, down at 5 PM)
- Availability zones for HA
- Use deployment slots for zero-downtime updates

### Scenario 3: IoT Real-Time Analytics
**Requirements:**
- 10,000 IoT devices sending data every minute
- Real-time dashboard updates
- Store data for 5 years
- Variable load throughout day

**Questions:**
1. How to ingest 10,000 messages/min?
2. Real-time processing?
3. Long-term storage?

**Sample Answer:**
- Event Hub for ingestion (partition per device)
- Azure Functions for real-time processing
- Stream Analytics for real-time aggregation
- Data Lake for long-term storage (archive to cold storage after 90 days)

---

## Common Architecture Mistakes

❌ **Mistake 1:** Choosing AKS for a simple CRUD app  
✅ **Fix:** Use App Service - simpler, cheaper, managed

❌ **Mistake 2:** Assuming all services scale well together  
✅ **Fix:** Design each tier to scale independently, plan for bottlenecks

❌ **Mistake 3:** Not considering operational overhead  
✅ **Fix:** Simpler platforms = lower operational cost (even if slightly higher resource cost)

❌ **Mistake 4:** Tight coupling between services  
✅ **Fix:** Use async communication (queues) for loose coupling

❌ **Mistake 5:** Ignoring stateless design  
✅ **Fix:** Use managed state (databases, caches) instead of session state

---

## Exam Tips for Application Architecture

1. **Understand trade-offs:** Cost vs control vs operational overhead
2. **Ask "what are we optimizing for?"** - If scale, choose auto-scaling services; if cost, choose serverless; if control, choose VMs
3. **Pattern recognition:** Recognize when to apply microservices, event-driven, n-tier patterns
4. **Scaling strategies:** Auto-scale rules, scheduled scaling, burst capacity
5. **Always consider:** HA, DR, monitoring, security for each choice

---

## Quick Reference: Compute Service Selection

**Simple internal app? Low traffic?** → App Service Free/Basic  
**High-traffic web app?** → App Service Standard + auto-scale  
**Needs full OS control?** → VM or VMSS  
**Container workload?** → AKS or App Service containers  
**Microservices?** → AKS  
**Event-driven workload?** → Azure Functions  
**Workflow integration?** → Logic Apps  

---

## Key Microsoft Learn Resources

1. **[Choose the right compute service](https://learn.microsoft.com/training/modules/choose-compute-service-azure/)** - 45 min
2. **[Design scalable applications](https://learn.microsoft.com/training/modules/app-service-plan-azure-apps/)** - 40 min
3. **[Microservices architecture with AKS](https://learn.microsoft.com/training/paths/azure-kubernetes-service-microservices/)** - 2 hours
4. **[Azure Functions fundamentals](https://learn.microsoft.com/training/modules/azure-functions-core-components/)** - 30 min

**Total Study Time:** 2-3 hours  
**Hands-On Labs:** 2 hours

---

## Next Steps

1. ✅ Read this guide
2. **→ Take the decision matrix test (pick service for 5 scenarios)**
3. **→ Complete Microsoft Learn modules**
4. **→ Build one app on each platform (App Service, AKS, Functions)**
5. **→ Practice exam questions on architecture choices**

**Remember:** No single "right" answer - it's about matching service to requirements. 🎯
