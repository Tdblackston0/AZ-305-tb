> 📝 **Hands-On Labs:** [Entra ID Labs](./Labs/Azure-EntraID-Labs.md)

# Microsoft Entra ID Cheat Sheet + AZ-305 Exam Prep

**Audience:** Senior Cloud Solution Architect  
**Exam focus:** AZ-305 - Design identity, governance, and monitoring solutions  
**Use this for:** fast revision, architecture decisions, and scenario-based exam thinking

---

## 1. Microsoft Entra ID Overview

Microsoft Entra ID is **Microsoft's cloud identity and access management platform** for workforce, application, device, and external identities. It is **not only "Azure AD renamed"**; the rename reflects a broader identity portfolio under Microsoft Entra, with Entra ID remaining the core cloud directory and policy engine.

### What it is
- Cloud identity provider for **authentication**, **authorization**, **SSO**, **Conditional Access**, **identity governance**, and **application identity**.
- Provides identities for:
  - Users
  - Groups
  - Devices
  - Applications
  - Managed identities
  - External users
- Integrates with Microsoft 365, Azure, SaaS apps, custom apps, and hybrid environments.

### Entra ID vs on-premises AD DS

| Area | Microsoft Entra ID | Active Directory Domain Services (AD DS) |
|---|---|---|
| Core purpose | Cloud identity and access control | Traditional domain directory and Windows domain management |
| Protocols | OAuth 2.0, OpenID Connect, SAML, SCIM | Kerberos, NTLM, LDAP, Group Policy |
| Device management | Identity registration/join, integrates with Intune | Domain join and Group Policy |
| Authentication location | Internet-facing cloud service | Primarily internal/domain-connected infrastructure |
| App model | SaaS, cloud apps, APIs, modern auth | Windows/server-centric, legacy enterprise apps |
| Policy engine | Conditional Access, Identity Protection | Group Policy, AD FS, domain controls |
| Best use | Cloud-first/hybrid identity | Legacy line-of-business/domain workloads |

**Exam point:** Entra ID does **not** replace AD DS features like OU/GPO/LDAP/Kerberos for legacy apps. In hybrid designs, the two often coexist.

### Licensing tiers

| Feature | Free | P1 | P2 |
|---|---:|---:|---:|
| User/group management | Yes | Yes | Yes |
| SSO to Microsoft 365/Azure/SaaS | Yes | Yes | Yes |
| Security defaults | Yes | Yes | Yes |
| Basic MFA capability | Limited | Yes | Yes |
| Conditional Access | No | Yes | Yes |
| Dynamic groups | No | Yes | Yes |
| Self-service password reset (writeback advanced scenarios) | Limited | Yes | Yes |
| Hybrid identity advanced features | Limited | Yes | Yes |
| Identity Protection | No | No | Yes |
| Privileged Identity Management (PIM) | No | No | Yes |
| Access reviews | No | No | Yes |
| Entitlement management | No | No | Yes |

**Remember:**
- **P1** = Conditional Access and many enterprise identity controls.
- **P2** = identity governance + risk-based identity protection + PIM.

### Tenant concepts
- A **tenant** is the dedicated Entra ID instance for an organization.
- A tenant contains:
  - Users
  - Groups
  - App registrations
  - Enterprise applications
  - Policies
  - Devices
  - Administrative units
- Common tenant patterns:
  - **Single tenant**: one organization, one identity boundary.
  - **Multi-tenant app**: one application serving users from many tenants.
  - **B2B cross-tenant**: collaboration across separate workforce tenants.
- Azure subscriptions trust a tenant for identity.

### Quick commands

```powershell
Connect-MgGraph -Scopes "User.Read.All","Directory.Read.All","Policy.Read.All"
Get-MgOrganization
Get-MgUser -Top 5
```

```powershell
# Legacy module still seen in older docs/exams
Connect-AzureAD
Get-AzureADTenantDetail
Get-AzureADUser -Top 5
```

```bash
az login
az account show
az ad signed-in-user show
az ad user list --top 5
```

---

## 2. Authentication Methods

### Password-based authentication
**When to use:** legacy compatibility only; avoid as the long-term primary strategy.

- Most common baseline method.
- Weakest standalone option because passwords can be phished, sprayed, reused, or leaked.
- Still needed in many hybrid and transitional environments.

**Architect guidance:** Pair password auth with **MFA**, **Conditional Access**, password protection, and eventually **passwordless**.

### Multi-factor authentication (MFA)
**When to use:** almost everywhere for workforce access, privileged operations, external admin access, and risky sign-ins.

Common MFA methods:
- Microsoft Authenticator push
- Authenticator number matching
- OATH hardware/software tokens
- SMS/voice (weaker; avoid if stronger methods available)
- FIDO2 security keys
- Certificate-based auth (for certain high-assurance scenarios)

#### Per-user MFA vs Conditional Access MFA

| Option | Use case | Recommendation |
|---|---|---|
| Per-user MFA | Old/simple environments | Legacy approach; avoid for modern design |
| Conditional Access MFA | Risk-based, app-based, role-based, device-based control | Preferred |

**Exam rule:** if asked for **flexible**, **context-aware**, or **scalable** MFA, choose **Conditional Access**, not per-user MFA.

### Passwordless authentication
**When to use:** modern workforce, phishing-resistant strategies, privileged admins, device-based sign-in.

| Method | Best fit | Notes |
|---|---|---|
| Windows Hello for Business | Corporate Windows endpoints | Strong device-bound credentials using biometrics/PIN |
| FIDO2 security keys | Frontline workers, shared devices, admins, phishing-resistant auth | Excellent for strong assurance |
| Microsoft Authenticator passwordless | Mobile-centric workforce | Easy rollout, lower friction |

### Certificate-based authentication (CBA)
**When to use:** smart card/PKI-heavy enterprises, regulated industries, high-assurance device/user auth.

- Useful when org already has PKI investments.
- Often appears in hybrid enterprises requiring cert trust models.
- More operationally complex than app-based or passwordless options.

### Authentication strengths
**When to use:** when you must require a **specific class** of authentication, not just generic MFA.

Authentication strengths let you require methods such as:
- MFA
- Passwordless MFA
- Phishing-resistant MFA

**Example:** Require **phishing-resistant MFA** for Azure admin portals.

### Decision matrix: scenario -> authentication method

| Scenario | Best choice | Why |
|---|---|---|
| General workforce cloud access | Password + Conditional Access MFA | Broad compatibility |
| Highly privileged admins | FIDO2 or Windows Hello + auth strength | Strong phishing resistance |
| Shared kiosk/frontline | FIDO2 keys | Portable and passwordless |
| Mobile-first users | Microsoft Authenticator passwordless | Easy user adoption |
| PKI-heavy regulated org | Certificate-based auth | Aligns with existing controls |
| Legacy app transition | Password + MFA | Transitional state |
| Remote/hybrid workforce | Passwordless + Conditional Access | Better security and UX |

### Useful commands

```powershell
Connect-MgGraph -Scopes "UserAuthenticationMethod.Read.All","Policy.Read.All"
Get-MgUserAuthenticationMethod -UserId user@contoso.com
```

```bash
# Review authentication methods policy via Microsoft Graph using az rest
az rest --method GET \
  --url "https://graph.microsoft.com/beta/policies/authenticationMethodsPolicy"
```

**Exam tip:** "phishing-resistant" usually points to **FIDO2**, **Windows Hello for Business**, or **certificate-based auth**, not SMS.

---

## 3. Conditional Access

Conditional Access is the **policy engine** that evaluates signals and applies access controls.

### Policy components

| Component | What it does |
|---|---|
| Assignments | Who/what is included or excluded: users, groups, roles, apps |
| Conditions | Signals such as location, device platform, client app, sign-in risk |
| Access controls | Grant or block access; require MFA, compliant device, hybrid join, terms of use |
| Session controls | Restrict session behavior with app enforced restrictions, sign-in frequency, CAE-related controls |

### Common policy patterns
- Require MFA for admins.
- Require MFA for all users outside trusted locations.
- Block legacy authentication.
- Require compliant device for Microsoft 365.
- Require hybrid joined or compliant devices for sensitive apps.
- Require phishing-resistant MFA for privileged actions.

### Named locations
**When to use:** trusted corporate egress IPs, countries/regions, branch offices.

- Use for policy exceptions and location-aware controls.
- Do **not** assume trusted location = no security forever.

### Risk-based policies
Requires **P2**.

Examples:
- If sign-in risk = medium or high -> require MFA.
- If user risk = high -> require password reset.

### Report-only mode
**Best practice for testing.**
- Deploy policy in **report-only** first.
- Review impact before enforcement.
- Essential when rolling out broad policies.

### Emergency access accounts (break-glass)
- Create at least two cloud-only emergency accounts.
- Exclude them from Conditional Access policies.
- Protect with strong passwords and monitoring.
- Use only for tenant lockout scenarios.

### Policy evaluation order
- All applicable policies are evaluated.
- **Block** wins over grant.
- Multiple grant controls may be cumulative if configured as "require all selected controls".
- Exclusions matter greatly.

**Exam trap:** Conditional Access is not processed like old firewall rule order. Think **applicable policies + combined effect**, not simple top-down sequence.

### Example commands

```powershell
Connect-MgGraph -Scopes "Policy.Read.All","Policy.ReadWrite.ConditionalAccess"
Get-MgIdentityConditionalAccessPolicy
```

```bash
az rest --method GET \
  --url "https://graph.microsoft.com/v1.0/identity/conditionalAccess/policies"
```

```powershell
# Legacy awareness only
Connect-AzureAD
Get-AzureADMSConditionalAccessPolicy
```

### Architect checklist
- Start with report-only.
- Exclude break-glass accounts.
- Block legacy auth early.
- Separate admin policies from workforce policies.
- Use authentication strengths for high-value apps.

---

## 4. Identity Protection

Identity Protection analyzes signals to detect risky users and risky sign-ins.

### Risk detection types

| Type | Meaning | Example |
|---|---|---|
| Sign-in risk | Probability that a sign-in is not legitimate | Anonymous IP, unfamiliar sign-in, malware-linked IP |
| User risk | Probability that the user account is compromised | Leaked credentials, suspicious activity, impossible travel patterns contributing to risk |

### Risk levels
- Low
- Medium
- High

### Automated remediation policies
Requires **P2**.

Typical policies:
- **Sign-in risk policy** -> require MFA.
- **User risk policy** -> require secure password reset.

### Investigation and remediation workflow
1. Review risky users/sign-ins.
2. Correlate with logs, device state, and location.
3. Confirm compromise likelihood.
4. Trigger remediation:
   - Force password reset
   - Require MFA
   - Revoke sessions/tokens
   - Block sign-in temporarily
5. Close or dismiss risk event when validated.

### Example commands

```powershell
Connect-MgGraph -Scopes "IdentityRiskyUser.Read.All","IdentityRiskEvent.Read.All"
Get-MgRiskyUser
Get-MgIdentityRiskyServicePrincipal  # if available in environment/module
```

```bash
az rest --method GET \
  --url "https://graph.microsoft.com/v1.0/identityProtection/riskyUsers"
```

**Exam point:** If the scenario says **detect compromised identities automatically**, think **Identity Protection (P2)**, not just Conditional Access.

---

## 5. Privileged Identity Management (PIM)

PIM provides **just-in-time (JIT)** and governed privileged access.

### Just-in-time access concept
- Users are **eligible** for privileged roles.
- They activate roles only when needed.
- Activation can require justification, MFA, approval, or ticket info.
- Reduces standing privilege.

### Eligible vs active assignments

| Assignment type | Meaning | Use |
|---|---|---|
| Eligible | User can activate role when needed | Preferred for admins |
| Active | Role is always assigned | Use sparingly |

### Activation workflow and approvals
Typical flow:
1. User requests activation.
2. MFA and/or justification required.
3. Optional approval required.
4. Time-bound activation granted.
5. Access expires automatically.

### Access reviews
**When to use:** recurring validation of privileged or guest access.
- Helps certify whether access is still needed.
- Useful for compliance and governance.

### PIM for Groups
- Apply eligibility and governance to **groups**, then assign group to roles/apps/resources.
- Simplifies large-scale privileged administration.

### When to use PIM vs permanent assignments

| Situation | Use PIM? |
|---|---|
| Global admin, privileged role admin, subscription owner | Yes |
| Temporary project admin | Yes |
| Break-glass account | No; keep separate, tightly protected |
| Standard user access | Usually no |
| Service principal runtime access | No; use app identity model |

### Example commands

```powershell
Connect-MgGraph -Scopes "RoleManagement.Read.Directory","RoleEligibilitySchedule.Read.Directory"
Get-MgRoleManagementDirectoryRoleEligibilitySchedule
Get-MgRoleManagementDirectoryRoleAssignmentSchedule
```

```bash
az role assignment list --assignee user@contoso.com
az role definition list --name Owner
```

**Exam point:** If asked to reduce standing admin rights and require approval/time-bound elevation, choose **PIM**.

---

## 6. Managed Identities

Managed identities are Entra ID identities automatically managed by Azure for workloads.

### System-assigned vs user-assigned

| Area | System-assigned managed identity | User-assigned managed identity |
|---|---|---|
| Lifecycle | Tied to one Azure resource | Independent Azure resource |
| Deletion behavior | Deleted when parent resource is deleted | Persists independently |
| Reuse across resources | No | Yes |
| Best for | One app/one resource identity | Shared identity across multiple resources |
| Operational overhead | Lower | Slightly higher |
| Rotation of credentials | Managed by Azure | Managed by Azure |

### When to use each
- **System-assigned**:
  - App Service needing Key Vault access
  - Function app accessing Storage/SQL
  - VM with unique identity requirements
- **User-assigned**:
  - Shared identity used by many apps
  - Stable identity across resource recreation
  - Multiple resources needing the same RBAC grants

### Supported services
Common services supporting managed identity include:
- Virtual Machines
- App Service
- Azure Functions
- Logic Apps
- Container Apps
- AKS (via workload identity patterns)
- Azure Automation
- Azure Data Factory
- Azure Synapse and others

### Authenticate to Azure resources
Use managed identity with:
- Key Vault
- Storage
- SQL Database
- Service Bus
- Event Hubs
- Azure Resource Manager APIs

#### Example: enable and assign

```bash
az webapp identity assign -g rg-identity -n app-contoso
az functionapp identity assign -g rg-identity -n func-contoso
az identity create -g rg-identity -n uaid-shared
```

```bash
# Grant Key Vault access via RBAC example
PRINCIPAL_ID=$(az webapp identity show -g rg-identity -n app-contoso --query principalId -o tsv)
az role assignment create \
  --assignee-object-id $PRINCIPAL_ID \
  --assignee-principal-type ServicePrincipal \
  --role "Key Vault Secrets User" \
  --scope /subscriptions/<subId>/resourceGroups/rg-sec/providers/Microsoft.KeyVault/vaults/kv-contoso
```

```powershell
Connect-AzAccount
Set-AzWebApp -AssignIdentity $true -ResourceGroupName rg-identity -Name app-contoso
```

### Managed Identity vs Service Principal decision tree

| Question | Choose |
|---|---|
| Is the workload running in Azure on a supported service? | Managed identity |
| Does the identity need to exist independently of compute? | User-assigned MI or service principal |
| Is the workload outside Azure/GitHub/on-prem/external? | Service principal or workload identity federation |
| Do you want to avoid secret/certificate management? | Managed identity |

### Federation with external identity providers
Modern pattern: **workload identity federation**.
- Useful for GitHub Actions, Kubernetes, or external workloads.
- Avoids storing long-lived secrets.
- Often preferred over traditional client secrets.

### Architect guidance
- Prefer **managed identity** for Azure-hosted workloads.
- Prefer **user-assigned** when identity reuse or stable lifecycle matters.
- Use RBAC least privilege on target resources.

---

## 7. Service Principals & App Registrations

### App registration vs enterprise application

| Object | Meaning |
|---|---|
| App registration | Global definition/application blueprint in home tenant |
| Enterprise application | Service principal instance of the app in a tenant |

**Shortcut:** app registration = template; enterprise app = tenant-local instantiated identity.

### Client credentials flow
**When to use:** daemon/service/API-to-API calls with no signed-in user.

- App authenticates with:
  - Client secret
  - Certificate
  - Federated credential
- Requests application permissions to call APIs.

### Certificates vs secrets

| Credential type | Recommendation | Why |
|---|---|---|
| Client secret | Avoid if possible | Easier to leak/expire poorly |
| Certificate | Preferred | Stronger and more manageable |
| Federated credential | Excellent where supported | Secretless trust model |

### API permissions

| Permission type | Use case |
|---|---|
| Delegated | App acts on behalf of signed-in user |
| Application | App acts as itself/background process |

### When to use Service Principal vs Managed Identity

| Scenario | Best choice |
|---|---|
| Azure VM/App Service/Function accessing Azure service | Managed identity |
| Third-party CI/CD pipeline calling Azure APIs | Service principal or federated workload identity |
| Multi-cloud/on-prem automation | Service principal |
| Azure-hosted app with no reason to manage credentials | Managed identity |

### Example commands

```bash
az ad app create --display-name "contoso-api"
az ad sp create-for-rbac --name "sp-contoso-automation" --role Reader \
  --scopes /subscriptions/<subId>/resourceGroups/rg-app
```

```powershell
Connect-MgGraph -Scopes "Application.ReadWrite.All","AppRoleAssignment.ReadWrite.All"
New-MgApplication -DisplayName "contoso-api"
Get-MgServicePrincipal -Filter "displayName eq 'contoso-api'"
```

```powershell
# Legacy awareness
Connect-AzureAD
New-AzureADApplication -DisplayName "contoso-api"
New-AzureADServicePrincipal -AppId <appId>
```

**Exam point:** if the app runs **inside Azure**, think **managed identity first**. If it runs **outside Azure**, think **service principal/federation**.

---

## 8. Hybrid Identity

### Azure AD Connect vs Azure AD Connect Cloud Sync

| Area | Azure AD Connect | Cloud Sync |
|---|---|---|
| Engine location | Full sync engine on server | Lightweight agents |
| Complexity | Higher | Lower |
| Feature breadth | Broadest feature set | Simpler targeted sync |
| Best for | Complex hybrid environments | Simpler/more modern sync deployments |
| Device/object scenarios | More mature for complex needs | Good for many common user/group sync needs |
| Operational model | Heavier server dependency | Lighter agent-based |

### Sync options

| Method | What it does | Best fit |
|---|---|---|
| Password Hash Sync (PHS) | Syncs password hash representation to cloud | Default/recommended for many orgs |
| Pass-through Authentication (PTA) | Validates password against on-prem AD in real time | Need on-prem auth validation |
| Federation (AD FS) | Redirects auth to federation service | Specialized/sign-in policy/legacy/regulatory cases |

### Decision tree for hybrid identity method

| Requirement | Best choice |
|---|---|
| Simplest, resilient, cloud-first hybrid auth | PHS |
| Password validation must stay on-prem | PTA |
| Need advanced custom sign-in/federation behavior | Federation |
| Want leaked credential detection with simplest model | PHS |

### Seamless SSO
- Reduces sign-in prompts for domain-joined users.
- Commonly paired with **PHS** or **PTA**.
- Good user experience enhancer, not a replacement for strong auth.

### Device registration states

| State | Best fit |
|---|---|
| Entra joined (Azure AD joined) | Cloud-native corporate devices |
| Hybrid Entra joined | Traditional domain-joined devices needing cloud registration |
| Entra registered | BYOD/personal devices |

### Example commands

```powershell
# Graph device review
Connect-MgGraph -Scopes "Device.Read.All"
Get-MgDevice -Top 10
```

```bash
az ad device list --top 10
```

**Exam heuristics:**
- Default to **PHS** unless a requirement explicitly pushes you to PTA/federation.
- Choose **Hybrid Entra joined** for existing domain-joined enterprise Windows fleets.
- Choose **Entra joined** for cloud-first endpoint strategy.

---

## 9. B2B Collaboration

### What it is
B2B collaboration lets external users access your workforce tenant resources as **guest users**.

### Guest user lifecycle
1. Invite guest.
2. Guest redeems invitation.
3. Guest becomes object in resource tenant.
4. Guest gets assigned apps/groups/resources.
5. Access reviewed and eventually removed.

### Invitation and redemption

```powershell
Connect-MgGraph -Scopes "User.Invite.All"
New-MgInvitation -InvitedUserEmailAddress partner@fabrikam.com \
  -InviteRedirectUrl "https://myapps.microsoft.com" \
  -SendInvitationMessage
```

```powershell
# Legacy awareness
Connect-AzureAD
New-AzureADMSInvitation -InvitedUserEmailAddress partner@fabrikam.com -InviteRedirectUrl "https://myapps.microsoft.com"
```

### Cross-tenant access settings
- Control inbound/outbound trust between tenants.
- Used for collaboration with specific partner organizations.
- Helps govern MFA/device/compliance trust across tenants.

### External collaboration settings
- Restrict who can invite guests.
- Limit allowed/blocked domains.
- Control guest permissions.

### B2B direct connect
- Designed for deeper cross-tenant collaboration scenarios, including Microsoft Teams shared channels.
- Different from standard guest invitation model.

**Exam point:** partners accessing **your apps/resources** = usually **B2B**, not B2C.

---

## 10. B2C (External Identities)

### When to use B2C vs B2B

| Need | Choose |
|---|---|
| Partner/vendor collaboration in workforce apps | B2B |
| Customer-facing sign-up/sign-in at scale | B2C |

### User flows vs custom policies

| Option | Use when |
|---|---|
| User flows | Standard sign-up, sign-in, profile edit, password reset |
| Custom policies | Complex journeys, advanced claims/rules/federation logic |

### Identity providers
- Local accounts
- Social identities
- OpenID Connect providers
- SAML providers

### Token customization
- Add claims for app behavior.
- Useful when apps need profile, role, or business-specific attributes.

### B2C pricing model
- Generally consumption/MAU-oriented rather than traditional workforce licensing.
- Exam focus is usually **fit-for-purpose**, not memorizing every price detail.

### Example design guidance
- Use B2C for customer portals, retail apps, citizen services, or public apps.
- Avoid B2C for workforce guest collaboration scenarios.

---

## 11. Entra External ID

### What it is
Microsoft Entra External ID is the broader external identity platform for managing external users and customer/workforce external access scenarios.

### Workforce vs external scenarios

| Scenario | Typical choice |
|---|---|
| Partners/vendors using your workforce apps | B2B / External ID workforce collaboration |
| Consumer/customer application | External ID customer scenario / B2C-style requirement |

### Comparison with B2B and B2C

| Capability | B2B | B2C | Entra External ID |
|---|---|---|---|
| Purpose | Partner collaboration | Customer identity | Broader external identity platform |
| User type | Guests/partners | Consumers/customers | Both, depending on scenario |
| Typical app | Workforce apps | Public-facing apps | Modern external identity scenarios |

**Exam tip:** If question wording is older, it may still explicitly say **Azure AD B2C**. If wording is newer, think **Entra External ID** for external customer/partner identity scenarios.

---

## 12. Security Best Practices

### Zero Trust principles applied to identity
- Verify explicitly.
- Use least privilege.
- Assume breach.

### Least privilege access
- Grant only required roles.
- Prefer group-based assignments.
- Use PIM for elevated roles.
- Review guest access regularly.

### Security defaults vs Conditional Access

| Option | Best for | Notes |
|---|---|---|
| Security defaults | Small/simple orgs | Quick baseline, limited customization |
| Conditional Access | Enterprise environments | Flexible, granular, preferred for AZ-305 scenarios |

### Legacy authentication blocking
- Block legacy protocols that bypass MFA.
- Common exam answer for reducing attack surface against password spray/basic auth.

### Monitoring and alerting on risky sign-ins
- Monitor sign-in logs.
- Review risky users/sign-ins.
- Alert on impossible travel, unfamiliar properties, and admin role activations.
- Stream logs to Log Analytics / Microsoft Sentinel when required.

### Example commands

```powershell
Connect-MgGraph -Scopes "AuditLog.Read.All","Directory.Read.All"
Get-MgAuditLogSignIn -Top 20
```

```bash
az rest --method GET \
  --url "https://graph.microsoft.com/v1.0/auditLogs/signIns?$top=10"
```

**Architect pattern:** Security defaults are fine for basic protection, but **Conditional Access + PIM + Identity Protection** is the enterprise design answer.

---

## 13. AZ-305 Decision Scenarios

### 1. Multi-tenant SaaS identity design
**Scenario:** You are designing a SaaS application used by customers from many organizations. Each organization wants its own tenant to authenticate its users.

**Best answer:** Multi-tenant app registration in Entra ID using OpenID Connect/OAuth 2.0.

**Why:** Customer identities remain in their home tenants; your app trusts tokens from multiple tenants.

### 2. Hybrid identity for enterprise
**Scenario:** A global enterprise wants the simplest hybrid sign-in model, cloud resiliency, and leaked credential detection.

**Best answer:** Azure AD Connect with **Password Hash Sync** plus Seamless SSO.

**Why:** Simplest, resilient, and enables cloud-based detections.

### 3. B2B partner collaboration
**Scenario:** External legal partners need access to SharePoint and Teams in your tenant using their own credentials.

**Best answer:** Entra B2B collaboration with cross-tenant access settings as needed.

**Why:** Partners are guest users in your tenant; do not build a consumer identity platform for this.

### 4. Customer-facing app
**Scenario:** A public retail app needs millions of customer identities, social sign-in, and custom sign-up journeys.

**Best answer:** Entra External ID / Azure AD B2C pattern.

**Why:** This is customer identity and access management, not workforce collaboration.

### 5. Privileged access management
**Scenario:** Security team wants subscription Owners to have admin access only when approved, for four hours max.

**Best answer:** PIM eligible assignments with approval, MFA, and time-bound activation.

**Why:** Removes standing privilege and enforces JIT elevation.

### 6. Passwordless rollout
**Scenario:** Executives and administrators need phishing-resistant sign-in with minimal user friction.

**Best answer:** Windows Hello for Business or FIDO2, enforced with authentication strengths.

**Why:** Stronger than generic MFA and aligned with zero trust.

### 7. Conditional Access design
**Scenario:** Org wants to test a new policy requiring MFA for finance users accessing ERP from outside trusted locations, without disrupting users.

**Best answer:** Conditional Access in **report-only** mode using named locations.

**Why:** Safest rollout path.

### 8. Managed Identity implementation
**Scenario:** An Azure Function must read secrets from Key Vault without storing credentials.

**Best answer:** System-assigned managed identity plus RBAC/Key Vault permissions.

**Why:** Native secretless Azure-hosted workload identity.

### 9. Service principal vs managed identity
**Scenario:** GitHub Actions pipeline needs to deploy to Azure without storing long-lived secrets.

**Best answer:** Workload identity federation for app/service principal.

**Why:** Managed identity is not available directly to GitHub-hosted runners; federation avoids secrets.

### 10. Conditional Access vs security defaults
**Scenario:** Small company asks for immediate baseline identity protection with minimal administration.

**Best answer:** Security defaults.

**Why:** Quick baseline when full CA design is unnecessary.

### 11. Device strategy
**Scenario:** New cloud-native laptop fleet with Intune and no dependency on on-prem GPO.

**Best answer:** Entra joined devices.

**Why:** Cloud-first device identity model.

### 12. Risk-based remediation
**Scenario:** Company needs automatic response when a user is flagged high risk due to leaked credentials.

**Best answer:** Identity Protection user risk policy requiring password reset.

**Why:** This directly addresses compromised identity remediation.

---

## 14. Quick Reference Trigger Table

| If the scenario says... | Think... |
|---|---|
| Partners need access to apps | B2B Collaboration |
| Customers/consumers sign up | B2C or External ID |
| Just-in-time privileged access | PIM |
| Password never validated in cloud | PTA |
| Simplest hybrid sign-in | PHS |
| Detect leaked credentials | PHS + Identity Protection |
| Risky sign-ins | Identity Protection |
| Require MFA only for certain apps/users/locations | Conditional Access |
| Need to test policy before enforcing | Report-only mode |
| Exclude emergency admin from policy | Break-glass accounts |
| Block old mail/basic auth protocols | Block legacy authentication |
| Azure app needs Key Vault secret access | Managed identity |
| Shared identity across multiple Azure resources | User-assigned managed identity |
| Identity tied to one Azure resource lifecycle | System-assigned managed identity |
| App runs outside Azure | Service principal or federation |
| API daemon with no user | Client credentials flow |
| App acting for signed-in user | Delegated permissions |
| App acting as itself | Application permissions |
| Need phishing-resistant MFA | FIDO2 / WHfB / CBA / auth strengths |
| Corporate Windows device passwordless | Windows Hello for Business |
| Frontline/shared device strong auth | FIDO2 |
| Small org, quick protection | Security defaults |
| Enterprise granular identity controls | Conditional Access |
| Partner uses own tenant credentials | B2B |
| Public customer identity platform | External ID / B2C |
| Cloud-native device join | Entra joined |
| Existing domain-joined fleet + cloud identity | Hybrid Entra joined |
| BYOD personal device | Entra registered |
| Need approval before admin elevation | PIM activation approval |
| Periodic review of guest/admin access | Access reviews |
| Manage app identity without secrets in Azure | Managed identity |
| CI/CD without stored secrets | Workload identity federation |
| Flexible MFA targeting | Conditional Access, not per-user MFA |
| Complex custom customer sign-up journey | B2C custom policies |
| Teams shared channels with external orgs | B2B direct connect |
| Require specific auth method class | Authentication strengths |

---

## 15. Common Exam Traps

### 1. B2B vs B2C confusion
- **B2B** = partners, vendors, guest users accessing your workforce resources.
- **B2C/External ID** = customers using your public-facing apps.

### 2. PHS vs PTA vs Federation
- Default to **PHS** unless a requirement says password validation must remain on-prem or you need special federation behavior.
- **PTA** is not automatically more secure; it is a requirements fit decision.
- **Federation** is usually for specialized cases, not the default recommendation.

### 3. Managed Identity limitations
- Managed identities are for **Azure-hosted supported services**.
- For external apps/pipelines, use **service principals** or **federated credentials**.
- System-assigned identities cannot be shared across resources.

### 4. Conditional Access policy conflicts
- Multiple policies can apply at once.
- Block overrides grant.
- Exclusions are critical.
- Forgetting break-glass exclusions is a classic design mistake.

### 5. PIM licensing requirements
- PIM requires **Entra ID P2**.
- If a question asks for JIT privileged access and governance, verify licensing assumptions.

### 6. Per-user MFA vs Conditional Access
- Per-user MFA is old/limited.
- Modern enterprise answer is usually **Conditional Access**.

### 7. Security defaults vs Conditional Access
- Security defaults are good for basic tenants.
- If the question needs exceptions, targeting, app scoping, named locations, device state, or report-only mode -> choose **Conditional Access**.

### 8. App registration vs enterprise app
- App registration is the definition.
- Enterprise application is the service principal in a tenant.

---

## Senior Architect Exam Strategy

### Fast elimination rules
- If it says **partner** -> start with **B2B**.
- If it says **customer portal** -> start with **B2C/External ID**.
- If it says **Azure-hosted workload needs Azure resource access** -> start with **managed identity**.
- If it says **privileged admin with approval/JIT** -> start with **PIM**.
- If it says **risk-based access** -> start with **Identity Protection + Conditional Access**.
- If it says **granular sign-in control** -> start with **Conditional Access**.
- If it says **simple hybrid** -> start with **PHS**.

### What AZ-305 usually rewards
- Least privilege
- Zero Trust design
- Secretless identity where possible
- Cloud-native controls over legacy complexity
- Governance and operational simplicity
- Phased rollout using report-only/testing

---

## Command Snippets to Remember

```bash
# Users, groups, apps
az ad user list --top 5
az ad group list --display-name "Admins"
az ad app list --display-name "contoso-api"
az ad sp list --display-name "contoso-api"
```

```powershell
# Microsoft Graph PowerShell
Connect-MgGraph -Scopes "User.Read.All","Group.Read.All","Application.Read.All","Policy.Read.All"
Get-MgUser -Top 10
Get-MgGroup -Top 10
Get-MgApplication -Top 10
Get-MgServicePrincipal -Top 10
```

```powershell
# AzureAD legacy examples still useful for exam context
Connect-AzureAD
Get-AzureADUser -Top 10
Get-AzureADGroup -Top 10
Get-AzureADApplication -Top 10
Get-AzureADServicePrincipal -Top 10
```

```bash
# Conditional Access / sign-in logs via Graph
az rest --method GET --url "https://graph.microsoft.com/v1.0/identity/conditionalAccess/policies"
az rest --method GET --url "https://graph.microsoft.com/v1.0/auditLogs/signIns?$top=10"
```

---

## Final Review Questions

1. A company wants external suppliers to use their own corporate credentials to access a procurement app in your tenant. Which identity model should you recommend, and why is B2C the wrong answer?
2. Your security team wants all privileged admins to have no standing access, require approval, and activate roles only when needed. Which Entra feature and license tier are required?
3. A cloud-hosted web app in Azure App Service must access Key Vault secrets without storing passwords or certificates. Which identity option is best, and when would user-assigned be better than system-assigned?
4. A hybrid enterprise wants the least complex sign-in model while still enabling cloud-based leaked credential detection. Should you choose PHS, PTA, or federation, and why?
5. An exam scenario asks for phishing-resistant authentication for administrators. Which methods should you consider first, and why is SMS MFA a weak answer?

---

**Bottom line:** For AZ-305, the best Entra ID answer is usually the one that is **least privileged, cloud-native, risk-aware, and easiest to govern at scale**.