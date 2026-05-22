# Azure Governance – AZ-305 Cheat Sheet + Exam Prep

> 📝 **Hands-On Labs:** [Governance Labs](./Labs/Azure-Governance-Labs.md)
> 
> **Perspective:** Senior Cloud Solution Architect preparing for AZ-305  
> **Focus:** Governance design, enforcement, operating model, and exam decision patterns

---

## 1. Azure Governance Overview

Azure governance is how you keep a cloud estate **secure, cost-controlled, compliant, and consistent at scale**.

### Why governance matters
- **Cost:** Prevent sprawl, enforce tagging, allocate spend, and control SKU usage.
- **Security:** Limit access, reduce privilege, enforce encryption, and require diagnostics.
- **Compliance:** Apply standards consistently across subscriptions and teams.
- **Consistency:** Standardize naming, regions, network patterns, and deployment guardrails.

### Governance hierarchy

```text
Tenant Root Management Group
└─ Management Groups
   └─ Subscriptions
      └─ Resource Groups
         └─ Resources
```

**Design principle:** Govern high in the hierarchy when the rule is enterprise-wide; govern lower when teams need autonomy.

### CAF governance disciplines
The **Cloud Adoption Framework (CAF)** governance model commonly focuses on:
- **Cost management**
- **Security baseline**
- **Resource consistency**
- **Identity baseline**
- **Deployment acceleration**

> **AZ-305 lens:** Governance is not one feature. It is the combined use of **management groups, subscriptions, RBAC, Policy, tags, cost controls, landing zones, and monitoring**.

---

## 2. Management Groups

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

## 3. Subscriptions

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

## 4. Resource Groups

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

## 5. Azure RBAC (Role-Based Access Control)

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

## 6. Azure Policy

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

## 7. Azure Blueprints

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

## 8. Resource Locks

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

## 9. Tags

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

## 10. Cost Management & Governance

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

## 11. Compliance & Regulatory

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

## 12. Landing Zones (CAF)

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

## 13. AZ-305 Decision Scenarios

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

## 14. Quick Reference Trigger Table

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

## 15. Common Exam Traps

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

## Governance Command Reference

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

## Final AZ-305 Memory Aids

- **Access problem?** RBAC.  
- **Compliance/configuration problem?** Policy.  
- **Accidental deletion problem?** Lock.  
- **Enterprise-wide inheritance?** Management group.  
- **Billing/quota/isolation problem?** Subscription.  
- **Lifecycle/delegation problem?** Resource group.  
- **Metadata/cost allocation problem?** Tags.  
- **Standardized governed foundation?** Landing zone.

> **Senior architect takeaway:** In AZ-305, the best answer is usually the one that scales operationally across teams, not the one that solves only today’s ticket.
