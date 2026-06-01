# 🔐 AZ-305: Design Identity, Governance, and Monitoring Solutions

> **Study this to pass.** This is the master overview + cheat sheet + index for the AZ-305 domain covering identity, governance, and monitoring design decisions.

---

## 1. AZ-305 Exam Domain Overview

- **Exam weight:** **25–30%**
- **What Microsoft tests:** your ability to design identity solutions, build governance hierarchies, and recommend monitoring, logging, alerting, and security visibility patterns.
- **What the exam rewards:** choosing the **least-privilege**, **lowest-ops**, **policy-driven**, **well-architected** option that still meets security, compliance, and operational requirements.

### What Microsoft is really looking for

| Area | What they care about |
|------|----------------------|
| Identity | Authentication strength, external identity patterns, hybrid identity, privileged access, Zero Trust |
| Governance | Landing zone hierarchy, role design, policy enforcement, compliance scope, cost/accountability controls |
| Monitoring | Metrics vs logs, diagnostic settings, retention, alerting, APM, security operations integration |

### Key skills measured

- Design a solution for logging and monitoring
- Design authentication and authorization solutions
- Design governance
- Design identity solutions

### Expanded study focus in this cheat sheet

- Design for Microsoft Entra ID, external identities, and hybrid identity
- Design role assignment and least-privilege access models
- Design governance using management groups, subscriptions, policy, and tagging
- Design monitoring strategies with Azure Monitor, Log Analytics, alerts, and workbooks
- Design security monitoring using Defender for Cloud and Microsoft Sentinel
- Recommend workspace, retention, archival, and incident response patterns

> **Exam tip:** if the question says **secure, scalable, low-admin, least privilege**, Microsoft usually wants **Managed Identity + RBAC + Policy + monitoring by default**.

---

## 2. Services Overview Table

| Service | Category | Best For | Key Features | SLA |
|---------|----------|----------|--------------|-----|
| **Microsoft Entra ID (Azure AD)** | Identity platform | Workforce identity, SSO, app access, directory services | Users, groups, app registrations, enterprise apps, Conditional Access, MFA, PIM, hybrid identity | **99.99%** |
| **Entra ID B2B / B2C** | External identity | Partner collaboration (B2B) and customer-facing identity (B2C) | Guest access, federation, social identity, custom sign-up/sign-in journeys, tenant isolation patterns | Varies by SKU/plan |
| **Entra External ID** | External identity platform | Modern external workforce/customer identity scenarios | External tenants, partner/customer identity, flexible external access patterns, branded journeys | Varies by SKU/plan |
| **Managed Identities (system/user-assigned)** | Workload identity | Azure-hosted workloads that need Azure resource access without secrets | Automatic credential rotation, no secret storage, system-assigned and reusable user-assigned identities | Inherits host service SLA |
| **Azure RBAC** | Authorization | Controlling **who** can perform actions on Azure resources | Role assignments, built-in/custom roles, scope inheritance, least privilege, deny assignments in limited scenarios | Inherits Azure Resource Manager |
| **Azure Policy** | Governance/compliance | Controlling **what** can be deployed or required | Deny, Audit, Modify, DeployIfNotExists, initiatives, exemptions, remediation tasks, compliance view | Inherits Azure Resource Manager |
| **Azure Blueprints** | Governance orchestration | Legacy packaged governance deployments | Bundled policies, role assignments, ARM templates, locks; largely superseded by landing zones + Policy + Template Specs/Deployment Stacks | Legacy / varies |
| **Management Groups** | Governance hierarchy | Organizing subscriptions at scale | Hierarchical inheritance for Policy and RBAC, enterprise landing zones, platform vs workload segmentation | Inherits Azure Resource Manager |
| **Azure Monitor** | Monitoring platform | Central metrics, logs, alerts, dashboards, telemetry routing | Metrics, logs, alerts, diagnostic settings, autoscale hooks, workbooks integration | Varies by SKU/plan |
| **Log Analytics** | Log store / query engine | Centralized or regional operational/security log analysis | KQL, workspace-based retention, data collection, cross-resource queries, archive/search jobs | Varies by SKU/plan |
| **Application Insights** | APM | Application performance monitoring and troubleshooting | Distributed tracing, dependency maps, live metrics, failures, availability tests, OpenTelemetry integration | Varies by SKU/plan |
| **Azure Alerts** | Alerting | Proactive detection and notification | Metric alerts, log alerts, activity log alerts, smart detection, dynamic thresholds, alert processing rules | Varies by SKU/plan |
| **Action Groups** | Notification/response | Routing alert actions to people or automation | Email, SMS, push, voice, webhooks, Logic Apps, Functions, Automation runbooks, ITSM | Varies by SKU/plan |
| **Workbooks** | Visualization/reporting | Interactive observability and security dashboards | Parameterized dashboards, KQL/metrics visualization, drilldowns, Azure Monitor and Sentinel views | Varies by SKU/plan |
| **Azure Advisor** | Optimization/governance | Cost, reliability, security, and performance recommendations | Personalized recommendations, right-sizing, high availability guidance, service best practices | Varies by SKU/plan |
| **Microsoft Defender for Cloud** | CSPM/CWPP | Security posture, recommendations, workload protection, compliance | Secure Score, regulatory compliance, workload protection plans, attack path analysis, integration with Sentinel | Varies by plan |
| **Microsoft Sentinel** | SIEM/SOAR | Central security analytics, incident correlation, automation | Analytics rules, incidents, hunting, UEBA, data connectors, playbooks, SOC workflows | Varies by SKU/plan |
| **Azure Arc** | Hybrid/multicloud management | Governing and monitoring servers/Kubernetes/SQL outside Azure | Azure policy/monitoring for hybrid resources, Arc-enabled servers/K8s/data services, centralized governance | Varies by connected service |

> **Exam tip:** **Policy** and **RBAC** are not competitors. Use **RBAC for who**, **Policy for what**, and **Management Groups for scale**.

### Fast service memory map

| If you need... | Usually choose... |
|----------------|-------------------|
| Employee identity + SSO | Microsoft Entra ID |
| Partner access with their own credentials | Entra ID B2B |
| Customer sign-up/sign-in | Entra ID B2C / Entra External ID |
| Secretless Azure workload auth | Managed Identity |
| Least-privilege admin access | RBAC + PIM |
| Enforce standards across subscriptions | Management Groups + Azure Policy |
| Troubleshoot apps | Application Insights + Log Analytics |
| Detect threats and run SOC | Defender for Cloud + Sentinel |
| Govern hybrid/on-prem resources | Azure Arc |

---

## 3. Design Considerations Framework

### Identity Design

#### Authentication methods

| Method | Best Use | Strength | Exam Notes |
|--------|----------|----------|------------|
| Password only | Legacy/low-security only | Low | Usually wrong unless explicitly constrained |
| Password + MFA | Standard workforce protection | Medium-High | Common baseline answer |
| Passwordless (Microsoft Authenticator, WHfB) | Strong user experience + security | High | Great for workforce scenarios |
| **FIDO2 security keys** | Admins, high assurance, phishing-resistant auth | Very High | Best answer when exam says **phishing-resistant** |
| Certificate-based auth | Specialized enterprise/device scenarios | High | More niche; usually for legacy/regulated environments |

**Design rules:**
- For privileged roles, prefer **MFA or passwordless**, and if the question says **phishing-resistant**, think **FIDO2** or **Windows Hello for Business**.
- If the scenario emphasizes user convenience and reduced password reset cost, think **passwordless**.
- If the requirement is "no change to partner identities," think **federation/B2B**.

#### Single sign-on (SSO) patterns

| Pattern | Use When | Notes |
|--------|----------|-------|
| Cloud-only SSO | SaaS/modern apps | Simplest pattern |
| Hybrid SSO | On-prem AD + Microsoft 365/Azure apps | Use Azure AD Connect or Cloud Sync based on scenario |
| Federation | External IdP or special auth requirements | Higher complexity; use when required |
| App proxy / pre-auth | Publishing on-prem apps securely | Good for remote access without full VPN |

#### Conditional Access policies

**Core signals Microsoft expects you to know:**
- User/group
- Role
- Device compliance/join state
- Location
- Application
- Sign-in risk / user risk
- Client app type (legacy auth matters)

**Typical design pattern:**
1. Block legacy authentication
2. Require MFA for admins
3. Require compliant device for sensitive apps
4. Use named locations for risky countries or trusted networks
5. Layer with PIM for admin access

> **Exam tip:** Conditional Access is a **grant/session control engine**, not a permission model. It decides **under what conditions** access is allowed.

#### Identity Protection

Use when the requirement includes:
- Risky sign-ins
- Leaked credentials
- Automated response to sign-in risk
- User risk remediation

**Design shortcut:**
- If the question says **risk-based policies**, **leaked credentials**, or **adaptive access**, think **Identity Protection (P2)**.

#### Privileged Identity Management (PIM)

| Need | Best Design |
|------|-------------|
| Reduce standing admin access | **Eligible** rather than permanent active assignment |
| Approval for elevated access | PIM activation with approvers |
| Time-bound privilege | Just-in-time activation |
| Access attestations | Access Reviews |
| Stronger admin controls | MFA + justification + ticketing for activation |

**Default exam answer:**
- For admin roles, use **PIM Eligible**, not Active, unless continuous access is explicitly required.

#### Hybrid identity

| Option | Best For | Notes |
|-------|----------|-------|
| Azure AD Connect | Richer hybrid needs, traditional sync, complex coexistence | Most common exam answer for classic hybrid identity |
| Cloud Sync | Lighter-weight sync, simpler deployments, cloud-managed agent model | Great when the question emphasizes simplicity |
| Federation | Only when specific auth/control needs justify complexity | Avoid unless required |

**Shortcut:**
- If the scenario says **existing on-prem AD users must use same identities in Azure**, think **hybrid identity sync**.
- If it says **minimal infrastructure**, lean toward **Cloud Sync**.

#### B2B vs B2C decision criteria

| Choose | When | Avoid When |
|-------|------|------------|
| **B2B** | Partners, suppliers, vendors, guests, collaboration | You need consumer-scale sign-up/sign-in journeys |
| **B2C** | Customer-facing apps, social identities, branded sign-up | You only need partner collaboration |
| **Entra External ID** | Broader external identity modernization scenarios | The exam specifically anchors on classic B2B/B2C wording |

**One-line memory aid:**
- **B2B = business partners**
- **B2C = customers**

---

### Governance Design

#### Management group hierarchy design

**Good enterprise pattern:**

```text
Tenant Root Group
├─ Platform
│  ├─ Identity
│  ├─ Management
│  └─ Connectivity
├─ Landing Zones
│  ├─ Production
│  ├─ Non-Production
│  └─ Sandbox
└─ Decommissioned / Legacy
```

**Design guidance:**
- Put broad guardrails high in the tree.
- Put environment-specific controls lower.
- Use management groups for **inheritance at scale**.
- Avoid overly deep hierarchies unless governance boundaries require them.

> **Exam tip:** If the requirement says **apply standards to many subscriptions**, the answer is usually **assign Policy/initiatives at a Management Group**.

#### Subscription organization strategies

| Strategy | Best For | Trade-off |
|---------|----------|-----------|
| By environment | Prod vs non-prod isolation | Simple, common |
| By business unit | Chargeback, autonomy | More subscriptions to manage |
| By geography | Data sovereignty and legal boundaries | Operational duplication |
| By application/platform | Large enterprise operating models | Can grow quickly |
| Sandbox subscriptions | Innovation with tighter spend controls | Must cap risk |

#### RBAC role design

| Decision | Guidance |
|---------|----------|
| Built-in vs custom | Prefer **built-in** unless exact permissions are not available |
| Scope | Assign at the **lowest scope** that meets requirements |
| Identity target | Prefer groups over direct user assignments |
| Admin model | Combine RBAC with PIM for privileged roles |
| Exceptions | Use custom roles carefully and document them |

**Built-in vs custom shortcut:**
- Use **built-in** first for operational simplicity and lower audit burden.
- Use **custom roles** only when built-in roles are too broad or too narrow.

#### Azure Policy strategy

| Capability | Best Use |
|-----------|----------|
| Initiative | Bundle many policies for a standard or landing zone |
| Exemption | Temporary or approved exception with traceability |
| Remediation | Fix existing drift with DeployIfNotExists / Modify |
| Deny | Hard enforcement for noncompliant resources |
| Audit | Visibility first when organization is not ready to block |

**Recommended rollout pattern:**
1. Start with **Audit**
2. Measure impact
3. Add exemptions where justified
4. Move critical controls to **Deny**
5. Use remediation for at-scale correction

> **Exam tip:** If the requirement says **automatically add missing configuration**, think **DeployIfNotExists** or **Modify**, not just Audit.

#### Resource tagging strategy

**Common tags:**
- `CostCenter`
- `Owner`
- `Environment`
- `Application`
- `BusinessUnit`
- `DataClassification`
- `Criticality`

**Design tips:**
- Standardize tag names and allowed values.
- Use Policy to require tags and **Modify** to inherit from resource group when appropriate.
- Do not rely on manual tagging for enterprise governance.

#### Cost management integration

Governance is not only security; it is also accountability.

| Need | Design Choice |
|------|---------------|
| Chargeback/showback | Tags + subscription structure |
| Reduce waste | Azure Advisor + budgets + policy guardrails |
| Prevent expensive SKUs | Policy allow/deny SKU strategy |
| Team ownership | Subscription and tag alignment |

#### Compliance and regulatory requirements

If the scenario mentions:
- ISO, PCI-DSS, NIST, HIPAA, GDPR
- Auditors
- Mandatory encryption/logging/retention
- Regional data residency

Think:
- **Azure Policy initiatives**
- **Defender for Cloud regulatory compliance**
- **Management Group scoped governance**
- **Regional workspace/log placement**

#### Blueprints note

> **Important:** **Azure Blueprints is legacy/deprecated directionally for new design work.** For modern governance answers, favor **Azure landing zones**, **Management Groups**, **Azure Policy**, **RBAC**, **Template Specs**, and **Deployment Stacks**. If an exam question explicitly mentions Blueprints, understand its role, but recommend modern equivalents for new implementations.

---

### Monitoring Design

#### Metrics vs Logs decision

| Choose | When | Why |
|-------|------|-----|
| **Metrics** | Near-real-time alerting, numeric thresholds, fast detection | Lower latency, ideal for health/performance alerts |
| **Logs** | Deep investigation, correlation, auditing, security analysis | Rich context, KQL queries, flexible retention |

**Shortcut:**
- **Need alert in ~1 minute?** Think **metric alert**.
- **Need investigation, trend analysis, or security hunting?** Think **logs**.

#### Log Analytics workspace design

| Model | Best For | Strengths | Risks |
|------|----------|-----------|------|
| Centralized | SOC, shared operations, Sentinel, cross-resource analytics | Single pane of glass, easier correlation, commitment tiers | Data residency and blast-radius concerns |
| Distributed | Regional autonomy, legal boundaries, large decentralized orgs | Data sovereignty, team ownership | Harder correlation, more admin overhead |
| Hybrid | Large enterprises | Balance of local ownership + central SOC | More design complexity |

**Exam default:**
- If security team needs one view, use **centralized**.
- If the question stresses sovereignty or strict regional retention, use **distributed/regional**.
- If both appear, recommend a **hybrid** model.

#### Data retention and archival

| Requirement | Design Choice |
|------------|---------------|
| Short operational troubleshooting | Standard workspace retention |
| Long-term compliance retention | Archive / export to Storage |
| Cheap historical retention | Storage account archival patterns |
| Frequent interactive query | Keep hot data in workspace longer |

**Design rule:** keep expensive searchable data only as long as needed; archive the rest.

#### Alert design

| Design Element | Guidance |
|---------------|----------|
| Severity | Use Sev 0-4 consistently |
| Action Groups | Reuse by audience: ops, security, app team, exec |
| Suppression | Avoid alert storms with processing rules/suppression |
| Dynamic thresholds | Good for variable baselines |
| Metric vs log alert | Metric for speed, log for context |
| Escalation | Pair alerts with ITSM/automation when response matters |

#### Diagnostic settings patterns

**Recommended pattern for critical workloads:**
- Send platform/resource logs to **Log Analytics** for operational query
- Send long-term/audit copies to **Storage** if retention/compliance requires it
- Stream when needed to **Event Hubs** for SIEM/third-party tools

> **Exam tip:** If the requirement says **retain logs for 7 years cheaply**, do not keep everything in Log Analytics only. Add **Storage/archive**.

#### Application performance monitoring (APM)

Use **Application Insights** when the requirement mentions:
- Request latency
- Dependency failures
- Distributed tracing
- Availability tests
- Slow transactions
- Exception investigation

**Quick rule:**
- **App health** → Application Insights
- **Platform health** → Azure Monitor
- **Security incidents** → Sentinel / Defender

#### Security monitoring integration

| Need | Best Design |
|------|-------------|
| Posture/compliance recommendations | Defender for Cloud |
| Threat analytics and incidents | Sentinel |
| Unified SOC workflow | Central Log Analytics workspace + Sentinel connectors |
| Hybrid security visibility | Azure Arc + Defender + Monitor |

**Sentinel + Defender pattern:**
- Defender provides posture and some protection signals.
- Sentinel correlates data across sources and drives incidents/playbooks.
- Together they form a common exam-ready security architecture answer.

---

## 4. Decision Flowcharts

### Which identity solution?

```text
Start
 │
 ├─ Are the users employees/internal workforce?
 │   ├─ Yes → Microsoft Entra ID
 │   │   └─ Need hybrid with on-prem AD?
 │   │       ├─ Yes → Azure AD Connect or Cloud Sync
 │   │       └─ No  → Cloud-only Entra ID
 │   └─ No
 │
 ├─ Are they partners/suppliers/vendors using their own identities?
 │   ├─ Yes → Entra ID B2B
 │   └─ No
 │
 ├─ Are they end customers signing up to an app?
 │   ├─ Yes → Entra ID B2C / Entra External ID
 │   └─ No
 │
 └─ Need workload-to-Azure authentication?
     ├─ Azure-hosted workload → Managed Identity
     └─ External/non-Azure workload → Service Principal
```

### Managed Identity vs Service Principal?

```text
Need an app/workload to access Azure resources
 │
 ├─ Is the workload running in Azure?
 │   ├─ Yes
 │   │   ├─ One Azure resource needs identity?
 │   │   │   ├─ Yes → System-assigned Managed Identity
 │   │   │   └─ No  → User-assigned Managed Identity
 │   │   └─ Prefer Managed Identity by default (no secrets)
 │   └─ No
 │
 └─ External host, CI/CD, on-prem, multicloud runtime?
     ├─ Yes → Service Principal
     └─ If possible, use certificate or federated credential instead of client secret
```

### Azure Policy vs RBAC?

```text
What problem are you solving?
 │
 ├─ "Who can do something?"
 │   └─ Azure RBAC
 │       ├─ Assign built-in role if possible
 │       ├─ Scope to MG / Subscription / RG / Resource
 │       └─ Use PIM for privileged roles
 │
 └─ "What is allowed/required on resources?"
     └─ Azure Policy
         ├─ Deny → block noncompliant deployments
         ├─ Audit → report only
         ├─ Modify → add/fix properties/tags
         └─ DeployIfNotExists → deploy missing config
```

### Centralized vs distributed Log Analytics?

```text
Need log/workspace design
 │
 ├─ Is cross-subscription/SOC correlation the top priority?
 │   ├─ Yes → Centralized workspace
 │   └─ No
 │
 ├─ Are data residency or regulatory boundaries strict?
 │   ├─ Yes → Distributed/regional workspaces
 │   └─ No
 │
 ├─ Do different teams need autonomy and different retention models?
 │   ├─ Yes → Distributed or hybrid
 │   └─ No → Centralized
 │
 └─ Need both central security view and regional control?
     └─ Hybrid pattern (regional workspaces + central SOC/Sentinel view)
```

---

## 5. Cheat Sheet Navigation

| Topic | File | Covers |
|-------|------|--------|
| Microsoft Entra ID | [Azure-EntraID.md](./Azure-EntraID.md) | Authentication, authorization, B2B/B2C, hybrid, PIM |
| Governance | [Azure-Governance.md](./Azure-Governance.md) | RBAC, Policy, Blueprints, Management Groups, compliance |
| Monitoring & Observability | [Azure-Monitoring.md](./Azure-Monitoring.md) | Monitor, Log Analytics, Alerts, App Insights, Sentinel |

---

## 6. Labs Navigation

| Topic | File | Labs |
|-------|------|------|
| Entra ID Labs | [Azure-EntraID-Labs.md](./Labs/Azure-EntraID-Labs.md) | Conditional Access, MFA, B2B, Managed Identity, PIM |
| Governance Labs | [Azure-Governance-Labs.md](./Labs/Azure-Governance-Labs.md) | RBAC, Policy, Management Groups, Blueprints, tagging |
| Monitoring Labs | [Azure-Monitoring-Labs.md](./Labs/Azure-Monitoring-Labs.md) | Log Analytics, Alerts, Workbooks, App Insights, Defender |

---

## 7. AZ-305 Exam Tips

### Common exam traps for this domain

| Trap | Correct Thinking |
|------|------------------|
| "Need permissions" | RBAC |
| "Need enforcement/compliance" | Policy |
| "Need both" | RBAC + Policy |
| "Owners must still be blocked" | Policy or lock/deny assignment pattern |
| "Partners" | B2B |
| "Customers" | B2C / External ID |
| "Admins should not be permanent" | PIM Eligible |
| "No secret rotation burden" | Managed Identity |
| "Fast alerting" | Metrics |
| "Investigation/correlation" | Logs |
| "Security operations center" | Sentinel |
| "Posture/compliance recommendations" | Defender for Cloud |

### Decision patterns Microsoft expects

- **Least privilege** → narrowest RBAC scope, groups over users, built-in roles first, PIM for privileged access
- **Zero Trust** → verify explicitly, use Conditional Access, require MFA/passwordless, assume breach, log everything important
- **Scale** → assign governance at management group, not subscription-by-subscription
- **Compliance** → use Policy initiatives, exemptions, remediation, and Defender regulatory views
- **Operational efficiency** → prefer platform services over custom tooling; use action groups and automation for response
- **Hybrid/multicloud** → think Azure Arc when governance/monitoring must extend beyond Azure

### Quick memorization aids

| Cue | Remember |
|-----|----------|
| **WHO** can do it? | RBAC |
| **WHAT** is allowed? | Policy |
| **WHEN/HOW** can sign-in happen? | Conditional Access |
| **Just-in-time admin** | PIM |
| **Phishing-resistant** | FIDO2 / WHfB |
| **Partners** | B2B |
| **Customers** | B2C |
| **Fast alert** | Metric |
| **Deep query** | Log Analytics |
| **Threat + incident** | Sentinel |

### Trigger phrases and likely answers

| Trigger phrase | Think |
|---------------|-------|
| "phishing-resistant" | FIDO2 / Windows Hello for Business |
| "partner collaboration" | Entra ID B2B |
| "consumer sign-up" | Entra ID B2C / External ID |
| "no secrets" | Managed Identity |
| "govern all subscriptions" | Management Groups + Policy |
| "auto-remediate" | DeployIfNotExists / Modify |
| "single SOC view" | Centralized workspace + Sentinel |
| "regional data residency" | Distributed/regional workspaces |
| "improve security posture" | Defender for Cloud |
| "chargeback" | Tags + subscription strategy |

---

## 8. Quick Reference Trigger Table

| If the scenario says... | Think... | Why |
|------------------------|----------|-----|
| Employees need SSO to Microsoft 365 and SaaS apps | Microsoft Entra ID | Core workforce identity platform |
| Partners need access using their own company credentials | Entra ID B2B | Guest/federated partner collaboration |
| Customers must sign in with Google/Facebook accounts | Entra ID B2C / Entra External ID | Consumer identity and social sign-in |
| Admin sign-in must be phishing-resistant | FIDO2 or Windows Hello for Business | Strongest exam-favorite answer |
| Require stronger access controls based on user risk or sign-in risk | Identity Protection + Conditional Access | Risk-based access needs P2 features |
| Admins should only elevate when needed | PIM Eligible assignment | Just-in-time privilege |
| Quarterly review of privileged access | Access Reviews | Governance/attestation pattern |
| Azure VM needs Key Vault access without secrets | System-assigned Managed Identity | Best for one Azure resource |
| Multiple apps need the same identity | User-assigned Managed Identity | Reusable identity across resources |
| GitHub Actions pipeline needs Azure auth | Service Principal with federated credential | Better than long-lived client secret |
| Need to control who can manage only one resource group | Azure RBAC at resource group scope | Least privilege at smallest useful scope |
| Built-in role is too broad | Custom RBAC role | Fine-grained permissions |
| Enforce tags on all new resources | Azure Policy (Deny) | Blocks noncompliant deployments |
| Automatically add missing tags from resource group | Azure Policy (Modify) | Remediates/inherits tags |
| Automatically deploy diagnostics to resources | Azure Policy (DeployIfNotExists) | Auto-remediation pattern |
| Need one governance model across many subscriptions | Management Groups | Scope and inheritance at scale |
| Need audit before enforcement | Azure Policy (Audit) | Visibility-first rollout |
| Need packaged legacy governance artifact with policies and assignments | Azure Blueprints | Legacy service exam awareness |
| Need modern replacement for Blueprints | Landing zones + Policy + Template Specs/Deployment Stacks | Current recommended direction |
| Need fast alert on CPU or memory spike | Metric alert | Near-real-time alerting |
| Need to investigate failures over time with rich context | Log Analytics | Query logs with KQL |
| Need application dependency tracing | Application Insights | APM and distributed tracing |
| Need reusable notification targets for alerts | Action Groups | Common alert routing pattern |
| Need dashboards for operations or SOC | Workbooks | Interactive visual reports |
| Need recommendations to reduce cost or improve reliability/security | Azure Advisor | Built-in recommendation engine |
| Need regulatory compliance dashboard and security posture score | Defender for Cloud | Secure Score + compliance mapping |
| Need SIEM/SOAR across cloud and on-prem sources | Microsoft Sentinel | Security analytics and automation |
| Need Azure governance for on-prem servers | Azure Arc | Extend Azure control plane to hybrid |
| Need centralized SOC visibility across subscriptions | Centralized Log Analytics workspace | Easier cross-resource correlation |
| Need logs to stay in-country/region | Distributed/regional workspaces | Meets sovereignty constraints |
| Need both regional ownership and central SOC view | Hybrid workspace design | Balances autonomy and central operations |
| Need long-term cheap log retention | Storage/archive strategy | Log Analytics alone is costly for long retention |
| Need to block legacy authentication | Conditional Access | Common Zero Trust control |
| Need policy exceptions for approved edge cases | Azure Policy exemptions | Governed compliance exceptions |
| Need to prevent accidental deletion, even by authorized users | Resource locks / deny pattern | RBAC alone does not prevent accidental delete |
| Need cost accountability by team | Tags + subscription strategy | Chargeback/showback design |
| Need security alerts to trigger automation | Sentinel playbooks or alert action groups | Automated response pattern |

---

## Final Cram List

- **RBAC = who. Policy = what. Conditional Access = sign-in conditions. PIM = just-in-time privilege.**
- **B2B = partners. B2C/External ID = customers. Managed Identity = Azure workloads with no secrets.**
- **Management Groups are the scale unit for governance inheritance across subscriptions.**
- **Metrics are for fast alerts; logs are for investigation, retention, correlation, and security analytics.**
- **Defender for Cloud improves posture; Sentinel correlates threats and drives SOC response.**
