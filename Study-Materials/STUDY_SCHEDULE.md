# AZ-305 Retake Study Schedule
## June 14th Week Exam - 2 Week Intensive Study Plan

**Start Date:** May 31, 2026  
**Exam Date:** Week of June 14, 2026  
**Current Score:** 672/700 (28 points away)  
**Study Duration:** 14 days (2 weeks)

---

## 📅 Week 1: Priority Skills Crash Course (June 1-7)

### Goal
Master the **Top 3 Priority Skills** that are your biggest weaknesses:
1. Design governance
2. Design application architecture  
3. Design data integration

**Time Commitment:** 4-5 hours daily

---

### **Day 1-2: Design Governance (Mon-Tue, June 1-2)**

#### Day 1 (Monday)
**Goal:** Understand governance fundamentals

**Activities:**
- [ ] Read: 01-Governance.md (sections: Management Groups, Azure Policy, RBAC)
- [ ] Microsoft Learn: [Manage Azure subscriptions and governance](https://learn.microsoft.com/training/modules/govern-subscriptions/) (45 min)
- [ ] **Key Takeaway:** Management Groups hierarchy, policy vs initiative, scope inheritance

**Time:** 2 hours

**Hands-On Lab:**
```
1. Create a Management Group hierarchy (test subscription)
2. Practice: Create/assign 3 built-in policies
3. Test: Verify inheritance from parent MG
```
**Time:** 1 hour

#### Day 2 (Tuesday)
**Goal:** Master RBAC and custom roles

**Activities:**
- [ ] Read: 01-Governance.md (sections: RBAC, Custom Roles)
- [ ] Microsoft Learn: [Secure Azure resources with RBAC](https://learn.microsoft.com/training/modules/secure-azure-resources-with-rbac/) (50 min)
- [ ] **Key Takeaway:** Scope hierarchy, built-in roles, custom role creation, least privilege

**Time:** 2 hours

**Hands-On Lab:**
```
1. Create a custom RBAC role (VM operator - can start/stop but not delete)
2. Assign to a user/group
3. Verify permissions with Azure CLI
4. Test: Try to delete a VM (should fail)
```
**Time:** 1 hour

**Day 1-2 Quiz:**
- [ ] What's the difference between policy and initiative?
- [ ] Draw the scope inheritance chain and explain it
- [ ] When would you use a custom role instead of built-in?

---

### **Day 3-4: Design Application Architecture (Wed-Thu, June 3-4)**

#### Day 3 (Wednesday)
**Goal:** Master compute service decision matrix

**Activities:**
- [ ] Read: 02-App-Architecture.md (first 3 services: VMs, App Service, AKS)
- [ ] Microsoft Learn: [Choose the right compute service](https://learn.microsoft.com/training/modules/choose-compute-service-azure/) (45 min)
- [ ] **Key Takeaway:** Service matrix, when to use each, tradeoffs

**Time:** 2 hours

**Decision Practice:**
```
Answer 5 quick scenarios:
1. E-commerce web app, needs auto-scale
2. Legacy monolithic app, needs full OS control
3. Microservices with 15 independent services
4. Batch IoT data processing
5. Internal CRUD app, low traffic
```
**Time:** 30 min

#### Day 4 (Thursday)
**Goal:** Master scaling patterns and architecture decisions

**Activities:**
- [ ] Read: 02-App-Architecture.md (Scaling Strategies, Practice Scenarios)
- [ ] Microsoft Learn: [Design scalable applications](https://learn.microsoft.com/training/modules/app-service-plan-azure-apps/) (40 min)
- [ ] **Key Takeaway:** Auto-scale rules, metric vs schedule-based, architecture patterns

**Time:** 2 hours

**Hands-On Lab:**
```
1. Deploy a sample app (App Service or VMSS)
2. Configure auto-scale rule (CPU-based)
3. Create test load to trigger scaling
4. Observe scaling behavior
```
**Time:** 1-2 hours

**Day 3-4 Quiz:**
- [ ] List 5 compute services and when to use each
- [ ] Design architecture for: "10x traffic surge on Black Friday"
- [ ] What's the difference between HPA and VMSS scaling?

---

### **Day 5-6: Design Data Integration (Fri-Sat, June 5-6)**

#### Day 5 (Friday)
**Goal:** Understand ETL/ELT and data pipeline patterns

**Activities:**
- [ ] Read: 03-Data-Integration.md (Core Concepts, Pipeline Patterns)
- [ ] Microsoft Learn: [Design data integration](https://learn.microsoft.com/training/modules/design-data-integration/) (50 min)
- [ ] **Key Takeaway:** ETL vs ELT, when to use batch vs real-time, pipeline design

**Time:** 2 hours

**Scenario Practice:**
```
1. Design batch ETL (daily data warehouse load)
2. Design real-time streaming (IoT data analytics)
3. Describe the data lake layered architecture
```
**Time:** 30 min

#### Day 6 (Saturday)
**Goal:** Master service selection for data workloads

**Activities:**
- [ ] Read: 03-Data-Integration.md (Services, Decision Trees, Scenarios)
- [ ] Microsoft Learn: [Azure Data Factory fundamentals](https://learn.microsoft.com/training/modules/data-factory-fundamentals/) (45 min)
- [ ] **Key Takeaway:** ADF, Synapse, Data Lake structure, when to use each

**Time:** 2 hours

**Hands-On Lab:**
```
1. Create Azure Data Lake structure (raw/processed/curated)
2. (Optional) Create simple ADF pipeline
3. Design data flow for: "150GB raw data daily ingestion"
```
**Time:** 1-2 hours

**Day 5-6 Quiz:**
- [ ] When is ETL better than ELT? When is ELT better?
- [ ] What's the advantage of a layered Data Lake?
- [ ] Design a data pipeline for: "Process 100K events/sec in real-time"

---

### **Day 7: Review & Practice (Sunday, June 7)**

**Goal:** Solidify top 3 priority skills

**Activities:**
- [ ] **Review:** Re-read the 3 priority study guides (skim, 30 min each)
- [ ] **Practice Exam Questions:** Take a timed 20-question test focusing on:
  - Governance scenarios (5 questions)
  - Architecture decisions (5 questions)
  - Data integration design (5 questions)
  - Bonus questions (5 questions)
- [ ] Use [06-Practice-Exam.md](06-Practice-Exam.md) and complete questions 1-20 under timed conditions
- [ ] **Time:** 1.5 hours
- [ ] **Review Mistakes:** For any wrong answers, understand why

**Hands-On Review:**
```
1. Recreate a Management Group structure from scratch (15 min)
2. Describe 3 application architecture patterns (15 min)
3. Design a data integration solution (15 min)
```

---

## 📅 Week 2: Secondary Skills + Full Exam Prep (June 8-14)

### Goal
Strengthen secondary domains and prepare for full exam

**Time Commitment:** 3-4 hours daily (lighter week)

---

### **Day 8-9: Identity, Governance & Monitoring (Mon-Tue, June 8-9)**

#### Day 8 (Monday)
**Goal:** Learn conditional access and managed identity

**Activities:**
- [ ] Read: 04-Identity-Governance-Monitoring.md (Identity section)
- [ ] Microsoft Learn: [Conditional Access policies](https://learn.microsoft.com/training/modules/conditional-access-azure-ad/) (40 min)
- [ ] **Key Takeaway:** Conditional Access use cases, MFA, device compliance

**Time:** 1.5 hours

#### Day 9 (Tuesday)
**Goal:** Learn monitoring and alerting

**Activities:**
- [ ] Read: 04-Identity-Governance-Monitoring.md (Monitoring section)
- [ ] Microsoft Learn: [Monitor Azure resources](https://learn.microsoft.com/training/modules/monitor-azure-resources/) (50 min)
- [ ] **Key Takeaway:** Azure Monitor, alerts, log analytics basics

**Time:** 1.5 hours

---

### **Day 10-11: Infrastructure Solutions (Wed-Thu, June 10-11)**

#### Day 10 (Wednesday)
**Goal:** Master VNets, NSGs, Load Balancing

**Activities:**
- [ ] Read: 05-Infrastructure-Solutions.md (Networking, Load Balancing sections)
- [ ] Microsoft Learn: [Design network solutions](https://learn.microsoft.com/training/modules/design-network-solutions/) (50 min)
- [ ] **Key Takeaway:** VNet design, NSG rules, Load Balancer vs Application Gateway

**Time:** 1.5 hours

**Quick Quiz:**
- [ ] Design 3-tier network (web/app/db) with NSGs
- [ ] When would you use each load balancing service?

#### Day 11 (Thursday)
**Goal:** Learn HA/DR and hybrid connectivity

**Activities:**
- [ ] Read: 05-Infrastructure-Solutions.md (HA/DR, Connectivity sections)
- [ ] Microsoft Learn: [Design resilient solutions](https://learn.microsoft.com/training/modules/design-resilient-applications/) (45 min)
- [ ] **Key Takeaway:** Availability Zones, Failover Groups, RTO/RPO, ExpressRoute vs VPN

**Time:** 1.5 hours

---

### **Day 12: Data Storage Review (Friday, June 12)**

**Goal:** Strengthen any weak areas in data storage domain

**Activities:**
- [ ] **Quick Review:** Data Storage domain fundamentals
  - SQL vs NoSQL decision trees
  - Cosmos DB vs SQL Database
  - Storage account types (LRS, GRS, RA-GRS)
- [ ] Microsoft Learn: Pick any missed module from [Data Storage path](https://learn.microsoft.com/training/paths/design-data-storage-solutions/)
- [ ] **Time:** 1.5 hours

---

### **Day 13: Full Practice Exam (Saturday, June 13)**

**Goal:** Take a complete practice exam to assess readiness

**Activities:**
- [ ] **Official Practice Test:** Microsoft Learn practice exam
  - Time limit: 120 minutes (same as real exam)
  - Full 40 questions (real exam has ~50)
  - Score goal: 700+
- [ ] **Local Practice Test:** Complete [06-Practice-Exam.md](06-Practice-Exam.md) and review every rationale
- [ ] **If passed:** Review weak areas, do targeted study
- [ ] **If failed:** Focus on remaining mistakes, review those domains

**Time:** 2.5 hours

**Post-Exam Analysis:**
```
Questions missed:
- Governance: _____ (target: 0)
- Architecture: _____ (target: 0-1)
- Data Integration: _____ (target: 0-1)
- Other: _____ (review but less critical)

Action items for tomorrow:
- Deep dive any missed concepts
- Do targeted hands-on labs
```

---

### **Day 14: Final Prep & Rest (Sunday, June 13)**

**Goal:** Final touches and mental preparation

**Activities:**
- [ ] **Review Mistakes:** Spend 2 hours reviewing any failed practice test questions
- [ ] **Quick Refresher:** Review cheat sheets (below)
- [ ] **Exam Tips:** Read "Exam-Taking Tips" section in README.md
- [ ] **Rest:** Get good sleep before exam

**Final Checklist:**
- [ ] Review Microsoft Learn resources one more time
- [ ] Make sure you know when to use each service
- [ ] Practice governance, architecture, data integration scenarios once more
- [ ] Set up exam day logistics (arrival time, ID, etc.)

---

## 📝 Daily Study Time Summary

| Week | Mon | Tue | Wed | Thu | Fri | Sat | Sun |
|------|-----|-----|-----|-----|-----|-----|-----|
| **Week 1** | 3h | 3h | 2.5h | 2.5h | 2.5h | 2.5h | 2h |
| **Week 2** | 1.5h | 1.5h | 1.5h | 1.5h | 1.5h | 2.5h | 2h |

**Total Study Hours:** ~44 hours (achievable in 2 weeks!)

---

## 🎯 Quick Reference: What to Know by Each Date

### By June 7 (End of Week 1)
✅ Management Groups hierarchy  
✅ Azure Policy (all policy effects)  
✅ RBAC (scope, built-in roles, custom roles)  
✅ Compute service decision matrix  
✅ When to use each compute service  
✅ Application architecture patterns  
✅ ETL vs ELT  
✅ Data pipeline patterns  
✅ Data integration service selection  

### By June 14 (Exam Day)
✅ Everything above +  
✅ Conditional Access scenarios  
✅ Azure Monitor alerting  
✅ VNet, NSG, Load Balancer design  
✅ HA/DR patterns  
✅ ExpressRoute vs VPN  
✅ All governance, architecture, data integration scenarios  
✅ Confidence level: 8-9/10

---

## 🔗 Quick Links to Study Materials

**Priority Study Guides (read first):**
1. [01-Governance.md](01-Governance.md) - CRITICAL
2. [02-App-Architecture.md](02-App-Architecture.md) - CRITICAL
3. [03-Data-Integration.md](03-Data-Integration.md) - CRITICAL

**Secondary Guides:**
4. [04-Identity-Governance-Monitoring.md](04-Identity-Governance-Monitoring.md)
5. [05-Infrastructure-Solutions.md](05-Infrastructure-Solutions.md)

**Reference:**
- [README.md](README.md) - Overview and exam tips

---

## 💡 Study Tips for Maximum Effectiveness

### 1. Active Learning Only
❌ **Don't:** Just read the guides passively  
✅ **Do:** After each section, answer quiz questions, do hands-on labs

### 2. Hands-On Labs Are Critical
- Reading alone won't be enough
- You need Azure portal experience
- Each guide has hands-on labs - **DO THEM**

### 3. Scenario-Based Thinking
- Exam questions are scenario-based
- Practice designing solutions, not memorizing facts
- Think: "What would I recommend if I were the architect?"

### 4. Manage Time Wisely
- Spend 60% time on governance + architecture + data integration
- Spend 20% time on identity + monitoring
- Spend 20% time on infrastructure + data storage

### 5. Take Official Practice Exam
- On Day 13, take a full practice exam
- Microsoft provides free practice exams
- This will tell you if you're ready

---

## 🚀 You've Got This!

**Why you'll pass this time:**
- ✅ You were only 28 points away
- ✅ Your fundamentals are strong
- ✅ You now have targeted, focused study materials
- ✅ You have 2 weeks of focused time
- ✅ This study plan is realistic and achievable

**Success requires:**
1. Following this schedule
2. Completing all hands-on labs
3. Taking the practice exam
4. Reviewing your mistakes
5. Believing you can do it

**You can definitely pass this exam.** The fact that you were so close (672 vs 700) means you just need to fill in those specific knowledge gaps. This study guide is exactly targeted at those gaps.

---

## 📞 Need Help?

If you get stuck on a topic:
1. Re-read the relevant study guide section
2. Watch the Microsoft Learn module
3. Do the hands-on lab again
4. Search Microsoft Learn docs for more details
5. Ask colleagues or mentors

**Don't give up!** You're 28 points away. That's achievable with focused study.

---

**Exam Date:** Week of June 14, 2026  
**Target Score:** 700+  
**Your Goal:** PASS ✅

**Let's do this! 🎯**
