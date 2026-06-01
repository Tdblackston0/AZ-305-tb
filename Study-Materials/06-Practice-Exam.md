# AZ-305 Practice Exam

This practice exam is **modeled after the style and decision-making patterns** used in AZ-305. It is scenario-based and focuses on choosing the **best** solution based on requirements, trade-offs, and constraints.

---

## 📋 Exam Instructions

- **Questions:** 40
- **Recommended time:** 100-120 minutes
- **Goal:** Practice architecture decisions, not memorization
- **Method:** Read each scenario carefully and choose the **best** answer

### Domain Coverage
| Domain | Questions |
|--------|-----------|
| Infrastructure Solutions | 1-12 |
| Identity, Governance, Monitoring | 13-22 |
| Data Storage and Integration | 23-32 |
| Business Continuity Solutions | 33-40 |

---

## Questions

### Infrastructure Solutions

**1.** A company is migrating a public-facing web application to Azure. The app must support autoscaling, use built-in deployment slots, and minimize operating system management. Which service should you recommend?

A. Azure Virtual Machines  
B. Azure App Service  
C. Azure Kubernetes Service (AKS)  
D. Azure Batch

**2.** A manufacturing company runs a legacy Windows application that requires custom COM components and direct access to the operating system. The solution must support manual scaling and custom patch scheduling. Which compute platform is most appropriate?

A. Azure Functions  
B. Azure Container Apps  
C. Azure Virtual Machines  
D. Azure Logic Apps

**3.** A solution will host 20 microservices with independent release cycles and scaling requirements. The operations team already has Kubernetes experience. Which hosting option is the best fit?

A. Azure App Service Environment  
B. Azure Kubernetes Service (AKS)  
C. Azure Virtual Machine Scale Sets  
D. Azure Container Instances

**4.** A global retail application serves users from North America, Europe, and Asia. The application requires layer 7 routing, web application firewall protection, and the ability to direct users to the nearest healthy region. What should you use?

A. Azure Load Balancer  
B. Azure Front Door  
C. Azure Traffic Manager only  
D. Azure Bastion

**5.** An internal line-of-business application uses TCP on a custom port and must distribute traffic across virtual machines in a single region. URL-based routing is not required. Which service should you recommend?

A. Azure Application Gateway  
B. Azure Front Door  
C. Azure Load Balancer  
D. Azure API Management

**6.** A company needs private connectivity between its on-premises datacenter and Azure for a mission-critical SAP workload. The business requires predictable latency and does not want traffic sent over the public internet. What is the best solution?

A. Site-to-Site VPN  
B. Point-to-Site VPN  
C. ExpressRoute  
D. Azure Firewall

**7.** A three-tier application is hosted in Azure. The database tier must not be reachable from the internet, and only the application subnet should communicate with it. What should you recommend first?

A. Place all resources in one subnet  
B. Apply NSGs and subnet segmentation  
C. Add a public IP to the database tier  
D. Use Azure Bastion for all traffic

**8.** A web application must remain available if one instance fails, and the company wants Azure to automatically increase or decrease the number of instances based on CPU load. Which option should you choose?

A. Availability Set only  
B. Single VM with premium storage  
C. VM Scale Set with autoscale  
D. Azure Firewall Manager

**9.** An API backend is exposed only to applications in a VNet. The design must avoid public endpoints and keep traffic on the Microsoft backbone. Which design best meets the requirement?

A. Public endpoint with IP restrictions  
B. Private Endpoint  
C. Service Endpoint from the internet  
D. Azure CDN

**10.** A company must support a blue-green deployment strategy for a web application with minimal downtime and quick rollback. Which feature best meets the requirement?

A. App Service deployment slots  
B. VM availability sets  
C. Azure Policy exemptions  
D. ExpressRoute FastPath

**11.** Developers need a messaging platform that decouples application components and supports reliable asynchronous communication between services. Which Azure service is the best choice?

A. Azure Front Door  
B. Azure Service Bus  
C. Azure Firewall  
D. Azure Files

**12.** A solution must cache frequently requested content close to users worldwide to reduce latency and offload the origin application. What should you recommend?

A. Azure CDN  
B. Azure NAT Gateway  
C. Azure DDoS Protection  
D. Azure Route Server

### Identity, Governance, and Monitoring

**13.** A company wants to organize Azure subscriptions by department and enforce policies across all subscriptions in a department. What should you use?

A. Resource groups  
B. Management groups  
C. Availability zones  
D. Application security groups

**14.** Developers must be allowed to start and restart virtual machines but must not be able to delete them. The organization wants to follow least privilege. What is the best approach?

A. Assign Owner at the subscription scope  
B. Assign Contributor at the resource group scope  
C. Create a custom RBAC role  
D. Use a resource lock only

**15.** The security team must prevent creation of storage accounts that do not require secure transfer. The control must block noncompliant deployments automatically. What should you recommend?

A. Azure Monitor alert  
B. Azure Policy with Deny effect  
C. Microsoft Sentinel workbook  
D. Cost Management budget

**16.** An administrator needs to review who deleted a resource group last week and from which identity. Where should the administrator look first?

A. NSG flow logs  
B. Azure Activity Log  
C. Azure Advisor  
D. Azure Policy compliance view

**17.** A company wants to allow administrators to access the Azure portal only from compliant devices and only when sign-in risk is low. What should you recommend?

A. Conditional Access  
B. Resource locks  
C. Azure Reservations  
D. Update Management

**18.** A web application running on App Service must access secrets in Azure Key Vault without storing credentials in code. Which design is best?

A. Store the secret in appsettings.json  
B. Use a managed identity  
C. Embed a service principal secret in the deployment pipeline  
D. Place the secret in a storage account container

**19.** The operations team needs centralized queries across logs from multiple Azure resources and applications. They also want Kusto Query Language support. Which service should back the design?

A. Log Analytics workspace  
B. Azure Policy initiative  
C. Azure Resource Graph only  
D. Azure Blueprints

**20.** A solution architect must ensure that every production resource has tags for CostCenter and Environment. Resources missing the tags should be flagged, and compliant resources should be reported centrally. What should be used?

A. Azure Policy  
B. Network security groups  
C. Azure Bastion  
D. Application Gateway WAF policy

**21.** A company wants to reduce the standing access of global administrators and require approval before elevation to privileged roles. Which service capability should you recommend?

A. Azure Monitor autoscale  
B. Privileged Identity Management  
C. Service Health alerts  
D. Azure Arc

**22.** Developers need application-level telemetry such as request rates, dependency failures, and end-to-end transaction details for a distributed application. What should you use?

A. Application Insights  
B. Azure Cost Management  
C. DDoS Protection Standard  
D. Azure Policy

### Data Storage and Integration

**23.** An application stores relational financial records and requires ACID transactions, predictable schema, and strong consistency. Which data store should you recommend?

A. Azure Cosmos DB  
B. Azure SQL Database  
C. Azure Table Storage  
D. Azure Cache for Redis

**24.** A globally distributed shopping cart application requires single-digit millisecond latency, horizontal scaling, and flexible JSON documents. Which service is the best fit?

A. Azure SQL Managed Instance  
B. Azure Blob Storage  
C. Azure Cosmos DB  
D. Azure Files

**25.** A company needs to ingest large amounts of raw data into a centralized analytics platform, then transform and curate the data later for reporting and machine learning. Which storage design should you recommend?

A. Azure Data Lake Storage Gen2 with layered zones  
B. Azure Files with premium tier  
C. Azure NetApp Files  
D. Azure Queue Storage

**26.** An enterprise needs to orchestrate scheduled data movement from on-premises SQL Server, SaaS applications, and Azure storage into a cloud analytics environment. What should you use?

A. Azure Data Factory  
B. Azure Load Balancer  
C. Azure Bastion  
D. Azure Firewall

**27.** A solution must process millions of telemetry events per minute with near real-time analytics dashboards. Which architecture is best?

A. Daily batch import with AzCopy  
B. Event streaming with Event Hubs and Stream Analytics  
C. Azure Files replication  
D. SQL transactional replication only

**28.** A company wants to minimize storage costs for blobs that are rarely accessed for 90 days but must remain immediately retrievable when needed. Which access tier should you recommend?

A. Premium  
B. Hot  
C. Cool  
D. Archive

**29.** Architects need a backup copy of blobs in a secondary region that can be read even if the primary region is unavailable. Which redundancy option best meets the requirement?

A. LRS  
B. ZRS  
C. GRS  
D. RA-GRS

**30.** A reporting database must scale read workloads without affecting writes to the primary OLTP database. Which option is best?

A. Put all workloads on one larger database  
B. Use read replicas or read scale-out  
C. Move the database to Blob Storage  
D. Use Azure Policy to distribute queries

**31.** A company wants to migrate existing SSIS-style ETL processes to Azure with the least redesign effort and centralized pipeline orchestration. Which service should be recommended?

A. Azure Data Factory  
B. Azure DNS  
C. Azure Front Door  
D. Azure Backup

**32.** Data engineers need to query files directly in a data lake using SQL or Spark without first moving data into a traditional database. Which service should you recommend?

A. Azure Synapse Analytics  
B. Azure Automation  
C. Azure Load Testing  
D. Azure Policy

### Business Continuity Solutions

**33.** A mission-critical application requires an RPO of less than 5 minutes for its relational database and automatic failover to a paired region. Which design is the best fit?

A. Manual export to storage account once per day  
B. Active geo-replication or failover groups  
C. Local backup to the VM disk  
D. Availability set only

**34.** A company wants to protect Azure virtual machines with centralized backup policies, retention settings, and recovery points managed from Azure. What should you recommend?

A. Azure Backup  
B. Azure DevOps  
C. Azure Policy Guest Configuration  
D. Azure Monitor metrics

**35.** Architects are designing a multi-region web application. The solution must continue serving traffic if an entire Azure region fails. Which approach best meets the goal?

A. Deploy to one region with larger VM sizes  
B. Use Availability Zones in a single region only  
C. Deploy active-active or active-passive across regions with global routing  
D. Use one storage account with LRS

**36.** The business states that workloads can be offline for up to 4 hours after a disaster, and data loss of up to 15 minutes is acceptable. What should architects document first when selecting a recovery design?

A. CPU utilization and memory thresholds  
B. RTO and RPO requirements  
C. Number of subscriptions  
D. Azure Advisor recommendations

**37.** A company needs to replicate on-premises VMware virtual machines to Azure so they can be failed over during a site outage. Which service is most appropriate?

A. Azure Site Recovery  
B. Azure Policy  
C. Azure Cost Management  
D. Azure Lighthouse

**38.** A storage solution must remain available even if a datacenter in the primary region fails, but cross-region failover is not required. Which redundancy option is best?

A. LRS  
B. ZRS  
C. GRS  
D. RA-GRS

**39.** An architect must design a backup strategy for Azure SQL Database that supports long-term retention for compliance while minimizing operational overhead. Which approach should be used?

A. Native automated backups with long-term retention  
B. Manual BACPAC export every month only  
C. Daily VM snapshots  
D. Store query results in Blob Storage

**40.** A company wants to regularly validate disaster recovery procedures without affecting the production environment. What should be included in the design?

A. Remove the secondary region to reduce cost  
B. Periodic failover testing  
C. Disable backups during peak periods  
D. Use a single-region deployment

---

## ✅ Answer Key and Rationales

| Q | Answer | Why |
|---|--------|-----|
| 1 | B | App Service provides managed hosting, autoscale, and deployment slots with low operational overhead. |
| 2 | C | VMs are the best fit when full OS access and legacy component support are required. |
| 3 | B | AKS is designed for microservices with independent deployment and scaling. |
| 4 | B | Front Door provides global layer 7 routing, WAF integration, and regional failover. |
| 5 | C | Load Balancer is the right choice for regional layer 4 TCP/UDP traffic distribution. |
| 6 | C | ExpressRoute provides private connectivity with more predictable performance than VPN. |
| 7 | B | Subnet segmentation with NSGs is the foundational design for restricting east-west traffic. |
| 8 | C | VM Scale Sets support autoscaling and improved availability across multiple instances. |
| 9 | B | Private Endpoint keeps access private and on the Microsoft network backbone. |
| 10 | A | Deployment slots support blue-green style swaps and rapid rollback. |
| 11 | B | Service Bus supports reliable asynchronous messaging and decoupled architecture. |
| 12 | A | CDN reduces latency by caching content close to users globally. |
| 13 | B | Management groups let you organize subscriptions and apply governance at scale. |
| 14 | C | A custom RBAC role gives only the required VM actions without delete permissions. |
| 15 | B | Azure Policy with Deny blocks noncompliant resources at deployment time. |
| 16 | B | The Activity Log records subscription-level control plane operations such as deletions. |
| 17 | A | Conditional Access evaluates device compliance and sign-in risk before allowing access. |
| 18 | B | Managed identities let the app access Key Vault without embedded credentials. |
| 19 | A | Log Analytics provides centralized querying and KQL across many data sources. |
| 20 | A | Azure Policy can audit and enforce required tags across resources. |
| 21 | B | PIM supports just-in-time elevation, approval workflows, and reduced standing privilege. |
| 22 | A | Application Insights captures request, dependency, and distributed tracing telemetry. |
| 23 | B | Azure SQL Database is ideal for relational workloads needing ACID transactions. |
| 24 | C | Cosmos DB fits globally distributed, low-latency, JSON-based workloads. |
| 25 | A | ADLS Gen2 with raw/processed/curated zones supports scalable analytics patterns. |
| 26 | A | Data Factory is built for pipeline orchestration across hybrid and cloud data sources. |
| 27 | B | Event Hubs plus Stream Analytics supports near real-time event ingestion and processing. |
| 28 | C | Cool tier lowers cost for infrequently accessed data that still needs fast retrieval. |
| 29 | D | RA-GRS provides secondary-region replication with read access to the secondary copy. |
| 30 | B | Read replicas or read scale-out separate reporting reads from OLTP writes. |
| 31 | A | Data Factory is the closest managed orchestration option for existing ETL workflows. |
| 32 | A | Synapse supports querying data lake files with SQL and Spark engines. |
| 33 | B | Geo-replication or failover groups best support low RPO and regional failover for SQL. |
| 34 | A | Azure Backup centralizes policy-based VM backup and recovery management. |
| 35 | C | Multi-region deployment plus global routing is required for regional failure resilience. |
| 36 | B | RTO and RPO define acceptable downtime and data loss and drive DR architecture. |
| 37 | A | Azure Site Recovery replicates and orchestrates failover for on-premises VMs to Azure. |
| 38 | B | ZRS protects against datacenter failure within a region without cross-region replication. |
| 39 | A | Automated SQL backups with long-term retention meet compliance needs with minimal ops. |
| 40 | B | DR plans must include regular failover testing to validate recovery procedures. |

---

## 📊 Score Interpretation

| Score | Meaning |
|------|---------|
| 36-40 correct | Strong exam readiness |
| 30-35 correct | Close, but review weak domains |
| 24-29 correct | Moderate risk - focus on gaps before exam |
| Below 24 correct | Revisit core study guides before retaking |

### Domain Review Guide
- **Missed infrastructure questions?** Revisit [05-Infrastructure-Solutions.md](05-Infrastructure-Solutions.md) and [02-App-Architecture.md](02-App-Architecture.md)
- **Missed governance or monitoring questions?** Revisit [01-Governance.md](01-Governance.md) and [04-Identity-Governance-Monitoring.md](04-Identity-Governance-Monitoring.md)
- **Missed storage or integration questions?** Revisit [03-Data-Integration.md](03-Data-Integration.md)
- **Missed continuity questions?** Review the business continuity and infrastructure materials in the repository

---

## 🎯 How to Use This Practice Exam

1. Take it once under timed conditions.
2. Review every wrong answer and explain **why** the correct option is best.
3. Revisit the related study guide before taking it again.
4. Focus on trade-offs: **cost, security, operations, performance, and resilience**.

---

**Tip:** If two answers seem correct, choose the one that best matches the **primary business requirement** with the **least unnecessary complexity**.
