> 📝 **Hands-On Labs:** [Backup Labs](./Labs/Azure-Backup-Labs.md)

# Azure Backup Cheat Sheet + AZ-305 Exam Prep

**Audience:** Senior Cloud Solution Architect  
**Primary exam domain:** Design business continuity solutions (15-20%)  
**Secondary domains:** Data storage; identity, governance, and monitoring  
**Use this for:** backup architecture decisions, vault selection, RPO/RTO mapping, security controls, restore design, and fast exam revision

> 🎯 **Senior Architect view:** AZ-305 backup questions are rarely about memorizing a menu path. They test whether you can design the **right protection model** for the workload while balancing **RPO, RTO, compliance, security, cost, and operations**.

---

## 1. Azure Backup Overview

### Backup vs disaster recovery

| Capability | Backup | Disaster Recovery |
|---|---|---|
| Primary goal | Recover from deletion, corruption, ransomware, or point-in-time data loss | Keep the application available during site/region failure |
| Best metric | **RPO** and retention | **RTO** and failover capability |
| Typical tools | Azure Backup, SQL PITR/LTR, snapshots | Azure Site Recovery, failover groups, geo-replication, Front Door |
| Recovery target | Data, VM, file, database, disk, cluster state | Entire running service/app stack |
| Exam shortcut | **Backup restores data** | **DR fails over services** |

> 💡 **AZ-305 heuristic:** If the scenario says **accidental deletion, corruption, ransomware, audit retention, or point-in-time restore**, think **backup** first. If it says **keep the app running during a regional outage**, think **DR/replication/failover** first.

### Recovery Services Vault vs Backup Vault

| Item | Recovery Services Vault (RSV) | Backup Vault |
|---|---|---|
| Best known for | Traditional Azure Backup workloads and Site Recovery metadata | Newer Azure Data Protection workloads |
| Common protected workloads | Azure VM, Azure Files, SQL Server in Azure VM, SAP HANA in Azure VM, hybrid/on-prem backup agents | Azure Blobs, Azure Managed Disks, AKS, Azure Database for PostgreSQL (where supported) |
| Control plane | `az backup` / Az.RecoveryServices | `az dataprotection` / Az.DataProtection |
| Security features | Soft delete, immutability, MUA/Resource Guard, private endpoints | Vault security, identities, Resource Guard support depending on workload |
| Exam mindset | Default answer for classic Azure Backup questions | Default answer for newer Data Protection workload questions |

### Supported workloads matrix

| Workload | Protection model | Primary vault/service | Key AZ-305 note |
|---|---|---|---|
| Azure Virtual Machines | Policy-based VM backup | **Recovery Services Vault** | Backup is not DR; use Site Recovery for failover |
| SQL Server in Azure VM | Workload-aware backup | **Recovery Services Vault** | Log backups drive lower RPO |
| SAP HANA in Azure VM | Workload-aware backup | **Recovery Services Vault** | App-aware recovery, not just VM image restore |
| Azure Files | Share snapshot backup | **Recovery Services Vault** | Item-level file restore is a common exam trigger |
| Azure SQL Database / SQL Managed Instance | Built-in automated backups, PITR, LTR | **Native SQL service** | Not protected through RSV |
| Azure Blobs | Operational backup / vaulted backup | **Backup Vault** | Soft delete is not the same as backup |
| Azure Managed Disks | Disk backup | **Backup Vault** | Lower-cost alternative when VM-level protection is unnecessary |
| Azure Database for PostgreSQL Flexible Server | Native backup and Azure Backup options | **Native service and/or Backup Vault** | Clarify whether exam wants PITR only or centralized backup governance |
| AKS | CSI-based backup | **Backup Vault** | Protect persistent volumes/Kubernetes objects, not full cluster failover |
| On-premises files/servers | MARS/MABS/DPM to Azure | **Recovery Services Vault** | Hybrid backup = backup, not full application DR |

### Quick commands

```bash
az backup vault list -o table
az dataprotection backup-vault list -o table
```

```powershell
Get-AzRecoveryServicesVault
Get-AzDataProtectionBackupVault
```

---

## 2. Recovery Services Vault

Recovery Services Vault is the central management plane for many Azure Backup workloads and for Site Recovery metadata.

### Vault redundancy options

| Redundancy | Best for | Tradeoff |
|---|---|---|
| **LRS** | Lowest cost, single-region backup durability | No cross-region restore |
| **ZRS** | In-region zonal resilience | No cross-region restore |
| **GRS** | Highest backup durability with geo-copy | Higher cost; paired-region dependency |

**Exam fact:** To use **cross-region restore (CRR)** for supported workloads, you need **GRS** vault storage and CRR enabled.

### Cross-region restore

- Restores from the secondary paired region when the primary region is unavailable or when CRR is allowed for the workload.
- Excellent for **backup-based regional recovery**, but still slower than full DR/failover services.
- Common trap: **CRR ≠ active-active** and **CRR ≠ Azure Site Recovery**.

### Soft delete

- Protects deleted backup items/recovery points from immediate purge.
- **Default retention is 14 days** and can be increased **up to 180 days**.
- Architect takeaway: keep it enabled; disabling to save cost is usually the wrong design answer.

### Multi-user authorization (MUA)

- Uses **Resource Guard** for approval on critical operations.
- Designed to reduce risk from malicious or mistaken admin actions.
- Strong exam answer when the scenario says **ransomware**, **separation of duties**, or **backup admin compromise**.

### Private endpoints for backup

- Use when the organization requires **private connectivity**, no public exposure, or strict exfiltration controls.
- Especially relevant for regulated environments with central backup governance.

### Immutable vaults

- Protect recovery points from deletion/modification for the immutability period.
- Best answer when the requirement says **WORM**, **ransomware resilience**, **tamper resistance**, or **compliance retention**.
- **Locked immutability** should be treated as a deliberate governance decision.

### Architect decisions that matter

- Choose redundancy **before** large-scale onboarding; changing storage redundancy later can be constrained once items are protected.
- Use **GRS + CRR** only when backup-based geo-restore is a real requirement; do not pay for it automatically everywhere.
- Combine **soft delete + MUA + immutability + RBAC** for high-confidence ransomware protection.

### Common commands

```bash
az backup vault create \
  --resource-group rg-backup \
  --name rsv-prod-eastus \
  --location eastus

az backup vault backup-properties set \
  --resource-group rg-backup \
  --name rsv-prod-eastus \
  --backup-storage-redundancy GeoRedundant \
  --cross-region-restore-flag True \
  --soft-delete-feature-state AlwaysOn \
  --soft-delete-duration 90

az backup vault backup-properties show \
  --resource-group rg-backup \
  --name rsv-prod-eastus
```

```powershell
$vault = New-AzRecoveryServicesVault -Name "rsv-prod-eastus" -ResourceGroupName "rg-backup" -Location "EastUS"
Set-AzRecoveryServicesVaultContext -Vault $vault
Get-AzRecoveryServicesVault -ResourceGroupName "rg-backup" -Name "rsv-prod-eastus"
```

---

## 3. Azure VM Backup

Azure VM Backup protects entire VMs using policy-driven backup with snapshot and vault integration.

### Application-consistent vs crash-consistent vs file-consistent snapshots

| Type | Meaning | Best use |
|---|---|---|
| **Application-consistent** | Uses VSS or pre/post scripts to flush data and memory-aware app state | Databases and transactional apps |
| **Crash-consistent** | Equivalent to pulling power from the VM; disk state is preserved without app quiescing | Stateless or tolerant workloads |
| **File-consistent** | File system is consistent, but app transaction state may not be | Linux/legacy scenarios where app consistency is unavailable |

**Exam rule:** If the app is transactional and the question emphasizes data integrity, pick **application-consistent** where supported.

### Backup policies

- Define **frequency**, **time**, **instant restore retention**, and **longer-term retention**.
- Standard policy is typically enough for daily protection.
- **Enhanced policy** is the key answer when the scenario requires **multiple backups per day**, **hourly RPO**, or newer VM/security capabilities.

### Instant restore (snapshot tier)

- Uses the snapshot tier for fast restore before the backup fully transitions to vault storage.
- Best fit for low RTO operational restores.
- Tradeoff: snapshot retention improves restore speed but increases cost.

### Selective disk backup

- Exclude non-critical data disks to reduce protected instance/storage cost.
- Strong answer when the workload has ephemeral, replicated, or rebuildable disks.
- Do **not** exclude a disk that is required for application recovery just to save money.

### Cross-region restore

- Supported when the vault uses **GRS** and **CRR** is enabled.
- Useful for recovering a VM backup into the paired region.
- Still not a substitute for continuous replication and orchestrated failover.

### Enhanced backup policy

- Better fit for **hourly scheduling**, improved operational flexibility, and newer features.
- Common exam trigger: “RPO must be under 24 hours for VM backups without deploying ASR.”

### Trusted launch VMs backup

- Supported with the appropriate backup model/policy support.
- Exam takeaway: when the requirement says **Secure Boot**, **vTPM**, or **Trusted Launch**, verify backup compatibility and choose the policy that supports it.

### Commands

```bash
az backup policy get-default-for-vm

az backup protection enable-for-vm \
  --resource-group rg-backup \
  --vault-name rsv-prod-eastus \
  --vm vm-app-01 \
  --policy-name DailyPolicy

az backup protection backup-now \
  --resource-group rg-backup \
  --vault-name rsv-prod-eastus \
  --container-name iaasvmcontainer;iaasvmcontainerv2;rg-app;vm-app-01 \
  --item-name vm-app-01 \
  --retain-until 31-12-2026

az backup recoverypoint list \
  --resource-group rg-backup \
  --vault-name rsv-prod-eastus \
  --backup-management-type AzureIaasVM \
  --container-name iaasvmcontainer;iaasvmcontainerv2;rg-app;vm-app-01 \
  --item-name vm-app-01
```

```powershell
$container = Get-AzRecoveryServicesBackupContainer -ContainerType AzureVM -Status Registered
$item = Get-AzRecoveryServicesBackupItem -Container $container[0] -WorkloadType AzureVM
Get-AzRecoveryServicesBackupRecoveryPoint -Item $item[0] -StartDate (Get-Date).AddDays(-7) -EndDate (Get-Date)
```

> 💡 **AZ-305 heuristic:** If the business says **“restore the VM fast”**, think **instant restore**. If it says **“fail over the workload fast”**, think **Azure Site Recovery**.

---

## 4. Azure SQL Backup

### SQL Database automated backup (PITR)

Azure SQL Database and SQL Managed Instance include automated backups by design.

- **PITR** supports restore to a specific point in time within the retention window.
- Retention is commonly **7 to 35 days**, depending on configuration/service tier.
- SQL service handles full, differential, and log backups under the platform.

### Long-term retention (LTR)

- Extends retention for weekly/monthly/yearly backups.
- Good answer for **audit**, **financial**, **medical**, or **legal hold** style scenarios.
- Can retain backups for **years** without building your own vault process.

### Backup storage redundancy

- Azure SQL supports backup storage redundancy choices such as **local**, **zone**, or **geo** depending on service/region support.
- If the scenario says **regional outage + restore from backup**, prefer **geo-redundant backup storage**.

### SQL Server in VM backup (workload backup)

- Protects SQL at the workload level inside the VM.
- Better than raw VM backup when the requirement is **transaction-log-aware restore** or **database-level recovery**.
- Frequently a better design than VM-only backup for production SQL workloads.

### Backup frequency and RPO

| Workload | Typical backup model | RPO design point |
|---|---|---|
| **Azure SQL Database / MI** | Automated full/diff/log | Minutes, service-managed |
| **SQL Server in Azure VM** | Full + differential + log backup policy | Can be much lower than daily VM backup; log backups often every 15 minutes |

### Commands

```bash
az sql db restore \
  --resource-group rg-data \
  --server sqlprod01 \
  --name appdb \
  --dest-name appdb-restore \
  --time "2026-01-15T14:30:00Z"

az sql db ltr-policy set \
  --resource-group rg-data \
  --server sqlprod01 \
  --name appdb \
  --weekly-retention P12W \
  --monthly-retention P12M \
  --yearly-retention P7Y \
  --week-of-year 1
```

```powershell
Restore-AzSqlDatabase -FromPointInTimeBackup -PointInTime "2026-01-15T14:30:00Z" -ResourceGroupName "rg-data" -ServerName "sqlprod01" -TargetDatabaseName "appdb-restore"
Set-AzSqlDatabaseBackupLongTermRetentionPolicy -ResourceGroupName "rg-data" -ServerName "sqlprod01" -DatabaseName "appdb" -WeeklyRetention "P12W" -MonthlyRetention "P12M" -YearlyRetention "P7Y" -WeekOfYear 1
```

> 💡 **Exam shortcut:** **Azure SQL Database/MI backup is built in**. **SQL Server in Azure VM** is where Recovery Services Vault workload backup matters most.

---

## 5. Azure Files Backup

Azure Files backup uses **share snapshots** coordinated by Azure Backup.

### Share snapshots

- Recovery points are snapshot-based.
- Fast for operational recovery.
- Great for file-share scenarios where users accidentally delete or overwrite files.

### Backup policies

- Configure daily backup schedules and retention.
- Good fit when the organization wants centralized policy governance across multiple storage accounts.

### Item-level recovery

- A common exam differentiator.
- You can restore individual files/folders instead of the entire share.
- Strong answer when the requirement is **low-impact restore for a small subset of data**.

### Architect tradeoffs

- Azure Files backup is easy to operationalize, but snapshot retention and storage growth must be cost-managed.
- For large file workloads, align retention with business value; not every file share needs long yearly retention.

### Commands

```bash
az backup protection enable-for-azurefileshare \
  --resource-group rg-backup \
  --vault-name rsv-prod-eastus \
  --policy-name FilesDailyPolicy \
  --storage-account stfilesprod01 \
  --azure-file-share finance-share

az backup item list \
  --resource-group rg-backup \
  --vault-name rsv-prod-eastus \
  --backup-management-type AzureStorage \
  --workload-type AzureFileShare -o table
```

```powershell
Get-AzRecoveryServicesBackupItem -WorkloadType AzureFiles
Get-AzRecoveryServicesBackupRecoveryPoint -Item $fileItem -StartDate (Get-Date).AddDays(-30) -EndDate (Get-Date)
```

---

## 6. Azure Blob Backup

Azure Blob protection has two major design patterns.

### Operational backup (continuous)

- Designed for **continuous protection** and fast operational recovery.
- Works with blob service capabilities such as **versioning**, **change feed**, and **point-in-time restore**.
- Best answer when the workload needs frequent restore points with lower operational friction.

### Vaulted backup (periodic)

- Uses Backup Vault/Data Protection style backup for longer-term retention.
- Better when the scenario emphasizes **central governance**, **vault isolation**, or **longer retention** beyond operational rollback needs.

### Point-in-time restore

- Restores container/account data to a previous point within the supported retention window.
- Best fit when accidental overwrite/delete must be reversed quickly.

### Soft delete vs backup

| Capability | Soft delete | Backup |
|---|---|---|
| Recovers deleted data | Yes | Yes |
| Protection from overwrite/corruption across time | Limited | Better |
| Retention governance | Simpler | Stronger |
| Isolation from compromised account settings | Limited | Better with vault-based controls |

**Exam fact:** If the question says **regulatory retention**, **isolated copies**, or **centralized protection**, soft delete alone is not enough.

### Commands

```bash
az storage account blob-service-properties update \
  --account-name stblobprod01 \
  --resource-group rg-storage \
  --enable-versioning true \
  --enable-change-feed true \
  --enable-delete-retention true \
  --delete-retention-days 30 \
  --enable-restore-policy true \
  --restore-days 14

az storage account blob-service-properties show \
  --account-name stblobprod01 \
  --resource-group rg-storage
```

```powershell
Update-AzStorageBlobServiceProperty -ResourceGroupName "rg-storage" -StorageAccountName "stblobprod01" -EnableVersioning $true -EnableChangeFeed $true -EnableDeleteRetentionPolicy $true -DeleteRetentionDays 30
Get-AzStorageBlobServiceProperty -ResourceGroupName "rg-storage" -StorageAccountName "stblobprod01"
```

---

## 7. Other Workload Backup

### SAP HANA backup

- Use workload-aware backup in **Recovery Services Vault**.
- Strong answer when app consistency and database-aware restore are mandatory.
- Better than relying only on VM-level backup for enterprise SAP data protection.

### Azure Database for PostgreSQL

- Native service backups already provide restore capability.
- Azure Backup/Data Protection may be chosen when centralized vault governance and policy separation are required.
- Architect question: **Is native PITR enough, or does the enterprise require centralized backup governance?**

### Azure Kubernetes Service backup

- AKS backup protects Kubernetes resources and supported CSI-backed persistent volumes.
- Backup helps recover cluster state/data, but **multi-region AKS continuity** still needs DR architecture beyond backup.

### Azure Managed Disks backup

- Best fit when the requirement is **disk-level protection** without full VM backup overhead.
- Useful for cost optimization or app patterns where restoring a disk is enough.

### Commands

```bash
az backup protection auto-enable-for-azurewl \
  --resource-group rg-backup \
  --vault-name rsv-prod-eastus \
  --policy-name SQLPolicy15MinLogs \
  --server-name vm-sql-01 \
  --protectable-item-name MSSQLSERVER \
  --protectable-item-type SQLInstance \
  --workload-type MSSQL

az postgres flexible-server restore \
  --resource-group rg-data \
  --name pgflex-restore01 \
  --source-server pgflex-prod01 \
  --restore-time "2026-01-15T14:30:00+00:00"

az dataprotection backup-vault list -o table
```

```powershell
Get-AzRecoveryServicesBackupItem -WorkloadType MSSQL
Get-AzRecoveryServicesBackupItem -WorkloadType SAPHANA
Get-AzPostgreSqlFlexibleServer
```

---

## 8. Backup Policies and Retention

### Daily, weekly, monthly, yearly retention

Retention should map to real business needs:

- **Daily** -> operational restore and recent rollback
- **Weekly** -> short-term business continuity and reporting cycles
- **Monthly** -> audit and monthly close support
- **Yearly** -> legal, tax, or regulatory retention

### GFS (Grandfather-Father-Son) scheme

| Tier | Purpose | Example |
|---|---|---|
| **Son** | Daily restore points | Retain 30 daily backups |
| **Father** | Weekly restore points | Retain 12 weekly backups |
| **Grandfather** | Monthly/yearly restore points | Retain 12 monthly + 7 yearly backups |

**Why examiners like GFS:** it balances operational recovery with compliance retention and avoids keeping every daily backup forever.

### Compliance requirements mapping

| Requirement | Good design pattern |
|---|---|
| 30-90 day operational recovery | Daily backups + short weekly retention |
| 1 year audit history | Monthly retention + immutable controls |
| 7-10 year legal/financial retention | Yearly retention + immutability + approval controls |
| Ransomware-resistant retention | Soft delete + immutable vault + MUA |

### Cost optimization strategies

- Use **LRS** unless geo-restore is required.
- Use **ZRS** when zonal resilience is needed but geo-restore is not.
- Use **GRS** selectively for high-value workloads.
- Limit **instant restore snapshot retention** to what the RTO truly needs.
- Use **selective disk backup** for non-critical disks.
- Avoid over-retention on dev/test workloads.

### Commands

```bash
az backup policy list \
  --resource-group rg-backup \
  --vault-name rsv-prod-eastus -o table

az backup policy show \
  --resource-group rg-backup \
  --vault-name rsv-prod-eastus \
  --name DailyPolicy
```

```powershell
Get-AzRecoveryServicesBackupProtectionPolicy
```

> 💡 **AZ-305 heuristic:** If the question says **compliance** and **minimum cost**, do not jump straight to daily-for-10-years. Use a **GFS retention design**.

---

## 9. Restore Operations

### Original location vs alternate location

| Restore target | Best use |
|---|---|
| **Original location** | Fast operational recovery when you want the workload back exactly where it was |
| **Alternate location** | Validation, forensics, side-by-side restore, or when original infrastructure is unavailable |

### Cross-region restore

- Backup-based recovery to the paired secondary region.
- Best when the requirement is **recoverable in another region**, not necessarily **fail over immediately**.

### Item-level recovery

- Strong answer for Azure Files and some workload-aware backups.
- Lowest operational impact when only a subset of data is needed.

### File recovery from VM backup

- Useful when the business needs only a few files, not a full VM restore.
- Typical exam distinction: **file recovery** is faster/cheaper than rebuilding an entire VM.

### Restore points management

- Know where restore points live: snapshot tier, vault tier, or service-native backup storage.
- Move/archive recovery points only when retention and restore-time tradeoffs are acceptable.

### Commands

```bash
az backup recoverypoint show \
  --resource-group rg-backup \
  --vault-name rsv-prod-eastus \
  --backup-management-type AzureIaasVM \
  --container-name iaasvmcontainer;iaasvmcontainerv2;rg-app;vm-app-01 \
  --item-name vm-app-01 \
  --name <recovery-point-name>
```

```powershell
Get-AzRecoveryServicesBackupRecoveryPoint -Item $item[0] -StartDate (Get-Date).AddDays(-14) -EndDate (Get-Date)
```

---

## 10. Backup Security

### Soft delete protection

- First protection layer against accidental/malicious delete.
- Always a good baseline answer.

### Multi-user authorization (MUA)

- Adds approval protection for critical backup operations.
- Pair with **separate subscriptions/resource groups** for better blast-radius control.

### Immutable vaults

- Best answer for ransomware and tamper-resistant retention.
- Particularly strong for finance, healthcare, and regulated workloads.

### Private endpoints

- Use for restricted network paths and private access models.
- Good answer when security policy disallows public endpoints.

### Encryption (platform vs customer-managed keys)

| Option | Use when |
|---|---|
| **Platform-managed keys (PMK)** | Default answer for most workloads |
| **Customer-managed keys (CMK)** | Requirement says customer controls keys, separation of duties, or external key governance |

### Security design pattern to remember

**Good exam answer for ransomware:**
1. Recovery Services Vault or Backup Vault with least privilege RBAC
2. Soft delete enabled
3. MUA/Resource Guard for destructive actions
4. Immutable vault for protected recovery points
5. Private endpoints where required
6. Monitoring and alerts for backup disable/delete attempts

### Commands

```bash
az backup vault backup-properties set \
  --resource-group rg-backup \
  --name rsv-prod-eastus \
  --soft-delete-feature-state AlwaysOn \
  --soft-delete-duration 180
```

```powershell
Set-AzRecoveryServicesBackupProperty -VaultId $vault.ID -SoftDeleteFeatureState AlwaysON
```

---

## 11. Backup Monitoring and Reporting

### Backup center

- Single pane of glass for backup posture across vaults/subscriptions.
- Useful when the exam says **central operations**, **enterprise governance**, or **backup posture dashboard**.

### Backup reports (Log Analytics)

- Use Log Analytics for trend analysis, compliance reporting, failed jobs, protected item counts, and storage growth.
- Strong answer when the requirement includes **reporting**, **operations dashboards**, or **executive visibility**.

### Alerts configuration

- Configure job failure alerts and operational alerting.
- Do not assume backup failures are always acted on unless alerts are designed and routed.

### Azure Monitor integration

- Diagnostic settings from vaults to Log Analytics.
- Azure Monitor alerts + Action Groups for notification/automation.
- KQL for backup analytics.

### Commands

```bash
az monitor diagnostic-settings create \
  --name backup-diag \
  --resource /subscriptions/<subId>/resourceGroups/rg-backup/providers/Microsoft.RecoveryServices/vaults/rsv-prod-eastus \
  --workspace /subscriptions/<subId>/resourceGroups/rg-monitor/providers/Microsoft.OperationalInsights/workspaces/law-prod \
  --logs '[{"category":"AzureBackupReport","enabled":true},{"category":"AddonAzureBackupJobs","enabled":true},{"category":"AddonAzureBackupAlerts","enabled":true}]'

az monitor log-analytics query \
  --workspace /subscriptions/<subId>/resourceGroups/rg-monitor/providers/Microsoft.OperationalInsights/workspaces/law-prod \
  --analytics-query "AddonAzureBackupJobs | where JobStatus == 'Failed' | project TimeGenerated, BackupItemFriendlyName, Operation, ErrorCode"
```

```powershell
Get-AzRecoveryServicesBackupJob -VaultId $vault.ID -Status Failed
```

```kusto
AddonAzureBackupJobs
| where TimeGenerated > ago(24h)
| summarize Failures=countif(JobStatus == "Failed"), Successes=countif(JobStatus == "Completed") by BackupManagementType
| order by Failures desc
```

---

## 12. AZ-305 Decision Scenarios

| Scenario | Best answer | Why |
|---|---|---|
| 1. A production VM must recover quickly from accidental file deletion | **Azure VM Backup with instant restore + file recovery** | Faster and cheaper than full VM rebuild |
| 2. A finance workload needs 7-year retention and tamper resistance | **LTR/GFS + immutability + MUA** | Compliance + ransomware resilience |
| 3. A regional outage must still allow restoring a critical VM backup | **GRS Recovery Services Vault + CRR** | Backup-based regional recovery |
| 4. The company wants app failover in minutes, not backup restore in hours | **Azure Site Recovery / replication-based DR** | Backup alone will not meet low RTO |
| 5. SQL Server on Azure VM needs 15-minute RPO | **SQL workload backup with log backups** | Better than once-per-day VM backup |
| 6. A storage team only needs rollback for accidental blob changes | **Blob operational backup / versioning + PITR** | Fast operational recovery |
| 7. Central security requires approval before disabling backup | **MUA with Resource Guard** | Protects critical destructive actions |
| 8. A file share needs single-folder restore without disrupting users | **Azure Files backup with item-level recovery** | Minimizes operational impact |
| 9. Dev/test VMs need cheap backup but no geo-restore | **LRS vault + short retention** | Meets requirement at lowest cost |
| 10. An AKS workload needs persistent volume recovery, not full multi-region failover | **AKS backup in Backup Vault** | Backup for data/state; DR still separate |

### Scenario-based exam review questions

1. A workload requires **backup-based recovery in another region**, but not always-on failover. Which vault redundancy option best fits and why?  
2. A SQL Server running in Azure VM needs **15-minute RPO** and **database-level restore**. Why is VM backup alone a poor answer?  
3. A regulated file workload must be recoverable for **7 years** and protected from admin tampering. Which controls would you combine?  
4. An app team asks for **point-in-time restore** after an accidental blob overwrite. Why is soft delete alone often insufficient?  
5. A company says “we need the app online in 5 minutes if East US fails.” Why should you challenge a design that only proposes Azure Backup?

---

## 13. Quick Reference Trigger Table

| Trigger phrase in the question | Think first | Why |
|---|---|---|
| Accidental deletion | **Backup** | HA does not restore deleted data |
| Point-in-time restore | **Backup/PITR** | Need time-based recovery |
| Minimal data loss | **Low RPO** | Frequency/log backups matter |
| Minimal downtime | **Low RTO / DR** | Restore speed or failover matters |
| Regional restore from backup | **GRS + CRR** | LRS/ZRS cannot do cross-region restore |
| Tamper-resistant backups | **Immutable vault** | Protects recovery points |
| Approval for destructive actions | **MUA / Resource Guard** | Separation of duties |
| Back up Azure VM | **Recovery Services Vault** | Classic Azure Backup workload |
| Back up SQL in VM | **Recovery Services Vault workload backup** | Log-aware restore |
| Back up Azure Files | **Recovery Services Vault** | Snapshot + item restore |
| Back up Azure Blobs | **Backup Vault / operational backup** | Newer data protection model |
| Back up AKS PVs | **Backup Vault** | CSI-based Kubernetes protection |
| Back up Managed Disks only | **Backup Vault** | Cheaper than full VM in some cases |
| Native SQL backup | **PITR/LTR** | SQL Database/MI are service-managed |
| Long-term compliance retention | **LTR or yearly retention** | Not every backup needs daily forever |
| Ransomware recovery | **Soft delete + immutability + MUA** | Layered protection |
| Private-only backup access | **Private endpoints** | Network isolation |
| Fast operational VM restore | **Instant restore** | Snapshot tier improves RTO |
| Sub-hour VM backup frequency | **Enhanced VM backup policy** | Multiple backups per day |
| File-level restore | **Azure Files backup or VM file recovery** | Avoid full restore |
| Lowest-cost non-critical backup | **LRS + short retention** | Don’t over-design |
| Zonal resilience only | **ZRS** | Better than LRS, cheaper than GRS |
| Geo-backup requirement | **GRS** | Needed for paired-region durability |
| Restore whole app stack in another region | **DR, not backup only** | Failover question |
| Audit/reporting on backup posture | **Backup Center + Log Analytics** | Central visibility |
| Existing disk is the only thing that matters | **Managed Disk backup** | VM-level backup may be overkill |

---

## 14. Common Exam Traps

1. **Backup is not disaster recovery.** Azure Backup restores data; it does not keep the application running during an outage.
2. **Recovery Services Vault is not the answer for every workload.** Newer workloads such as blobs, disks, and AKS often point to **Backup Vault**.
3. **Azure SQL Database/Managed Instance backups are built in.** Do not force an RSV answer where native PITR/LTR is the right one.
4. **GRS costs more for a reason.** If the scenario does not require geo-restore, LRS or ZRS may be the better architect answer.
5. **CRR is still backup restore, not failover.** It improves recoverability, not application availability.
6. **Soft delete is not the same as immutable backup.** Soft delete helps, but immutability is stronger for ransomware/compliance.
7. **VM backup is not enough for low-RPO SQL workloads.** Use SQL workload backup when log-aware recovery is required.
8. **Blob soft delete/versioning are not always sufficient.** If the scenario demands isolated retention/governance, backup is stronger.
9. **Do not over-retain by default.** Compliance questions often want **GFS**, not “daily backups for 10 years.”
10. **File-level recovery can be the best answer.** Rebuilding an entire VM/share is often unnecessary when only a few files are needed.
11. **Enhanced VM policy matters.** If multiple backups per day are needed, standard daily VM backup is often the wrong choice.
12. **Private endpoints and MUA are design signals.** When the question stresses security or insider risk, add them deliberately.

### Final architect checklist

- What is the workload: VM, SQL in VM, Azure Files, Blob, Disk, AKS, PostgreSQL, or native PaaS database?  
- Is the requirement for **backup**, **DR**, or both?  
- What are the target **RPO**, **RTO**, and **retention** values?  
- Does the organization need **geo-restore**, **immutability**, **approval workflow**, or **private access**?  
- Can you meet the requirement with **native platform backup** instead of adding unnecessary vault complexity?  
- Are you paying for **GRS**, long retention, or snapshot retention only where the business actually needs it?
