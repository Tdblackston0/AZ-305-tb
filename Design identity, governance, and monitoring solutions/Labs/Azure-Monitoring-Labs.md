# Azure Monitoring Hands-On Labs (AZ-305)

> 📖 **Cheat Sheet:** [Azure Monitoring](../Azure-Monitoring.md)

> **Primary exam domain:** Design identity, governance, and monitoring solutions (25-30%)  
> **Tools used:** Azure CLI, Azure PowerShell, KQL, Azure Monitor, Log Analytics, Application Insights, Microsoft Defender for Cloud, Microsoft Sentinel  
> **Important:** Use a non-production subscription or isolated resource group. Several labs generate ingestion, alerting, and security-plan costs.

---

## Lab 1: Log Analytics Workspace Setup

### Objective
Create a Log Analytics workspace, configure retention and access mode, connect Azure resources, deploy a data collection rule, and run basic KQL queries.

**When to Use This:** Use a Log Analytics workspace when you need centralized log collection, cross-resource analysis, and a shared monitoring plane for Azure Monitor, AMA, Defender, and Sentinel.

**Key AZ-305 Concepts:** Workspace-based monitoring, retention planning, RBAC scope design, data collection rules (DCRs), and centralized operational visibility.

### Exam Domain Mapping
- **Primary:** Design identity, governance, and monitoring solutions (25-30%)
- **Secondary:** Design infrastructure solutions (30-35%)

### Prerequisites
- Contributor or Monitoring Contributor on the target subscription/resource group
- Azure CLI logged in with `az login`
- Azure PowerShell logged in with `Connect-AzAccount`
- `Microsoft.OperationalInsights`, `Microsoft.Insights`, and `Microsoft.Compute` providers registered
- A test VM or App Service to connect to the workspace

### Architecture and Design Rationale
A single workspace simplifies cross-resource troubleshooting, workbook reporting, and alerting. For the exam, know the tradeoff between **workspace-context access** (central operations teams) and **resource-context access** (application/resource owners query only what they can access). Use shorter retention for cost-sensitive environments and archive/export where long-term retention is required.

### Implementation Steps
1. Register required resource providers.
2. Create a resource group and Log Analytics workspace.
3. Set retention and daily quota if required.
4. Toggle workspace-context vs. resource-context access.
5. Create a DCR for Windows events or performance counters.
6. Associate the DCR to a VM.
7. Validate ingestion with KQL.

### Full CLI + PowerShell + KQL Commands

#### Azure CLI
```bash
RG="rg-monitor-lab01"
LOCATION="eastus"
LAW="lawaz305lab01$RANDOM"
DCR="dcr-lab01"
VM_NAME="<existing-vm-name>"
SUB_ID=$(az account show --query id -o tsv)
VM_ID=$(az vm show -g $RG -n $VM_NAME --query id -o tsv)

az provider register --namespace Microsoft.OperationalInsights
az provider register --namespace Microsoft.Insights
az provider register --namespace Microsoft.Compute

az group create --name $RG --location $LOCATION

az monitor log-analytics workspace create \
  --resource-group $RG \
  --workspace-name $LAW \
  --location $LOCATION \
  --retention-time 30

# Optional: switch to resource-context queries only
az resource update \
  --ids "/subscriptions/$SUB_ID/resourceGroups/$RG/providers/Microsoft.OperationalInsights/workspaces/$LAW" \
  --set properties.features.enableLogAccessUsingOnlyResourcePermissions=true

WORKSPACE_ID=$(az monitor log-analytics workspace show -g $RG -n $LAW --query customerId -o tsv)
WORKSPACE_ARM_ID=$(az monitor log-analytics workspace show -g $RG -n $LAW --query id -o tsv)

az monitor data-collection rule create \
  --resource-group $RG \
  --location $LOCATION \
  --name $DCR \
  --data-flows streams=Microsoft-Perf destinations=laDest \
  --destinations log-analytics name=laDest workspace-resource-id=$WORKSPACE_ARM_ID \
  --performance-counters name=perfCounter streams=Microsoft-Perf sampling-frequency=60 counter-specifiers="\\Processor(_Total)\\% Processor Time"

az monitor data-collection rule association create \
  --name "assoc-$VM_NAME" \
  --rule-id $(az monitor data-collection rule show -g $RG -n $DCR --query id -o tsv) \
  --resource $VM_ID

az monitor log-analytics query \
  --workspace $WORKSPACE_ID \
  --analytics-query "Heartbeat | summarize LastSeen=max(TimeGenerated) by Computer" \
  --timespan P1D
```

#### PowerShell
```powershell
$RG = "rg-monitor-lab01"
$Location = "eastus"
$Law = "lawaz305lab01$(Get-Random)"
$Dcr = "dcr-lab01"
$VmName = "<existing-vm-name>"
$SubId = (Get-AzContext).Subscription.Id

Register-AzResourceProvider -ProviderNamespace Microsoft.OperationalInsights
Register-AzResourceProvider -ProviderNamespace Microsoft.Insights
Register-AzResourceProvider -ProviderNamespace Microsoft.Compute
New-AzResourceGroup -Name $RG -Location $Location

$workspace = New-AzOperationalInsightsWorkspace `
  -ResourceGroupName $RG `
  -Name $Law `
  -Location $Location `
  -Sku PerGB2018 `
  -RetentionInDays 30

# Switch to resource-context access using REST when needed
Invoke-AzRestMethod -Method PATCH -Path "/subscriptions/$SubId/resourceGroups/$RG/providers/Microsoft.OperationalInsights/workspaces/$Law?api-version=2022-10-01" -Payload '{"properties":{"features":{"enableLogAccessUsingOnlyResourcePermissions":true}}}'

$vm = Get-AzVM -ResourceGroupName $RG -Name $VmName
$dcrBody = @{
  location   = $Location
  properties = @{
    dataSources = @{
      performanceCounters = @(
        @{
          name                          = 'perfCounter'
          streams                       = @('Microsoft-Perf')
          samplingFrequencyInSeconds    = 60
          counterSpecifiers             = @('\\Processor(_Total)\\% Processor Time')
        }
      )
    }
    destinations = @{
      logAnalytics = @(
        @{
          name                = 'laDest'
          workspaceResourceId = $workspace.ResourceId
        }
      )
    }
    dataFlows = @(
      @{
        streams      = @('Microsoft-Perf')
        destinations = @('laDest')
      }
    )
  }
} | ConvertTo-Json -Depth 10

Invoke-AzRestMethod -Method PUT -Path "/subscriptions/$SubId/resourceGroups/$RG/providers/Microsoft.Insights/dataCollectionRules/$Dcr?api-version=2023-03-11" -Payload $dcrBody
```

#### KQL
```kusto
Heartbeat
| where TimeGenerated > ago(1d)
| summarize LastSeen=max(TimeGenerated) by Computer, OSType
| order by LastSeen desc

Perf
| where TimeGenerated > ago(1h)
| where ObjectName == "Processor" and CounterName == "% Processor Time"
| summarize AvgCPU=avg(CounterValue) by bin(TimeGenerated, 5m), Computer
| render timechart
```

### IaC Implementation

#### Bicep
```bicep
param workspaceName string
param location string = resourceGroup().location
param retentionInDays int = 30

resource law 'Microsoft.OperationalInsights/workspaces@2022-10-01' = {
  name: workspaceName
  location: location
  properties: {
    retentionInDays: retentionInDays
    sku: {
      name: 'PerGB2018'
    }
    features: {
      enableLogAccessUsingOnlyResourcePermissions: true
    }
  }
}
```

#### Terraform
```hcl
resource "azurerm_log_analytics_workspace" "law" {
  name                = var.workspace_name
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  sku                 = "PerGB2018"
  retention_in_days   = 30
  internet_ingestion_enabled = true
  internet_query_enabled     = true
}
```

### Validation and Success Criteria
- Workspace deploys successfully with the intended retention.
- Access mode matches the intended operating model.
- DCR is associated to the VM.
- `Heartbeat` and `Perf` data appear in the workspace.

### Verification Steps
- Run `az monitor log-analytics workspace show -g $RG -n $LAW -o table`
- Run the KQL queries above and confirm recent data.
- Confirm the DCR association in Azure portal or with `az monitor data-collection rule association list --resource $VM_ID`

### Cleanup

#### Azure CLI
```bash
az monitor data-collection rule association delete --name "assoc-$VM_NAME" --resource $VM_ID
az group delete --name $RG --yes --no-wait
```

#### PowerShell
```powershell
Remove-AzResourceGroup -Name $RG -Force
```

### Exam Tip
> 💡 Prefer **resource-context access** when app teams should query only the resources they already have access to. Prefer **workspace-context access** for centralized SOC/NOC operations.

### Exam-style Review Questions
1. When would you choose one shared workspace instead of per-application workspaces?
2. Why might resource-context access be preferable for decentralized operations teams?
3. What is the role of a DCR in the AMA architecture?

---

## Lab 2: Diagnostic Settings Configuration

### Objective
Enable diagnostic settings on a VM, storage account, and Azure SQL resource; route telemetry to Log Analytics and Storage; then evaluate policy-based enforcement.

**When to Use This:** Use diagnostic settings when you must capture resource logs and platform metrics for troubleshooting, auditing, alerting, or long-term retention.

**Key AZ-305 Concepts:** Multi-destination telemetry, resource logs vs. platform metrics, archival to Storage, and Azure Policy `DeployIfNotExists` for scale.

### Exam Domain Mapping
- **Primary:** Design identity, governance, and monitoring solutions (25-30%)
- **Secondary:** Design data storage solutions (20-25%)

### Prerequisites
- Existing VM, storage account, and Azure SQL logical server or database
- Contributor or Monitoring Contributor permissions
- Log Analytics workspace and storage account for archival
- `Microsoft.Insights` provider registered

### Architecture and Design Rationale
Diagnostic settings are the control point for routing resource telemetry. For AZ-305, know when to send data to **Log Analytics** (querying and alerting), **Storage** (low-cost archival), and **Event Hubs** (downstream SIEM/streaming). Policy automation matters for enterprise consistency.

### Implementation Steps
1. Create or identify the Log Analytics workspace and archive storage account.
2. Enable diagnostic settings on VM, storage account, and SQL.
3. Route logs/metrics to the workspace and storage.
4. Assign a `DeployIfNotExists` policy for automatic deployment.
5. Validate ingestion by querying the target tables.

### Full CLI + PowerShell + KQL Commands

#### Azure CLI
```bash
RG="rg-monitor-lab02"
LAW_RG="rg-monitor-lab01"
LAW_NAME="<existing-law-name>"
ARCHIVE_SA="stmonarch$RANDOM"
VM_ID="<vm-resource-id>"
STORAGE_ID="<storage-resource-id>"
SQL_ID="<sql-server-or-db-resource-id>"

az group create -n $RG -l eastus
az storage account create -n $ARCHIVE_SA -g $RG -l eastus --sku Standard_LRS --kind StorageV2
LAW_ID=$(az monitor log-analytics workspace show -g $LAW_RG -n $LAW_NAME --query id -o tsv)
ARCHIVE_ID=$(az storage account show -g $RG -n $ARCHIVE_SA --query id -o tsv)

az monitor diagnostic-settings create \
  --name diag-vm \
  --resource $VM_ID \
  --workspace $LAW_ID \
  --storage-account $ARCHIVE_ID \
  --logs '[{"categoryGroup":"allLogs","enabled":true}]' \
  --metrics '[{"category":"AllMetrics","enabled":true}]'

az monitor diagnostic-settings create \
  --name diag-storage \
  --resource $STORAGE_ID \
  --workspace $LAW_ID \
  --storage-account $ARCHIVE_ID \
  --logs '[{"categoryGroup":"allLogs","enabled":true}]' \
  --metrics '[{"category":"Transaction","enabled":true}]'

az monitor diagnostic-settings create \
  --name diag-sql \
  --resource $SQL_ID \
  --workspace $LAW_ID \
  --storage-account $ARCHIVE_ID \
  --logs '[{"categoryGroup":"allLogs","enabled":true}]' \
  --metrics '[{"category":"AllMetrics","enabled":true}]'

# Discover a built-in DeployIfNotExists policy for VM diagnostics
az policy definition list \
  --query "[?contains(displayName,'Deploy') && contains(displayName,'Diagnostic')].{Name:displayName,Id:id}" -o table
```

#### PowerShell
```powershell
$ArchiveSa = "stmonarch$(Get-Random)"
$Law = Get-AzOperationalInsightsWorkspace -ResourceGroupName "rg-monitor-lab01" -Name "<existing-law-name>"
$Archive = New-AzStorageAccount -ResourceGroupName "rg-monitor-lab02" -Name $ArchiveSa -Location "eastus" -SkuName Standard_LRS -Kind StorageV2

Set-AzDiagnosticSetting -Name "diag-vm" -ResourceId "<vm-resource-id>" -WorkspaceId $Law.ResourceId -StorageAccountId $Archive.Id -Enabled $true
Set-AzDiagnosticSetting -Name "diag-storage" -ResourceId "<storage-resource-id>" -WorkspaceId $Law.ResourceId -StorageAccountId $Archive.Id -Enabled $true
Set-AzDiagnosticSetting -Name "diag-sql" -ResourceId "<sql-resource-id>" -WorkspaceId $Law.ResourceId -StorageAccountId $Archive.Id -Enabled $true

Get-AzPolicyDefinition | Where-Object DisplayName -match 'Diagnostic' | Select-Object DisplayName, PolicyType
```

#### KQL
```kusto
AzureDiagnostics
| where TimeGenerated > ago(1h)
| summarize Records=count() by ResourceType, Category
| order by Records desc

StorageBlobLogs
| where TimeGenerated > ago(1h)
| summarize Requests=count() by AuthenticationType, StatusText
```

### IaC Implementation

#### Bicep
```bicep
param diagName string = 'diag-vm'
param vmName string
param workspaceId string
param storageAccountId string

resource vm 'Microsoft.Compute/virtualMachines@2023-09-01' existing = {
  name: vmName
}

resource diag 'Microsoft.Insights/diagnosticSettings@2021-05-01-preview' = {
  name: diagName
  scope: vm
  properties: {
    workspaceId: workspaceId
    storageAccountId: storageAccountId
    logs: [
      {
        categoryGroup: 'allLogs'
        enabled: true
      }
    ]
    metrics: [
      {
        category: 'AllMetrics'
        enabled: true
      }
    ]
  }
}
```

> **Note:** Repeat the same diagnostic-setting pattern for storage and SQL resources by changing the `existing` resource type and supported categories.

#### Terraform
```hcl
resource "azurerm_monitor_diagnostic_setting" "diag" {
  name                       = "diag-default"
  target_resource_id         = var.target_resource_id
  log_analytics_workspace_id = var.workspace_id
  storage_account_id         = var.storage_account_id

  enabled_log {
    category_group = "allLogs"
  }

  metric {
    category = "AllMetrics"
    enabled  = true
  }
}
```

### Validation and Success Criteria
- Diagnostic settings exist for each target resource.
- Logs land in Log Analytics and archive to storage.
- A built-in or custom policy is identified for enforcement at scale.

### Verification Steps
- Run `az monitor diagnostic-settings list --resource $VM_ID -o table`
- Confirm blobs appear in the archive storage account.
- Use KQL to confirm logs from each resource type.

### Cleanup

#### Azure CLI
```bash
az monitor diagnostic-settings delete --name diag-vm --resource $VM_ID
az monitor diagnostic-settings delete --name diag-storage --resource $STORAGE_ID
az monitor diagnostic-settings delete --name diag-sql --resource $SQL_ID
az group delete -n $RG --yes --no-wait
```

#### PowerShell
```powershell
Remove-AzDiagnosticSetting -Name "diag-vm" -ResourceId "<vm-resource-id>"
Remove-AzDiagnosticSetting -Name "diag-storage" -ResourceId "<storage-resource-id>"
Remove-AzDiagnosticSetting -Name "diag-sql" -ResourceId "<sql-resource-id>"
Remove-AzResourceGroup -Name "rg-monitor-lab02" -Force
```

### Exam Tip
> 💡 Diagnostic settings are configured **per resource**. Azure Policy is the exam-friendly answer when you need the same telemetry baseline across many subscriptions or landing zones.

### Exam-style Review Questions
1. Why would you send the same telemetry to both Log Analytics and Storage?
2. When is `DeployIfNotExists` a better fit than manual deployment?
3. What is the difference between resource logs and platform metrics?

---

## Lab 3: Metric Alerts

### Objective
Create metric alerts for VM CPU and storage availability, configure an action group with email and SMS, test alerting, review history, and suppress noise during maintenance.

**When to Use This:** Use metric alerts when you need low-latency, near real-time alerting on platform metrics without running KQL queries.

**Key AZ-305 Concepts:** Stateful alerting, action groups, alert processing rules, maintenance suppression, and metric dimensions.

### Exam Domain Mapping
- **Primary:** Design identity, governance, and monitoring solutions (25-30%)
- **Secondary:** Design infrastructure solutions (30-35%)

### Prerequisites
- Existing VM and storage account
- Monitoring Contributor on the subscription or resource group
- Valid email and phone number for notifications

### Architecture and Design Rationale
Metric alerts are cheaper and faster than log alerts for platform metrics. Use them for CPU, availability, latency, and capacity thresholds. Use action groups for reusable notification routing and alert processing rules to prevent alert fatigue during planned maintenance.

### Implementation Steps
1. Create or reuse an action group.
2. Create a CPU alert on the VM.
3. Create an availability alert on the storage account.
4. Create an alert processing rule to suppress alerts during a maintenance window.
5. Generate load to test alert triggering.

### Full CLI + PowerShell + KQL Commands

#### Azure CLI
```bash
RG="rg-monitor-lab03"
LOCATION="eastus"
ACTION_GROUP="ag-ops"
VM_ID="<vm-resource-id>"
STORAGE_ID="<storage-resource-id>"
EMAIL="ops@contoso.com"
PHONE="4255550100"

az group create -n $RG -l $LOCATION

az monitor action-group create \
  --name $ACTION_GROUP \
  --resource-group $RG \
  --short-name AGOPS \
  --action email OpsEmail $EMAIL \
  --action sms OpsSms 1 $PHONE

ACTION_GROUP_ID=$(az monitor action-group show -g $RG -n $ACTION_GROUP --query id -o tsv)

az monitor metrics alert create \
  --name vm-high-cpu \
  --resource-group $RG \
  --scopes $VM_ID \
  --condition "avg Percentage CPU > 80" \
  --window-size 5m \
  --evaluation-frequency 1m \
  --action $ACTION_GROUP_ID \
  --description "Alert when VM CPU exceeds 80 percent"

az monitor metrics alert create \
  --name storage-availability \
  --resource-group $RG \
  --scopes $STORAGE_ID \
  --condition "avg Availability < 99" \
  --window-size 5m \
  --evaluation-frequency 1m \
  --action $ACTION_GROUP_ID

# Optional test: create CPU load on a Linux VM
az vm run-command invoke \
  --ids $VM_ID \
  --command-id RunShellScript \
  --scripts "sudo apt-get update && sudo apt-get -y install stress && stress --cpu 2 --timeout 300"
```

#### PowerShell
```powershell
$rg = "rg-monitor-lab03"
$agName = "ag-ops"
$vmId = "<vm-resource-id>"
$storageId = "<storage-resource-id>"
$emailReceiver = New-AzActionGroupReceiver -Name 'OpsEmail' -EmailReceiver -EmailAddress 'ops@contoso.com'
$smsReceiver = New-AzActionGroupReceiver -Name 'OpsSms' -SmsReceiver -CountryCode '1' -PhoneNumber '4255550100'
Set-AzActionGroup -ResourceGroupName $rg -Name $agName -ShortName 'AGOPS' -Receiver $emailReceiver,$smsReceiver
$ag = Get-AzActionGroup -ResourceGroupName $rg -Name $agName

Add-AzMetricAlertRuleV2 -Name 'vm-high-cpu' -ResourceGroupName $rg -WindowSize 00:05:00 -Frequency 00:01:00 -TargetResourceId $vmId -Condition (New-AzMetricAlertRuleV2Criteria -MetricName 'Percentage CPU' -TimeAggregation Average -Operator GreaterThan -Threshold 80) -ActionGroupId $ag.Id -Severity 2
Add-AzMetricAlertRuleV2 -Name 'storage-availability' -ResourceGroupName $rg -WindowSize 00:05:00 -Frequency 00:01:00 -TargetResourceId $storageId -Condition (New-AzMetricAlertRuleV2Criteria -MetricName 'Availability' -TimeAggregation Average -Operator LessThan -Threshold 99) -ActionGroupId $ag.Id -Severity 2
```

#### KQL
```kusto
AlertsManagementResources
| where TimeGenerated > ago(1d)
| summarize Alerts=count() by MonitorCondition, AlertState, Severity

AzureMetrics
| where TimeGenerated > ago(1h)
| where MetricName in ("Percentage CPU", "Availability")
| summarize AvgValue=avg(Average) by MetricName, ResourceId, bin(TimeGenerated, 5m)
| render timechart
```

### IaC Implementation

#### Bicep
```bicep
param actionGroupId string
param targetResourceId string

resource cpuAlert 'Microsoft.Insights/metricAlerts@2018-03-01' = {
  name: 'vm-high-cpu'
  location: 'global'
  properties: {
    severity: 2
    enabled: true
    scopes: [ targetResourceId ]
    evaluationFrequency: 'PT1M'
    windowSize: 'PT5M'
    criteria: {
      'odata.type': 'Microsoft.Azure.Monitor.SingleResourceMultipleMetricCriteria'
      allOf: [
        {
          name: 'HighCPU'
          metricName: 'Percentage CPU'
          timeAggregation: 'Average'
          operator: 'GreaterThan'
          threshold: 80
          criterionType: 'StaticThresholdCriterion'
        }
      ]
    }
    actions: [
      {
        actionGroupId: actionGroupId
      }
    ]
  }
}
```

#### Terraform
```hcl
resource "azurerm_monitor_metric_alert" "cpu" {
  name                = "vm-high-cpu"
  resource_group_name = azurerm_resource_group.rg.name
  scopes              = [var.vm_id]
  severity            = 2
  window_size         = "PT5M"
  frequency           = "PT1M"

  criteria {
    metric_namespace = "Microsoft.Compute/virtualMachines"
    metric_name      = "Percentage CPU"
    aggregation      = "Average"
    operator         = "GreaterThan"
    threshold        = 80
  }

  action {
    action_group_id = var.action_group_id
  }
}
```

### Validation and Success Criteria
- Action group exists with the intended notification receivers.
- Both metric alerts evaluate successfully.
- Test load triggers the CPU alert.
- Alert history is visible and maintenance suppression can be applied.

### Verification Steps
- Run `az monitor metrics alert list -g $RG -o table`
- Open **Monitor > Alerts** and confirm fired/resolved history.
- Validate the action group test notification in portal.

### Cleanup

#### Azure CLI
```bash
az monitor metrics alert delete -g $RG -n vm-high-cpu
az monitor metrics alert delete -g $RG -n storage-availability
az monitor action-group delete -g $RG -n $ACTION_GROUP
az group delete -n $RG --yes --no-wait
```

#### PowerShell
```powershell
Remove-AzMetricAlertRuleV2 -ResourceGroupName 'rg-monitor-lab03' -Name 'vm-high-cpu'
Remove-AzMetricAlertRuleV2 -ResourceGroupName 'rg-monitor-lab03' -Name 'storage-availability'
Remove-AzActionGroup -ResourceGroupName 'rg-monitor-lab03' -Name 'ag-ops'
Remove-AzResourceGroup -Name 'rg-monitor-lab03' -Force
```

### Exam Tip
> 💡 If the requirement is **fast and inexpensive alerting on metrics**, choose a metric alert. If the requirement is **complex correlation across logs**, choose a log alert.

### Exam-style Review Questions
1. Why is a metric alert usually preferred over a log alert for CPU threshold monitoring?
2. What problem do alert processing rules solve?
3. Why should action groups be reused instead of hardcoding notifications in each alert?

---

## Lab 4: Log Alerts (KQL-based)

### Objective
Create scheduled query rules to alert on failed sign-ins and application exceptions, tune frequency/lookback, test results, and optimize for cost.

**When to Use This:** Use log alerts when you need query-based detection, cross-table correlation, or filtering that metrics alone cannot provide.

**Key AZ-305 Concepts:** Scheduled query rules, evaluation frequency, lookback windows, KQL optimization, and cost-aware detection design.

### Exam Domain Mapping
- **Primary:** Design identity, governance, and monitoring solutions (25-30%)

### Prerequisites
- Log Analytics workspace
- Entra ID logs exported to the workspace for `SigninLogs` and `AuditLogs`
- Application Insights or workspace-based app logs sending `AppExceptions`
- Action group from Lab 3 or equivalent

### Architecture and Design Rationale
Log alerts trade lower cost efficiency and higher latency for richer detection logic. For failed sign-ins, use `SigninLogs`; keep `AuditLogs` enabled for administrative and identity change events. Minimize cost by filtering early, summarizing, and limiting lookback windows.

### Implementation Steps
1. Ensure Entra ID and app telemetry reach the workspace.
2. Build and test KQL queries interactively.
3. Create one alert rule for failed sign-ins and one for application exceptions.
4. Tune frequency and lookback based on business tolerance.
5. Review alert behavior and reduce noisy query scans.

### Full CLI + PowerShell + KQL Commands

#### Azure CLI
```bash
RG="rg-monitor-lab04"
LAW_RG="rg-monitor-lab01"
LAW_NAME="<existing-law-name>"
AG_ID="<action-group-id>"
WORKSPACE_ID=$(az monitor log-analytics workspace show -g $LAW_RG -n $LAW_NAME --query id -o tsv)

FAILED_SIGNIN_QUERY="SigninLogs | where ResultType != 0 | summarize Failed=count() by bin(TimeGenerated, 5m), AppDisplayName | where Failed >= 5"
APP_ERROR_QUERY="AppExceptions | where SeverityLevel >= 3 | summarize Errors=count() by bin(TimeGenerated, 5m), AppRoleName | where Errors >= 3"

az monitor scheduled-query create \
  --resource-group $RG \
  --name alert-failed-signins \
  --scopes $WORKSPACE_ID \
  --condition-query "$FAILED_SIGNIN_QUERY" \
  --condition "count 'FailedSignins' > 0" \
  --evaluation-frequency 5m \
  --window-size 15m \
  --action-groups $AG_ID \
  --description "Alert on repeated failed sign-ins"

az monitor scheduled-query create \
  --resource-group $RG \
  --name alert-app-errors \
  --scopes $WORKSPACE_ID \
  --condition-query "$APP_ERROR_QUERY" \
  --condition "count 'AppErrors' > 0" \
  --evaluation-frequency 5m \
  --window-size 15m \
  --action-groups $AG_ID
```

#### PowerShell
```powershell
$rg = 'rg-monitor-lab04'
$workspace = Get-AzOperationalInsightsWorkspace -ResourceGroupName 'rg-monitor-lab01' -Name '<existing-law-name>'
$actionGroupId = '<action-group-id>'
$failedSigninQuery = "SigninLogs | where ResultType != 0 | summarize Failed=count() by bin(TimeGenerated, 5m), AppDisplayName | where Failed >= 5"
$appErrorQuery = "AppExceptions | where SeverityLevel >= 3 | summarize Errors=count() by bin(TimeGenerated, 5m), AppRoleName | where Errors >= 3"

New-AzScheduledQueryRule -ResourceGroupName $rg -Name 'alert-failed-signins' -Location 'eastus' -ActionGroupResourceId $actionGroupId -Scope $workspace.ResourceId -Enabled $true -Severity 2 -EvaluationFrequency (New-TimeSpan -Minutes 5) -WindowSize (New-TimeSpan -Minutes 15) -CriterionAllOf @(New-AzScheduledQueryRuleConditionObject -Query $failedSigninQuery -TimeAggregation Count -Operator GreaterThan -Threshold 0)
New-AzScheduledQueryRule -ResourceGroupName $rg -Name 'alert-app-errors' -Location 'eastus' -ActionGroupResourceId $actionGroupId -Scope $workspace.ResourceId -Enabled $true -Severity 2 -EvaluationFrequency (New-TimeSpan -Minutes 5) -WindowSize (New-TimeSpan -Minutes 15) -CriterionAllOf @(New-AzScheduledQueryRuleConditionObject -Query $appErrorQuery -TimeAggregation Count -Operator GreaterThan -Threshold 0)
```

#### KQL
```kusto
// Failed sign-ins
SigninLogs
| where TimeGenerated > ago(1d)
| where ResultType != 0
| summarize Failed=count() by UserPrincipalName, AppDisplayName, bin(TimeGenerated, 15m)
| order by Failed desc

// Identity administration events
AuditLogs
| where TimeGenerated > ago(1d)
| summarize Changes=count() by OperationName, Result, bin(TimeGenerated, 1h)

// Application exceptions
AppExceptions
| where TimeGenerated > ago(1h)
| summarize Exceptions=count(), Samples=make_set(OuterMessage, 3) by AppRoleName, bin(TimeGenerated, 5m)
| render timechart
```

### IaC Implementation

#### Bicep
```bicep
param workspaceId string
param actionGroupId string
param query string

resource logAlert 'Microsoft.Insights/scheduledQueryRules@2023-12-01' = {
  name: 'alert-failed-signins'
  location: resourceGroup().location
  properties: {
    enabled: true
    scopes: [ workspaceId ]
    evaluationFrequency: 'PT5M'
    windowSize: 'PT15M'
    severity: 2
    criteria: {
      allOf: [
        {
          query: query
          timeAggregation: 'Count'
          operator: 'GreaterThan'
          threshold: 0
        }
      ]
    }
    actions: {
      actionGroups: [ actionGroupId ]
    }
  }
}
```

#### Terraform
```hcl
resource "azurerm_monitor_scheduled_query_rules_alert_v2" "failed_signins" {
  name                = "alert-failed-signins"
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  scopes              = [var.workspace_id]
  evaluation_frequency = "PT5M"
  window_duration      = "PT15M"
  severity             = 2
  query                = var.failed_signin_query

  criteria {
    operator                = "GreaterThan"
    threshold               = 0
    time_aggregation_method = "Count"
  }

  action {
    action_groups = [var.action_group_id]
  }
}
```

### Validation and Success Criteria
- KQL returns expected rows.
- Scheduled query rules deploy successfully.
- Alerts fire when thresholds are exceeded.
- Query cost remains controlled through scoped filtering and short windows.

### Verification Steps
- Test queries in Logs before alert creation.
- Run `az monitor scheduled-query list -g $RG -o table`
- Review alert history and query run statistics.

### Cleanup

#### Azure CLI
```bash
az monitor scheduled-query delete -g $RG -n alert-failed-signins -y
az monitor scheduled-query delete -g $RG -n alert-app-errors -y
```

#### PowerShell
```powershell
Remove-AzScheduledQueryRule -ResourceGroupName 'rg-monitor-lab04' -Name 'alert-failed-signins'
Remove-AzScheduledQueryRule -ResourceGroupName 'rg-monitor-lab04' -Name 'alert-app-errors'
```

### Exam Tip
> 💡 For sign-in failures, prefer `SigninLogs`; use `AuditLogs` for administrative or directory change events. On the exam, table selection matters as much as alert-rule configuration.

### Exam-style Review Questions
1. Why might a scheduled query rule be more expensive than a metric alert?
2. How do evaluation frequency and lookback window affect alert sensitivity?
3. What KQL tuning steps reduce log-alert cost?

---

## Lab 5: Application Insights Setup

### Objective
Create a workspace-based Application Insights resource, instrument a web app, review application map and failures, configure sampling, and emit custom telemetry.

**When to Use This:** Use Application Insights when you need application performance monitoring (APM), distributed tracing, dependency visibility, exception analysis, and synthetic availability monitoring.

**Key AZ-305 Concepts:** Workspace-based Application Insights, connection strings, sampling, end-to-end transaction monitoring, and synthetic testing.

### Exam Domain Mapping
- **Primary:** Design identity, governance, and monitoring solutions (25-30%)
- **Secondary:** Design infrastructure solutions (30-35%)

### Prerequisites
- Existing Log Analytics workspace
- Existing App Service or test web app
- Contributor permissions on the application resources

### Architecture and Design Rationale
Choose **workspace-based** Application Insights to unify app telemetry with platform and security telemetry. Use sampling for cost control, availability tests for user-experience monitoring, and custom telemetry for domain-specific business events.

### Implementation Steps
1. Create a workspace-based Application Insights resource.
2. Connect the web app with the Application Insights connection string.
3. Enable auto-instrumentation for supported runtimes.
4. Review requests, dependencies, and exceptions.
5. Create an availability test.
6. Add or validate sampling and custom events.

### Full CLI + PowerShell + KQL Commands

#### Azure CLI
```bash
RG="rg-monitor-lab05"
LOCATION="eastus"
LAW_NAME="<existing-law-name>"
APPINSIGHTS="appi-az305-$RANDOM"
WEBAPP="<existing-webapp-name>"
LAW_ID=$(az monitor log-analytics workspace show -g rg-monitor-lab01 -n $LAW_NAME --query id -o tsv)

az group create -n $RG -l $LOCATION

az monitor app-insights component create \
  --app $APPINSIGHTS \
  --location $LOCATION \
  --resource-group $RG \
  --workspace $LAW_ID \
  --application-type web

CONN_STR=$(az monitor app-insights component show -a $APPINSIGHTS -g $RG --query connectionString -o tsv)

az webapp config appsettings set \
  --name $WEBAPP \
  --resource-group <webapp-rg> \
  --settings APPLICATIONINSIGHTS_CONNECTION_STRING="$CONN_STR" ApplicationInsightsAgent_EXTENSION_VERSION="~3" XDT_MicrosoftApplicationInsights_Mode="recommended"
```

#### PowerShell
```powershell
$rg = 'rg-monitor-lab05'
$location = 'eastus'
$law = Get-AzOperationalInsightsWorkspace -ResourceGroupName 'rg-monitor-lab01' -Name '<existing-law-name>'
$appiName = "appi-az305-$(Get-Random)"

$appi = New-AzApplicationInsights -ResourceGroupName $rg -Name $appiName -Location $location -Kind web -WorkspaceResourceId $law.ResourceId
$connectionString = $appi.ConnectionString

$webApp = Get-AzWebApp -ResourceGroupName '<webapp-rg>' -Name '<existing-webapp-name>'
$settings = @{
  'APPLICATIONINSIGHTS_CONNECTION_STRING' = $connectionString
  'ApplicationInsightsAgent_EXTENSION_VERSION' = '~3'
  'XDT_MicrosoftApplicationInsights_Mode' = 'recommended'
}
Set-AzWebApp -ResourceGroupName '<webapp-rg>' -Name '<existing-webapp-name>' -AppSettings $settings
```

#### KQL
```kusto
requests
| where timestamp > ago(1h)
| summarize Requests=count(), AvgDurationMs=avg(duration) by name, resultCode
| order by Requests desc

dependencies
| where timestamp > ago(1h)
| summarize Calls=count(), AvgDurationMs=avg(duration) by target, type, success

exceptions
| where timestamp > ago(1h)
| summarize Exceptions=count() by type, outerMessage
```

### IaC Implementation

#### Bicep
```bicep
param appInsightsName string
param workspaceId string
param location string = resourceGroup().location

resource appi 'Microsoft.Insights/components@2020-02-02' = {
  name: appInsightsName
  location: location
  kind: 'web'
  properties: {
    Application_Type: 'web'
    WorkspaceResourceId: workspaceId
  }
}
```

#### Terraform
```hcl
resource "azurerm_application_insights" "appi" {
  name                = var.appinsights_name
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  application_type    = "web"
  workspace_id        = var.workspace_id
  sampling_percentage = 25
}
```

### Validation and Success Criteria
- Application Insights is workspace-based.
- The web app has the correct connection string.
- Requests, dependencies, and exceptions appear in KQL.
- Availability monitoring is configured.

### Verification Steps
- Browse the app and generate a few requests.
- Open **Application Map** and **Failures** in the portal.
- Verify data with the KQL queries above.

### Cleanup

#### Azure CLI
```bash
az monitor app-insights component delete -a $APPINSIGHTS -g $RG
az group delete -n $RG --yes --no-wait
```

#### PowerShell
```powershell
Remove-AzApplicationInsights -ResourceGroupName 'rg-monitor-lab05' -Name '<appinsights-name>'
Remove-AzResourceGroup -Name 'rg-monitor-lab05' -Force
```

### Exam Tip
> 💡 Workspace-based Application Insights is the strategic direction. On AZ-305, it is usually the better answer when teams want one analytics and alerting plane for app and platform telemetry.

### Exam-style Review Questions
1. Why choose Application Insights instead of only Log Analytics platform monitoring?
2. How does sampling help control cost?
3. When is synthetic availability monitoring more useful than infrastructure metrics?

---

## Lab 6: Workbook Creation

### Objective
Create a custom workbook that combines Log Analytics queries, metrics visualizations, parameters, and interactive drilldowns for operations reporting.

**When to Use This:** Use workbooks when you need interactive dashboards, scoped troubleshooting views, and reusable reporting for operations, engineering, or leadership.

**Key AZ-305 Concepts:** Parameterized visualizations, cross-resource analysis, workbook sharing, and interactive reporting.

### Exam Domain Mapping
- **Primary:** Design identity, governance, and monitoring solutions (25-30%)

### Prerequisites
- Log Analytics workspace with data
- Reader on data sources and Workbook Contributor or Contributor to save workbooks
- Sample resources that emit metrics and logs

### Architecture and Design Rationale
Workbooks are richer than Azure dashboards for investigative reporting because they support parameters, KQL, rich text, links, and dynamic visuals. Use them for troubleshooting and curated operations experiences; use dashboards for high-level wallboard summaries.

### Implementation Steps
1. Create workbook parameters for subscription, resource group, and time range.
2. Add log visuals backed by KQL.
3. Add metric charts.
4. Add drill-through links to related resources.
5. Share the workbook with the operations team.

### Full CLI + PowerShell + KQL Commands

#### Azure CLI
```bash
RG="rg-monitor-lab06"
LOCATION="eastus"
WORKBOOK_NAME="wb-monitor-ops"
WORKBOOK_ID=$(uuidgen)
LAW_ID="<workspace-resource-id>"

cat > workbook.json <<'EOF'
{
  "location": "eastus",
  "kind": "shared",
  "properties": {
    "displayName": "AZ-305 Monitoring Workbook",
    "serializedData": "{\"version\":\"Notebook/1.0\",\"items\":[{\"type\":1,\"content\":{\"json\":\"# Monitoring workbook\"}},{\"type\":3,\"content\":{\"version\":\"KqlItem/1.0\",\"query\":\"Heartbeat | summarize Count=count() by Computer\",\"resourceType\":\"microsoft.operationalinsights/workspaces\",\"resourceIds\":[\"<workspace-resource-id>\"]}}],\"fallbackResourceIds\":[\"<workspace-resource-id>\"]}"
  }
}
EOF

az resource create \
  --resource-group $RG \
  --resource-type Microsoft.Insights/workbooks \
  --name $WORKBOOK_ID \
  --is-full-object \
  --properties @workbook.json
```

#### PowerShell
```powershell
$rg = 'rg-monitor-lab06'
$location = 'eastus'
$workbookId = (New-Guid).Guid
$body = @{
  location   = $location
  kind       = 'shared'
  properties = @{
    displayName    = 'AZ-305 Monitoring Workbook'
    serializedData = '{"version":"Notebook/1.0","items":[{"type":1,"content":{"json":"# Monitoring workbook"}}]}'
  }
} | ConvertTo-Json -Depth 10

New-AzResource -ResourceGroupName $rg -ResourceType 'Microsoft.Insights/workbooks' -Name $workbookId -ApiVersion '2022-04-01' -PropertyObject ($body | ConvertFrom-Json) -Location $location -Force
```

#### KQL
```kusto
Heartbeat
| where TimeGenerated > ago(1h)
| summarize LastSeen=max(TimeGenerated) by Computer

Perf
| where TimeGenerated > ago(1h)
| summarize AvgCPU=avg(CounterValue) by Computer, bin(TimeGenerated, 5m)
| render timechart
```

### IaC Implementation

#### Bicep
```bicep
param workbookId string = newGuid()
param location string = resourceGroup().location
param workspaceId string

resource workbook 'Microsoft.Insights/workbooks@2022-04-01' = {
  name: workbookId
  location: location
  kind: 'shared'
  properties: {
    displayName: 'AZ-305 Monitoring Workbook'
    serializedData: '{"version":"Notebook/1.0","fallbackResourceIds":["${workspaceId}"]}'
  }
}
```

#### Terraform
```hcl
resource "azurerm_application_insights_workbook" "wb" {
  name                = uuid()
  resource_group_name = azurerm_resource_group.rg.name
  location            = azurerm_resource_group.rg.location
  display_name        = "AZ-305 Monitoring Workbook"
  data_json           = jsonencode({ version = "Notebook/1.0" })
}
```

### Validation and Success Criteria
- Workbook saves successfully.
- Log and metric visuals render without errors.
- Parameters filter the output correctly.
- The workbook can be shared with other team members.

### Verification Steps
- Open the workbook in Azure Monitor.
- Change the time range or parameter scope and confirm results update.
- Share the workbook with a test reader account.

### Cleanup

#### Azure CLI
```bash
az resource delete --resource-group $RG --resource-type Microsoft.Insights/workbooks --name $WORKBOOK_ID
rm -f workbook.json
az group delete -n $RG --yes --no-wait
```

#### PowerShell
```powershell
Remove-AzResource -ResourceGroupName 'rg-monitor-lab06' -ResourceType 'Microsoft.Insights/workbooks' -Name '<workbook-guid>' -ApiVersion '2022-04-01' -Force
Remove-AzResourceGroup -Name 'rg-monitor-lab06' -Force
```

### Exam Tip
> 💡 Use **workbooks** for interactive analysis and **dashboards** for simple, at-a-glance operational summaries. This distinction appears frequently in design questions.

### Exam-style Review Questions
1. Why would you choose a workbook instead of an Azure dashboard?
2. How do parameters improve workbook usability for multi-team operations?
3. What makes workbooks suitable for troubleshooting runbooks?

---

## Lab 7: Azure Monitor Agent (AMA) Deployment

### Objective
Deploy Azure Monitor Agent, create DCRs for Windows events and performance counters, compare AMA with the legacy Log Analytics agent, and validate data collection.

**When to Use This:** Use AMA when you need the modern, policy-driven, scalable agent for OS-level event and performance collection from Azure or Arc-connected machines.

**Key AZ-305 Concepts:** AMA vs. MMA, DCR-driven collection, managed identity support, and private ingestion options with DCE.

### Exam Domain Mapping
- **Primary:** Design identity, governance, and monitoring solutions (25-30%)
- **Secondary:** Design infrastructure solutions (30-35%)

### Prerequisites
- Existing Windows VM
- Log Analytics workspace
- Contributor permissions on the VM and monitoring resources
- `Microsoft.Azure.Monitor` extension support on the VM image

### Architecture and Design Rationale
AMA is the strategic replacement for the Log Analytics agent (MMA). The exam expects you to know that AMA separates **agent deployment** from **collection configuration** through DCRs. This enables reusable, centralized monitoring policy and cleaner lifecycle management.

### Implementation Steps
1. Deploy AMA to the VM.
2. Create a DCR for Windows event logs.
3. Create a DCR for performance counters.
4. Associate both rules with the VM.
5. Validate `Heartbeat`, `Event`, and `Perf` data.

### Full CLI + PowerShell + KQL Commands

#### Azure CLI
```bash
RG="rg-monitor-lab07"
LOCATION="eastus"
LAW_ID="<workspace-resource-id>"
VM_NAME="<windows-vm-name>"
VM_ID=$(az vm show -g $RG -n $VM_NAME --query id -o tsv)

az vm extension set \
  --resource-group $RG \
  --vm-name $VM_NAME \
  --publisher Microsoft.Azure.Monitor \
  --name AzureMonitorWindowsAgent

cat > dcr-windows.json <<'EOF'
{
  "location": "eastus",
  "properties": {
    "dataSources": {
      "windowsEventLogs": [
        {
          "name": "winEvents",
          "streams": ["Microsoft-Event"],
          "xPathQueries": ["System!*[System[(Level=1 or Level=2 or Level=3)]]"]
        }
      ],
      "performanceCounters": [
        {
          "name": "cpuPerf",
          "streams": ["Microsoft-Perf"],
          "samplingFrequencyInSeconds": 60,
          "counterSpecifiers": ["\\Processor(_Total)\\% Processor Time"]
        }
      ]
    },
    "destinations": {
      "logAnalytics": [
        {
          "name": "laDest",
          "workspaceResourceId": "<workspace-resource-id>"
        }
      ]
    },
    "dataFlows": [
      {
        "streams": ["Microsoft-Event"],
        "destinations": ["laDest"]
      },
      {
        "streams": ["Microsoft-Perf"],
        "destinations": ["laDest"]
      }
    ]
  }
}
EOF

az monitor data-collection rule create -g $RG -n dcr-win-ama --location $LOCATION --rule-file dcr-windows.json
az monitor data-collection rule association create --name assoc-ama --resource $VM_ID --rule-id $(az monitor data-collection rule show -g $RG -n dcr-win-ama --query id -o tsv)
```

#### PowerShell
```powershell
$rg = 'rg-monitor-lab07'
$vmName = '<windows-vm-name>'
Set-AzVMExtension -ResourceGroupName $rg -VMName $vmName -Publisher 'Microsoft.Azure.Monitor' -ExtensionType 'AzureMonitorWindowsAgent' -Name 'AzureMonitorWindowsAgent' -Location 'eastus'

# Compare with legacy agent conceptually
'AMA uses DCRs and is the recommended agent. MMA is legacy and should not be selected for new designs.'
```

#### KQL
```kusto
Heartbeat
| where TimeGenerated > ago(1h)
| where Category == 'Azure Monitor Agent'
| summarize LastSeen=max(TimeGenerated) by Computer, Version

Event
| where TimeGenerated > ago(1h)
| summarize Events=count() by Computer, EventLevelName, EventLog

Perf
| where TimeGenerated > ago(1h)
| where CounterName == '% Processor Time'
| summarize AvgCPU=avg(CounterValue) by Computer, bin(TimeGenerated, 5m)
| render timechart
```

### IaC Implementation

#### Bicep
```bicep
param vmName string
param vmLocation string = resourceGroup().location

resource ama 'Microsoft.Compute/virtualMachines/extensions@2023-09-01' = {
  name: '${vmName}/AzureMonitorWindowsAgent'
  location: vmLocation
  properties: {
    publisher: 'Microsoft.Azure.Monitor'
    type: 'AzureMonitorWindowsAgent'
    typeHandlerVersion: '1.0'
    autoUpgradeMinorVersion: true
  }
}
```

#### Terraform
```hcl
resource "azurerm_virtual_machine_extension" "ama" {
  name                 = "AzureMonitorWindowsAgent"
  virtual_machine_id   = var.vm_id
  publisher            = "Microsoft.Azure.Monitor"
  type                 = "AzureMonitorWindowsAgent"
  type_handler_version = "1.0"
}
```

### Validation and Success Criteria
- AMA extension is installed successfully.
- DCR is associated to the VM.
- `Heartbeat`, `Event`, and `Perf` tables receive data.
- You can explain why AMA is preferred over MMA.

### Verification Steps
- Run `az vm extension list -g $RG --vm-name $VM_NAME -o table`
- Query KQL after waiting several minutes for ingestion.
- Confirm DCR association in portal or CLI.

### Cleanup

#### Azure CLI
```bash
az monitor data-collection rule association delete --name assoc-ama --resource $VM_ID
az monitor data-collection rule delete -g $RG -n dcr-win-ama -y
rm -f dcr-windows.json
```

#### PowerShell
```powershell
Remove-AzVMExtension -ResourceGroupName 'rg-monitor-lab07' -VMName '<windows-vm-name>' -Name 'AzureMonitorWindowsAgent' -Force
```

### Exam Tip
> 💡 For new designs, choose **AMA + DCR**. The legacy Log Analytics agent is the distractor answer in many AZ-305 questions.

### Exam-style Review Questions
1. Why is AMA easier to manage at scale than MMA?
2. What problem do DCRs solve?
3. When would you add a Data Collection Endpoint (DCE)?

---

## Lab 8: Microsoft Defender for Cloud

### Objective
Enable Defender for Cloud, review Secure Score, enable workload protection plans, configure auto-provisioning, and review regulatory compliance findings.

**When to Use This:** Use Defender for Cloud when you need cloud security posture management (CSPM), workload protection, recommendations, and regulatory compliance reporting.

**Key AZ-305 Concepts:** Secure Score, Defender plans, compliance dashboards, just-in-time access, and posture remediation.

### Exam Domain Mapping
- **Primary:** Design identity, governance, and monitoring solutions (25-30%)

### Prerequisites
- Owner, Security Admin, or equivalent role on the subscription
- `Microsoft.Security` provider registered
- Test resources such as VMs, SQL, and storage accounts for recommendations

### Architecture and Design Rationale
Defender for Cloud combines posture management with workload protection. For the exam, know the difference between **foundational posture management** and **paid Defender plans** such as Servers, SQL, and Storage. Enable only the plans needed to balance risk coverage and cost.

### Implementation Steps
1. Register the provider and review current pricing tiers.
2. Enable Defender plans for Servers, SQL, and Storage.
3. Configure auto-provisioning where appropriate.
4. Review Secure Score and recommendations.
5. Review regulatory compliance controls.

### Full CLI + PowerShell + KQL Commands

#### Azure CLI
```bash
SUB_ID=$(az account show --query id -o tsv)

az provider register --namespace Microsoft.Security
az security pricing list -o table

az security pricing create --name VirtualMachines --tier Standard --subplan P2
az security pricing create --name SqlServers --tier Standard
az security pricing create --name StorageAccounts --tier Standard --subplan DefenderForStorageV2

az security auto-provisioning-setting update --name default --auto-provision On
az security secure-score-controls list --query "[].{Control:displayName,Score:current,Max:max}" -o table
az security assessment list --query "[?status.code=='Unhealthy'].{Recommendation:displayName,Resource:resourceDetails.id}" -o table
az security regulatory-compliance-standards list -o table
```

#### PowerShell
```powershell
Register-AzResourceProvider -ProviderNamespace Microsoft.Security
Get-AzSecurityPricing | Select-Object Name, PricingTier, SubPlan
Set-AzSecurityPricing -Name 'VirtualMachines' -PricingTier 'Standard' -SubPlan 'P2'
Set-AzSecurityPricing -Name 'SqlServers' -PricingTier 'Standard'
Set-AzSecurityPricing -Name 'StorageAccounts' -PricingTier 'Standard' -SubPlan 'DefenderForStorageV2'
Set-AzSecurityAutoProvisioningSetting -Name 'default' -EnableAutoProvision
Get-AzSecuritySecureScoreControl | Select-Object DisplayName, Current, Max
```

#### KQL
```kusto
SecurityRecommendation
| summarize Recommendations=count() by RecommendationSeverity, RecommendationName
| order by Recommendations desc

SecurityAlert
| where TimeGenerated > ago(7d)
| summarize Alerts=count() by AlertSeverity, ProductName
```

### IaC Implementation

#### Bicep
```bicep
resource vmPlan 'Microsoft.Security/pricings@2024-01-01' = {
  name: 'VirtualMachines'
  properties: {
    pricingTier: 'Standard'
    subPlan: 'P2'
  }
}
```

#### Terraform
```hcl
resource "azurerm_security_center_subscription_pricing" "vm" {
  tier          = "Standard"
  resource_type = "VirtualMachines"
  subplan       = "P2"
}
```

### Validation and Success Criteria
- Relevant Defender plans are enabled.
- Secure Score and recommendations are visible.
- Auto-provisioning is configured as intended.
- Compliance standards can be reviewed.

### Verification Steps
- Review Defender for Cloud in the portal.
- Confirm pricing tiers from CLI/PowerShell.
- Query `SecurityAlert` or `SecurityRecommendation` if data exists.

### Cleanup

#### Azure CLI
```bash
az security pricing create --name VirtualMachines --tier Free
az security pricing create --name SqlServers --tier Free
az security pricing create --name StorageAccounts --tier Free
az security auto-provisioning-setting update --name default --auto-provision Off
```

#### PowerShell
```powershell
Set-AzSecurityPricing -Name 'VirtualMachines' -PricingTier 'Free'
Set-AzSecurityPricing -Name 'SqlServers' -PricingTier 'Free'
Set-AzSecurityPricing -Name 'StorageAccounts' -PricingTier 'Free'
Set-AzSecurityAutoProvisioningSetting -Name 'default' -DisableAutoProvision
```

### Exam Tip
> 💡 Defender for Cloud is both a **governance/security posture** service and a **workload protection** service. Pick the right plan mix based on risk and cost, not “enable everything” by default.

### Exam-style Review Questions
1. What is the design difference between Secure Score and a Defender plan?
2. Why might an architect enable Defender for Servers but not every other paid plan?
3. How does Defender for Cloud complement Sentinel rather than replace it?

---

## Lab 9: Activity Log Monitoring

### Objective
Configure Activity Log export, route it to Log Analytics, create an alert on administrative operations, and plan for long-term retention.

**When to Use This:** Use Activity Log monitoring when you need auditing for control-plane operations such as resource creation, deletion, policy assignment, or RBAC changes.

**Key AZ-305 Concepts:** Subscription-level audit logs, administrative operation alerts, retention design, and separating control-plane from data-plane logging.

### Exam Domain Mapping
- **Primary:** Design identity, governance, and monitoring solutions (25-30%)

### Prerequisites
- Subscription-level permissions
- Log Analytics workspace and optional storage account for long-term retention
- `Microsoft.Insights` provider registered

### Architecture and Design Rationale
Activity Log captures **control-plane** events automatically at the subscription scope. Export it when you need long-term retention, central querying, or correlation with operational and security data. Use it for governance and change tracking, not application troubleshooting.

### Implementation Steps
1. Enable diagnostic settings at the subscription scope.
2. Send Activity Log categories to Log Analytics and optionally storage.
3. Create an Activity Log alert for administrative operations.
4. Query the log data to confirm ingestion.

### Full CLI + PowerShell + KQL Commands

#### Azure CLI
```bash
SUB_ID=$(az account show --query id -o tsv)
LAW_ID="<workspace-resource-id>"
ARCHIVE_ID="<storage-account-resource-id>"

az monitor diagnostic-settings create \
  --name activitylog-to-law \
  --resource "/subscriptions/$SUB_ID" \
  --workspace $LAW_ID \
  --storage-account $ARCHIVE_ID \
  --logs '[{"category":"Administrative","enabled":true},{"category":"Policy","enabled":true},{"category":"Security","enabled":true},{"category":"ServiceHealth","enabled":true}]'

az monitor activity-log alert create \
  --name admin-ops-alert \
  --resource-group rg-monitor-lab03 \
  --scopes "/subscriptions/$SUB_ID" \
  --condition category=Administrative level=Informational \
  --action-group <action-group-id>
```

#### PowerShell
```powershell
$subId = (Get-AzContext).Subscription.Id
$lawId = '<workspace-resource-id>'
$storageId = '<storage-account-resource-id>'

Set-AzDiagnosticSetting -Name 'activitylog-to-law' -ResourceId "/subscriptions/$subId" -WorkspaceId $lawId -StorageAccountId $storageId -Enabled $true

Set-AzActivityLogAlert -Name 'admin-ops-alert' -ResourceGroupName 'rg-monitor-lab03' -Location 'Global' -Scope "/subscriptions/$subId" -Condition @(New-AzActivityLogAlertAlertRuleAnyOfOrLeafConditionObject -Field 'category' -Equal 'Administrative') -ActionGroupId '<action-group-id>'
```

#### KQL
```kusto
AzureActivity
| where TimeGenerated > ago(24h)
| summarize Events=count() by OperationNameValue, ActivityStatusValue, Caller
| order by Events desc

AzureActivity
| where Category == 'Administrative'
| where OperationNameValue has 'roleAssignments'
| project TimeGenerated, Caller, OperationNameValue, ActivityStatusValue, ResourceGroup, ResourceId
```

### IaC Implementation

#### Bicep
```bicep
param workspaceId string
param storageAccountId string

resource subDiag 'Microsoft.Insights/diagnosticSettings@2021-05-01-preview' = {
  scope: subscription()
  name: 'activitylog-to-law'
  properties: {
    workspaceId: workspaceId
    storageAccountId: storageAccountId
    logs: [
      {
        category: 'Administrative'
        enabled: true
      }
    ]
  }
}
```

#### Terraform
```hcl
resource "azurerm_monitor_diagnostic_setting" "activity_log" {
  name                       = "activitylog-to-law"
  target_resource_id         = "/subscriptions/${data.azurerm_client_config.current.subscription_id}"
  log_analytics_workspace_id = var.workspace_id
  storage_account_id         = var.storage_account_id

  enabled_log {
    category = "Administrative"
  }
}
```

### Validation and Success Criteria
- Activity Log export is enabled.
- Administrative events appear in Log Analytics.
- Alerting is configured for governance-sensitive operations.
- Long-term retention destination is available.

### Verification Steps
- Create a harmless tag change on a test resource to generate an administrative event.
- Query `AzureActivity`.
- Review the activity log alert in Azure Monitor.

### Cleanup

#### Azure CLI
```bash
az monitor activity-log alert delete -g rg-monitor-lab03 -n admin-ops-alert
az monitor diagnostic-settings delete --name activitylog-to-law --resource "/subscriptions/$SUB_ID"
```

#### PowerShell
```powershell
Remove-AzActivityLogAlert -ResourceGroupName 'rg-monitor-lab03' -Name 'admin-ops-alert'
Remove-AzDiagnosticSetting -Name 'activitylog-to-law' -ResourceId "/subscriptions/$((Get-AzContext).Subscription.Id)"
```

### Exam Tip
> 💡 Activity Log is for **control-plane** events. If a question asks about administrative changes, policy assignments, or RBAC modifications, Activity Log is usually the correct source.

### Exam-style Review Questions
1. What is the difference between Activity Log and resource diagnostic logs?
2. Why export Activity Log if Azure already captures it natively?
3. Which types of events belong in Activity Log rather than Application Insights?

---

## Lab 10: Multi-Resource Monitoring Dashboard

### Objective
Create a shared Azure dashboard that surfaces metrics, log query results, Service Health, and Advisor recommendations for an operations team.

**When to Use This:** Use Azure dashboards when the team needs a single-pane-of-glass operational summary without deep interactivity.

**Key AZ-305 Concepts:** Shared dashboards, pinned metrics, pinned query results, service health visibility, and role-based sharing.

### Exam Domain Mapping
- **Primary:** Design identity, governance, and monitoring solutions (25-30%)

### Prerequisites
- Existing resources with metrics and logs
- Access to Azure Monitor, Service Health, and Advisor
- Reader or higher on the underlying resources and dashboard resource group

### Architecture and Design Rationale
Dashboards are best for quick status views across multiple resources. They are simpler than workbooks and better suited for NOC wallboards or daily operations summaries. Use pinned charts and queries; use workbooks when filtering and drilldowns matter more.

### Implementation Steps
1. Pin VM, storage, and App Service metrics.
2. Pin a Log Analytics query result.
3. Add Service Health and Advisor tiles.
4. Share the dashboard with the operations team.

### Full CLI + PowerShell + KQL Commands

#### Azure CLI
```bash
RG="rg-monitor-lab10"
DASHBOARD="db-ops-overview"
DASHBOARD_FILE="dashboard.json"

cat > $DASHBOARD_FILE <<'EOF'
{
  "location": "eastus",
  "properties": {
    "lenses": {
      "0": {
        "order": 0,
        "parts": {
          "0": {
            "position": {"x": 0, "y": 0, "rowSpan": 4, "colSpan": 6},
            "metadata": {"type": "Extension/HubsExtension/PartType/MarkdownPart", "settings": {"content": {"content": "# AZ-305 Operations Dashboard"}}}
          }
        }
      }
    },
    "metadata": {"model": {}}
  }
}
EOF

az resource create \
  --resource-group $RG \
  --resource-type Microsoft.Portal/dashboards \
  --name $DASHBOARD \
  --api-version 2020-09-01-preview \
  --is-full-object \
  --properties @$DASHBOARD_FILE
```

#### PowerShell
```powershell
$rg = 'rg-monitor-lab10'
$dashboardName = 'db-ops-overview'
$payload = @{
  location   = 'eastus'
  properties = @{
    lenses = @{
      '0' = @{
        order = 0
        parts = @{}
      }
    }
    metadata = @{ model = @{} }
  }
} | ConvertTo-Json -Depth 20

New-AzResource -ResourceGroupName $rg -ResourceType 'Microsoft.Portal/dashboards' -Name $dashboardName -ApiVersion '2020-09-01-preview' -PropertyObject ($payload | ConvertFrom-Json) -Location 'eastus' -Force
```

#### KQL
```kusto
Heartbeat
| summarize Healthy=countif(TimeGenerated > ago(15m)), Total=dcount(Computer)

AzureActivity
| where TimeGenerated > ago(24h)
| summarize Ops=count() by ActivityStatusValue
```

### IaC Implementation

#### Bicep
```bicep
param dashboardName string
param location string = resourceGroup().location

resource dashboard 'Microsoft.Portal/dashboards@2020-09-01-preview' = {
  name: dashboardName
  location: location
  properties: {
    lenses: {
      '0': {
        order: 0
        parts: {}
      }
    }
    metadata: {
      model: {}
    }
  }
}
```

#### Terraform
```hcl
resource "azurerm_dashboard" "ops" {
  name                = "db-ops-overview"
  resource_group_name = azurerm_resource_group.rg.name
  location            = azurerm_resource_group.rg.location
  dashboard_properties = jsonencode({ lenses = { "0" = { order = 0, parts = {} } }, metadata = { model = {} } })
}
```

### Validation and Success Criteria
- Dashboard is created and visible to the team.
- Multiple metrics and at least one log query tile are pinned.
- Service Health and Advisor views are available.

### Verification Steps
- Open the dashboard and validate each tile loads.
- Share the dashboard to another user/group and verify access.
- Confirm the dashboard reflects current metric and log data.

### Cleanup

#### Azure CLI
```bash
az resource delete --resource-group $RG --resource-type Microsoft.Portal/dashboards --name $DASHBOARD
rm -f $DASHBOARD_FILE
```

#### PowerShell
```powershell
Remove-AzResource -ResourceGroupName 'rg-monitor-lab10' -ResourceType 'Microsoft.Portal/dashboards' -Name 'db-ops-overview' -ApiVersion '2020-09-01-preview' -Force
```

### Exam Tip
> 💡 Dashboards are optimized for **summary views**. Workbooks are optimized for **interactive analysis**. If the requirement says “single pane of glass for the ops team,” dashboards are often the best answer.

### Exam-style Review Questions
1. Why would an architect choose a dashboard instead of a workbook?
2. Which monitoring data is best pinned to a dashboard for executives vs. operators?
3. How do Azure RBAC assignments affect shared dashboard access?

---

## Lab 11: Sentinel Basics (SIEM)

### Objective
Create a Sentinel-enabled workspace, connect Azure Activity and Entra data sources, create a simple analytics rule, review incidents, and trigger a Logic App playbook.

**When to Use This:** Use Microsoft Sentinel when you need SIEM/SOAR capabilities, security correlation, investigation, incident management, and automation.

**Key AZ-305 Concepts:** SIEM vs. Defender for Cloud, data connectors, analytics rules, incidents, automation, and workspace-based onboarding.

### Exam Domain Mapping
- **Primary:** Design identity, governance, and monitoring solutions (25-30%)

### Prerequisites
- Log Analytics workspace
- `Microsoft.SecurityInsights` provider registered
- Security Reader or Sentinel Contributor permissions
- Optional Logic App permissions for playbook execution

### Architecture and Design Rationale
Sentinel uses Log Analytics as its data platform and adds SIEM/SOAR capabilities on top. For the exam, remember the relationship: **Defender for Cloud produces security posture and workload protection insights; Sentinel aggregates, correlates, investigates, and automates response**.

### Implementation Steps
1. Enable Sentinel on an existing Log Analytics workspace.
2. Connect Azure Activity and Entra logs.
3. Create an analytics rule using KQL.
4. Review generated incidents.
5. Link a Logic App playbook for basic response automation.

### Full CLI + PowerShell + KQL Commands

#### Azure CLI
```bash
RG="rg-monitor-lab11"
LOCATION="eastus"
LAW_NAME="law-sentinel-$RANDOM"
SUB_ID=$(az account show --query id -o tsv)

az provider register --namespace Microsoft.SecurityInsights
az group create -n $RG -l $LOCATION
az monitor log-analytics workspace create -g $RG -n $LAW_NAME -l $LOCATION --retention-time 30

# Enable Sentinel onboarding
az rest --method PUT \
  --url "https://management.azure.com/subscriptions/$SUB_ID/resourceGroups/$RG/providers/Microsoft.OperationalInsights/workspaces/$LAW_NAME/providers/Microsoft.SecurityInsights/onboardingStates/default?api-version=2024-03-01" \
  --body '{"properties":{}}'

# Create a simple scheduled analytics rule
cat > sentinel-rule.json <<'EOF'
{
  "kind": "Scheduled",
  "properties": {
    "displayName": "Multiple failed sign-ins",
    "enabled": true,
    "query": "SigninLogs | where ResultType != 0 | summarize Failed=count() by UserPrincipalName, bin(TimeGenerated, 5m) | where Failed >= 5",
    "queryFrequency": "PT5M",
    "queryPeriod": "PT15M",
    "severity": "Medium",
    "triggerOperator": "GreaterThan",
    "triggerThreshold": 0
  }
}
EOF

az rest --method PUT \
  --url "https://management.azure.com/subscriptions/$SUB_ID/resourceGroups/$RG/providers/Microsoft.OperationalInsights/workspaces/$LAW_NAME/providers/Microsoft.SecurityInsights/alertRules/multiple-failed-signins?api-version=2024-03-01" \
  --body @sentinel-rule.json
```

#### PowerShell
```powershell
$rg = 'rg-monitor-lab11'
$location = 'eastus'
$lawName = "law-sentinel-$(Get-Random)"
$subId = (Get-AzContext).Subscription.Id

Register-AzResourceProvider -ProviderNamespace Microsoft.SecurityInsights
New-AzResourceGroup -Name $rg -Location $location
$law = New-AzOperationalInsightsWorkspace -ResourceGroupName $rg -Name $lawName -Location $location -Sku PerGB2018 -RetentionInDays 30

Invoke-AzRestMethod -Method PUT -Path "/subscriptions/$subId/resourceGroups/$rg/providers/Microsoft.OperationalInsights/workspaces/$lawName/providers/Microsoft.SecurityInsights/onboardingStates/default?api-version=2024-03-01" -Payload '{"properties":{}}'
```

#### KQL
```kusto
SigninLogs
| where TimeGenerated > ago(1d)
| where ResultType != 0
| summarize Failed=count() by UserPrincipalName, IPAddress, bin(TimeGenerated, 15m)
| order by Failed desc

SecurityIncident
| where TimeGenerated > ago(7d)
| summarize Incidents=count() by Status, Severity
```

### IaC Implementation

#### Bicep
```bicep
param workspaceName string

resource onboarding 'Microsoft.OperationalInsights/workspaces/providers/onboardingStates@2024-03-01' = {
  name: '${workspaceName}/Microsoft.SecurityInsights/default'
  properties: {}
}
```

#### Terraform
```hcl
resource "azurerm_log_analytics_workspace" "sentinel" {
  name                = var.workspace_name
  location            = azurerm_resource_group.rg.location
  resource_group_name = azurerm_resource_group.rg.name
  sku                 = "PerGB2018"
  retention_in_days   = 30
}
```

### Validation and Success Criteria
- Sentinel is enabled on the workspace.
- At least one relevant data source is connected.
- A scheduled analytics rule is active.
- Incidents can be reviewed from the workspace.

### Verification Steps
- Open Sentinel in the Azure portal and confirm onboarding.
- Validate the alert rule exists.
- Query `SecurityIncident` after rule activity occurs.

### Cleanup

#### Azure CLI
```bash
az rest --method DELETE \
  --url "https://management.azure.com/subscriptions/$SUB_ID/resourceGroups/$RG/providers/Microsoft.OperationalInsights/workspaces/$LAW_NAME/providers/Microsoft.SecurityInsights/alertRules/multiple-failed-signins?api-version=2024-03-01"
rm -f sentinel-rule.json
az group delete -n $RG --yes --no-wait
```

#### PowerShell
```powershell
Remove-AzResourceGroup -Name 'rg-monitor-lab11' -Force
```

### Exam Tip
> 💡 Sentinel is the right answer when the requirement includes **correlation across many data sources, incident investigation, and automated response**. Defender for Cloud alone is not a SIEM.

### Exam-style Review Questions
1. Why is Sentinel considered a SIEM/SOAR layer instead of just another monitoring tool?
2. How does Sentinel relate to Log Analytics and Defender for Cloud?
3. When should a playbook be attached to an analytics rule?

---

## Decision Summary Table

| Requirement | Best-fit Azure monitoring feature | Why | Common exam trap |
|---|---|---|---|
| Centralized operational log collection | Log Analytics workspace | Cross-resource KQL, alerts, workbooks, Sentinel integration | Using only Activity Log for workload telemetry |
| Capture resource telemetry from Azure services | Diagnostic settings | Routes logs/metrics to Log Analytics, Storage, Event Hubs | Assuming resources send logs automatically without diag settings |
| Fast threshold-based CPU or availability alerting | Metric alerts | Lower latency and lower cost than log alerts | Choosing a log alert for a simple platform metric |
| Correlated detection from logs | Scheduled query (log) alerts | KQL-based filtering, aggregation, multi-table logic | Forgetting query cost and lookback tuning |
| Application performance monitoring | Application Insights | Requests, dependencies, exceptions, distributed tracing | Using platform metrics only for app troubleshooting |
| Interactive troubleshooting and reporting | Azure Monitor Workbooks | Parameters, KQL visuals, drill-through | Choosing dashboards when interactivity is required |
| OS event and performance collection from VMs | Azure Monitor Agent + DCR | Modern agent model, reusable collection rules | Selecting the legacy Log Analytics agent for new designs |
| Security posture and workload protection | Defender for Cloud | Secure Score, recommendations, Defender plans | Confusing Defender for Cloud with Sentinel SIEM capabilities |
| Subscription control-plane auditing | Activity Log + export | Administrative, policy, service health, RBAC events | Using App Insights or resource logs for control-plane auditing |
| Single pane of glass summary | Azure Dashboard | Shared, lightweight, pinned visuals | Choosing workbooks when only a wallboard is needed |
| SIEM/SOAR and incident response | Microsoft Sentinel | Correlation, incidents, hunting, automation | Expecting Defender for Cloud alone to replace a SIEM |
