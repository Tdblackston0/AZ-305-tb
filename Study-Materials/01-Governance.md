# Design Governance - CRITICAL PRIORITY ⭐⭐⭐

**Exam Weight:** Part of Identity, Governance, Monitoring (25-30%)  
**Your Performance:** ⚠️⚠️ WEAKEST AREA  
**Potential Points:** +10-15

---

## Overview

**Governance** is one of your three lowest-scoring areas and represents a significant opportunity for improvement. Azure governance ensures organizations can:
- Control and audit resource usage
- Enforce compliance policies
- Manage costs and access
- Maintain security posture at scale

---

## Core Concepts You Must Master

### 1. Management Groups

**What they are:** Hierarchical containers for managing access, policies, and compliance across multiple Azure subscriptions

**Key Points:**
- Organize subscriptions into a hierarchy
- Apply policies and RBAC at any level (inherited downward)
- Useful for large organizations with multiple departments/teams
- Root Management Group: Tenant Root Group (cannot be renamed/deleted)

**Governance Decision:**
```
Tenant Root Group
├── Production (MG)
│   ├── App1-Prod (Subscription)
│   └── App2-Prod (Subscription)
├── Non-Production (MG)
│   ├── App1-Dev (Subscription)
│   └── App1-Test (Subscription)
└── Shared Services (MG)
    └── Shared-Prod (Subscription)
```

**Exam Scenario:** "Your company has 50 subscriptions across multiple departments and regions. You need to ensure all subscriptions follow the same tagging standard and enforce a firewall policy. What's the best approach?"
- **Answer:** Create Management Groups to organize subscriptions, then apply policies at the appropriate MG level.

**Microsoft Learn:** [Manage Azure subscriptions and governance](https://learn.microsoft.com/training/modules/govern-subscriptions/)

---

### 2. Azure Policy

**What it is:** A service for creating, assigning, and managing policies that enforce organizational standards

**Policy vs Initiative:**
- **Policy:** Single rule (e.g., "All VMs must have backups")
- **Initiative:** Collection of related policies (e.g., "Security baseline" with 10 policies)

**Key Policy Types:**
1. **Audit** - Reports non-compliance (doesn't block)
2. **Deny** - Blocks non-compliant resource creation
3. **DeployIfNotExists** - Automatically deploys remediation
4. **Modify** - Changes resource properties (tags, encryption settings)

**Governance Decisions:**
| Requirement | Policy Type | Example |
|-------------|------------|---------|
| Mandatory tagging | Deny/Modify | Require "Environment" and "CostCenter" tags |
| Audit compliance | Audit | Track VMs without Managed Disks |
| Auto-remediate | DeployIfNotExists | Deploy VM extensions for monitoring |
| Enforce location | Deny | Block resources in non-approved regions |

**Exam Scenario:** "You need to ensure all resources have a 'Department' tag, and the tag should match one of 5 approved values. Existing resources without tags should be automatically tagged with 'Department: General'."
- **Answer:** Use a Modify policy with conditions to enforce the tag and auto-remediate existing resources.

**Hands-On Lab:**
```
1. Create custom policy definition
2. Assign policy at subscription level
3. Review compliance reports
4. Test remediation
```

**Microsoft Learn:** [Implement Azure Policy](https://learn.microsoft.com/training/modules/manage-azure-policy/)

---

### 3. Role-Based Access Control (RBAC)

**Three key elements:**
1. **Security Principal** (Who) - User, group, service principal, managed identity
2. **Role** (What) - Collection of permissions
3. **Scope** (Where) - Management Group, Subscription, Resource Group, Resource

**Scope Hierarchy (inheritance):**
```
Management Group (highest scope)
  ↓ (role assignments inherited down)
Subscription
  ↓
Resource Group
  ↓
Resource (lowest scope)
```

**Built-in Roles You Must Know:**
| Role | Permissions |
|------|-------------|
| **Owner** | Full access including role assignment |
| **Contributor** | Full access except cannot grant roles |
| **Reader** | Read-only access |
| **User Access Administrator** | Can manage role assignments |

**Custom Roles:**
- Create when built-in roles don't fit your needs
- Include specific actions/permissions
- Scope: Management Group, Subscription, or Resource Group

**Principle of Least Privilege:**
- Assign minimum permissions needed for the job
- Use custom roles to enforce this
- Regular access reviews

**Exam Scenario:** "A developer needs to create/modify VMs in a test resource group but should not be able to delete resources or modify networking. Additionally, a DBA needs to manage SQL databases but only for a specific environment. Design RBAC."
- **Answer:**
  - Developer: Custom role with limited VM creation/management permissions at RG scope
  - DBA: Built-in "SQL DB Contributor" at database scope

**Hands-On Lab:**
```
1. Create custom RBAC role
2. Assign role to a user/group
3. Verify access with Azure CLI
4. Test permission boundaries
```

**Microsoft Learn:** [Manage access with RBAC](https://learn.microsoft.com/training/modules/secure-azure-resources-with-rbac/)

---

### 4. Cost Management & Governance

**Key Concepts:**
- **Cost Analysis** - Visualize and analyze spending
- **Budgets** - Set spending limits with alerts
- **Reservations** - Pre-pay for compute for 1-3 years (up to 72% savings)
- **Spot VMs** - Use unused capacity (70% discount, interruptible)

**Governance Use Cases:**
1. Tag-based chargeback (charge departments by tags)
2. Enforce reserved instances for predictable workloads
3. Alert when spending exceeds threshold
4. Disable/auto-shutdown non-production resources

**Exam Scenario:** "Your company wants to charge back cloud costs to individual departments. Each resource must have a 'Department' tag. How do you implement this governance model?"
- **Answer:** Use cost analysis grouped by Department tag, create budget alerts per department tag, enforce department tagging via policy.

**Microsoft Learn:** [Manage costs in Azure](https://learn.microsoft.com/training/modules/manage-azure-costs/)

---

### 5. Compliance & Regulatory Frameworks

**Key Compliance Standards:**
- **HIPAA** (Healthcare) - Protected health information
- **PCI-DSS** (Payment card) - Cardholder data security
- **SOC 2** (Service organizations) - Security, availability, confidentiality
- **GDPR** (EU) - Personal data protection
- **FedRAMP** (US Government) - Federal cloud requirements

**Governance Implementation:**
1. **Azure Blueprints** - Repeatable governance packages
2. **Azure Resource Manager policies** - Enforce standards
3. **Compliance Manager** - Track compliance status
4. **Regulatory compliance dashboard** - Visibility into compliance state

**Exam Scenario:** "Your company must comply with HIPAA. Design a governance structure to ensure all resources meet HIPAA requirements."
- **Answer:** Use Azure Blueprints with HIPAA template, enforce encryption policies, implement network isolation, enable audit logging, use Azure Policy for compliance standards.

**Microsoft Learn:** [Azure compliance and governance](https://learn.microsoft.com/training/modules/azure-governance-compliance/)

---

## Decision Trees

### "Should I use a Policy or an Initiative?"
```
Do I need to enforce a single rule?
├─ YES → Use Policy
└─ NO (multiple related rules) → Use Initiative
```

### "What scope should I assign RBAC?"
```
Same role for multiple subscriptions?
├─ YES → Assign at Management Group level
└─ NO (single subscription)
    ├─ Same role across all RGs? 
    │   ├─ YES → Assign at Subscription level
    │   └─ NO → Assign at Resource Group level
```

### "When to use Custom Roles?"
```
Does a built-in role match your needs?
├─ YES (uses all needed permissions, no extra) → Use built-in
└─ NO
    ├─ Add too many extra permissions → Custom role
    ├─ Need to limit specific actions → Custom role
```

---

## Common Governance Mistakes (Avoid These!)

❌ **Mistake 1:** Assigning Owner role to everyone  
✅ **Fix:** Use Contributor or custom roles with least privilege

❌ **Mistake 2:** Setting policies but not reviewing compliance  
✅ **Fix:** Schedule monthly compliance reviews, set up alerts

❌ **Mistake 3:** Flat subscription structure (no Management Groups)  
✅ **Fix:** Organize early, even for small organizations

❌ **Mistake 4:** No tagging strategy  
✅ **Fix:** Define tagging standard early, enforce with policy

❌ **Mistake 5:** Ignoring cost governance  
✅ **Fix:** Set budgets, track spending by department/project

---

## Practice Scenarios

### Scenario 1: Multi-Tenant Organization
**Context:** Your organization has 5 business units, each with 3-5 subscriptions. Each BU needs autonomy over resource creation but must follow company security policies.

**Questions:**
1. How do you organize subscriptions?
2. How do you enforce security policies across all BUs?
3. How do you prevent one BU from overspending?
4. How do you manage developers with different permission needs?

**Answer Outline:**
- Create MG per BU under a parent "Organization" MG
- Assign security policies at Organization level (inherited)
- Create budgets per BU subscription
- Use custom RBAC roles for different job functions (Dev, DBA, Admin)

### Scenario 2: Regulatory Compliance
**Context:** Your company must comply with GDPR (data protection) and SOC 2 (security controls).

**Questions:**
1. What governance mechanisms enforce encryption?
2. How do you audit data access?
3. How do you ensure only authorized personnel access data?
4. How do you track compliance over time?

**Answer Outline:**
- Azure Policy to enforce encryption at rest/transit
- Azure audit logs and Application Insights for access tracking
- RBAC with Data Owner role for authorized access
- Compliance Manager to track compliance status

### Scenario 3: Cost Optimization with Governance
**Context:** Your organization wants to reduce cloud costs by 30% while maintaining security and compliance.

**Questions:**
1. How do you identify wasteful spending?
2. How do you enforce cost-conscious decisions?
3. How do you balance cost with other governance requirements?

**Answer Outline:**
- Use cost analysis to identify high-cost resource groups/tags
- Policy to enforce reserved instances for predictable workloads
- Policy to auto-shutdown non-production resources
- Budget alerts per department with cost center tags

---

## Exam Tips for Governance

1. **Policy effects matter**: Audit vs Deny vs DeployIfNotExists have different use cases
2. **Scope is everything**: Understand inheritance and scope boundaries
3. **Least privilege**: Always recommend the minimum permissions needed
4. **Consider compliance**: Many governance scenarios have regulatory requirements
5. **Cost as governance**: Cost control is often a governance requirement

---

## Quick Reference: Governance Checklist

- [ ] Understand Management Group hierarchy
- [ ] Know Policy vs Initiative differences
- [ ] Master RBAC (principals, roles, scopes)
- [ ] Understand custom role creation
- [ ] Know when to use each policy effect (Audit, Deny, DeployIfNotExists, Modify)
- [ ] Understand compliance frameworks (HIPAA, PCI, GDPR, SOC2)
- [ ] Know cost governance mechanisms (budgets, tags, reservations)
- [ ] Understand role inheritance across scopes

---

## Key Microsoft Learn Paths

1. **[Manage Azure subscriptions and governance](https://learn.microsoft.com/training/modules/govern-subscriptions/)** - 45 min
2. **[Implement resource management locks](https://learn.microsoft.com/training/modules/implement-resource-management-locks/)** - 30 min
3. **[Configure Azure Policy](https://learn.microsoft.com/training/modules/configure-azure-policy/)** - 40 min
4. **[Secure Azure resources with RBAC](https://learn.microsoft.com/training/modules/secure-azure-resources-with-rbac/)** - 50 min

**Total Study Time:** 2-3 hours  
**Hands-On Labs:** 1 hour

---

## Next Steps

1. ✅ Read this guide (you're here!)
2. **→ Complete Microsoft Learn modules above**
3. **→ Do hands-on labs in Azure Portal (RBAC, Policy, MG)**
4. **→ Work through the practice scenarios**
5. **→ Take practice exam questions on governance**

**You've got this! Governance is learnable, and you're close to passing.** 🚀
