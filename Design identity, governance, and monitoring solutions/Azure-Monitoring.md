# Azure Monitoring - AZ-305 Comprehensive Cheat Sheet

> 📝 **Hands-On Labs:** [Monitoring Labs](./Labs/Azure-Monitoring-Labs.md)

> 🎯 **Exam Focus:** AZ-305 tests your ability to **design** monitoring architectures that balance operations, security, cost, and compliance.

---

## Table of Contents

- [Azure Monitoring Family Overview](#azure-monitoring-family-overview)
- [When to Choose Which — Decision Tree](#when-to-choose-which--decision-tree)
- [Metrics](#metrics)
- [Logs (Azure Monitor Logs / Log Analytics)](#logs-azure-monitor-logs--log-analytics)
- [Diagnostic Settings](#diagnostic-settings)
- [Application Insights](#application-insights)
- [Alerts](#alerts)
- [Workbooks](#workbooks)
- [Azure Advisor](#azure-advisor)
- [Service Health](#service-health)
- [Microsoft Defender for Cloud](#microsoft-defender-for-cloud)
- [Microsoft Sentinel (SIEM/SOAR)](#microsoft-sentinel-siemsoar)
- [Azure Arc (Monitoring Hybrid)](#azure-arc-monitoring-hybrid)
- [Availability & Resilience](#availability--resilience)
- [Cost Optimization](#cost-optimization)
- [Monitoring Design Patterns](#monitoring-design-patterns)
- [AZ-305 Decision Scenarios](#az-305-decision-scenarios)
- [Quick Reference Trigger Table](#quick-reference-trigger-table)
- [Common Exam Traps](#common-exam-traps)
- [🎯 Final AZ-305 Exam Tips](#-final-az-305-exam-tips)
- [📐 Architecture Decision Flowchart](#-architecture-decision-flowchart)
- [Exam-Style Review Questions](#exam-style-review-questions)

---

<a id="azure-monitoring-family-overview"></a>
## 1. Azure Monitoring Family Overview

| Service | Primary Role | Data / Store Focus | Best For | Architect Trigger |
|---------|---------------|--------------------|----------|-------------------|
| **Azure Monitor** | Umbrella observability platform | Metrics + logs + alerts + workbooks | Unified Azure-native monitoring strategy | Need one platform for telemetry collection, analysis, and action |
| **Log Analytics** | Central log store + KQL analysis | Workspace-based log ingestion and querying | Investigation, correlation, compliance, forensics | Need deep queries, cross-resource analysis, or retention control |
| **Application Insights** | Application performance monitoring (APM) | Requests, dependencies, traces, exceptions | App telemetry, distributed tracing, UX diagnostics | Need end-to-end app transaction visibility |
| **Metrics** | Near real-time numeric monitoring | Azure Monitor metrics database | Fast health signals, autoscale, threshold alerts | Need fast threshold alerting or lightweight time-series data |
| **Alerts** | Automated response engine | Metric, log, activity log, smart detection rules | Operations response and escalation | Need notifications, automation, or incident routing |
| **Defender for Cloud** | Security posture + workload protection | Secure score, recommendations, threat protections | CSPM, hardening, regulatory posture | Need posture management or workload protection guidance |
| **Microsoft Sentinel** | SIEM/SOAR and SOC analytics | Security logs in Log Analytics + analytics rules | Incident correlation, hunting, playbooks | Need centralized SOC, incident triage, or SOAR automation |

> 💡 **AZ-305 Tip:** Think of **Azure Monitor** as the platform, **Log Analytics** as the deep query engine, **Application Insights** as the APM layer, **Metrics/Alerts** as the fast detection path, **Defender** as security posture, and **Sentinel** as SecOps at scale.

### Azure Monitor
Azure Monitor is the **umbrella observability platform** for Azure and hybrid resources. It ingests telemetry from infrastructure, applications, guest OSs, and Azure services, then exposes that data through metrics, logs, dashboards, alerts, APIs, and integrations.

**Real-World Examples:**
- A platform team standardizes monitoring for App Service, SQL Database, and Key Vault across multiple subscriptions using one Azure-native control plane
- An operations team uses Azure Monitor alerts and workbooks to detect VM CPU spikes, failed deployments, and service degradations from a central dashboard
- A hybrid enterprise connects Azure VMs and Arc-enabled servers into one monitoring architecture with shared governance and action groups

### Log Analytics
Log Analytics is the **query and retention engine** for Azure Monitor logs. It stores operational, security, and platform telemetry in workspaces and lets architects use **KQL** for correlation, reporting, auditing, and long-term investigations.

**Real-World Examples:**
- A SOC correlates AzureActivity, SigninLogs, and SecurityAlert data to investigate suspicious administrative changes across subscriptions
- A production support team queries VM heartbeats and performance counters to identify systems that stopped sending telemetry after a patch window
- A regulated workload keeps searchable logs for 180 days in a dedicated workspace to satisfy audit requirements while separating prod from dev/test access

### Application Insights
Application Insights is the **APM layer** of Azure Monitor. It captures requests, dependencies, exceptions, traces, availability tests, and distributed traces so architects can design observability for application health and user experience.

**Real-World Examples:**
- A microservices application traces checkout requests across frontend, API, and payment dependencies to isolate latency bottlenecks
- A web application uses synthetic availability tests from multiple geographies to validate external availability before users report outages
- A development team tracks exceptions and custom business events to see whether a new release increased sign-up failures or payment drop-offs

### Metrics
Metrics are **lightweight numeric time-series signals** stored in the Azure Monitor metrics database. They are optimized for near real-time charting, threshold alerting, autoscale, and operational health indicators.

**Real-World Examples:**
- A VM scale set uses CPU percentage and queue length metrics to trigger autoscale rules during traffic spikes
- A database administrator watches DTU percentage and storage metrics to predict when Azure SQL capacity must be increased
- A support team creates a 1-minute metric alert when App Service HTTP 5xx counts exceed a threshold

### Alerts
Alerts are the **action layer** that turns telemetry into notifications, tickets, runbooks, webhooks, or remediation flows. Architecturally, alerts matter because the signal type, evaluation frequency, and routing design affect response speed, noise, and cost.

**Real-World Examples:**
- A critical production API sends Sev2 alerts to an action group that opens an ITSM ticket and calls a webhook for incident automation
- A platform team suppresses non-critical alerts during maintenance windows using alert processing rules instead of deleting alert logic
- A governance team creates activity log alerts for policy assignment changes and subscription-level service health incidents

### Defender for Cloud
Defender for Cloud is the **cloud security posture and workload protection service**. It surfaces secure score, misconfiguration recommendations, regulatory compliance posture, and protection signals for servers, databases, storage, and containers.

**Real-World Examples:**
- A security architect uses secure score recommendations to harden internet-exposed virtual machines and storage accounts
- A compliance team reviews the regulatory dashboard to measure alignment with internal standards and industry controls
- An operations team enables Defender plans for SQL and Servers to detect suspicious activity while improving baseline hardening posture

### Microsoft Sentinel
Microsoft Sentinel is Azure's **cloud-native SIEM/SOAR** built on Log Analytics. It ingests security data, applies analytics rules, groups incidents, supports hunting and investigation, and automates response through playbooks.

**Real-World Examples:**
- A global SOC aggregates identity, endpoint, firewall, and cloud workload logs into Sentinel for centralized triage and incident response
- A security team uses analytics rules and entity mapping to detect impossible travel and suspicious sign-ins, then triggers a Logic App playbook
- A regulated enterprise runs threat hunting queries across Azure, Microsoft 365, and third-party security connectors from one incident platform

### Azure Monitor Platform Context

Azure Monitor is the **unified observability platform** for Azure and hybrid resources. It collects, stores, analyzes, and acts on telemetry from infrastructure, platforms, applications, and security tools.

### Core Monitoring Flow

```text
Sources -> Collection -> Storage -> Analysis -> Response -> Integration
```

| Layer | What It Includes |
|------|-------------------|
| **Data sources** | Metrics, logs, traces, activity events, resource changes, guest OS telemetry, application telemetry |
| **Collection** | Azure Monitor Agent (AMA), diagnostic settings, Application Insights SDK/auto-instrumentation, Data Collection Rules (DCRs) |
| **Data stores** | **Metrics database** and **Log Analytics workspace** |
| **Consumption** | Metrics Explorer, Logs/KQL, Workbooks, Alerts, Dashboards, APIs, Event Hub integration |
| **Response** | Action Groups, Logic Apps, Functions, ITSM, webhooks, autoscale |

### Data Sources You Must Know for AZ-305

- **Metrics** -> numeric, time-series, near real-time, lightweight, ideal for alerting
- **Logs** -> rich, queryable, retained longer, ideal for investigations and analytics
- **Traces** -> distributed application flow and dependency correlation
- **Changes** -> configuration drift, activity log, update tracking, resource changes

### Two Data Stores = Core Exam Fact

| Store | Best For | Example |
|------|----------|---------|
| **Metrics database** | Fast numeric time-series analysis and near real-time alerting | CPU %, DTU %, request count |
| **Log Analytics workspace** | Deep analysis, correlation, retention, KQL | VM heartbeat, AzureActivity, AppRequests, SecurityEvent |

### Analyze, Visualize, Respond, Integrate

- **Analyze**: KQL, Metrics Explorer, Application Map, Transaction Search
- **Visualize**: Workbooks, dashboards, Azure portal views, Grafana integration
- **Respond**: alerts, autoscale, ITSM tickets, webhooks, Functions, Logic Apps
- **Integrate**: Event Hub, Storage, partner SIEM/APM, REST APIs, Power BI, Sentinel

### Azure Monitor vs Third-Party Tools

| Choose | When |
|-------|------|
| **Azure Monitor first** | Azure-native workloads, RBAC integration, Azure Policy enforcement, lower operational friction, unified control plane |
| **Third-party observability** | Strong multi-cloud requirement, existing enterprise standard, advanced APM/network analytics already standardized outside Azure |
| **Combined model** | Azure Monitor for collection/governance + third-party tool for enterprise correlation or executive reporting |

> 💡 **AZ-305 heuristic:** If the scenario emphasizes **Azure-native governance, Policy, RBAC, low-friction integration, and PaaS monitoring**, choose Azure Monitor first.

### Starter Commands

```bash
az monitor log-analytics workspace create \
  --resource-group rg-monitoring \
  --workspace-name law-prod-eastus \
  --location eastus
```

```powershell
New-AzOperationalInsightsWorkspace `
  -ResourceGroupName "rg-monitoring" `
  -Name "law-prod-eastus" `
  -Location "EastUS" `
  -Sku "PerGB2018"
```

---

<a id="when-to-choose-which--decision-tree"></a>
## 2. When to Choose Which — Decision Tree

```text
┌──────────────────────────────────────────────────────────────────────────────┐
│ What are you primarily trying to design?                                    │
└───────────────────────────────┬──────────────────────────────────────────────┘
                                │
                ┌───────────────┼───────────────────────────────┐
                │               │                               │
                ▼               ▼                               ▼
      ┌────────────────┐  ┌──────────────────────┐   ┌──────────────────────┐
      │ Fast health /  │  │ Deep investigation / │   │ Security posture or  │
      │ threshold alert│  │ retention / KQL      │   │ SOC operations       │
      └───────┬────────┘  └──────────┬───────────┘   └──────────┬───────────┘
              │                      │                          │
              ▼                      ▼                          ▼
     Use **Metrics**        Use **Log Analytics**      Need hardening posture?
              │                      │                  ├── YES -> **Defender for Cloud**
              │                      │                  └── NO  -> **Sentinel**
              │                      │
              ▼                      ▼
 Need application request     Need app code, traces,
 flow, dependencies,          availability tests, or
 exceptions, or UX?           distributed tracing?
 ├── YES -> **Application     ├── YES -> **Application Insights**
 │          Insights**        └── NO  -> **Log Analytics**
 └── NO  -> **Metrics**

 Need automated action after detection?
 └── Add **Alerts + Action Groups** to the chosen signal path
```

### Quick Decision Matrix

| Scenario | Best Fit |
|----------|----------|
| Need one Azure-native observability platform | **Azure Monitor** |
| Need deep log queries, retention, and correlation | **Log Analytics** |
| Need distributed tracing and request-level visibility | **Application Insights** |
| Need near real-time threshold alerting or autoscale | **Metrics** |
| Need notifications, webhooks, runbooks, or ITSM routing | **Alerts** |
| Need posture recommendations and secure score | **Defender for Cloud** |
| Need SIEM, incident investigation, and SOAR playbooks | **Microsoft Sentinel** |

---

<a id="metrics"></a>
## 3. Metrics

Metrics are **numeric time-series values** stored in the Azure Monitor metrics database.

### Platform Metrics vs Custom Metrics

| Type | Source | Example | Best Use |
|------|--------|---------|----------|
| **Platform metrics** | Emitted automatically by Azure resources | CPU %, DTU %, Transactions, Disk Read Ops/Sec | Health, threshold alerting, autoscale |
| **Custom metrics** | Sent by applications or agents | OrdersProcessed, QueueLagSeconds | Business KPI alerting, app-specific SLOs |

### Key Facts

- **Retention**: **93 days** for Azure Monitor metrics
- **Latency**: near real-time ingestion, ideal for fast alerting
- **Tool**: **Metrics Explorer** for charting, splitting, filtering, pinning, and alert creation
- **Dimensions**: allow filtering/splitting a metric by attributes such as instance, response code, API name, or geo

### Metric Dimensions Matter

If a metric supports dimensions, you can alert on a subset instead of the full aggregate.

**Example:** Alert only when `Http5xx > 20` for `Region = EastUS` and `Endpoint = /checkout`.

### Metrics vs Logs

| Use Metrics When | Use Logs When |
|------------------|---------------|
| You need fast threshold alerting | You need root-cause analysis |
| You need low-overhead time-series data | You need correlation across resources |
| You need autoscale inputs | You need long retention and historical forensics |
| The question says near real-time | The question says complex query or audit/investigation |

> 💡 **AZ-305 heuristic:** **Metrics for detection, logs for diagnosis**.

### Metrics Example

```bash
az monitor metrics list \
  --resource /subscriptions/<subId>/resourceGroups/rg-app/providers/Microsoft.Web/sites/contoso-api \
  --metric Requests \
  --interval PT1M \
  --aggregation Total
```

```kusto
// Example metric-style log analysis when metrics are not enough
AppRequests
| where TimeGenerated > ago(1h)
| summarize Requests=count(), FailureRate=100.0 * countif(Success == false) / count() by bin(TimeGenerated, 5m)
| render timechart
```

---

<a id="logs-azure-monitor-logs--log-analytics"></a>
## 4. Logs (Azure Monitor Logs / Log Analytics)

Logs are stored primarily in a **Log Analytics workspace** and queried with **Kusto Query Language (KQL)**.

### Workspace Design Patterns

| Pattern | Best For | Tradeoff |
|--------|----------|----------|
| **Centralized workspace** | Enterprise SOC, shared operations, central governance | Larger blast radius, more RBAC planning |
| **Decentralized workspaces** | Business unit autonomy, strict segmentation | Harder cross-team correlation |
| **Workspace per environment** | Dev/Test/Prod separation, retention isolation, chargeback | More management overhead |
| **Shared workspace across environments** | Smaller estates, easier correlation | Must use table/resource filters carefully |

### Architect Decision Guidance

- **Centralized** if the requirement is **single pane of glass**, SOC analytics, common policy, enterprise dashboards
- **Per environment** if the requirement stresses **isolation, chargeback, or different retention/compliance needs**
- **Per region** only when data residency or latency drives the design
- **Per app** is usually too fragmented unless compliance requires strict segregation

### Access Control Modes

| Mode | Meaning | Exam Relevance |
|------|---------|----------------|
| **Workspace-context access** | Access based on workspace permissions | Simpler enterprise operations |
| **Resource-context access** | Access based on Azure resource RBAC | Better for app/resource owners who should only see logs for resources they manage |

> 💡 **Exam tip:** If the scenario says developers should query logs only for resources they own without broad workspace rights, think **resource-context access**.

### Data Collection Rules (DCR)

DCRs define **what data to collect, how to transform it, and where to send it**. They are the policy layer for **Azure Monitor Agent (AMA)**.

Use DCRs for:
- Windows/Linux performance counters
- Windows events / Syslog
- Custom text logs
- Filtering before ingestion
- Sending data to one or more destinations

```bash
az monitor data-collection rule create \
  --resource-group rg-monitoring \
  --name dcr-prod-vm \
  --location eastus
```

### Tables and Schemas

| Table Type | Example |
|-----------|---------|
| **Platform tables** | `AzureActivity`, `Heartbeat`, `Perf`, `InsightsMetrics` |
| **App Insights tables** | `AppRequests`, `AppDependencies`, `AppExceptions`, `AppTraces` |
| **Security tables** | `SecurityAlert`, `SecurityEvent`, `SigninLogs` |
| **Diagnostic tables** | `AzureDiagnostics` or resource-specific tables |

### Resource-Specific Tables vs AzureDiagnostics

- **Resource-specific mode** -> modern, better schema, better performance, easier per-service analysis
- **AzureDiagnostics mode** -> legacy catch-all table, less ideal at scale

### Retention and Archiving

- **Default analytics retention**: **30 days**
- **Configurable analytics retention**: up to **730 days** (2 years)
- **Archive tier**: up to **12 years** with lower cost; data can be restored to analytics tier for querying when needed
- For compliance scenarios requiring immutable copies outside Log Analytics, use **Storage export** via diagnostic settings or data export rules

| Retention Tier | Duration | Cost | Query Capability |
|---|---|---|---|
| **Interactive (Analytics)** | 30-730 days | Higher | Full KQL, alerting |
| **Archive** | Up to 12 years | Lower | Restore required before querying |
| **Storage export** | Unlimited | Lowest | External tools required |

> 💡 **AZ-305 tip:** For compliance scenarios, consider **archive tier** for long-term retention within Log Analytics, and **Storage export** for immutable audit trails.

```bash
az monitor log-analytics workspace update \
  --resource-group rg-monitoring \
  --workspace-name law-prod-eastus \
  --retention-time 180
```

```powershell
Set-AzOperationalInsightsWorkspace `
  -ResourceGroupName "rg-monitoring" `
  -Name "law-prod-eastus" `
  -RetentionInDays 180
```

### Basic vs Analytics Logs

| Tier | Best For | Tradeoff |
|------|----------|----------|
| **Analytics logs** | Full query experience, alerting, advanced hunting, frequent analysis | Higher cost |
| **Basic logs** | High-volume, low-value, infrequently queried data | Lower cost, reduced analytics capabilities |

> 💡 **AZ-305 heuristic:** Put **high-value operational/security data** in **Analytics**. Put **high-volume diagnostic data with occasional access** in **Basic** if the feature set is sufficient.

### Dedicated Clusters / Commitment Tiers

- Use **commitment tiers** when ingestion volume is predictable and high
- Use **dedicated clusters** for large enterprises needing isolation/performance and large-scale workspace strategy
- Architect lens: optimize **cost per GB** and governance consistency, not just raw feature count

### Cross-Workspace Query Example

```kusto
union
  workspace('law-prod-eastus').Heartbeat,
  workspace('law-dr-centralus').Heartbeat
| where TimeGenerated > ago(15m)
| summarize LastSeen=max(TimeGenerated) by Computer, _ResourceId
```

### KQL Basics You Must Know

| Operator | Purpose | Example |
|---------|---------|---------|
| `where` | Filter rows | `| where TimeGenerated > ago(1h)` |
| `project` | Select columns | `| project TimeGenerated, Computer, CounterValue` |
| `summarize` | Aggregate | `| summarize avg(CounterValue) by Computer` |
| `join` | Correlate data | `Heartbeat | join kind=inner Perf on Computer` |

### Real KQL Queries for Exam Prep

```kusto
// 1. Top failing application requests
AppRequests
| where TimeGenerated > ago(24h)
| where Success == false
| summarize Failures=count() by Name, ResultCode
| top 10 by Failures desc
```

```kusto
// 2. CPU trend for VMs
Perf
| where TimeGenerated > ago(1h)
| where ObjectName == "Processor" and CounterName == "% Processor Time" and InstanceName == "_Total"
| summarize AvgCPU=avg(CounterValue) by Computer, bin(TimeGenerated, 5m)
| render timechart
```

```kusto
// 3. Heartbeat gap detection
Heartbeat
| summarize LastHeartbeat=max(TimeGenerated) by Computer
| where LastHeartbeat < ago(10m)
```

```kusto
// 4. Activity log changes to production resources
AzureActivity
| where TimeGenerated > ago(24h)
| where ResourceGroup =~ "rg-prod"
| project TimeGenerated, Caller, OperationNameValue, ActivityStatusValue, ResourceId
```

```kusto
// 5. Join VM heartbeat with performance data
Heartbeat
| where TimeGenerated > ago(30m)
| summarize LastHeartbeat=max(TimeGenerated) by Computer
| join kind=inner (
    Perf
    | where TimeGenerated > ago(30m)
    | where ObjectName == "Memory" and CounterName == "Available MBytes"
    | summarize AvgFreeMB=avg(CounterValue) by Computer
) on Computer
| project Computer, LastHeartbeat, AvgFreeMB
```

### Architect Guidance

- Use **metrics** for fast alerts
- Use **logs** for deep analysis and audit/compliance
- Use **multiple workspaces intentionally**, not accidentally
- Design for **RBAC, retention, and chargeback** up front

---

<a id="diagnostic-settings"></a>
## 5. Diagnostic Settings

Diagnostic settings route platform logs and metrics from Azure resources to downstream destinations.

### What Supports Diagnostic Settings?

Most Azure resource providers support diagnostic settings, including:
- VMs
- Key Vault
- Storage Accounts
- App Service
- SQL Database / Managed Instance
- Azure Firewall
- Application Gateway
- AKS
- Cosmos DB
- Event Hubs
- Recovery Services vaults

### Supported Destinations

| Destination | Best For |
|------------|----------|
| **Log Analytics workspace** | Central analysis, alerting, KQL, workbooks |
| **Storage account** | Long-term retention, compliance, low-cost archive |
| **Event Hub** | Stream to SIEM/SOAR or external analytics platform |
| **Partner solution** | Direct integration with approved ecosystem tools |

### Categories

- **Logs** -> events, audit trails, engine-specific operations
- **Metrics** -> time-series values where the resource supports export

### Resource-Specific vs Azure Diagnostics Mode

| Mode | Recommendation |
|------|----------------|
| **Resource-specific** | Preferred for modern designs |
| **AzureDiagnostics** | Legacy/compatibility only |

### Policy Automation

Use **Azure Policy DeployIfNotExists** to enforce baseline diagnostic settings across subscriptions and landing zones.

> 💡 **Exam tip:** If the requirement says **automatically enable diagnostics for all new resources**, think **Azure Policy + DeployIfNotExists**.

### Diagnostic Settings Examples

```bash
az monitor diagnostic-settings create \
  --name send-to-law \
  --resource /subscriptions/<subId>/resourceGroups/rg-app/providers/Microsoft.KeyVault/vaults/kv-prod-01 \
  --workspace /subscriptions/<subId>/resourceGroups/rg-monitoring/providers/Microsoft.OperationalInsights/workspaces/law-prod-eastus \
  --logs '[{"category":"AuditEvent","enabled":true}]' \
  --metrics '[{"category":"AllMetrics","enabled":true}]'
```

```powershell
$law = Get-AzOperationalInsightsWorkspace -ResourceGroupName "rg-monitoring" -Name "law-prod-eastus"
$kv = Get-AzResource -ResourceGroupName "rg-app" -Name "kv-prod-01" -ResourceType "Microsoft.KeyVault/vaults"
New-AzDiagnosticSetting `
  -Name "send-to-law" `
  -ResourceId $kv.ResourceId `
  -WorkspaceId $law.ResourceId `
  -Enabled $true
```

### Design Guidance

- Send **operational logs** to Log Analytics
- Send **regulatory retention** data to Storage
- Send **external SIEM feed** to Event Hub
- Standardize categories with Policy so teams do not manually configure every resource

---

<a id="application-insights"></a>
## 6. Application Insights

Application Insights is the **application performance monitoring (APM)** part of Azure Monitor.

### Workspace-Based vs Classic

- **Use workspace-based Application Insights** for modern designs
- It unifies application telemetry with Log Analytics, Defender, and Sentinel workflows
- **Classic** is legacy and should not be the architect default

### Instrumentation Choices

| Option | Best For | Tradeoff |
|-------|----------|----------|
| **Auto-instrumentation** | Fast onboarding, App Service/VM scenarios, limited code change | Less customization |
| **SDK instrumentation** | Deep telemetry control, custom events, business metrics, distributed tracing | More implementation effort |

### Telemetry Types

- **Requests**
- **Dependencies**
- **Exceptions**
- **Traces**
- **Page views**
- **Custom events / custom metrics**

### Features You Must Know

| Feature | Why It Matters |
|--------|-----------------|
| **Application Map** | Dependency topology and blast-radius view |
| **Live Metrics** | Near real-time debugging without waiting for full ingestion |
| **Availability tests** | Synthetic tests from multiple locations |
| **Smart detection** | Automatic anomaly detection and issue identification |
| **Distributed tracing** | Correlates requests across services using operation IDs |

### Sampling Strategies

| Strategy | Use When |
|---------|----------|
| **Adaptive sampling** | High-volume apps, dynamic control |
| **Fixed-rate sampling** | Predictable ingestion/cost control |
| **Ingestion sampling** | Reduce cost after data reaches service |

> ⚠️ **Exam trap:** Sampling reduces cost but can reduce fidelity. If the requirement says **full forensic capture** or **every transaction must be retained**, be careful with aggressive sampling.

### Connection String vs Instrumentation Key

- Prefer **connection string**
- Treat standalone instrumentation key guidance as legacy/older design language
- Workspace-based architecture aligns naturally with connection-string configuration

### Application Insights Example

```bash
az monitor app-insights component create \
  --app contoso-api-ai \
  --location eastus \
  --resource-group rg-app \
  --workspace /subscriptions/<subId>/resourceGroups/rg-monitoring/providers/Microsoft.OperationalInsights/workspaces/law-prod-eastus \
  --application-type web
```

### Useful KQL

```kusto
// Slow dependencies by target
AppDependencies
| where TimeGenerated > ago(6h)
| summarize AvgDurationMs=avg(DurationMs), Calls=count() by Target, DependencyType
| top 10 by AvgDurationMs desc
```

```kusto
// Exceptions correlated with requests
AppExceptions
| where TimeGenerated > ago(24h)
| summarize Exceptions=count() by ProblemId, Method, OuterMessage
| top 10 by Exceptions desc
```

### Architect Guidance

- Use Application Insights for **user experience + transaction flow**, not just raw logging
- Pair it with **availability tests + action groups** for critical apps
- Use workspace-based mode to align with **central monitoring and security analytics**

---

<a id="alerts"></a>
## 7. Alerts

Alerts turn telemetry into action.

### Alert Types

| Type | Best For |
|------|----------|
| **Metric alerts** | Fast threshold-based detection |
| **Log alerts** | Complex KQL-based conditions |
| **Activity log alerts** | Control plane events, policy changes, service health |
| **Smart detection alerts** | App Insights anomaly-driven findings |

### Alert Rule Components

- **Scope** -> target resource/resource group/subscription
- **Condition** -> threshold, KQL query, signal logic
- **Action** -> Action Group
- **Details** -> severity, rule name, description, frequency

### Severity Levels

| Severity | Meaning |
|---------|---------|
| **0** | Critical production outage / security incident |
| **1** | High impact, immediate attention |
| **2** | Significant but not outage |
| **3** | Warning / degraded condition |
| **4** | Informational |

### Action Groups

Action Groups can trigger:
- Email
- SMS
- Push/voice (where supported)
- Webhook
- Logic App
- Azure Function
- Automation Runbook
- ITSM connector

### Alert Processing Rules

Use alert processing rules to:
- suppress noisy alerts during maintenance windows
- route alerts differently after hours
- change action groups without editing every rule

### Common Alert Schema

Use **common alert schema** to normalize payloads across alert types for downstream automation.

### Alert State Management

- Design stateful logic where supported to reduce duplicate noise
- Pair with deduplication and maintenance suppression
- Monitor alert volume, not just resource health

### Cost Considerations

- Metric alerts are typically cheaper and faster for simple thresholds
- Log alerts are more powerful but can cost more, especially with frequent evaluation and large query scope
- Over-alerting is both an **operations cost** and a **human reliability issue**

### Alert Examples

```bash
az monitor action-group create \
  --resource-group rg-monitoring \
  --name ag-critical \
  --short-name critops
```

```bash
az monitor metrics alert create \
  --resource-group rg-monitoring \
  --name vm-high-cpu \
  --scopes /subscriptions/<subId>/resourceGroups/rg-app/providers/Microsoft.Compute/virtualMachines/vm-prod-01 \
  --condition "avg Percentage CPU > 80" \
  --window-size 5m \
  --evaluation-frequency 1m \
  --severity 2 \
  --action ag-critical
```

```kusto
// Log alert query: repeated authentication failures
SigninLogs
| where TimeGenerated > ago(15m)
| where ResultType != 0
| summarize FailedAttempts=count() by UserPrincipalName, IPAddress
| where FailedAttempts >= 10
```

### Architect Guidance

- Use **metric alerts** for fast, common health conditions
- Use **log alerts** where correlation is required
- Use **activity log alerts** for governance/control-plane changes
- Use **alert processing rules** to fight alert fatigue

---

<a id="workbooks"></a>
## 8. Workbooks

Workbooks provide **interactive reporting and operational dashboards**.

### What They Are Good At

- Consolidating metrics + logs + Resource Graph into one view
- Building role-specific dashboards for ops, architects, security, executives
- Enabling parameters and drill-downs without building a custom app

### Templates vs Custom Workbooks

| Option | When to Use |
|-------|--------------|
| **Templates** | Quick start, standard Azure scenarios |
| **Custom workbooks** | Enterprise dashboards, specific KPIs, cross-resource drill-through |

### Data Sources

- Log Analytics
- Metrics
- Azure Resource Graph
- Azure Resource Health / Azure Monitor data sources

### Parameters and Filters

Common parameters:
- subscription
- region
- environment
- application name
- severity
- time range

### Sharing and Permissions

- Workbook access follows Azure RBAC on the workbook plus underlying data source access
- Sharing a workbook does **not** bypass workspace/resource permissions

### Architect Guidance

Use Workbooks when the question asks for:
- **single pane of glass**
- **interactive troubleshooting dashboard**
- **cross-subscription visual reporting**
- **operations dashboard without custom code**

---

<a id="azure-advisor"></a>
## 9. Azure Advisor

Azure Advisor provides architecture recommendations across five categories.

### Five Categories

| Category | Focus |
|---------|-------|
| **Reliability** | Resilience, redundancy, backup, HA |
| **Security** | Exposure reduction, hardening |
| **Performance** | Efficiency and sizing |
| **Cost** | Waste reduction and rightsizing |
| **Operational Excellence** | Best practices and manageability |

### Important Concepts

- **Recommendation digest** -> curated recommendations for improvement
- **Suppressing recommendations** -> hide accepted/irrelevant items without deleting the service capability
- **Advisor score** -> tracks improvement posture over time

```bash
az advisor recommendation list --category Cost
```

> 💡 **Exam tip:** Advisor is about **recommendations**, not deep telemetry analysis. If the scenario asks for **continuous real-time monitoring**, the answer is not Advisor.

---

<a id="service-health"></a>
## 10. Service Health

### Know the Three Services

| Service | Scope | Use |
|--------|-------|-----|
| **Azure Status** | Public/global | Broad public service outage visibility |
| **Service Health** | Your tenant/subscription | Personalized incidents, planned maintenance, advisories |
| **Resource Health** | Specific resource | Whether an individual resource is available/unavailable/degraded |

### Key Capabilities

- **Health alerts** for incidents affecting your subscriptions
- **Planned maintenance notifications**
- **Service issue visibility** scoped to your tenant
- **Root Cause Analysis (RCA)** access after major incidents

> 💡 **AZ-305 heuristic:**
> - Need **personalized tenant impact** -> **Service Health**
> - Need **single resource condition** -> **Resource Health**
> - Need **public internet-facing status page** -> **Azure Status**

### Architect Guidance

Use Service Health for:
- executive communications
- incident response routing
- planned maintenance planning
- subscription-scoped awareness of platform events

---

<a id="microsoft-defender-for-cloud"></a>
## 11. Microsoft Defender for Cloud

Microsoft Defender for Cloud is the cloud-native security posture and workload protection service.

### What It Does

| Capability | Meaning |
|-----------|---------|
| **CSPM** | Cloud Security Posture Management - assesses configuration/security posture |
| **Secure score** | Quantified posture improvement score |
| **Recommendations** | Remediation guidance for misconfigurations |
| **Regulatory compliance dashboard** | Maps environment posture to standards/frameworks |
| **CWP plans** | Workload protection for servers, databases, storage, containers, etc. |

### Defender Plans by Resource Type

Examples include plans for:
- Servers
- SQL
- Storage
- Containers / Kubernetes
- App Service
- Key Vault
- DNS / ARM / APIs depending on workload coverage and plan options

### Recommendations and Remediation

- Prioritize by severity and blast radius
- Use Policy/automation for repeatable remediation
- Secure score helps drive governance conversations with leadership

### Integration with Sentinel

- Defender for Cloud surfaces posture + protection findings
- Sentinel adds **SIEM/SOAR**, correlation, incidents, automation, threat hunting
- Together they create a stronger SecOps design

```bash
az security pricing create --name VirtualMachines --tier Standard
```

```bash
az security assessment list --resource-group rg-app
```

### Architect Guidance

- Use Defender for Cloud when the question is about **posture, hardening, recommendations, and workload protection**
- Do not confuse it with a full SIEM

---

<a id="microsoft-sentinel-siemsoar"></a>
## 12. Microsoft Sentinel (SIEM/SOAR)

Sentinel is Microsoft's cloud-native **SIEM/SOAR** built on Log Analytics.

### Sentinel vs Defender for Cloud

| Need | Choose |
|------|--------|
| Security posture management, recommendations, secure score | **Defender for Cloud** |
| SIEM, incident correlation, SOC workflows, playbooks | **Sentinel** |
| End-to-end cloud security operations | **Both** |

### Key Components

| Component | Purpose |
|----------|---------|
| **Data connectors** | Ingest Microsoft and third-party security data |
| **Analytics rules** | Generate alerts/incidents from detections |
| **Incidents** | Case management and triage unit |
| **Investigation graph** | Entity relationships and attack chain context |
| **Playbooks** | Logic App-based automation for SOAR |
| **Workbooks** | Security dashboards and hunting views |

### Cost Considerations

- Sentinel is **ingestion-based**
- Design for data tiering, selective connector enablement, retention strategy, and noise reduction
- Cost is driven by **volume + retention + analytics scope**, not just license presence

### Example KQL for Sentinel Hunting

```kusto
SigninLogs
| where TimeGenerated > ago(1d)
| where ResultType == 0
| summarize SuccessfulLogins=count(), DistinctIPs=dcount(IPAddress) by UserPrincipalName
| where DistinctIPs > 5
| top 20 by DistinctIPs desc
```

```kusto
SecurityAlert
| where TimeGenerated > ago(24h)
| project TimeGenerated, AlertName, CompromisedEntity, Severity, ProviderName
| order by TimeGenerated desc
```

### Architect Guidance

- Choose Sentinel when the requirement says **centralized SIEM**, **SOC**, **incident investigation**, or **SOAR playbooks**
- Choose Defender for Cloud when the requirement says **posture** or **hardening recommendations**

---

<a id="azure-arc-monitoring-hybrid"></a>
## 13. Azure Arc (Monitoring Hybrid)

Azure Arc extends Azure management and monitoring to on-premises and multicloud servers.

### Core Concepts

| Capability | Why It Matters |
|-----------|-----------------|
| **Arc-enabled servers** | Bring hybrid servers under Azure management plane |
| **Monitoring extensions** | Install AMA and related extensions on hybrid resources |
| **Policy integration** | Enforce configuration and agent deployment at scale |
| **Hybrid observability** | One operational model across Azure + non-Azure estate |

### AMA vs Log Analytics Agent

| Agent | Status | Guidance |
|------|--------|----------|
| **Azure Monitor Agent (AMA)** | Current strategic agent | Use for new designs |
| **Log Analytics agent / MMA / OMS agent** | ⚠️ **Deprecated August 31, 2024** | Do not choose for new architectures |

> **Important:** The Log Analytics agent (also known as MMA or OMS agent) was officially deprecated on **August 31, 2024**. All new monitoring designs must use the **Azure Monitor Agent (AMA)** with **Data Collection Rules (DCRs)**. Migrate existing deployments to AMA before support ends.

### Hybrid Monitoring Architecture

```text
On-prem / Other Cloud Servers
        -> Azure Arc
        -> Azure Monitor Agent
        -> DCR
        -> Log Analytics / Metrics / Sentinel / Defender
```

### Arc Monitoring Commands

```bash
az connectedmachine extension create \
  --resource-group rg-hybrid \
  --machine-name arc-sql-01 \
  --name AzureMonitorWindowsAgent \
  --publisher Microsoft.Azure.Monitor \
  --type AzureMonitorWindowsAgent
```

```powershell
New-AzConnectedMachineExtension `
  -ResourceGroupName "rg-hybrid" `
  -MachineName "arc-sql-01" `
  -Name "AzureMonitorWindowsAgent" `
  -Publisher "Microsoft.Azure.Monitor" `
  -ExtensionType "AzureMonitorWindowsAgent" `
  -Location "EastUS"
```

### Architect Guidance

- Use Azure Arc when the requirement is **consistent monitoring/governance for hybrid servers**
- Standardize on **AMA + DCR + Policy** for enterprise rollout

---

<a id="vm-insights"></a>
## 13b. VM Insights

VM Insights is a pre-built monitoring solution for **Azure VMs and Arc-enabled servers** that provides performance and dependency visualization.

### What VM Insights Provides

| Feature | Purpose |
|---------|---------|
| **Performance view** | CPU, memory, disk, and network metrics with historical trends |
| **Map view** | Interactive dependency visualization showing connections between VMs and external services |
| **Health monitoring** | Guest OS health signals and alerting |
| **At-scale monitoring** | Monitor hundreds of VMs from a single view |

### Requirements

- **Azure Monitor Agent (AMA)** with appropriate DCR configuration
- **Dependency Agent** for Map functionality (optional but recommended)
- Log Analytics workspace for data storage

### When to Use VM Insights

| Scenario | VM Insights Value |
|----------|-------------------|
| Understanding application dependencies | Map view shows service-to-service connections |
| Capacity planning | Performance trends help predict resource needs |
| Troubleshooting performance issues | Correlate CPU, memory, disk, and network metrics |
| Hybrid monitoring | Consistent view for Azure VMs and Arc-enabled servers |

### Enable VM Insights

```bash
# Enable VM Insights using Azure Policy (recommended for scale)
az policy assignment create \
  --name "enable-vm-insights" \
  --policy-set-definition "Enable Azure Monitor for VMs" \
  --scope "/subscriptions/<sub-id>"
```

```powershell
# Enable for a single VM
Set-AzVMExtension `
  -ResourceGroupName "rg-app" `
  -VMName "vm-prod-01" `
  -Name "AzureMonitorWindowsAgent" `
  -Publisher "Microsoft.Azure.Monitor" `
  -ExtensionType "AzureMonitorWindowsAgent" `
  -TypeHandlerVersion "1.0"
```

**Exam tip:** If the scenario asks for **VM dependency mapping**, **at-scale VM monitoring**, or **understanding which services connect to which**, think **VM Insights**.

---

<a id="change-analysis"></a>
## 13c. Change Analysis

Azure Monitor Change Analysis detects **configuration changes** to Azure resources, helping with root cause analysis and troubleshooting.

### What Change Analysis Tracks

| Change Type | Examples |
|-------------|----------|
| **Resource property changes** | VM size change, IP address modification |
| **Configuration changes** | Web app settings, connection strings |
| **Dependency changes** | New service connections, removed integrations |
| **Code/content changes** | App Service file changes (requires integration) |

### Key Features

- **Timeline view**: See what changed and when
- **Diff view**: Compare before and after states
- **Correlation**: Link changes to application issues
- **Integration**: Works with Application Insights and App Service diagnostics

### When to Use Change Analysis

| Scenario | Change Analysis Value |
|----------|----------------------|
| "The app broke after deployment" | See exactly what configuration changed |
| Performance degradation investigation | Correlate changes with performance timeline |
| Security incident response | Identify unauthorized configuration changes |
| Audit and compliance | Track configuration drift over time |

### Enable Change Analysis

```bash
# Register the Change Analysis resource provider
az provider register --namespace Microsoft.ChangeAnalysis

# Change Analysis is automatically enabled for most Azure resources
# For web app in-guest changes, enable in App Service Diagnostic settings
```

**Exam tip:** If the scenario asks **"what changed before the incident?"** or mentions **root cause analysis** and **configuration troubleshooting**, think **Change Analysis**.

---

<a id="container-monitoring"></a>
## 13d. Container & Kubernetes Monitoring

For containerized workloads, Azure provides specialized monitoring through **Container Insights**, **Azure Monitor managed service for Prometheus**, and **Azure Managed Grafana**.

### Monitoring Options

| Service | Best For | Data Model |
|---------|----------|------------|
| **Container Insights** | AKS monitoring with Log Analytics integration | Logs + Metrics |
| **Azure Monitor managed Prometheus** | Kubernetes-native metrics at scale | Prometheus metrics |
| **Azure Managed Grafana** | Dashboards for Prometheus and Azure Monitor | Visualization layer |

### Container Insights

Container Insights provides comprehensive monitoring for AKS and Arc-enabled Kubernetes clusters:

- **Node and pod performance metrics**
- **Container logs and stdout/stderr**
- **Kubernetes events**
- **Live data streaming**
- **Recommended alerts**

### Azure Monitor managed service for Prometheus

For Kubernetes-native monitoring, Azure offers a fully managed Prometheus service:

| Feature | Value |
|---------|-------|
| **Prometheus compatibility** | Standard Prometheus query language (PromQL) |
| **Scale** | Handles large-scale Kubernetes environments |
| **No infrastructure management** | Fully managed by Azure |
| **Native AKS integration** | Easy enablement for AKS clusters |

### Azure Managed Grafana

Managed Grafana provides visualization for:
- Azure Monitor metrics
- Prometheus metrics
- Log Analytics data
- Azure Data Explorer

### Decision Matrix

| Need | Choose |
|------|--------|
| Quick AKS monitoring with Log Analytics | **Container Insights** |
| Prometheus-native monitoring at scale | **Azure Monitor managed Prometheus** |
| Advanced dashboards and PromQL | **Azure Managed Grafana** |
| Complete Kubernetes observability stack | **All three together** |

```bash
# Enable Container Insights for AKS
az aks enable-addons \
  --resource-group rg-aks \
  --name aks-prod \
  --addons monitoring \
  --workspace-resource-id /subscriptions/<sub-id>/resourceGroups/rg-monitoring/providers/Microsoft.OperationalInsights/workspaces/law-prod

# Enable Prometheus metrics collection
az aks update \
  --resource-group rg-aks \
  --name aks-prod \
  --enable-azure-monitor-metrics
```

**Exam tip:** If the scenario mentions **AKS**, **Kubernetes**, or **container monitoring**, think **Container Insights** for logs and basic metrics, **Prometheus** for Kubernetes-native metrics, and **Grafana** for dashboards.

---

<a id="availability--resilience"></a>
## 14. Availability & Resilience

Monitoring design must remain useful **during platform incidents, regional failures, and application outages**. AZ-305 often rewards answers that keep telemetry, alerting, and visibility available even when the primary workload is degraded.

### Resilience Design Priorities

| Requirement | Recommended Design Choice | Why It Matters |
|-------------|---------------------------|----------------|
| Regional workload failover | Use multi-region monitoring views, cross-workspace queries, and regional dashboards | Ops must keep visibility after failover, not just after recovery |
| Monitoring platform survivability | Avoid overly fragile single-region assumptions for critical telemetry | Central monitoring that disappears with the workload is a design flaw |
| Endpoint availability checks | Use Application Insights availability tests from multiple geographies | Synthetic monitoring can detect user-facing outages before internal teams notice |
| Platform incident visibility | Use Service Health + Resource Health + activity log alerts | Teams need tenant-scoped awareness during Azure-side incidents |
| Action continuity | Route critical alerts through resilient Action Groups and multiple notification paths | One broken notification target should not silence incident response |

### Availability Patterns

| Pattern | Best For | Tradeoff |
|--------|----------|----------|
| **Centralized monitoring workspace** | Single-pane-of-glass operations and SOC analytics | Must plan carefully for regional dependencies and RBAC |
| **Regional workspaces with cross-workspace queries** | Multi-region mission-critical apps and data residency requirements | More workspace management overhead |
| **Workspace per environment + regional dashboards** | Regulated prod workloads with failover visibility | Additional governance and reporting design |
| **Synthetic availability + health alerts** | Internet-facing applications with strict SLAs | Adds extra monitoring components and cost |

### Architect Guidance

- Design monitoring so **telemetry and alerts outlive the outage scenario**
- Use **Service Health** for tenant impact, **Resource Health** for resource condition, and **Application Insights availability tests** for user-path verification
- For mission-critical workloads, monitor **primary and failover regions separately and together**
- Validate that alerting paths include **multiple recipients or automation targets**, not a single fragile endpoint

> 💡 **AZ-305 heuristic:** If the app is deployed across regions, the monitoring design should also show **regional awareness, failover visibility, and incident communications**.

---

<a id="cost-optimization"></a>
## 15. Cost Optimization

Monitoring cost is driven mainly by **log ingestion, retention, analytics usage, alert frequency, and security data volume**. AZ-305 questions often test whether you can reduce cost **without losing the signals that matter**.

### Primary Cost Drivers

| Cost Driver | Why It Increases Spend | Optimization Lever |
|-------------|------------------------|--------------------|
| **Log ingestion** | High-volume diagnostics, verbose app telemetry, noisy connectors | Filter at source, use DCR transforms, sample app telemetry, avoid duplicate collection |
| **Analytics retention** | Keeping interactive data for long periods | Retain searchable data only as long as needed; archive/export long-term data |
| **Basic vs Analytics logs** | Using Analytics for low-value, infrequently queried data | Move suitable high-volume logs to **Basic logs** |
| **Sentinel ingestion** | Broad connector enablement and long retention | Enable only needed connectors/tables and tune detections |
| **Alert evaluations** | Frequent log queries over broad scopes | Prefer metric alerts for simple thresholds and reduce noisy rule frequency |
| **Workspace sprawl** | Duplicate data and fragmented chargeback | Standardize workspace strategy and routing rules |

### Log Ingestion and Retention Strategy

| Requirement | Cost-Aware Choice |
|-------------|-------------------|
| Need 1-minute CPU threshold alert | **Metric alert**, not log query alert |
| Need rare access to massive verbose diagnostics | **Basic logs** or archive/export pattern |
| Need long-term audit retention at low cost | Send copies to **Storage** and keep shorter interactive retention in Log Analytics |
| Need predictable, high daily ingestion | Evaluate **commitment tiers** or **dedicated clusters** |
| Need app insights without runaway spend | Use **adaptive/fixed-rate sampling** where full fidelity is not mandatory |

### Practical Cost Controls

- Collect **high-value operational and security data** first; do not default to collecting every possible category
- Use **resource-specific tables** and DCR filtering to avoid paying for low-value noise
- Keep **critical analytics data** in Analytics logs and move lower-value, infrequently queried data to **Basic logs** where feature limitations are acceptable
- Review **retention by workspace/environment** so dev/test does not inherit production retention costs
- Revisit **Sentinel connector scope, alert volume, and workbook/query usage** as part of ongoing optimization

> ⚠️ **Exam trap:** The cheapest design is not the best design. AZ-305 expects the **lowest-cost architecture that still satisfies operations, security, compliance, and investigation requirements**.

---

<a id="monitoring-design-patterns"></a>
## 16. Monitoring Design Patterns

### 1) Single Pane of Glass Architecture

- Central Log Analytics workspace strategy
- Standard diagnostic settings via Policy
- Workbooks for operations/SOC views
- Shared Action Groups and routing logic

### 2) Multi-Tier Application Monitoring

Monitor each layer separately and together:
- **Frontend** -> availability tests, response time, failure rate
- **API/App tier** -> requests, dependencies, exceptions, traces
- **Data tier** -> DTU/vCore, deadlocks, latency, storage, failover health

### 3) Microservices Observability

Use:
- Application Insights distributed tracing
- Correlation IDs
- dependency maps
- KQL across services
- noise-aware sampling strategy

### 4) Infrastructure vs Application Monitoring

| Layer | Typical Signals |
|------|------------------|
| **Infrastructure** | CPU, memory, disk, network, heartbeat, platform availability |
| **Application** | request latency, error rate, dependency failures, business transactions |

### 5) Security Monitoring Integration

- Defender for Cloud for posture/protection
- Sentinel for SIEM/SOAR
- Azure Monitor + Activity Log + diagnostics for control-plane visibility

### 6) Cost-Effective Monitoring Design

- Metrics for fast/simple alerting
- Logs for deep investigations only where needed
- Basic logs for lower-value, high-volume telemetry
- Commitment tiers for stable large ingestion
- Sampling for noisy application telemetry
- Policy-driven standardization to prevent duplicate collection

> 💡 **Architect rule:** Collect everything only if you can afford to store, search, and act on it. Good observability is **intentional**, not just comprehensive.

---

<a id="az-305-decision-scenarios"></a>
## 17. AZ-305 Decision Scenarios

### Scenario 1 - Enterprise Log Analytics Workspace Design
A global company wants a single SOC view across 40 subscriptions, but app teams should only see their own resources.

**Best answer:** Centralized workspace strategy with **resource-context access** and RBAC.

**Why:** Central SOC correlation + reduced broad workspace exposure for app teams.

### Scenario 2 - Multi-Region Monitoring Architecture
A mission-critical app runs in East US and Central US. Operations wants failover visibility even if one region is unavailable.

**Best answer:** Dual-region telemetry collection with cross-workspace queries or resilient centralized monitoring design, plus Service Health and regional dashboards.

**Why:** Monitoring must survive regional failure, not just the application.

### Scenario 3 - Application Performance Monitoring Strategy
Developers need request traces across web app, API, and database calls with minimal code delay.

**Best answer:** Workspace-based **Application Insights**, auto-instrumentation where possible, SDK where custom events or richer tracing is required.

### Scenario 4 - Security Monitoring and SIEM Decision
The customer wants posture recommendations for Azure resources and also a SOC-driven incident platform with playbooks.

**Best answer:** **Defender for Cloud + Sentinel**.

**Why:** Defender = posture/protection. Sentinel = SIEM/SOAR.

### Scenario 5 - Alert Fatigue Reduction
Ops receives hundreds of repeated alerts during planned maintenance.

**Best answer:** Use **alert processing rules** to suppress or reroute alerts during maintenance windows.

### Scenario 6 - Cost Optimization for Monitoring
A platform team ingests massive low-value verbose diagnostics, but only investigates them occasionally.

**Best answer:** Move suitable data to **Basic logs** or archive/export patterns; keep critical operational/security data in **Analytics**.

### Scenario 7 - Hybrid Monitoring with Arc
A company must monitor on-premises Windows and Linux servers using the same Azure operational model as Azure VMs.

**Best answer:** **Azure Arc-enabled servers + AMA + DCR + Log Analytics workspace**.

### Scenario 8 - Compliance Logging Requirements
An auditor requires 1-year searchable logs and multi-year cheap retention for critical resources.

**Best answer:** Set Log Analytics analytics retention as needed up to **730 days** where interactive search is required; send long-term copies to **Storage** for cheaper retention.

### Scenario 9 - Metrics vs Logs for Alerting
The requirement is to trigger an alert within 1 minute if CPU exceeds 85%.

**Best answer:** **Metric alert**.

**Why:** Faster and cheaper than log query alerting for simple thresholds.

### Scenario 10 - Workspace Segmentation
A regulated production environment must have different retention and access rules than dev/test.

**Best answer:** **Workspace per environment**.

**Why:** Isolation, different retention, easier chargeback and compliance control.

---

<a id="quick-reference-trigger-table"></a>
## 18. Quick Reference Trigger Table

| If the scenario says X... | Think Y |
|---------------------------|---------|
| Need near real-time threshold alerting | **Metric alert** |
| Need complex correlation across data | **Log alert + KQL** |
| Need single pane of glass | **Centralized workspace + Workbooks** |
| Teams should only see their resources | **Resource-context access** |
| Need centralized SOC | **Centralized Log Analytics + Sentinel** |
| Need app transaction tracing | **Application Insights** |
| Need topology/dependency view | **Application Map** |
| Need synthetic availability checks | **Availability tests** |
| Need anomaly-based app insights | **Smart detection** |
| Need long-term low-cost retention | **Storage destination / archive strategy** |
| Need 93-day time-series retention | **Metrics database** |
| Need 30-730 day interactive log retention | **Log Analytics workspace** |
| Need log collection governance at scale | **DCR + AMA + Policy** |
| Need automatic diagnostics on new resources | **DeployIfNotExists Policy** |
| Need SIEM/SOAR | **Microsoft Sentinel** |
| Need security posture recommendations | **Defender for Cloud** |
| Need secure score | **Defender for Cloud** |
| Need public Azure outage info | **Azure Status** |
| Need tenant-specific outage info | **Service Health** |
| Need health of one resource | **Resource Health** |
| Need hybrid server monitoring | **Azure Arc + AMA** |
| Need avoid legacy agents | **Choose AMA, not MMA/OMS** |
| Need low-cost high-volume logs | **Basic logs** |
| Need full analytics and alerting on logs | **Analytics logs** |
| Need cost optimization for stable high ingestion | **Commitment tier / dedicated cluster strategy** |
| Need export to external SIEM | **Diagnostic settings -> Event Hub** |
| Need audit retention | **Diagnostic settings -> Storage** |
| Need control-plane change visibility | **AzureActivity / Activity log alerts** |
| Need custom business KPI telemetry | **Custom metrics or custom events** |
| Need fast autoscale signal | **Metrics** |
| Need detailed troubleshooting | **Logs + KQL** |
| Need route/suppress alerts centrally | **Alert processing rules** |
| Need normalized alert payloads | **Common alert schema** |
| Need security dashboards | **Sentinel workbooks** |
| Need optimization recommendations | **Azure Advisor** |
| Need multi-cloud/hybrid Azure governance | **Azure Arc** |

---

<a id="common-exam-traps"></a>
## 19. Common Exam Traps

### 1) Log Analytics Workspace Scope Decisions
- **Trap:** Assuming one workspace per subscription is always best
- **Reality:** Choose based on **RBAC, retention, residency, operations, and chargeback**

### 2) Metrics vs Logs for Alerting
- **Trap:** Using a log alert for every threshold
- **Reality:** Use **metrics** first for fast/simple threshold alerting

### 3) Application Insights Sampling Impact
- **Trap:** Enabling aggressive sampling without considering forensic or compliance needs
- **Reality:** Sampling saves cost but may reduce visibility into rare events

### 4) Sentinel Cost Misconceptions
- **Trap:** Thinking Sentinel cost is just a flat add-on
- **Reality:** It is heavily affected by **ingestion volume, retention, and detection scope**

### 5) Diagnostic Settings Destination Limits
- **Trap:** Forgetting that not every destination pattern is appropriate for every workload
- **Reality:** Match destination to purpose: **Log Analytics = analysis**, **Storage = retention**, **Event Hub = streaming integration**

### 6) AMA vs Legacy Agent
- **Trap:** Selecting Log Analytics agent/MMA because of older study material
- **Reality:** **AMA is the current strategic choice** for new designs

### 7) Service Health vs Resource Health vs Azure Status
- **Trap:** Mixing the three services
- **Reality:** **Azure Status = public**, **Service Health = your tenant**, **Resource Health = individual resource**

### 8) Defender for Cloud vs Sentinel
- **Trap:** Treating them as interchangeable
- **Reality:** **Defender = posture/protection**, **Sentinel = SIEM/SOAR**

### 9) Workspace-Based Application Insights
- **Trap:** Defaulting to classic Application Insights
- **Reality:** Choose **workspace-based** unless legacy compatibility is explicitly required

### 10) Monitoring Only Azure Resources
- **Trap:** Ignoring hybrid monitoring requirements
- **Reality:** Use **Azure Arc** when the scenario includes on-premises or multicloud servers

---

<a id="-final-az-305-exam-tips"></a>
## 🎯 Final AZ-305 Exam Tips

1. **Start with the monitoring objective**: health detection, deep diagnostics, app tracing, posture management, or SIEM.
2. **Prefer metrics for fast thresholds** and logs for diagnosis, audit, and correlation.
3. **Design workspace scope intentionally** around RBAC, retention, chargeback, and compliance.
4. **Use workspace-based Application Insights** as the default modern APM architecture.
5. **Standardize collection with AMA + DCR + Policy** rather than manual per-resource setup.
6. **Choose Defender for Cloud for posture** and **Sentinel for SOC operations**; they are complementary, not interchangeable.
7. **Make resilience visible** by monitoring primary and failover regions, service health, and synthetic availability.
8. **Control ingestion cost early** with filtering, sampling, Basic logs, retention tuning, and commitment tiers.
9. **Reduce alert fatigue** with alert processing rules, severity discipline, and action-group design.
10. **Pick the most Azure-native governed design** that still meets business, compliance, and operational goals.

### Final Architect Review Checklist

Before picking an answer on AZ-305, ask:

1. **Do I need metrics or logs?**
2. **Who needs access to the data, and at what scope?**
3. **What retention/compliance requirement exists?**
4. **Is this about operations, application performance, security posture, or SIEM?**
5. **Can I enforce it with Policy and standardize it at scale?**
6. **What is the cheapest design that still meets operational and compliance needs?**

> 🧠 **Exam mindset:** The best answer is usually the one that is **Azure-native, governed at scale, cost-aware, and operationally sustainable**.

---

<a id="-architecture-decision-flowchart"></a>
## 📐 Architecture Decision Flowchart

```text
Start
  │
  ├─ Need a unified Azure-native monitoring platform?
  │    └─ YES -> Azure Monitor
  │
  ├─ Need near real-time numeric signal, autoscale, or threshold alerting?
  │    └─ YES -> Metrics + Metric Alerts
  │
  ├─ Need queryable history, cross-resource correlation, compliance retention, or KQL?
  │    └─ YES -> Log Analytics
  │
  ├─ Need request tracing, dependency maps, exceptions, or synthetic app tests?
  │    └─ YES -> Application Insights
  │
  ├─ Need control-plane, service issue, or maintenance awareness?
  │    └─ YES -> Activity Log Alerts + Service Health + Resource Health
  │
  ├─ Need posture management, secure score, or cloud workload protection?
  │    └─ YES -> Defender for Cloud
  │
  ├─ Need SOC correlation, incidents, hunting, or SOAR playbooks?
  │    └─ YES -> Microsoft Sentinel
  │
  └─ Need hybrid server consistency?
       └─ YES -> Azure Arc + AMA + DCR + Azure Monitor
```

---

<a id="exam-style-review-questions"></a>
## Exam-Style Review Questions

1. A company needs 1-minute alerting when CPU exceeds 85% on production VMs, with the lowest practical cost. What should you choose and why?
2. A regulated workload needs 180 days of searchable logs, 7 years of cheap retention, and strict separation between prod and dev access. How would you design the workspace and retention strategy?
3. A developer team needs distributed tracing across a web app, API, and downstream dependencies with minimal custom code. Which monitoring services should be used together?
4. A security operations center wants centralized incidents, playbooks, and hunting, while the governance team wants secure score and hardening recommendations. What combination best fits?
5. A mission-critical app runs active/passive across two Azure regions. What monitoring architecture ensures failover visibility even if the primary region is impaired?

---

<footer>

**Footer:** Pair this cheat sheet with [Monitoring Labs](./Labs/Azure-Monitoring-Labs.md) and validate every design against **operations, security, resilience, retention, and cost**.

</footer>
