# Azure Hands-On Lab — Interview Prep (One Day Sprint)

## Scenario
You are an ops engineer setting up a production-like environment for a web application in Azure.
This exercise covers ALL JD topics through hands-on work + concept sections for things that can't be labbed cheaply in a day.

## Architecture We're Building
```
Entra ID Tenant
  └── Subscription: "Pay-As-You-Go" (or Free Trial)
       ├── rg-prod (Production resource group)
       │    ├── VNet: vnet-prod (10.0.0.0/16)
       │    │    ├── subnet-web (10.0.1.0/24) + NSG (allow HTTP/SSH)
       │    │    └── subnet-data (10.0.2.0/24) + NSG (allow only from web subnet)
       │    ├── VM: vm-web-01 (Ubuntu, Nginx web server)
       │    │    └── System-assigned Managed Identity
       │    ├── Key Vault: kv-prod-xxx (stores DB connection string)
       │    ├── Storage Account — ADLS Gen2: stprodxxx (data lake)
       │    ├── Storage Account — Blob: stblobxxx (regular blob storage)
       │    ├── Log Analytics Workspace: law-prod (central log store)
       │    ├── Recovery Services Vault: rsv-prod (backup)
       │    ├── Alert Rules + Action Groups (CPU alert → email)
       │    └── Azure Policy (enforce tagging)
       └── rg-dev (Dev resource group — for RBAC testing)

Entra ID:
  ├── User: devuser (test user with restricted access)
  ├── Group: "Developers" → Reader on rg-prod, Contributor on rg-dev
  └── Group: "Ops" → Contributor on Subscription
```

---

## Progress Tracker

- [ ] Phase 1: Foundation — Subscription, Entra ID, RBAC, Conditional Access
- [ ] Phase 2: Networking — VNet, Subnets, NSG, DNS concepts
- [ ] Phase 3: Compute — VM, App Service, VM Scale Sets concepts
- [ ] Phase 4: Identity & Security — Managed Identity, Key Vault
- [ ] Phase 5: Monitoring — Log Analytics, Alerts, Action Groups, KQL
- [ ] Phase 6: Backup & DR — Recovery Services Vault, Azure Site Recovery
- [ ] Phase 7: Governance — Tags, Policy, Cost Management
- [ ] Phase 8: Defender for Cloud — Security Posture
- [ ] Phase 9: Storage — Blob, File, Disk + ADLS Gen2
- [ ] Phase 10: Data Platform — Data Factory, Microsoft Fabric concepts
- [ ] Phase 11: Automation & DevOps — Runbooks, CI/CD, OS Patching
- [ ] Phase 12: Networking Advanced — Load Balancer, App Gateway, VPN/ExpressRoute concepts
- [ ] Phase 13: Cleanup & Cost Control

---

## PHASE 1: Foundation — Subscription, Entra ID, RBAC

### What we're doing
Setting up the base: a subscription (billing boundary), two resource groups (logical containers for resources), users, groups, and role-based access control. This is the equivalent of setting up AWS Organizations + IAM users/groups + permission sets.

---

### 1.1 Get a Subscription

**What**: A subscription is Azure's billing boundary (like an AWS Account). Every resource lives inside a subscription. You can have multiple subscriptions under one Entra ID tenant.

**Portal**:
1. Go to portal.azure.com
2. Search "Subscriptions" in the top search bar
3. If empty: Click "Add" → Start free trial ($200 credit for 30 days)
4. If you already have one: Note the subscription name and ID

**CLI**:
```bash
# Check if you have a subscription
az account list -o table
```

---

### 1.2 Install Azure CLI

**What**: Azure CLI (`az`) is the command-line tool for managing Azure resources. Equivalent of `aws` CLI.

```bash
# macOS
brew install azure-cli

# Login — opens browser for authentication
az login

# Verify — should show your subscription
az account show -o table
```

---

### 1.3 Create Resource Groups

**What**: Resource Groups (RGs) are logical containers that hold related Azure resources. Every resource MUST belong to exactly one RG. Think of them as labeled folders. AWS has no direct equivalent — the closest is tagging, but RGs are enforced containers.

**Why two RGs**: We create rg-prod and rg-dev to test RBAC — developers get Contributor on dev but only Reader on prod.

**Portal**:
1. Search "Resource groups" → Click "+ Create"
2. Subscription: Select yours
3. Resource group name: `rg-prod`
4. Region: `East US`
5. Tags tab → Add: `Environment` = `Production`, `Owner` = `Ops`, `CostCenter` = `IT`
6. Review + Create → Create
7. Repeat for `rg-dev` with tags `Environment` = `Development`

**CLI**:
```bash
# Production resource group with tags
az group create \
  --name rg-prod \
  --location eastus \
  --tags Environment=Production Owner=Ops CostCenter=IT

# Dev resource group with tags
az group create \
  --name rg-dev \
  --location eastus \
  --tags Environment=Development Owner=DevTeam CostCenter=Dev

# Verify
az group list -o table
```

---

### 1.4 Create a Test User in Entra ID

**What**: Creating a user identity in Entra ID (Azure's identity service). This is like creating an IAM user in AWS. We'll use this user to test RBAC restrictions.

**Portal**:
1. Search "Microsoft Entra ID" → Click "Users" in left menu
2. Click "+ New user" → "Create new user"
3. User principal name: `devuser` (it will append @yourdomain.onmicrosoft.com)
4. Display name: `Dev User`
5. Auto-generate password → Copy it (you'll need it to test login)
6. Click "Create"

**CLI**:
```bash
# First, find your tenant domain
DOMAIN=$(az rest --method get \
  --url https://graph.microsoft.com/v1.0/organization \
  --query "value[0].verifiedDomains[0].name" -o tsv)

echo "Your domain: $DOMAIN"

# Create the test user
az ad user create \
  --display-name "Dev User" \
  --user-principal-name devuser@$DOMAIN \
  --password "TempPass@1234" \
  --force-change-password-next-sign-in true

echo "Created user: devuser@$DOMAIN"
```

---

### 1.5 Create Groups and Add User

**What**: Groups let you manage permissions in bulk. Instead of assigning roles to 8 individual developers, assign to the "Developers" group once. Same concept as IAM Groups in AWS.

**Portal**:
1. Entra ID → Groups → "+ New group"
2. Group type: Security
3. Group name: `Developers`
4. Members: Click "No members selected" → Search "Dev User" → Select → Create
5. Repeat for `Ops` group (no members needed for now)

**CLI**:
```bash
# Create groups
az ad group create --display-name "Developers" --mail-nickname "developers"
az ad group create --display-name "Ops" --mail-nickname "ops"

# Add devuser to Developers group
DEV_USER_ID=$(az ad user show --id devuser@$DOMAIN --query id -o tsv)
DEV_GROUP_ID=$(az ad group show --group "Developers" --query id -o tsv)
az ad group member add --group $DEV_GROUP_ID --member-id $DEV_USER_ID

# Verify
az ad group member list --group "Developers" --query "[].displayName" -o tsv
```

---

### 1.6 Assign RBAC Roles

**What**: RBAC (Role-Based Access Control) is how Azure controls who can do what on which resources. Every role assignment has three parts:

| Part | Meaning | Example |
|------|---------|---------|
| WHO | Security principal (user, group, managed identity) | Developers group |
| WHAT | Role definition (set of allowed actions) | Reader, Contributor, Owner |
| WHERE | Scope (where the role applies) | Subscription, Resource Group, or Resource |

**Built-in roles**:
- **Reader**: View all resources, change nothing
- **Contributor**: Create/modify/delete resources, but CANNOT assign roles to others
- **Owner**: Everything + can assign roles (like AdministratorAccess in AWS)

Roles INHERIT downward: assign at subscription → applies to all RGs and resources below.

**What we're assigning**:
- Developers → **Reader** on rg-prod (can view production, can't change anything)
- Developers → **Contributor** on rg-dev (full control in dev)

**Portal**:
1. Go to Resource Group `rg-prod`
2. Click "Access control (IAM)" in left menu
3. Click "+ Add" → "Add role assignment"
4. Role tab: Search and select "Reader" → Next
5. Members tab: Click "+ Select members" → Search "Developers" group → Select → Next
6. Review + Assign
7. Repeat for `rg-dev` with "Contributor" role

**CLI**:
```bash
SUBSCRIPTION_ID=$(az account show --query id -o tsv)

# Developers: Reader on rg-prod (can only view, not modify)
az role assignment create \
  --assignee-object-id $DEV_GROUP_ID \
  --assignee-principal-type Group \
  --role "Reader" \
  --scope "/subscriptions/$SUBSCRIPTION_ID/resourceGroups/rg-prod"

# Developers: Contributor on rg-dev (can create/modify/delete)
az role assignment create \
  --assignee-object-id $DEV_GROUP_ID \
  --assignee-principal-type Group \
  --role "Contributor" \
  --scope "/subscriptions/$SUBSCRIPTION_ID/resourceGroups/rg-dev"

# Verify assignments
az role assignment list --resource-group rg-prod -o table
az role assignment list --resource-group rg-dev -o table
```

---

### 1.7 TEST IT — Verify RBAC Works

**What we're testing**: Login as the restricted devuser and confirm they can't create resources in prod but can in dev.

1. Open an **incognito/private browser window**
2. Go to portal.azure.com
3. Login as `devuser@YOUR_DOMAIN` with the password you set
4. Change password when prompted
5. Navigate to `rg-prod` → Try to create anything (e.g., a Storage Account) → **Should FAIL** with "You don't have permission"
6. Navigate to `rg-dev` → Try to create a Storage Account → **Should SUCCEED**

**Interview story**: "I set up RBAC with scoped role assignments — developers got Reader on production and Contributor on dev. I verified by logging in as a test user and confirming they couldn't create resources in production but could in dev."

---

### 1.8 Conditional Access (Portal Only)

**What**: Conditional Access policies are if-then rules for sign-ins. Think of them as a smart firewall for authentication. "IF admin signs in FROM outside office, THEN require MFA." This is a feature of Entra ID — no AWS direct equivalent (closest is AWS SSO + SCPs combined).

**Note**: Requires Entra ID P1 license. Free tier may not support this. If you can't access it, understand the concept — it's guaranteed to come up in the interview.

**Portal** (if available):
1. Entra ID → Security → Conditional Access → "+ New policy"
2. Name: `Require MFA for Developers`
3. Assignments:
   - Users: Include → Select "Developers" group
   - Target resources: All cloud apps
4. Conditions: (leave as default — applies to all locations/devices)
5. Grant: Select "Require multifactor authentication"
6. Enable policy: **Report-only** (safe testing mode — logs what WOULD happen without actually enforcing)
7. Create

**Common Conditional Access scenarios (interview favourites)**:

| Policy | Who | Condition | Action |
|--------|-----|-----------|--------|
| MFA for all admins | Admin group | Any location | Require MFA |
| Block foreign logins | All users | Outside India/office country | Block |
| Allow office without MFA | All users | From office IP range | Grant (skip MFA) |
| Block legacy protocols | All users | Legacy auth (POP, IMAP) | Block |
| Require compliant device | Finance team | Any | Require Intune-managed device |

**Troubleshooting sign-in failures** (interview scenario):
- Go to Entra ID → Sign-in logs → Filter by user
- Every login shows: status, failure reason, which Conditional Access policies applied, MFA result
- This is your FIRST stop when any user says "I can't log in"

---

## PHASE 2: Networking — VNet, Subnets, NSG

### What we're doing
Building the network foundation: a virtual network with two subnets (web-facing and data/backend), secured with Network Security Groups. This maps directly to AWS VPC + Subnets + Security Groups.

**Key difference from AWS**: Azure subnets are NOT labeled "public" or "private". A subnet becomes "private" based on its route table (no internet gateway route) and NSG rules (block internet inbound). Same concept, different implementation.

---

### 2.1 Create Virtual Network (VNet) with Subnets

**What**: A VNet is an isolated network in Azure — identical to AWS VPC. We're creating one VNet with two subnets:
- `subnet-web` (10.0.1.0/24): For web-facing VMs. Will have NSG allowing HTTP (80) and SSH (22).
- `subnet-data` (10.0.2.0/24): For databases/storage. Will have NSG allowing traffic ONLY from subnet-web. No internet access.

**Portal**:
1. Search "Virtual networks" → "+ Create"
2. Resource group: `rg-prod`
3. Name: `vnet-prod`
4. Region: `East US`
5. IP Addresses tab:
   - Address space: `10.0.0.0/16` (65,536 IPs)
   - Click "+ Add a subnet":
     - Name: `subnet-web`, Range: `10.0.1.0/24` (256 IPs)
   - Click "+ Add a subnet":
     - Name: `subnet-data`, Range: `10.0.2.0/24` (256 IPs)
6. Review + Create → Create

**CLI**:
```bash
# Create VNet with address space 10.0.0.0/16
az network vnet create \
  --resource-group rg-prod \
  --name vnet-prod \
  --address-prefix 10.0.0.0/16 \
  --location eastus

# Create subnet for web servers (will have public-facing NSG)
az network vnet subnet create \
  --resource-group rg-prod \
  --vnet-name vnet-prod \
  --name subnet-web \
  --address-prefix 10.0.1.0/24

# Create subnet for data/backend (will be locked down)
az network vnet subnet create \
  --resource-group rg-prod \
  --vnet-name vnet-prod \
  --name subnet-data \
  --address-prefix 10.0.2.0/24

# Verify
az network vnet subnet list --resource-group rg-prod --vnet-name vnet-prod -o table
```

---

### 2.2 Create Network Security Groups (NSGs)

**What**: NSGs are stateful firewalls attached to subnets or NICs — equivalent to AWS Security Groups. Each NSG contains rules with:
- **Priority**: 100-4096. Lower number = evaluated first. First matching rule wins.
- **Direction**: Inbound or Outbound
- **Source/Destination**: IP, CIDR, service tag (e.g., "Internet"), or ASG
- **Port**: Single, range, or wildcard (*)
- **Protocol**: TCP, UDP, ICMP, or any (*)
- **Action**: Allow or Deny

**Key difference from AWS Security Groups**: NSGs have explicit **priority-based ordering** and support **Deny rules**. AWS SGs only allow — you can't explicitly deny. This means in Azure, rule ordering matters.

**What we're creating**:

**nsg-web** (for subnet-web):
| Priority | Name | Direction | Source | Port | Protocol | Action | Why |
|----------|------|-----------|--------|------|----------|--------|-----|
| 100 | AllowHTTP | Inbound | Any | 80 | TCP | Allow | Web traffic to Nginx |
| 110 | AllowSSH | Inbound | Your IP only | 22 | TCP | Allow | SSH admin access (restricted to your IP for security) |
| 120 | AllowICMP | Inbound | Any | * | ICMP | Allow | Allow ping for troubleshooting |

**nsg-data** (for subnet-data):
| Priority | Name | Direction | Source | Port | Protocol | Action | Why |
|----------|------|-----------|--------|------|----------|--------|-----|
| 100 | AllowFromWebSubnet | Inbound | 10.0.1.0/24 | * | Any | Allow | Only web subnet can reach data subnet |
| 200 | DenyAllInternet | Inbound | Internet | * | Any | Deny | Block all internet traffic to data subnet |

**Portal**:
1. Search "Network security groups" → "+ Create"
2. Resource group: `rg-prod`, Name: `nsg-web`, Region: `East US` → Create
3. Open `nsg-web` → "Inbound security rules" → "+ Add"
   - Source: Any, Destination: Any, Service: HTTP, Priority: 100, Name: AllowHTTP → Add
4. Add another rule:
   - Source: IP Addresses, Source IP: `YOUR_PUBLIC_IP/32`, Service: SSH, Priority: 110, Name: AllowSSH → Add
5. Add ICMP rule:
   - Source: Any, Protocol: ICMP, Priority: 120, Name: AllowICMP → Add
6. Repeat for `nsg-data` with the two rules above
7. **Attach to subnets**: Go to each NSG → "Subnets" → "+ Associate" → Select vnet-prod and the matching subnet

**CLI**:
```bash
# ---- NSG for web subnet ----
az network nsg create --resource-group rg-prod --name nsg-web

# Rule 1: Allow HTTP (port 80) from anywhere — so users can reach Nginx
az network nsg rule create \
  --resource-group rg-prod \
  --nsg-name nsg-web \
  --name AllowHTTP \
  --priority 100 \
  --direction Inbound \
  --access Allow \
  --protocol Tcp \
  --destination-port-ranges 80 \
  --source-address-prefixes "*"

# Rule 2: Allow SSH (port 22) from YOUR IP only — for admin access
# $(curl -s ifconfig.me) fetches your current public IP
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

# Rule 3: Allow ICMP (ping) from anywhere — useful for network troubleshooting
az network nsg rule create \
  --resource-group rg-prod \
  --nsg-name nsg-web \
  --name AllowICMP \
  --priority 120 \
  --direction Inbound \
  --access Allow \
  --protocol Icmp \
  --destination-port-ranges "*" \
  --source-address-prefixes "*"

# ---- NSG for data subnet ----
az network nsg create --resource-group rg-prod --name nsg-data

# Rule 1: Allow ALL traffic from web subnet only (10.0.1.0/24)
# This means only VMs in subnet-web can talk to subnet-data
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

# Rule 2: Deny ALL inbound from internet — data subnet is fully private
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

# ---- Attach NSGs to subnets ----
# This links the firewall rules to the actual subnet
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

# Verify NSG rules
az network nsg rule list --resource-group rg-prod --nsg-name nsg-web -o table
az network nsg rule list --resource-group rg-prod --nsg-name nsg-data -o table
```

---

### 2.3 DNS Concepts (Theory — Interview Knowledge)

**What**: Azure DNS lets you host DNS zones and manage records. Equivalent of AWS Route 53.

**Key concepts for interview**:
- **Public DNS Zone**: Host your domain (e.g., example.com) → create A, CNAME, TXT records
- **Private DNS Zone**: Name resolution within VNets (e.g., db-prod-01.internal.example.com resolves to private IP)
- **Record types**: A (IP), CNAME (alias), MX (mail), TXT (verification), NS (nameservers)
- **TTL**: Time-to-live — how long DNS resolvers cache the record

**Interview scenario**: "Users report intermittent connectivity after a migration."
Answer: "I'd check DNS — the old A record might still be cached with the old IP. I'd lower the TTL before migration, update the record, and verify resolution with `nslookup` or `dig`."

---

## PHASE 3: Compute — VM Deployment + App Service + Scale Sets

### What we're doing
Deploying an Ubuntu VM into subnet-web, installing Nginx web server, and testing network connectivity. Also covering App Service and VM Scale Sets concepts.

---

### 3.1 Create a Linux VM

**What**: Deploying a virtual machine in the web subnet. We use `Standard_B1s` (1 vCPU, 1GB RAM) — cheapest VM size, costs ~$0.01/hour. We skip creating a new NSG (`--nsg ""`) because the subnet already has nsg-web attached.

**Portal**:
1. Search "Virtual machines" → "+ Create" → "Azure virtual machine"
2. Resource group: `rg-prod`
3. VM name: `vm-web-01`
4. Region: `East US`
5. Image: `Ubuntu Server 22.04 LTS`
6. Size: Click "See all sizes" → Search `B1s` → Select
7. Authentication: SSH public key → Username: `azureuser` → Generate new key pair
8. Networking tab:
   - Virtual network: `vnet-prod`
   - Subnet: `subnet-web`
   - Public IP: Create new → Name: `pip-web-01`
   - NIC NSG: None (subnet NSG handles it)
9. Tags tab: `Environment=Production`, `Role=WebServer`, `Owner=Ops`
10. Review + Create → Create → Download the SSH private key when prompted

**CLI**:
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

# Save the public IP for later use
VM_IP=$(az vm show -d --resource-group rg-prod --name vm-web-01 --query publicIps -o tsv)
echo "VM Public IP: $VM_IP"
```

---

### 3.2 Install Nginx Web Server

**What**: SSH into the VM and install Nginx to make it a web server. This gives us something to test HTTP connectivity against.

```bash
# SSH into the VM
ssh azureuser@$VM_IP

# --- Inside the VM ---
# Update package list and install Nginx
sudo apt update && sudo apt install -y nginx

# Start Nginx and enable it to start on boot
sudo systemctl start nginx
sudo systemctl enable nginx

# Verify it's running
sudo systemctl status nginx

# Exit back to your local machine
exit
```

---

### 3.3 TEST IT — Verify Connectivity

**What**: Testing that our NSG rules work correctly — HTTP should work, ping should work, SSH works (from our IP).

```bash
# Test 1: HTTP — should return Nginx welcome page HTML
curl http://$VM_IP
# Expected: "Welcome to nginx!" HTML page

# Test 2: Ping — should succeed (we allowed ICMP in NSG)
ping -c 3 $VM_IP

# Test 3: SSH — already working since you logged in above
```

---

### 3.4 SCENARIO — NSG Troubleshooting Exercise

**What**: Simulating a real production issue. We deliberately add a deny rule that blocks HTTP, then troubleshoot and fix it. This gives you a real debugging story for the interview.

**The problem**: A higher-priority Deny rule overrides the Allow rule.

```bash
# Step 1: Add a Deny rule for HTTP with priority 90 (lower = evaluated first)
# This OVERRIDES the AllowHTTP rule at priority 100
az network nsg rule create \
  --resource-group rg-prod \
  --nsg-name nsg-web \
  --name DenyHTTP \
  --priority 90 \
  --direction Inbound \
  --access Deny \
  --protocol Tcp \
  --destination-port-ranges 80

# Step 2: Test — HTTP should now TIMEOUT (traffic is being blocked)
curl --connect-timeout 5 http://$VM_IP
# Expected: Connection timed out

# Step 3: Troubleshoot — List all NSG rules sorted by priority
az network nsg rule list --resource-group rg-prod --nsg-name nsg-web \
  --query "sort_by([].{Priority:priority, Name:name, Access:access, Port:destinationPortRange, Direction:direction}, &Priority)" \
  -o table
# You'll see DenyHTTP at priority 90 BEFORE AllowHTTP at priority 100

# Step 4: Fix — Remove the offending deny rule
az network nsg rule delete \
  --resource-group rg-prod \
  --nsg-name nsg-web \
  --name DenyHTTP

# Step 5: Verify fix — HTTP should work again
curl http://$VM_IP
# Expected: Nginx welcome page is back
```

**Portal troubleshooting path**:
1. VM → Networking → "Effective security rules" (shows ALL rules from all NSGs affecting this VM)
2. Or: NSG → "Inbound security rules" → Sort by priority → Spot the conflict

**Interview story**: "I debugged a production issue where HTTP traffic was blocked. I checked the NSG effective rules, found a deny rule at priority 90 overriding the allow rule at priority 100. Lower priority number means it's evaluated first in Azure NSGs. I removed the conflicting rule and restored access. I also used this as a learning moment to document NSG rule ordering in our runbook."

---

### 3.5 Azure App Service (Concept — Know for Interview)

**What**: App Service is Azure's PaaS for hosting web apps. You deploy your code, Azure manages the VM, OS, patching, scaling. Equivalent of AWS Elastic Beanstalk.

**Key points**:
- You deploy code (Python, Node, .NET, Java) — not VMs
- **App Service Plan**: The underlying compute (like choosing instance size). Defines CPU/RAM/price tier.
- **Deployment slots**: Deploy to a staging slot, test, then swap to production with zero downtime
- Supports custom domains, SSL/TLS, authentication
- Auto-scales based on rules (CPU, memory, request count)

**When to use VM vs App Service**:
| Use VM when | Use App Service when |
|-------------|---------------------|
| Need full OS control | Just deploying a web app |
| Custom software/drivers | Standard web frameworks |
| Legacy applications | Modern web apps / APIs |
| Specific networking needs | Simple deployment model |

**Interview answer**: "For standard web applications, I'd recommend App Service over VMs — less operational overhead, built-in scaling, and Microsoft handles OS patching. VMs are for when you need full control over the OS."

---

### 3.6 VM Scale Sets (VMSS) — Concept

**What**: A group of identical VMs that auto-scales based on demand. Equivalent of AWS Auto Scaling Groups (ASG).

**Key points**:
- All VMs are created from the same image/configuration
- **Scaling rules**: Scale out when CPU > 80%, scale in when CPU < 30%
- **Availability Zones**: Distribute VMs across physically separate datacenters (like AWS AZs)
- **Availability Sets**: Distribute across fault domains within one datacenter (older approach)
- Integrates with Azure Load Balancer or Application Gateway

**Interview answer**: "For high-availability web workloads, I'd use VM Scale Sets behind a Load Balancer. Configure auto-scale rules based on CPU metrics, and distribute across Availability Zones for datacenter-level resilience."

---

## PHASE 4: Identity & Security — Managed Identity, Key Vault

### What we're doing
Enabling Managed Identity on the VM so it can securely access Key Vault (secrets store) and Storage without any hardcoded credentials. This is the Azure equivalent of attaching an IAM Role to an EC2 instance.

---

### 4.1 Enable Managed Identity on VM

**What**: A Managed Identity is an automatically managed identity in Entra ID for your Azure resource. The VM gets an identity that can authenticate to other Azure services — no passwords, no keys, no secrets to manage.

Two types:
- **System-assigned**: Created with the VM, deleted when VM is deleted. One per resource. (Like EC2 instance profile)
- **User-assigned**: Created independently, can be shared across multiple VMs. (Like a reusable IAM role)

We're using system-assigned here (simpler for a single VM).

**Portal**:
1. Go to VM `vm-web-01` → "Identity" in left menu
2. System assigned tab → Toggle Status to "On" → Save

**CLI**:
```bash
# Enable system-assigned managed identity
az vm identity assign --resource-group rg-prod --name vm-web-01

# Verify — note the principalId (this is the identity's ID in Entra ID)
az vm show --resource-group rg-prod --name vm-web-01 \
  --query identity.principalId -o tsv
```

---

### 4.2 Create Key Vault

**What**: Key Vault is Azure's secrets management service (like AWS Secrets Manager + KMS combined). It stores:
- **Secrets**: Connection strings, API keys, passwords
- **Keys**: Encryption keys
- **Certificates**: SSL/TLS certificates

We enable RBAC authorization so access is controlled through RBAC roles (not the older vault access policies).

**Portal**:
1. Search "Key vaults" → "+ Create"
2. Resource group: `rg-prod`
3. Key vault name: `kv-prod-UNIQUE` (must be globally unique — add random chars)
4. Region: `East US`
5. Access configuration tab: Permission model → Select "Azure role-based access control"
6. Review + Create → Create

**CLI**:
```bash
# Name must be globally unique — add random suffix
KV_NAME="kv-prod-$(openssl rand -hex 3)"

az keyvault create \
  --resource-group rg-prod \
  --name $KV_NAME \
  --location eastus \
  --enable-rbac-authorization true

echo "Key Vault created: $KV_NAME"
```

---

### 4.3 Grant Yourself Access + Store a Secret

**What**: Even as the subscription owner, with RBAC-enabled Key Vault you need an explicit role assignment to manage secrets. We'll assign "Key Vault Administrator" to ourselves, then store a fake database connection string.

```bash
# Grant yourself Key Vault Administrator role
MY_USER_ID=$(az ad signed-in-user show --query id -o tsv)
KV_ID=$(az keyvault show --name $KV_NAME --query id -o tsv)

az role assignment create \
  --assignee-object-id $MY_USER_ID \
  --assignee-principal-type User \
  --role "Key Vault Administrator" \
  --scope $KV_ID

# Wait a minute for the role to propagate, then store a secret
az keyvault secret set \
  --vault-name $KV_NAME \
  --name "db-connection-string" \
  --value "Server=db-prod-01;Database=appdb;User=admin;Password=secret123"

# Verify
az keyvault secret show --vault-name $KV_NAME --name "db-connection-string" --query value -o tsv
```

**Portal**:
1. Key Vault → "Secrets" → "+ Generate/Import"
2. Name: `db-connection-string`
3. Value: `Server=db-prod-01;Database=appdb;User=admin;Password=secret123`
4. Create

---

### 4.4 Grant VM's Managed Identity Access to Key Vault

**What**: We give the VM's identity the "Key Vault Secrets User" role (read-only access to secrets). The VM can now read secrets without storing any credentials.

**Portal**:
1. Key Vault → "Access control (IAM)" → "+ Add" → "Add role assignment"
2. Role: "Key Vault Secrets User" → Next
3. Members: "Managed Identity" → "+ Select members" → Select `vm-web-01` → Next
4. Review + Assign

**CLI**:
```bash
VM_IDENTITY=$(az vm show --resource-group rg-prod --name vm-web-01 \
  --query identity.principalId -o tsv)

az role assignment create \
  --assignee-object-id $VM_IDENTITY \
  --assignee-principal-type ServicePrincipal \
  --role "Key Vault Secrets User" \
  --scope $KV_ID
```

---

### 4.5 TEST IT — Access Secret from VM Without Any Credentials

**What**: SSH into the VM, login with Managed Identity (no username/password!), and read the secret from Key Vault. This proves zero-credential access works.

```bash
ssh azureuser@$VM_IP

# --- Inside the VM ---

# Install Azure CLI
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash

# Login using Managed Identity — no username, no password, no key!
az login --identity
# Should say "Logged in with Managed Identity"

# Read the secret from Key Vault
az keyvault secret show --vault-name YOUR_KV_NAME --name db-connection-string --query value -o tsv
# Should output: Server=db-prod-01;Database=appdb;User=admin;Password=secret123

exit
```

**Interview story**: "I configured Managed Identity on a VM to access Key Vault secrets without storing any credentials. A junior engineer had hardcoded credentials in a config file — I replaced that with Managed Identity, assigned the Key Vault Secrets User role, and the app authenticated automatically using Azure-managed tokens. Same pattern as IAM Roles on ECS tasks, which I've implemented in AWS."

---

## PHASE 5: Monitoring — Log Analytics, Alerts, Action Groups, KQL

### What we're doing
Setting up centralized logging and monitoring. We create a Log Analytics workspace (central log store), install the monitoring agent on the VM, create alert rules for CPU, and configure notifications. Equivalent of your DataDog + Prometheus + Grafana stack.

---

### 5.1 Create Log Analytics Workspace

**What**: A Log Analytics workspace is the central destination for all logs and metrics in Azure. Everything sends logs here — VMs, App Services, NSGs, Key Vault, etc. You query it using KQL (Kusto Query Language). Equivalent of DataDog Log Explorer or CloudWatch Logs.

**Portal**:
1. Search "Log Analytics workspaces" → "+ Create"
2. Resource group: `rg-prod`
3. Name: `law-prod`
4. Region: `East US`
5. Review + Create → Create

**CLI**:
```bash
az monitor log-analytics workspace create \
  --resource-group rg-prod \
  --workspace-name law-prod \
  --location eastus

# Get workspace ID (needed later)
WORKSPACE_ID=$(az monitor log-analytics workspace show \
  --resource-group rg-prod \
  --workspace-name law-prod \
  --query id -o tsv)
```

---

### 5.2 Enable VM Monitoring with Azure Monitor Agent

**What**: Installing the Azure Monitor Agent (AMA) on the VM so it sends logs and performance metrics to the Log Analytics workspace. Equivalent of installing the DataDog agent on a container/VM.

**Portal**:
1. Go to VM `vm-web-01` → "Extensions + applications" → "+ Add"
2. Select "Azure Monitor Linux Agent" → Install
3. Or: Go to Azure Monitor → "Virtual Machines" → Select vm-web-01 → Enable monitoring

**CLI**:
```bash
# Install Azure Monitor Agent extension on the VM
az vm extension set \
  --resource-group rg-prod \
  --vm-name vm-web-01 \
  --name AzureMonitorLinuxAgent \
  --publisher Microsoft.Azure.Monitor

# Note: You may also need a Data Collection Rule (DCR) to specify which logs/metrics to collect
# Portal: Azure Monitor → Data Collection Rules → Create → Associate with VM
```

---

### 5.3 Create Action Group

**What**: An Action Group defines WHO gets notified and HOW when an alert fires. It can send emails, SMS, push notifications, call webhooks (Slack/PagerDuty), or trigger automation runbooks. Equivalent of DataDog notification channels or AWS SNS topics.

**Portal**:
1. Search "Monitor" → "Alerts" → "Action groups" → "+ Create"
2. Resource group: `rg-prod`
3. Action group name: `ag-ops-team`, Display name: `OpsTeam`
4. Notifications tab:
   - Type: Email/SMS/Push/Voice
   - Name: `ops-email`
   - Email: `your-email@gmail.com`
5. Actions tab (optional):
   - Type: Webhook → URL: Your Slack webhook URL
6. Review + Create → Create

**CLI**:
```bash
az monitor action-group create \
  --resource-group rg-prod \
  --name ag-ops-team \
  --short-name OpsTeam \
  --action email ops-email "YOUR_EMAIL@gmail.com"

# To add a Slack webhook too:
# az monitor action-group update \
#   --resource-group rg-prod \
#   --name ag-ops-team \
#   --add-action webhook slack-notify "https://hooks.slack.com/services/YOUR/WEBHOOK/URL"
```

---

### 5.4 Create Alert Rule — CPU > 85%

**What**: An alert rule monitors a metric and fires when a condition is met. We're creating: "If average CPU on vm-web-01 exceeds 85% for 5 minutes, fire an alert and notify the ops team." Equivalent of a DataDog Monitor or CloudWatch Alarm.

Alert components:
- **Scope**: Which resource to monitor (our VM)
- **Condition**: What triggers it (CPU > 85% average over 5 minutes)
- **Action Group**: Who to notify (ag-ops-team → email)
- **Severity**: 0 (Critical) to 4 (Verbose). We use 2 (Warning).

**Portal**:
1. Go to VM `vm-web-01` → "Alerts" → "+ Create alert rule"
2. Condition: Signal name → "Percentage CPU"
   - Threshold: Static > 85
   - Aggregation: Average
   - Period: 5 minutes
   - Evaluation frequency: 1 minute
3. Actions: Select action group → `ag-ops-team`
4. Details: Name: `alert-cpu-high`, Severity: Warning (Sev 2)
5. Review + Create → Create

**CLI**:
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

---

### 5.5 TEST IT — Trigger the CPU Alert

**What**: We deliberately spike the CPU to trigger the alert and verify the entire monitoring pipeline works end-to-end (metric → alert → action group → email).

```bash
ssh azureuser@$VM_IP

# Install stress tool
sudo apt install -y stress

# Spike CPU for 5 minutes (will push both cores to 100%)
stress --cpu 2 --timeout 300

exit
```

**After 5-10 minutes**:
1. Check your email — you should receive an alert notification
2. Portal: Azure Monitor → Alerts → You should see a fired alert
3. Click the alert to see details: which metric, what value, when it fired

**Portal to review**:
- Monitor → Alerts → See fired alerts with severity, state, and time
- Click alert → "View query" to see the underlying metric query

**Interview story**: "I set up Azure Monitor with metric alert rules and Action Groups. I stress-tested a VM to spike CPU above 85%, verified the alert fired within the 5-minute evaluation window, and confirmed email notifications reached the ops team. I've also set up similar alerting in DataDog with custom monitors and Slack integrations."

---

### 5.6 Explore KQL in Log Analytics

**What**: KQL (Kusto Query Language) is how you query logs in Log Analytics. It's the Azure equivalent of DataDog's log search syntax or CloudWatch Insights. You don't need to be an expert — just know the basics.

**Portal**: Log Analytics workspace → law-prod → "Logs"

Try these queries (paste directly into the query editor):

```kql
// Query 1: Check if the monitoring agent is sending heartbeats
Heartbeat
| where TimeGenerated > ago(1h)
| summarize count() by Computer
// Shows how many heartbeat signals each VM sent in the last hour

// Query 2: CPU performance over last hour in 5-minute buckets
Perf
| where ObjectName == "Processor" and CounterName == "% Processor Time"
| where TimeGenerated > ago(1h)
| summarize avg(CounterValue) by Computer, bin(TimeGenerated, 5m)
| render timechart
// This should show the CPU spike from your stress test

// Query 3: Find errors in Azure Activity Log (resource operations)
AzureActivity
| where TimeGenerated > ago(24h)
| where Level == "Error"
| project TimeGenerated, OperationName, Caller, ResourceGroup
| order by TimeGenerated desc

// Query 4: Count of events by severity
Event
| summarize count() by EventLevelName
| order by count_ desc
```

**KQL basics to remember**:
- `|` pipes to next operation (like bash pipes)
- `where` = filter rows
- `summarize` = aggregate (GROUP BY equivalent)
- `project` = select specific columns
- `ago(1h)` = relative time
- `bin(TimeGenerated, 5m)` = time buckets
- `render timechart` = visualize as graph

---

## PHASE 6: Backup & Disaster Recovery

### What we're doing
Setting up Azure Backup for the VM using a Recovery Services Vault, triggering a backup, and verifying it. Also covering Azure Site Recovery (DR) concepts for the interview.

---

### 6.1 Create Recovery Services Vault

**What**: A Recovery Services Vault is the central management container for all backups and DR configurations. Think of it as the "backup server" — it stores backup policies, recovery points, and replication configurations. No direct AWS equivalent (AWS Backup is the closest).

**Portal**:
1. Search "Recovery Services vaults" → "+ Create"
2. Resource group: `rg-prod`
3. Vault name: `rsv-prod`
4. Region: `East US` (MUST be same region as the VMs you're backing up)
5. Review + Create → Create

**CLI**:
```bash
az backup vault create \
  --resource-group rg-prod \
  --name rsv-prod \
  --location eastus
```

---

### 6.2 Configure Backup for VM

**What**: Enabling backup for vm-web-01. The DefaultPolicy runs a daily backup and retains for 30 days. A backup policy defines:
- **Schedule**: When backups run (daily at a specific time)
- **Retention**: How long to keep them (daily, weekly, monthly, yearly)
- **RPO** (Recovery Point Objective): Maximum acceptable data loss. Daily backup = 24h RPO.

**Portal**:
1. Recovery Services vault `rsv-prod` → "Backup"
2. Where is your workload running? → "Azure"
3. What do you want to back up? → "Virtual Machine" → Continue
4. Select `vm-web-01` → Enable backup
5. Policy: DefaultPolicy (daily backup, 30 day retention)

**CLI**:
```bash
az backup protection enable-for-vm \
  --resource-group rg-prod \
  --vault-name rsv-prod \
  --vm vm-web-01 \
  --policy-name DefaultPolicy
```

---

### 6.3 Trigger On-Demand Backup and Verify

**What**: Instead of waiting for the scheduled backup, we trigger one now to verify the pipeline works.

```bash
# Find the backup container and item names (Azure uses specific naming format)
az backup container list \
  --resource-group rg-prod \
  --vault-name rsv-prod \
  --backup-management-type AzureIaasVM \
  -o table

az backup item list \
  --resource-group rg-prod \
  --vault-name rsv-prod \
  -o table

# Trigger backup now (use the container/item names from above)
# Format varies — use the values from the list commands
az backup protection backup-now \
  --resource-group rg-prod \
  --vault-name rsv-prod \
  --container-name "IaasVMContainer;iaasvmcontainerv2;rg-prod;vm-web-01" \
  --item-name "VM;iaasvmcontainerv2;rg-prod;vm-web-01" \
  --retain-until 2026-06-14

# Check backup job status
az backup job list --resource-group rg-prod --vault-name rsv-prod -o table
```

**Portal**: Recovery Services vault → "Backup Jobs" → See the running/completed job

---

### 6.4 Azure Site Recovery — DR Concepts (Interview Knowledge)

**What**: Azure Site Recovery (ASR) is for Disaster Recovery — it continuously replicates VMs to a secondary Azure region. If your primary region goes down, you failover to the DR region. Different from backup:

| | Azure Backup | Azure Site Recovery |
|---|---|---|
| Purpose | Data protection (recover files/VMs) | Business continuity (keep running during outage) |
| RTO | Hours (restore from backup) | Minutes (failover to pre-replicated copy) |
| RPO | Hours-daily (depends on backup schedule) | Seconds-minutes (continuous replication) |
| Scope | Individual VMs/files | Entire application stack |

**Key DR concepts for interview**:
- **Failover**: Switch production traffic to DR region when primary is down
- **Failback**: Return to primary region after it recovers
- **Test Failover (DR Drill)**: Test the DR process without affecting production — critical for compliance
- **Reprotect**: After failback, re-enable replication to DR region

**Interview scenario**: "Primary region goes down. Walk me through the DR process."
Answer: "ASR continuously replicates VMs to the secondary region. I'd initiate failover from the Recovery Services vault — this boots the replicated VMs in the DR region. I'd update DNS or Traffic Manager to point traffic to the DR region. Once the primary recovers, I'd reprotect and failback, then verify replication is active again. We do test failovers quarterly to validate the process."

---

## PHASE 7: Governance — Tags, Policy, Cost Management

### What we're doing
Implementing resource governance: enforcing tagging with Azure Policy, reviewing costs. This shows the interviewer you understand operational discipline, not just technical setup.

---

### 7.1 Verify Tags on All Resources

**What**: Tags are key-value labels on resources for cost tracking, ownership, and automation. Critical for MSP environments with multiple clients. Equivalent of AWS resource tags.

```bash
# List all resources in rg-prod with their tags
az resource list --resource-group rg-prod \
  --query "[].{Name:name, Type:type, Tags:tags}" -o table
```

**Portal**: Resource group → Click any resource → "Tags" in left menu

---

### 7.2 Create Azure Policy — Enforce Tagging

**What**: Azure Policy lets you enforce rules across your environment. We're creating a policy that flags any resource group without an "Environment" tag. Equivalent of AWS Config rules.

**Portal**:
1. Search "Policy" → "Assignments" → "+ Assign policy"
2. Scope: Your subscription
3. Policy definition: Search "Require a tag on resource groups"
4. Parameters: Tag name: `Environment`
5. Create

**CLI**:
```bash
az policy assignment create \
  --name "require-env-tag" \
  --display-name "Require Environment tag on Resource Groups" \
  --policy "/providers/Microsoft.Authorization/policyDefinitions/96670d01-0a4d-4649-9c89-2d3abc0a5025" \
  --params '{"tagName": {"value": "Environment"}}' \
  --scope "/subscriptions/$(az account show --query id -o tsv)"
```

---

### 7.3 TEST IT — Create RG Without Tag

```bash
# Create resource group without the required tag
az group create --name rg-test-no-tag --location eastus

# Note: Azure Policy evaluates every ~15 minutes
# Check compliance in portal: Policy → Compliance
# rg-test-no-tag should show as non-compliant
```

**Portal**: Policy → Compliance → Click the policy → See which resources are non-compliant

---

### 7.4 Cost Management

**What**: Azure Cost Management helps you track, analyze, and optimize cloud spending. Equivalent of AWS Cost Explorer.

**Portal**:
1. Search "Cost Management" → "Cost analysis"
   - Group by: Resource group → See cost per RG
   - Group by: Tag (Environment) → See cost for Production vs Dev
   - Filter: Date range, service type, region
2. "Budgets" → "+ Add"
   - Create a $50 budget for rg-prod
   - Alert at 80% ($40) → Send to your email

**Interview story**: "I set up Azure Policy to enforce tagging, created budgets with alerts in Cost Management, and used tag-based cost analysis to track spending per environment. In AWS, I did similar cost optimization with DataDog ingestion rules to control monitoring costs."

---

## PHASE 8: Defender for Cloud

### What we're doing
Reviewing Azure's built-in security posture management. Defender for Cloud scans your environment and gives you a security score with actionable recommendations.

---

### 8.1 Review Security Posture

**What**: Microsoft Defender for Cloud is Azure's CSPM (Cloud Security Posture Management) tool. It scans your environment and gives you a "Secure Score" (0-100%) with prioritized recommendations. Equivalent of AWS Security Hub + GuardDuty + Inspector combined.

**Portal**:
1. Search "Microsoft Defender for Cloud"
2. **Overview**: Check your Secure Score
3. **Recommendations**: Sorted by severity (High, Medium, Low)
4. Common findings you'll see from our setup:
   - "Restrict SSH access" (we allowed SSH from our IP — might still flag)
   - "Enable encryption at rest" for storage
   - "Enable MFA for accounts with owner permissions"
   - "Virtual machines should have backups configured" (should be green — we set this up)

### 8.2 Remediate a Finding

1. Click any recommendation
2. Read the description — understand WHY it's a risk
3. Click "View affected resources"
4. Follow the remediation steps
5. Example: If it flags SSH open → Tighten NSG to only your IP (which we already did)

**Interview story**: "I regularly review Defender for Cloud recommendations. I found our Secure Score was at 65% — top findings included open management ports and missing encryption. I remediated the critical ones by tightening NSG rules, enabling disk encryption, and configuring backup policies. Brought the score to 85% within a week."

---

## PHASE 9: Storage — Blob, File, Disk + ADLS Gen2

### What we're doing
Understanding Azure's three storage types, then creating an ADLS Gen2 data lake with proper directory structure and Managed Identity access.

---

### 9.1 Azure Storage Types (Know for Interview)

| Type | What it is | AWS Equivalent | Use Case |
|------|-----------|----------------|----------|
| **Blob Storage** | Object storage | S3 | Images, videos, backups, logs, any unstructured data |
| **File Storage** | Managed file shares (SMB/NFS) | EFS | Shared file system across VMs, lift-and-shift |
| **Disk Storage** | Block storage for VMs | EBS | OS disks, data disks attached to VMs |
| **ADLS Gen2** | Blob Storage + hierarchical namespace | S3 for analytics | Data lake, big data, analytics workloads |

**Blob Storage tiers**:
- **Hot**: Frequently accessed. Higher storage cost, lower access cost.
- **Cool**: Infrequently accessed (30+ days). Lower storage cost, higher access cost.
- **Archive**: Rarely accessed (180+ days). Cheapest storage, hours to retrieve.

**Redundancy options**:
- **LRS** (Locally Redundant): 3 copies in one datacenter. Cheapest.
- **ZRS** (Zone Redundant): 3 copies across availability zones. AZ-level protection.
- **GRS** (Geo Redundant): 6 copies — 3 local + 3 in paired region. Region-level protection.

---

### 9.2 Create Regular Blob Storage Account

**What**: A standard storage account for application files, logs, and general blob storage.

**Portal**:
1. Search "Storage accounts" → "+ Create"
2. Resource group: `rg-prod`
3. Name: `stblob` + random chars (globally unique)
4. Region: `East US`
5. Performance: Standard
6. Redundancy: LRS (cheapest for lab)
7. Review + Create

**CLI**:
```bash
BLOB_STORAGE="stblob$(openssl rand -hex 3)"

az storage account create \
  --resource-group rg-prod \
  --name $BLOB_STORAGE \
  --location eastus \
  --sku Standard_LRS \
  --kind StorageV2 \
  --tags Environment=Production Type=BlobStorage

echo "Blob Storage: $BLOB_STORAGE"
```

---

### 9.3 Create a Blob Container and Upload a File

**What**: A container in Blob Storage is like an S3 bucket — a top-level grouping for blobs (files).

**Portal**:
1. Storage account → "Containers" → "+ Container"
2. Name: `app-logs`, Access level: Private → Create
3. Open `app-logs` → "Upload" → Select a file

**CLI**:
```bash
# Create a container
az storage container create \
  --name app-logs \
  --account-name $BLOB_STORAGE \
  --auth-mode login

# Create a sample log file and upload
echo "[2026-06-04 10:30:00] INFO: Application started
[2026-06-04 10:30:05] INFO: Connected to database
[2026-06-04 10:31:00] ERROR: Payment service timeout
[2026-06-04 10:31:30] WARN: Retrying payment service" > /tmp/app.log

az storage blob upload \
  --account-name $BLOB_STORAGE \
  --container-name app-logs \
  --file /tmp/app.log \
  --name "2026/06/04/app.log" \
  --auth-mode login

# List blobs
az storage blob list --account-name $BLOB_STORAGE --container-name app-logs -o table --auth-mode login
```

---

### 9.4 Create ADLS Gen2 Storage Account (Data Lake)

**What**: ADLS Gen2 = Blob Storage + **Hierarchical Namespace (HNS)**. HNS gives you real directories (not just prefix-based like S3). This enables:
- Atomic directory rename/move operations (critical for big data pipelines)
- Fine-grained ACLs at folder/file level
- Better performance for analytics workloads

The `--hns true` flag is the ONLY difference from regular Blob Storage.

**Portal**:
1. Storage accounts → "+ Create"
2. Same as above, BUT on the "Advanced" tab:
   - **Enable hierarchical namespace**: CHECK THIS BOX (this makes it ADLS Gen2)
3. Name: `stdatalake` + random chars → Create

**CLI**:
```bash
DATALAKE_STORAGE="stdl$(openssl rand -hex 3)"

az storage account create \
  --resource-group rg-prod \
  --name $DATALAKE_STORAGE \
  --location eastus \
  --sku Standard_LRS \
  --kind StorageV2 \
  --hns true \
  --tags Environment=Production Type=DataLake

echo "ADLS Gen2 Storage: $DATALAKE_STORAGE"
```

---

### 9.5 Create Data Lake Structure and Upload Data

**What**: Setting up a proper data lake directory structure — raw zone for ingested data, processed zone for transformed data. This is the pattern used in every data lake architecture.

```bash
# Create a filesystem (top-level container in ADLS Gen2)
az storage fs create \
  --name datalake \
  --account-name $DATALAKE_STORAGE \
  --auth-mode login

# Create directory structure following bronze/silver/gold pattern
# raw/ = landing zone for raw ingested data (bronze)
# processed/ = cleaned/transformed data (silver)
# curated/ = business-ready data (gold)
az storage fs directory create --name raw --file-system datalake --account-name $DATALAKE_STORAGE --auth-mode login
az storage fs directory create --name raw/customers/2026/06/04 --file-system datalake --account-name $DATALAKE_STORAGE --auth-mode login
az storage fs directory create --name raw/orders/2026/06/04 --file-system datalake --account-name $DATALAKE_STORAGE --auth-mode login
az storage fs directory create --name processed --file-system datalake --account-name $DATALAKE_STORAGE --auth-mode login
az storage fs directory create --name curated --file-system datalake --account-name $DATALAKE_STORAGE --auth-mode login

# Create sample data files
cat > /tmp/customers.csv << 'EOF'
customer_id,name,email,city,signup_date
1,Amit Sharma,amit@example.com,Mumbai,2026-01-15
2,Priya Verma,priya@example.com,Delhi,2026-02-20
3,Rahul Joshi,rahul@example.com,Jaipur,2026-03-10
4,Sneha Patel,sneha@example.com,Bangalore,2026-04-05
5,Vikram Singh,vikram@example.com,Pune,2026-05-12
EOF

cat > /tmp/orders.csv << 'EOF'
order_id,customer_id,product,amount,order_date,status
101,1,Azure VM License,15000,2026-06-01,completed
102,2,Storage 1TB,2300,2026-06-01,completed
103,3,SQL Database,8000,2026-06-02,completed
104,1,App Service Plan,5000,2026-06-02,pending
105,4,Key Vault,800,2026-06-03,completed
106,5,Load Balancer,3500,2026-06-03,failed
107,2,Backup Vault,1200,2026-06-04,completed
EOF

# Upload files to date-partitioned directories
az storage fs file upload \
  --source /tmp/customers.csv \
  --path raw/customers/2026/06/04/customers.csv \
  --file-system datalake \
  --account-name $DATALAKE_STORAGE \
  --auth-mode login

az storage fs file upload \
  --source /tmp/orders.csv \
  --path raw/orders/2026/06/04/orders.csv \
  --file-system datalake \
  --account-name $DATALAKE_STORAGE \
  --auth-mode login

# List the directory structure
az storage fs file list --file-system datalake --account-name $DATALAKE_STORAGE --auth-mode login -o table
```

---

### 9.6 Grant VM Managed Identity Access to Data Lake

**What**: Same Managed Identity pattern — let the VM read from the data lake without credentials.

```bash
DATALAKE_ID=$(az storage account show --name $DATALAKE_STORAGE --query id -o tsv)
VM_IDENTITY=$(az vm show --resource-group rg-prod --name vm-web-01 --query identity.principalId -o tsv)

az role assignment create \
  --assignee-object-id $VM_IDENTITY \
  --assignee-principal-type ServicePrincipal \
  --role "Storage Blob Data Reader" \
  --scope $DATALAKE_ID
```

**TEST from VM**:
```bash
ssh azureuser@$VM_IP

az login --identity
az storage fs file list --file-system datalake --account-name DATALAKE_NAME --auth-mode login -o table
az storage fs file download --path raw/customers/2026/06/04/customers.csv --file-system datalake --account-name DATALAKE_NAME --auth-mode login -f /tmp/downloaded.csv
cat /tmp/downloaded.csv

exit
```

**Interview story**: "I set up an ADLS Gen2 data lake with hierarchical namespace, organized data in bronze/silver/gold zones with date partitioning, and configured Managed Identity for secure access. I've also built ETL pipelines with Azure Data Factory to ingest from sources into the raw zone."

---

## PHASE 10: Data Platform — Data Factory, Microsoft Fabric Concepts

### What we're doing
Covering the data platform concepts from the JD. Azure Data Factory is something you've used before — this is a refresher. Microsoft Fabric is newer — you need concept-level knowledge.

---

### 10.1 Azure Data Factory (ADF) — Refresher

**What you already know**: You've used ADF to create ETL pipelines to fetch data and upload to Blob Storage. Here's the structured recap.

**Key components**:
| Component | What it does | Analogy |
|-----------|-------------|---------|
| **Pipeline** | A workflow of activities (steps) | CI/CD pipeline |
| **Activity** | A single step (copy data, run SQL, transform) | A job step |
| **Dataset** | Pointer to data (which table, which file, which blob) | Data source config |
| **Linked Service** | Connection to an external system (SQL Server, Blob, API) | Database connection string |
| **Trigger** | When the pipeline runs (schedule, event, manual) | Cron job |
| **Integration Runtime** | Compute that executes the pipeline (Azure, self-hosted, SSIS) | Execution environment |

**Common pipeline patterns**:
1. **Copy Activity**: Move data from source to destination (e.g., on-prem SQL → ADLS Gen2)
2. **Data Flow**: Visual drag-and-drop transformations (filter, join, aggregate)
3. **Lookup + ForEach**: Dynamic pipelines (look up a list, loop through each)

**Your interview angle**: "I've used Azure Data Factory to build ETL pipelines that extract data from external sources and load into Blob Storage. I configured Linked Services for source/destination, used Copy Activities for data movement, and scheduled triggers for daily ingestion."

---

### 10.2 Microsoft Fabric — Concepts (New, Interview-Level)

**What**: Microsoft Fabric is an all-in-one analytics platform launched in 2023. It combines Data Factory + Synapse + Power BI + Data Lake into ONE unified experience.

**Key components**:
| Component | What it does | Equivalent |
|-----------|-------------|-----------|
| **OneLake** | Unified data lake for the entire organization | One S3 bucket for everything |
| **Lakehouse** | Combines data lake files + SQL query capability | Delta Lake + SQL Engine |
| **Warehouse** | Traditional SQL data warehouse | Azure Synapse / AWS Redshift |
| **Data Pipeline** | ETL/ELT — same as Azure Data Factory | ADF but inside Fabric |
| **Dataflow Gen2** | Low-code data transformations | Power Query at scale |
| **Notebooks** | Spark notebooks for data engineering | Databricks notebooks |

**The Lakehouse concept** (interview favourite):
- Store raw files (CSV, Parquet, JSON) in the data lake
- Query them with SQL using an auto-generated SQL endpoint
- No need to load into a separate database — query files directly
- Uses **Delta format** under the hood (versioned, ACID-compliant files)

**Data lake zones in Fabric**:
```
OneLake
  └── Lakehouse
       ├── Files/ (raw data — CSVs, Parquets)
       │    ├── raw/ (bronze — as ingested)
       │    ├── processed/ (silver — cleaned)
       │    └── curated/ (gold — business-ready)
       └── Tables/ (managed Delta tables — queryable with SQL)
```

**Your interview angle**: "I understand that Fabric unifies the analytics stack — I've worked with Azure Data Factory for ETL which is now part of Fabric's pipeline capability, and I've set up ADLS Gen2 data lakes which map to Fabric's OneLake. The Lakehouse concept of querying files directly with SQL eliminates the need to load data into a separate warehouse for exploration."

---

### 10.3 SQL in Data Platform Context

**What the JD means**: As an ops engineer supporting data workloads, you'll need to:
- **Validate data after pipeline runs**: Row counts, NULL checks, date ranges
- **Troubleshoot data issues**: Find missing records, duplicates, mismatches
- **Write reports**: Aggregations, joins across tables

Refer to `sql-practice-recap.md` for query patterns. Key ones for data ops:

```sql
-- Validate pipeline: source vs target row count
SELECT 'source' AS table_name, COUNT(*) AS row_count FROM source_table
UNION ALL
SELECT 'target', COUNT(*) FROM target_table;

-- Find rows that didn't make it (data loss detection)
SELECT s.id FROM source_table s
LEFT JOIN target_table t ON s.id = t.id
WHERE t.id IS NULL;

-- Check for NULLs in critical columns after ingestion
SELECT COUNT(*) AS null_count
FROM staging_customers
WHERE customer_id IS NULL OR email IS NULL;

-- Verify date range of loaded data
SELECT MIN(order_date) AS earliest, MAX(order_date) AS latest, COUNT(*) AS total
FROM staging_orders;
```

---

## PHASE 11: Automation & DevOps — Runbooks, CI/CD, OS Patching

### What we're doing
Covering the automation parts of the JD: Azure Automation runbooks, CI/CD pipelines, and OS patching.

---

### 11.1 Azure Automation Runbooks (Concept + Simple Practical)

**What**: Azure Automation lets you run PowerShell or Python scripts (called runbooks) on a schedule or triggered by events. Equivalent of AWS Lambda + SSM Automation combined. Used for:
- Auto-remediation: VM runs out of disk → runbook extends the disk
- Scheduled tasks: Clean up old snapshots, rotate keys
- Operational tasks: Start/stop VMs on schedule (cost saving)

**Common interview scenario**: "How would you automatically stop dev VMs at 7 PM to save cost?"
Answer: "I'd create an Azure Automation runbook with a PowerShell script that stops all VMs in rg-dev. Schedule it to run daily at 7 PM using an Automation schedule. The runbook uses a system-assigned Managed Identity with Contributor role on rg-dev."

**Portal**:
1. Search "Automation Accounts" → "+ Create"
2. Resource group: `rg-prod`, Name: `aa-ops`, Region: `East US`
3. Create → Go to the Automation Account
4. "Runbooks" → "+ Create a runbook"
5. Name: `Stop-Dev-VMs`, Type: PowerShell, Runtime: 7.2 → Create
6. Paste this script:
```powershell
# Stop all VMs in rg-dev
Connect-AzAccount -Identity
$vms = Get-AzVM -ResourceGroupName "rg-dev"
foreach ($vm in $vms) {
    Stop-AzVM -ResourceGroupName "rg-dev" -Name $vm.Name -Force
    Write-Output "Stopped VM: $($vm.Name)"
}
```
7. Save → Publish
8. "Schedules" → Link a schedule (daily at 7 PM)

---

### 11.2 CI/CD Pipelines (Concept — Interview Knowledge)

**What the JD expects**: "Manage and support CI/CD pipelines at intermediate level." You already have GitHub Actions and GitLab CI experience — this maps directly.

**Azure DevOps CI/CD**:
| Component | What it does | Your Equivalent |
|-----------|-------------|-----------------|
| Azure Repos | Git repository hosting | GitHub / GitLab |
| Azure Pipelines | CI/CD pipeline execution | GitHub Actions / GitLab CI |
| Azure Artifacts | Package registry (npm, pip, Docker) | GitHub Packages |
| Azure Boards | Work item tracking | Jira |

**Azure Pipelines YAML** (similar to GitHub Actions):
```yaml
# azure-pipelines.yml (equivalent of .github/workflows/deploy.yml)
trigger:
  branches:
    include:
      - main

pool:
  vmImage: 'ubuntu-latest'

stages:
  - stage: Build
    jobs:
      - job: BuildApp
        steps:
          - script: npm install && npm run build
          - task: Docker@2
            inputs:
              command: buildAndPush
              repository: myapp
              containerRegistry: myACR

  - stage: Deploy
    dependsOn: Build
    jobs:
      - job: DeployToProd
        steps:
          - task: AzureWebApp@1
            inputs:
              appName: my-web-app
              package: $(Build.ArtifactStagingDirectory)
```

**Interview angle**: "I've built CI/CD pipelines in GitHub Actions and GitLab CI with Docker builds and deployments to AWS ECS. Azure Pipelines uses similar YAML-based definitions — the concepts are identical. I also use Terraform in my pipelines for infrastructure changes, which is cloud-agnostic."

---

### 11.3 OS Patching (Operational Knowledge)

**What**: Regular OS patching is a key operational responsibility in an MSP. The JD specifically mentions it.

**Azure Update Management** (now part of Azure Update Manager):
- Scans VMs for missing OS patches (Windows and Linux)
- Schedule maintenance windows for patch deployment
- Pre/post scripts for graceful handling (drain from load balancer → patch → rejoin)
- Compliance reporting — which VMs are patched, which are behind

**Portal**:
1. Search "Update Manager" (or go to VM → "Updates")
2. See pending updates for your VM
3. Schedule a one-time or recurring update deployment
4. Set maintenance window (e.g., Sunday 2 AM - 4 AM)
5. Review update history

**Linux patching (manual on our VM)**:
```bash
ssh azureuser@$VM_IP

# Check available updates
sudo apt update
sudo apt list --upgradable

# Apply updates
sudo apt upgrade -y

# Check if reboot is needed
cat /var/run/reboot-required 2>/dev/null && echo "REBOOT NEEDED" || echo "No reboot needed"

exit
```

**Interview answer**: "For OS patching, I use Azure Update Manager to schedule maintenance windows — typically during off-peak hours on weekends. I ensure VMs are gracefully drained from load balancers before patching. For Linux, standard apt/yum updates; for Windows, Windows Update through Azure. I track compliance through the Update Manager dashboard and escalate VMs that haven't been patched within the SLA."

---

## PHASE 12: Networking Advanced — Load Balancer, App Gateway, VPN/ExpressRoute

### Concepts for Interview (not practical — too expensive for lab)

---

### 12.1 Azure Load Balancer vs Application Gateway

| Feature | Azure Load Balancer | Application Gateway |
|---------|-------------------|-------------------|
| Layer | Layer 4 (TCP/UDP) | Layer 7 (HTTP/HTTPS) |
| AWS Equivalent | NLB | ALB |
| Routing | IP + Port based | URL path, hostname, headers |
| SSL Termination | No | Yes |
| WAF | No | Yes (optional add-on) |
| Use case | Raw TCP traffic, non-HTTP | Web applications, APIs |
| Health probes | TCP/HTTP | HTTP/HTTPS with custom paths |

**When to use which**:
- **Web application** → Application Gateway (can route /api to one backend, /web to another)
- **Database traffic, TCP services** → Azure Load Balancer
- **Need WAF** → Application Gateway with WAF policy

**Interview answer**: "For web workloads, I'd use Application Gateway with WAF for Layer 7 routing and security. For TCP services like databases, Azure Load Balancer at Layer 4. In AWS, this maps to ALB vs NLB — same decision criteria."

---

### 12.2 VPN Gateway & ExpressRoute

**VPN Gateway** (AWS equivalent: Site-to-Site VPN):
- Encrypted tunnel over public internet between on-prem and Azure VNet
- Cheaper, slower, variable latency
- Good for: Dev/test, small offices, backup connectivity

**ExpressRoute** (AWS equivalent: Direct Connect):
- Private dedicated connection between on-prem and Azure
- Does NOT go over public internet
- Higher bandwidth, consistent latency, more expensive
- Good for: Production workloads, large data transfers, compliance requirements

**Interview scenario**: "Users in the office can't reach Azure VMs through VPN."
Answer: "I'd check: 1) VPN Gateway status in Azure portal — is it connected? 2) On-prem VPN device — is the tunnel up? 3) Route tables — are routes being advertised? 4) NSG rules — is traffic from on-prem IP range allowed? 5) Check VPN diagnostic logs in Log Analytics."

---

### 12.3 Azure Bastion (Important for Security)

**What**: Azure Bastion provides secure RDP/SSH access to VMs through the Azure portal without exposing a public IP. You don't need to open SSH/RDP ports in NSG.

**How it works**:
1. Deploy Bastion in your VNet (dedicated subnet)
2. Connect to VM through portal — Bastion proxies the connection
3. No public IP needed on VM, no NSG rule for SSH needed

**Interview angle**: "Instead of exposing SSH to the internet, I'd recommend Azure Bastion for secure access. It eliminates the need for public IPs on VMs and removes SSH port exposure from NSGs. For emergency access, Bastion through the portal is the most secure option."

---

## PHASE 13: Cleanup

### IMPORTANT — Do this when you're done to avoid charges

**Portal**:
1. Search "Resource groups"
2. Delete `rg-prod` (this deletes ALL resources inside it — VMs, VNet, Key Vault, everything)
3. Delete `rg-dev`
4. Delete `rg-test-no-tag`
5. Entra ID → Users → Delete `devuser`
6. Entra ID → Groups → Delete `Developers` and `Ops`
7. Policy → Assignments → Delete `require-env-tag`

**CLI**:
```bash
# Delete resource groups (this deletes ALL contained resources)
az group delete --name rg-prod --yes --no-wait
az group delete --name rg-dev --yes --no-wait
az group delete --name rg-test-no-tag --yes --no-wait

# Delete test user and groups
az ad user delete --id devuser@$DOMAIN
az ad group delete --group "Developers"
az ad group delete --group "Ops"

# Remove policy assignment
az policy assignment delete --name "require-env-tag"

# Verify everything is gone (after a few minutes)
az group list -o table
```

---

## Interview Stories Summary

| JD Topic | Your Hands-On Story |
|----------|-------------------|
| **Entra ID / RBAC** | "Created user groups with scoped RBAC — Contributor on dev, Reader on prod — verified by logging in as restricted user and confirming access denied in production" |
| **Conditional Access** | "Configured CA policy to require MFA for developers group in report-only mode first, then enforced. Troubleshot blocked logins using sign-in logs" |
| **Networking / NSG** | "Built VNet with web and data subnets, configured NSGs with priority-based rules. Debugged a case where a higher-priority deny rule blocked HTTP — found it by listing NSG rules and checking priority ordering" |
| **Compute** | "Deployed Ubuntu VM, installed Nginx, tested connectivity. For production web apps, I'd recommend App Service or VM Scale Sets behind Application Gateway" |
| **Managed Identity** | "Configured system-assigned Managed Identity on VM to access Key Vault and ADLS Gen2 — zero credentials stored anywhere. Same pattern as IAM Roles on ECS tasks in AWS" |
| **Key Vault** | "Stored database connection strings in Key Vault with RBAC authorization. VMs access secrets through Managed Identity — no hardcoded credentials" |
| **Monitoring** | "Set up Log Analytics workspace, Azure Monitor Agent, metric alerts with Action Groups. Stress-tested VM to trigger CPU alert and verified email notification end-to-end" |
| **KQL** | "Query Log Analytics for performance data, heartbeat checks, and error investigation using KQL — similar to DataDog log queries" |
| **Backup** | "Configured Recovery Services Vault with backup policies, triggered on-demand backups, validated job completion" |
| **DR** | "Understand ASR for cross-region replication — failover, failback, test failover drills for compliance" |
| **Governance** | "Enforced tagging with Azure Policy, set up cost budgets with alerts, tracked spending by resource group and environment tag" |
| **Security** | "Reviewed Defender for Cloud Secure Score, remediated findings like open SSH ports, tracked compliance improvement" |
| **Blob Storage** | "Created storage accounts for app logs and data lake, configured lifecycle policies, managed access with RBAC" |
| **ADLS Gen2** | "Set up ADLS Gen2 with hierarchical namespace, bronze/silver/gold directory structure, date-partitioned raw data, Managed Identity access" |
| **Data Factory** | "Built ETL pipelines in ADF with Copy Activities, Linked Services, and scheduled triggers to ingest data into Blob Storage and ADLS Gen2" |
| **Microsoft Fabric** | "Understand Fabric as unified analytics — OneLake, Lakehouse (query files with SQL), data pipelines. My ADF and ADLS Gen2 experience maps directly" |
| **Automation** | "Created runbooks for automated VM start/stop to save costs, scheduled with Azure Automation" |
| **CI/CD** | "Built CI/CD in GitHub Actions and GitLab CI. Azure Pipelines uses same YAML-based approach — concepts transfer directly" |
| **OS Patching** | "Use Azure Update Manager for scheduled maintenance windows, track compliance, ensure graceful drain before patching" |
| **Load Balancing** | "Application Gateway for Layer 7 web traffic with WAF, Azure Load Balancer for Layer 4 TCP. Maps to ALB/NLB in AWS" |
| **VPN/ExpressRoute** | "VPN Gateway for encrypted tunnel over internet (like AWS Site-to-Site VPN), ExpressRoute for dedicated private connection (like Direct Connect)" |
| **Bastion** | "Recommend Azure Bastion for secure VM access — eliminates public IP exposure and open SSH ports" |
| **ITIL** | "Follow incident → problem → change management. Standard changes are pre-approved, normal changes go through CAB, emergency changes get retrospective approval" |
| **ITSM** | "Experience with PagerDuty/alert integration. Ticket triage, categorization, SLA tracking, and runbook documentation for L1 teams" |
