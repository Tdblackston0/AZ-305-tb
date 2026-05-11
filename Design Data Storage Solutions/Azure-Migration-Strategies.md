# Azure Data Migration Strategies — AZ-305 Cheat Sheet & Exam Prep

> 📝 **Hands-On Labs:** [Azure Migration Labs](../Labs/Azure-Migration-Labs.md)

> **Purpose:** Definitive migration reference for AZ-305 — study this to pass.
> **Perspective:** Senior Cloud Solution Architect
> **Scope:** ALL database platforms, ALL migration tools, ALL decision scenarios

---

## 1. Migration Overview for AZ-305

### What Migration Topics Appear on the Exam

| Exam Domain | Migration Coverage |
|---|---|
| Design data storage solutions (25-30%) | Database migration paths, tool selection, online vs offline |
| Design infrastructure solutions (25-30%) | Server migration, ASR, Azure Migrate |
| Design business continuity (10-15%) | Migration with minimal downtime, DR during migration |
| Design identity & access (25-30%) | Post-migration security, managed identity for migrated workloads |

**Key exam skill:** Given a scenario, select the correct migration tool, target service, and approach (online vs offline).

---

### The 5 R's of Migration

| Strategy | Definition | When to Use | Azure Tools |
|---|---|---|---|
| **Rehost** (Lift & Shift) | Move as-is to Azure VMs | Quick migration, no code changes, legacy apps | Azure Migrate, ASR, Data Box |
| **Refactor** (Replatform) | Move to PaaS with minimal changes | Reduce ops overhead, gain managed services | DMS, DMA, SSMA, Azure SQL DB/MI |
| **Rearchitect** | Redesign for cloud-native patterns | Scale, performance, microservices | ADF, Cosmos DB, Synapse |
| **Rebuild** | Rewrite from scratch | Legacy code too costly to migrate | App Service, AKS, Functions |
| **Replace** | Switch to SaaS | Commodity functionality available as SaaS | Dynamics 365, Power Platform, M365 |

> 🎯 **Exam Tip:** AZ-305 heavily favors **Refactor** (PaaS) answers unless the scenario explicitly requires VM-level control or unsupported features.

---

### Azure Migration Framework

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   ASSESS    │───▶│   MIGRATE   │───▶│  OPTIMIZE   │───▶│   SECURE    │
│             │    │             │    │             │    │             │
│ • Discovery │    │ • Rehost    │    │ • Right-size│    │ • RBAC      │
│ • Dependency│    │ • Refactor  │    │ • Reserved  │    │ • Encryption│
│ • Compat.   │    │ • Rearchitect│   │ • Auto-scale│    │ • Firewall  │
│ • TCO       │    │ • Test      │    │ • Monitor   │    │ • Audit     │
└─────────────┘    └─────────────┘    └─────────────┘    └─────────────┘
```

**Phase Details:**

| Phase | Key Activities | Tools |
|---|---|---|
| **Assess** | Discover workloads, map dependencies, check compatibility, estimate costs | Azure Migrate, DMA, SSMA, Service Map |
| **Migrate** | Execute migration, validate data integrity, test applications | DMS, ASR, AzCopy, Data Box, BACPAC |
| **Optimize** | Right-size resources, implement autoscaling, cost management | Azure Advisor, Cost Management, Monitor |
| **Secure** | Apply RBAC, encryption, network security, compliance | Defender for Cloud, Key Vault, Private Link |

---

## 2. Migration Tools Ecosystem

### Azure Migrate

| Aspect | Details |
|---|---|
| **Purpose** | Central hub for discovery, assessment, and migration |
| **When to Use** | Starting any migration project; need full inventory |
| **Capabilities** | Server discovery, dependency mapping (agent/agentless), assessment reports, VM migration |
| **Supported Sources** | VMware, Hyper-V, physical servers, AWS VMs, GCP VMs |
| **Key Features** | Agentless discovery, application dependency mapping, TCO/cost estimation, Azure readiness |
| **Integration** | Works with DMS, ASR, Data Box, and partner tools (Movere, Carbonite, Cloudamize) |
| **Exam Focus** | First tool to recommend for any "assess and plan" scenario |

**When to Use:** Any migration planning scenario. Azure Migrate is always the starting point.

---

### Azure Database Migration Service (DMS)

| Aspect | Details |
|---|---|
| **Purpose** | Managed service for database migrations to Azure |
| **Online Mode** | Continuous sync, minimal downtime (minutes), requires ongoing replication |
| **Offline Mode** | One-time migration, longer downtime, simpler setup |
| **Supported Sources** | SQL Server, MySQL, PostgreSQL, MongoDB, Oracle (via partner) |
| **Supported Targets** | Azure SQL DB, SQL MI, Azure DB for MySQL, Azure DB for PostgreSQL, Cosmos DB (MongoDB API) |

**DMS Supported Migration Pairs:**

| Source | Target | Online | Offline |
|---|---|---|---|
| SQL Server | Azure SQL Database | ✅ | ✅ |
| SQL Server | Azure SQL MI | ✅ | ✅ |
| SQL Server | SQL Server on Azure VM | ❌ | ✅ |
| MySQL (5.6, 5.7, 8.0) | Azure DB for MySQL | ✅ | ✅ |
| PostgreSQL (9.4+) | Azure DB for PostgreSQL | ✅ | ✅ |
| MongoDB (3.4+) | Cosmos DB (MongoDB API) | ✅ | ✅ |
| Oracle | Azure DB for PostgreSQL | ❌ | ✅ (with ora2pg) |

> 🎯 **Exam Tip:** DMS Online = near-zero downtime but requires more setup. DMS Offline = simpler but requires maintenance window. If the question says "minimize downtime," pick **Online**.

---

### Data Migration Assistant (DMA)

| Aspect | Details |
|---|---|
| **Purpose** | **Assessment only** — compatibility checking for SQL Server → Azure SQL |
| **When to Use** | Before migrating SQL Server to Azure SQL DB or MI |
| **Capabilities** | Feature parity analysis, breaking changes, deprecated features, performance recommendations |
| **Does NOT** | Actually migrate data — it only assesses |
| **Output** | Readiness report with blocking issues, warnings, and information messages |

> 🎯 **Exam Tip:** DMA = **Assess**. DMS = **Migrate**. They sound similar but serve different purposes. If the question asks about finding compatibility issues before migration, the answer is DMA.

---

### SQL Server Migration Assistant (SSMA)

| Aspect | Details |
|---|---|
| **Purpose** | Migrate **non-SQL Server** databases to Azure SQL |
| **Supported Sources** | Oracle, MySQL, Access, SAP ASE (Sybase), IBM Db2 |
| **Target** | Azure SQL Database, Azure SQL MI, SQL Server on Azure VM |
| **Capabilities** | Schema conversion, data migration, assessment reports, type mapping |
| **When to Use** | Migrating from Oracle, MySQL, Access, SAP ASE, or Db2 to Azure SQL family |

**SSMA Variants:**

| SSMA Version | Source Database | Key Considerations |
|---|---|---|
| SSMA for Oracle | Oracle 10g+ | PL/SQL → T-SQL conversion, complex type mappings |
| SSMA for MySQL | MySQL 5.x, 8.x | When target is Azure SQL (not Azure DB for MySQL) |
| SSMA for Access | Access 97–365 | Desktop DB modernization, linked table conversion |
| SSMA for SAP ASE | Sybase ASE 12+ | Transact-SQL dialect differences |
| SSMA for Db2 | IBM Db2 LUW, z/OS | Mainframe modernization scenarios |

> 🎯 **Exam Tip:** SSMA is the answer when migrating **heterogeneous** databases (non-SQL Server) to Azure SQL. If source is Oracle/MySQL/Access/SAP ASE/Db2 → Azure SQL, think SSMA.

---

### Azure Data Studio — Migration Extension

| Aspect | Details |
|---|---|
| **Purpose** | GUI-based SQL Server migration assessment and execution |
| **When to Use** | DBA-friendly alternative to DMA for SQL Server → Azure SQL MI or Azure SQL DB |
| **Capabilities** | SKU recommendation, compatibility assessment, migration wizard |
| **Integration** | Calls DMS under the hood for actual migration |

---

### Azure Site Recovery (ASR)

| Aspect | Details |
|---|---|
| **Purpose** | VM-level replication for lift-and-shift or DR |
| **When to Use** | Migrating entire VMs (including SQL Server on VMs) |
| **Sources** | VMware, Hyper-V, physical servers, AWS, other clouds |
| **Target** | Azure VMs |
| **RPO** | As low as 30 seconds continuous replication |
| **Downtime** | Minutes (failover time only) |

> 🎯 **Exam Tip:** ASR = **VM migration** (lift and shift). Not for PaaS database migration. If the scenario says "migrate SQL Server with minimal changes and retain full OS control," think ASR.

---

### Data Transfer Tools

#### AzCopy

| Aspect | Details |
|---|---|
| **Purpose** | Command-line tool for copying data to/from Azure Storage |
| **When to Use** | Network transfer up to ~10 TB, blob/file migration |
| **Capabilities** | Parallel upload, SAS/OAuth auth, resume support, sync mode |
| **Performance** | Saturates network bandwidth, automatic parallelism |

#### Azure Storage Explorer

| Aspect | Details |
|---|---|
| **Purpose** | GUI tool for Azure Storage management and transfer |
| **When to Use** | Ad-hoc transfers, visual browsing, uses AzCopy under the hood |

#### Data Box Family

| Product | Capacity | When to Use | Transfer Time |
|---|---|---|---|
| **Data Box Disk** | Up to 35 TB (5 × 8 TB SSDs) | 1–35 TB, limited bandwidth | Days + shipping |
| **Data Box** | Up to 80 TB | 40–500 TB, offline transfer | Days + shipping |
| **Data Box Heavy** | Up to 1 PB | 500 TB–1 PB, massive datasets | Days + shipping |
| **Data Box Gateway** | Continuous | Ongoing cloud tiering | Continuous |

> 🎯 **Exam Tip:** Use the **breakeven formula** — if network transfer takes > 1 week, consider Data Box. Data Box becomes cost-effective at ~40 TB over typical enterprise bandwidth.

#### Azure Storage Mover

| Aspect | Details |
|---|---|
| **Purpose** | Managed migration for NAS/file shares to Azure Storage |
| **When to Use** | Migrating SMB/NFS file shares from on-premises NAS devices |
| **Target** | Azure Blob Storage, Azure Files |
| **Key Feature** | Agent-based, supports scheduled jobs, preserves metadata |

---

### Azure Data Factory (ADF) / Synapse Pipelines

| Aspect | Details |
|---|---|
| **Purpose** | ETL/ELT-based data migration and transformation |
| **When to Use** | Complex migrations needing transformation, multi-source, data warehouse loading |
| **Supported Sources** | 100+ connectors (SQL, Oracle, Teradata, Netezza, MongoDB, DynamoDB, SAP, S3, etc.) |
| **Key Features** | Mapping data flows, schema drift, incremental copy, scheduling |
| **Exam Focus** | Best for data warehouse migrations, heterogeneous ETL scenarios |

---

### Log Replay Service (LRS)

| Aspect | Details |
|---|---|
| **Purpose** | Native backup/restore migration to Azure SQL MI |
| **When to Use** | SQL Server → SQL MI when you need full control over backup chain |
| **How It Works** | Upload .bak/.log files to Azure Blob → LRS replays them on MI |
| **Key Benefit** | Uses native SQL Server backup, familiar to DBAs |
| **Downtime** | Minimal — continuous log replay, cutover when ready |

---

### Additional Migration Methods

| Tool/Method | Purpose | Source → Target |
|---|---|---|
| **Transactional Replication** | Near-zero downtime SQL migration | SQL Server → Azure SQL DB (subscriber) |
| **BACPAC (sqlpackage)** | Schema + data portability | SQL Server ↔ Azure SQL DB |
| **BCP (Bulk Copy Program)** | Data-only transfer (no schema) | Any SQL → Any SQL |
| **PolyBase** | Query external data, load into Synapse | Hadoop, Blob, SQL → Synapse |
| **COPY INTO** | High-performance loading into Synapse | Blob Storage → Synapse dedicated pool |
| **mongodump/mongorestore** | Native MongoDB backup/restore | MongoDB → Cosmos DB (MongoDB API) |
| **pg_dump/pg_restore** | Native PostgreSQL backup/restore | PostgreSQL → Azure DB for PostgreSQL |
| **mysqldump** | Native MySQL backup/restore | MySQL → Azure DB for MySQL |

---

### Comprehensive Tool Comparison Table

| Tool | Source | Target | Online | Offline | Downtime | Best For |
|---|---|---|---|---|---|---|
| **Azure Migrate** | VMware, Hyper-V, Physical, AWS | Azure VMs | ✅ | ✅ | Minutes | Full infrastructure assessment & VM migration |
| **DMS (Online)** | SQL, MySQL, PostgreSQL, MongoDB | Azure PaaS DBs | ✅ | — | Minutes | Minimal-downtime database migration |
| **DMS (Offline)** | SQL, MySQL, PostgreSQL, MongoDB | Azure PaaS DBs | — | ✅ | Hours | Simple database migration with maintenance window |
| **DMA** | SQL Server | Assessment only | — | — | — | Pre-migration compatibility check |
| **SSMA** | Oracle, MySQL, Access, SAP ASE, Db2 | Azure SQL family | — | ✅ | Hours | Heterogeneous DB migration to Azure SQL |
| **ASR** | VMware, Hyper-V, Physical, AWS | Azure VMs | ✅ | — | Minutes | Lift-and-shift VM migration |
| **AzCopy** | On-prem storage, AWS S3 | Azure Storage | — | ✅ | N/A | File/blob transfer up to ~10 TB |
| **Data Box** | On-prem any | Azure Storage | — | ✅ | N/A | Bulk offline transfer 40 TB–1 PB |
| **Storage Mover** | NAS (SMB/NFS) | Azure Blob/Files | — | ✅ | N/A | File share migration |
| **ADF/Synapse** | 100+ sources | Azure data services | — | ✅ | Varies | ETL-based migration with transformation |
| **LRS** | SQL Server | Azure SQL MI | ✅ | ✅ | Minutes | Native backup restore to MI |
| **Transactional Replication** | SQL Server | Azure SQL DB | ✅ | — | Minutes | Near-zero downtime SQL migration |
| **BACPAC (sqlpackage)** | SQL Server | Azure SQL DB | — | ✅ | Hours | Schema + data portability |
| **BCP** | Any SQL Server | Any SQL Server | — | ✅ | Varies | Data-only bulk transfer |
| **PolyBase / COPY INTO** | Blob, HDFS, SQL | Synapse | — | ✅ | N/A | Data warehouse loading |
| **mongodump/restore** | MongoDB | Cosmos DB (Mongo API) | — | ✅ | Hours | Offline MongoDB migration |
| **pg_dump/restore** | PostgreSQL | Azure DB for PostgreSQL | — | ✅ | Hours | Offline PostgreSQL migration |
| **mysqldump** | MySQL | Azure DB for MySQL | — | ✅ | Hours | Offline MySQL migration |

---

## 3. Source Database Migration Paths

### SQL Server → Azure SQL Database

**Assessment:**
```
DMA → Identify blocking issues → Resolve incompatibilities → DMS migration
```

**Migration Methods (ranked by preference):**

| Method | Downtime | Complexity | Best For |
|---|---|---|---|
| DMS Online | Minutes | Medium | Production with low downtime tolerance |
| DMS Offline | Hours | Low | Dev/test or maintenance window available |
| Transactional Replication | Minutes | High | Complex sync requirements |
| BACPAC import | Hours | Low | Small databases (< 50 GB) |
| BCP | Hours | Medium | Data-only migration |

**Key Compatibility Considerations:**
- ❌ No SQL Agent Jobs (use Elastic Jobs or Azure Automation)
- ❌ No cross-database queries (use Elastic Query)
- ❌ No CLR assemblies (limited support)
- ❌ No Service Broker (use Service Bus)
- ❌ No linked servers (use Elastic Query)
- ✅ Containment level must be set for contained DB features

**PowerShell Commands:**

```powershell
# Export BACPAC
az sql db export --admin-password $password `
  --admin-user $adminUser `
  --storage-key $storageKey `
  --storage-key-type StorageAccessKey `
  --storage-uri "https://storage.blob.core.windows.net/bacpac/db.bacpac" `
  --name mydb --server myserver --resource-group myRG

# Import BACPAC
az sql db import --admin-password $password `
  --admin-user $adminUser `
  --storage-key $storageKey `
  --storage-key-type StorageAccessKey `
  --storage-uri "https://storage.blob.core.windows.net/bacpac/db.bacpac" `
  --name mydb --server myserver --resource-group myRG

# PowerShell cmdlet — Export
New-AzSqlDatabaseExport -ResourceGroupName "myRG" `
  -ServerName "myserver" -DatabaseName "mydb" `
  -StorageKeyType "StorageAccessKey" -StorageKey $storageKey `
  -StorageUri "https://storage.blob.core.windows.net/bacpac/db.bacpac" `
  -AdministratorLogin $adminUser -AdministratorLoginPassword $securePassword

# PowerShell cmdlet — Import
New-AzSqlDatabaseImport -ResourceGroupName "myRG" `
  -ServerName "myserver" -DatabaseName "mydb" `
  -StorageKeyType "StorageAccessKey" -StorageKey $storageKey `
  -StorageUri "https://storage.blob.core.windows.net/bacpac/db.bacpac" `
  -AdministratorLogin $adminUser -AdministratorLoginPassword $securePassword `
  -Edition "Standard" -ServiceObjectiveName "S3" -DatabaseMaxSizeBytes 268435456000

# sqlpackage — Export
sqlpackage /Action:Export /ssn:"myserver.database.windows.net" `
  /sdn:"mydb" /su:$adminUser /sp:$password `
  /tf:"C:\migration\mydb.bacpac"

# sqlpackage — Import
sqlpackage /Action:Import /tsn:"myserver.database.windows.net" `
  /tdn:"mydb" /tu:$adminUser /tp:$password `
  /sf:"C:\migration\mydb.bacpac"
```

**Post-Migration Validation:**
- Compare row counts across all tables
- Verify stored procedures, views, functions compile
- Test application connectivity and query performance
- Check index fragmentation and update statistics
- Validate security (logins, users, permissions)

---

### SQL Server → Azure SQL Managed Instance

**Why Choose MI Over SQL DB:**
- Need SQL Agent Jobs
- Need cross-database queries
- Need CLR, Service Broker, linked servers
- Need instance-scoped features (server-level collation, TDE with customer-managed keys)
- Near 100% SQL Server compatibility

**Migration Methods:**

| Method | Downtime | Best For |
|---|---|---|
| DMS Online | Minutes | Production, minimal downtime |
| Log Replay Service (LRS) | Minutes | DBA-controlled, native backup chain |
| DMS Offline | Hours | Simple migrations |
| Backup/Restore via URL | Hours | Small databases, familiar process |

**Network Requirements:**
- MI deployed in a dedicated VNet subnet
- Minimum /27 subnet (32 addresses)
- NSG rules for management traffic (ports 9000, 9003, 1438, 1440, 1452)
- UDR for Azure management (required)
- ExpressRoute or VPN for on-premises connectivity during migration

**PowerShell Commands:**

```powershell
# DMS — Online migration to MI
az datamigration sql-managed-instance create `
  --managed-instance-name "myMI" `
  --resource-group "myRG" `
  --scope "/subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.Sql/managedInstances/myMI" `
  --source-location '{\"fileShare\":{\"path\":\"\\\\server\\share\",\"username\":\"admin\",\"password\":\"pass\"}}' `
  --target-db-name "mydb" `
  --migration-service "/subscriptions/{sub}/resourceGroups/{rg}/providers/Microsoft.DataMigration/sqlMigrationServices/myDMS"

# Native backup restore via URL
# Step 1: Create credential on MI
# CREATE CREDENTIAL [https://storage.blob.core.windows.net/backups]
# WITH IDENTITY = 'SHARED ACCESS SIGNATURE', SECRET = '<SAS>'

# Step 2: Restore
# RESTORE DATABASE [mydb] FROM URL = 'https://storage.blob.core.windows.net/backups/mydb.bak'
```

---

### SQL Server → SQL Server on Azure VM

**When This Is the Only Option:**
- Application requires specific SQL Server version/build
- Need OS-level access for custom software
- Unsupported features in PaaS (certain replication, FILESTREAM, etc.)
- Licensing constraints (bring-your-own-license)
- Vendor requirement for SQL Server on VM

**Migration Methods:**

| Method | Downtime | Best For |
|---|---|---|
| ASR Replication | Minutes | Full VM lift-and-shift |
| Backup/Restore | Hours | Database-level migration |
| Always On AG (hybrid) | Minutes | Zero-downtime with AG extension |
| Log Shipping | Minutes | Familiar to DBAs, simple setup |
| Data Box + Restore | Hours | Huge databases (> 10 TB), limited bandwidth |

**PowerShell:**

```powershell
# Register SQL VM with SQL IaaS Agent Extension
az sql vm create --name "mySqlVM" `
  --resource-group "myRG" `
  --location "eastus2" `
  --license-type "PAYG" `
  --sql-mgmt-type "Full"
```

---

### Oracle → Azure

**Option A: Oracle → Azure SQL Database / MI (via SSMA)**

```
SSMA Assessment → Schema Conversion (PL/SQL → T-SQL) → Data Migration → Validation
```

| Step | Tool | Details |
|---|---|---|
| Assess | SSMA for Oracle | Identify conversion complexity, type mappings |
| Convert Schema | SSMA for Oracle | PL/SQL → T-SQL, sequences → identity/sequences |
| Migrate Data | SSMA for Oracle | BCP-based data transfer |
| Validate | Manual + testing | Verify stored procedures, triggers, data types |

**Key Conversion Challenges:**
- `NUMBER` → `decimal`, `int`, `bigint` (context-dependent)
- `VARCHAR2` → `nvarchar`
- `DATE` (includes time in Oracle) → `datetime2`
- Sequences → `IDENTITY` columns or `SEQUENCE` objects
- Packages → Stored procedures (no direct equivalent)
- `CONNECT BY` → Recursive CTEs
- `ROWNUM` → `TOP` / `ROW_NUMBER()`

**Option B: Oracle → Azure Database for PostgreSQL (via ora2pg + DMS)**

```
ora2pg Assessment → ora2pg Schema Export → Manual Fixes → DMS Data Migration
```

Best when PostgreSQL is a better functional fit than SQL Server for the Oracle workload.

---

### MySQL → Azure Database for MySQL

**Migration Methods:**

| Method | Downtime | Best For |
|---|---|---|
| DMS Online | Minutes | Production, continuous sync |
| mysqldump + mysqlimport | Hours | Small databases (< 1 GB) |
| MySQL Replication (binlog) | Minutes | Custom replication control |
| ADF with MySQL connector | Hours | Transformation during migration |

**Native Tool Commands:**

```bash
# Export with mysqldump
mysqldump -h sourcehost -u root -p --databases mydb \
  --single-transaction --routines --triggers --events > mydb.sql

# Import to Azure MySQL
mysql -h myserver.mysql.database.azure.com -u admin@myserver -p mydb < mydb.sql

# For large databases — use mysqlpump (parallel)
mysqlpump -h sourcehost -u root -p --databases mydb \
  --default-parallelism=4 > mydb.sql
```

**Key Considerations:**
- Azure MySQL supports 5.7 and 8.0 (check version compatibility)
- `SUPER` privilege not available — remove DEFINER clauses from dump
- `local_infile` must be enabled for `LOAD DATA LOCAL INFILE`
- SSL enforced by default — configure client certificates
- Max connections varies by tier (review SKU limits)

---

### PostgreSQL → Azure Database for PostgreSQL

**Migration Methods:**

| Method | Downtime | Best For |
|---|---|---|
| DMS Online | Minutes | Production, continuous sync |
| pg_dump / pg_restore | Hours | Small-to-medium databases |
| Logical Replication | Minutes | Selective table migration |
| ADF with PostgreSQL connector | Hours | Transformation during migration |

**Native Tool Commands:**

```bash
# pg_dump (custom format, parallel)
pg_dump -h sourcehost -U postgres -Fc -j 4 mydb > mydb.dump

# pg_restore (parallel, to Azure)
pg_restore -h myserver.postgres.database.azure.com \
  -U admin@myserver -d mydb -Fc -j 4 mydb.dump

# Plain SQL format
pg_dump -h sourcehost -U postgres mydb > mydb.sql
psql -h myserver.postgres.database.azure.com \
  -U admin@myserver -d mydb -f mydb.sql
```

**Key Considerations:**
- Use Flexible Server (recommended over Single Server — Single Server is deprecated)
- Extensions must be allowlisted (`shared_preload_libraries`)
- Check extension compatibility (not all extensions available)
- `pg_stat_statements` and `pgAudit` available
- PgBouncer built into Flexible Server for connection pooling

---

### MariaDB → Azure Database for MySQL

**Why MySQL?** Azure retired Azure Database for MariaDB. MySQL 8.0 is the recommended target.

| Method | Details |
|---|---|
| DMS | Online migration from MariaDB to Azure MySQL |
| mysqldump | Compatible — MariaDB is a MySQL fork |
| ADF | MariaDB connector available |

**Key Considerations:**
- MariaDB-specific features (e.g., `CONNECT` engine, `ColumnStore`) need alternatives
- Test stored procedures for syntax differences
- MariaDB sequences → MySQL `AUTO_INCREMENT` or workarounds

---

### MongoDB → Azure Cosmos DB (MongoDB API)

**Migration Methods:**

| Method | Downtime | Best For |
|---|---|---|
| DMS Online | Minutes | Production, continuous sync |
| DMS Offline | Hours | Simple one-time migration |
| mongodump / mongorestore | Hours | Small databases, familiar tooling |
| Spark connector | Hours | Large-scale, transformation needed |

**Native Tool Commands:**

```bash
# mongodump from source
mongodump --host sourcehost --port 27017 --db mydb --out /backup/

# mongorestore to Cosmos DB (MongoDB API)
mongorestore --host myaccount.mongo.cosmos.azure.com --port 10255 \
  --ssl --username myaccount --password $key \
  --db mydb /backup/mydb/
```

**Critical Design Decision — Partition Key = Shard Key:**
- Choose partition key during migration (cannot change later without re-creating container)
- High cardinality, even distribution, frequently used in queries
- Maps to MongoDB shard key concept
- Provision RU/s based on workload (start with autoscale)

**Key Considerations:**
- MongoDB wire protocol version compatibility (check supported APIs)
- Aggregation pipeline support (most stages supported, some limitations)
- Indexing: Cosmos DB auto-indexes all fields (wildcard policy default)
- Increase RU/s during migration, scale down after

---

### Cassandra → Azure Cosmos DB (Cassandra API)

**Migration Methods:**

| Method | Downtime | Best For |
|---|---|---|
| Spark Migration Utility | Minutes (dual-write) | Production, large datasets |
| CQLSH COPY | Hours | Small datasets, offline ok |
| Dual-write + bulk load | Minutes | Application-level control |

**Spark Migration (Recommended):**

```
1. Set up Spark cluster with Cosmos DB Cassandra connector
2. Read from source Cassandra using Spark
3. Write to Cosmos DB Cassandra API
4. Implement dual-write in application during cutover
5. Validate data, switch reads to Cosmos DB
```

**Offline with CQLSH:**

```bash
# Export from source Cassandra
cqlsh sourcehost -e "COPY keyspace.table TO 'data.csv' WITH HEADER=TRUE"

# Import to Cosmos DB Cassandra API
cqlsh myaccount.cassandra.cosmos.azure.com 10350 --ssl \
  -u myaccount -p $key \
  -e "COPY keyspace.table FROM 'data.csv' WITH HEADER=TRUE"
```

---

### DynamoDB (AWS) → Azure Cosmos DB

**Migration Methods:**

| Method | Details |
|---|---|
| ADF Pipeline | DynamoDB connector → Cosmos DB sink (recommended) |
| Custom SDK Migration | Read DynamoDB → Transform → Write Cosmos DB |
| AWS DMS to S3 → ADF | Export to S3, then ingest via ADF |

**ADF Approach:**

```
DynamoDB (Source) → ADF Copy Activity → Cosmos DB (SQL/MongoDB/Table API)
```

- Map DynamoDB partition key → Cosmos DB partition key
- Map sort key → additional property (or composite key)
- Handle DynamoDB-specific types (sets, lists, maps)
- Provision sufficient RU/s during bulk load

---

### Redis → Azure Cache for Redis

**Migration Methods:**

| Method | Downtime | Best For |
|---|---|---|
| DUMP/RESTORE per key | Minutes–Hours | Small datasets |
| RDB Import | Hours | Full snapshot import |
| Dual-write pattern | Minutes | Production, online migration |
| redis-copy tools | Hours | Automated key transfer |

**Approach Details:**

```bash
# RDB Import — export RDB file from source
redis-cli -h sourcehost BGSAVE
# Upload .rdb file to Azure Blob Storage
# Import via Azure Portal: Azure Cache for Redis → Import data

# DUMP/RESTORE (per key)
redis-cli -h source DUMP mykey | redis-cli -h target.redis.cache.windows.net \
  -a $accessKey --tls RESTORE mykey 0
```

**Dual-Write Pattern:**
1. Deploy Azure Cache for Redis
2. Update application to write to both source and Azure Redis
3. Backfill existing keys using migration script
4. Switch reads to Azure Redis
5. Remove source Redis writes

---

### Teradata / Netezza → Azure Synapse Analytics

**Migration Methods:**

| Method | Source | Best For |
|---|---|---|
| ADF + Teradata Connector | Teradata | Managed ETL migration |
| ADF + Netezza Connector | Netezza | Managed ETL migration |
| PolyBase External Tables | Both | Query without moving data first |
| SSMA Assessment | Teradata | Compatibility reporting |
| Data Box + PolyBase | Both | Huge datasets, limited bandwidth |

**ADF Pipeline Pattern:**

```
Teradata/Netezza → ADF Copy Activity → ADLS Gen2 (Parquet) → COPY INTO Synapse
```

**Key Considerations:**
- Convert proprietary SQL dialects to T-SQL
- Map data types (Teradata `BYTEINT`, `PERIOD` types)
- Redesign distribution strategy (Hash, Round-Robin, Replicated)
- Convert stored procedures to T-SQL or ADF pipelines
- Implement statistics and indexes differently in Synapse

---

### IBM Db2 → Azure SQL

| Step | Tool |
|---|---|
| Assessment | SSMA for Db2 |
| Schema Conversion | SSMA for Db2 |
| Data Migration | SSMA + BCP |
| Validation | Manual testing |

**Key Conversion Challenges:**
- Db2 `TIMESTAMP WITH TIME ZONE` → `datetimeoffset`
- Db2 sequences → SQL Server `SEQUENCE`
- MQTs (Materialized Query Tables) → Indexed views
- Db2 SQL PL → T-SQL stored procedures

---

### SAP ASE (Sybase) → Azure SQL

| Step | Tool |
|---|---|
| Assessment | SSMA for SAP ASE |
| Schema Conversion | SSMA for SAP ASE |
| Data Migration | SSMA + BCP |
| Validation | Manual testing |

**Key Considerations:**
- Similar T-SQL dialects (Sybase/SQL Server share heritage)
- Convert Sybase-specific system procedures
- Handle `text`/`image` → `nvarchar(max)`/`varbinary(max)`

---

### Access → Azure SQL

| Step | Tool |
|---|---|
| Assessment | SSMA for Access |
| Schema Conversion | SSMA for Access |
| Data Migration | SSMA |
| Front-end Update | Relink to Azure SQL via ODBC |

**Key Considerations:**
- AutoNumber → `IDENTITY`
- Yes/No → `bit`
- OLE Object → `varbinary(max)`
- Linked tables → Azure SQL connection strings
- Access forms → Power Apps or web app replacement (Rearchitect)

---

## 4. Data & Storage Migration

### File Servers → Azure Files

| Method | Best For | Details |
|---|---|---|
| **Azure File Sync** | Hybrid scenarios | Keeps on-prem servers as cache, tiering to Azure Files |
| **Storage Mover** | NAS devices | Agent-based, scheduled, metadata preservation |
| **Robocopy + AzCopy** | Windows file servers | Robocopy for local, AzCopy for cloud transfer |
| **Data Box** | > 40 TB, limited bandwidth | Ship drives, offline import |

**Azure File Sync Architecture:**

```
On-Prem File Server ←→ Azure File Sync Agent ←→ Storage Sync Service ←→ Azure Files
                                                                            ↑
                                                        Cloud Tiering (hot/cool)
```

---

### Data Warehouse → Azure Synapse Analytics

**Loading Methods (ranked by performance):**

| Method | Performance | Best For |
|---|---|---|
| **COPY INTO** | Fastest | Bulk loading from ADLS/Blob (Parquet, CSV, ORC) |
| **PolyBase** | Fast | External tables, query-in-place, then CTAS |
| **ADF Copy Activity** | Good | Managed pipeline, scheduling, monitoring |
| **BCP** | Moderate | Legacy tools, small datasets |
| **INSERT..SELECT** | Slowest | Small ad-hoc loads only |

```sql
-- COPY INTO (recommended for Synapse dedicated pools)
COPY INTO dbo.FactSales
FROM 'https://storage.blob.core.windows.net/data/sales/*.parquet'
WITH (FILE_TYPE = 'PARQUET', CREDENTIAL = (IDENTITY = 'Managed Identity'));

-- PolyBase — Create External Table
CREATE EXTERNAL TABLE ext.Sales (...)
WITH (
    LOCATION = '/data/sales/',
    DATA_SOURCE = AzureBlobStorage,
    FILE_FORMAT = ParquetFormat
);

-- Load via CTAS
CREATE TABLE dbo.FactSales
WITH (DISTRIBUTION = HASH(ProductKey), CLUSTERED COLUMNSTORE INDEX)
AS SELECT * FROM ext.Sales;
```

---

### HDFS / Hadoop → ADLS Gen2

| Method | Best For |
|---|---|
| **distcp** | Hadoop-native, parallel distributed copy |
| **AzCopy** | Post-export file transfer |
| **ADF** | Managed pipeline with HDFS connector |
| **Data Box** | Massive datasets with limited bandwidth |

```bash
# distcp from HDFS to ADLS Gen2
hadoop distcp \
  hdfs://namenode:8020/data/ \
  abfss://container@storageaccount.dfs.core.windows.net/data/
```

---

### Bulk Data Transfer Decision Guide

| Data Size | Available Bandwidth | Recommended Method | Estimated Transfer Time |
|---|---|---|---|
| < 10 GB | Any | AzCopy / Storage Explorer | Minutes |
| 10 GB – 1 TB | > 100 Mbps | AzCopy | Hours |
| 1 TB – 10 TB | > 1 Gbps | AzCopy + ExpressRoute | Hours–1 day |
| 10 TB – 40 TB | > 1 Gbps | AzCopy + ExpressRoute | Days |
| 10 TB – 40 TB | < 100 Mbps | **Data Box Disk** | Days + shipping |
| 40 TB – 500 TB | Any | **Data Box** | Days + shipping |
| 500 TB – 1 PB | Any | **Data Box Heavy** | Days + shipping |
| > 1 PB | Any | Multiple Data Box Heavy | Weeks |

**Transfer Time Formula:**

```
Transfer Time (hours) = Data Size (GB) / (Bandwidth (Mbps) × 0.8 × 0.125 × 3600)

Simplified:
Transfer Time (hours) = Data Size (GB) / (Bandwidth (Mbps) × 0.36)

Example: 10 TB over 1 Gbps
= 10,240 GB / (1000 × 0.36) = 28.4 hours ≈ 1.2 days
```

> 🎯 **Exam Tip:** If calculated network transfer exceeds **1 week**, Data Box is the recommended choice. If the scenario mentions "limited bandwidth" or "remote location," think Data Box.

---

## 5. PowerShell & CLI Command Reference

### BACPAC Operations

```powershell
# ── BACPAC Export (Azure CLI) ──
az sql db export \
  --admin-password "<password>" \
  --admin-user "<admin>" \
  --storage-key "<storage-account-key>" \
  --storage-key-type "StorageAccessKey" \
  --storage-uri "https://<storage>.blob.core.windows.net/bacpac/mydb.bacpac" \
  --name "mydb" --server "myserver" --resource-group "myRG"

# ── BACPAC Import (Azure CLI) ──
az sql db import \
  --admin-password "<password>" \
  --admin-user "<admin>" \
  --storage-key "<storage-account-key>" \
  --storage-key-type "StorageAccessKey" \
  --storage-uri "https://<storage>.blob.core.windows.net/bacpac/mydb.bacpac" \
  --name "mydb" --server "myserver" --resource-group "myRG"

# ── BACPAC Export (PowerShell) ──
$exportRequest = New-AzSqlDatabaseExport `
  -ResourceGroupName "myRG" `
  -ServerName "myserver" `
  -DatabaseName "mydb" `
  -StorageKeyType "StorageAccessKey" `
  -StorageKey $storageKey `
  -StorageUri "https://storage.blob.core.windows.net/bacpac/mydb.bacpac" `
  -AdministratorLogin "admin" `
  -AdministratorLoginPassword $securePass

# Check export status
Get-AzSqlDatabaseImportExportStatus -OperationStatusLink $exportRequest.OperationStatusLink

# ── BACPAC Import (PowerShell) ──
$importRequest = New-AzSqlDatabaseImport `
  -ResourceGroupName "myRG" `
  -ServerName "myserver" `
  -DatabaseName "mydb" `
  -StorageKeyType "StorageAccessKey" `
  -StorageKey $storageKey `
  -StorageUri "https://storage.blob.core.windows.net/bacpac/mydb.bacpac" `
  -AdministratorLogin "admin" `
  -AdministratorLoginPassword $securePass `
  -Edition "Standard" `
  -ServiceObjectiveName "S3" `
  -DatabaseMaxSizeBytes 268435456000
```

### sqlpackage Operations

```powershell
# ── Export BACPAC ──
sqlpackage /Action:Export `
  /SourceServerName:"myserver.database.windows.net" `
  /SourceDatabaseName:"mydb" `
  /SourceUser:"admin" `
  /SourcePassword:"<password>" `
  /TargetFile:"C:\migration\mydb.bacpac"

# ── Import BACPAC ──
sqlpackage /Action:Import `
  /TargetServerName:"myserver.database.windows.net" `
  /TargetDatabaseName:"mydb" `
  /TargetUser:"admin" `
  /TargetPassword:"<password>" `
  /SourceFile:"C:\migration\mydb.bacpac"

# ── Extract DACPAC (schema only) ──
sqlpackage /Action:Extract `
  /SourceServerName:"myserver.database.windows.net" `
  /SourceDatabaseName:"mydb" `
  /TargetFile:"C:\migration\mydb.dacpac"

# ── Publish DACPAC (deploy schema) ──
sqlpackage /Action:Publish `
  /SourceFile:"C:\migration\mydb.dacpac" `
  /TargetServerName:"myserver.database.windows.net" `
  /TargetDatabaseName:"mydb"
```

### DMS Migration Commands

```powershell
# ── DMS: SQL Server → Azure SQL DB ──
az datamigration sql-db create `
  --sqldb-instance-name "myserver" `
  --source-sql-connection "data source=source;user id=sa;password=pass;initial catalog=mydb" `
  --target-sql-connection "data source=myserver.database.windows.net;user id=admin;password=pass;initial catalog=mydb" `
  --resource-group "myRG" `
  --target-db-name "mydb" `
  --scope "/subscriptions/{subId}/resourceGroups/myRG/providers/Microsoft.Sql/servers/myserver"

# ── DMS: SQL Server → Azure SQL MI ──
az datamigration sql-managed-instance create `
  --managed-instance-name "myMI" `
  --resource-group "myRG" `
  --target-db-name "mydb" `
  --scope "/subscriptions/{subId}/resourceGroups/myRG/providers/Microsoft.Sql/managedInstances/myMI" `
  --source-location '{"fileShare":{"path":"\\\\server\\share","username":"admin","password":"pass"}}' `
  --migration-service "/subscriptions/{subId}/resourceGroups/myRG/providers/Microsoft.DataMigration/sqlMigrationServices/myDMS"

# ── Check migration status ──
az datamigration sql-db show `
  --sqldb-instance-name "myserver" `
  --resource-group "myRG" `
  --target-db-name "mydb"
```

### AzCopy Operations

```powershell
# ── Upload to Blob Storage ──
azcopy copy "C:\data\*" "https://storage.blob.core.windows.net/container?<SAS>" `
  --recursive --put-md5

# ── Download from Blob Storage ──
azcopy copy "https://storage.blob.core.windows.net/container/*?<SAS>" "C:\data\" `
  --recursive

# ── Sync (mirror) ──
azcopy sync "C:\data" "https://storage.blob.core.windows.net/container?<SAS>" `
  --recursive --delete-destination=true

# ── Copy between storage accounts ──
azcopy copy "https://source.blob.core.windows.net/container?<SAS>" `
  "https://dest.blob.core.windows.net/container?<SAS>" `
  --recursive

# ── Copy from AWS S3 ──
azcopy copy "https://s3.amazonaws.com/mybucket" `
  "https://storage.blob.core.windows.net/container?<SAS>" `
  --recursive
```

### BCP Operations

```powershell
# ── Export data ──
bcp mydb.dbo.MyTable out "C:\data\mytable.dat" `
  -S "sourceserver" -U sa -P pass -n

# ── Import data to Azure SQL ──
bcp mydb.dbo.MyTable in "C:\data\mytable.dat" `
  -S "myserver.database.windows.net" -U admin -P pass -n `
  -b 10000 -a 16384
```

---

## 6. Comprehensive Decision Matrix

### Source Database × Target Service × Tool × Method

| Source Database | Target Azure Service | Primary Tool | Alternative Tool | Online? | Expected Downtime | Notes |
|---|---|---|---|---|---|---|
| SQL Server | Azure SQL Database | DMS | BACPAC, BCP, Transactional Repl. | ✅ Online | Minutes | Most common exam scenario |
| SQL Server | Azure SQL MI | DMS / LRS | Backup/Restore via URL | ✅ Online | Minutes | LRS for DBA-controlled |
| SQL Server | SQL on Azure VM | ASR | Backup/Restore, AG, Log Ship | ✅ Online | Minutes | Lift-and-shift only |
| Oracle | Azure SQL DB / MI | SSMA for Oracle | Manual conversion | ❌ Offline | Hours | Complex schema conversion |
| Oracle | Azure DB for PostgreSQL | ora2pg + DMS | ADF | ❌ Offline | Hours | Better functional fit |
| MySQL | Azure DB for MySQL | DMS | mysqldump, binlog repl. | ✅ Online | Minutes | Version compatibility critical |
| MySQL | Azure SQL DB | SSMA for MySQL | Manual conversion | ❌ Offline | Hours | Cross-platform migration |
| PostgreSQL | Azure DB for PostgreSQL | DMS | pg_dump/pg_restore | ✅ Online | Minutes | Check extension support |
| MariaDB | Azure DB for MySQL | DMS | mysqldump | ✅ Online | Minutes | MariaDB → MySQL target |
| MongoDB | Cosmos DB (MongoDB API) | DMS | mongodump/mongorestore | ✅ Online | Minutes | Partition key = shard key |
| Cassandra | Cosmos DB (Cassandra API) | Spark Utility | CQLSH COPY | ✅ Dual-write | Minutes | Spark for large scale |
| DynamoDB | Cosmos DB | ADF | Custom SDK | ❌ Offline | Hours | Map partition/sort keys |
| Redis | Azure Cache for Redis | RDB Import | DUMP/RESTORE, Dual-write | ⚠️ Dual-write | Minutes–Hours | No native DMS support |
| Teradata | Synapse Analytics | ADF | PolyBase, Data Box | ❌ Offline | Hours–Days | SQL dialect conversion |
| Netezza | Synapse Analytics | ADF | PolyBase | ❌ Offline | Hours–Days | IBM to Microsoft |
| IBM Db2 | Azure SQL DB / MI | SSMA for Db2 | BCP | ❌ Offline | Hours | Mainframe modernization |
| SAP ASE | Azure SQL DB / MI | SSMA for SAP ASE | BCP | ❌ Offline | Hours | Sybase heritage |
| Access | Azure SQL DB | SSMA for Access | Manual | ❌ Offline | Hours | Desktop to cloud |
| File Server | Azure Files | File Sync / Storage Mover | AzCopy, Data Box | ✅ Sync | N/A | Hybrid or full migration |
| HDFS | ADLS Gen2 | distcp | ADF, AzCopy | ❌ Offline | Hours–Days | Hadoop modernization |
| On-prem Storage | Azure Blob | AzCopy / Data Box | Storage Explorer | ❌ Offline | Hours–Days | Size determines method |

---

## 7. AZ-305 Decision Scenarios

### Scenario 1: Minimal Downtime SQL Server Migration

> **A company has a 500 GB SQL Server 2019 database running on-premises. They need to migrate to Azure with less than 5 minutes of downtime. The database uses SQL Agent jobs and Service Broker.**

**Answer:** Azure SQL **Managed Instance** with **DMS Online Migration**

**Why:**
- MI required: SQL Agent Jobs + Service Broker not supported in Azure SQL DB
- DMS Online: Continuous sync provides < 5 min downtime at cutover
- 500 GB is within DMS online limits
- MI provides near 100% SQL Server compatibility

---

### Scenario 2: Oracle ERP Migration

> **An enterprise runs a 2 TB Oracle 19c database for their ERP system. They want to move to Azure and reduce licensing costs. The application team can handle moderate code changes.**

**Answer:** Use **SSMA for Oracle** to migrate to **Azure SQL Managed Instance** (or Azure SQL Database depending on feature needs)

**Why:**
- SSMA for Oracle handles PL/SQL → T-SQL conversion
- Reduces Oracle licensing costs
- MI if instance-scoped features needed
- "Moderate code changes" = Refactor strategy acceptable
- Alternative: Oracle → Azure DB for PostgreSQL via ora2pg if PostgreSQL is a better fit

---

### Scenario 3: Multi-Database Application

> **A SaaS company has 200 tenant databases on MySQL 8.0. They need minimal downtime and want a managed service. Some tenants have unique schemas.**

**Answer:** **DMS Online Migration** to **Azure Database for MySQL Flexible Server**

**Why:**
- DMS Online for minimal downtime
- Azure DB for MySQL = natural PaaS target for MySQL
- Flexible Server supports MySQL 8.0
- Each tenant gets its own database on the server
- Unique schemas preserved in migration

---

### Scenario 4: Data Warehouse Modernization

> **A company has a 50 TB Teradata data warehouse. Network bandwidth is 100 Mbps. They want to migrate to Azure Synapse Analytics.**

**Answer:** **Azure Data Box** for initial load + **ADF with Teradata connector** for incremental sync

**Why:**
- 50 TB over 100 Mbps = ~12.8 days transfer time (> 1 week → Data Box)
- Data Box for bulk initial load to ADLS Gen2
- ADF pipeline: ADLS Gen2 → Synapse (COPY INTO)
- Teradata SQL needs conversion to T-SQL
- Ongoing sync via ADF incremental pipelines

---

### Scenario 5: MongoDB to Cosmos DB

> **A startup runs MongoDB 4.4 in AWS. They have 100 GB of data and need zero-downtime migration. The application uses aggregation pipelines heavily.**

**Answer:** **DMS Online Migration** to **Cosmos DB for MongoDB (vCore or RU-based)**

**Why:**
- DMS Online for zero-downtime continuous sync
- 100 GB is manageable for online migration
- Cosmos DB MongoDB API supports most aggregation stages
- Map MongoDB shard key → Cosmos DB partition key
- Verify aggregation pipeline compatibility before migration
- vCore model if they need full MongoDB compatibility

---

### Scenario 6: Hybrid File Server

> **A company has 10 TB of files across 3 on-premises file servers. They want branch offices to access files locally while centralizing in Azure. Remote offices have limited bandwidth.**

**Answer:** **Azure File Sync** with cloud tiering enabled

**Why:**
- Azure File Sync keeps local cache on each server
- Cloud tiering moves cold files to Azure, saves local storage
- Branch offices get local-speed access to hot files
- Central Azure Files share for backup, DR, compliance
- Limited bandwidth handled by sync agent's bandwidth throttling

---

### Scenario 7: Legacy Access Database

> **A department runs a critical Access database with 50 users. Management wants it modernized to a cloud solution with proper security and multi-user support.**

**Answer:** **SSMA for Access** → **Azure SQL Database** + **Power Apps** for front-end replacement

**Why:**
- SSMA for Access converts schema and data to Azure SQL DB
- Access forms → Power Apps (modern cloud front-end)
- Azure SQL DB provides proper multi-user, security, backup
- This is a **Rearchitect** scenario (not just Refactor)

---

### Scenario 8: Multi-Source NoSQL Consolidation

> **A company has Cassandra (on-prem), DynamoDB (AWS), and MongoDB (on-prem). They want to consolidate into a single Azure database platform.**

**Answer:** **Azure Cosmos DB** with appropriate APIs:

| Source | Target API | Migration Tool |
|---|---|---|
| Cassandra | Cosmos DB Cassandra API | Spark migration utility |
| DynamoDB | Cosmos DB Table API or SQL API | ADF with DynamoDB connector |
| MongoDB | Cosmos DB MongoDB API | DMS Online |

**Why:**
- Cosmos DB supports multiple APIs — each source maps to a compatible API
- Single platform with global distribution, SLA-backed
- Each migration uses the optimal tool for that source
- Consider NoSQL API (SQL API) for new development to maximize Cosmos DB features

---

### Scenario 9: PostgreSQL with Extensions

> **A company runs PostgreSQL 14 with PostGIS, pg_trgm, and custom C extensions. They need to migrate to Azure with minimal changes.**

**Answer:** **Azure Database for PostgreSQL Flexible Server** with **DMS Online**

**Why:**
- PostGIS and pg_trgm are supported extensions on Flexible Server
- ⚠️ Custom C extensions are NOT supported — need alternative (Azure VM with PostgreSQL if no alternatives)
- DMS Online for minimal downtime
- Check `pg_available_extensions` on target first
- If custom extensions are critical → SQL Server on Azure VM with PostgreSQL installed

---

### Scenario 10: Regulated Industry SQL Migration

> **A healthcare company needs to migrate SQL Server to Azure. Data must remain encrypted in transit and at rest. They need customer-managed encryption keys and audit logging.**

**Answer:** **Azure SQL Managed Instance** with **DMS Online**

**Why:**
- MI supports TDE with customer-managed keys (CMK via Key Vault)
- Always Encrypted for column-level encryption
- Azure SQL DB also supports CMK, but MI offers more control
- SQL Audit to Azure Storage or Log Analytics
- Private endpoint for network isolation
- DMS Online for minimal downtime
- HIPAA/HITRUST compliance built-in

---

## 8. Quick Reference Trigger Table

> When you see these keywords in an exam question, immediately think of the corresponding answer.

| # | If the Scenario Says... | Think... |
|---|---|---|
| 1 | "Minimize downtime" or "near-zero downtime" | DMS **Online** mode |
| 2 | "Maintenance window available" or "downtime acceptable" | DMS **Offline** mode (simpler) |
| 3 | "SQL Agent Jobs" or "Service Broker" or "CLR" | Azure SQL **Managed Instance** (not SQL DB) |
| 4 | "Full OS access" or "specific SQL version" | SQL Server on **Azure VM** |
| 5 | "Oracle to Azure" | **SSMA for Oracle** → Azure SQL |
| 6 | "MySQL to Azure" | **DMS** → Azure DB for MySQL |
| 7 | "PostgreSQL to Azure" | **DMS** → Azure DB for PostgreSQL |
| 8 | "MongoDB to Azure" | **DMS** → Cosmos DB MongoDB API |
| 9 | "Cassandra to Azure" | **Spark utility** → Cosmos DB Cassandra API |
| 10 | "DynamoDB to Azure" | **ADF pipeline** → Cosmos DB |
| 11 | "Assess compatibility" (SQL Server) | **DMA** (Data Migration Assistant) |
| 12 | "Migrate schema" (Oracle/MySQL/Access/Db2/SAP) | **SSMA** (SQL Server Migration Assistant) |
| 13 | "Lift and shift VMs" | **Azure Site Recovery (ASR)** |
| 14 | "Plan and discover" or "assess infrastructure" | **Azure Migrate** |
| 15 | "Limited bandwidth" + large data (> 40 TB) | **Azure Data Box** |
| 16 | "10 TB+ data" + "fast network" | **AzCopy** + ExpressRoute |
| 17 | "File server migration" + "hybrid access" | **Azure File Sync** |
| 18 | "NAS migration" or "SMB/NFS shares" | **Azure Storage Mover** |
| 19 | "Data warehouse" + "Teradata or Netezza" | **ADF** → Synapse Analytics |
| 20 | "Load data into Synapse" | **COPY INTO** or **PolyBase** |
| 21 | "Small database" + "schema and data export" | **BACPAC** (sqlpackage) |
| 22 | "Hadoop" or "HDFS" | **distcp** → ADLS Gen2 |
| 23 | "Reduce Oracle licensing costs" | **SSMA** → Azure SQL (or ora2pg → PostgreSQL) |
| 24 | "Native backup restore" + "MI" | **Log Replay Service (LRS)** |
| 25 | "ETL during migration" or "transform while migrating" | **Azure Data Factory** |
| 26 | "Data-only transfer" (no schema) | **BCP** (Bulk Copy Program) |
| 27 | "Replication-based" + "SQL Server" | **Transactional Replication** |
| 28 | "Access database modernization" | **SSMA for Access** + Power Apps |
| 29 | "Redis migration" | **RDB Import** or **Dual-write** pattern |
| 30 | "IBM Db2 to Azure" | **SSMA for Db2** |

---

## 9. Common Exam Traps

### Trap 1: DMS Online vs Offline Confusion

| | Online | Offline |
|---|---|---|
| **Downtime** | Minutes (cutover only) | Hours (full migration window) |
| **Complexity** | Higher (requires ongoing sync) | Lower (one-time copy) |
| **Network** | Requires continuous connection | Connection needed only during copy |
| **When to Pick** | "Minimize downtime" in question | "Maintenance window" in question |

**The Trap:** Picking Online when the question says downtime is acceptable (waste of complexity), or Offline when it says minimize downtime.

---

### Trap 2: When Managed Instance is NOT the Answer

MI is **NOT** the answer when:
- Application only uses basic SQL features (pick Azure SQL DB — cheaper, simpler)
- Need > 16 TB storage (pick SQL on Azure VM)
- Need multiple instances on one VM for cost (pick SQL on Azure VM)
- Serverless compute model needed (pick Azure SQL DB Serverless)
- Hyperscale needed with 100 TB+ (pick Azure SQL DB Hyperscale)
- Need full OS access (pick Azure SQL on VM)

MI **IS** the answer when:
- SQL Agent Jobs, Service Broker, CLR, linked servers, cross-DB queries needed
- Instance-scoped collation or TDE with BYOK needed
- Need near-100% SQL Server compatibility with PaaS benefits

---

### Trap 3: Data Box Thresholds

| Scenario | Answer |
|---|---|
| 5 TB, 1 Gbps connection | **AzCopy** — network transfer < 1 day |
| 20 TB, 100 Mbps connection | **Data Box Disk** — network would take > 5 days |
| 50 TB, any bandwidth | **Data Box** — always faster than network |
| 500 TB+ | **Data Box Heavy** — only option at this scale |

**The Trap:** Recommending Data Box for small datasets with good bandwidth, or recommending network transfer for large datasets with limited bandwidth.

**Quick Rule:** Calculate transfer time. If > 1 week → Data Box. If > 500 TB → Data Box Heavy.

---

### Trap 4: SSMA vs DMA vs DMS Confusion

```
          ┌──────────────────────────────────────────────────────┐
          │               Migration Workflow                     │
          │                                                      │
          │  ┌─────┐     ┌──────┐     ┌─────┐                  │
          │  │ DMA │────▶│ Fix  │────▶│ DMS │                  │
          │  │Assess│     │Issues│     │Migrate│                 │
          │  └─────┘     └──────┘     └─────┘                  │
          │  SQL Server    Only         SQL Server               │
          │  → Azure SQL   Step         → Azure PaaS             │
          │                                                      │
          │  ┌──────┐    ┌──────┐     ┌──────┐                  │
          │  │ SSMA │───▶│Convert│───▶│ SSMA │                  │
          │  │Assess│    │Schema │    │Migrate│                  │
          │  └──────┘    └──────┘    └──────┘                   │
          │  Oracle/MySQL/  T-SQL      Data                      │
          │  Access/Db2/    conversion  migration                 │
          │  SAP ASE                                             │
          └──────────────────────────────────────────────────────┘
```

| Tool | Input Source | Purpose | Migrates Data? |
|---|---|---|---|
| **DMA** | SQL Server only | Assess compatibility | ❌ No |
| **DMS** | SQL, MySQL, PostgreSQL, MongoDB | Execute migration | ✅ Yes |
| **SSMA** | Oracle, MySQL, Access, Db2, SAP ASE | Convert schema + migrate | ✅ Yes |

**The Trap:** Using DMA to migrate (it only assesses). Using DMS for Oracle migration (it doesn't convert Oracle schema). Using SSMA for SQL Server → Azure SQL (that's DMA + DMS territory).

---

### Trap 5: Azure SQL DB vs MI vs VM Feature Support

| Feature | SQL DB | SQL MI | SQL VM |
|---|---|---|---|
| SQL Agent Jobs | ❌ (use Elastic Jobs) | ✅ | ✅ |
| Service Broker | ❌ | ✅ | ✅ |
| CLR | ❌ (limited) | ✅ | ✅ |
| Cross-database queries | ❌ (use Elastic Query) | ✅ | ✅ |
| Linked Servers | ❌ | ✅ | ✅ |
| FILESTREAM | ❌ | ❌ | ✅ |
| Full-text Search | ✅ | ✅ | ✅ |
| TDE (service-managed) | ✅ | ✅ | ✅ |
| TDE (customer-managed) | ✅ | ✅ | ✅ |
| Serverless | ✅ | ❌ | ❌ |
| Hyperscale | ✅ | ❌ | ❌ |
| Max storage | 100 TB (Hyperscale) | 16 TB | Unlimited |
| OS access | ❌ | ❌ | ✅ |
| Custom collation | ❌ (DB-level only) | ✅ (instance-level) | ✅ |

---

### Trap 6: Cosmos DB API Selection

| Source Database | Best Cosmos DB API | Why |
|---|---|---|
| MongoDB | MongoDB API (vCore or RU) | Wire protocol compatibility, minimal code changes |
| Cassandra | Cassandra API | CQL compatibility |
| DynamoDB | Table API or NoSQL API | Table API for key-value, NoSQL for richer queries |
| Custom NoSQL | NoSQL API (SQL) | Most flexible, best tooling |

**The Trap:** Always picking NoSQL (SQL) API. Pick the API that matches the source for minimal code changes. Only pick NoSQL API for greenfield or when Rearchitecting.

---

### Trap 7: PostgreSQL Single Server vs Flexible Server

- **Single Server is being retired** — always recommend Flexible Server for new migrations
- Flexible Server has built-in PgBouncer, zone-redundant HA, better performance
- If exam question mentions Single Server → it's testing if you know to recommend Flexible Server instead

---

### Trap 8: DMS Network Requirements

- DMS needs network connectivity to **both** source and target
- For on-premises sources: ExpressRoute, VPN, or public endpoint
- DMS runs in an Azure VNet — must be able to reach source DB
- Common trap: Selecting DMS Online without considering network connectivity

---

## Final Quick Review — The "30-Second" Decision Tree

```
Start: What are you migrating?
│
├── VMs/Servers → Azure Migrate + ASR
│
├── SQL Server → Azure?
│   ├── Need SQL Agent/Service Broker/CLR? → SQL MI (DMS Online)
│   ├── Need OS access or FILESTREAM? → SQL VM (ASR)
│   ├── Need Hyperscale/Serverless? → SQL DB Hyperscale/Serverless
│   └── Basic features sufficient? → SQL DB (DMS Online)
│
├── Oracle → Azure? → SSMA for Oracle → SQL MI/DB
│                      OR ora2pg → Azure PostgreSQL
│
├── MySQL → Azure? → DMS → Azure DB for MySQL (Flexible Server)
│
├── PostgreSQL → Azure? → DMS → Azure DB for PostgreSQL (Flexible Server)
│
├── MongoDB → Azure? → DMS → Cosmos DB (MongoDB API)
│
├── Cassandra → Azure? → Spark → Cosmos DB (Cassandra API)
│
├── DynamoDB → Azure? → ADF → Cosmos DB
│
├── Teradata/Netezza → Azure? → ADF → Synapse Analytics
│
├── Db2/SAP ASE/Access → Azure? → SSMA → Azure SQL
│
├── Files/NAS → Azure? → File Sync (hybrid) or Storage Mover (full)
│
├── Bulk data > 40 TB? → Data Box
│
└── Need transformation during migration? → Azure Data Factory
```

---

> **Last Updated:** Study guide for AZ-305 exam preparation
> **Remember:** The exam tests your ability to **select the right tool and target** for a given scenario. Focus on the decision criteria, not implementation details.
