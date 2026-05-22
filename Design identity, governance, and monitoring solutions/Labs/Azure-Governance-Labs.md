# Azure Governance Hands-On Labs (AZ-305)

> 📖 **Cheat Sheet:** [Azure Governance](../Azure-Governance.md)

> **Exam Focus:** AZ-305 – Design identity, governance, and monitoring solutions  
> **Primary Domain:** Design identity, governance, and monitoring solutions (25-30%)  
> **Tools Used:** Azure CLI, Azure PowerShell, Azure Policy, Resource Graph, Defender for Cloud, Cost Management

---

## Lab 1: Management Group Hierarchy Design

### Objective
Design and deploy a management group hierarchy that separates environments from workloads, then validate inheritance for policy and RBAC.

### When to Use This
Use management groups when you need enterprise-wide governance, subscription segmentation, and inherited policy/RBAC across many subscriptions.

### Exam domain mapping
- **Primary:** Design identity, governance, and monitoring solutions (25-30%)
- **Secondary:** Design infrastructure solutions (30-35%)

### Key AZ-305 Concepts
- Tenant root group vs. custom management groups
- Subscription organization by environment and workload
- Inheritance of policy and RBAC
- Governance at scale

### Prerequisites
- Azure tenant with at least two test subscriptions
- **Management Group Contributor** and **User Access Administrator** at tenant root or parent management group
- Security group object ID for RBAC testing
- Azure CLI logged in with `az login`
- Azure PowerShell logged in with `Connect-AzAccount`

### Architecture and design rationale
Recommended hierarchy for the exam:

| Level | Example | Why it exists |
|---|---|---|
| Tenant root group | Built-in | Top-level inheritance boundary |
| Enterprise group | `az305-enterprise` | Common landing zone parent |
| Environment groups | `mg-prod`, `mg-nonprod` | Separate risk and policy baselines |
| Workload groups | `mg-prod-shared`, `mg-prod-apps` | Delegate ownership without losing guardrails |

Design guidance:
- Put broad governance at higher scopes.
- Put workload-specific RBAC and policy lower in the tree.
- Avoid deep hierarchies that make inheritance hard to troubleshoot.

### Implementation steps

#### Azure CLI
```bash
# Variables
ROOT_ID=$(az account management-group list --query "[?details.parent==null].name | [0]" -o tsv)
ENTERPRISE_MG="az305-enterprise"
PROD_MG="mg-prod"
NONPROD_MG="mg-nonprod"
PROD_APPS_MG="mg-prod-apps"
PROD_SHARED_MG="mg-prod-shared"
NONPROD_APPS_MG="mg-nonprod-apps"
PROD_SUB="<prod-subscription-id>"
NONPROD_SUB="<nonprod-subscription-id>"
AAD_GROUP_OBJECT_ID="<entra-group-object-id>"

# Create hierarchy under tenant root
az account management-group create --name $ENTERPRISE_MG --display-name "AZ305 Enterprise" --parent $ROOT_ID
az account management-group create --name $PROD_MG --display-name "Production" --parent $ENTERPRISE_MG
az account management-group create --name $NONPROD_MG --display-name "Non-Production" --parent $ENTERPRISE_MG
az account management-group create --name $PROD_APPS_MG --display-name "Prod Apps" --parent $PROD_MG
az account management-group create --name $PROD_SHARED_MG --display-name "Prod Shared Services" --parent $PROD_MG
az account management-group create --name $NONPROD_APPS_MG --display-name "NonProd Apps" --parent $NONPROD_MG

# Move subscriptions into the correct environment groups
az account management-group subscription add --name $PROD_MG --subscription $PROD_SUB
az account management-group subscription add --name $NONPROD_MG --subscription $NONPROD_SUB

# View hierarchy
az account management-group show --name $ENTERPRISE_MG --expand --recurse -o jsonc

# View effective policy assignments at different scopes
az policy assignment list --scope "/providers/Microsoft.Management/managementGroups/$PROD_MG" --include-inherited -o table
az policy assignment list --scope "/providers/Microsoft.Management/managementGroups/$PROD_APPS_MG" --include-inherited -o table

# Assign RBAC at management group scope
az role assignment create \
  --assignee-object-id $AAD_GROUP_OBJECT_ID \
  --assignee-principal-type Group \
  --role Reader \
  --scope "/providers/Microsoft.Management/managementGroups/$PROD_MG"

# Review RBAC inheritance at workload level
az role assignment list \
  --scope "/providers/Microsoft.Management/managementGroups/$PROD_APPS_MG" \
  --include-inherited -o table
```

#### PowerShell
```powershell
# Variables
$RootId = (Get-AzManagementGroup | Where-Object { -not $_.ParentId }).Name
$EnterpriseMg = "az305-enterprise"
$ProdMg = "mg-prod"
$NonProdMg = "mg-nonprod"
$ProdAppsMg = "mg-prod-apps"
$ProdSharedMg = "mg-prod-shared"
$NonProdAppsMg = "mg-nonprod-apps"
$ProdSub = "<prod-subscription-id>"
$NonProdSub = "<nonprod-subscription-id>"
$GroupObjectId = "<entra-group-object-id>"

# Create hierarchy
New-AzManagementGroup -GroupName $EnterpriseMg -DisplayName "AZ305 Enterprise" -ParentId $RootId
New-AzManagementGroup -GroupName $ProdMg -DisplayName "Production" -ParentId $EnterpriseMg
New-AzManagementGroup -GroupName $NonProdMg -DisplayName "Non-Production" -ParentId $EnterpriseMg
New-AzManagementGroup -GroupName $ProdAppsMg -DisplayName "Prod Apps" -ParentId $ProdMg
New-AzManagementGroup -GroupName $ProdSharedMg -DisplayName "Prod Shared Services" -ParentId $ProdMg
New-AzManagementGroup -GroupName $NonProdAppsMg -DisplayName "NonProd Apps" -ParentId $NonProdMg

# Move subscriptions
New-AzManagementGroupSubscription -GroupName $ProdMg -SubscriptionId $ProdSub
New-AzManagementGroupSubscription -GroupName $NonProdMg -SubscriptionId $NonProdSub

# View hierarchy
Get-AzManagementGroup -GroupName $EnterpriseMg -Expand -Recurse

# View policy assignments by scope
$ProdScope = "/providers/Microsoft.Management/managementGroups/$ProdMg"
$ProdAppsScope = "/providers/Microsoft.Management/managementGroups/$ProdAppsMg"
Get-AzPolicyAssignment -Scope $ProdScope | Select-Object Name, Scope
Get-AzPolicyAssignment -Scope $ProdAppsScope | Select-Object Name, Scope

# Assign RBAC at management group scope
New-AzRoleAssignment -ObjectId $GroupObjectId -RoleDefinitionName Reader -Scope $ProdScope

# Review inherited RBAC at workload scope
Get-AzRoleAssignment -Scope $ProdAppsScope | Select-Object DisplayName, RoleDefinitionName, Scope
```

### IaC implementation
**Bicep (management group creation starter):**
```bicep
targetScope = 'tenant'

resource enterprise 'Microsoft.Management/managementGroups@2023-04-01' = {
  name: 'az305-enterprise'
  properties: {
    displayName: 'AZ305 Enterprise'
    details: {
      parent: {
        id: tenantResourceId('Microsoft.Management/managementGroups', tenant().tenantId)
      }
    }
  }
}
```

### Verification steps
- Confirm the hierarchy renders correctly in Azure portal and CLI/PowerShell output.
- Confirm subscriptions appear under the intended environment groups.
- Compare `include-inherited` policy assignment output at parent and child scopes.
- Confirm the Entra group can read child scopes through inherited management group RBAC.

### Cleanup

#### Azure CLI
```bash
az role assignment delete --assignee-object-id $AAD_GROUP_OBJECT_ID --role Reader --scope "/providers/Microsoft.Management/managementGroups/$PROD_MG"
az account management-group subscription remove --name $PROD_MG --subscription $PROD_SUB
az account management-group subscription remove --name $NONPROD_MG --subscription $NONPROD_SUB
az account management-group delete --name $PROD_APPS_MG
az account management-group delete --name $PROD_SHARED_MG
az account management-group delete --name $NONPROD_APPS_MG
az account management-group delete --name $PROD_MG
az account management-group delete --name $NONPROD_MG
az account management-group delete --name $ENTERPRISE_MG
```

#### PowerShell
```powershell
Remove-AzRoleAssignment -ObjectId $GroupObjectId -RoleDefinitionName Reader -Scope "/providers/Microsoft.Management/managementGroups/$ProdMg"
Remove-AzManagementGroupSubscription -GroupName $ProdMg -SubscriptionId $ProdSub
Remove-AzManagementGroupSubscription -GroupName $NonProdMg -SubscriptionId $NonProdSub
Remove-AzManagementGroup -GroupName $ProdAppsMg
Remove-AzManagementGroup -GroupName $ProdSharedMg
Remove-AzManagementGroup -GroupName $NonProdAppsMg
Remove-AzManagementGroup -GroupName $ProdMg
Remove-AzManagementGroup -GroupName $NonProdMg
Remove-AzManagementGroup -GroupName $EnterpriseMg
```

### Exam Tip
**Use management groups for governance inheritance, not for networking or deployment boundaries.** A common trap is overusing subscription-level assignments when a management group assignment would reduce operational overhead.

---

## Lab 2: Custom RBAC Role Creation

### Objective
Create, assign, test, and update a custom RBAC role when built-in roles are too broad or too narrow.

### When to Use This
Use a custom role when built-in roles do not align with least-privilege requirements, especially for operations, support, or automation accounts.

### Exam domain mapping
- **Primary:** Design identity, governance, and monitoring solutions (25-30%)
- **Secondary:** Design infrastructure solutions (30-35%)

### Key AZ-305 Concepts
- Role definition JSON structure
- Management plane permissions vs. data plane permissions
- Assignable scopes
- Custom role lifecycle and propagation delay

### Prerequisites
- Subscription-level **Owner** or **User Access Administrator**
- Test resource group
- Test service principal or Entra security principal
- Azure CLI and Azure PowerShell authenticated

### Architecture and design rationale
This lab creates a role for a support team that can read resources and restart/deallocate VMs, but cannot delete resources or create new ones.

| Decision | Rationale |
|---|---|
| Custom role instead of Contributor | Contributor is too broad |
| RG scope assignment | Limits blast radius |
| Service principal for testing | Repeatable least-privilege validation |

### Implementation steps

#### Azure CLI
```bash
# Variables
SUBSCRIPTION_ID=$(az account show --query id -o tsv)
RG="rg-rbac-lab"
LOCATION="eastus"
VM_NAME="vmrbaclab01"
ROLE_NAME="AZ305 VM Support Operator"
SP_NAME="az305-rbac-sp"

# Create test resource group and small VM
az group create --name $RG --location $LOCATION
az vm create \
  --resource-group $RG \
  --name $VM_NAME \
  --image Ubuntu2204 \
  --size Standard_B1s \
  --admin-username azureuser \
  --generate-ssh-keys

# Review built-in roles
az role definition list --name Reader -o jsonc
az role definition list --name Contributor -o jsonc
az role definition list --name "Virtual Machine Contributor" -o jsonc

# Create a custom role definition file
cat > vm-support-role.json <<EOF
{
  "Name": "$ROLE_NAME",
  "IsCustom": true,
  "Description": "Read resources and manage VM power state without create/delete rights.",
  "Actions": [
    "*/read",
    "Microsoft.Compute/virtualMachines/start/action",
    "Microsoft.Compute/virtualMachines/restart/action",
    "Microsoft.Compute/virtualMachines/deallocate/action"
  ],
  "NotActions": [
    "Microsoft.Authorization/*/Delete",
    "Microsoft.Compute/virtualMachines/delete",
    "Microsoft.Resources/subscriptions/resourceGroups/delete"
  ],
  "DataActions": [],
  "NotDataActions": [],
  "AssignableScopes": [
    "/subscriptions/$SUBSCRIPTION_ID"
  ]
}
EOF

az role definition create --role-definition vm-support-role.json

# Create a service principal to test the role
read APP_ID SP_SECRET TENANT_ID <<< $(az ad sp create-for-rbac --name $SP_NAME --skip-assignment --query "[appId,password,tenant]" -o tsv)
SP_OBJECT_ID=$(az ad sp show --id $APP_ID --query id -o tsv)

# Assign the custom role at resource group scope
az role assignment create \
  --assignee-object-id $SP_OBJECT_ID \
  --assignee-principal-type ServicePrincipal \
  --role "$ROLE_NAME" \
  --scope "/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RG"

# Test permissions in a new session
az login --service-principal -u $APP_ID -p $SP_SECRET --tenant $TENANT_ID
az resource list --resource-group $RG -o table
az vm restart --resource-group $RG --name $VM_NAME
az storage account create --name "strbac$RANDOM" --resource-group $RG --location $LOCATION --sku Standard_LRS
# Expected: authorization failure because create permission is not granted

# Sign back in as your admin account before updating the role
az login

# Update the custom role to add VM power off
cat > vm-support-role.json <<EOF
{
  "Name": "$ROLE_NAME",
  "IsCustom": true,
  "Description": "Read resources and manage VM power state without create/delete rights.",
  "Actions": [
    "*/read",
    "Microsoft.Compute/virtualMachines/start/action",
    "Microsoft.Compute/virtualMachines/restart/action",
    "Microsoft.Compute/virtualMachines/deallocate/action",
    "Microsoft.Compute/virtualMachines/powerOff/action"
  ],
  "NotActions": [
    "Microsoft.Authorization/*/Delete",
    "Microsoft.Compute/virtualMachines/delete",
    "Microsoft.Resources/subscriptions/resourceGroups/delete"
  ],
  "DataActions": [],
  "NotDataActions": [],
  "AssignableScopes": [
    "/subscriptions/$SUBSCRIPTION_ID"
  ]
}
EOF

az role definition update --role-definition vm-support-role.json
```

#### PowerShell
```powershell
# Variables
$SubscriptionId = (Get-AzContext).Subscription.Id
$TenantId = (Get-AzContext).Tenant.Id
$ResourceGroup = "rg-rbac-lab"
$Location = "eastus"
$VmName = "vmrbaclab01"
$RoleName = "AZ305 VM Support Operator"
$SpDisplayName = "az305-rbac-sp"

# Create test resource group and VM
New-AzResourceGroup -Name $ResourceGroup -Location $Location
New-AzVM -ResourceGroupName $ResourceGroup -Name $VmName -Location $Location -VirtualNetworkName "vnet-rbac-lab" -SubnetName "default" -SecurityGroupName "nsg-rbac-lab" -PublicIpAddressName "pip-rbac-lab" -OpenPorts 22 -Image Ubuntu2204 -Credential (Get-Credential)

# Review built-in roles
Get-AzRoleDefinition -Name Reader
Get-AzRoleDefinition -Name Contributor
Get-AzRoleDefinition -Name "Virtual Machine Contributor"

# Create custom role definition file
$RoleJson = @"
{
  "Name": "$RoleName",
  "IsCustom": true,
  "Description": "Read resources and manage VM power state without create/delete rights.",
  "Actions": [
    "*/read",
    "Microsoft.Compute/virtualMachines/start/action",
    "Microsoft.Compute/virtualMachines/restart/action",
    "Microsoft.Compute/virtualMachines/deallocate/action"
  ],
  "NotActions": [
    "Microsoft.Authorization/*/Delete",
    "Microsoft.Compute/virtualMachines/delete",
    "Microsoft.Resources/subscriptions/resourceGroups/delete"
  ],
  "DataActions": [],
  "NotDataActions": [],
  "AssignableScopes": [
    "/subscriptions/$SubscriptionId"
  ]
}
"@
$RoleJson | Set-Content -Path .\vm-support-role.json
New-AzRoleDefinition -InputFile .\vm-support-role.json

# Create service principal for testing
$App = New-AzADApplication -DisplayName $SpDisplayName
$AppSecret = New-AzADAppCredential -ApplicationId $App.AppId -EndDate (Get-Date).AddYears(1)
$Sp = New-AzADServicePrincipal -ApplicationId $App.AppId
$SpSecret = $AppSecret.SecretText
$SpObjectId = $Sp.Id

# Assign custom role at RG scope
New-AzRoleAssignment -ObjectId $SpObjectId -RoleDefinitionName $RoleName -Scope "/subscriptions/$SubscriptionId/resourceGroups/$ResourceGroup"

# Test permissions in a fresh context
$SecureSecret = ConvertTo-SecureString $SpSecret -AsPlainText -Force
$Cred = New-Object System.Management.Automation.PSCredential($App.AppId, $SecureSecret)
Connect-AzAccount -ServicePrincipal -Tenant $TenantId -Credential $Cred
Get-AzResource -ResourceGroupName $ResourceGroup
Restart-AzVM -ResourceGroupName $ResourceGroup -Name $VmName
New-AzStorageAccount -ResourceGroupName $ResourceGroup -Name ("strbac" + (Get-Random)) -Location $Location -SkuName Standard_LRS
# Expected: authorization failure

# Reconnect with admin account and update role
Connect-AzAccount
(Get-Content .\vm-support-role.json) -replace '"Microsoft.Compute/virtualMachines/deallocate/action"', '"Microsoft.Compute/virtualMachines/deallocate/action",`n    "Microsoft.Compute/virtualMachines/powerOff/action"' | Set-Content .\vm-support-role.json
Set-AzRoleDefinition -InputFile .\vm-support-role.json
```

### IaC implementation
**Terraform starter:**
```hcl
resource "azurerm_role_definition" "vm_support" {
  name        = "AZ305 VM Support Operator"
  scope       = data.azurerm_subscription.current.id
  description = "Read resources and manage VM power state without create/delete rights."

  permissions {
    actions = [
      "*/read",
      "Microsoft.Compute/virtualMachines/start/action",
      "Microsoft.Compute/virtualMachines/restart/action",
      "Microsoft.Compute/virtualMachines/deallocate/action"
    ]
    not_actions = ["Microsoft.Compute/virtualMachines/delete"]
  }

  assignable_scopes = [data.azurerm_subscription.current.id]
}
```

### Verification steps
- Confirm the custom role appears with `IsCustom = true`.
- Confirm the test principal can list resources and restart the VM.
- Confirm the test principal cannot create or delete resources.
- Confirm the updated action appears after role update.

### Cleanup

#### Azure CLI
```bash
az role assignment delete --assignee-object-id $SP_OBJECT_ID --role "$ROLE_NAME" --scope "/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RG"
az role definition delete --name "$ROLE_NAME"
az ad app delete --id $APP_ID
az group delete --name $RG --yes --no-wait
rm -f vm-support-role.json
```

#### PowerShell
```powershell
Remove-AzRoleAssignment -ObjectId $SpObjectId -RoleDefinitionName $RoleName -Scope "/subscriptions/$SubscriptionId/resourceGroups/$ResourceGroup"
Remove-AzRoleDefinition -Name $RoleName -Force
Remove-AzADApplication -ApplicationId $App.AppId
Remove-AzResourceGroup -Name $ResourceGroup -Force -AsJob
Remove-Item .\vm-support-role.json -Force
```

### Exam Tip
**Prefer built-in roles first.** Use a custom role only when least privilege cannot be met otherwise, and remember custom roles are still management-plane only unless you explicitly include data actions.

---

## Lab 3: Azure Policy - Enforce Tagging

### Objective
Create custom tag policies, combine them into an initiative, enforce them with **Deny**, create an exemption, and review compliance.

### When to Use This
Use this pattern when your organization requires mandatory tags for cost reporting, ownership, and auditability.

### Exam domain mapping
- **Primary:** Design identity, governance, and monitoring solutions (25-30%)
- **Secondary:** Design business continuity solutions (15-20%)

### Key AZ-305 Concepts
- Policy definition vs. initiative
- `Deny` effect
- Scope and inheritance
- Policy exemptions
- Compliance evaluation

### Prerequisites
- Contributor or Owner on a test resource group
- Azure CLI and PowerShell installed
- A target resource group for assignments

### Architecture and design rationale
Required tags in this lab:
- `costCenter`
- `owner`

Why initiative instead of separate assignments?
- Single assignment for multiple standards
- Easier compliance reporting
- Easier exemption handling

### Implementation steps

#### Azure CLI
```bash
# Variables
SUBSCRIPTION_ID=$(az account show --query id -o tsv)
RG="rg-tag-policy-lab"
LOCATION="eastus"
ASSIGNMENT_SCOPE="/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RG"

az group create --name $RG --location $LOCATION

# Policy: require costCenter tag
cat > require-costcenter.json <<'EOF'
{
  "mode": "Indexed",
  "parameters": {},
  "policyRule": {
    "if": {
      "allOf": [
        { "field": "type", "notEquals": "Microsoft.Resources/subscriptions/resourceGroups" },
        { "field": "tags['costCenter']", "exists": false }
      ]
    },
    "then": { "effect": "deny" }
  }
}
EOF

# Policy: require owner tag
cat > require-owner.json <<'EOF'
{
  "mode": "Indexed",
  "parameters": {},
  "policyRule": {
    "if": {
      "allOf": [
        { "field": "type", "notEquals": "Microsoft.Resources/subscriptions/resourceGroups" },
        { "field": "tags['owner']", "exists": false }
      ]
    },
    "then": { "effect": "deny" }
  }
}
EOF

az policy definition create --name require-costcenter-tag --display-name "Require costCenter tag" --mode Indexed --rules @require-costcenter.json
az policy definition create --name require-owner-tag --display-name "Require owner tag" --mode Indexed --rules @require-owner.json

# Initiative combining both policies
cat > tag-initiative.json <<'EOF'
{
  "properties": {
    "displayName": "AZ305 Tagging Baseline",
    "policyType": "Custom",
    "metadata": { "category": "Tags" },
    "policyDefinitions": [
      {
        "policyDefinitionReferenceId": "costcenter",
        "policyDefinitionId": "/subscriptions/${SUBSCRIPTION_ID}/providers/Microsoft.Authorization/policyDefinitions/require-costcenter-tag"
      },
      {
        "policyDefinitionReferenceId": "owner",
        "policyDefinitionId": "/subscriptions/${SUBSCRIPTION_ID}/providers/Microsoft.Authorization/policyDefinitions/require-owner-tag"
      }
    ]
  }
}
EOF
sed -i "s|\${SUBSCRIPTION_ID}|$SUBSCRIPTION_ID|g" tag-initiative.json
az policy set-definition create --name az305-tagging-baseline --definitions @tag-initiative.json

# Assign initiative
az policy assignment create \
  --name az305-tagging-assignment \
  --display-name "AZ305 Tagging Assignment" \
  --scope $ASSIGNMENT_SCOPE \
  --policy-set-definition az305-tagging-baseline

# Test deny - should fail
az storage account create \
  --name "sttagdeny$RANDOM" \
  --resource-group $RG \
  --location $LOCATION \
  --sku Standard_LRS

# Create compliant resource
az storage account create \
  --name "sttagok$RANDOM" \
  --resource-group $RG \
  --location $LOCATION \
  --sku Standard_LRS \
  --tags costCenter=CC100 owner=cloudops

# Create exemption for a migration workload
az policy exemption create \
  --name migration-exemption \
  --scope $ASSIGNMENT_SCOPE \
  --policy-assignment az305-tagging-assignment \
  --display-name "Migration wave exemption" \
  --description "Temporary exemption for migration resources"

# View compliance
az policy state summarize --resource-group $RG -o jsonc
```

#### PowerShell
```powershell
# Variables
$SubscriptionId = (Get-AzContext).Subscription.Id
$ResourceGroup = "rg-tag-policy-lab"
$Location = "eastus"
$AssignmentScope = "/subscriptions/$SubscriptionId/resourceGroups/$ResourceGroup"

New-AzResourceGroup -Name $ResourceGroup -Location $Location

# Policy files
@"
{
  "mode": "Indexed",
  "parameters": {},
  "policyRule": {
    "if": {
      "allOf": [
        { "field": "type", "notEquals": "Microsoft.Resources/subscriptions/resourceGroups" },
        { "field": "tags['costCenter']", "exists": false }
      ]
    },
    "then": { "effect": "deny" }
  }
}
"@ | Set-Content .\require-costcenter.json

@"
{
  "mode": "Indexed",
  "parameters": {},
  "policyRule": {
    "if": {
      "allOf": [
        { "field": "type", "notEquals": "Microsoft.Resources/subscriptions/resourceGroups" },
        { "field": "tags['owner']", "exists": false }
      ]
    },
    "then": { "effect": "deny" }
  }
}
"@ | Set-Content .\require-owner.json

New-AzPolicyDefinition -Name require-costcenter-tag -DisplayName "Require costCenter tag" -Policy (Get-Content .\require-costcenter.json -Raw) -Mode Indexed
New-AzPolicyDefinition -Name require-owner-tag -DisplayName "Require owner tag" -Policy (Get-Content .\require-owner.json -Raw) -Mode Indexed

$Initiative = @"
{
  "properties": {
    "displayName": "AZ305 Tagging Baseline",
    "policyType": "Custom",
    "metadata": { "category": "Tags" },
    "policyDefinitions": [
      {
        "policyDefinitionReferenceId": "costcenter",
        "policyDefinitionId": "/subscriptions/$SubscriptionId/providers/Microsoft.Authorization/policyDefinitions/require-costcenter-tag"
      },
      {
        "policyDefinitionReferenceId": "owner",
        "policyDefinitionId": "/subscriptions/$SubscriptionId/providers/Microsoft.Authorization/policyDefinitions/require-owner-tag"
      }
    ]
  }
}
"@
$Initiative | Set-Content .\tag-initiative.json
New-AzPolicySetDefinition -Name az305-tagging-baseline -DisplayName "AZ305 Tagging Baseline" -PolicyDefinition (Get-Content .\tag-initiative.json -Raw)

New-AzPolicyAssignment -Name az305-tagging-assignment -DisplayName "AZ305 Tagging Assignment" -Scope $AssignmentScope -PolicySetDefinition (Get-AzPolicySetDefinition -Name az305-tagging-baseline)

# Test deny - should fail
New-AzStorageAccount -ResourceGroupName $ResourceGroup -Name ("sttagdeny" + (Get-Random)) -Location $Location -SkuName Standard_LRS

# Create compliant resource
New-AzStorageAccount -ResourceGroupName $ResourceGroup -Name ("sttagok" + (Get-Random)) -Location $Location -SkuName Standard_LRS -Tag @{ costCenter = 'CC100'; owner = 'cloudops' }

# Exemption
New-AzPolicyExemption -Name migration-exemption -Scope $AssignmentScope -PolicyAssignment (Get-AzPolicyAssignment -Name az305-tagging-assignment -Scope $AssignmentScope) -DisplayName "Migration wave exemption" -Description "Temporary exemption for migration resources"

# Compliance view
Get-AzPolicyStateSummary -ResourceGroupName $ResourceGroup
```

### IaC implementation
**Bicep starter:**
```bicep
targetScope = 'resourceGroup'

resource assignment 'Microsoft.Authorization/policyAssignments@2022-06-01' = {
  name: 'az305-tagging-assignment'
  properties: {
    displayName: 'AZ305 Tagging Assignment'
    policyDefinitionId: resourceId('Microsoft.Authorization/policySetDefinitions', 'az305-tagging-baseline')
  }
}
```

### Verification steps
- Confirm an untagged resource deployment is denied.
- Confirm a tagged resource deploys successfully.
- Confirm the exemption appears on the assignment scope.
- Review policy compliance in portal or by using `policy state summarize`.

### Cleanup

#### Azure CLI
```bash
az policy exemption delete --name migration-exemption --scope $ASSIGNMENT_SCOPE
az policy assignment delete --name az305-tagging-assignment --scope $ASSIGNMENT_SCOPE
az policy set-definition delete --name az305-tagging-baseline
az policy definition delete --name require-costcenter-tag
az policy definition delete --name require-owner-tag
az group delete --name $RG --yes --no-wait
rm -f require-costcenter.json require-owner.json tag-initiative.json
```

#### PowerShell
```powershell
Remove-AzPolicyExemption -Name migration-exemption -Scope $AssignmentScope
Remove-AzPolicyAssignment -Name az305-tagging-assignment -Scope $AssignmentScope
Remove-AzPolicySetDefinition -Name az305-tagging-baseline -Force
Remove-AzPolicyDefinition -Name require-costcenter-tag -Force
Remove-AzPolicyDefinition -Name require-owner-tag -Force
Remove-AzResourceGroup -Name $ResourceGroup -Force -AsJob
Remove-Item .\require-costcenter.json,.\require-owner.json,.\tag-initiative.json -Force
```

### Exam Tip
**Use exemptions for time-bound exceptions, not to bypass governance permanently.** If a team needs a long-term exception, revisit scope and policy design.

---

## Lab 4: Azure Policy - DeployIfNotExists

### Objective
Automatically deploy diagnostic settings to a Log Analytics workspace by assigning a `DeployIfNotExists` policy with a managed identity and running remediation.

### When to Use This
Use `DeployIfNotExists` when you need resources to be automatically configured for logging, security, or monitoring without relying on every deployment team to remember manual steps.

### Exam domain mapping
- **Primary:** Design identity, governance, and monitoring solutions (25-30%)
- **Secondary:** Design infrastructure solutions (30-35%)

### Key AZ-305 Concepts
- `DeployIfNotExists` effect
- Managed identity on policy assignment
- Remediation tasks
- Diagnostic settings and centralized monitoring

### Prerequisites
- Contributor/Owner on a test subscription or resource group
- `policyinsights` Azure CLI extension
- Log Analytics workspace naming standard

### Architecture and design rationale
Use a central Log Analytics workspace and policy-driven deployment so that:
- Existing resources can be remediated at scale
- New resources become compliant automatically
- Logging becomes consistent across subscriptions

### Implementation steps

#### Azure CLI
```bash
az extension add --name policyinsights

# Variables
SUBSCRIPTION_ID=$(az account show --query id -o tsv)
RG="rg-dine-lab"
LOCATION="eastus"
LAW="law-dine-lab"
KV_NAME="kvdine$RANDOM"
SCOPE="/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RG"

# Create resource group and Log Analytics workspace
az group create --name $RG --location $LOCATION
az monitor log-analytics workspace create --resource-group $RG --workspace-name $LAW --location $LOCATION
LAW_ID=$(az monitor log-analytics workspace show --resource-group $RG --workspace-name $LAW --query id -o tsv)

# Discover a built-in policy for diagnostic settings
POLICY_NAME=$(az policy definition list --query "[?contains(displayName, 'Deploy Diagnostic Settings') && contains(displayName, 'Key Vault') && contains(displayName, 'Log Analytics')].name | [0]" -o tsv)
az policy definition show --name $POLICY_NAME --query properties.parameters -o jsonc

# Create policy assignment with system-assigned managed identity
cat > dine-params.json <<EOF
{
  "logAnalytics": {
    "value": "$LAW_ID"
  }
}
EOF

az policy assignment create \
  --name kv-diag-dine \
  --display-name "Deploy KV diagnostics to Log Analytics" \
  --scope $SCOPE \
  --policy $POLICY_NAME \
  --params @dine-params.json \
  --location $LOCATION \
  --assign-identity \
  --mi-system-assigned \
  --identity-scope $SCOPE

# Grant the assignment identity permission to remediate
ASSIGNMENT_MI=$(az policy assignment show --name kv-diag-dine --scope $SCOPE --query identity.principalId -o tsv)
az role assignment create --assignee-object-id $ASSIGNMENT_MI --assignee-principal-type ServicePrincipal --role Contributor --scope $SCOPE

# Create an existing non-compliant resource, then remediate
az keyvault create --name $KV_NAME --resource-group $RG --location $LOCATION --enable-rbac-authorization true
az policy remediation create --name remediate-kv-diags --policy-assignment kv-diag-dine --scope $SCOPE --resource-discovery-mode ReEvaluateCompliance

# Verify a new resource also gets configured after policy evaluation
KV_NAME2="kvdine2$RANDOM"
az keyvault create --name $KV_NAME2 --resource-group $RG --location $LOCATION --enable-rbac-authorization true
sleep 180
KV2_ID=$(az keyvault show --name $KV_NAME2 --resource-group $RG --query id -o tsv)
az monitor diagnostic-settings list --resource $KV2_ID -o jsonc
```

#### PowerShell
```powershell
# Variables
$SubscriptionId = (Get-AzContext).Subscription.Id
$ResourceGroup = "rg-dine-lab"
$Location = "eastus"
$WorkspaceName = "law-dine-lab"
$VaultName = "kvdine" + (Get-Random)
$Scope = "/subscriptions/$SubscriptionId/resourceGroups/$ResourceGroup"

# Create resource group and Log Analytics workspace
New-AzResourceGroup -Name $ResourceGroup -Location $Location
New-AzOperationalInsightsWorkspace -ResourceGroupName $ResourceGroup -Name $WorkspaceName -Location $Location -Sku PerGB2018
$WorkspaceId = (Get-AzOperationalInsightsWorkspace -ResourceGroupName $ResourceGroup -Name $WorkspaceName).ResourceId

# Discover built-in diagnostic policy
$Policy = Get-AzPolicyDefinition | Where-Object { $_.Properties.DisplayName -like '*Deploy Diagnostic Settings*Key Vault*Log Analytics*' } | Select-Object -First 1
$Policy.Properties.Parameters

# Assign with managed identity
$Assignment = New-AzPolicyAssignment -Name kv-diag-dine -DisplayName "Deploy KV diagnostics to Log Analytics" -Scope $Scope -PolicyDefinition $Policy -PolicyParameterObject @{ logAnalytics = $WorkspaceId } -Location $Location -AssignIdentity

# Grant remediation permissions
New-AzRoleAssignment -ObjectId $Assignment.Identity.PrincipalId -RoleDefinitionName Contributor -Scope $Scope

# Create non-compliant resource and remediate
New-AzKeyVault -VaultName $VaultName -ResourceGroupName $ResourceGroup -Location $Location -EnableRbacAuthorization
Start-AzPolicyRemediation -Name remediate-kv-diags -PolicyAssignmentId $Assignment.ResourceId -Scope $Scope -ResourceDiscoveryMode ReEvaluateCompliance

# Create a second vault and verify diagnostics appear after evaluation
$VaultName2 = "kvdine2" + (Get-Random)
New-AzKeyVault -VaultName $VaultName2 -ResourceGroupName $ResourceGroup -Location $Location -EnableRbacAuthorization
Start-Sleep -Seconds 180
$Vault2Id = (Get-AzKeyVault -VaultName $VaultName2 -ResourceGroupName $ResourceGroup).ResourceId
Get-AzDiagnosticSetting -ResourceId $Vault2Id
```

### IaC implementation
**Bicep starter:**
```bicep
targetScope = 'resourceGroup'

param logAnalyticsId string

resource assignment 'Microsoft.Authorization/policyAssignments@2022-06-01' = {
  name: 'kv-diag-dine'
  location: resourceGroup().location
  identity: {
    type: 'SystemAssigned'
  }
  properties: {
    displayName: 'Deploy KV diagnostics to Log Analytics'
    policyDefinitionId: subscriptionResourceId('Microsoft.Authorization/policyDefinitions', 'built-in-policy-name-here')
    parameters: {
      logAnalytics: {
        value: logAnalyticsId
      }
    }
  }
}
```

### Verification steps
- Confirm the policy assignment has a system-assigned managed identity.
- Confirm the remediation task reaches `Succeeded`.
- Confirm existing and new Key Vaults receive diagnostic settings.
- Confirm logs flow to the Log Analytics workspace.

### Cleanup

#### Azure CLI
```bash
az policy remediation delete --name remediate-kv-diags --resource-group $RG
az role assignment delete --assignee-object-id $ASSIGNMENT_MI --role Contributor --scope $SCOPE
az policy assignment delete --name kv-diag-dine --scope $SCOPE
az group delete --name $RG --yes --no-wait
rm -f dine-params.json
```

#### PowerShell
```powershell
Remove-AzPolicyRemediation -Name remediate-kv-diags -Scope $Scope
Remove-AzRoleAssignment -ObjectId $Assignment.Identity.PrincipalId -RoleDefinitionName Contributor -Scope $Scope
Remove-AzPolicyAssignment -Name kv-diag-dine -Scope $Scope
Remove-AzResourceGroup -Name $ResourceGroup -Force -AsJob
```

### Exam Tip
**`DeployIfNotExists` and `Modify` usually require a managed identity and remediation permissions.** `Deny` blocks immediately, but it does not fix existing resources.

---

## Lab 5: Azure Policy - Allowed Locations & SKUs

### Objective
Control resource sprawl by limiting allowed regions and resource SKUs, first in `Audit` mode and then in `Deny` mode.

### When to Use This
Use this lab when you need residency control, standardization, and cost governance across subscriptions or landing zones.

### Exam domain mapping
- **Primary:** Design identity, governance, and monitoring solutions (25-30%)
- **Secondary:** Design infrastructure solutions (30-35%)

### Key AZ-305 Concepts
- Built-in policy reuse
- Initiative design
- Audit before deny
- Cost and compliance guardrails

### Prerequisites
- Contributor/Owner on subscription or RG
- Policy assignment permissions
- At least one test resource group

### Architecture and design rationale
A strong enterprise pattern is:
- **Allowed locations:** hard compliance boundary
- **Allowed VM SKUs:** cost/performance boundary
- **Audit first:** discover impact before enforcement

### Implementation steps

#### Azure CLI
```bash
# Variables
SUBSCRIPTION_ID=$(az account show --query id -o tsv)
RG="rg-location-sku-lab"
LOCATION="eastus"
SCOPE="/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RG"

az group create --name $RG --location $LOCATION

# Discover built-in policies
ALLOWED_LOCATIONS=$(az policy definition list --query "[?displayName=='Allowed locations'].name | [0]" -o tsv)
ALLOWED_VM_SKUS=$(az policy definition list --query "[?displayName=='Allowed virtual machine size SKUs'].name | [0]" -o tsv)

# Create custom storage SKU policy with effect parameter so you can start in Audit
cat > storage-sku-policy.json <<'EOF'
{
  "mode": "Indexed",
  "parameters": {
    "allowedSkus": {
      "type": "Array"
    },
    "effect": {
      "type": "String",
      "allowedValues": ["Audit", "Deny", "Disabled"],
      "defaultValue": "Audit"
    }
  },
  "policyRule": {
    "if": {
      "allOf": [
        { "field": "type", "equals": "Microsoft.Storage/storageAccounts" },
        { "not": { "field": "Microsoft.Storage/storageAccounts/sku.name", "in": "[parameters('allowedSkus')]" } }
      ]
    },
    "then": {
      "effect": "[parameters('effect')]"
    }
  }
}
EOF
az policy definition create --name allowed-storage-skus-custom --display-name "Allowed storage SKUs (custom)" --mode Indexed --rules @storage-sku-policy.json

# Create initiative
cat > location-sku-initiative.json <<EOF
{
  "properties": {
    "displayName": "AZ305 Location and SKU Guardrails",
    "policyType": "Custom",
    "metadata": { "category": "Governance" },
    "parameters": {
      "allowedLocations": { "type": "Array" },
      "allowedVmSkus": { "type": "Array" },
      "allowedStorageSkus": { "type": "Array" },
      "storageEffect": { "type": "String", "defaultValue": "Audit" }
    },
    "policyDefinitions": [
      {
        "policyDefinitionReferenceId": "allowedLocations",
        "policyDefinitionId": "/providers/Microsoft.Authorization/policyDefinitions/$ALLOWED_LOCATIONS",
        "parameters": {
          "listOfAllowedLocations": { "value": "[parameters('allowedLocations')]" }
        }
      },
      {
        "policyDefinitionReferenceId": "allowedVmSkus",
        "policyDefinitionId": "/providers/Microsoft.Authorization/policyDefinitions/$ALLOWED_VM_SKUS",
        "parameters": {
          "listOfAllowedSKUs": { "value": "[parameters('allowedVmSkus')]" }
        }
      },
      {
        "policyDefinitionReferenceId": "allowedStorageSkus",
        "policyDefinitionId": "/subscriptions/$SUBSCRIPTION_ID/providers/Microsoft.Authorization/policyDefinitions/allowed-storage-skus-custom",
        "parameters": {
          "allowedSkus": { "value": "[parameters('allowedStorageSkus')]" },
          "effect": { "value": "[parameters('storageEffect')]" }
        }
      }
    ]
  }
}
EOF
az policy set-definition create --name az305-location-sku-initiative --definitions @location-sku-initiative.json

# Assign initiative in Audit mode first
cat > initiative-params-audit.json <<'EOF'
{
  "allowedLocations": { "value": ["eastus", "eastus2", "centralus"] },
  "allowedVmSkus": { "value": ["Standard_B1s", "Standard_B2s", "Standard_D2s_v5"] },
  "allowedStorageSkus": { "value": ["Standard_LRS", "Standard_GRS", "Standard_ZRS"] },
  "storageEffect": { "value": "Audit" }
}
EOF

az policy assignment create --name az305-location-sku-assignment --scope $SCOPE --policy-set-definition az305-location-sku-initiative --params @initiative-params-audit.json

# Test in audit mode - may succeed but become non-compliant
az storage account create --name "stpremium$RANDOM" --resource-group $RG --location eastus --sku Premium_LRS
az policy state summarize --resource-group $RG -o jsonc

# Move to deny mode
cat > initiative-params-deny.json <<'EOF'
{
  "allowedLocations": { "value": ["eastus", "eastus2", "centralus"] },
  "allowedVmSkus": { "value": ["Standard_B1s", "Standard_B2s", "Standard_D2s_v5"] },
  "allowedStorageSkus": { "value": ["Standard_LRS", "Standard_GRS", "Standard_ZRS"] },
  "storageEffect": { "value": "Deny" }
}
EOF
az policy assignment update --name az305-location-sku-assignment --scope $SCOPE --params @initiative-params-deny.json

# Test deny for blocked region and blocked SKU
az group create --name rg-blocked-region --location westeurope
# Expected: denied if scoped more broadly than current RG
az storage account create --name "stdeny$RANDOM" --resource-group $RG --location eastus --sku Premium_LRS
# Expected: denied
```

#### PowerShell
```powershell
# Variables
$SubscriptionId = (Get-AzContext).Subscription.Id
$ResourceGroup = "rg-location-sku-lab"
$Location = "eastus"
$Scope = "/subscriptions/$SubscriptionId/resourceGroups/$ResourceGroup"

New-AzResourceGroup -Name $ResourceGroup -Location $Location

# Discover built-in policies
$AllowedLocations = (Get-AzPolicyDefinition | Where-Object { $_.Properties.DisplayName -eq 'Allowed locations' } | Select-Object -First 1).Name
$AllowedVmSkus = (Get-AzPolicyDefinition | Where-Object { $_.Properties.DisplayName -eq 'Allowed virtual machine size SKUs' } | Select-Object -First 1).Name

# Custom storage SKU policy
@"
{
  "mode": "Indexed",
  "parameters": {
    "allowedSkus": { "type": "Array" },
    "effect": { "type": "String", "allowedValues": ["Audit", "Deny", "Disabled"], "defaultValue": "Audit" }
  },
  "policyRule": {
    "if": {
      "allOf": [
        { "field": "type", "equals": "Microsoft.Storage/storageAccounts" },
        { "not": { "field": "Microsoft.Storage/storageAccounts/sku.name", "in": "[parameters('allowedSkus')]" } }
      ]
    },
    "then": { "effect": "[parameters('effect')]" }
  }
}
"@ | Set-Content .\storage-sku-policy.json
New-AzPolicyDefinition -Name allowed-storage-skus-custom -DisplayName "Allowed storage SKUs (custom)" -Policy (Get-Content .\storage-sku-policy.json -Raw) -Mode Indexed

# Initiative definition
@"
{
  "properties": {
    "displayName": "AZ305 Location and SKU Guardrails",
    "policyType": "Custom",
    "metadata": { "category": "Governance" },
    "parameters": {
      "allowedLocations": { "type": "Array" },
      "allowedVmSkus": { "type": "Array" },
      "allowedStorageSkus": { "type": "Array" },
      "storageEffect": { "type": "String", "defaultValue": "Audit" }
    },
    "policyDefinitions": [
      {
        "policyDefinitionReferenceId": "allowedLocations",
        "policyDefinitionId": "/providers/Microsoft.Authorization/policyDefinitions/$AllowedLocations",
        "parameters": {
          "listOfAllowedLocations": { "value": "[parameters('allowedLocations')]" }
        }
      },
      {
        "policyDefinitionReferenceId": "allowedVmSkus",
        "policyDefinitionId": "/providers/Microsoft.Authorization/policyDefinitions/$AllowedVmSkus",
        "parameters": {
          "listOfAllowedSKUs": { "value": "[parameters('allowedVmSkus')]" }
        }
      },
      {
        "policyDefinitionReferenceId": "allowedStorageSkus",
        "policyDefinitionId": "/subscriptions/$SubscriptionId/providers/Microsoft.Authorization/policyDefinitions/allowed-storage-skus-custom",
        "parameters": {
          "allowedSkus": { "value": "[parameters('allowedStorageSkus')]" },
          "effect": { "value": "[parameters('storageEffect')]" }
        }
      }
    ]
  }
}
"@ | Set-Content .\location-sku-initiative.json
New-AzPolicySetDefinition -Name az305-location-sku-initiative -DisplayName "AZ305 Location and SKU Guardrails" -PolicyDefinition (Get-Content .\location-sku-initiative.json -Raw)

# Assign in Audit mode first
New-AzPolicyAssignment -Name az305-location-sku-assignment -Scope $Scope -PolicySetDefinition (Get-AzPolicySetDefinition -Name az305-location-sku-initiative) -PolicyParameterObject @{ allowedLocations = @('eastus','eastus2','centralus'); allowedVmSkus = @('Standard_B1s','Standard_B2s','Standard_D2s_v5'); allowedStorageSkus = @('Standard_LRS','Standard_GRS','Standard_ZRS'); storageEffect = 'Audit' }

# Test audit mode
New-AzStorageAccount -ResourceGroupName $ResourceGroup -Name ("stpremium" + (Get-Random)) -Location $Location -SkuName Premium_LRS
Get-AzPolicyStateSummary -ResourceGroupName $ResourceGroup

# Update to Deny
Set-AzPolicyAssignment -Name az305-location-sku-assignment -Scope $Scope -PolicyParameterObject @{ allowedLocations = @('eastus','eastus2','centralus'); allowedVmSkus = @('Standard_B1s','Standard_B2s','Standard_D2s_v5'); allowedStorageSkus = @('Standard_LRS','Standard_GRS','Standard_ZRS'); storageEffect = 'Deny' }

# Test deny
New-AzStorageAccount -ResourceGroupName $ResourceGroup -Name ("stdeny" + (Get-Random)) -Location $Location -SkuName Premium_LRS
# Expected: denied
```

### IaC implementation
**Terraform starter:**
```hcl
resource "azurerm_policy_set_definition" "location_sku" {
  name         = "az305-location-sku-initiative"
  display_name = "AZ305 Location and SKU Guardrails"
  policy_type  = "Custom"
}
```

### Verification steps
- Confirm audit mode records non-compliance without blocking deployment.
- Confirm deny mode blocks the premium storage deployment.
- Confirm initiative shows combined compliance posture.
- Review impact before broadening assignment scope.

### Cleanup

#### Azure CLI
```bash
az policy assignment delete --name az305-location-sku-assignment --scope $SCOPE
az policy set-definition delete --name az305-location-sku-initiative
az policy definition delete --name allowed-storage-skus-custom
az group delete --name $RG --yes --no-wait
rm -f storage-sku-policy.json location-sku-initiative.json initiative-params-audit.json initiative-params-deny.json
```

#### PowerShell
```powershell
Remove-AzPolicyAssignment -Name az305-location-sku-assignment -Scope $Scope
Remove-AzPolicySetDefinition -Name az305-location-sku-initiative -Force
Remove-AzPolicyDefinition -Name allowed-storage-skus-custom -Force
Remove-AzResourceGroup -Name $ResourceGroup -Force -AsJob
Remove-Item .\storage-sku-policy.json,.\location-sku-initiative.json -Force
```

### Exam Tip
**Start with `Audit` when business impact is uncertain, then move to `Deny` after you understand exceptions.** This is a classic AZ-305 governance design tradeoff.

---

## Lab 6: Resource Locks

### Objective
Apply `CanNotDelete` and `ReadOnly` locks, validate behavior, then compare locks to Azure Policy.

### When to Use This
Use resource locks when accidental changes are the primary risk and you need a simple control that applies even to privileged users.

### Exam domain mapping
- **Primary:** Design identity, governance, and monitoring solutions (25-30%)
- **Secondary:** Design business continuity solutions (15-20%)

### Key AZ-305 Concepts
- `CanNotDelete` vs. `ReadOnly`
- Inheritance from resource group scope
- Lock behavior for management plane vs. data plane
- Locks vs. Policy decision points

### Prerequisites
- Owner or Contributor on a test subscription
- Storage account and resource group for testing

### Architecture and design rationale
| Control | Best use case | Limitation |
|---|---|---|
| `CanNotDelete` lock | Prevent accidental deletions | Resource can still be modified |
| `ReadOnly` lock | Freeze critical config | Can block automation and updates |
| Azure Policy | Enforce standards | Does not directly protect existing objects from delete |

### Implementation steps

#### Azure CLI
```bash
RG="rg-lock-lab"
LOCATION="eastus"
SA="stlock$RANDOM"

az group create --name $RG --location $LOCATION
az storage account create --name $SA --resource-group $RG --location $LOCATION --sku Standard_LRS

# Apply CanNotDelete lock to resource group
az lock create --name rg-cannotdelete --lock-type CanNotDelete --resource-group $RG --notes "Protect resource group from accidental deletion"

# Apply ReadOnly lock to storage account
az lock create \
  --name sa-readonly \
  --lock-type ReadOnly \
  --resource-group $RG \
  --resource-name $SA \
  --resource-type Microsoft.Storage/storageAccounts \
  --notes "Protect storage account settings"

# Test delete at RG scope - should fail
az group delete --name $RG --yes --no-wait

# Test modify on storage account - should fail
az storage account update --name $SA --resource-group $RG --https-only false

# List locks
az lock list --resource-group $RG -o table
```

#### PowerShell
```powershell
$ResourceGroup = "rg-lock-lab"
$Location = "eastus"
$StorageName = "stlock" + (Get-Random)

New-AzResourceGroup -Name $ResourceGroup -Location $Location
New-AzStorageAccount -ResourceGroupName $ResourceGroup -Name $StorageName -Location $Location -SkuName Standard_LRS

# Apply locks
New-AzResourceLock -LockName rg-cannotdelete -LockLevel CanNotDelete -ResourceGroupName $ResourceGroup -LockNotes "Protect resource group from accidental deletion"
$StorageId = (Get-AzStorageAccount -ResourceGroupName $ResourceGroup -Name $StorageName).Id
New-AzResourceLock -LockName sa-readonly -LockLevel ReadOnly -ResourceId $StorageId -LockNotes "Protect storage account settings"

# Test delete - should fail
Remove-AzResourceGroup -Name $ResourceGroup -Force

# Test modify - should fail
Set-AzStorageAccount -ResourceGroupName $ResourceGroup -Name $StorageName -EnableHttpsTrafficOnly $false

# List locks
Get-AzResourceLock -ResourceGroupName $ResourceGroup
```

### IaC implementation
**Bicep starter:**
```bicep
targetScope = 'resourceGroup'

resource rgLock 'Microsoft.Authorization/locks@2020-05-01' = {
  name: 'rg-cannotdelete'
  properties: {
    level: 'CanNotDelete'
    notes: 'Protect resource group from accidental deletion'
  }
}
```

### Verification steps
- Confirm RG delete is blocked by `CanNotDelete`.
- Confirm storage account update is blocked by `ReadOnly`.
- Confirm locks appear in portal and command output.
- Compare whether the scenario is better handled by lock, policy, or both.

### Cleanup

#### Azure CLI
```bash
az lock delete --name sa-readonly --resource-group $RG --resource-name $SA --resource-type Microsoft.Storage/storageAccounts
az lock delete --name rg-cannotdelete --resource-group $RG
az group delete --name $RG --yes --no-wait
```

#### PowerShell
```powershell
Remove-AzResourceLock -LockName sa-readonly -ResourceId $StorageId -Force
Remove-AzResourceLock -LockName rg-cannotdelete -ResourceGroupName $ResourceGroup -Force
Remove-AzResourceGroup -Name $ResourceGroup -Force -AsJob
```

### Exam Tip
**Locks protect against accidental change; Policy governs desired state.** If the question asks about preventing deletion of a critical resource immediately, a lock is usually the best answer.

---

## Lab 7: Tag Governance Strategy

### Objective
Design a tag schema, inherit tags from the resource group, require tags on resources, use `Modify`, and query compliance with Resource Graph.

### When to Use This
Use tag governance for cost allocation, ownership, lifecycle management, and operational filtering across large environments.

### Exam domain mapping
- **Primary:** Design identity, governance, and monitoring solutions (25-30%)
- **Secondary:** Design data storage solutions (20-25%)

### Key AZ-305 Concepts
- Standard tag schema design
- Built-in `Modify` and inherit tag policies
- Resource Graph queries for governance reporting
- Tag inheritance vs. required-tag denial

### Prerequisites
- Test resource group
- Resource Graph access
- Contributor/Owner to assign policy

### Architecture and design rationale
Recommended schema for AZ-305 scenarios:

| Tag | Example | Why it matters |
|---|---|---|
| `costCenter` | `FIN-001` | Chargeback/showback |
| `owner` | `cloudops@contoso.com` | Operational accountability |
| `environment` | `prod` | Lifecycle and risk segmentation |
| `application` | `erp-api` | Workload grouping |

Design rule: require tags at deployment time, then use `Modify`/inherit policies to reduce manual drift.

### Implementation steps

#### Azure CLI
```bash
SUBSCRIPTION_ID=$(az account show --query id -o tsv)
RG="rg-tag-strategy-lab"
LOCATION="eastus"
SCOPE="/subscriptions/$SUBSCRIPTION_ID/resourceGroups/$RG"

# Create tagged resource group
az group create --name $RG --location $LOCATION --tags costCenter=FIN-001 owner=cloudops@contoso.com environment=prod application=erp-api

# Discover built-in inherit policy
INHERIT_POLICY=$(az policy definition list --query "[?displayName=='Inherit a tag from the resource group if missing'].name | [0]" -o tsv)

# Assign inherit policy for application tag
cat > inherit-application-params.json <<'EOF'
{
  "tagName": { "value": "application" }
}
EOF
az policy assignment create --name inherit-application-tag --scope $SCOPE --policy $INHERIT_POLICY --params @inherit-application-params.json --location $LOCATION --assign-identity --mi-system-assigned --identity-scope $SCOPE

# Grant modify permission to the assignment identity
INHERIT_MI=$(az policy assignment show --name inherit-application-tag --scope $SCOPE --query identity.principalId -o tsv)
az role assignment create --assignee-object-id $INHERIT_MI --assignee-principal-type ServicePrincipal --role Contributor --scope $SCOPE

# Create a custom policy to require costCenter, owner, and environment tags
cat > require-core-tags.json <<'EOF'
{
  "mode": "Indexed",
  "parameters": {},
  "policyRule": {
    "if": {
      "allOf": [
        { "field": "tags['costCenter']", "exists": false },
        { "field": "type", "notEquals": "Microsoft.Resources/subscriptions/resourceGroups" }
      ]
    },
    "then": { "effect": "deny" }
  }
}
EOF
az policy definition create --name require-core-costcenter --display-name "Require costCenter tag" --mode Indexed --rules @require-core-tags.json
az policy assignment create --name require-core-costcenter --scope $SCOPE --policy require-core-costcenter

# Deploy a resource without application tag - policy should inherit it from RG
SA="sttaggov$RANDOM"
az storage account create --name $SA --resource-group $RG --location $LOCATION --sku Standard_LRS --tags costCenter=FIN-001 owner=cloudops@contoso.com environment=prod

# Query by tag with Resource Graph
az graph query -q "Resources | where tags.environment =~ 'prod' | project name, type, resourceGroup, application=tostring(tags.application), owner=tostring(tags.owner)" -o table
```

#### PowerShell
```powershell
$SubscriptionId = (Get-AzContext).Subscription.Id
$ResourceGroup = "rg-tag-strategy-lab"
$Location = "eastus"
$Scope = "/subscriptions/$SubscriptionId/resourceGroups/$ResourceGroup"

# Create tagged resource group
New-AzResourceGroup -Name $ResourceGroup -Location $Location -Tag @{ costCenter = 'FIN-001'; owner = 'cloudops@contoso.com'; environment = 'prod'; application = 'erp-api' }

# Discover built-in inherit policy
$InheritPolicy = Get-AzPolicyDefinition | Where-Object { $_.Properties.DisplayName -eq 'Inherit a tag from the resource group if missing' } | Select-Object -First 1

# Assign inherit policy
$Assignment = New-AzPolicyAssignment -Name inherit-application-tag -Scope $Scope -PolicyDefinition $InheritPolicy -PolicyParameterObject @{ tagName = 'application' } -Location $Location -AssignIdentity
New-AzRoleAssignment -ObjectId $Assignment.Identity.PrincipalId -RoleDefinitionName Contributor -Scope $Scope

# Create a simple required tag policy
@"
{
  "mode": "Indexed",
  "parameters": {},
  "policyRule": {
    "if": {
      "allOf": [
        { "field": "tags['costCenter']", "exists": false },
        { "field": "type", "notEquals": "Microsoft.Resources/subscriptions/resourceGroups" }
      ]
    },
    "then": { "effect": "deny" }
  }
}
"@ | Set-Content .\require-core-tags.json
New-AzPolicyDefinition -Name require-core-costcenter -DisplayName "Require costCenter tag" -Policy (Get-Content .\require-core-tags.json -Raw) -Mode Indexed
New-AzPolicyAssignment -Name require-core-costcenter -Scope $Scope -PolicyDefinition (Get-AzPolicyDefinition -Name require-core-costcenter)

# Deploy resource without application tag - policy should inherit it
$StorageName = "sttaggov" + (Get-Random)
New-AzStorageAccount -ResourceGroupName $ResourceGroup -Name $StorageName -Location $Location -SkuName Standard_LRS -Tag @{ costCenter = 'FIN-001'; owner = 'cloudops@contoso.com'; environment = 'prod' }

# Query by tag with Resource Graph
Search-AzGraph -Query "Resources | where tags.environment =~ 'prod' | project name, type, resourceGroup, application=tostring(tags.application), owner=tostring(tags.owner)"
```

### IaC implementation
**Terraform starter:**
```hcl
resource "azurerm_resource_group" "tagged" {
  name     = "rg-tag-strategy-lab"
  location = "eastus"
  tags = {
    costCenter  = "FIN-001"
    owner       = "cloudops@contoso.com"
    environment = "prod"
    application = "erp-api"
  }
}
```

### Verification steps
- Confirm resources inherit the `application` tag from the RG.
- Confirm required-tag policy blocks untagged resources.
- Confirm Resource Graph returns tagged resources.
- Review how tag inheritance improves cost and operations reporting.

### Cleanup

#### Azure CLI
```bash
az role assignment delete --assignee-object-id $INHERIT_MI --role Contributor --scope $SCOPE
az policy assignment delete --name inherit-application-tag --scope $SCOPE
az policy assignment delete --name require-core-costcenter --scope $SCOPE
az policy definition delete --name require-core-costcenter
az group delete --name $RG --yes --no-wait
rm -f inherit-application-params.json require-core-tags.json
```

#### PowerShell
```powershell
Remove-AzRoleAssignment -ObjectId $Assignment.Identity.PrincipalId -RoleDefinitionName Contributor -Scope $Scope
Remove-AzPolicyAssignment -Name inherit-application-tag -Scope $Scope
Remove-AzPolicyAssignment -Name require-core-costcenter -Scope $Scope
Remove-AzPolicyDefinition -Name require-core-costcenter -Force
Remove-AzResourceGroup -Name $ResourceGroup -Force -AsJob
Remove-Item .\require-core-tags.json -Force
```

### Exam Tip
**Tags are only useful if they are standardized and queryable.** For AZ-305, governance questions often test whether tagging is being used for cost control, operations, or both.

---

## Lab 8: Azure Blueprints (Legacy) & Deployment Stacks

### Objective
Create a legacy Azure Blueprint with policy/RBAC/template artifacts, publish and assign it, then compare it with the modern deployment stack approach.

### When to Use This
Use this lab to understand legacy estate support and to compare it with the recommended modern replacement for repeatable, governed environment deployment.

### Exam domain mapping
- **Primary:** Design identity, governance, and monitoring solutions (25-30%)
- **Secondary:** Design infrastructure solutions (30-35%)

### Key AZ-305 Concepts
- Azure Blueprints is **legacy/deprecated**
- Blueprint artifacts: policy, RBAC, ARM/Bicep template
- Deployment stacks and deny settings
- Repeatable governed deployments

### Prerequisites
- Subscription-level Owner
- Tenant where Blueprint APIs are still enabled
- Azure CLI and Azure PowerShell with `az rest` / `Invoke-AzRestMethod`
- A test resource group for deployment stack validation

### Architecture and design rationale
| Option | Best fit | Design note |
|---|---|---|
| Azure Blueprints (legacy) | Existing tenants already using Blueprints | Plan migration away before retirement |
| Deployment Stacks | New governed deployments | Modern approach with deny settings and lifecycle control |

### Implementation steps

#### Azure CLI
```bash
# Variables
SUBSCRIPTION_ID=$(az account show --query id -o tsv)
RG="rg-stack-lab"
LOCATION="eastus"
BLUEPRINT_NAME="bp-governed-rg"
VERSION="v1"
ASSIGNMENT_NAME="bp-governed-rg-assignment"
STACK_NAME="gov-stack"

# Create local Bicep template for the modern deployment stack example
cat > stack.bicep <<'EOF'
param location string = resourceGroup().location
param storageName string

resource sa 'Microsoft.Storage/storageAccounts@2023-05-01' = {
  name: storageName
  location: location
  sku: {
    name: 'Standard_LRS'
  }
  kind: 'StorageV2'
  tags: {
    environment: 'governed'
  }
}
EOF

az group create --name $RG --location $LOCATION

# --- Blueprint (legacy) via REST ---
# Create a blueprint definition shell
az rest --method PUT \
  --url "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/providers/Microsoft.Blueprint/blueprints/$BLUEPRINT_NAME?api-version=2018-11-01-preview" \
  --body '{"properties":{"description":"Legacy governed environment blueprint","targetScope":"subscription","parameters":{},"resourceGroups":{}}}'

# Add a policy artifact (allowed locations example)
az rest --method PUT \
  --url "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/providers/Microsoft.Blueprint/blueprints/$BLUEPRINT_NAME/artifacts/allowedLocations?api-version=2018-11-01-preview" \
  --body '{"kind":"policyAssignment","properties":{"displayName":"Allowed Locations","policyDefinitionId":"/providers/Microsoft.Authorization/policyDefinitions/e56962a6-4747-49cd-b67b-bf8b01975c4c","parameters":{"listOfAllowedLocations":{"value":["eastus","eastus2"]}}}}'

# Add a role assignment artifact
az rest --method PUT \
  --url "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/providers/Microsoft.Blueprint/blueprints/$BLUEPRINT_NAME/artifacts/readerAssignment?api-version=2018-11-01-preview" \
  --body '{"kind":"roleAssignment","properties":{"displayName":"Reader assignment","roleDefinitionId":"acdd72a7-3385-48ef-bd42-f606fba81ae7","principalIds":["<entra-group-object-id>"]}}'

# Add an ARM template artifact for a governed resource group
az rest --method PUT \
  --url "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/providers/Microsoft.Blueprint/blueprints/$BLUEPRINT_NAME/artifacts/storageTemplate?api-version=2018-11-01-preview" \
  --body '{"kind":"template","properties":{"displayName":"Storage template artifact","template":{"$schema":"https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#","contentVersion":"1.0.0.0","parameters":{"storageAccountName":{"type":"string"},"location":{"type":"string"}},"resources":[{"type":"Microsoft.Storage/storageAccounts","apiVersion":"2023-05-01","name":"[parameters('"'"'storageAccountName'"'"')]","location":"[parameters('"'"'location'"'"')]","sku":{"name":"Standard_LRS"},"kind":"StorageV2"}]},"parameters":{"storageAccountName":{"value":"stbp$RANDOM"},"location":{"value":"$LOCATION"}}}}'

# Publish version
az rest --method PUT \
  --url "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/providers/Microsoft.Blueprint/blueprints/$BLUEPRINT_NAME/versions/$VERSION?api-version=2018-11-01-preview" \
  --body '{}'

# Assign blueprint with delete protection on managed resources
az rest --method PUT \
  --url "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/providers/Microsoft.Blueprint/blueprintAssignments/$ASSIGNMENT_NAME?api-version=2018-11-01-preview" \
  --body "{\"identity\":{\"type\":\"SystemAssigned\"},\"properties\":{\"blueprintId\":\"/subscriptions/$SUBSCRIPTION_ID/providers/Microsoft.Blueprint/blueprints/$BLUEPRINT_NAME/versions/$VERSION\",\"locks\":{\"mode\":\"AllResourcesDoNotDelete\"},\"parameters\":{},\"resourceGroups\":{}}}"

# Update blueprint by publishing a new version
VERSION="v2"
az rest --method PUT \
  --url "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/providers/Microsoft.Blueprint/blueprints/$BLUEPRINT_NAME/versions/$VERSION?api-version=2018-11-01-preview" \
  --body '{}'

# --- Deployment Stacks (modern) ---
az stack group create \
  --resource-group $RG \
  --name $STACK_NAME \
  --template-file stack.bicep \
  --parameters storageName="ststack$RANDOM" \
  --deny-settings-mode DenyDelete

az stack group show --resource-group $RG --name $STACK_NAME -o jsonc
```

#### PowerShell
```powershell
$SubscriptionId = (Get-AzContext).Subscription.Id
$ResourceGroup = "rg-stack-lab"
$Location = "eastus"
$BlueprintName = "bp-governed-rg"
$Version = "v1"
$AssignmentName = "bp-governed-rg-assignment"
$StackName = "gov-stack"

@"
param location string = resourceGroup().location
param storageName string

resource sa 'Microsoft.Storage/storageAccounts@2023-05-01' = {
  name: storageName
  location: location
  sku: {
    name: 'Standard_LRS'
  }
  kind: 'StorageV2'
  tags: {
    environment: 'governed'
  }
}
"@ | Set-Content .\stack.bicep

New-AzResourceGroup -Name $ResourceGroup -Location $Location

# Blueprint create
$BlueprintUri = "https://management.azure.com/subscriptions/$SubscriptionId/providers/Microsoft.Blueprint/blueprints/$BlueprintName?api-version=2018-11-01-preview"
$BlueprintBody = @{ properties = @{ description = 'Legacy governed environment blueprint'; targetScope = 'subscription'; parameters = @{}; resourceGroups = @{} } } | ConvertTo-Json -Depth 10
Invoke-AzRestMethod -Method PUT -Uri $BlueprintUri -Body $BlueprintBody -ContentType 'application/json'

# Policy artifact
$PolicyArtifactUri = "https://management.azure.com/subscriptions/$SubscriptionId/providers/Microsoft.Blueprint/blueprints/$BlueprintName/artifacts/allowedLocations?api-version=2018-11-01-preview"
$PolicyArtifactBody = @{ kind = 'policyAssignment'; properties = @{ displayName = 'Allowed Locations'; policyDefinitionId = '/providers/Microsoft.Authorization/policyDefinitions/e56962a6-4747-49cd-b67b-bf8b01975c4c'; parameters = @{ listOfAllowedLocations = @{ value = @('eastus','eastus2') } } } } | ConvertTo-Json -Depth 20
Invoke-AzRestMethod -Method PUT -Uri $PolicyArtifactUri -Body $PolicyArtifactBody -ContentType 'application/json'

# ARM template artifact
$TemplateArtifactUri = "https://management.azure.com/subscriptions/$SubscriptionId/providers/Microsoft.Blueprint/blueprints/$BlueprintName/artifacts/storageTemplate?api-version=2018-11-01-preview"
$TemplateArtifactBody = @{
  kind = 'template'
  properties = @{
    displayName = 'Storage template artifact'
    template = @{
      '$schema' = 'https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#'
      contentVersion = '1.0.0.0'
      parameters = @{
        storageAccountName = @{ type = 'string' }
        location = @{ type = 'string' }
      }
      resources = @(
        @{
          type = 'Microsoft.Storage/storageAccounts'
          apiVersion = '2023-05-01'
          name = "[parameters('storageAccountName')]"
          location = "[parameters('location')]"
          sku = @{ name = 'Standard_LRS' }
          kind = 'StorageV2'
        }
      )
    }
    parameters = @{
      storageAccountName = @{ value = ('stbp' + (Get-Random)) }
      location = @{ value = $Location }
    }
  }
} | ConvertTo-Json -Depth 30
Invoke-AzRestMethod -Method PUT -Uri $TemplateArtifactUri -Body $TemplateArtifactBody -ContentType 'application/json'

# Publish version
$PublishUri = "https://management.azure.com/subscriptions/$SubscriptionId/providers/Microsoft.Blueprint/blueprints/$BlueprintName/versions/$Version?api-version=2018-11-01-preview"
Invoke-AzRestMethod -Method PUT -Uri $PublishUri -Body '{}' -ContentType 'application/json'

# Assign blueprint
$AssignUri = "https://management.azure.com/subscriptions/$SubscriptionId/providers/Microsoft.Blueprint/blueprintAssignments/$AssignmentName?api-version=2018-11-01-preview"
$AssignBody = @{ identity = @{ type = 'SystemAssigned' }; properties = @{ blueprintId = "/subscriptions/$SubscriptionId/providers/Microsoft.Blueprint/blueprints/$BlueprintName/versions/$Version"; locks = @{ mode = 'AllResourcesDoNotDelete' }; parameters = @{}; resourceGroups = @{} } } | ConvertTo-Json -Depth 20
Invoke-AzRestMethod -Method PUT -Uri $AssignUri -Body $AssignBody -ContentType 'application/json'

# Publish v2
$Version = 'v2'
$PublishUri = "https://management.azure.com/subscriptions/$SubscriptionId/providers/Microsoft.Blueprint/blueprints/$BlueprintName/versions/$Version?api-version=2018-11-01-preview"
Invoke-AzRestMethod -Method PUT -Uri $PublishUri -Body '{}' -ContentType 'application/json'

# Deployment stack (modern)
New-AzResourceGroupDeploymentStack -ResourceGroupName $ResourceGroup -Name $StackName -TemplateFile .\stack.bicep -DenySettingsMode DenyDelete -TemplateParameterObject @{ storageName = ('ststack' + (Get-Random)) }
Get-AzResourceGroupDeploymentStack -ResourceGroupName $ResourceGroup -Name $StackName
```

### IaC implementation
**Modern recommendation:** keep Bicep/Terraform as the source of truth and use deployment stacks for lifecycle/deny settings rather than creating new Blueprint estates.

### Verification steps
- Confirm the Blueprint definition, published version, and assignment exist.
- Confirm assignment lock mode is set.
- Confirm the deployment stack created the storage account and deny settings are active.
- Compare operational effort and lifecycle behavior between the two approaches.

### Cleanup

#### Azure CLI
```bash
az stack group delete --resource-group $RG --name $STACK_NAME --delete-resources yes
az rest --method DELETE --url "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/providers/Microsoft.Blueprint/blueprintAssignments/$ASSIGNMENT_NAME?api-version=2018-11-01-preview"
az rest --method DELETE --url "https://management.azure.com/subscriptions/$SUBSCRIPTION_ID/providers/Microsoft.Blueprint/blueprints/$BLUEPRINT_NAME?api-version=2018-11-01-preview"
az group delete --name $RG --yes --no-wait
rm -f stack.bicep
```

#### PowerShell
```powershell
Remove-AzResourceGroupDeploymentStack -ResourceGroupName $ResourceGroup -Name $StackName -DeleteResources
Invoke-AzRestMethod -Method DELETE -Uri "https://management.azure.com/subscriptions/$SubscriptionId/providers/Microsoft.Blueprint/blueprintAssignments/$AssignmentName?api-version=2018-11-01-preview"
Invoke-AzRestMethod -Method DELETE -Uri "https://management.azure.com/subscriptions/$SubscriptionId/providers/Microsoft.Blueprint/blueprints/$BlueprintName?api-version=2018-11-01-preview"
Remove-AzResourceGroup -Name $ResourceGroup -Force -AsJob
Remove-Item .\stack.bicep -Force
```

### Exam Tip
**For new designs, prefer Policy + IaC + Deployment Stacks over Azure Blueprints.** Blueprints may still appear in exam questions as a legacy option, so know both the concept and the migration direction.

---

## Lab 9: Compliance Assessment

### Objective
Enable Microsoft Defender for Cloud plans, review regulatory compliance, export findings, and create a custom compliance baseline.

### When to Use This
Use this lab when your organization must measure compliance against frameworks and continuously track drift.

### Exam domain mapping
- **Primary:** Design identity, governance, and monitoring solutions (25-30%)
- **Secondary:** Design business continuity solutions (15-20%)

### Key AZ-305 Concepts
- Defender for Cloud plans
- Regulatory compliance dashboard
- Policy-based compliance controls
- Reporting and export strategy

### Prerequisites
- Owner or Security Admin permissions
- `security` Azure CLI extension
- Az.Security PowerShell module
- Test subscription or resource group

### Architecture and design rationale
Compliance assessment is strongest when you combine:
- Defender for Cloud for security/regulatory visibility
- Azure Policy for enforceable standards
- Export/reporting for auditors and platform teams

### Implementation steps

#### Azure CLI
```bash
az extension add --name security

SUBSCRIPTION_ID=$(az account show --query id -o tsv)
RG="rg-compliance-lab"
LOCATION="eastus"
STANDARD_NAME="Azure-CIS-1.3.0"

az group create --name $RG --location $LOCATION

# Enable Defender plans
az security pricing create --name VirtualMachines --tier Standard
az security pricing create --name StorageAccounts --tier Standard

# Review current pricing
az security pricing list -o table

# Review regulatory compliance standards
az security regulatory-compliance-standards list --resource-group $RG --subscription $SUBSCRIPTION_ID -o table

# Review controls and assessments for a standard
az security regulatory-compliance-controls list --standard-name $STANDARD_NAME --resource-group $RG --subscription $SUBSCRIPTION_ID -o table
az security regulatory-compliance-assessments list --standard-name $STANDARD_NAME --control-name "1.1" --resource-group $RG --subscription $SUBSCRIPTION_ID -o json > compliance-assessment.json

# Export a policy-centric compliance summary
az policy state summarize --resource-group $RG -o json > policy-compliance-summary.json

# Create a custom compliance initiative for internal standards
REQUIRE_TAG_POLICY=$(az policy definition list --query "[?displayName=='Require a tag on resources'].id | [0]" -o tsv)
HTTPS_ONLY_POLICY=$(az policy definition list --query "[?displayName=='Secure transfer to storage accounts should be enabled'].id | [0]" -o tsv)
cat > org-compliance-initiative.json <<EOF
{
  "properties": {
    "displayName": "Org Custom Compliance Standard",
    "policyType": "Custom",
    "metadata": {
      "category": "Regulatory Compliance"
    },
    "parameters": {
      "requiredTagName": {
        "type": "String",
        "defaultValue": "environment"
      }
    },
    "policyDefinitions": [
      {
        "policyDefinitionReferenceId": "requireEnvironmentTag",
        "policyDefinitionId": "$REQUIRE_TAG_POLICY",
        "parameters": {
          "tagName": {
            "value": "[parameters('requiredTagName')]"
          }
        }
      },
      {
        "policyDefinitionReferenceId": "secureTransferEnabled",
        "policyDefinitionId": "$HTTPS_ONLY_POLICY"
      }
    ]
  }
}
EOF
az policy set-definition create --name org-custom-compliance --definitions @org-compliance-initiative.json
```

#### PowerShell
```powershell
$SubscriptionId = (Get-AzContext).Subscription.Id
$ResourceGroup = "rg-compliance-lab"
$Location = "eastus"
$StandardName = "Azure-CIS-1.3.0"

New-AzResourceGroup -Name $ResourceGroup -Location $Location

# Enable Defender plans
Set-AzSecurityPricing -Name VirtualMachines -PricingTier Standard
Set-AzSecurityPricing -Name StorageAccounts -PricingTier Standard

# Review plans
Get-AzSecurityPricing

# Review compliance standards, controls, and assessments
Get-AzRegulatoryComplianceStandard -ResourceGroupName $ResourceGroup -SubscriptionId $SubscriptionId
Get-AzRegulatoryComplianceControl -StandardName $StandardName -ResourceGroupName $ResourceGroup -SubscriptionId $SubscriptionId
Get-AzRegulatoryComplianceAssessment -StandardName $StandardName -ControlName "1.1" -ResourceGroupName $ResourceGroup -SubscriptionId $SubscriptionId | ConvertTo-Json -Depth 20 | Set-Content .\compliance-assessment.json

# Export policy compliance summary
Get-AzPolicyStateSummary -ResourceGroupName $ResourceGroup | ConvertTo-Json -Depth 20 | Set-Content .\policy-compliance-summary.json

# Create custom compliance initiative
$RequireTagPolicyId = (Get-AzPolicyDefinition | Where-Object { $_.Properties.DisplayName -eq 'Require a tag on resources' } | Select-Object -First 1).PolicyDefinitionId
$HttpsOnlyPolicyId = (Get-AzPolicyDefinition | Where-Object { $_.Properties.DisplayName -eq 'Secure transfer to storage accounts should be enabled' } | Select-Object -First 1).PolicyDefinitionId
@"
{
  "properties": {
    "displayName": "Org Custom Compliance Standard",
    "policyType": "Custom",
    "metadata": {
      "category": "Regulatory Compliance"
    },
    "parameters": {
      "requiredTagName": {
        "type": "String",
        "defaultValue": "environment"
      }
    },
    "policyDefinitions": [
      {
        "policyDefinitionReferenceId": "requireEnvironmentTag",
        "policyDefinitionId": "$RequireTagPolicyId",
        "parameters": {
          "tagName": {
            "value": "[parameters('requiredTagName')]"
          }
        }
      },
      {
        "policyDefinitionReferenceId": "secureTransferEnabled",
        "policyDefinitionId": "$HttpsOnlyPolicyId"
      }
    ]
  }
}
"@ | Set-Content .\org-compliance-initiative.json
New-AzPolicySetDefinition -Name org-custom-compliance -DisplayName "Org Custom Compliance Standard" -PolicyDefinition (Get-Content .\org-compliance-initiative.json -Raw)
```

### IaC implementation
**Bicep starter:**
```bicep
targetScope = 'subscription'

resource initiative 'Microsoft.Authorization/policySetDefinitions@2021-06-01' = {
  name: 'org-custom-compliance'
  properties: {
    displayName: 'Org Custom Compliance Standard'
    policyType: 'Custom'
    metadata: {
      category: 'Regulatory Compliance'
    }
    policyDefinitions: []
  }
}
```

### Verification steps
- Confirm Defender plans show `Standard`.
- Confirm the selected regulatory standard returns controls and assessments.
- Confirm JSON exports are created for reporting.
- Confirm the custom initiative appears under policy set definitions.

### Cleanup

#### Azure CLI
```bash
az policy set-definition delete --name org-custom-compliance
az group delete --name $RG --yes --no-wait
rm -f compliance-assessment.json policy-compliance-summary.json org-compliance-initiative.json
# Optional: disable Defender plans if this is a temporary lab only
# az security pricing create --name VirtualMachines --tier Free
# az security pricing create --name StorageAccounts --tier Free
```

#### PowerShell
```powershell
Remove-AzPolicySetDefinition -Name org-custom-compliance -Force
Remove-AzResourceGroup -Name $ResourceGroup -Force -AsJob
Remove-Item .\compliance-assessment.json,.\policy-compliance-summary.json,.\org-compliance-initiative.json -Force
# Optional: Set-AzSecurityPricing -Name VirtualMachines -PricingTier Free
# Optional: Set-AzSecurityPricing -Name StorageAccounts -PricingTier Free
```

### Exam Tip
**Defender for Cloud helps assess and visualize compliance, but Azure Policy is still the enforcement engine underneath many controls.** Expect questions that separate visibility from enforcement.

---

## Lab 10: Cost Governance

### Objective
Create a budget with alerts, review allocation and anomaly options, inspect Azure Advisor cost recommendations, and validate spending-limit considerations.

### When to Use This
Use cost governance when you need proactive budget controls, better chargeback, and early detection of waste or unexpected spend.

### Exam domain mapping
- **Primary:** Design identity, governance, and monitoring solutions (25-30%)
- **Secondary:** Design data storage solutions (20-25%)

### Key AZ-305 Concepts
- Budgets and alert thresholds
- Cost allocation rules
- Anomaly detection
- Azure Advisor cost recommendations
- Spending limit eligibility

### Prerequisites
- Cost Management permissions
- Billing or subscription scope access
- Test email address for budget notifications

### Architecture and design rationale
A practical cost governance baseline includes:
- Budget thresholds at subscription or RG scope
- Allocation rules for shared services or platform costs
- Advisor for right-sizing and idle resource findings
- Anomaly detection for unexpected spikes

### Implementation steps

#### Azure CLI
```bash
SUBSCRIPTION_ID=$(az account show --query id -o tsv)
RG="rg-cost-governance-lab"
LOCATION="eastus"
BUDGET_NAME="az305-monthly-budget"
CONTACT_EMAIL="alerts@contoso.com"
START_DATE=$(date +%Y-%m-01)
END_DATE=$(date -d "+12 months" +%Y-%m-01)

az group create --name $RG --location $LOCATION

# Create a budget with alerts
az consumption budget create \
  --name $BUDGET_NAME \
  --amount 500 \
  --category Cost \
  --time-grain Monthly \
  --start-date $START_DATE \
  --end-date $END_DATE \
  --resource-group $RG \
  --notifications '{"Actual80":{"enabled":true,"operator":"GreaterThan","threshold":80,"contactEmails":["'$CONTACT_EMAIL'"]},"Forecast100":{"enabled":true,"operator":"GreaterThan","threshold":100,"contactEmails":["'$CONTACT_EMAIL'"]}}'

# Review budget
az consumption budget show --name $BUDGET_NAME --resource-group $RG -o jsonc

# Cost allocation rule example
az costmanagement allocation-rule create \
  --scope "/subscriptions/$SUBSCRIPTION_ID" \
  --name "shared-services-split" \
  --allocation-type Split \
  --source "shared-services" \
  --targets "app1,app2" \
  --percentages "50,50"

# Configure anomaly detection (preview/support varies by tenant)
az costmanagement anomaly-detection setting create \
  --scope "/subscriptions/$SUBSCRIPTION_ID" \
  --name "default-anomaly-setting" \
  --threshold 20 \
  --look-back-period 30 \
  --contact-emails $CONTACT_EMAIL

# Review Azure Advisor cost recommendations
az advisor recommendation list --category Cost -o table

# Spending limit check (offer-dependent)
az account subscription show --subscription $SUBSCRIPTION_ID --query "{name:displayName, state:state, subscriptionId:subscriptionId}" -o table
```

#### PowerShell
```powershell
$SubscriptionId = (Get-AzContext).Subscription.Id
$ResourceGroup = "rg-cost-governance-lab"
$Location = "eastus"
$BudgetName = "az305-monthly-budget"
$ContactEmail = "alerts@contoso.com"
$StartDate = (Get-Date -Day 1).ToString('yyyy-MM-dd')
$EndDate = (Get-Date).AddMonths(12).ToString('yyyy-MM-dd')

New-AzResourceGroup -Name $ResourceGroup -Location $Location

# Create budget with alerts
New-AzConsumptionBudget -Name $BudgetName -ResourceGroupName $ResourceGroup -Amount 500 -TimeGrain Monthly -Category Cost -StartDate $StartDate -EndDate $EndDate -NotificationKey Actual80 -NotificationThreshold 80 -ContactEmail $ContactEmail

# Review budget
Get-AzConsumptionBudget -Name $BudgetName -ResourceGroupName $ResourceGroup

# Cost allocation rule via REST when cmdlet coverage is limited
Invoke-AzRestMethod -Method PUT -Path "/subscriptions/$SubscriptionId/providers/Microsoft.CostManagement/allocationRules/shared-services-split?api-version=2023-03-01" -Payload '{"properties":{"allocationType":"Split"}}'

# Anomaly detection via REST when cmdlet coverage is limited
Invoke-AzRestMethod -Method PUT -Path "/subscriptions/$SubscriptionId/providers/Microsoft.CostManagement/anomalyDetectionSettings/default-anomaly-setting?api-version=2023-12-01" -Payload '{"properties":{"threshold":20}}'

# Advisor cost recommendations
Get-AzAdvisorRecommendation -Category Cost

# Spending limit check (portal action may still be required depending on offer type)
Get-AzSubscription -SubscriptionId $SubscriptionId | Select-Object Name, Id, State
```

### IaC implementation
**Terraform starter:**
```hcl
resource "azurerm_consumption_budget_resource_group" "monthly" {
  name              = "az305-monthly-budget"
  resource_group_id = azurerm_resource_group.cost.id
  amount            = 500
  time_grain        = "Monthly"

  notification {
    enabled        = true
    threshold      = 80
    operator       = "GreaterThan"
    contact_emails = ["alerts@contoso.com"]
  }
}
```

### Verification steps
- Confirm the budget exists and thresholds are present.
- Confirm Advisor returns cost recommendations.
- Confirm anomaly detection and allocation-rule API calls succeed in your tenant.
- Confirm whether your subscription offer supports a spending limit.

### Cleanup

#### Azure CLI
```bash
az consumption budget delete --name $BUDGET_NAME --resource-group $RG
az costmanagement allocation-rule delete --scope "/subscriptions/$SUBSCRIPTION_ID" --name "shared-services-split"
az costmanagement anomaly-detection setting delete --scope "/subscriptions/$SUBSCRIPTION_ID" --name "default-anomaly-setting"
az group delete --name $RG --yes --no-wait
```

#### PowerShell
```powershell
Remove-AzConsumptionBudget -Name $BudgetName -ResourceGroupName $ResourceGroup
Invoke-AzRestMethod -Method DELETE -Path "/subscriptions/$SubscriptionId/providers/Microsoft.CostManagement/allocationRules/shared-services-split?api-version=2023-03-01"
Invoke-AzRestMethod -Method DELETE -Path "/subscriptions/$SubscriptionId/providers/Microsoft.CostManagement/anomalyDetectionSettings/default-anomaly-setting?api-version=2023-12-01"
Remove-AzResourceGroup -Name $ResourceGroup -Force -AsJob
```

### Exam Tip
**Budgets alert; they do not stop spending.** If the question asks for hard governance, combine budgets with Policy, SKU restrictions, and approval processes.

---

## Decision Summary Table

| Governance need | Best Azure service/control | Why | Watch out for |
|---|---|---|---|
| Organize subscriptions for enterprise governance | **Management Groups** | Enables inherited policy and RBAC at scale | Overly deep hierarchy complicates troubleshooting |
| Delegate least privilege for operations | **RBAC / Custom RBAC** | Fine-grained access control | Custom roles add maintenance overhead |
| Enforce standards like tags, locations, SKUs | **Azure Policy** | Prevents or audits non-compliant deployments | `Deny` can break deployments if rolled out too broadly |
| Prevent accidental deletion or modification | **Resource Locks** | Simple, immediate protection even for admins | Can block automation; not a replacement for Policy |
| Repeatable governed environments | **Deployment Stacks** (modern) / **Blueprints** (legacy) | Stacks add lifecycle control and deny settings | Blueprints are legacy and should not be the default for new designs |
| Continuous security/regulatory posture review | **Defender for Cloud + Policy** | Visibility, recommendations, and compliance dashboards | Visibility is not the same as enforcement |
| Cost control and early overspend warning | **Budgets + Advisor + Allocation/Anomaly tooling** | Alerts, waste detection, and chargeback support | Budgets do not enforce a hard stop on spend |

## Exam-style review questions
1. A company needs to ensure all production subscriptions inherit the same security baseline while allowing app teams to manage their own workloads. Would you choose management groups, resource groups, or separate policy assignments per subscription, and why?
2. Your platform team needs to block premium storage SKUs in dev but only report on violations in prod for one month before enforcement. Which Azure governance controls and rollout sequence best fit this requirement?
3. A critical storage account must not be deleted even by subscription Owners, but app teams still need to read compliance status and cost reports. Which combination of RBAC, Policy, and locks would you design?
4. A security team wants every new Key Vault to send diagnostic logs to a central Log Analytics workspace automatically, including existing vaults. Why is `DeployIfNotExists` a better fit than `Deny`?
5. Your organization still uses Azure Blueprints in a few subscriptions. What is the strategic replacement for new designs, and what migration considerations would you raise?
