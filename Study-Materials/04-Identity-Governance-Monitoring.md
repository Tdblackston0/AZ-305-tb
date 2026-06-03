# Identity, Governance & Monitoring Solutions - SECONDARY PRIORITY ⭐⭐

**Exam Weight:** 25-30% of exam  
**Your Performance:** ⚠️ WEAK SECTION (medium bar)  
**Potential Points:** +5-8

---

## Overview

This domain combines three critical areas:
1. **Identity** - Authentication, authorization, access management
2. **Governance** - Policy, compliance, management at scale (see 01-Governance.md for deep dive)
3. **Monitoring** - Observability, alerts, diagnostics

You did better here than governance alone, but still need improvement.

---

## Identity Solutions

### Azure Entra ID (formerly Azure AD)

**Core Concept:** Identity provider for Azure and cloud applications

**Key Components:**

#### 1. Authentication Methods

| Method | Use Case | Security |
|--------|----------|----------|
| **Passwords** | Basic | Low |
| **Multi-Factor Auth (MFA)** | All users | High |
| **Passwordless** | Modern apps | Very High |
| **Windows Hello** | Local sign-in | Very High |
| **FIDO2 | Enterprise | Very High |

**Exam Scenario:** "Your company must comply with NIST guidelines requiring MFA for all cloud access."
- **Answer:** Implement conditional access policy requiring MFA for all cloud app access

#### 2. Conditional Access

**What it is:** Fine-grained access control based on conditions

**Conditions Include:**
- User risk (impossible travel, suspicious activity)
- Device compliance (patched, encrypted)
- Location (only allow from corporate network)
- Client app (allow only managed apps)
- Application sensitivity

**Example Policy:**
```
IF:
  - User signs in from outside US
  - Device is not domain-joined
  - Application is "Production"

THEN:
  - REQUIRE MFA
  - OR Block access
```

**Common Exam Scenario:** "How do you allow secure access to sensitive apps while preventing compromised accounts from accessing them?"
- **Answer:** Conditional Access policy with MFA + device compliance checks

#### 3. Managed Identity

**What it is:** Azure-managed credentials for services (no passwords to manage)

**Types:**
- **System-assigned:** One per resource, deleted with resource
- **User-assigned:** Standalone, can be assigned to multiple resources

**Use Case:** Azure Function needs to access SQL Database without storing connection string

```
Function
  ↓ (uses system-assigned managed identity)
Entra ID
  ↓ (requests token)
SQL Database
  ↓ (allows function)
Access granted
```

**Hands-On:**
```
1. Enable managed identity on App Service
2. Grant identity permissions to SQL DB
3. Connect from app using identity (no credentials)
```

### RBAC Deep Dive (See 01-Governance.md)

**Key Refresh:**
- Scope: Management Group → Subscription → RG → Resource
- Roles: Owner, Contributor, Reader, custom roles
- Principle of least privilege

---

## Governance Solutions

### See 01-Governance.md for Deep Dive

**Quick Review:**
- Management Groups for hierarchy
- Azure Policy for enforcement
- RBAC for access control
- Compliance Manager for tracking

---

## Monitoring & Observability

### 1. Azure Monitor

**What it is:** Central platform for monitoring Azure resources and applications

**Data Types:**
- **Metrics** - Numeric values (CPU %, disk space, request count)
- **Logs** - Events, errors, traces
- **Traces** - Request tracking across services

**Core Features:**

#### Metrics & Alerts
```
Resource Metric
  ↓
Time-series data (1-minute intervals)
  ↓
Alert Rule (if CPU > 80%)
  ↓
Action Group
  ├─ Email admin
  ├─ SMS
  ├─ Create incident in ITSM
  └─ Trigger runbook (auto-remediate)
```

**Exam Scenario:** "Set up monitoring for production VMs. Alert if CPU > 85% for 5 minutes. Automatically scale up if this occurs."
- **Answer:** 
  - Metric alert in Azure Monitor
  - Action group with runbook to trigger scaling
  - VMSS auto-scale rule as backup

#### Log Analytics & Kusto Query Language (KQL)

**What it is:** Centralized logging with powerful query capabilities

**Workflow:**
```
Resource Logs
  ↓
Log Analytics Workspace
  ↓
Store in tables (Event, Perf, Syslog, etc.)
  ↓
Query with KQL
  ↓
Create alerts, dashboards
```

**KQL Example:**
```kusto
Perf
| where ObjectName == "Processor" 
  and InstanceName == "Total"
| where CounterName == "% Processor Time"
| where TimeGenerated > ago(1h)
| summarize AvgCPU = avg(CounterValue) by Computer
| where AvgCPU > 80
| sort by AvgCPU desc
```

**Use Cases:**
- Investigate security incidents
- Performance troubleshooting
- Custom reporting
- Anomaly detection

### 2. Application Insights

**What it is:** Application-level monitoring (requests, dependencies, exceptions)

**Monitors:**
- Request rates and latencies
- Exception rates
- Dependency calls (databases, external APIs)
- Custom events (business metrics)

**Use Case:** Monitor e-commerce checkout flow

```
Application Insights tracks:
├─ Request: /checkout
├─ Dependency: SQL query (inventory)
├─ Dependency: Payment API call
├─ Exception: If any step fails
└─ Duration: End-to-end latency
```

**Exam Scenario:** "Your web app has high error rates. How do you identify the root cause?"
- **Answer:** 
  1. Check Application Insights for failed requests
  2. Analyze dependency calls (DB, API failures)
  3. Look at exceptions and stack traces
  4. Correlate with server logs

### 3. Service Health & Resource Health

**Service Health:** Platform-wide issues (Azure service outages)  
**Resource Health:** Individual resource issues (specific VM down)

---

## Monitoring Best Practices

### 1. Define Metrics That Matter

Not all metrics are important. Focus on:
- **Application metrics:** Request latency, error rate, throughput
- **Infrastructure metrics:** CPU, memory, disk (only if monitoring manually)
- **Business metrics:** Transactions/sec, customer satisfaction

### 2. Alert Thresholds

**Poor:** Alert on every spike  
**Good:** Alert only on sustained problems
```
Alert if CPU > 85% for 5 minutes (not single spike)
Alert if error rate > 5% for 10 minutes
```

### 3. Alert Fatigue Prevention

❌ **Too many alerts** → Ignore important ones  
✅ **Smart alerts** → Only on real issues

**Techniques:**
- Set appropriate thresholds
- Use intelligent detection (baseline deviations)
- Group related alerts
- Route to right teams

---

## Practical Integration Example

### Complete Monitoring Architecture

```
Application
├─ Application Insights (app-level)
├─ Log Analytics (logs)
└─ Metrics (infrastructure)
    ↓
Azure Monitor
├─ Alert Rules
├─ Action Groups
└─ Dashboards
    ↓
Distribution
├─ Email/SMS
├─ ITSM integration (ServiceNow)
├─ Slack/Teams
└─ Auto-remediation (runbooks)
```

---

## Compliance & Monitoring

### Audit Logging

**What to monitor:**
- Who accessed what resources (RBAC changes)
- What policy changes were made
- Configuration changes to security settings
- Administrative actions

**Where it goes:**
```
Azure Activity Log
  ↓
Send to Log Analytics
  ↓
Query and alert on suspicious activities
```

**Exam Scenario:** "How do you ensure someone didn't grant themselves Admin rights?"
- **Answer:** Monitor Activity Log for role assignment changes, set up alerts, review audit logs monthly

---

## Identity + Governance + Monitoring Integration

### Scenario: Secure Access with Audit

```
User wants to access Production SQL Database
    ↓
Entra ID (identity) - Is this user legitimate?
    ↓
Conditional Access (governance) - Is device compliant? MFA passed?
    ↓
RBAC (governance) - Does user have permission?
    ↓
Activity Log (monitoring) - Record this access
    ↓
Log Analytics (monitoring) - Store for audit
    ↓
Azure Monitor (monitoring) - Alert if unusual pattern
    ↓
Access granted (or denied)
```

---

## Quick Reference: Common Scenarios

### Scenario: Cloud Security Posture

**Requirement:** Continuously monitor security compliance

**Solution Stack:**
- **Entra ID:** Conditional Access policies
- **Governance:** Azure Policy for resource compliance
- **Monitoring:** Azure Monitor + Defender for Cloud
- **Logging:** All actions to Log Analytics
- **Alerting:** Real-time alerts for policy violations

### Scenario: Cost Control + Compliance

**Requirement:** Ensure tags are applied and track spending

**Solution:**
- **Governance:** Azure Policy to enforce tags
- **Monitoring:** Cost Analysis grouped by tags
- **Alerts:** Alert if spending exceeds budget
- **Logging:** Track who created resources without tags

### Scenario: Disaster Response

**Requirement:** Know what happened during incident

**Investigation Flow:**
```
Incident occurs
    ↓
Check Service Health (platform issue?)
    ↓
Check Resource Health (specific resource issue?)
    ↓
Review Application Insights (errors?)
    ↓
Query Log Analytics (access patterns?)
    ↓
Check Activity Log (who changed what?)
    ↓
Root cause identified
```

---

## Practice Exam Questions

### Question 1
Your company requires that all users use MFA when accessing production applications. Non-production applications should allow single-factor auth. How do you implement this?

**Answer:** Conditional Access policy that requires MFA specifically for production app access based on application name/sensitivity

---

### Question 2
You need to prevent a compromised user account from accessing company resources until the threat is resolved. What should you use?

**Answer:** Conditional Access with user risk detection, automatically requiring verification or blocking

---

### Question 3
Your application needs to read secrets from Key Vault without storing credentials. What's the best approach?

**Answer:** Use managed identity with Key Vault access policy granting read permissions to that identity

---

## Study Plan for This Section

**Priority:** After finishing 01-Governance, 02-App-Architecture, 03-Data-Integration  
**Time Required:** 1-2 hours  
**Hands-On:** 1 hour

### What to Focus On:
1. ✅ Conditional Access (highest exam frequency)
2. ✅ Managed Identity (service authentication)
3. ✅ Azure Monitor fundamentals
4. ⚠️ Log Analytics/KQL (nice to know, less critical)
5. ✅ RBAC role assignment scenarios

---

## Key Microsoft Learn Resources

1. **[Training for Microsoft Entra ID](https://learn.microsoft.com/en-us/training/entra/)** - 45 min
2. **[Plan, implement, and administer Conditional Access](https://learn.microsoft.com/en-us/training/modules/plan-implement-administer-conditional-access/)** - 60 min
3. **[AZ-104: Monitor and back up Azure resources](https://learn.microsoft.com/en-us/training/paths/az-104-monitor-backup-resources/)** - 50 min
4. **[Application Insights for application performance monitoring](https://learn.microsoft.com/en-us/azure/azure-monitor/app/app-insights-overview)** - 45 min

---

## Next Steps After High-Priority Domains

1. Complete 01-Governance, 02-App-Architecture, 03-Data-Integration **first**
2. Then deep-dive this domain (Identity + Governance + Monitoring)
3. Focus on conditional access + managed identity hands-on labs
4. Practice identifying monitoring solutions for scenarios

**You're on track! Focus on the top 3 priority domains first.** 📊
