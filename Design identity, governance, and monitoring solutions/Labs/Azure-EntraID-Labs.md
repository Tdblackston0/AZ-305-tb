# Microsoft Entra ID Hands-On Labs (AZ-305)

> 📖 **Cheat Sheet:** [Microsoft Entra ID](../Azure-EntraID.md)

> **Primary exam domain:** Design identity, governance, and monitoring solutions (25-30%)  
> **Recommended tools:** Azure CLI 2.56+, Microsoft Graph PowerShell, Az PowerShell  
> **Important:** Use a dedicated lab tenant or pilot groups. Several labs require Microsoft Entra ID P1/P2, Identity Governance, or Identity Protection licensing.

---

## Lab 1: Conditional Access Policy Configuration

### Objective
Create test users and groups, configure a named location, deploy a Conditional Access policy in report-only mode, then enable and verify MFA enforcement for targeted cloud apps.

### Exam Domain Mapping
**Primary:** Design identity, governance, and monitoring solutions (25-30%)  
**Secondary:** Design infrastructure solutions (30-35%) when protecting administrative apps and Azure management planes.

### When to Use This
Use Conditional Access when you must enforce security requirements based on identity, application, location, device state, or risk level.

### Key AZ-305 Concepts
- Conditional Access is the primary policy engine for modern identity controls.
- Named locations let you treat trusted IP ranges differently from unknown locations.
- Report-only mode reduces change risk before production enforcement.
- Always exclude emergency access accounts from broad MFA policies.

### Prerequisites
- Microsoft Entra ID P1 or higher
- Conditional Access Administrator, Security Administrator, or Global Administrator
- A break-glass account excluded from policy scope
- Microsoft Graph PowerShell installed: `Install-Module Microsoft.Graph -Scope CurrentUser`

### Architecture and Design Rationale
Design Conditional Access with **pilot groups first**, **trusted network exceptions only where justified**, and **report-only validation before enforcement**. In production, target specific apps and identities, exclude emergency access accounts, and prefer app-specific or risk-based policies over one global catch-all rule.

### Implementation Steps
1. Create a pilot group for Conditional Access testing.
2. Create two cloud-only test users.
3. Add a trusted named location for your corporate public IP range.
4. Create a report-only Conditional Access policy requiring MFA for Azure Management.
5. Attempt sign-ins from trusted and untrusted locations and review sign-in logs.
6. Change the policy state from report-only to enabled.

### Full CLI + PowerShell Commands

#### Azure CLI
```bash
# Variables
RG="rg-entra-ca-lab"
LOCATION="eastus"
GROUP_NAME="ca-pilot-users"
USER1_UPN="causer1@<tenant>.onmicrosoft.com"
USER2_UPN="causer2@<tenant>.onmicrosoft.com"
USER_PWD='P@ssw0rd1234!'
TRUSTED_IP_CIDR="203.0.113.10/32"   # Replace with your public IP/CIDR
AZURE_MGMT_APP_ID="797f4846-ba00-4fd7-ba43-dac1f8f63013"

# Create pilot group
az ad group create --display-name $GROUP_NAME --mail-nickname $GROUP_NAME
GROUP_ID=$(az ad group list --display-name $GROUP_NAME --query "[0].id" -o tsv)

# Create test users
az ad user create --display-name "CA User 1" --user-principal-name $USER1_UPN --password $USER_PWD
az ad user create --display-name "CA User 2" --user-principal-name $USER2_UPN --password $USER_PWD

USER1_ID=$(az ad user show --id $USER1_UPN --query id -o tsv)
USER2_ID=$(az ad user show --id $USER2_UPN --query id -o tsv)

# Add users to pilot group
az ad group member add --group $GROUP_ID --member-id $USER1_ID
az ad group member add --group $GROUP_ID --member-id $USER2_ID

# Create named location for trusted IPs
az rest --method POST \
  --url "https://graph.microsoft.com/v1.0/identity/conditionalAccess/namedLocations" \
  --body "{\"@odata.type\":\"#microsoft.graph.ipNamedLocation\",\"displayName\":\"Corp-HQ\",\"isTrusted\":true,\"ipRanges\":[{\"@odata.type\":\"#microsoft.graph.iPv4CidrRange\",\"cidrAddress\":\"$TRUSTED_IP_CIDR\"}]}"

NAMED_LOCATION_ID=$(az rest --method GET \
  --url "https://graph.microsoft.com/v1.0/identity/conditionalAccess/namedLocations" \
  --query "value[?displayName=='Corp-HQ'].id | [0]" -o tsv)

# Create report-only Conditional Access policy requiring MFA for Azure management
az rest --method POST \
  --url "https://graph.microsoft.com/v1.0/identity/conditionalAccess/policies" \
  --body "{\"displayName\":\"CA-Lab-Require-MFA-Azure-Management\",\"state\":\"enabledForReportingButNotEnforced\",\"conditions\":{\"users\":{\"includeGroups\":[\"$GROUP_ID\"],\"excludeUsers\":[\"<break-glass-object-id>\"]},\"applications\":{\"includeApplications\":[\"$AZURE_MGMT_APP_ID\"]},\"locations\":{\"includeLocations\":[\"All\"],\"excludeLocations\":[\"$NAMED_LOCATION_ID\"]},\"clientAppTypes\":[\"browser\",\"mobileAppsAndDesktopClients\"]},\"grantControls\":{\"operator\":\"OR\",\"builtInControls\":[\"mfa\"]}}"

# Review policy state
az rest --method GET \
  --url "https://graph.microsoft.com/v1.0/identity/conditionalAccess/policies" \
  --query "value[?displayName=='CA-Lab-Require-MFA-Azure-Management'].{Name:displayName,State:state}" -o table

# Enable the policy after testing
POLICY_ID=$(az rest --method GET \
  --url "https://graph.microsoft.com/v1.0/identity/conditionalAccess/policies" \
  --query "value[?displayName=='CA-Lab-Require-MFA-Azure-Management'].id | [0]" -o tsv)

az rest --method PATCH \
  --url "https://graph.microsoft.com/v1.0/identity/conditionalAccess/policies/$POLICY_ID" \
  --body '{"state":"enabled"}'
```

#### PowerShell (Microsoft Graph)
```powershell
Connect-MgGraph -Scopes "User.ReadWrite.All","Group.ReadWrite.All","Policy.ReadWrite.ConditionalAccess","Directory.Read.All","AuditLog.Read.All"

$password = ConvertTo-SecureString 'P@ssw0rd1234!' -AsPlainText -Force
$group = New-MgGroup -DisplayName 'ca-pilot-users-ps' -MailNickname 'ca-pilot-users-ps' -SecurityEnabled:$true -MailEnabled:$false

$user1 = New-MgUser -DisplayName 'CA User 3' -AccountEnabled:$true -MailNickname 'causer3' -UserPrincipalName 'causer3@<tenant>.onmicrosoft.com' -PasswordProfile @{ password = 'P@ssw0rd1234!'; forceChangePasswordNextSignIn = $true }
$user2 = New-MgUser -DisplayName 'CA User 4' -AccountEnabled:$true -MailNickname 'causer4' -UserPrincipalName 'causer4@<tenant>.onmicrosoft.com' -PasswordProfile @{ password = 'P@ssw0rd1234!'; forceChangePasswordNextSignIn = $true }

New-MgGroupMemberByRef -GroupId $group.Id -BodyParameter @{ '@odata.id' = "https://graph.microsoft.com/v1.0/directoryObjects/$($user1.Id)" }
New-MgGroupMemberByRef -GroupId $group.Id -BodyParameter @{ '@odata.id' = "https://graph.microsoft.com/v1.0/directoryObjects/$($user2.Id)" }

$namedLocationBody = @{
    '@odata.type' = '#microsoft.graph.ipNamedLocation'
    displayName   = 'Corp-Branch-PS'
    isTrusted     = $true
    ipRanges      = @(
        @{
            '@odata.type' = '#microsoft.graph.iPv4CidrRange'
            cidrAddress   = '203.0.113.20/32'
        }
    )
} | ConvertTo-Json -Depth 8

$namedLocation = Invoke-MgGraphRequest -Method POST -Uri 'https://graph.microsoft.com/v1.0/identity/conditionalAccess/namedLocations' -Body $namedLocationBody

$policyBody = @{
    displayName = 'CA-Lab-Require-MFA-PS'
    state       = 'enabledForReportingButNotEnforced'
    conditions  = @{
        users = @{
            includeGroups = @($group.Id)
            excludeUsers  = @('<break-glass-object-id>')
        }
        applications = @{
            includeApplications = @('797f4846-ba00-4fd7-ba43-dac1f8f63013')
        }
        locations = @{
            includeLocations = @('All')
            excludeLocations = @($namedLocation.id)
        }
        clientAppTypes = @('browser','mobileAppsAndDesktopClients')
    }
    grantControls = @{
        operator        = 'OR'
        builtInControls = @('mfa')
    }
} | ConvertTo-Json -Depth 10

$policy = Invoke-MgGraphRequest -Method POST -Uri 'https://graph.microsoft.com/v1.0/identity/conditionalAccess/policies' -Body $policyBody

# Enable after report-only validation
Invoke-MgGraphRequest -Method PATCH -Uri "https://graph.microsoft.com/v1.0/identity/conditionalAccess/policies/$($policy.id)" -Body '{"state":"enabled"}'
```

### Verification Steps
```bash
# Review recent sign-ins and Conditional Access outcomes
az rest --method GET \
  --url "https://graph.microsoft.com/v1.0/auditLogs/signIns?$top=10" \
  --query "value[].{User:userDisplayName,App:appDisplayName,CAStatus:conditionalAccessStatus,Result:status.errorCode}" -o table
```

```powershell
Get-MgIdentityConditionalAccessPolicy | Select-Object DisplayName, State
Get-MgAuditLogSignIn -Top 10 | Select-Object UserDisplayName, AppDisplayName, ConditionalAccessStatus
```

### Cleanup
```bash
az rest --method DELETE --url "https://graph.microsoft.com/v1.0/identity/conditionalAccess/policies/$POLICY_ID"
az rest --method DELETE --url "https://graph.microsoft.com/v1.0/identity/conditionalAccess/namedLocations/$NAMED_LOCATION_ID"
az ad user delete --id $USER1_UPN
az ad user delete --id $USER2_UPN
az ad group delete --group $GROUP_ID
```

```powershell
Invoke-MgGraphRequest -Method DELETE -Uri "https://graph.microsoft.com/v1.0/identity/conditionalAccess/policies/$($policy.id)"
Invoke-MgGraphRequest -Method DELETE -Uri "https://graph.microsoft.com/v1.0/identity/conditionalAccess/namedLocations/$($namedLocation.id)"
Remove-MgUser -UserId $user1.Id
Remove-MgUser -UserId $user2.Id
Remove-MgGroup -GroupId $group.Id
Disconnect-MgGraph
```

### Exam Tip
> **AZ-305 tip:** Conditional Access is the preferred control for MFA enforcement. Per-user MFA is legacy. On the exam, choose **report-only + pilot group + break-glass exclusion** over tenant-wide immediate enforcement.

---

## Lab 2: Multi-Factor Authentication Setup

### Objective
Configure MFA registration policy, enable authentication methods, compare per-user MFA with Conditional Access-based MFA, and validate user enrollment and sign-in experience.

### Exam Domain Mapping
**Primary:** Design identity, governance, and monitoring solutions (25-30%)

### When to Use This
Use MFA when you need a second factor for user authentication and want to reduce password-only attack risk.

### Key AZ-305 Concepts
- Authentication methods policy controls which MFA methods are allowed.
- Registration campaigns improve MFA enrollment rates.
- Conditional Access MFA is recommended over legacy per-user MFA.
- Production design should support phishing-resistant methods where possible.

### Prerequisites
- Microsoft Entra ID P1 for Conditional Access-based MFA
- Authentication Policy Administrator or Global Administrator
- MSOnline module for legacy per-user MFA comparison: `Install-Module MSOnline -Scope CurrentUser`
- Microsoft Graph PowerShell installed

### Architecture and Design Rationale
Enable modern MFA with **authentication methods policy** and a **registration campaign**, then enforce MFA using Conditional Access. Keep per-user MFA only for legacy comparison or transitional scenarios because it scales poorly and has less policy context.

### Implementation Steps
1. Create a pilot group for MFA rollout.
2. Allow Microsoft Authenticator and phone-based authentication methods.
3. Enable a registration campaign.
4. Compare legacy per-user MFA and Conditional Access MFA.
5. Sign in as a test user and complete registration.
6. Review authentication method registration and sign-in logs.

### Full CLI + PowerShell Commands

#### Azure CLI
```bash
MFA_GROUP="mfa-pilot-users"
MFA_USER_UPN="mfauser1@<tenant>.onmicrosoft.com"
MFA_USER_PWD='P@ssw0rd1234!'

az ad group create --display-name $MFA_GROUP --mail-nickname $MFA_GROUP
MFA_GROUP_ID=$(az ad group list --display-name $MFA_GROUP --query "[0].id" -o tsv)

az ad user create --display-name "MFA User 1" --user-principal-name $MFA_USER_UPN --password $MFA_USER_PWD
MFA_USER_ID=$(az ad user show --id $MFA_USER_UPN --query id -o tsv)
az ad group member add --group $MFA_GROUP_ID --member-id $MFA_USER_ID

# Enable Microsoft Authenticator registration campaign for the pilot group
az rest --method PATCH \
  --url "https://graph.microsoft.com/v1.0/policies/authenticationMethodsPolicy/authenticationMethodConfigurations/MicrosoftAuthenticator" \
  --headers "Content-Type=application/json" \
  --body "{\"state\":\"enabled\",\"isSoftwareOathEnabled\":true,\"includeTargets\":[{\"id\":\"$MFA_GROUP_ID\",\"targetType\":\"group\",\"isRegistrationRequired\":true}],\"featureSettings\":{\"displayLocationInformationRequiredState\":{\"state\":\"default\"},\"companionAppAllowedState\":{\"state\":\"default\"}}}"

# Enable SMS sign-in / phone method for the pilot group
az rest --method PATCH \
  --url "https://graph.microsoft.com/v1.0/policies/authenticationMethodsPolicy/authenticationMethodConfigurations/Sms" \
  --headers "Content-Type=application/json" \
  --body "{\"state\":\"enabled\",\"includeTargets\":[{\"id\":\"$MFA_GROUP_ID\",\"targetType\":\"group\"}]}"

# Conditional Access MFA policy (recommended approach)
az rest --method POST \
  --url "https://graph.microsoft.com/v1.0/identity/conditionalAccess/policies" \
  --body "{\"displayName\":\"CA-Lab-Require-MFA-All-Cloud-Apps\",\"state\":\"enabledForReportingButNotEnforced\",\"conditions\":{\"users\":{\"includeGroups\":[\"$MFA_GROUP_ID\"]},\"applications\":{\"includeApplications\":[\"All\"]},\"clientAppTypes\":[\"browser\",\"mobileAppsAndDesktopClients\"]},\"grantControls\":{\"operator\":\"OR\",\"builtInControls\":[\"mfa\"]}}"
```

#### PowerShell (Graph + MSOnline)
```powershell
Connect-MgGraph -Scopes "Policy.ReadWrite.AuthenticationMethod","Policy.ReadWrite.ConditionalAccess","User.ReadWrite.All","Group.ReadWrite.All","AuditLog.Read.All"

$group = New-MgGroup -DisplayName 'mfa-pilot-users-ps' -MailNickname 'mfa-pilot-users-ps' -SecurityEnabled:$true -MailEnabled:$false
$user = New-MgUser -DisplayName 'MFA User 2' -AccountEnabled:$true -MailNickname 'mfauser2' -UserPrincipalName 'mfauser2@<tenant>.onmicrosoft.com' -PasswordProfile @{ password = 'P@ssw0rd1234!'; forceChangePasswordNextSignIn = $true }
New-MgGroupMemberByRef -GroupId $group.Id -BodyParameter @{ '@odata.id' = "https://graph.microsoft.com/v1.0/directoryObjects/$($user.Id)" }

# Enable Microsoft Authenticator for the pilot group
$authenticatorBody = @{
    state = 'enabled'
    includeTargets = @(
        @{ id = $group.Id; targetType = 'group'; isRegistrationRequired = $true }
    )
} | ConvertTo-Json -Depth 6
Invoke-MgGraphRequest -Method PATCH -Uri 'https://graph.microsoft.com/v1.0/policies/authenticationMethodsPolicy/authenticationMethodConfigurations/MicrosoftAuthenticator' -Body $authenticatorBody

# Enable SMS method for the pilot group
$smsBody = @{
    state = 'enabled'
    includeTargets = @(
        @{ id = $group.Id; targetType = 'group' }
    )
} | ConvertTo-Json -Depth 6
Invoke-MgGraphRequest -Method PATCH -Uri 'https://graph.microsoft.com/v1.0/policies/authenticationMethodsPolicy/authenticationMethodConfigurations/Sms' -Body $smsBody

# Conditional Access MFA policy
$caBody = @{
    displayName = 'CA-Lab-MFA-PS'
    state = 'enabled'
    conditions = @{
        users = @{ includeGroups = @($group.Id) }
        applications = @{ includeApplications = @('All') }
        clientAppTypes = @('browser','mobileAppsAndDesktopClients')
    }
    grantControls = @{ operator = 'OR'; builtInControls = @('mfa') }
} | ConvertTo-Json -Depth 10
$caPolicy = Invoke-MgGraphRequest -Method POST -Uri 'https://graph.microsoft.com/v1.0/identity/conditionalAccess/policies' -Body $caBody

# Legacy per-user MFA comparison (not recommended for new designs)
Connect-MsolService
$requirement = New-Object -TypeName Microsoft.Online.Administration.StrongAuthenticationRequirement
$requirement.RelyingParty = '*'
$requirement.State = 'Enabled'
Set-MsolUser -UserPrincipalName $user.UserPrincipalName -StrongAuthenticationRequirements @($requirement)
```

### Verification Steps
```bash
# Review authentication methods policy
az rest --method GET \
  --url "https://graph.microsoft.com/v1.0/policies/authenticationMethodsPolicy/authenticationMethodConfigurations" \
  --query "value[].{Method:id,State:state}" -o table

# Review sign-in outcomes
az rest --method GET \
  --url "https://graph.microsoft.com/v1.0/auditLogs/signIns?$top=10" \
  --query "value[].{User:userDisplayName,Status:conditionalAccessStatus,AuthReq:authenticationRequirement}" -o table
```

```powershell
Get-MgUserAuthenticationMethod -UserId $user.Id
Get-MgAuditLogSignIn -Top 10 | Select-Object UserDisplayName, ConditionalAccessStatus, AuthenticationRequirement
```

### Cleanup
```bash
MFA_POLICY_ID=$(az rest --method GET --url "https://graph.microsoft.com/v1.0/identity/conditionalAccess/policies" --query "value[?displayName=='CA-Lab-Require-MFA-All-Cloud-Apps'].id | [0]" -o tsv)
az rest --method DELETE --url "https://graph.microsoft.com/v1.0/identity/conditionalAccess/policies/$MFA_POLICY_ID"
az ad user delete --id $MFA_USER_UPN
az ad group delete --group $MFA_GROUP_ID
```

```powershell
Set-MsolUser -UserPrincipalName $user.UserPrincipalName -StrongAuthenticationRequirements @()
Invoke-MgGraphRequest -Method DELETE -Uri "https://graph.microsoft.com/v1.0/identity/conditionalAccess/policies/$($caPolicy.id)"
Remove-MgUser -UserId $user.Id
Remove-MgGroup -GroupId $group.Id
Disconnect-MgGraph
```

### Exam Tip
> **AZ-305 tip:** If the exam asks how to enforce MFA in a scalable, context-aware way, the answer is usually **Conditional Access**, not per-user MFA.

---

## Lab 3: Managed Identity Implementation

### Objective
Create system-assigned and user-assigned managed identities, grant least-privilege access to Key Vault and Storage, and validate credential-free authentication from a VM.

### Exam Domain Mapping
**Primary:** Design identity, governance, and monitoring solutions (25-30%)  
**Secondary:** Design infrastructure solutions (30-35%) and Design data storage solutions (20-25%).

### When to Use This
Use managed identities when Azure resources need to authenticate to Azure services without storing secrets or certificates.

### Key AZ-305 Concepts
- System-assigned identities are lifecycle-coupled to one resource.
- User-assigned identities are reusable across multiple resources.
- RBAC should grant only the exact data-plane or control-plane permission required.
- Managed identities are preferred over service principals with secrets for Azure-hosted workloads.

### Prerequisites
- Contributor or Owner on the subscription/resource group
- Key Vault and Storage Account permissions to assign roles
- Azure CLI and Az PowerShell installed

### Architecture and Design Rationale
Use a **system-assigned identity** for a single VM workload and a **user-assigned identity** when the same identity may be reused across multiple workloads. Prefer **Azure RBAC** for Key Vault and Storage access, and validate access from the workload itself using `az login --identity`.

### Implementation Steps
1. Create a resource group, VM, Key Vault, and Storage Account.
2. Enable a system-assigned managed identity on the VM.
3. Create a user-assigned managed identity and attach it to the VM.
4. Assign Key Vault Secrets User and Storage Blob Data Reader roles.
5. Test token-based access from inside the VM.
6. Retrieve a Key Vault secret and list a storage container using each identity.

### Full CLI + PowerShell Commands

#### Azure CLI
```bash
RG="rg-entra-mi-lab"
LOCATION="eastus"
VM_NAME="vm-entra-mi-lab"
ADMIN_USER="azureuser"
KV_NAME="kventrami$RANDOM"
SA_NAME="stentrami$RANDOM"
UAMI_NAME="uami-entra-lab"
SECRET_NAME="app-secret"
CONTAINER="appdata"

az group create --name $RG --location $LOCATION

# Create VM with system-assigned managed identity
az vm create \
  --resource-group $RG \
  --name $VM_NAME \
  --image Ubuntu2204 \
  --admin-username $ADMIN_USER \
  --generate-ssh-keys \
  --assign-identity

SYSTEM_PRINCIPAL_ID=$(az vm show -g $RG -n $VM_NAME --query identity.principalId -o tsv)

# Create user-assigned managed identity
az identity create --name $UAMI_NAME --resource-group $RG --location $LOCATION
UAMI_ID=$(az identity show --name $UAMI_NAME --resource-group $RG --query id -o tsv)
UAMI_CLIENT_ID=$(az identity show --name $UAMI_NAME --resource-group $RG --query clientId -o tsv)
UAMI_PRINCIPAL_ID=$(az identity show --name $UAMI_NAME --resource-group $RG --query principalId -o tsv)

# Attach user-assigned identity to the VM
az vm identity assign --resource-group $RG --name $VM_NAME --identities $UAMI_ID

# Create Key Vault and Storage Account
az keyvault create --name $KV_NAME --resource-group $RG --location $LOCATION --enable-rbac-authorization true
az keyvault secret set --vault-name $KV_NAME --name $SECRET_NAME --value "SuperSecretValue123!"

az storage account create --name $SA_NAME --resource-group $RG --location $LOCATION --sku Standard_LRS --kind StorageV2
az storage container create --name $CONTAINER --account-name $SA_NAME --auth-mode login

KV_ID=$(az keyvault show --name $KV_NAME --resource-group $RG --query id -o tsv)
SA_ID=$(az storage account show --name $SA_NAME --resource-group $RG --query id -o tsv)

# Assign RBAC roles
az role assignment create --assignee-object-id $SYSTEM_PRINCIPAL_ID --role "Key Vault Secrets User" --scope $KV_ID
az role assignment create --assignee-object-id $SYSTEM_PRINCIPAL_ID --role "Storage Blob Data Reader" --scope $SA_ID
az role assignment create --assignee-object-id $UAMI_PRINCIPAL_ID --role "Key Vault Secrets User" --scope $KV_ID
az role assignment create --assignee-object-id $UAMI_PRINCIPAL_ID --role "Storage Blob Data Reader" --scope $SA_ID

# Test system-assigned identity from the VM
az vm run-command invoke --resource-group $RG --name $VM_NAME --command-id RunShellScript --scripts \
"az login --identity && az keyvault secret show --vault-name $KV_NAME --name $SECRET_NAME --query value -o tsv && az storage container list --account-name $SA_NAME --auth-mode login -o table"

# Test user-assigned identity from the VM
az vm run-command invoke --resource-group $RG --name $VM_NAME --command-id RunShellScript --scripts \
"az login --identity --username $UAMI_CLIENT_ID && az keyvault secret show --vault-name $KV_NAME --name $SECRET_NAME --query value -o tsv"
```

#### PowerShell (Az)
```powershell
Connect-AzAccount

$rg = 'rg-entra-mi-lab-ps'
$location = 'eastus'
$vmName = 'vm-entramips'
$kvName = ('kventramips' + (Get-Random -Maximum 99999))
$saName = ('stentramips' + (Get-Random -Maximum 99999))
$uamiName = 'uami-entra-ps'

New-AzResourceGroup -Name $rg -Location $location

# Create user-assigned identity
$uami = New-AzUserAssignedIdentity -ResourceGroupName $rg -Name $uamiName -Location $location

# Create VM and enable system-assigned identity
$cred = Get-Credential -Message 'Enter local admin credentials for the lab VM'
$vm = New-AzVM -ResourceGroupName $rg -Name $vmName -Location $location -Credential $cred -Image Ubuntu2204 -IdentityType SystemAssigned
$vm = Get-AzVM -ResourceGroupName $rg -Name $vmName
Update-AzVM -ResourceGroupName $rg -VM $vm -IdentityType SystemAssigned,UserAssigned -IdentityId $uami.Id

# Create Key Vault and Storage
$keyVault = New-AzKeyVault -Name $kvName -ResourceGroupName $rg -Location $location -EnableRbacAuthorization
Set-AzKeyVaultSecret -VaultName $kvName -Name 'app-secret' -SecretValue (ConvertTo-SecureString 'SuperSecretValue123!' -AsPlainText -Force)

$storage = New-AzStorageAccount -ResourceGroupName $rg -Name $saName -Location $location -SkuName Standard_LRS -Kind StorageV2
New-AzRoleAssignment -ObjectId $vm.Identity.PrincipalId -RoleDefinitionName 'Key Vault Secrets User' -Scope $keyVault.ResourceId
New-AzRoleAssignment -ObjectId $vm.Identity.PrincipalId -RoleDefinitionName 'Storage Blob Data Reader' -Scope $storage.Id
New-AzRoleAssignment -ObjectId $uami.PrincipalId -RoleDefinitionName 'Key Vault Secrets User' -Scope $keyVault.ResourceId
New-AzRoleAssignment -ObjectId $uami.PrincipalId -RoleDefinitionName 'Storage Blob Data Reader' -Scope $storage.Id

# Test from the VM
Invoke-AzVMRunCommand -ResourceGroupName $rg -VMName $vmName -CommandId 'RunShellScript' -ScriptString @"
az login --identity
az keyvault secret show --vault-name $kvName --name app-secret --query value -o tsv
az login --identity --username $($uami.ClientId)
az storage container list --account-name $saName --auth-mode login -o table
"@
```

### Verification Steps
```bash
az vm show -g $RG -n $VM_NAME --query identity
az role assignment list --scope $KV_ID --query "[].{Role:roleDefinitionName,Principal:principalId}" -o table
az role assignment list --scope $SA_ID --query "[].{Role:roleDefinitionName,Principal:principalId}" -o table
```

```powershell
Get-AzUserAssignedIdentity -ResourceGroupName $rg -Name $uamiName
Get-AzRoleAssignment -Scope $keyVault.ResourceId | Select-Object RoleDefinitionName, ObjectId
```

### Cleanup
```bash
az group delete --name $RG --yes --no-wait
```

```powershell
Remove-AzResourceGroup -Name $rg -Force
```

### Exam Tip
> **AZ-305 tip:** For Azure-hosted workloads, prefer **managed identity** over stored secrets. Choose **system-assigned** for one resource and **user-assigned** when multiple resources need the same identity.

---

## Lab 4: Privileged Identity Management (PIM)

### Objective
Enable PIM for Entra roles, create an eligible assignment, configure activation controls, activate the role, review audit logs, and create an access review.

### Exam Domain Mapping
**Primary:** Design identity, governance, and monitoring solutions (25-30%)

### When to Use This
Use PIM when administrators need just-in-time access instead of standing privileged assignments.

### Key AZ-305 Concepts
- Eligible assignments reduce persistent privilege exposure.
- Activation can require MFA, justification, ticket details, and approval.
- Audit logs and access reviews support governance and compliance.
- PIM for Entra roles requires Microsoft Entra ID P2.

### Prerequisites
- Microsoft Entra ID P2
- Privileged Role Administrator or Global Administrator
- Microsoft Graph PowerShell with role management scopes

### Architecture and Design Rationale
PIM enforces **least privilege** by keeping privileged users eligible rather than permanently active. In production, use approval for high-impact roles, short activation durations, and periodic access reviews.

### Implementation Steps
1. Select a low-risk role for testing such as Security Reader or User Administrator.
2. Create a test administrator or admin group.
3. Make the assignment **eligible** instead of active.
4. Configure activation controls: MFA, justification, and approval.
5. Activate the role as the test user.
6. Review directory audit logs and create an access review.

### Full CLI + PowerShell Commands

#### Azure CLI
```bash
PIM_GROUP="pim-eligible-admins"
REVIEWER_OBJECT_ID="<reviewer-object-id>"

az ad group create --display-name $PIM_GROUP --mail-nickname $PIM_GROUP
PIM_GROUP_ID=$(az ad group list --display-name $PIM_GROUP --query "[0].id" -o tsv)

ROLE_DEF_ID=$(az rest --method GET \
  --url "https://graph.microsoft.com/v1.0/roleManagement/directory/roleDefinitions?$filter=displayName eq 'User Administrator'" \
  --query "value[0].id" -o tsv)

# Create eligible role assignment request
az rest --method POST \
  --url "https://graph.microsoft.com/v1.0/roleManagement/directory/roleEligibilityScheduleRequests" \
  --body "{\"action\":\"adminAssign\",\"justification\":\"AZ-305 PIM lab\",\"principalId\":\"$PIM_GROUP_ID\",\"roleDefinitionId\":\"$ROLE_DEF_ID\",\"directoryScopeId\":\"/\",\"scheduleInfo\":{\"startDateTime\":\"2026-01-01T00:00:00Z\",\"expiration\":{\"type\":\"afterDuration\",\"duration\":\"PT8H\"}}}"

# Review policy rules used for activation requirements
az rest --method GET \
  --url "https://graph.microsoft.com/v1.0/policies/roleManagementPolicies?$filter=scopeId eq '/' and scopeType eq 'DirectoryRole'" \
  --query "value[].{Id:id,DisplayName:displayName}" -o table

# Example activation policy update (replace POLICY_ID as needed)
POLICY_ID="<policy-id>"
az rest --method PATCH \
  --url "https://graph.microsoft.com/v1.0/policies/roleManagementPolicies/$POLICY_ID/rules/Enablement_EndUser_Assignment" \
  --body '{"@odata.type":"#microsoft.graph.unifiedRoleManagementPolicyEnablementRule","id":"Enablement_EndUser_Assignment","enabledRules":["MultiFactorAuthentication","Justification"]}'

# Activate role (self-activate)
az rest --method POST \
  --url "https://graph.microsoft.com/v1.0/roleManagement/directory/roleAssignmentScheduleRequests" \
  --body "{\"action\":\"selfActivate\",\"justification\":\"Activating for lab task\",\"principalId\":\"<user-or-group-member-object-id>\",\"roleDefinitionId\":\"$ROLE_DEF_ID\",\"directoryScopeId\":\"/\",\"scheduleInfo\":{\"startDateTime\":\"2026-01-01T01:00:00Z\",\"expiration\":{\"type\":\"afterDuration\",\"duration\":\"PT1H\"}}}"

# Create access review for the eligible admin group
az rest --method POST \
  --url "https://graph.microsoft.com/v1.0/identityGovernance/accessReviews/definitions" \
  --body "{\"displayName\":\"Quarterly Review - PIM Admins\",\"descriptionForAdmins\":\"Review privileged eligibility\",\"scope\":{\"@odata.type\":\"#microsoft.graph.accessReviewQueryScope\",\"query\":\"/groups/$PIM_GROUP_ID/transitiveMembers\",\"queryType\":\"MicrosoftGraph\"},\"reviewers\":[{\"query\":\"/users/$REVIEWER_OBJECT_ID\",\"queryType\":\"MicrosoftGraph\"}],\"settings\":{\"mailNotificationsEnabled\":true,\"reminderNotificationsEnabled\":true,\"justificationRequiredOnApproval\":true,\"defaultDecisionEnabled\":false,\"instanceDurationInDays\":7,\"recurrence\":{\"pattern\":{\"type\":\"absoluteMonthly\",\"interval\":3,\"dayOfMonth\":1},\"range\":{\"type\":\"numbered\",\"numberOfOccurrences\":4,\"startDate\":\"2026-01-01\"}}}}"
```

#### PowerShell (Microsoft Graph)
```powershell
Connect-MgGraph -Scopes "RoleManagement.ReadWrite.Directory","Directory.ReadWrite.All","AuditLog.Read.All","AccessReview.ReadWrite.All"

$group = New-MgGroup -DisplayName 'pim-eligible-admins-ps' -MailNickname 'pim-eligible-admins-ps' -SecurityEnabled:$true -MailEnabled:$false
$role = Invoke-MgGraphRequest -Method GET -Uri "https://graph.microsoft.com/v1.0/roleManagement/directory/roleDefinitions?`$filter=displayName eq 'User Administrator'"
$roleId = $role.value[0].id

# Create eligible assignment
$eligibilityBody = @{
    action = 'adminAssign'
    justification = 'AZ-305 PIM lab'
    principalId = $group.Id
    roleDefinitionId = $roleId
    directoryScopeId = '/'
    scheduleInfo = @{
        startDateTime = '2026-01-01T00:00:00Z'
        expiration = @{ type = 'afterDuration'; duration = 'PT8H' }
    }
} | ConvertTo-Json -Depth 8
Invoke-MgGraphRequest -Method POST -Uri 'https://graph.microsoft.com/v1.0/roleManagement/directory/roleEligibilityScheduleRequests' -Body $eligibilityBody

# Configure activation requirements (MFA + justification)
$policyRuleBody = @{
    '@odata.type' = '#microsoft.graph.unifiedRoleManagementPolicyEnablementRule'
    id = 'Enablement_EndUser_Assignment'
    enabledRules = @('MultiFactorAuthentication','Justification')
} | ConvertTo-Json -Depth 6
Invoke-MgGraphRequest -Method PATCH -Uri 'https://graph.microsoft.com/v1.0/policies/roleManagementPolicies/<policy-id>/rules/Enablement_EndUser_Assignment' -Body $policyRuleBody

# Review audit logs
Get-MgAuditLogDirectoryAudit -Top 20 | Where-Object CategoryDisplayName -match 'RoleManagement'
```

### Verification Steps
```bash
az rest --method GET \
  --url "https://graph.microsoft.com/v1.0/roleManagement/directory/roleEligibilitySchedules?$filter=principalId eq '$PIM_GROUP_ID'" \
  --query "value[].{Principal:principalId,Role:roleDefinitionId,Status:status}" -o table

az rest --method GET \
  --url "https://graph.microsoft.com/v1.0/auditLogs/directoryAudits?$top=10" \
  --query "value[].{Activity:activityDisplayName,Category:category,Result:result}" -o table
```

```powershell
Invoke-MgGraphRequest -Method GET -Uri "https://graph.microsoft.com/v1.0/roleManagement/directory/roleEligibilitySchedules?`$filter=principalId eq '$($group.Id)'"
Get-MgAuditLogDirectoryAudit -Top 10 | Select-Object ActivityDisplayName, CategoryDisplayName, Result
```

### Cleanup
```bash
# Remove the access review and group after recording IDs
az ad group delete --group $PIM_GROUP_ID
```

```powershell
Remove-MgGroup -GroupId $group.Id
Disconnect-MgGraph
```

### Exam Tip
> **AZ-305 tip:** If a question emphasizes reducing standing admin rights, improving governance, or enforcing just-in-time elevation, choose **PIM with eligible assignments**, not permanent role assignment.

---

## Lab 5: B2B Guest User Collaboration

### Objective
Invite an external guest, configure external collaboration and cross-tenant settings, test guest access, review activity, and clean up the guest lifecycle.

### Exam Domain Mapping
**Primary:** Design identity, governance, and monitoring solutions (25-30%)

### When to Use This
Use B2B collaboration when partners, vendors, or contractors need controlled access to your tenant resources while authenticating with their home identity.

### Key AZ-305 Concepts
- B2B is for external collaboration; B2C is for customer identity.
- Cross-tenant access policies control inbound and outbound trust.
- Guest users should be constrained with Conditional Access and periodic review.
- Lifecycle controls matter for compliance and least privilege.

### Prerequisites
- External user email address from another tenant or MSA
- External Identities Administrator or Global Administrator
- Optional second test tenant for cross-tenant access testing

### Architecture and Design Rationale
A strong B2B design uses **guest invitation + security group assignment + Conditional Access + access review**. Trust partner MFA only when the partner tenant is governed and verified. Keep guest permissions narrow and time-bound.

### Implementation Steps
1. Create a partner access group.
2. Invite a guest user.
3. Configure external collaboration settings and cross-tenant partner trust.
4. Assign the guest to a group or app.
5. Review guest sign-ins and external user state.
6. Remove access when the collaboration ends.

### Full CLI + PowerShell Commands

#### Azure CLI
```bash
GUEST_EMAIL="partner.user@fabrikam.com"
PARTNER_TENANT_ID="<partner-tenant-id>"
GUEST_GROUP="b2b-partner-access"

az ad group create --display-name $GUEST_GROUP --mail-nickname $GUEST_GROUP
GUEST_GROUP_ID=$(az ad group list --display-name $GUEST_GROUP --query "[0].id" -o tsv)

# Invite guest user
az rest --method POST \
  --url "https://graph.microsoft.com/v1.0/invitations" \
  --body "{\"invitedUserEmailAddress\":\"$GUEST_EMAIL\",\"inviteRedirectUrl\":\"https://portal.azure.com\",\"sendInvitationMessage\":true}"

# Configure external collaboration settings
az rest --method PATCH \
  --url "https://graph.microsoft.com/v1.0/policies/authorizationPolicy" \
  --body '{"allowInvitesFrom":"adminsAndGuestInviters"}'

# Configure cross-tenant access policy for partner tenant
az rest --method PUT \
  --url "https://graph.microsoft.com/v1.0/policies/crossTenantAccessPolicy/partners/$PARTNER_TENANT_ID" \
  --body '{"b2bCollaborationInbound":{"usersAndGroups":{"accessType":"allowed","targets":[{"target":"AllUsers","targetType":"user"}]},"applications":{"accessType":"allowed","targets":[{"target":"AllApplications","targetType":"application"}]}}}'

# Get invited guest object ID and add to group
GUEST_OBJECT_ID=$(az ad user list --filter "mail eq '$GUEST_EMAIL'" --query "[0].id" -o tsv)
az ad group member add --group $GUEST_GROUP_ID --member-id $GUEST_OBJECT_ID
```

#### PowerShell (Microsoft Graph)
```powershell
Connect-MgGraph -Scopes "User.Invite.All","Policy.ReadWrite.CrossTenantAccess","Policy.ReadWrite.Authorization","Group.ReadWrite.All","AuditLog.Read.All"

$group = New-MgGroup -DisplayName 'b2b-partner-access-ps' -MailNickname 'b2b-partner-access-ps' -SecurityEnabled:$true -MailEnabled:$false

$inviteBody = @{
    invitedUserEmailAddress = 'partner.user@fabrikam.com'
    inviteRedirectUrl = 'https://portal.azure.com'
    sendInvitationMessage = $true
} | ConvertTo-Json
$invite = Invoke-MgGraphRequest -Method POST -Uri 'https://graph.microsoft.com/v1.0/invitations' -Body $inviteBody

# Allow only admins and guest inviters to send invitations
Invoke-MgGraphRequest -Method PATCH -Uri 'https://graph.microsoft.com/v1.0/policies/authorizationPolicy' -Body '{"allowInvitesFrom":"adminsAndGuestInviters"}'

# Create/Update partner cross-tenant settings
$partnerBody = @{
    b2bCollaborationInbound = @{
        usersAndGroups = @{ accessType = 'allowed'; targets = @(@{ target = 'AllUsers'; targetType = 'user' }) }
        applications = @{ accessType = 'allowed'; targets = @(@{ target = 'AllApplications'; targetType = 'application' }) }
    }
    inboundTrust = @{ isMfaAccepted = $true }
} | ConvertTo-Json -Depth 10
Invoke-MgGraphRequest -Method PUT -Uri 'https://graph.microsoft.com/v1.0/policies/crossTenantAccessPolicy/partners/<partner-tenant-id>' -Body $partnerBody

$guest = Get-MgUser -Filter "mail eq 'partner.user@fabrikam.com'"
New-MgGroupMemberByRef -GroupId $group.Id -BodyParameter @{ '@odata.id' = "https://graph.microsoft.com/v1.0/directoryObjects/$($guest.Id)" }
```

### Verification Steps
```bash
az ad user list --filter "userType eq 'Guest'" --query "[].{Name:displayName,UPN:userPrincipalName,State:externalUserState}" -o table
az rest --method GET --url "https://graph.microsoft.com/v1.0/auditLogs/signIns?$top=10" --query "value[?userPrincipalName!=null].{User:userDisplayName,App:appDisplayName,Status:status.errorCode}" -o table
```

```powershell
Get-MgUser -Filter "userType eq 'Guest'" | Select-Object DisplayName, Mail, ExternalUserState
Get-MgAuditLogSignIn -Top 10 | Select-Object UserDisplayName, AppDisplayName, Status
```

### Cleanup
```bash
az ad group delete --group $GUEST_GROUP_ID
az ad user delete --id $GUEST_OBJECT_ID
az rest --method DELETE --url "https://graph.microsoft.com/v1.0/policies/crossTenantAccessPolicy/partners/$PARTNER_TENANT_ID"
```

```powershell
Remove-MgGroup -GroupId $group.Id
Remove-MgUser -UserId $guest.Id
Disconnect-MgGraph
```

### Exam Tip
> **AZ-305 tip:** For partner access, choose **B2B guest collaboration**. Guests authenticate in their **home tenant**, but authorization happens in the **resource tenant**.

---

## Lab 6: App Registration & Service Principal

### Objective
Create an app registration, configure Microsoft Graph permissions, create a secret and certificate, provision a service principal, assign RBAC, and test client credentials authentication.

### Exam Domain Mapping
**Primary:** Design identity, governance, and monitoring solutions (25-30%)  
**Secondary:** Design infrastructure solutions (30-35%).

### When to Use This
Use app registrations and service principals when applications, automation, or external tools need their own identity to call Microsoft Graph or Azure resources.

### Key AZ-305 Concepts
- App registration defines the application identity configuration.
- Service principal is the tenant-local security object that gets RBAC assignments.
- Application permissions enable app-only access.
- Prefer certificates or workload identity federation over long-lived secrets.

### Prerequisites
- Application Administrator, Cloud Application Administrator, or Global Administrator
- Contributor/Owner on Azure scope for RBAC assignment
- OpenSSL or PowerShell certificate tools available

### Architecture and Design Rationale
Use **app registrations** only when managed identity is not possible. For Azure-hosted workloads, managed identity is usually better. If an app registration is required, prefer **certificate-based auth** or **federated credentials** over secrets.

### Implementation Steps
1. Create an app registration.
2. Add delegated or application permissions for Microsoft Graph.
3. Create a service principal.
4. Add a client secret and certificate.
5. Assign least-privilege RBAC to an Azure scope.
6. Test client credentials flow.

### Full CLI + PowerShell Commands

#### Azure CLI
```bash
APP_NAME="app-entra-lab"
RG_SCOPE="/subscriptions/<subscription-id>/resourceGroups/<resource-group>"
TENANT_ID=$(az account show --query tenantId -o tsv)

# Create app registration and service principal
az ad app create --display-name $APP_NAME
APP_ID=$(az ad app list --display-name $APP_NAME --query "[0].appId" -o tsv)
APP_OBJECT_ID=$(az ad app list --display-name $APP_NAME --query "[0].id" -o tsv)
az ad sp create --id $APP_ID
SP_OBJECT_ID=$(az ad sp list --filter "appId eq '$APP_ID'" --query "[0].id" -o tsv)

# Add Microsoft Graph application permission: User.Read.All
az ad app permission add \
  --id $APP_ID \
  --api 00000003-0000-0000-c000-000000000000 \
  --api-permissions df021288-bdef-4463-88db-98f22de89214=Role

# Grant admin consent
az ad app permission admin-consent --id $APP_ID

# Create client secret
az ad app credential reset --id $APP_ID --append --display-name "lab-secret" --years 1

# Assign Reader RBAC to the service principal
az role assignment create --assignee-object-id $SP_OBJECT_ID --role Reader --scope $RG_SCOPE
```

```powershell
# Create a self-signed certificate for the app
$cert = New-SelfSignedCertificate -Subject "CN=app-entra-lab" -CertStoreLocation "Cert:\CurrentUser\My" -KeyExportPolicy Exportable -NotAfter (Get-Date).AddYears(2)
Export-Certificate -Cert $cert -FilePath .\app-entra-lab.cer
```

```bash
# Authenticate with client credentials flow using the secret value returned earlier
CLIENT_SECRET="<saved-secret-value>"
curl -X POST "https://login.microsoftonline.com/$TENANT_ID/oauth2/v2.0/token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "client_id=$APP_ID&client_secret=$CLIENT_SECRET&scope=https%3A%2F%2Fgraph.microsoft.com%2F.default&grant_type=client_credentials"
```

#### PowerShell (Microsoft Graph + PowerShell)
```powershell
Connect-MgGraph -Scopes "Application.ReadWrite.All","AppRoleAssignment.ReadWrite.All","Directory.Read.All"

$app = New-MgApplication -DisplayName 'app-entra-lab-ps' -SignInAudience 'AzureADMyOrg'
$sp = New-MgServicePrincipal -AppId $app.AppId

# Add Microsoft Graph application permission
$graphSp = Get-MgServicePrincipal -Filter "appId eq '00000003-0000-0000-c000-000000000000'"
$userReadAll = $graphSp.AppRoles | Where-Object { $_.Value -eq 'User.Read.All' -and $_.AllowedMemberTypes -contains 'Application' }
New-MgServicePrincipalAppRoleAssignment -ServicePrincipalId $sp.Id -PrincipalId $sp.Id -ResourceId $graphSp.Id -AppRoleId $userReadAll.Id

# Create client secret
$secret = Add-MgApplicationPassword -ApplicationId $app.Id -BodyParameter @{ passwordCredential = @{ displayName = 'lab-secret'; endDateTime = (Get-Date).AddYears(1) } }
$secret.SecretText

# Upload certificate
$certBytes = [System.Convert]::ToBase64String((Get-Content .\app-entra-lab.cer -Encoding Byte))
Add-MgApplicationKey -ApplicationId $app.Id -BodyParameter @{ keyCredential = @{ displayName = 'lab-cert'; type = 'AsymmetricX509Cert'; usage = 'Verify'; key = [System.Convert]::FromBase64String($certBytes) } }

# Optional: assign Azure RBAC with Az PowerShell
Connect-AzAccount
New-AzRoleAssignment -ObjectId $sp.Id -RoleDefinitionName Reader -Scope '/subscriptions/<subscription-id>/resourceGroups/<resource-group>'

# Test client credentials flow
$tenantId = (Get-AzContext).Tenant.Id
$token = Invoke-RestMethod -Method POST -Uri "https://login.microsoftonline.com/$tenantId/oauth2/v2.0/token" -Body @{
    client_id     = $app.AppId
    client_secret = $secret.SecretText
    scope         = 'https://graph.microsoft.com/.default'
    grant_type    = 'client_credentials'
}
$token.access_token
```

### Verification Steps
```bash
az ad app show --id $APP_ID --query "{AppId:appId,DisplayName:displayName}" -o table
az ad sp show --id $APP_ID --query "{ServicePrincipalId:id,DisplayName:displayName}" -o table
az role assignment list --assignee-object-id $SP_OBJECT_ID --scope $RG_SCOPE -o table
```

```powershell
Get-MgApplication -ApplicationId $app.Id | Select-Object DisplayName, AppId
Get-MgServicePrincipal -ServicePrincipalId $sp.Id | Select-Object DisplayName, AppId
```

### Cleanup
```bash
az role assignment delete --assignee-object-id $SP_OBJECT_ID --role Reader --scope $RG_SCOPE
az ad app delete --id $APP_OBJECT_ID
rm -f ./app-entra-lab.cer
```

```powershell
Remove-MgApplication -ApplicationId $app.Id
Remove-Item .\app-entra-lab.cer -ErrorAction SilentlyContinue
Disconnect-MgGraph
```

### Exam Tip
> **AZ-305 tip:** If the workload runs in Azure, first ask whether **managed identity** can replace the app registration. Use app registrations when the workload is external, multi-tenant, or otherwise cannot use managed identity.

---

## Lab 7: Hybrid Identity with Azure AD Connect (Conceptual + Config)

### Objective
Review Microsoft Entra Connect prerequisites, walk through installation choices, configure Password Hash Sync and Seamless SSO, verify sync status, and practice troubleshooting common hybrid identity issues.

### Exam Domain Mapping
**Primary:** Design identity, governance, and monitoring solutions (25-30%)

### When to Use This
Use hybrid identity when on-premises Active Directory identities must be synchronized to Microsoft Entra ID for a unified identity plane.

### Key AZ-305 Concepts
- Password Hash Sync is the default recommendation for most organizations.
- Pass-through Authentication and federation are alternatives for specific requirements.
- Seamless SSO improves sign-in experience for domain-joined users.
- Staging mode is critical for migrations and high-availability planning.

### Prerequisites
- Dedicated Windows Server joined to the on-prem AD domain
- Enterprise Admin credentials for AD and Hybrid Identity Administrator/Global Administrator in Entra
- Microsoft Entra Connect installer downloaded
- Test OU and test users created on-premises

### Architecture and Design Rationale
For AZ-305, the default design answer is usually **Password Hash Sync + Seamless SSO** unless regulatory or third-party identity provider requirements force federation. Use **staging mode** for cutover safety and scope synchronization to test OUs first.

### Implementation Steps
1. Review supported topologies, UPN suffixes, and health prerequisites.
2. Launch the Entra Connect wizard on a dedicated sync server.
3. Choose **Custom installation** to review sign-in methods.
4. Enable **Password Hash Sync** and **Seamless SSO**.
5. Limit synchronization scope to a test OU for the lab.
6. Run an initial sync and validate synchronized attributes.
7. Troubleshoot common issues such as duplicate UPNs, missing sourceAnchor, or sync rule conflicts.

### Full CLI + PowerShell Commands

#### Azure CLI
```bash
# Cloud-side verification after sync
az rest --method GET \
  --url "https://graph.microsoft.com/v1.0/users?$filter=onPremisesSyncEnabled eq true" \
  --query "value[].{DisplayName:displayName,UPN:userPrincipalName,ImmutableId:onPremisesImmutableId}" -o table

# Review synchronized devices (if hybrid join is part of testing)
az rest --method GET \
  --url "https://graph.microsoft.com/v1.0/devices?$filter=trustType eq 'ServerAd'" \
  --query "value[].{Name:displayName,TrustType:trustType,Enabled:accountEnabled}" -o table
```

#### PowerShell (on Entra Connect server + Graph)
```powershell
# Launch the installer (GUI walkthrough is expected for this lab)
Start-Process .\AzureADConnect.msi

# After installation, validate scheduler and run profiles
Import-Module ADSync
Get-ADSyncScheduler
Start-ADSyncSyncCycle -PolicyType Initial
Get-ADSyncConnectorRunStatus

# Common troubleshooting commands
Get-WinEvent -LogName Application -MaxEvents 50 | Where-Object ProviderName -match 'ADSync'
Get-ADSyncScheduler | Select-Object SyncCycleEnabled, NextSyncCyclePolicyType, NextSyncCycleStartTimeInUTC

# On a domain-joined client, validate Seamless SSO / join state
cmd /c dsregcmd /status

# Cloud verification
Connect-MgGraph -Scopes "User.Read.All"
Get-MgUser -Filter "onPremisesSyncEnabled eq true" | Select-Object DisplayName, UserPrincipalName, OnPremisesImmutableId
```

### Verification Steps
- In the wizard or Synchronization Service Manager, confirm connectors are healthy.
- Verify the test OU objects appear in Entra ID.
- Confirm `onPremisesSyncEnabled = true` on synchronized users.
- Sign in from a domain-joined device and confirm Seamless SSO behavior.
- Check the **Microsoft Entra Connect Health** and Event Viewer logs for errors.

### Cleanup
```powershell
# Lab-only cleanup on a dedicated sync server
Import-Module ADSync
Set-ADSyncScheduler -SyncCycleEnabled $false

# If this is a disposable lab, disable staging/sync in the wizard and uninstall Microsoft Entra Connect manually.
# Do not do this in production without a rollback plan.
```

### Exam Tip
> **AZ-305 tip:** For most hybrid identity scenarios, the best answer is **Password Hash Sync + Seamless SSO**. Choose federation only when a requirement explicitly demands it.

---

## Lab 8: Identity Protection & Risk Policies

### Objective
Configure sign-in risk and user risk protections, simulate a risky sign-in scenario, review risk detections, and remediate risky users.

### Exam Domain Mapping
**Primary:** Design identity, governance, and monitoring solutions (25-30%)

### When to Use This
Use Identity Protection when you need automated response to identity-based threats such as leaked credentials, anonymous IP use, or impossible travel.

### Key AZ-305 Concepts
- Identity Protection requires Microsoft Entra ID P2.
- Sign-in risk and user risk are different signals and should drive different responses.
- Risk-based Conditional Access is preferred for adaptive controls.
- Remediation actions include MFA, password change, dismiss, and confirm compromised.

### Prerequisites
- Microsoft Entra ID P2
- Security Administrator, Conditional Access Administrator, or Global Administrator
- A test account in a pilot group
- Safe test method such as Tor Browser or anonymous IP service for risk simulation

### Architecture and Design Rationale
Use **risk-based Conditional Access** for adaptive response. A common design is **require MFA for medium/high sign-in risk** and **require password change for high user risk**. Start with pilot groups and review detections before tenant-wide rollout.

### Implementation Steps
1. Create a pilot group for Identity Protection testing.
2. Create one sign-in risk policy and one user risk policy.
3. Sign in using a risky source such as Tor Browser.
4. Review risky users and detections.
5. Remediate by confirming compromise, dismissing risk, or forcing password reset.

### Full CLI + PowerShell Commands

#### Azure CLI
```bash
RISK_GROUP="identity-protection-pilot"
RISK_USER_UPN="riskuser1@<tenant>.onmicrosoft.com"
RISK_USER_PWD='P@ssw0rd1234!'

az ad group create --display-name $RISK_GROUP --mail-nickname $RISK_GROUP
RISK_GROUP_ID=$(az ad group list --display-name $RISK_GROUP --query "[0].id" -o tsv)

az ad user create --display-name "Risk User 1" --user-principal-name $RISK_USER_UPN --password $RISK_USER_PWD
RISK_USER_ID=$(az ad user show --id $RISK_USER_UPN --query id -o tsv)
az ad group member add --group $RISK_GROUP_ID --member-id $RISK_USER_ID

# Sign-in risk policy: require MFA for medium/high risk
az rest --method POST \
  --url "https://graph.microsoft.com/v1.0/identity/conditionalAccess/policies" \
  --body "{\"displayName\":\"IP-Lab-SignInRisk-MFA\",\"state\":\"enabled\",\"conditions\":{\"users\":{\"includeGroups\":[\"$RISK_GROUP_ID\"]},\"applications\":{\"includeApplications\":[\"All\"]},\"signInRiskLevels\":[\"medium\",\"high\"]},\"grantControls\":{\"operator\":\"OR\",\"builtInControls\":[\"mfa\"]}}"

# User risk policy: require password change for high user risk
az rest --method POST \
  --url "https://graph.microsoft.com/v1.0/identity/conditionalAccess/policies" \
  --body "{\"displayName\":\"IP-Lab-UserRisk-PasswordChange\",\"state\":\"enabled\",\"conditions\":{\"users\":{\"includeGroups\":[\"$RISK_GROUP_ID\"]},\"applications\":{\"includeApplications\":[\"All\"]},\"userRiskLevels\":[\"high\"]},\"grantControls\":{\"operator\":\"AND\",\"builtInControls\":[\"mfa\",\"passwordChange\"]}}"

# Review risky users and detections
az rest --method GET --url "https://graph.microsoft.com/v1.0/identityProtection/riskyUsers" --query "value[].{User:userDisplayName,Risk:userRiskLevel,RiskState:riskState}" -o table
az rest --method GET --url "https://graph.microsoft.com/v1.0/identityProtection/riskDetections" --query "value[].{User:userDisplayName,Type:riskEventType,Risk:riskLevel}" -o table

# Example remediation: dismiss risk
az rest --method POST --url "https://graph.microsoft.com/v1.0/identityProtection/riskyUsers/dismiss" --body "{\"userIds\":[\"$RISK_USER_ID\"]}"
```

#### PowerShell (Microsoft Graph)
```powershell
Connect-MgGraph -Scopes "Policy.ReadWrite.ConditionalAccess","IdentityRiskyUser.ReadWrite.All","IdentityRiskEvent.Read.All","User.ReadWrite.All","Group.ReadWrite.All"

$group = New-MgGroup -DisplayName 'identity-protection-pilot-ps' -MailNickname 'identity-protection-pilot-ps' -SecurityEnabled:$true -MailEnabled:$false
$user = New-MgUser -DisplayName 'Risk User 2' -AccountEnabled:$true -MailNickname 'riskuser2' -UserPrincipalName 'riskuser2@<tenant>.onmicrosoft.com' -PasswordProfile @{ password = 'P@ssw0rd1234!'; forceChangePasswordNextSignIn = $true }
New-MgGroupMemberByRef -GroupId $group.Id -BodyParameter @{ '@odata.id' = "https://graph.microsoft.com/v1.0/directoryObjects/$($user.Id)" }

# Sign-in risk policy
$signInRiskPolicy = @{
    displayName = 'IP-Lab-SignInRisk-MFA-PS'
    state = 'enabled'
    conditions = @{
        users = @{ includeGroups = @($group.Id) }
        applications = @{ includeApplications = @('All') }
        signInRiskLevels = @('medium','high')
    }
    grantControls = @{ operator = 'OR'; builtInControls = @('mfa') }
} | ConvertTo-Json -Depth 10
$signInPolicy = Invoke-MgGraphRequest -Method POST -Uri 'https://graph.microsoft.com/v1.0/identity/conditionalAccess/policies' -Body $signInRiskPolicy

# User risk policy
$userRiskPolicy = @{
    displayName = 'IP-Lab-UserRisk-PasswordChange-PS'
    state = 'enabled'
    conditions = @{
        users = @{ includeGroups = @($group.Id) }
        applications = @{ includeApplications = @('All') }
        userRiskLevels = @('high')
    }
    grantControls = @{ operator = 'AND'; builtInControls = @('mfa','passwordChange') }
} | ConvertTo-Json -Depth 10
$userRiskCa = Invoke-MgGraphRequest -Method POST -Uri 'https://graph.microsoft.com/v1.0/identity/conditionalAccess/policies' -Body $userRiskPolicy

# Review risky users and detections
Invoke-MgGraphRequest -Method GET -Uri 'https://graph.microsoft.com/v1.0/identityProtection/riskyUsers'
Invoke-MgGraphRequest -Method GET -Uri 'https://graph.microsoft.com/v1.0/identityProtection/riskDetections'

# Example remediation: confirm compromised
Invoke-MgGraphRequest -Method POST -Uri 'https://graph.microsoft.com/v1.0/identityProtection/riskyUsers/confirmCompromised' -Body (@{ userIds = @($user.Id) } | ConvertTo-Json)
```

### Verification Steps
- Sign in as the pilot user from Tor Browser or another anonymous IP source.
- Review the **Risk detections** and **Risky users** blade in Entra admin center.
- Validate that the sign-in risk policy challenges for MFA and the user risk policy forces password reset when applicable.

```powershell
Invoke-MgGraphRequest -Method GET -Uri 'https://graph.microsoft.com/v1.0/identityProtection/riskyUsers'
Invoke-MgGraphRequest -Method GET -Uri 'https://graph.microsoft.com/v1.0/identityProtection/riskDetections'
```

### Cleanup
```bash
SIGNIN_POLICY_ID=$(az rest --method GET --url "https://graph.microsoft.com/v1.0/identity/conditionalAccess/policies" --query "value[?displayName=='IP-Lab-SignInRisk-MFA'].id | [0]" -o tsv)
USERRISK_POLICY_ID=$(az rest --method GET --url "https://graph.microsoft.com/v1.0/identity/conditionalAccess/policies" --query "value[?displayName=='IP-Lab-UserRisk-PasswordChange'].id | [0]" -o tsv)
az rest --method DELETE --url "https://graph.microsoft.com/v1.0/identity/conditionalAccess/policies/$SIGNIN_POLICY_ID"
az rest --method DELETE --url "https://graph.microsoft.com/v1.0/identity/conditionalAccess/policies/$USERRISK_POLICY_ID"
az ad user delete --id $RISK_USER_UPN
az ad group delete --group $RISK_GROUP_ID
```

```powershell
Invoke-MgGraphRequest -Method DELETE -Uri "https://graph.microsoft.com/v1.0/identity/conditionalAccess/policies/$($signInPolicy.id)"
Invoke-MgGraphRequest -Method DELETE -Uri "https://graph.microsoft.com/v1.0/identity/conditionalAccess/policies/$($userRiskCa.id)"
Remove-MgUser -UserId $user.Id
Remove-MgGroup -GroupId $group.Id
Disconnect-MgGraph
```

### Exam Tip
> **AZ-305 tip:** If the requirement mentions leaked credentials, impossible travel, anonymous IPs, or automated identity threat response, look for **Identity Protection** and **risk-based Conditional Access**.

---

## Decision Summary

| Scenario | Best Entra feature | Why it fits | Licensing notes |
|---|---|---|---|
| Require MFA only for risky or app-specific sign-ins | Conditional Access | Policy-based control by user, app, location, device, or risk | Entra ID P1 |
| Improve MFA enrollment and method readiness | MFA registration policy + authentication methods policy | Separates enrollment from enforcement and standardizes methods | Included / some controls with P1 |
| Azure workload needs Key Vault or Storage access without secrets | Managed Identity | Removes credential storage and supports RBAC | No extra Entra premium required |
| Admins need just-in-time privileged access | PIM | Reduces standing privilege and adds approval/MFA | Entra ID P2 |
| External partners need controlled access to apps/groups | B2B Guest Collaboration | Lets guests use home credentials while you govern authorization | Core feature; advanced governance may require P2/Identity Governance |
| Non-Azure or multi-tenant app needs app-only auth | App Registration + Service Principal | Supports delegated/app permissions and Azure RBAC | No premium required |
| On-prem AD must sync to cloud identity | Hybrid Identity with Entra Connect | Unifies identity across on-prem and cloud | Core feature; extras may require P1/P2 |
| Need automated response to risky sign-ins/users | Identity Protection | Detects and remediates identity threats using risk signals | Entra ID P2 |

> **Final AZ-305 reminder:** Default to **least privilege**, **pilot groups**, **report-only/testing**, and **managed identities over secrets** whenever the scenario allows it.
