# Azure Hands-On Lab — Interview Prep (One Day Sprint)

## Scenario
You are setting up a production-like environment for a web application in Azure.
This single exercise covers ALL JD topics: Entra ID, RBAC, Networking, Compute, Monitoring, Backup, Security, Governance, Data Platform.

## Architecture We're Building
```
Entra ID Tenant
  └── Subscription: "Pay-As-You-Go" (or Free Trial)
       ├── rg-prod (Production resource group)
       │    ├── VNet: vnet-prod (10.0.0.0/16)
       │    │    ├── subnet-web (10.0.1.0/24) + NSG
       │    │    └── subnet-data (10.0.2.0/24) + NSG
       │    ├── VM: vm-web-01 (Linux, Nginx)
       │    │    └── Managed Identity → Key Vault access
       │    ├── Key Vault: kv-prod-xxx
       │    ├── Storage Account (ADLS Gen2): stprodxxx
       │    ├── Log Analytics Workspace: law-prod
       │    ├── Recovery Services Vault: rsv-prod
       │    └── Alert Rules + Action Groups
       └── rg-dev (Dev resource group — for RBAC testing)

Users & Groups:
  ├── Group: "Developers" → Reader on rg-prod, Contributor on rg-dev
  └── Group: "Ops" → Contributor on Subscription
```

---

## Progress Tracker

- [ ] Phase 1: Foundation — Subscription, Entra ID, RBAC
- [ ] Phase 2: Networking — VNet, Subnets, NSG
- [ ] Phase 3: Compute — VM Deployment
- [ ] Phase 4: Identity & Security — Managed Identity, Key Vault
- [ ] Phase 5: Monitoring — Log Analytics, Alerts, Action Groups
- [ ] Phase 6: Backup & DR
- [ ] Phase 7: Governance — Tags, Policy, Cost Management
- [ ] Phase 8: Defender for Cloud — Security Posture
- [ ] Phase 9: Data Platform — Storage Account, ADLS Gen2
- [ ] Phase 10: Cleanup & Cost Control

---

## Phase 1: Foundation — Subscription, Entra ID, RBAC

### 1.1 Get a Subscription
- If no subscription: Start Azure Free Trial (gives $200 credit for 30 days)
- Portal: portal.azure.com → search "Subscriptions"

### 1.2 Install Azure CLI (if not installed)
```bash
# macOS
brew install azure-cli

# Login
az login

# Verify
az account show
```

### 1.3 Create Resource Groups
```bash
# Production resource group
az group create --name rg-prod --location eastus --tags Environment=Production Owner=Ops CostCenter=IT

# Dev resource group
az group create --name rg-dev --location eastus --tags Environment=Development Owner=DevTeam CostCenter=Dev
```

### 1.4 Create Users in Entra ID
```bash
# Note: You need your tenant domain. Find it:
az rest --method get --url https://graph.microsoft.com/v1.0/organization --query "value[0].verifiedDomains[0].name" -o tsv

# Create a test user (replace YOUR_DOMAIN)
az ad user create \
  --display-name "Dev User" \
  --user-principal-name devuser@YOUR_DOMAIN \
  --password "TempPass@1234" \
  --force-change-password-next-sign-in true
```

### 1.5 Create Groups
```bash
# Create Developers group
az ad group create --display-name "Developers" --mail-nickname "developers"

# Create Ops group
az ad group create --display-name "Ops" --mail-nickname "ops"

# Add dev user to Developers group
DEV_USER_ID=$(az ad user show --id devuser@YOUR_DOMAIN --query id -o tsv)
DEV_GROUP_ID=$(az ad group show --group "Developers" --query id -o tsv)
az ad group member add --group $DEV_GROUP_ID --member-id $DEV_USER_ID
```

### 1.6 Assign RBAC Roles
```bash
SUBSCRIPTION_ID=$(az account show --query id -o tsv)

# Developers: Reader on rg-prod
az role assignment create \
  --assignee-object-id $DEV_GROUP_ID \
  --assignee-principal-type Group \
  --role "Reader" \
  --scope "/subscriptions/$SUBSCRIPTION_ID/resourceGroups/rg-prod"

# Developers: Contributor on rg-dev
az role assignment create \
  --assignee-object-id $DEV_GROUP_ID \
  --assignee-principal-type Group \
  --role "Contributor" \
  --scope "/subscriptions/$SUBSCRIPTION_ID/resourceGroups/rg-dev"
```

### 1.7 TEST IT — Scenario: Verify RBAC Works
```
1. Open an incognito browser window
2. Go to portal.azure.com
3. Login as devuser@YOUR_DOMAIN
4. Navigate to rg-prod → Try to create a VM → Should FAIL (Reader only)
5. Navigate to rg-dev → Try to create a VM → Should SUCCEED (Contributor)
```
This gives you a real story: "I tested RBAC by logging in as a restricted user and verified they couldn't create resources in production."

### 1.8 Conditional Access (Portal Only — requires Entra ID P1)
```
Entra ID → Security → Conditional Access → New Policy
- Name: "Require MFA for Developers"
- Users: Developers group
- Cloud Apps: All cloud apps
- Grant: Require multi-factor authentication
- Enable: Report-only (test mode first)
```
Note: Free tier may not support Conditional Access. If it doesn't, just know the concept — you've already learned it.

---

## Phase 2: Networking — VNet, Subnets, NSG

### 2.1 Create VNet and Subnets
```bash
# Create VNet
az network vnet create \
  --resource-group rg-prod \
  --name vnet-prod \
  --address-prefix 10.0.0.0/16 \
  --location eastus

# Create web subnet
az network vnet subnet create \
  --resource-group rg-prod \
  --vnet-name vnet-prod \
  --name subnet-web \
  --address-prefix 10.0.1.0/24

# Create data subnet
az network vnet subnet create \
  --resource-group rg-prod \
  --vnet-name vnet-prod \
  --name subnet-data \
  --address-prefix 10.0.2.0/24
```

### 2.2 Create NSGs
```bash
# NSG for web subnet — allow HTTP and SSH
az network nsg create --resource-group rg-prod --name nsg-web

# Allow HTTP (port 80) from anywhere
az network nsg rule create \
  --resource-group rg-prod \
  --nsg-name nsg-web \
  --name AllowHTTP \
  --priority 100 \
  --direction Inbound \
  --access Allow \
  --protocol Tcp \
  --destination-port-ranges 80

# Allow SSH (port 22) from your IP only
az network nsg rule create \
  --resource-group rg-prod \
  --nsg-name nsg-web \
  --name AllowSSH \
  --priority 110 \
  --direction Inbound \
  --access Allow \
  --protocol Tcp \
  --destination-port-ranges 22 \
  --source-address-prefixes $(curl -s ifconfig.me)/32

# NSG for data subnet — deny all inbound from internet, allow from web subnet only
az network nsg create --resource-group rg-prod --name nsg-data

az network nsg rule create \
  --resource-group rg-prod \
  --nsg-name nsg-data \
  --name AllowFromWebSubnet \
  --priority 100 \
  --direction Inbound \
  --access Allow \
  --protocol "*" \
  --source-address-prefixes 10.0.1.0/24 \
  --destination-port-ranges "*"

az network nsg rule create \
  --resource-group rg-prod \
  --nsg-name nsg-data \
  --name DenyAllInternet \
  --priority 200 \
  --direction Inbound \
  --access Deny \
  --protocol "*" \
  --source-address-prefixes Internet \
  --destination-port-ranges "*"

# Attach NSGs to subnets
az network vnet subnet update \
  --resource-group rg-prod \
  --vnet-name vnet-prod \
  --name subnet-web \
  --network-security-group nsg-web

az network vnet subnet update \
  --resource-group rg-prod \
  --vnet-name vnet-prod \
  --name subnet-data \
  --network-security-group nsg-data
```

### 2.3 TEST IT — Scenario: NSG Debugging
After deploying the VM in Phase 3, you'll test:
1. Can you SSH into the VM? (should work — port 22 allowed from your IP)
2. Can you access Nginx via HTTP? (should work — port 80 allowed)
3. Block SSH by removing the rule and verify you lose access
4. This gives you a real debugging story for the interview

---

## Phase 3: Compute — VM Deployment

### 3.1 Create a Linux VM
```bash
az vm create \
  --resource-group rg-prod \
  --name vm-web-01 \
  --image Ubuntu2204 \
  --size Standard_B1s \
  --vnet-name vnet-prod \
  --subnet subnet-web \
  --nsg "" \
  --public-ip-address pip-web-01 \
  --admin-username azureuser \
  --generate-ssh-keys \
  --tags Environment=Production Role=WebServer Owner=Ops
```

### 3.2 Install Nginx
```bash
# Get the public IP
VM_IP=$(az vm show -d --resource-group rg-prod --name vm-web-01 --query publicIps -o tsv)

# SSH in
ssh azureuser@$VM_IP

# Inside the VM:
sudo apt update && sudo apt install -y nginx
sudo systemctl start nginx
sudo systemctl enable nginx
exit
```

### 3.3 TEST IT — Verify HTTP Access
```bash
curl http://$VM_IP
# Should return Nginx welcome page
```

### 3.4 SCENARIO — NSG Troubleshooting Exercise
```bash
# Block HTTP by adding a deny rule with higher priority
az network nsg rule create \
  --resource-group rg-prod \
  --nsg-name nsg-web \
  --name DenyHTTP \
  --priority 90 \
  --direction Inbound \
  --access Deny \
  --protocol Tcp \
  --destination-port-ranges 80

# Test — should timeout now
curl --connect-timeout 5 http://$VM_IP

# "Troubleshoot" — check NSG rules
az network nsg rule list --resource-group rg-prod --nsg-name nsg-web -o table

# Fix — delete the deny rule
az network nsg rule delete --resource-group rg-prod --nsg-name nsg-web --name DenyHTTP

# Verify fixed
curl http://$VM_IP
```
Interview story: "I debugged an NSG issue where a higher-priority deny rule was blocking HTTP traffic. I listed the NSG rules, identified the conflicting rule, removed it, and restored access."

---

## Phase 4: Identity & Security — Managed Identity, Key Vault

### 4.1 Enable Managed Identity on VM
```bash
az vm identity assign --resource-group rg-prod --name vm-web-01
```

### 4.2 Create Key Vault
```bash
# Name must be globally unique — add random suffix
az keyvault create \
  --resource-group rg-prod \
  --name kv-prod-$(openssl rand -hex 3) \
  --location eastus \
  --enable-rbac-authorization true

# Save the vault name
KV_NAME=$(az keyvault list --resource-group rg-prod --query "[0].name" -o tsv)
```

### 4.3 Store a Secret
```bash
az keyvault secret set \
  --vault-name $KV_NAME \
  --name "db-connection-string" \
  --value "Server=db-prod-01;Database=appdb;User=admin;Password=secret123"
```

### 4.4 Grant VM's Managed Identity Access to Key Vault
```bash
VM_IDENTITY=$(az vm show --resource-group rg-prod --name vm-web-01 --query identity.principalId -o tsv)

KV_ID=$(az keyvault show --name $KV_NAME --query id -o tsv)

az role assignment create \
  --assignee-object-id $VM_IDENTITY \
  --assignee-principal-type ServicePrincipal \
  --role "Key Vault Secrets User" \
  --scope $KV_ID
```

### 4.5 TEST IT — Access Secret from VM (No Credentials!)
```bash
ssh azureuser@$VM_IP

# Inside the VM — install Azure CLI
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash

# Login using Managed Identity (no username/password!)
az login --identity

# Read the secret
az keyvault secret show --vault-name KV_NAME_HERE --name db-connection-string --query value -o tsv

exit
```
Interview story: "I configured Managed Identity on a VM to access Key Vault secrets without storing any credentials. The app authenticates automatically using Azure-managed tokens."

---

## Phase 5: Monitoring — Log Analytics, Alerts, Action Groups

### 5.1 Create Log Analytics Workspace
```bash
az monitor log-analytics workspace create \
  --resource-group rg-prod \
  --workspace-name law-prod \
  --location eastus
```

### 5.2 Enable VM Monitoring
```bash
WORKSPACE_ID=$(az monitor log-analytics workspace show \
  --resource-group rg-prod \
  --workspace-name law-prod \
  --query id -o tsv)

# Install Azure Monitor Agent on the VM
az vm extension set \
  --resource-group rg-prod \
  --vm-name vm-web-01 \
  --name AzureMonitorLinuxAgent \
  --publisher Microsoft.Azure.Monitor
```

### 5.3 Create Action Group (Email + Webhook)
```bash
# Create action group — sends email when alert fires
az monitor action-group create \
  --resource-group rg-prod \
  --name ag-ops-team \
  --short-name OpsTeam \
  --action email ops-email "YOUR_EMAIL@gmail.com"

# Optional: Add Slack webhook
# --action webhook slack-notify "https://hooks.slack.com/services/YOUR/WEBHOOK/URL"
```

### 5.4 Create Alert Rule — CPU > 85%
```bash
VM_ID=$(az vm show --resource-group rg-prod --name vm-web-01 --query id -o tsv)
AG_ID=$(az monitor action-group show --resource-group rg-prod --name ag-ops-team --query id -o tsv)

az monitor metrics alert create \
  --resource-group rg-prod \
  --name "alert-cpu-high" \
  --scopes $VM_ID \
  --condition "avg Percentage CPU > 85" \
  --window-size 5m \
  --evaluation-frequency 1m \
  --action $AG_ID \
  --description "CPU above 85% for 5 minutes on vm-web-01" \
  --severity 2
```

### 5.5 TEST IT — Trigger the Alert
```bash
ssh azureuser@$VM_IP

# Spike CPU to trigger alert
sudo apt install -y stress
stress --cpu 2 --timeout 300

exit
```
Wait 5-10 minutes. Check your email for the alert. Then check Azure Monitor → Alerts in the portal.

Interview story: "I set up Azure Monitor alerts with Action Groups. I stress-tested a VM to trigger a CPU alert and verified the notification pipeline end-to-end."

### 5.6 Explore Log Analytics (Portal)
```
Azure Portal → Log Analytics Workspace → law-prod → Logs
Try these KQL queries:

// Heartbeat — verify agent is sending data
Heartbeat
| where TimeGenerated > ago(1h)
| summarize count() by Computer

// Performance counters
Perf
| where ObjectName == "Processor" and CounterName == "% Processor Time"
| where TimeGenerated > ago(1h)
| summarize avg(CounterValue) by Computer, bin(TimeGenerated, 5m)
| render timechart
```

---

## Phase 6: Backup & DR

### 6.1 Create Recovery Services Vault
```bash
az backup vault create \
  --resource-group rg-prod \
  --name rsv-prod \
  --location eastus
```

### 6.2 Enable Backup for VM
```bash
az backup protection enable-for-vm \
  --resource-group rg-prod \
  --vault-name rsv-prod \
  --vm vm-web-01 \
  --policy-name DefaultPolicy
```

### 6.3 Trigger an On-Demand Backup
```bash
az backup protection backup-now \
  --resource-group rg-prod \
  --vault-name rsv-prod \
  --container-name "IaasVMContainer;iaasvmcontainerv2;rg-prod;vm-web-01" \
  --item-name "VM;iaasvmcontainerv2;rg-prod;vm-web-01" \
  --retain-until 2026-06-14
```
Note: Container/item names can be tricky. If the above fails, find them with:
```bash
az backup container list --resource-group rg-prod --vault-name rsv-prod --backup-management-type AzureIaasVM -o table
az backup item list --resource-group rg-prod --vault-name rsv-prod -o table
```

### 6.4 Verify Backup Status
```bash
az backup job list --resource-group rg-prod --vault-name rsv-prod -o table
```

Interview story: "I configured Azure Backup with a Recovery Services Vault, set up backup policies, and validated backups by triggering on-demand backup jobs and monitoring their status."

---

## Phase 7: Governance — Tags, Policy, Cost Management

### 7.1 Verify Tags on Resources
```bash
# List all resources with their tags
az resource list --resource-group rg-prod --query "[].{Name:name, Tags:tags}" -o table
```

### 7.2 Create Azure Policy — Require Tags
```bash
# Assign built-in policy: "Require a tag on resource groups"
az policy assignment create \
  --name "require-env-tag" \
  --display-name "Require Environment tag on Resource Groups" \
  --policy "/providers/Microsoft.Authorization/policyDefinitions/96670d01-0a4d-4649-9c89-2d3abc0a5025" \
  --params '{"tagName": {"value": "Environment"}}' \
  --scope "/subscriptions/$(az account show --query id -o tsv)"
```

### 7.3 TEST IT — Try Creating RG Without Tag
```bash
# This should show non-compliance after policy evaluation
az group create --name rg-test-no-tag --location eastus
# Check compliance in portal: Policy → Compliance
```

### 7.4 Cost Management (Portal)
```
Azure Portal → Cost Management + Billing → Cost Analysis
- View cost by Resource Group
- View cost by Tag
- Set a Budget with alert at 80% threshold
```

Interview story: "I enforced tagging governance using Azure Policy to ensure all resource groups have an Environment tag. This enables accurate cost tracking and resource ownership."

---

## Phase 8: Defender for Cloud

### 8.1 Enable and Review (Portal)
```
Azure Portal → Microsoft Defender for Cloud → Overview
- Check your Secure Score
- Review Recommendations (sorted by severity)
- Common findings: open SSH ports, missing backups, unencrypted storage
```

### 8.2 Review a Recommendation
Pick any recommendation and:
1. Read what it says
2. Understand why it's a risk
3. Check if your setup triggers it (your NSG with SSH open probably will)
4. Remediate it if applicable

Interview story: "I reviewed Defender for Cloud recommendations and found our NSG had SSH open too broadly. I tightened the source IP restriction to our office range."

---

## Phase 9: Data Platform — Storage Account, ADLS Gen2

### 9.1 Create ADLS Gen2 Storage Account
```bash
az storage account create \
  --resource-group rg-prod \
  --name stprod$(openssl rand -hex 3) \
  --location eastus \
  --sku Standard_LRS \
  --kind StorageV2 \
  --hns true \
  --tags Environment=Production

STORAGE_NAME=$(az storage account list --resource-group rg-prod --query "[0].name" -o tsv)
```
`--hns true` enables hierarchical namespace — this is what makes it ADLS Gen2 instead of regular Blob Storage.

### 9.2 Create a Container (Filesystem)
```bash
az storage fs create \
  --name datalake \
  --account-name $STORAGE_NAME \
  --auth-mode login
```

### 9.3 Create Directories and Upload a File
```bash
# Create directory structure
az storage fs directory create \
  --name raw/2026/06/04 \
  --file-system datalake \
  --account-name $STORAGE_NAME \
  --auth-mode login

# Create a sample CSV
echo "id,name,amount,date
1,Customer A,1500,2026-06-01
2,Customer B,2300,2026-06-02
3,Customer C,800,2026-06-03" > /tmp/sample-data.csv

# Upload
az storage fs file upload \
  --source /tmp/sample-data.csv \
  --path raw/2026/06/04/sample-data.csv \
  --file-system datalake \
  --account-name $STORAGE_NAME \
  --auth-mode login
```

### 9.4 Grant VM Managed Identity Access to Storage
```bash
STORAGE_ID=$(az storage account show --name $STORAGE_NAME --query id -o tsv)
VM_IDENTITY=$(az vm show --resource-group rg-prod --name vm-web-01 --query identity.principalId -o tsv)

az role assignment create \
  --assignee-object-id $VM_IDENTITY \
  --assignee-principal-type ServicePrincipal \
  --role "Storage Blob Data Reader" \
  --scope $STORAGE_ID
```

Interview story: "I set up ADLS Gen2 with hierarchical namespace for a data lake, organized raw data by date partitions, and configured Managed Identity for secure access without credentials."

---

## Phase 10: Cleanup

```bash
# DELETE EVERYTHING to avoid charges
az group delete --name rg-prod --yes --no-wait
az group delete --name rg-dev --yes --no-wait
az group delete --name rg-test-no-tag --yes --no-wait

# Delete the test user
az ad user delete --id devuser@YOUR_DOMAIN

# Delete the groups
az ad group delete --group "Developers"
az ad group delete --group "Ops"

# Remove policy assignment
az policy assignment delete --name "require-env-tag"
```

---

## Interview Stories Summary

| Topic | Your Story |
|-------|-----------|
| Entra ID / RBAC | "I created user groups with scoped RBAC — Contributor on dev, Reader on prod — and verified by logging in as the restricted user" |
| NSG Debugging | "I debugged a higher-priority deny rule blocking HTTP. Listed NSG rules, found the conflict, removed it, restored access" |
| Managed Identity | "I configured Managed Identity on a VM to access Key Vault and Storage — zero credentials stored" |
| Monitoring | "I set up Azure Monitor alerts with Action Groups, stress-tested a VM to trigger CPU alert, verified notifications end-to-end" |
| Backup | "I configured Recovery Services Vault, set backup policies, triggered on-demand backups, and validated backup jobs" |
| Governance | "I enforced tagging with Azure Policy, used Cost Management to track spend by resource group and tag" |
| Security | "I reviewed Defender for Cloud Secure Score, found open SSH in NSG, tightened source IP restriction" |
| Data Platform | "I set up ADLS Gen2 with hierarchical namespace, organized data by date partitions, secured access with Managed Identity" |
