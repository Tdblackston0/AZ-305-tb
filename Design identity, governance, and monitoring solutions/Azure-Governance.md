# Azure Governance - AZ-305 Comprehensive Cheat Sheet

> 📝 **Hands-On Labs:** [Governance Labs](./Labs/Azure-Governance-Labs.md)

> 🎯 **Exam Focus:** AZ-305 tests your ability to **design** governance hierarchies, policies, and access controls for enterprise Azure environments.

---

<a id="table-of-contents"></a>
## Table of Contents

- [Azure Governance Family Overview](#1-azure-governance-family-overview)
- [When to Choose Which — Decision Tree](#2-when-to-choose-which--decision-tree)
- [Management Groups](#3-management-groups)
- [Subscriptions](#4-subscriptions)
- [Resource Groups](#5-resource-groups)
- [Azure RBAC (Role-Based Access Control)](#6-azure-rbac-role-based-access-control)
- [Azure Policy](#7-azure-policy)
- [Azure Blueprints](#8-azure-blueprints)
- [Resource Locks](#9-resource-locks)
- [Tags](#10-tags)
- [Cost Optimization](#11-cost-optimization)
- [Availability & Resilience](#12-availability--resilience)
- [Compliance & Regulatory](#13-compliance--regulatory)
- [Landing Zones (CAF)](#14-landing-zones-caf)
- [AZ-305 Decision Scenarios](#15-az-305-decision-scenarios)
- [Quick Reference Trigger Table](#16-quick-reference-trigger-table)
- [Common Exam Traps](#17-common-exam-traps)
- [Governance Command Reference](#18-governance-command-reference)
- [🎯 Final AZ-305 Exam Tips](#19--final-az-305-exam-tips)
- [📐 Architecture Decision Flowchart](#20--architecture-decision-flowchart)
- [Exam-Style Review Questions](#21-exam-style-review-questions)

---

<a id="1-azure-governance-family-overview"></a>
## 1. Azure Governance Family Overview

Azure governance is how you keep a cloud estate **secure, cost-controlled, compliant, and consistent at scale**. In AZ-305, the real skill is knowing which governance control solves the requirement with the right scope, inheritance model, and operational overhead.

### Comparison Table

| Governance Control | Primary Scope | Best For | Strength | Common Pitfall |
|---|---|---|---|---|
| **Management Groups** | Above subscriptions | Applying common policy and RBAC across many subscriptions | Enterprise inheritance | Designing around short-term projects instead of stable governance boundaries |
| **Subscriptions** | Billing, quota, and admin boundary | Isolation for production, compliance, or chargeback | Strong blast-radius control | Creating too many subscriptions when a resource group would work |
| **RBAC** | Management group to resource | Controlling **who** can act | Least-privilege access delegation | Using RBAC to enforce configuration standards |
| **Azure Policy** | Management group to resource | Controlling **what** can be deployed or must exist | Compliance and continuous enforcement | Using Policy when the need is really identity or delegation |
| **Azure Blueprints** | Subscription/environment packaging | Legacy packaged governance artifacts | Bundles policy, RBAC, templates, and RGs | Recommending it as the modern default despite deprecation |
| **Resource Locks** | Subscription, RG, or resource | Preventing accidental deletion or modification | Simple protection layer | Treating locks as a substitute for RBAC or Policy |
| **Tags** | Resource groups and resources | Chargeback, automation, reporting, and ownership metadata | Operational visibility | Assuming tags inherit automatically without Policy |

### Governance Hierarchy

```text
Tenant Root Management Group
└─ Management Groups
   └─ Subscriptions
      └─ Resource Groups
         └─ Resources
```

**Design principle:** Govern high in the hierarchy when the rule is enterprise-wide; govern lower when teams need autonomy.

### Detailed Definitions + Real-World Examples

#### Management Groups
Management groups are hierarchy containers above subscriptions. They let architects apply **RBAC and Policy inheritance at scale** across business units, environments, or regulatory boundaries.

**Real-world examples:**
- A global enterprise places all production subscriptions under a `Prod` management group so security and diagnostic policies inherit automatically.
- A regulated organization creates separate `EU` and `US` management group branches to align allowed-region policies with residency obligations.
- A platform team creates `Platform`, `LandingZones`, `Sandbox`, and `Decommissioned` management groups to separate shared services from workload subscriptions.

#### Subscriptions
Subscriptions are Azure's core **billing, quota, and governance boundary**. They are the right design choice when you need strong separation for cost ownership, service limits, production risk, or compliance scope.

**Real-world examples:**
- A retailer separates production and non-production into different subscriptions to isolate quota consumption and reduce blast radius.
- A financial services company gives each regulated workload its own subscription for billing traceability and tighter access reviews.
- A central platform team provisions a shared connectivity subscription and separate workload subscriptions for application teams.

#### RBAC
Azure RBAC determines **who can do what** at a given scope. It is the primary least-privilege control for operators, developers, managed identities, and platform automation.

**Real-world examples:**
- An app team receives Contributor access only at its resource group so it cannot affect shared platform resources.
- A security audit group gets Reader at the management group scope to review posture across all subscriptions.
- An automation identity gets a custom role that can restart VMs but cannot create or delete them.

#### Azure Policy
Azure Policy determines **what configurations are allowed, required, appended, modified, audited, or deployed**. It is the main control for compliance, standardization, and drift reduction.

**Real-world examples:**
- A company uses Policy Deny to restrict deployments to East US and West Europe only.
- A platform team uses Modify to stamp `CostCenter` and `Environment` tags on resources from the resource group.
- A security baseline uses DeployIfNotExists to push diagnostic settings to supported resource types.

#### Azure Blueprints
Azure Blueprints were designed to package governance artifacts such as policy assignments, RBAC assignments, ARM templates, and resource groups into reusable environment definitions. They still appear in exam comparisons even though modern designs prefer newer tooling.

**Real-world examples:**
- A legacy landing zone rollout packaged subscription-level policy, RBAC, and logging templates in a single publishable artifact.
- A government program used Blueprint versions to standardize compliant environment deployment across multiple subscriptions.
- An exam scenario contrasts Blueprints with Policy and Bicep to test whether you know the historical packaging role and the modern replacement pattern.

#### Resource Locks
Resource locks are a lightweight safety mechanism that protect critical scopes from accidental changes. Use them when the main risk is operational error rather than authorization design.

**Real-world examples:**
- A production resource group has a `CanNotDelete` lock so no one can remove it accidentally during maintenance.
- A shared DNS zone is protected with a ReadOnly lock because accidental edits would impact multiple workloads.
- A finance database lock is used as a last-mile safeguard while RBAC and approval workflows handle routine access.

#### Tags
Tags are metadata key-value pairs used for ownership, reporting, automation, retention, and cost allocation. They are foundational for FinOps and operational governance when applied consistently.

**Real-world examples:**
- Finance filters monthly spend by `CostCenter`, `Environment`, and `Application` tags for showback.
- Operations uses a `BackupPolicy` or `Criticality` tag to drive automation and retention choices.
- A CMDB integration reads `Owner` and `ManagedBy` tags to route incidents and lifecycle workflows.

### Why Governance Matters

- **Cost:** Prevent sprawl, enforce tagging, allocate spend, and control SKU usage.
- **Security:** Limit access, reduce privilege, enforce encryption, and require diagnostics.
- **Compliance:** Apply standards consistently across subscriptions and teams.
- **Consistency:** Standardize naming, regions, network patterns, and deployment guardrails.

### CAF Governance Disciplines
The **Cloud Adoption Framework (CAF)** governance model commonly focuses on:
- **Cost management**
- **Security baseline**
- **Resource consistency**
- **Identity baseline**
- **Deployment acceleration**

> **AZ-305 lens:** Governance is not one feature. It is the combined use of **management groups, subscriptions, RBAC, Policy, tags, cost controls, landing zones, and monitoring**.

---

<a id="2-when-to-choose-which--decision-tree"></a>
## 2. When to Choose Which — Decision Tree

```text
Need to solve a governance requirement?
│
├─ Is the problem about WHO can perform an action?
│  └─ YES → Azure RBAC
│
├─ Is the problem about WHAT can be deployed or must exist?
│  └─ YES → Azure Policy
│
├─ Is the problem accidental deletion or accidental modification?
│  └─ YES → Resource Lock
│
├─ Is the problem cost ownership, chargeback, or automation metadata?
│  └─ YES → Tags (+ Policy for enforcement)
│
├─ Do the same controls need to apply across MANY subscriptions?
│  └─ YES → Management Groups
│
├─ Do you need billing, quota, compliance, or blast-radius isolation?
│  └─ YES → Separate Subscription
│
├─ Do you need a repeatable governed foundation for new workloads?
│  └─ YES → Landing Zone + Subscription Vending + Policy/RBAC
│
└─ Is this a legacy packaging question about policy + RBAC + templates together?
   ├─ YES → Know Azure Blueprints as the historical answer
   └─ MODERN ANSWER → Bicep/Terraform + Template Specs + Deployment Stacks + Azure Policy
```

### Quick Decision Matrix

| Requirement | Primary Choice | Usually Paired With |
|---|---|---|
| Enterprise-wide inheritance | Management groups | Policy, RBAC |
| Strong isolation boundary | Subscription | Landing zone standards, budgets |
| Team-level delegation | Resource group scope RBAC | Tags, Policy |
| Mandatory configuration baseline | Azure Policy | Management groups, remediation tasks |
| Delete protection | Resource lock | RBAC, approvals |
| Cost accountability | Tags | Budgets, Advisor, Policy |
| Repeatable governed onboarding | Landing zone | Subscription vending machine |

---

<a id="3-management-groups"></a>
## 3. Management Groups

Management groups let you organize subscriptions above the subscription level for **centralized policy and access inheritance**.

### Key concepts
- Support a hierarchy of **up to 6 management group levels below the tenant root management group**.
- The **root management group** exists automatically for every tenant.
- **Azure Policy** and **Azure RBAC** assignments inherit downward.
- A subscription can belong to **only one management group** at a time.

### Hierarchy design guidance
Design management groups based on stable business controls, not temporary projects.

| Pattern | Best When | Trade-off |
|---|---|---|
| **By environment** | Separate prod/non-prod controls | Can become repetitive across business units |
| **By business unit** | Decentralized ownership and chargeback | Harder to enforce global environment standards |
| **By geography** | Data residency / sovereignty matters | Can duplicate shared controls |
| **Hybrid model** | Large enterprises need both global and local controls | More design complexity |

### Common design patterns
1. **Environment-first**
   - Platform
   - Production
   - NonProduction
   - Sandbox

2. **Business-unit-first**
   - Corp
   - Finance
   - Retail
   - Manufacturing

3. **Geography-first**
   - Americas
   - EMEA
   - APAC

### Real-world hierarchy examples

#### Example A: Enterprise-scale hybrid hierarchy
```text
Tenant Root MG
├─ Platform
│  ├─ Identity
│  ├─ Connectivity
│  └─ Management
├─ LandingZones
│  ├─ Corp
│  │  ├─ Prod
│  │  └─ NonProd
│  └─ Online
│     ├─ Prod
│     └─ NonProd
├─ Sandbox
└─ Decommissioned
```

#### Example B: Geography + regulation driven
```text
Tenant Root MG
├─ Global
│  ├─ SharedServices
│  └─ Security
├─ EU
│  ├─ Prod
│  └─ NonProd
├─ US
│  ├─ Prod
│  └─ NonProd
└─ APAC
   ├─ Prod
   └─ NonProd
```

### Moving subscriptions between management groups
- Supported, but inherited controls change immediately after the move.
- Validate:
  - RBAC assignments that should continue to apply
  - Policy assignments and exemptions
  - Budget/reporting model
  - Landing zone dependencies

### Azure CLI
```bash
# List management groups
az account management-group list -o table

# Create a management group
az account management-group create --name Corp-Prod --display-name "Corp Production"

# Move a subscription into a management group
az account management-group subscription add \
  --name Corp-Prod \
  --subscription <subscription-id>
```

### PowerShell
```powershell
# List management groups
Get-AzManagementGroup

# Create a management group
New-AzManagementGroup -GroupName "Corp-Prod" -DisplayName "Corp Production"

# Move subscription to a management group
New-AzManagementGroupSubscription -GroupName "Corp-Prod" -SubscriptionId "<subscription-id>"
```

> **Exam tip:** Use management groups when the scenario says **"apply the same governance across many subscriptions"**.

---

<a id="4-subscriptions"></a>
## 4. Subscriptions

A subscription is both a **billing boundary** and a **governance boundary** for quotas, RBAC, policies, and cost reporting.

### Subscription design patterns

| Pattern | Use When | Risk |
|---|---|---|
| **Single subscription** | Small org, simple governance | Weak isolation, quota contention |
| **Multiple subscriptions by environment** | Prod vs non-prod separation | More operational overhead |
| **Multiple subscriptions by app/platform** | Clear ownership and blast-radius isolation | More landing zone standardization needed |
| **Multiple subscriptions by region/compliance** | Regulatory isolation required | Duplication of shared services |

### Subscription limits and quotas
- Azure service quotas are often enforced at the **subscription + region** level.
- Common reasons to split subscriptions:
  - Quota isolation
  - Billing separation
  - Administrative boundary
  - Compliance boundary
  - Blast-radius reduction
- Do **not** create extra subscriptions just because one team wants folders; use **resource groups** when isolation requirements are lighter.

### Subscription types
- **Enterprise Agreement (EA)**
- **Microsoft Customer Agreement (MCA)**
- **Cloud Solution Provider (CSP)**
- **Pay-As-You-Go (PAYG)**
- **Visual Studio / Dev/Test**
- **Free / Trial**

### When to create a new subscription
Create a new subscription when you need:
- Different **billing ownership**
- Different **quota pool**
- Separate **production boundary**
- Stronger **policy/RBAC segmentation**
- Separate **regulatory or sovereign handling**

Avoid creating a new subscription when the need is only:
- Different app components with same lifecycle
- Basic tagging/reporting differences
- Small team isolation that a resource group can handle

### Subscription vending machine pattern
A **subscription vending machine** automates the creation of governed subscriptions with:
- Predefined management group placement
- Policy assignments
- Diagnostic settings
- Budget/tag defaults
- Standard RBAC groups
- Networking integration

This is a core enterprise-scale pattern for reducing manual variance.

### Azure CLI
```bash
# Show current subscription
az account show -o table

# List subscriptions available to the signed-in identity
az account list --all -o table
```

### PowerShell
```powershell
# List subscriptions
Get-AzSubscription

# Set active subscription context
Set-AzContext -SubscriptionId "<subscription-id>"
```

> **Exam tip:** If the requirement is **billing, quota, or strong isolation**, think **new subscription**.

---

<a id="5-resource-groups"></a>
## 5. Resource Groups

A resource group is the primary **lifecycle and management boundary** inside a subscription.

### Lifecycle alignment principle
Group resources that:
- Share the same lifecycle
- Are deployed together
- Are retired together
- Are managed by the same team

**Bad design:** Put unrelated apps in one resource group just because they are in the same department.

### Naming conventions
Use predictable names that encode business meaning.

**Example:**
- `rg-payments-prod-eastus`
- `rg-shared-networking-hub`
- `rg-hr-nonprod-weu`

Recommended naming elements:
- Resource type prefix (`rg`)
- Workload or platform name
- Environment
- Region

### Tagging strategies at RG level
Apply baseline tags at the RG level, such as:
- `CostCenter`
- `Owner`
- `Environment`
- `Application`
- `Criticality`
- `DataClassification`

### Resource group vs resource-level RBAC

| Scope | Use When | Guidance |
|---|---|---|
| **Resource Group** | App team manages workload resources together | Default and preferred for most workload teams |
| **Resource** | Only one resource needs special delegation | Use sparingly; increases admin complexity |

### Azure CLI
```bash
# Create a resource group with tags
az group create \
  --name rg-payments-prod-eastus \
  --location eastus \
  --tags Environment=Prod Owner=PaymentsTeam CostCenter=FIN001
```

### PowerShell
```powershell
New-AzResourceGroup -Name "rg-payments-prod-eastus" -Location "EastUS" -Tag @{Environment="Prod";Owner="PaymentsTeam";CostCenter="FIN001"}
```

> **Architect rule:** Use the resource group as the default delegation unit for application teams.

---

<a id="6-azure-rbac-role-based-access-control"></a>
## 6. Azure RBAC (Role-Based Access Control)

Azure RBAC controls **who can do what on Azure resources**.

### RBAC vs Microsoft Entra ID roles

| Control Plane | Purpose | Example |
|---|---|---|
| **Azure RBAC** | Manage Azure resources | Reader on a subscription |
| **Microsoft Entra ID roles** | Manage directory objects and identity settings | Global Administrator, User Administrator |

**Use Azure RBAC** for Azure resources.  
**Use Entra ID roles** for tenant/directory administration.

### Built-in roles deep dive

| Role | What it can do | What it cannot do |
|---|---|---|
| **Owner** | Full resource management + can assign RBAC | No automatic Entra ID admin rights |
| **Contributor** | Manage resources | Cannot assign RBAC |
| **Reader** | View resources | Cannot modify |
| **User Access Administrator** | Manage access assignments | Does not manage resources by itself |

### Custom roles — when and how to create
Create a custom role when:
- Built-in roles are too broad
- A team needs a tightly scoped set of actions
- You need least privilege for a platform process or automation

Do **not** create custom roles if a built-in role already fits.

### Role assignment scope hierarchy
```text
Management Group
└─ Subscription
   └─ Resource Group
      └─ Resource
```
Assignments inherit downward.

### Deny assignments
- Deny assignments block actions even if RBAC would allow them.
- Commonly created by Azure-managed systems such as **deployment stacks** and historically **Blueprints**.
- Think of deny assignments as a hard prevention layer.

### RBAC best practices
- Prefer **groups over direct user assignments**.
- Apply **least privilege**.
- Assign at the **highest appropriate scope**, not the highest possible scope.
- Use **Privileged Identity Management (PIM)** for just-in-time elevation.
- Review stale assignments regularly.

### Common role patterns

| Scenario | Recommended Pattern |
|---|---|
| App team manages its workload | Contributor at RG + Reader on shared monitoring if needed |
| Security team audits everything | Reader or Security Reader at MG/subscription |
| Platform team manages RBAC only | User Access Administrator at required scope |
| Break-glass operational admin | PIM-enabled Owner or Contributor, time-bound |
| App needs Key Vault secrets | Managed identity + Key Vault data-plane role |

### Azure CLI
```bash
# Assign Reader to a group at resource group scope
az role assignment create \
  --assignee <group-object-id> \
  --role Reader \
  --scope /subscriptions/<sub-id>/resourceGroups/rg-payments-prod-eastus

# List role assignments for a principal
az role assignment list --assignee <group-object-id> -o table

# Create a custom role from JSON
az role definition create --role-definition ./custom-role.json
```

**Sample custom role definition**
```json
{
  "Name": "VM Operator - Restart Only",
  "IsCustom": true,
  "Description": "Can read VMs and restart them.",
  "Actions": [
    "Microsoft.Compute/virtualMachines/read",
    "Microsoft.Compute/virtualMachines/restart/action",
    "Microsoft.Compute/virtualMachines/start/action",
    "Microsoft.Compute/virtualMachines/powerOff/action"
  ],
  "NotActions": [],
  "AssignableScopes": [
    "/subscriptions/<sub-id>"
  ]
}
```

### PowerShell
```powershell
# Assign a role
New-AzRoleAssignment -ObjectId "<group-object-id>" -RoleDefinitionName "Reader" -ResourceGroupName "rg-payments-prod-eastus"

# Create a custom role
New-AzRoleDefinition -InputFile "./custom-role.json"
```

> **Exam tip:** If the scenario is about **controlling access**, think **RBAC**. If it is about **controlling allowed configuration**, think **Policy**.

---

<a id="7-azure-policy"></a>
## 7. Azure Policy

Azure Policy governs **what is allowed, required, appended, modified, audited, or auto-deployed**.

### Policy vs RBAC — decision matrix

| Requirement | Use RBAC | Use Policy |
|---|---:|---:|
| Control who can create/update/delete | ✅ |  |
| Restrict allowed regions/SKUs |  | ✅ |
| Require tags |  | ✅ |
| Auto-deploy diagnostics |  | ✅ |
| Audit non-compliance |  | ✅ |
| Let team manage only one RG | ✅ |  |

### Policy definition structure
Core JSON elements:
- `mode`
- `parameters`
- `policyRule`
- `if` / `then`
- `effect`

**Sample policy definition**
```json
{
  "mode": "Indexed",
  "parameters": {
    "allowedLocations": {
      "type": "Array",
      "metadata": {
        "displayName": "Allowed locations"
      }
    }
  },
  "policyRule": {
    "if": {
      "not": {
        "field": "location",
        "in": "[parameters('allowedLocations')]"
      }
    },
    "then": {
      "effect": "deny"
    }
  }
}
```

### Policy effects

| Effect | Purpose | Typical Use |
|---|---|---|
| **Audit** | Log non-compliance | Visibility without blocking |
| **Deny** | Block non-compliant create/update | Allowed locations, approved SKUs |
| **Append** | Add fields during request | Legacy metadata injection |
| **Modify** | Add or change properties/tags | Inherit or enforce tags |
| **DeployIfNotExists** | Deploy related resources after evaluation | Diagnostic settings, agents |
| **AuditIfNotExists** | Audit when related config is missing | Diagnostics/backup missing |
| **Disabled** | Turn off effect | Testing / staged rollout |

### Policy initiatives
An **initiative** is a group of policies managed as one package, such as:
- Corporate tagging baseline
- Security baseline
- Regulatory control set

### Built-in policies vs custom policies
- **Built-in:** Start here first; Microsoft maintains them.
- **Custom:** Use when built-ins do not satisfy a business requirement.

### Policy assignment and scope
Policies can be assigned at:
- Management group
- Subscription
- Resource group
- Individual resource

### Exemptions
Use exemptions for governed exceptions.

| Type | Meaning |
|---|---|
| **Waiver** | Business accepts temporary or approved non-compliance |
| **Mitigated** | Requirement is addressed by an alternate compensating control |

Best practice: include **owner, reason, approval, and expiration date**.

### Remediation tasks
- Used after `Modify` or `DeployIfNotExists` assignments.
- Often require a **managed identity** on the policy assignment.
- Important for bringing existing resources into compliance.

### Compliance dashboard
Use Azure Policy compliance views to answer:
- Which assignments are failing?
- Which resources are non-compliant?
- Which subscriptions are drifting most?

### Policy as code
Treat policy definitions, assignments, initiatives, and exemptions as source-controlled artifacts.
- Store JSON/Bicep in GitHub.
- Use CI/CD to promote policy changes.
- Separate **authoring**, **testing**, and **production assignment**.

### Common policy patterns
- Allowed locations
- Required tags
- Inherit tag from resource group
- Allowed VM sizes / SKUs
- Require HTTPS/TLS minimum version
- Enforce diagnostic settings
- Restrict public IP creation

### Azure CLI
```bash
# Create a custom policy definition
az policy definition create \
  --name allowed-locations-custom \
  --display-name "Allowed locations" \
  --rules ./allowed-locations.json \
  --mode Indexed

# Assign policy at subscription scope
az policy assignment create \
  --name enforce-allowed-locations \
  --policy allowed-locations-custom \
  --scope /subscriptions/<sub-id> \
  --params '{"allowedLocations":{"value":["eastus","westus2"]}}'

# Trigger a compliance scan
az policy state trigger-scan --subscription <sub-id>
```

### PowerShell
```powershell
# Create a custom policy definition
$definition = New-AzPolicyDefinition -Name "allowed-locations-custom" -DisplayName "Allowed locations" -Policy "./allowed-locations.json" -Mode Indexed

# Assign policy
New-AzPolicyAssignment -Name "enforce-allowed-locations" -PolicyDefinition $definition -Scope "/subscriptions/<sub-id>"
```

### Senior architect guidance
- Use **Deny** only when you are confident operations are ready.
- Start with **Audit**, then move to **Modify/DeployIfNotExists**, then **Deny** for mature controls.
- Assign broad standards at management group level; assign local exceptions lower.

---

<a id="8-azure-blueprints"></a>
## 8. Azure Blueprints

Azure Blueprints historically packaged governance artifacts for repeatable environment deployment.

### Blueprint vs ARM template vs Policy

| Tool | Best For | Limitation |
|---|---|---|
| **Blueprints** | Packaging policies, RBAC, templates, and RGs together | Being deprecated |
| **ARM/Bicep templates** | Resource deployment | Not a governance framework by itself |
| **Azure Policy** | Ongoing compliance and enforcement | Does not fully deploy application architecture |

### Blueprint artifacts
- Policy assignments
- RBAC assignments
- ARM templates
- Resource group creation

### Blueprint versioning and publishing
- Blueprints are authored, then **published as a version**.
- Assignments consume published versions.
- Versioning helped control rollout across subscriptions.

### Blueprint assignment and locking
Blueprint assignments could apply locks to protect deployed artifacts.

### Sequencing order
For exam purposes, think in dependency order:
1. Create target resource groups if needed
2. Apply governance artifacts such as policy/RBAC
3. Deploy template artifacts into the governed scope

### Current guidance
> **Important:** Azure Blueprints are being deprecated. For modern design, prefer:
- **Bicep/Terraform** for IaC
- **Template Specs** for reusable templates
- **Deployment Stacks** for lifecycle management and protection
- **Azure Policy** for ongoing compliance

> **AZ-305 note:** You may still see Blueprint comparison questions. Know the concept, but recommend modern replacements.

---

<a id="9-resource-locks"></a>
## 9. Resource Locks

Resource locks prevent accidental change or deletion.

### Lock types

| Lock | Effect |
|---|---|
| **CanNotDelete** | Resource can be modified but not deleted |
| **ReadOnly** | Resource cannot be modified or deleted |

### Lock inheritance
- Locks applied at subscription, RG, or resource level inherit downward.
- The most restrictive effective control wins.

### When to use locks vs Policy

| Need | Use |
|---|---|
| Prevent accidental deletion of a critical resource | **Lock** |
| Restrict creation of non-approved resources | **Policy** |
| Enforce required configuration | **Policy** |
| Protect a production RG from deletion | **CanNotDelete lock** |

### Who can manage locks
Users need permissions for `Microsoft.Authorization/locks/*` at the target scope. Typically:
- **Owner**
- **User Access Administrator** (with appropriate rights)
- Custom roles containing lock actions

### Azure CLI
```bash
# Add a delete lock to a resource group
az lock create \
  --name lock-rg-prod \
  --lock-type CanNotDelete \
  --resource-group rg-payments-prod-eastus
```

### PowerShell
```powershell
New-AzResourceLock -LockName "lock-rg-prod" -LockLevel CanNotDelete -ResourceGroupName "rg-payments-prod-eastus"
```

> **Exam tip:** Locks do not replace RBAC or Policy. They are for **accidental change protection**.

---

<a id="10-tags"></a>
## 10. Tags

Tags are lightweight metadata, but they are essential for governance and cost operations.

### Tagging strategy design
Build a tagging strategy that is:
- Mandatory for chargeback/showback
- Standardized across business units
- Enforced through Policy
- Aligned to reporting and automation needs

### Common required tags
- `CostCenter`
- `Owner`
- `Environment`
- `Application`

Optional high-value tags:
- `BusinessUnit`
- `Criticality`
- `DataClassification`
- `ServiceTier`
- `ManagedBy`

### Important exam fact
**Tags do not inherit automatically from a resource group to resources.** Use **Azure Policy Modify** or a built-in inheritance policy.

### Tag enforcement with Policy
Typical controls:
- Require specific tags at creation time
- Inherit tag from RG to resource
- Append default tag values
- Deny deployment if mandatory tags are missing

### Cost management and tags
Tags enable:
- Chargeback/showback
- Cost filtering by app, BU, environment
- Reserved instance coverage analysis by owning team

### Automation with tags
Use tags for:
- Backup selection
- Start/stop schedules
- Retention classification
- CMDB synchronization
- Incident routing

### Azure CLI
```bash
# Update tags on a resource group
az tag update \
  --resource-id /subscriptions/<sub-id>/resourceGroups/rg-payments-prod-eastus \
  --operation Merge \
  --tags CostCenter=FIN001 Owner=PaymentsTeam Environment=Prod Application=Payments
```

### PowerShell
```powershell
Update-AzTag -ResourceId "/subscriptions/<sub-id>/resourceGroups/rg-payments-prod-eastus" -Operation Merge -Tag @{CostCenter="FIN001";Owner="PaymentsTeam";Environment="Prod";Application="Payments"}
```

---

<a id="11-cost-optimization"></a>
## 11. Cost Optimization

Cost optimization is a governance outcome, not a separate discipline. Good governance makes the **right architecture the default** and makes waste visible early.

### Governance Levers That Reduce Cost

| Lever | How It Helps | AZ-305 Design Angle |
|---|---|---|
| **Tags** | Enable chargeback, showback, and owner accountability | Cost data without ownership metadata is weak |
| **Budgets and alerts** | Surface overspend before it becomes material | Alerts support operational response, not hard enforcement |
| **Azure Policy** | Restricts non-approved SKUs, regions, and deployment patterns | Guardrails reduce architectural drift and shadow cost |
| **Advisor** | Finds idle or oversized resources | Useful after deployment for optimization loops |
| **Reservation strategy** | Lowers steady-state compute/database spend | Best for predictable production baselines |
| **Subscription design** | Separates environments and spending models | Better accountability and reporting clarity |

Governance is incomplete if you cannot control and explain spend.

### Budgets and alerts
Use budgets to:
- Alert at planned spend thresholds
- Notify app owners and finance teams
- Trigger governance workflows before overspend becomes material

### Cost allocation with tags
Best practice:
- Make `CostCenter`, `Owner`, `Environment`, and `Application` standard
- Apply at RG creation
- Enforce via Policy
- Audit missing tags weekly

### Azure Advisor cost recommendations
Use Advisor for:
- Idle/unattached resources
- Rightsizing opportunities
- Reserved instance recommendations
- Underutilized compute/database services

### Reserved instances governance
Governance questions to ask:
- Which team owns reservation strategy?
- Is reservation scope **shared** or **single subscription**?
- How are savings tracked and charged back?
- Are workloads stable enough for 1-year or 3-year commitments?

### Spending limits
Spending limits are useful in some subscription models, especially for dev/test control, but enterprise governance usually relies more on:
- Budgets
- Alerts
- Policy guardrails
- Subscription ownership controls

> **Architect view:** Cost governance is a blend of **tags, budgets, Advisor, rightsizing, reservation strategy, and landing zone standards**.

---

<a id="12-availability--resilience"></a>
## 12. Availability & Resilience

Governance architecture must stay effective during outages, admin mistakes, and regional disruptions. AZ-305 expects you to connect resilience decisions with identity, management, and operational controls.

### Governance Resilience Design Areas

| Area | Design Guidance | Why It Matters |
|---|---|---|
| **Hierarchy resilience** | Keep management group design stable and simple | Reduces emergency rework during large-scale incidents |
| **Access resilience** | Use PIM, break-glass accounts, and group-based RBAC | Ensures emergency access without permanent privilege |
| **Monitoring resilience** | Send Activity Log and diagnostics to durable central stores | Audit and investigation data must survive incidents |
| **Regional resilience** | Align allowed-region policies with paired-region or multi-region strategy | Governance should not block approved failover paths |
| **Operational protection** | Apply locks to critical shared services and production scopes | Reduces accidental outage amplification |

### Design Patterns

- **Break-glass identity pattern:** Keep tightly controlled emergency accounts outside normal day-to-day workflows and monitor every use.
- **Policy-aware failover pattern:** If workloads may fail over to a secondary region, include that region in allowed-location policy design up front.
- **Central logging pattern:** Route Activity Log, platform diagnostics, and policy compliance data to centralized Log Analytics, Storage, or Event Hub destinations.
- **Shared services protection pattern:** Add locks and narrow RBAC around shared connectivity, identity, DNS, and monitoring resources because outages there have broad blast radius.
- **Subscription isolation pattern:** Separate sandbox, non-production, and production so experiments cannot consume resilience capacity needed by critical workloads.

### Real-World Examples

- A bank allows production workloads only in paired approved regions so failover remains compliant during a regional outage.
- A global enterprise sends Azure Activity Logs from every subscription to a central workspace and archive account for investigation continuity.
- A platform team uses PIM plus emergency access accounts so administrators can recover networking or policy issues during identity service disruption.

> **AZ-305 tip:** The resilient answer is usually the one that combines **access recovery, logging continuity, region design, and protected shared services**.

---

<a id="13-compliance--regulatory"></a>
## 13. Compliance & Regulatory

Azure governance must support both internal policy and external regulation.

### Microsoft Defender for Cloud regulatory compliance
Use Defender for Cloud to map Azure posture to frameworks such as:
- ISO 27001
- NIST
- PCI DSS
- CIS
- SOC-oriented baselines

It provides posture visibility, recommendations, and control mapping.

### Compliance Manager
Compliance Manager helps organizations track:
- Assessment status
- Improvement actions
- Evidence gathering
- Shared responsibility understanding

### Azure compliance offerings
Microsoft publishes compliance coverage for frameworks such as:
- **SOC**
- **ISO**
- **HIPAA/HITRUST**
- **FedRAMP**
- **GDPR support commitments**
- Regional/specific sovereignty-related offerings

### Audit logging requirements
At minimum, plan for:
- **Activity Log** for control-plane changes
- **Diagnostic settings** to Log Analytics / Storage / Event Hub
- Retention aligned to policy and regulation
- Centralized access for audit and security teams

### Data residency and sovereignty
Key design questions:
- Must data remain in a specific geography?
- Is cross-region replication allowed?
- Are sovereign clouds required?
- Do logs and backups have the same residency requirement as primary workloads?

> **Exam tip:** If the scenario mentions **regulatory boundary, sovereignty, or residency**, think about **management group structure, subscription placement, allowed regions policy, and landing zone design** together.

---

<a id="14-landing-zones-caf"></a>
## 14. Landing Zones (CAF)

### What is a landing zone?
A landing zone is a **preconfigured Azure environment** with foundational controls for identity, networking, governance, security, and management.

### Platform vs application landing zones

| Type | Purpose |
|---|---|
| **Platform landing zone** | Shared services such as connectivity, identity integration, management, and security tooling |
| **Application landing zone** | Workload subscription(s) where business apps are deployed under standard guardrails |

### Enterprise-scale architecture
A common CAF enterprise-scale model includes:
- Platform subscriptions for shared services
- App subscriptions for workloads
- Central governance through management groups
- Standard policy initiatives
- Logging and security baselines

### Key design areas
- **Identity** – Entra ID integration, PIM, break-glass, group strategy
- **Network** – Hub-spoke or Virtual WAN, private access patterns
- **Governance** – Policy, tags, budgets, naming, region restrictions
- **Management** – Monitoring, backup, update, diagnostics, Defender

> **AZ-305 pattern:** Landing zones are the answer when the requirement is **repeatable enterprise-scale Azure adoption with guardrails built in**.

---

<a id="15-az-305-decision-scenarios"></a>
## 15. AZ-305 Decision Scenarios

### Scenario 1: Enterprise governance hierarchy design
A global enterprise has shared networking, shared identity services, and separate prod/non-prod subscriptions for each business unit.

**Best answer:** Use a management group hierarchy with **Platform**, then **LandingZones**, then BU/environment branches.  
**Why:** Central controls stay high; workload autonomy stays lower.

### Scenario 2: Multi-team RBAC strategy
Three app teams need to manage only their own workloads. A platform team manages shared networking.

**Best answer:** Assign app teams at **resource group scope** and the platform team at **shared subscription/RG scope**. Use groups, not direct user assignments.  
**Why:** Best balance of least privilege and administrative simplicity.

### Scenario 3: Policy enforcement for compliance
The company must allow deployments only in East US and West Europe and must require cost center tags.

**Best answer:** Use **Azure Policy Deny** for locations and **Modify/Deny** for tagging.  
**Why:** RBAC cannot control allowed regions or mandatory metadata.

### Scenario 4: Tag strategy for cost allocation
Finance needs monthly cost by application owner, environment, and cost center.

**Best answer:** Standardize required tags and enforce them with Policy at management group or subscription scope.  
**Why:** Reporting quality depends on consistent metadata, not manual discipline.

### Scenario 5: Landing zone design
A company wants every new workload subscription to inherit networking, monitoring, and governance guardrails automatically.

**Best answer:** Use a **CAF landing zone + subscription vending machine** pattern.  
**Why:** It standardizes onboarding and reduces configuration drift.

### Scenario 6: Custom role creation
Operations staff need to restart VMs but must not create or delete resources.

**Best answer:** Create a **custom RBAC role** with read/start/restart/power-off actions only.  
**Why:** Built-in Contributor is too broad.

### Scenario 7: Blueprint vs template decision
A legacy question asks how to package policy, RBAC, and templates together for standardized environment rollout.

**Best answer:** Historically **Azure Blueprints**; for current design recommend **Bicep/Template Specs + Policy + Deployment Stacks**.  
**Why:** Know both exam history and current platform direction.

### Scenario 8: Subscription organization
A company has frequent quota collisions between dev/test and production compute workloads.

**Best answer:** Separate **prod** and **non-prod** into different subscriptions.  
**Why:** Subscription quotas and governance boundaries are cleaner.

### Scenario 9: Protecting production resources
A critical production resource group must never be deleted accidentally, even by experienced admins.

**Best answer:** Apply a **CanNotDelete lock** at the RG scope.  
**Why:** Policy does not directly replace lock behavior for delete protection.

### Scenario 10: Regulatory residency requirement
A European workload must only deploy resources in approved EU regions and must store logs in-region.

**Best answer:** Combine **allowed locations Policy**, EU-focused subscription placement, and landing zone design aligned to residency.  
**Why:** Residency is a design pattern, not one toggle.

---

<a id="16-quick-reference-trigger-table"></a>
## 16. Quick Reference Trigger Table

| If the scenario says... | Think... |
|---|---|
| Apply the same controls across many subscriptions | Management groups |
| Separate prod and non-prod strongly | Separate subscriptions |
| Team manages one app only | Resource group scope RBAC |
| Control who can do something | RBAC |
| Control what can be deployed | Azure Policy |
| Require approved Azure regions only | Policy Deny |
| Require tags on all resources | Policy Modify/Deny |
| Copy tag from RG to resource | Policy Modify / built-in inherit tag policy |
| Prevent accidental deletion | Resource lock |
| Need enterprise-scale onboarding | Landing zone + subscription vending machine |
| Need repeatable environment governance package (legacy) | Blueprints concept |
| Need modern replacement for Blueprint | Deployment Stacks + Template Specs + Policy |
| Need least privilege beyond built-in roles | Custom RBAC role |
| Need JIT admin access | PIM |
| Need audit without blocking | Policy Audit |
| Need deploy missing diagnostics automatically | DeployIfNotExists |
| Need check if diagnostics are missing | AuditIfNotExists |
| Need shared services governance | Platform management group / platform landing zone |
| Need chargeback/showback | Tags + Cost Management |
| Need business-approved exception | Policy exemption |
| Need compensating control exception | Exemption type = Mitigated |
| Need accepted temporary risk | Exemption type = Waiver |
| Need blast-radius reduction | More subscriptions |
| Need quota isolation | More subscriptions |
| Need resource lifecycle grouping | Resource groups |
| Need directory admin rights | Entra ID roles |
| Need Azure resource admin rights | Azure RBAC |
| Need deny even if RBAC allows | Deny assignment |
| Need central compliance dashboard | Azure Policy + Defender for Cloud |
| Need enterprise architecture foundation | CAF landing zones |
| Need standard role assignment model | Groups over users |
| Need region-specific sovereignty controls | Geography-aligned MG/subscription design |
| Need cost optimization insights | Azure Advisor |
| Need standard naming/tags/security baseline | Policy initiative |
| Need resource deletion protection in prod | CanNotDelete lock |
| Need immutable governance at higher scope | Assign high in hierarchy |

---

<a id="17-common-exam-traps"></a>
## 17. Common Exam Traps

### 1. Policy Deny vs RBAC deny
- **Azure Policy Deny** blocks resource configurations/requests based on rules.
- **Deny assignment** blocks actions even if RBAC allows them.
- They are not the same feature.

### 2. Management group depth limits
- The hierarchy supports **up to 6 management group levels below the tenant root management group**.
- Exam questions often try to confuse root, MG levels, and subscription placement.

### 3. Custom role scope limitations
- Custom roles can only be assigned within their **AssignableScopes**.
- If you want reuse across many subscriptions, define them high enough (often management group scope).

### 4. Blueprint deprecation status
- Know what Blueprints do.
- Also know Microsoft is steering customers toward **Bicep/ARM + Template Specs + Deployment Stacks + Policy**.

### 5. Tag inheritance misconceptions
- Tags on a resource group do **not** automatically flow to child resources.
- Use **Azure Policy** to inherit or enforce them.

### 6. Subscription vs resource group confusion
- Subscription = billing/quota/governance boundary.
- Resource group = lifecycle/management boundary.

### 7. Owner role misconception
- **Owner** can manage access and resources, but that does not mean unlimited tenant-level authority in Microsoft Entra ID.

### 8. Resource locks misconception
- Locks protect against accidental change/deletion.
- They do not replace RBAC, PIM, or Policy.

### 9. Policy effect misuse
- Use **Audit** first for discovery.
- Use **Deny** only when the organization is ready for hard enforcement.

### 10. One-tool thinking
- Real solutions usually combine multiple governance layers:
  - Management groups
  - Subscriptions
  - RBAC
  - Policy
  - Locks
  - Tags
  - Cost controls
  - Monitoring/compliance

---

<a id="18-governance-command-reference"></a>
## 18. Governance Command Reference

### Management groups
```bash
az account management-group list -o table
az account management-group show --name <mg-name>
az account management-group subscription add --name <mg-name> --subscription <subscription-id>
```

```powershell
Get-AzManagementGroup
Get-AzManagementGroup -GroupName "<mg-name>" -Expand
New-AzManagementGroupSubscription -GroupName "<mg-name>" -SubscriptionId "<subscription-id>"
```

### RBAC
```bash
az role assignment create --assignee <principal-id> --role Reader --scope /subscriptions/<sub-id>
az role assignment list --scope /subscriptions/<sub-id> -o table
az role definition create --role-definition ./custom-role.json
```

```powershell
New-AzRoleAssignment -ObjectId "<principal-id>" -RoleDefinitionName "Reader" -Scope "/subscriptions/<sub-id>"
Get-AzRoleAssignment -Scope "/subscriptions/<sub-id>"
New-AzRoleDefinition -InputFile "./custom-role.json"
```

### Policy
```bash
az policy definition create --name <policy-name> --rules ./policy.json --mode Indexed
az policy assignment create --name <assignment-name> --policy <policy-name> --scope /subscriptions/<sub-id>
az policy state summarize --subscription <sub-id>
az policy state trigger-scan --subscription <sub-id>
```

```powershell
New-AzPolicyDefinition -Name "<policy-name>" -Policy "./policy.json" -Mode Indexed
New-AzPolicyAssignment -Name "<assignment-name>" -PolicyDefinition (Get-AzPolicyDefinition -Name "<policy-name>") -Scope "/subscriptions/<sub-id>"
Get-AzPolicyStateSummary -SubscriptionId "<sub-id>"
Start-AzPolicyComplianceScan -SubscriptionId "<sub-id>"
```

### Locks and tags
```bash
az lock create --name lock-rg-prod --lock-type CanNotDelete --resource-group rg-payments-prod-eastus
az tag update --resource-id /subscriptions/<sub-id>/resourceGroups/rg-payments-prod-eastus --operation Merge --tags Environment=Prod
```

```powershell
New-AzResourceLock -LockName "lock-rg-prod" -LockLevel CanNotDelete -ResourceGroupName "rg-payments-prod-eastus"
Update-AzTag -ResourceId "/subscriptions/<sub-id>/resourceGroups/rg-payments-prod-eastus" -Operation Merge -Tag @{Environment="Prod"}
```

---

<a id="19--final-az-305-exam-tips"></a>
## 🎯 Final AZ-305 Exam Tips

1. **Start with scope.** Ask whether the requirement belongs at management group, subscription, resource group, or resource level.
2. **Separate identity from compliance.** Use RBAC for access and Policy for configuration enforcement.
3. **Choose subscriptions for strong isolation.** Billing, quota, residency, and blast-radius requirements usually point here.
4. **Assign high, but not too high.** Put controls at the highest appropriate scope without over-delegating authority.
5. **Use Policy rollout maturity.** Audit first, then remediate, then deny when operations are ready.
6. **Remember tags need enforcement.** Reporting quality depends on consistent metadata, not manual discipline.
7. **Treat locks as safety rails.** They protect from accidents, not from poor authorization design.
8. **Know the Blueprint story.** Blueprints are legacy exam knowledge; modern answers favor Policy + IaC + Deployment Stacks.
9. **Design for operations, not just deployment.** Monitoring, diagnostics, ownership, and exemptions matter in enterprise governance.
10. **Pick the scalable answer.** AZ-305 rewards designs that work across many teams and subscriptions, not one-off fixes.

### Memory Aids

- **Access problem?** RBAC.  
- **Compliance/configuration problem?** Policy.  
- **Accidental deletion problem?** Lock.  
- **Enterprise-wide inheritance?** Management group.  
- **Billing/quota/isolation problem?** Subscription.  
- **Lifecycle/delegation problem?** Resource group.  
- **Metadata/cost allocation problem?** Tags.  
- **Standardized governed foundation?** Landing zone.

> **Senior architect takeaway:** In AZ-305, the best answer is usually the one that scales operationally across teams, not the one that solves only today’s ticket.

---

<a id="20--architecture-decision-flowchart"></a>
## 📐 Architecture Decision Flowchart

```text
Need to design Azure governance for an enterprise workload
│
├─ Multiple subscriptions involved?
│  ├─ YES → Start with Management Group hierarchy
│  │        ├─ Need prod/non-prod separation? → Split subscriptions
│  │        ├─ Need geography/regulatory separation? → Add region/compliance branches
│  │        └─ Need shared services? → Create platform branch / platform subscriptions
│  └─ NO → Start with Subscription design + Resource Group delegation
│
├─ Need to control access? → RBAC + groups + PIM
├─ Need to control allowed configuration? → Azure Policy + initiatives + exemptions
├─ Need deletion protection? → Resource Locks
├─ Need cost visibility? → Tags + Budgets + Advisor
├─ Need repeatable onboarding? → Landing Zone + Subscription Vending
└─ Need resilient operations? → Central logging + approved failover regions + break-glass access
```

---

<a id="21-exam-style-review-questions"></a>
## Exam-Style Review Questions

1. **A company wants every production subscription to inherit the same region restrictions, diagnostic settings, and security baseline. What should you design first?**  
   **Answer:** A management group hierarchy with policy assignments at the appropriate parent scope.

2. **An application team must manage only its own workload resources, while the platform team manages shared networking. What is the best access model?**  
   **Answer:** Assign the app team RBAC at resource group scope and keep platform roles at the shared platform scope.

3. **A business requires cost reporting by owner, environment, and cost center, but teams often forget metadata during deployment. What should you recommend?**  
   **Answer:** Standard tags enforced through Azure Policy, with budgets and reporting aligned to those tags.

4. **A critical production resource group must not be deleted accidentally, even by experienced administrators. What control best fits?**  
   **Answer:** A `CanNotDelete` lock at the resource group scope.

5. **An exam scenario asks for a repeatable, enterprise-scale Azure foundation with built-in identity, networking, policy, and monitoring guardrails. What is the best design pattern?**  
   **Answer:** A CAF landing zone architecture with subscription vending, policy initiatives, and group-based RBAC.

---

**Azure Governance - AZ-305 Comprehensive Cheat Sheet** · Focus on scalable design decisions, least privilege, and operational guardrails.

[Back to top](#table-of-contents)
