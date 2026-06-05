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

**Microsoft Learn:** [Organize resources with Azure management groups](https://learn.microsoft.com/en-us/azure/governance/management-groups/overview)

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

**Microsoft Learn:** [Implement Azure Policy](https://learn.microsoft.com/en-us/training/modules/describe-purpose-of-azure-policy/)

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

**Microsoft Learn:** [Control Azure services with Role-Based Access Control (RBAC)](https://learn.microsoft.com/en-us/training/modules/secure-azure-resources-with-rbac/)

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

**Microsoft Learn:** [Analyze costs and create budgets with Cost Management](https://learn.microsoft.com/en-us/training/modules/analyze-costs-create-budgets-azure-cost-management/)

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

**Microsoft Learn:** [Governance in Azure](https://learn.microsoft.com/en-us/training/paths/govern-azure/)

---

### 6. Resource Locks

**What they are:** Lightweight protection mechanism preventing accidental deletion or modification

**Lock Types:**
- **CanNotDelete** - Resource can be modified but NOT deleted
- **ReadOnly** - Resource CANNOT be modified OR deleted (applies to all users except Owner)

**Key Points:**
- Applied at: Subscription, Resource Group, or Individual Resource level
- Locks inherit downward (lock on RG applies to all resources within it)
- Owner role can DELETE but NOT modify/bypass locks (must remove lock first)
- Locks do NOT replace RBAC or Policy—they complement them

**Use Cases:**
- Production resource groups (prevent accidental teardown)
- Critical shared resources (DNS zones, VNets, databases)
- Backup after compliance deployment (lock down compliant state)

**Exam Scenario:** "A production database was just deployed. Ensure no one can accidentally delete it, but it can still be modified for maintenance."
- **Answer:** Apply `CanNotDelete` lock at the database level

**Common Mistake:** Using locks instead of RBAC for access control. Locks are for **accident prevention**, not authorization.

**Microsoft Learn:** [Lock resources to prevent unexpected changes](https://learn.microsoft.com/en-us/azure/azure-resource-manager/management/lock-resources)

---

### 7. Privileged Identity Management (PIM)

**What it is:** Just-in-time (JIT) access elevation for admin roles—users get temporary, time-bound elevated permissions

**Problem it solves:**
- ❌ Without PIM: Admin roles assigned permanently (security risk if account compromised)
- ✅ With PIM: Admin roles activated only when needed, for limited duration

**How it works:**
```
User Request
  ↓
User activates Owner role (requests elevation)
  ↓
Approver notified (if approval required)
  ↓
Role activated for X hours (default 8)
  ↓
Access expires → role automatically deactivated
```

**Key Components:**

| Component | Purpose |
|-----------|---------|
| **Eligible Assignment** | User can activate the role when needed (not active by default) |
| **Active Assignment** | Role is always active (use sparingly) |
| **Activation Request** | User requests temporary access; approver can accept/deny |
| **MFA/Justification** | Require MFA and business justification for activation |
| **Time-bound Access** | Role expires automatically after X hours |
| **Audit Trail** | Track who activated what and when |

**Exam Scenario:** "Design access for production admins. They should have full access only when actively working, and every activation should be logged and require approval."
- **Answer:** 
  - Assign as PIM-eligible (not active) Owner/Contributor
  - Enable approval requirement for activation
  - Require MFA for activation
  - Set activation duration to 4 hours
  - Enable audit logging

**Common Patterns:**

| Scenario | Pattern |
|----------|---------|
| Break-glass emergency access | PIM Owner with minimal approval required |
| Production admin access | PIM Contributor with approval + MFA |
| Scheduled maintenance | PIM with pre-approved activation window |
| Service principal (automation) | Active assignment (cannot use PIM for non-interactive) |

**Microsoft Learn:** [PIM Overview](https://learn.microsoft.com/en-us/azure/active-directory/privileged-identity-management/pim-configure)

---

### 8. Access Reviews

**What it is:** Periodic audit to verify users still need their assigned roles

**Why it matters:**
- Compliance requirement (regulatory frameworks require access certification)
- Security risk: Stale access (user changed roles but still has old permissions)
- Least privilege enforcement: Remove unnecessary permissions

**How it works:**
```
1. Schedule review (quarterly/annually)
2. Reviewers verify each user assignment
3. Approve (keep access) or Deny (remove access)
4. Report shows compliance status
5. Automatic removal of denied access
```

**Who reviews:**
- **Managers** - Review their team's access
- **Resource Owners** - Review who has access to their resources
- **Security Team** - Review sensitive role assignments

**Exam Scenario:** "Design an access governance strategy. How do you ensure developers don't accumulate unnecessary permissions over time?"
- **Answer:** Implement quarterly Access Reviews with manager review for developer roles; auto-remove unapproved access

**Best Practices:**
- Schedule quarterly or semi-annually (compliance often requires annual)
- Include all role assignments above Contributor
- Enable auto-apply (automatically remove unapproved access)
- Require reviewers to certify they understand each user's role

**Microsoft Learn:** [Access Reviews](https://learn.microsoft.com/en-us/azure/active-directory/governance/access-reviews-overview)

---

### 9. Subscription & Resource Group Design

**Subscriptions: When to use**
- **Billing boundary** - Separate cost centers or business units
- **Quota boundary** - Avoid service limits (e.g., 1,000 VMs per subscription)
- **Blast radius** - Isolate production from non-production
- **Compliance isolation** - Separate regulated/sensitive workloads
- **Team autonomy** - Give teams independent infrastructure

**Subscriptions: When NOT to use**
- Simple resource grouping (use Resource Groups instead)
- Temporary projects (overhead to create/manage)
- Fine-grained access control (use Resource Groups + RBAC)

**Common subscription structure:**
```
Production Subscription
├─ RG: Payment-processing
├─ RG: Web-tier
└─ RG: Database

Non-Production Subscription
├─ RG: Dev
├─ RG: Test
└─ RG: Staging

Shared Services Subscription
├─ RG: Networking
├─ RG: Security
└─ RG: Monitoring
```

**Resource Groups: Design principles**
- **By application** - One RG per app (easier to delete/manage whole app)
- **By environment** - Separate dev/test/prod RGs for each app
- **By team** - One RG per team (team-based cost tracking)
- **By function** - One RG for databases, one for VMs (common anti-pattern)

**Naming conventions:**
```
Subscription: <org>-<environment>-<region>
RG: rg-<application>-<environment>-<region>
  Example: rg-payments-prod-eastus
```

---

### 10. Tagging Strategy

**Why tags matter:**
- **Cost chargeback** - Track spending by department/project
- **Operations** - Automation (auto-shutdown, backup policies)
- **Compliance** - Track data sensitivity, retention requirements
- **Ownership** - Identify who manages each resource
- **Reporting** - Filter and analyze resources

**Recommended tags:**

| Tag | Example Values | Purpose |
|-----|-----------------|---------|
| **Environment** | Prod, Dev, Test, Staging | Deployment stage |
| **CostCenter** | Finance, HR, Eng | Billing/chargeback |
| **Owner** | john@company.com | Contact for escalation |
| **Application** | Payments, HR-System | Business app |
| **DataClassification** | Public, Internal, Confidential | Sensitivity level |
| **BackupPolicy** | Daily, Weekly, None | Retention requirement |

**Enforce tagging with Policy:**
Use **Modify** policy to auto-tag resources:
```
Policy: Auto-tag with Application name (from RG tag)
├─ If resource created in RG tagged "Application": Payments
└─ Then automatically add tag to resource: Application: Payments
```

**Exam Scenario:** "Implement mandatory tagging for cost allocation. Tags should not be removable, and missing tags should be auto-filled."
- **Answer:**
  - Policy Deny: Prevent resource creation without required tags
  - Policy Modify: Auto-populate tags from Resource Group or defaults
  - Document approved tag values

**Tag limitations:**
- Max 50 tags per resource
- Tag keys max 512 characters
- Tag values max 256 characters
- Tags don't automatically inherit to resources (use Policy to enforce inheritance)

**Microsoft Learn:** [Use tags to organize Azure resources](https://learn.microsoft.com/en-us/azure/azure-resource-manager/management/tag-resources)

---

### 11. Licensing Requirements for Governance Features

**Important Context:** Not all governance features are included in every Azure subscription. Some require specific Entra ID licenses.

**Licensing Comparison:**

| Feature | License Required | Cost | Notes |
|---------|-----------------|------|-------|
| **Resource Locks** | ❌ None | Free | Included in all Azure subscriptions |
| **Management Groups** | ❌ None | Free | Included in all Azure subscriptions |
| **Azure Policy** | ❌ None | Free | Included in all Azure subscriptions |
| **RBAC** | ❌ None | Free | Included in all Azure subscriptions |
| **PIM (Privileged Identity Management)** | ✅ Azure AD Premium P2 | ~$25/user/month | Can license specific users who need it |
| **Access Reviews** | ✅ Azure AD Premium P2 | ~$25/user/month | Can license specific users who need it |

---

### **Azure AD Premium P2 Includes:**
- Privileged Identity Management (PIM)
- Access Reviews
- Identity Protection
- Conditional Access advanced features

**Alternative:** Microsoft 365 E5 license includes Azure AD Premium P2

---

### **Real-World Licensing Strategy:**

**Typical organization with 1,000 employees:**
- ✅ **Free features** apply to everyone:
  - Resource Locks, Management Groups, Policy, RBAC

- ✅ **P2 licenses** for ~50-100 people:
  - Admins (PIM-eligible roles)
  - Security/compliance teams (Access Reviews)
  - Sensitive users requiring identity protection

- ✅ **Cost estimate:** 50 users × $25 = $1,250/month

---

### **Exam Traps to Avoid:**

❌ **Trap 1:** "Implement PIM for all developers"
✅ **Better:** "Implement PIM for admin roles; developers get standard RBAC with quarterly Access Reviews"

❌ **Trap 2:** "Design Access Reviews organization-wide"
✅ **Better:** "Implement Access Reviews for privileged roles and critical resources; document review process for other roles"

❌ **Trap 3:** "Every user needs P2 licensing"
✅ **Better:** "License admins and security staff with P2; use alternative cost-effective solutions for other users"

---

### **Exam Scenario with Licensing:**
"Design access governance for a 500-person company with $5K/month IT budget. Ensure least privilege for admins and regular access reviews."

**Good Answer:**
- Use Resource Locks (free) for production resources
- Assign RBAC at MG level (free)
- Implement PIM for 30 admin accounts (~$750/month)
- Quarterly Access Reviews via P2 licensing
- Cost: ~$1,000/month (within budget)

**Poor Answer:**
- "Implement PIM for all 500 users" (unrealistic cost)
- "Use Access Reviews for everyone" (licensing expense)

---

### **When to Recommend Premium Licensing:**

✅ **Recommend P2 when:**
- Organization has regulatory/compliance requirements
- Admin accounts have high-risk access
- Sensitive data or systems involved
- Security incident response is critical

❌ **Don't recommend P2 for:**
- Small organizations (<50 people)
- Low-sensitivity environments
- Development/test accounts
- Cost-constrained startups

**Microsoft Learn:** [Azure AD Licensing](https://learn.microsoft.com/en-us/azure/active-directory/fundamentals/active-directory-whatis)

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
- [ ] Know Resource Lock types and use cases
- [ ] Understand PIM (Just-in-time access)
- [ ] Know Access Review requirements and processes
- [ ] Design subscription and resource group hierarchy
- [ ] Create comprehensive tagging strategy
- [ ] Enforce tagging with Policy
- [ ] Know when to use Locks vs RBAC vs Policy
- [ ] Understand licensing requirements (P2 for PIM/Access Reviews)
- [ ] Know when to recommend premium licensing
- [ ] Recognize licensing exam traps

---

## Key Microsoft Learn Paths

1. **[Organize resources with Azure management groups](https://learn.microsoft.com/en-us/azure/governance/management-groups/overview)** - 30 min
2. **[Control Azure services with Role-Based Access Control (RBAC)](https://learn.microsoft.com/en-us/training/modules/secure-azure-resources-with-rbac/)** - 50 min
3. **[Introduction to Azure Policy](https://learn.microsoft.com/en-us/training/modules/intro-to-azure-policy/)** - 45 min
4. **[Control Azure spending and manage bills](https://learn.microsoft.com/en-us/training/paths/control-spending-manage-bills/)** - 40 min
5. **[PIM Overview](https://learn.microsoft.com/en-us/azure/active-directory/privileged-identity-management/pim-configure)** - 30 min
6. **[Access Reviews](https://learn.microsoft.com/en-us/azure/active-directory/governance/access-reviews-overview)** - 20 min
7. **[Lock resources to prevent unexpected changes](https://learn.microsoft.com/en-us/azure/azure-resource-manager/management/lock-resources)** - 15 min
8. **[Use tags to organize Azure resources](https://learn.microsoft.com/en-us/azure/azure-resource-manager/management/tag-resources)** - 20 min
9. **[Azure AD Licensing](https://learn.microsoft.com/en-us/azure/active-directory/fundamentals/active-directory-whatis)** - 10 min

**Total Study Time:** 4.5-5.5 hours  
**Hands-On Labs:** 1.5 hours

---

## Next Steps

1. ✅ Read this guide (you're here!)
2. **→ Complete Microsoft Learn modules (all 9 paths above)**
3. **→ Understand licensing constraints:**
   - Identify which features are free vs Premium P2
   - Practice recommending cost-effective solutions
   - Avoid licensing traps in exam scenarios
4. **→ Do hands-on labs:**
   - Create Management Group hierarchy
   - Assign RBAC and custom roles
   - Create and test Azure Policies
   - Set up Resource Locks
   - Configure PIM and Access Reviews
   - Implement tagging strategy with Policy
5. **→ Work through the practice scenarios (pay attention to budget constraints)**
6. **→ Take practice exam questions on governance**

**Priority order if short on time:**
1. Management Groups + RBAC (foundation)
2. Azure Policy (enforcement)
3. PIM + Access Reviews (security controls) + Licensing context
4. Cost Management (business value)
5. Tags + Locks (operational excellence)

**Licensing-specific exam prep:**
- Practice recommending features within budget constraints
- Know which features are free (don't over-recommend P2)
- Understand total cost of ownership (licensing + Azure resources)
- Recognize when licensing questions are trick questions

**You've got this! Governance is learnable, and you're close to passing.** 🚀
