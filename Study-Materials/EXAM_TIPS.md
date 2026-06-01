# AZ-305 Exam Tips & Quick Reference Cheat Sheet

---

## 🧠 How to Approach Exam Questions

### AZ-305 Question Format
**Most questions:** "Given this scenario, what's the best solution?"

**Characteristics:**
- Scenario-focused (not memorization)
- Multiple technically correct answers - but one is BEST
- Requires trade-off analysis
- Tests architecture thinking, not just facts

---

## 🎯 Question Analysis Framework

### For Every Question, Ask:

**1. What's the PRIMARY constraint?**
```
Cost? → Choose cheapest option
Performance? → Choose fastest option
Compliance? → Choose secure/auditable option
Scalability? → Choose auto-scaling service
Operational complexity? → Choose managed service
```

**2. What are the THREE main decision factors?**
```
E.g., for compute:
- Performance requirements (SLA)
- Scaling needs (predictable vs variable load)
- Operational overhead (do we manage OS or not)
```

**3. Eliminate obviously wrong answers**
```
❌ VMs for a simple web app (overkill)
❌ Functions for 24/7 continuous processing (wrong use case)
❌ On-prem VPN for guaranteed bandwidth (wrong - use ExpressRoute)
```

**4. Compare the BEST two remaining answers**
```
Which one better meets the PRIMARY requirement?
- If cost focused → go cheaper
- If performance focused → go faster
- If hybrid → balance both
```

---

## 📋 Quick Decision Trees

### "Which Compute Service?"

```
START
  │
  ├─ Need full OS control?
  │  ├─ YES → VMs (or VMSS if scaling needed)
  │  └─ NO → Continue
  │
  ├─ Containerized app?
  │  ├─ YES → AKS or App Service containers
  │  └─ NO → Continue
  │
  ├─ Web/API application?
  │  ├─ YES → App Service
  │  └─ NO → Continue
  │
  ├─ Event-driven/short-running tasks?
  │  ├─ YES → Azure Functions
  │  └─ NO → Continue
  │
  └─ Workflow orchestration?
     ├─ YES → Logic Apps
     └─ NO → Reconsider requirements
```

### "VMs vs App Service vs AKS?"

```
App Service = Platform service, managed scaling, simpler ops
  ├─ CHOOSE IF: Traditional web app, predictable workload, want less ops
  └─ AVOID IF: Need full OS control, complex middleware

VMs = Maximum control, manual scaling, more ops overhead
  ├─ CHOOSE IF: Legacy app, custom middleware, specific OS needed
  └─ AVOID IF: Simple web app (use App Service instead)

AKS = Container orchestration, complex ops, very scalable
  ├─ CHOOSE IF: Microservices, need fine-grained scaling, team knows K8s
  └─ AVOID IF: Simple monolithic app (use App Service instead)
```

### "Load Balancer vs Application Gateway vs Front Door?"

```
Load Balancer = Layer 4, ultra-high performance, TCP/UDP
  ├─ CHOOSE IF: Non-HTTP protocol or extreme scale
  └─ EXAM SCORE: Ask yourself if you need HTTP routing

Application Gateway = Layer 7, URL routing, WAF, HTTP/HTTPS
  ├─ CHOOSE IF: URL-based routing, need WAF, HTTP/HTTPS apps
  └─ MOST COMMON: This is usually the answer for web apps

Front Door = Global load balancing, CDN, multi-region
  ├─ CHOOSE IF: Global users, need regional failover, CDN
  └─ EXAM SCORE: Ask yourself if you need global presence
```

### "ETL vs ELT?"

```
On-Premises or legacy → ETL
  └─ Transform before loading (limited cloud resources)

Cloud-native, big data → ELT
  └─ Load raw, transform in cloud (cheaper, faster)
```

### "SQL Database vs Cosmos DB?"

```
Structured data, ACID required → SQL Database
  └─ Relational, strong consistency, transactions

Unstructured/semi-structured, geo-distributed, high velocity → Cosmos DB
  └─ NoSQL, eventual consistency, horizontal scaling
```

### "Management Group, Subscription, or Resource Group?"

```
Apply policy across many subscriptions? → Management Group
  └─ Governance at scale

Billing boundary? → Subscription
  └─ Cost tracking, separate bills

Logical resource grouping? → Resource Group
  └─ All resources for one app
```

---

## 🔴 Common Exam Traps (Avoid These!)

### Trap 1: "All three options could work"
**Your brain:** "All these services could technically do it..."  
**Reality:** One is CLEARLY better based on the scenario  
**Fix:** Identify the PRIMARY constraint, eliminate wrong answers

### Trap 2: Choosing the most complex solution
**Your brain:** "The enterprise solution sounds impressive"  
**Reality:** AZ-305 rewards SIMPLE solutions that meet requirements  
**Fix:** Choose the simplest solution that solves the problem

### Trap 3: Ignoring operational overhead
**Your brain:** "That service has all the features!"  
**Reality:** If your team can't operate it, it's the wrong choice  
**Fix:** Consider "Can we support this?" for every answer

### Trap 4: Assuming "on-premises" = VPN
**Your brain:** "On-prem to Azure = Site-to-Site VPN"  
**Reality:** ExpressRoute might be better for large/critical workloads  
**Fix:** Consider bandwidth, reliability, cost for each scenario

### Trap 5: Forgetting about cost
**Your brain:** "This solution is technically best"  
**Reality:** Cost might be a hidden constraint  
**Fix:** If scenario doesn't mention cost, it might not be a factor

---

## ✅ Governance Exam Tips

### If the question mentions "organizational standards" or "compliance":
✅ **Think:** Policy, Management Groups, RBAC  
✅ **Ask:** How do we enforce this at scale?  
✅ **Answer:** Usually involves Azure Policy + Management Groups

### If the question mentions "Who can do what?":
✅ **Think:** RBAC  
✅ **Ask:** What's the minimum permission needed?  
✅ **Answer:** Assign specific role at right scope (least privilege)

### If the question mentions "cost tracking by department":
✅ **Think:** Tags + Cost Analysis  
✅ **Ask:** How do we group costs?  
✅ **Answer:** Tag all resources, group analysis by tags

### If the question mentions "prevent resources without encryption":
✅ **Think:** Azure Policy with Deny effect  
✅ **Ask:** How do we enforce this automatically?  
✅ **Answer:** Policy with Deny effect or DeployIfNotExists

---

## 🏗️ Architecture Exam Tips

### If the question mentions "auto-scaling" or "handles 10x load":
✅ **Think:** VMSS, App Service Standard tier, AKS  
✅ **Ask:** How does it scale?  
✅ **Answer:** Choose service with native auto-scaling

### If the question mentions "millions of concurrent connections":
✅ **Think:** Scale at multiple levels  
✅ **Ask:** Where's the bottleneck?  
✅ **Answer:** CDN + Load Balancer + VMSS, or App Service + databases

### If the question mentions "zero downtime deployment":
✅ **Think:** Deployment slots, blue-green, rolling updates  
✅ **Ask:** How do we update without downtime?  
✅ **Answer:** Use native deployment mechanisms (slots or rolling)

### If the question mentions "microservices" or "independent scaling":
✅ **Think:** AKS with HPA  
✅ **Ask:** Does each service scale independently?  
✅ **Answer:** AKS, not VMSS (VMSS scales all instances together)

---

## 📊 Data Integration Exam Tips

### If the question mentions "real-time analytics" or "streaming":
✅ **Think:** Event Hub, Stream Analytics, or Spark Streaming  
✅ **Ask:** How fast must results appear?  
✅ **Answer:** Streaming architecture (not batch)

### If the question mentions "daily warehouse load" or "batch processing":
✅ **Think:** ADF, Synapse, or Data Factory  
✅ **Ask:** How often does data load?  
✅ **Answer:** Batch architecture (simpler, cheaper)

### If the question mentions "legacy SSIS jobs to Azure":
✅ **Think:** ADF (Azure's modern replacement for SSIS)  
✅ **Ask:** How do we migrate workflows?  
✅ **Answer:** ADF pipelines

### If the question mentions "data discovery" or "exploratory queries on 100GB+":
✅ **Think:** Data Lake + Synapse SQL or Spark  
✅ **Ask:** Can we query directly from storage?  
✅ **Answer:** Data Lake Gen 2 + analytics service

---

## 🔐 Identity & Monitoring Exam Tips

### If the question mentions "prevents compromised accounts":
✅ **Think:** Conditional Access + Risk detection  
✅ **Ask:** How do we detect and prevent threats?  
✅ **Answer:** Conditional Access with user risk policy

### If the question mentions "authenticate without passwords":
✅ **Think:** Passwordless (Windows Hello, FIDO2)  
✅ **Ask:** What's more secure than password?  
✅ **Answer:** Passwordless authentication options

### If the question mentions "troubleshoot performance issues":
✅ **Think:** Application Insights + Log Analytics  
✅ **Ask:** Where's the bottleneck (app or infrastructure)?  
✅ **Answer:** Use Application Insights for app-level, Azure Monitor for infra-level

### If the question mentions "compliance audit trail":
✅ **Think:** Activity Log + Log Analytics  
✅ **Ask:** Can we query who did what when?  
✅ **Answer:** Log Analytics for historical audit

---

## 🌐 Infrastructure Exam Tips

### If the question mentions "internet-facing application":
✅ **Think:** Application Gateway (WAF) or Front Door  
✅ **Ask:** Does it need global presence?  
✅ **Answer:** Application Gateway if regional, Front Door if global

### If the question mentions "network isolation" or "security":
✅ **Think:** NSGs, subnets, private endpoints  
✅ **Ask:** How do we restrict traffic?  
✅ **Answer:** Design layered security with NSGs at each tier

### If the question mentions "on-premises to Azure with guaranteed bandwidth":
✅ **Think:** ExpressRoute  
✅ **Ask:** Is internet reliability acceptable?  
✅ **Answer:** ExpressRoute for guaranteed performance

### If the question mentions "survive region failure":
✅ **Think:** Multi-region, geo-replication, Traffic Manager  
✅ **Ask:** RTO/RPO requirements?  
✅ **Answer:** Replicate to secondary region with automated failover

---

## 🎬 During the Exam

### Time Management
- **Total time:** 120 minutes for ~50 questions
- **Per question:** ~2-2.5 minutes
- **Strategy:** Don't get stuck, flag hard questions, review at end

### For Each Question

**Step 1 (30 seconds):** Read scenario carefully
- Underline key requirements
- Identify constraints (cost, performance, compliance)

**Step 2 (30 seconds):** Eliminate obviously wrong answers
- If it violates a requirement, eliminate it
- If it's overkill, question why

**Step 3 (60 seconds):** Compare remaining 2-3 answers
- Which best meets PRIMARY requirement?
- Which is simplest that solves problem?
- Which has least operational overhead?

**Step 4 (if stuck):** 
- Make best educated guess
- Flag for review later
- Move on (don't waste time)

---

## 📝 Pre-Exam Checklist (Night Before)

- [ ] Get 8+ hours of sleep
- [ ] Review quick reference cheat sheet (this document)
- [ ] Don't study new material (just review)
- [ ] Prepare exam location details (address, parking, bathroom)
- [ ] Gather required ID documents
- [ ] Test camera/audio if remote
- [ ] Eat a good breakfast before exam
- [ ] Arrive 15 minutes early

---

## 🎯 During Exam - Mental Checklist

- ✅ Read each question fully before answering
- ✅ Look for key words: "best", "most", "should"
- ✅ Consider all answer options (don't pick first one that works)
- ✅ Think about trade-offs (cost vs performance)
- ✅ Flag questions you're unsure about
- ✅ Review flagged questions if you have time
- ✅ Don't change answers unless you have a GOOD reason
- ✅ Manage time (if stuck, guess and move on)

---

## 🚀 Final Words

**You're prepared. Trust your training.**

- You have comprehensive study materials
- You understand the content
- You know the patterns
- You've done hands-on labs
- You've practiced scenarios

**On exam day:**
- Stay calm and confident
- Read questions carefully
- Think about trade-offs
- Choose the best, not just "good enough"
- You know this content

**You've got this!** 💪

---

**Good luck on your exam! You're going to pass! 🎉**

*Retake Exam Date: Week of June 14, 2026*  
*Target Score: 700+*  
*You're 28 points away. This is achievable.*
