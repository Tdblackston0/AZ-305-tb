# Azure Backup - AZ-305 Comprehensive Cheat Sheet

> 📝 **Hands-On Labs:** [Backup Labs](./Labs/Azure-Backup-Labs.md)

> 🎯 **Exam Focus:** AZ-305 tests your ability to **design** backup solutions with appropriate RPO, retention, security, and compliance requirements.

## Table of Contents

- [1. Azure Backup Family Overview](#1-azure-backup-family-overview)
- [2. When to Choose Which — Decision Tree](#2-when-to-choose-which-decision-tree)
- [3. Recovery Services Vault](#3-recovery-services-vault)
- [4. Azure VM Backup](#4-azure-vm-backup)
- [5. Azure SQL Backup](#5-azure-sql-backup)
- [6. Azure Files Backup](#6-azure-files-backup)
- [7. Azure Blob Backup](#7-azure-blob-backup)
- [8. Other Workload Backup](#8-other-workload-backup)
- [9. Backup Policies and Retention](#9-backup-policies-and-retention)
- [10. Availability & Resilience](#10-availability-resilience)
- [11. Cost Optimization](#11-cost-optimization)
- [12. Restore Operations](#12-restore-operations)
- [13. Backup Security](#13-backup-security)
- [14. Backup Monitoring and Reporting](#14-backup-monitoring-and-reporting)
- [15. AZ-305 Decision Scenarios](#15-az-305-decision-scenarios)
- [16. Quick Reference Trigger Table](#16-quick-reference-trigger-table)
- [17. Common Exam Traps](#17-common-exam-traps)
- [18. 🎯 Final AZ-305 Exam Tips](#18-final-az-305-exam-tips)
- [19. 📐 Architecture Decision Flowchart](#19-architecture-decision-flowchart)
- [20. Exam-Style Review Questions](#20-exam-style-review-questions)

---

<a id="1-azure-backup-family-overview"></a>
## 1. Azure Backup Family Overview

### Comparison Table

| Capability | Recovery Services Vault | Backup Vault | VM Backup | SQL Backup | Azure Files Backup | Azure Blob Backup |
|---------|---------------------------|--------------|-----------|------------|--------------------|-------------------|
| **Primary purpose** | Central vault for classic Azure Backup workloads and Site Recovery metadata | Central vault for newer Azure Data Protection workloads | Policy-based protection for full Azure VMs | Database-aware protection for Azure SQL PaaS or SQL Server in Azure VM | Snapshot-based protection for Azure file shares | Operational or vaulted protection for blob data |
| **Typical control plane** | `az backup` / Az.RecoveryServices | `az dataprotection` / Az.DataProtection | Recovery Services Vault policies | Native SQL backups or Recovery Services Vault for SQL in VM | Recovery Services Vault | Backup Vault |
| **Best fit** | Traditional enterprise backup governance | Modern data protection for blobs, disks, AKS, PostgreSQL | Full-machine recovery and fast file restore | PITR, LTR, log backups, workload-aware restore | Share and item-level restore | Point-in-time blob recovery and isolated retention |
| **Low-RPO capability** | Depends on workload and policy | Depends on workload type | Better with enhanced policies, but not continuous DR | Strongest option when log backups/PITR are available | Good for share-level operational recovery | Strong for accidental overwrite/delete scenarios |
| **Typical restore granularity** | VM, file, database, share, app-aware item | Disk, blob, AKS object/PV, PostgreSQL item | Entire VM or selected files | Database, transaction-log point-in-time, long-term copy | Share, folder, file | Container/account point-in-time scope depending on design |
| **Common exam trigger** | “Back up Azure VM / Azure Files / SQL in VM” | “Protect blobs, disks, AKS, or modern data sources” | “Recover the VM or a few files quickly” | “Need lower RPO, PITR, or 7-year retention” | “Recover a folder without full share restore” | “Blob overwrite/delete with PITR and governance” |

### Recovery Services Vault
A **Recovery Services Vault (RSV)** is the classic Azure Backup control plane for Azure VMs, Azure Files, SQL Server in Azure VM, SAP HANA in Azure VM, and many hybrid workloads. It also stores Azure Site Recovery metadata, which makes it the default enterprise answer when the exam describes centralized backup administration for established Azure workloads.

**Real-World Examples:**
- A **banking platform** protects 200 production VMs with daily backups, instant restore, and strict RBAC from one central Recovery Services Vault.
- A **healthcare provider** backs up SQL Server running in Azure VMs with log backups to meet a 15-minute RPO for patient scheduling databases.
- A **hybrid manufacturer** uses MARS/MABS with a Recovery Services Vault to retain on-premises server backups in Azure for audit recovery.

### Backup Vault
A **Backup Vault** is the newer Azure Data Protection vault used for modern workloads such as Azure Blobs, Azure Managed Disks, AKS backups, and supported PostgreSQL scenarios. Use it when the requirement points to newer platform data protection capabilities rather than classic Azure Backup workflows.

**Real-World Examples:**
- A **media company** protects large blob datasets with vaulted backup so ransomware on the source account cannot immediately destroy retained recovery points.
- A **platform engineering team** backs up AKS persistent volumes and Kubernetes objects before cluster upgrades.
- A **SaaS provider** protects managed disks independently because only the data disks matter and full-VM backup would cost more.

### Azure VM Backup
**Azure VM Backup** captures full virtual machine protection using policy-driven snapshots plus vault retention. It is best when you need workload recovery for the VM as a machine, or quick file recovery from the VM backup without redesigning the application.

**Real-World Examples:**
- A **line-of-business app** on two Azure VMs needs daily backups plus instant restore for fast rollback after a failed patch.
- A **dev/test lab** protects non-critical VMs with LRS and short retention because rebuild speed matters more than geo-restore.
- A **retail branch server** uses enhanced VM backup policy to take multiple backups per day without deploying full DR tooling.

### Azure SQL Backup
**Azure SQL Backup** covers both native service-managed backups for Azure SQL Database/Managed Instance and workload-aware backup for SQL Server in Azure VM. It becomes the right design when the business requirement is framed in terms of **PITR, LTR, log backup frequency, or database-level recovery** rather than VM image recovery.

**Real-World Examples:**
- A **finance database** on Azure SQL Database uses PITR for 35 days and LTR for 7 years to satisfy audit retention requirements.
- An **ERP workload** running SQL Server in Azure VM relies on log backups every 15 minutes so the RPO is far lower than once-per-day VM backup.
- A **software vendor** restores one database to an alternate location for forensic validation after accidental data corruption.

### Azure Files Backup
**Azure Files Backup** protects file shares with share snapshots and Recovery Services Vault governance. It is the best answer when the exam emphasizes **item-level restore, user file recovery, or restoring folders without rebuilding a VM**.

**Real-World Examples:**
- A **legal department** restores a single folder from a share after a user accidentally deletes case files.
- A **regional office** protects shared departmental file shares with daily backups and monthly retention for audit support.
- A **construction firm** uses item-level recovery to restore project drawings without interrupting the full share.

### Azure Blob Backup
**Azure Blob Backup** protects object data with operational backup and/or vaulted backup depending on the recovery and governance requirement. It is usually the best answer when the scenario mentions **blob overwrite, accidental delete, point-in-time restore, or isolated retention for unstructured data**.

**Real-World Examples:**
- A **data lake team** uses blob point-in-time restore to recover from a bad ETL job that overwrote recent parquet files.
- An **insurance company** enables vaulted backup for claim documents so copies remain protected even if the source storage account is compromised.
- A **digital publishing platform** relies on operational backup for fast rollback of frequently updated media assets.

> 💡 **AZ-305 Tip:** Start by identifying the **workload type** first, then decide whether the design needs **classic backup governance (RSV)**, **modern data protection (Backup Vault)**, or **native service backup** such as Azure SQL PITR/LTR.

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

<a id="2-when-to-choose-which-decision-tree"></a>
## 2. When to Choose Which — Decision Tree

```
Start
 |
 +-- Is the requirement mainly backup of classic Azure workloads such as VMs,
 |   Azure Files, SQL Server in Azure VM, SAP HANA in Azure VM, or hybrid servers?
 |      |
 |      +-- YES --> Use Recovery Services Vault first
 |      |             |
 |      |             +-- Need full machine recovery or file recovery from a VM?
 |      |             |      +-- YES --> Azure VM Backup
 |      |             |
 |      |             +-- Need database-aware restore with low RPO?
 |      |             |      +-- YES --> SQL Server in Azure VM workload backup
 |      |             |
 |      |             +-- Need share/folder/file restore?
 |      |                    +-- YES --> Azure Files Backup
 |      |
 |      +-- NO --> Is the workload blob data, managed disks, AKS, or a newer
 |                 Azure Data Protection scenario?
 |                     |
 |                     +-- YES --> Use Backup Vault first
 |                     |             |
 |                     |             +-- Blob overwrite/delete/PITR? --> Azure Blob Backup
 |                     |             +-- Disk-only protection? -------> Managed Disk Backup
 |                     |             +-- AKS persistent volume state? -> AKS Backup
 |                     |
 |                     +-- NO --> Is it Azure SQL Database or SQL Managed Instance?
 |                                   |
 |                                   +-- YES --> Use native Azure SQL backups (PITR/LTR)
 |                                   +-- NO --> Re-check whether the requirement is really
 |                                              backup, or if it is a DR/failover question
```

### Quick Decision Matrix

| Scenario | Best starting design |
|---|---|
| Protect Azure VM and restore whole server or files | Recovery Services Vault + Azure VM Backup |
| Need low-RPO SQL recovery inside a VM | Recovery Services Vault + SQL workload backup |
| Need Azure SQL Database retention/compliance | Native PITR + LTR |
| Need item-level restore for file shares | Recovery Services Vault + Azure Files Backup |
| Need blob PITR or isolated object retention | Backup Vault + Azure Blob Backup |
| Need AKS persistent volume protection | Backup Vault + AKS backup |
| Need regional app availability in minutes | Azure Site Recovery / geo-replication, not backup alone |


<a id="3-recovery-services-vault"></a>
## 3. Recovery Services Vault

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

<a id="4-azure-vm-backup"></a>
## 4. Azure VM Backup

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

<a id="5-azure-sql-backup"></a>
## 5. Azure SQL Backup

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

<a id="6-azure-files-backup"></a>
## 6. Azure Files Backup

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

<a id="7-azure-blob-backup"></a>
## 7. Azure Blob Backup

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

<a id="8-other-workload-backup"></a>
## 8. Other Workload Backup

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

<a id="9-backup-policies-and-retention"></a>
## 9. Backup Policies and Retention

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

<a id="10-availability-resilience"></a>
## 10. Availability & Resilience

Backup architecture decisions are really **resilience decisions**: choose the backup storage redundancy, restore geography, and restore speed that align to business RPO/RTO instead of assuming every workload needs the most expensive option.

### Redundancy Comparison

| Option | What it protects against | Best use | Key tradeoff |
|---|---|---|---|
| **LRS** | Local storage faults in one datacenter | Lowest-cost dev/test or non-critical workloads | No zone or cross-region resilience |
| **ZRS** | Zonal failure within a region | Workloads needing in-region zonal durability | No cross-region restore |
| **GRS** | Regional durability via paired-region copy | Critical workloads needing backup-based geo-restore | Highest vault cost and paired-region dependency |

### Cross-Region Restore (CRR)

- **CRR requires GRS** for supported workloads and is the clearest exam signal for backup-based recovery in another region.
- CRR helps when the question says **restore in another region**; it is **not** the right answer when the question says **keep the app online during an outage**.
- Expect slower recovery than replication-based DR because backup recovery still involves restore operations, not immediate failover.

### Resilience Design Rules

1. Use **LRS** when cost is the priority and geo-recovery is not required.
2. Use **ZRS** when the business wants zonal resilience but can accept recovery staying in the same region.
3. Use **GRS + CRR** only for workloads that truly need backup-based paired-region recovery.
4. Pair backup with **Azure Site Recovery, failover groups, or geo-replication** when the business requirement is low-RTO application continuity.

> ⚠️ **AZ-305 Trap:** A vault with GRS does not make the application highly available. It only makes the **backup copies** more resilient.


<a id="11-cost-optimization"></a>
## 11. Cost Optimization

### Primary Cost Levers

| Cost lever | How to save | Watch out for |
|---|---|---|
| **Vault redundancy** | Prefer **LRS** unless geo-restore is a real business requirement | Overusing GRS for low-value workloads |
| **Retention length** | Use GFS-style daily/weekly/monthly/yearly retention instead of keeping every daily backup forever | Compliance workloads may still require yearly retention |
| **Instant restore retention** | Keep snapshot retention only as long as the target RTO needs | Longer snapshot retention increases cost |
| **Selective disk backup** | Exclude rebuildable or non-critical disks | Never exclude a disk required for app recovery |
| **Scope of protection** | Back up the data source that matters most (disk vs VM vs database) | Over-protecting whole machines can waste money |
| **Workload tiering** | Give prod stronger redundancy and longer retention than dev/test | One-size-fits-all backup policies are usually inefficient |

### Practical Optimization Patterns

- Put **dev/test** workloads on shorter retention and LRS unless policy forbids it.
- Choose **SQL workload backup** instead of VM-only backup when database recovery precision matters.
- Use **native service backup** where available instead of layering unnecessary vault complexity.
- Review **instant restore**, **GRS**, and **long retention** together because those are common backup cost multipliers.

> 💡 **AZ-305 Tip:** The cheapest design is not always the best answer — but the exam often rewards the **least expensive design that still meets stated RPO, RTO, retention, and compliance requirements**.


<a id="12-restore-operations"></a>
## 12. Restore Operations

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

<a id="13-backup-security"></a>
## 13. Backup Security

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

<a id="14-backup-monitoring-and-reporting"></a>
## 14. Backup Monitoring and Reporting

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

<a id="15-az-305-decision-scenarios"></a>
## 15. AZ-305 Decision Scenarios

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

<a id="16-quick-reference-trigger-table"></a>
## 16. Quick Reference Trigger Table

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

<a id="17-common-exam-traps"></a>
## 17. Common Exam Traps

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

---

<a id="18-final-az-305-exam-tips"></a>
## 🎯 Final AZ-305 Exam Tips

1. Identify the **workload first**: VM, SQL in VM, Azure SQL PaaS, Files, Blob, AKS, Disk, or hybrid server.
2. Separate **backup** questions from **DR/failover** questions before choosing a service.
3. Match the design to explicit **RPO, RTO, retention, security, and compliance** requirements.
4. Use **Recovery Services Vault** for classic workloads; use **Backup Vault** for newer Azure Data Protection scenarios.
5. For Azure SQL Database/Managed Instance, think **native PITR/LTR** before adding vault-based designs.
6. If the scenario says **ransomware** or **tamper resistance**, layer **soft delete + immutability + MUA**.
7. If the scenario says **another region**, ask whether it means **restore** (backup) or **failover** (DR).
8. Use **GRS + CRR** selectively; do not pay for geo-restore where the business never asked for it.
9. Use **GFS retention** to balance operational recovery with long-term compliance.
10. The best AZ-305 answer is usually the **simplest design that fully meets the stated business requirement**.

---

<a id="19-architecture-decision-flowchart"></a>
## 📐 Architecture Decision Flowchart

```
Business requirement
 |
 +-- Need app availability during outage in minutes?
 |      +-- YES --> Use DR/failover architecture (ASR, geo-replication, failover groups)
 |      +-- NO --> Continue with backup design
 |
 +-- What workload is being protected?
 |      +-- Azure VM ------------------> RSV + VM Backup
 |      +-- SQL Server in Azure VM ----> RSV + SQL workload backup
 |      +-- Azure SQL DB / MI ---------> Native PITR / LTR
 |      +-- Azure Files ---------------> RSV + Files Backup
 |      +-- Blob data -----------------> Backup Vault + Blob Backup
 |      +-- AKS / Disk / PostgreSQL ---> Backup Vault or native backup
 |
 +-- Is geo-restore required?
 |      +-- YES --> Consider GRS + CRR where supported
 |      +-- NO --> Prefer LRS or ZRS based on resilience needs
 |
 +-- Are ransomware or compliance controls required?
        +-- YES --> Add immutability, MUA, soft delete, private access as needed
        +-- NO --> Keep the design lean and cost-aligned
```

---

<a id="20-exam-style-review-questions"></a>
## Exam-Style Review Questions

1. A company must restore a critical Azure VM in a paired region after a regional outage, but it does not require active-active failover. What backup design best fits, and why is Azure Site Recovery not strictly required?
2. A finance team needs 7-year retention for Azure SQL Database with minimal operational overhead. Should you recommend Recovery Services Vault, Backup Vault, or native SQL backups, and what retention feature matters most?
3. An application stores user documents in Azure Blob Storage and needs rollback after accidental overwrite within the last 48 hours. Which Azure backup approach best fits, and why is soft delete alone not always enough?
4. A business asks for the cheapest backup design for 50 dev/test VMs with no geo-restore requirement. Which redundancy and retention choices best control cost without violating the requirement?
5. A regulated workload requires approval before disabling backup and must keep recovery points tamper-resistant. Which layered security controls should be designed into the backup architecture?

---

**Footer:** Pair this cheat sheet with [Backup Labs](./Labs/Azure-Backup-Labs.md). For every AZ-305 scenario, validate the design against **RPO, RTO, retention, security, compliance, and cost** before selecting the service.
