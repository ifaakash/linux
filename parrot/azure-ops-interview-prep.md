# Azure Operations Engineer - 2-Day Interview Prep

## Your Strategy
You have strong AWS fundamentals. The interview approach:
- **Map AWS concepts to Azure equivalents** (shows you learn fast)
- **Be honest** about AWS-primary experience, frame Azure as "returning to" not "learning"
- **Emphasize transferable skills**: Terraform, CI/CD, monitoring philosophy, incident response

---

## DAY 1: HIGH PRIORITY TOPICS

---

### 1. MICROSOFT ENTRA ID (formerly Azure AD) — YOUR #1 PRIORITY

**AWS Equivalent**: AWS IAM + AWS SSO + AWS Organizations

**What it is**: Microsoft's cloud identity and access management service. Every Azure subscription is tied to an Entra ID tenant.

**Key concepts you MUST know:**

| Concept | What it does | AWS Equivalent |
|---------|-------------|----------------|
| Tenant | Top-level identity boundary | AWS Account / Organization |
| Users | Individual identities | IAM Users |
| Groups | Collection of users for bulk access | IAM Groups |
| App Registrations | Identity for applications | IAM Roles for services |
| Service Principals | Runtime identity of an app registration | IAM Role assumed by service |
| Managed Identity | Auto-managed identity for Azure resources (no secrets) | IAM Role attached to EC2/ECS |
| RBAC (Role-Based Access Control) | Assign roles at scope (subscription/RG/resource) | IAM Policies attached at various levels |
| Conditional Access | Policies like "block login from outside India" or "require MFA for admins" | AWS SSO + SCP combined |
| SSO (Single Sign-On) | One login for multiple apps | AWS SSO |
| PIM (Privileged Identity Management) | Just-in-time admin access (request elevation, time-limited) | No direct AWS equivalent |

**RBAC - How it works:**
- RBAC = Who (user/group/SP) + What (role: Reader/Contributor/Owner) + Where (scope: subscription/RG/resource)
- Built-in roles: **Owner** (full access + can assign roles), **Contributor** (full access, can't assign roles), **Reader** (view only)
- Custom roles possible
- Principle of Least Privilege (LPP) — same as you follow in AWS

**Conditional Access — Interview favourite:**
- Policies evaluate: User/Group + App + Conditions (location, device, risk) → Grant/Block + Controls (require MFA, compliant device)
- Example: "All admins must use MFA from any location. Regular users can skip MFA from office IP range."

**Managed Identity — Will likely be asked:**
- **System-assigned**: Tied to one resource, deleted when resource is deleted
- **User-assigned**: Independent lifecycle, can be shared across resources
- Use case: VM needs to read from Key Vault → assign Managed Identity → grant Key Vault Reader role → no passwords stored anywhere

**Interview Q&A:**
- Q: "How do you give a VM access to a storage account without storing credentials?"
  A: Assign a Managed Identity to the VM, then grant Storage Blob Data Reader role on the storage account to that identity.

- Q: "A user can't access a resource. How do you troubleshoot?"
  A: Check: 1) Does user have RBAC role at correct scope? 2) Any Deny assignments? 3) Conditional Access blocking? 4) Check sign-in logs in Entra ID.

---

### 2. AZURE MONITOR & LOG ANALYTICS

**AWS Equivalent**: CloudWatch + CloudWatch Logs + DataDog (you know this well)

**Architecture:**
```
Azure Resources → Diagnostic Settings → Log Analytics Workspace
                                      → Metrics (Azure Monitor)
                                      → Alerts → Action Groups (email/SMS/webhook)
```

**Key components:**

| Component | What it does | AWS/Your Equivalent |
|-----------|-------------|---------------------|
| Azure Monitor | Umbrella monitoring platform | CloudWatch |
| Metrics | Numeric time-series data (CPU, memory, etc.) | CloudWatch Metrics |
| Log Analytics Workspace | Central log store, query with KQL | CloudWatch Logs / DataDog Log Explorer |
| KQL (Kusto Query Language) | Query language for logs | DataDog query language / CloudWatch Insights |
| Alerts | Rules that fire on metric/log conditions | CloudWatch Alarms / DataDog Monitors |
| Action Groups | What happens when alert fires (email, SMS, webhook, runbook) | SNS Topics / DataDog Notifications |
| Application Insights | APM for applications | DataDog APM |
| Workbooks | Dashboards and reports | Grafana Dashboards / DataDog Dashboards |

**KQL basics you should know (not deep, just read these):**
```kql
// Find errors in last 24 hours
AzureActivity
| where TimeGenerated > ago(24h)
| where Level == "Error"
| project TimeGenerated, OperationName, Caller, ResourceGroup

// VM CPU over 90%
Perf
| where ObjectName == "Processor" and CounterName == "% Processor Time"
| where CounterValue > 90
| summarize avg(CounterValue) by Computer, bin(TimeGenerated, 5m)

// Count events by type
Event
| summarize count() by EventLevelName
| order by count_ desc
```

**Interview Q&A:**
- Q: "Application is slow. How do you investigate?"
  A: Check Azure Monitor metrics (CPU, memory, disk IO). Check Log Analytics for application errors. Check Application Insights for request latency and dependency failures. Check NSG flow logs for network issues.

- Q: "How do you set up monitoring for a new VM?"
  A: Enable Diagnostic Settings to send logs to Log Analytics workspace. Install Azure Monitor Agent. Create alert rules for CPU > 85%, disk > 90%, memory > 90%. Attach action group to notify the team.

---

### 3. AZURE BACKUP & DISASTER RECOVERY

**AWS Equivalent**: AWS Backup + S3 Versioning/Replication + pilot light DR

**Azure Backup:**
- Backs up VMs, SQL DBs, File Shares, Blobs
- Central service: **Recovery Services Vault** (the container for all backups)
- Backup Policy = schedule (daily/weekly) + retention (how long to keep)
- Supports application-consistent snapshots (VSS on Windows, pre/post scripts on Linux)

**Key concepts:**
- **RPO (Recovery Point Objective)**: How much data can you afford to lose? (e.g., 1 hour = backup every hour)
- **RTO (Recovery Time Objective)**: How fast must you recover? (e.g., 4 hours)
- **Backup types**: Full, Incremental (Azure mostly does incremental)
- **Soft delete**: Deleted backups retained for 14 extra days (protection against accidental/malicious deletion)

**Azure Site Recovery (ASR):**
- DR solution — replicates VMs to a secondary Azure region
- Continuous replication (not just backups)
- **Failover**: Switch to DR region when primary is down
- **Failback**: Return to primary region after recovery
- **DR Drill/Test Failover**: Test DR without impacting production (critical for compliance)

**Interview Q&A:**
- Q: "Primary region goes down. Walk me through DR."
  A: ASR continuously replicates VMs to secondary region. Initiate failover from Recovery Services Vault. Update DNS/Traffic Manager to point to DR region. Validate services. When primary recovers, reprotect and failback.

- Q: "How do you validate backups?"
  A: Regular test restores. Check backup jobs in Recovery Services Vault for failures. Set up alerts for failed backup jobs. Participate in scheduled DR drills.

---

### 4. ADLS GEN2 & MICROSOFT FABRIC

**ADLS Gen2 (Azure Data Lake Storage Gen2):**
- Built on top of Azure Blob Storage + hierarchical namespace (real folders, not just prefixes like S3)
- Used for big data analytics workloads
- Supports RBAC + ACLs (Access Control Lists) at folder/file level
- You already know Blob Storage — ADLS Gen2 is Blob Storage with folder structure and better performance for analytics
- Works with: Data Factory (you've used this!), Databricks, Synapse, Fabric

**Microsoft Fabric — basics only (it's new, even interviewers may not go deep):**
- All-in-one analytics platform by Microsoft (announced 2023)
- Combines: Data Factory + Synapse + Power BI + Data Lake into ONE platform
- Key components:
  - **Lakehouse**: Combines data lake (files) + data warehouse (SQL tables). Store raw files, query with SQL.
  - **Warehouse**: Traditional SQL data warehouse in Fabric
  - **Data Pipelines**: ETL/ELT — similar to Azure Data Factory (you've used ADF, so this maps well)
  - **OneLake**: Single data lake for entire organization (like a unified S3 bucket for all analytics)
- **Your angle in interview**: "I've worked with Azure Data Factory for ETL pipelines and Blob Storage. Fabric consolidates these into a unified platform — I understand the underlying concepts and can ramp up quickly on the Fabric UI."

**SQL — since the JD emphasizes it:**
- Be ready to write: JOINs, GROUP BY, HAVING, window functions (ROW_NUMBER, RANK), CTEs
- Data validation queries: finding duplicates, NULL checks, row counts between source and target
- Example: "After a pipeline loads data, I'd validate with COUNT comparisons, check for NULLs in key columns, and verify date ranges match the expected window."

---

## DAY 2: MEDIUM PRIORITY TOPICS

---

### 5. AZURE NETWORKING (Quick AWS Mapping)

| Azure | AWS | Notes |
|-------|-----|-------|
| Virtual Network (VNet) | VPC | Same concept — isolated network |
| Subnet | Subnet | Same — but Azure has no "public/private" label. Controlled by route tables and NSGs |
| NSG (Network Security Group) | Security Group | Stateful rules, but NSGs have explicit priority numbers (lower = higher priority) |
| ASG (Application Security Group) | Security Group tagging | Group VMs logically, reference in NSG rules |
| Azure Load Balancer | NLB | Layer 4 (TCP/UDP) |
| Application Gateway | ALB | Layer 7 (HTTP/HTTPS), also has WAF |
| Azure DNS | Route 53 | DNS hosting |
| Azure Firewall | AWS Network Firewall | Central firewall, more feature-rich than NSGs |
| VNet Peering | VPC Peering | Connect two VNets |
| VPN Gateway | VPN Gateway | Site-to-site VPN to on-prem |
| ExpressRoute | Direct Connect | Private dedicated connection to Azure |
| NAT Gateway | NAT Gateway | Outbound internet for private subnets |
| Private Endpoint | VPC Endpoint | Access PaaS services privately |

**Key difference from AWS**: Azure subnets are NOT inherently public or private. A subnet is "private" because its route table doesn't have a route to an Internet Gateway, and its NSG blocks inbound internet traffic. Same concept, different implementation.

**NSG Rule Structure**: Priority (100-4096, lower = first evaluated) + Source + Destination + Port + Protocol + Allow/Deny

**Interview scenario:**
- Q: "VM in a subnet can't reach the internet. Troubleshoot."
  A: Check: 1) Route table — is there a 0.0.0.0/0 route to Internet or NAT Gateway? 2) NSG — outbound rules allowing traffic? 3) Public IP attached or NAT Gateway on subnet? 4) Azure Firewall blocking if using forced tunneling?

---

### 6. MICROSOFT DEFENDER FOR CLOUD

**AWS Equivalent**: Security Hub + GuardDuty + Inspector combined

**What it does:**
- **CSPM (Cloud Security Posture Management)**: Scans your Azure environment, gives a "Secure Score" (0-100%), lists recommendations
- **CWP (Cloud Workload Protection)**: Runtime threat detection for VMs, containers, databases, storage
- Recommendations like: "Enable MFA for accounts with owner permissions", "Encrypt storage at rest", "Restrict SSH access in NSGs"

**Interview-safe answer**: "It provides continuous security assessment with a Secure Score, actionable recommendations, and threat detection. I'd review the score regularly, prioritize critical recommendations, and work with the team to remediate findings — similar to how Security Hub works in AWS."

---

### 7. ITIL BASICS

Since this is an MSP role, know these:

| Process | What it is | Key point |
|---------|-----------|-----------|
| **Incident Management** | Restore service ASAP | Focus on speed, not root cause |
| **Problem Management** | Find root cause to prevent recurrence | RCA, known error database |
| **Change Management** | Control changes to production | CAB (Change Advisory Board), RFC (Request for Change), types: Standard/Normal/Emergency |
| **Service Request** | Pre-approved routine tasks (e.g., create user, add disk) | Fulfilled from service catalog |

**Change types:**
- **Standard**: Pre-approved, low risk (e.g., add a user). No CAB needed.
- **Normal**: Needs assessment and CAB approval. Scheduled maintenance window.
- **Emergency**: Urgent fix for critical incident. Retrospective CAB approval.

---

### 8. AZURE RESOURCE GOVERNANCE

**Key concepts:**
- **Management Groups** → **Subscriptions** → **Resource Groups** → **Resources** (hierarchy)
- **Tags**: Key-value pairs on resources (Environment:Prod, CostCenter:IT, Owner:TeamX). Critical for cost tracking.
- **Azure Policy**: Enforce rules like "all resources must have tags", "only allowed VM sizes", "only deploy in these regions"
- **Cost Management**: Azure Cost Management + Billing. Set budgets, alerts, analyze spending by tag/RG/subscription.

**Your angle**: "In AWS I manage cost with tagging and Terraform. Same in Azure — tag everything, use Azure Policy to enforce tagging, monitor with Cost Management, and right-size VMs using Azure Advisor recommendations."

---

### 9. AZURE COMPUTE & VM OPERATIONS

You've used Azure VMs before. Quick refresher:

- **VM Scale Sets (VMSS)**: Auto-scaling group of identical VMs (like AWS ASG)
- **Availability Sets**: Distribute VMs across fault/update domains within a datacenter
- **Availability Zones**: Distribute across physically separate datacenters in a region (like AWS AZs)
- **Azure App Service**: PaaS for web apps — you deploy code, Azure manages infra (like AWS Elastic Beanstalk)
- **Resizing a VM**: Stop → Change size → Start (some sizes available without stop, but safest to stop)

---

### 10. POWERSHELL — SURVIVAL KIT (just in case)

You prefer Bash, and Azure CLI uses Bash syntax. But know these PowerShell equivalents exist:

```powershell
# Azure CLI (Bash - your comfort zone):
az vm list --output table
az vm start --resource-group myRG --name myVM
az group list --output table

# PowerShell equivalent (just be aware):
Get-AzVM
Start-AzVM -ResourceGroupName "myRG" -Name "myVM"
Get-AzResourceGroup
```

**Interview angle**: "I primarily use Azure CLI and Bash for automation. I can read and modify PowerShell scripts but my go-to is Azure CLI, which achieves the same outcomes."

---

## INTERVIEW SCENARIOS — PRACTICE THESE

### Scenario 1: Production Outage
"Users report the web application is down. Walk me through your troubleshooting."
1. Check Azure Monitor alerts — any triggered?
2. Check Application Insights — is the app responding? Error rates?
3. Check VM/App Service health — is the compute healthy?
4. Check NSG / Load Balancer — is traffic reaching the backend?
5. Check Log Analytics — application logs showing errors?
6. Check recent changes — was anything deployed? (Change Management)
7. Communicate status to stakeholders, update incident ticket
8. Resolve → document RCA → create problem ticket if root cause needs deeper fix

### Scenario 2: New Application Deployment
"A new team needs a web application hosted in Azure. How do you set it up?"
1. Resource Group with proper tags (env, owner, cost center)
2. VNet + Subnet + NSG (network isolation)
3. App Service or VMs depending on requirements
4. Application Gateway / Load Balancer for traffic distribution
5. Azure Monitor + Log Analytics for observability
6. Azure Backup configured with appropriate RPO
7. Entra ID — Managed Identity for the app, RBAC for the team
8. Azure Policy to enforce compliance
9. Terraform to codify everything (your strength!)

### Scenario 3: Security Incident
"Defender for Cloud flags a VM with exposed SSH to the internet."
1. Verify the finding in Defender for Cloud
2. Check NSG — is port 22 open to 0.0.0.0/0?
3. Immediate fix: Restrict SSH to specific IPs or use Azure Bastion
4. Check sign-in logs — any unauthorized access?
5. If compromised: isolate VM, snapshot for forensics, rebuild from backup
6. Update policy to prevent recurrence (Azure Policy: deny public SSH)
7. Document in incident ticket, update runbook

---

## KEY PHRASES TO USE IN INTERVIEW

- "In AWS I did X, the Azure equivalent is Y — I can ramp up quickly"
- "I follow the principle of least privilege" (you already do this)
- "Infrastructure as Code with Terraform — cloud-agnostic, works across AWS and Azure"
- "SLA compliance and MTTR are my primary operational metrics"
- "I believe in documenting runbooks so L1 can handle known issues independently"
- "Monitoring is not just alerting — it's about reducing noise and having actionable alerts" (your DataDog cost optimization experience!)

---

## QUICK REFERENCE: AWS → AZURE NAMING

| AWS | Azure |
|-----|-------|
| EC2 | Virtual Machine |
| ASG | VM Scale Set |
| VPC | VNet |
| Security Group | NSG |
| IAM | Entra ID + RBAC |
| CloudWatch | Azure Monitor |
| CloudWatch Logs | Log Analytics |
| S3 | Blob Storage |
| S3 (analytics) | ADLS Gen2 |
| RDS | Azure SQL |
| ECS/EKS | ACI / AKS |
| Route 53 | Azure DNS |
| ALB | Application Gateway |
| NLB | Azure Load Balancer |
| Direct Connect | ExpressRoute |
| AWS Backup | Azure Backup (Recovery Services Vault) |
| Security Hub | Defender for Cloud |
| CloudFormation | ARM Templates / Bicep |
| AWS SSO | Entra ID + SSO |
| SNS | Action Groups |
| CloudTrail | Activity Log |
| AWS Config | Azure Policy |
| Cost Explorer | Cost Management |
