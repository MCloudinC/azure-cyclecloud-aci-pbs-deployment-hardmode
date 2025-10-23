# 🚀 Complete Setup Guide for CycleCloud ACI PBS Deployment

This guide walks you through the complete setup process to get the GitHub Actions workflows running successfully.

## ⚠️ Important Updates Applied

This repository has been updated to fix several critical issues:
- ✅ Fixed environment name mismatches between workflows
- ✅ Added better error messages and setup guidance
- ✅ Added retry logic for Azure operations
- ✅ Improved validation and troubleshooting steps

## 📋 Prerequisites Checklist

Before running the workflows, ensure you have:

### 1. Azure Infrastructure
- [ ] **Azure subscription** with appropriate permissions
- [ ] **Resource group** for deployments
- [ ] **Virtual Network (VNet)** with at least one subnet
- [ ] **Network access method** (VPN Gateway, Azure Bastion, or Jump Box)
- [ ] **Service principal** with required permissions

### 2. GitHub Repository Setup
- [ ] Repository secrets configured
- [ ] Repository variables configured  
- [ ] Workflows enabled in repository settings

---

## 🔧 Step-by-Step Setup

### Step 1: Create Azure Infrastructure

#### 1.1 Create Resource Group
```bash
az group create \
  --name "RG-BH_HPC_Cloud_Azure-NP-SUB-000005-EastUS-dev" \
  --location "eastus"
```

#### 1.2 Create Virtual Network
```bash
# Create VNet for HPC cluster
az network vnet create \
  --resource-group "RG-BH_HPC_Cloud_Azure-NP-SUB-000005-EastUS-dev" \
  --name "hpc-vnet" \
  --address-prefix 10.0.0.0/16 \
  --subnet-name "cyclecloud-subnet" \
  --subnet-prefix 10.0.1.0/24

# Create additional subnet for compute nodes
az network vnet subnet create \
  --resource-group "RG-BH_HPC_Cloud_Azure-NP-SUB-000005-EastUS-dev" \
  --vnet-name "hpc-vnet" \
  --name "compute-subnet" \
  --address-prefix 10.0.2.0/24
```

#### 1.3 Configure Subnet for Container Instances
```bash
# Enable container instance delegation on CycleCloud subnet
az network vnet subnet update \
  --resource-group "RG-BH_HPC_Cloud_Azure-NP-SUB-000005-EastUS-dev" \
  --vnet-name "hpc-vnet" \
  --name "cyclecloud-subnet" \
  --delegations Microsoft.ContainerInstance/containerGroups
```

### Step 2: Create Service Principal

#### 2.1 Create Service Principal with Permissions
```bash
# Create service principal for GitHub Actions
SP_OUTPUT=$(az ad sp create-for-rbac \
  --name "github-cyclecloud-deployment" \
  --role "Contributor" \
  --scopes "/subscriptions/$(az account show --query id -o tsv)/resourceGroups/RG-BH_HPC_Cloud_Azure-NP-SUB-000005-EastUS-dev" \
  --sdk-auth)

echo "$SP_OUTPUT"
```

#### 2.2 Add Network Permissions (if VNet is in different RG)
```bash
# If your VNet is in a different resource group, add network permissions
az role assignment create \
  --assignee $(echo "$SP_OUTPUT" | jq -r '.clientId') \
  --role "Network Contributor" \
  --scope "/subscriptions/$(az account show --query id -o tsv)/resourceGroups/YOUR_VNET_RESOURCE_GROUP"
```

### Step 3: Configure GitHub Repository

#### 3.1 Add Repository Secrets
Go to: **Settings** → **Secrets and Variables** → **Actions** → **Secrets**

Add these secrets:

| Secret Name | Value | Description |
|-------------|--------|-------------|
| `AZURE_CREDENTIALS` | Output from service principal creation | Full JSON from `az ad sp create-for-rbac --sdk-auth` |
| `CYCLECLOUD_ADMIN_USERNAME` | `admin` | CycleCloud admin username |
| `CYCLECLOUD_ADMIN_PASSWORD` | Strong password | CycleCloud admin password (12+ chars) |

**Example AZURE_CREDENTIALS format:**
```json
{
  "clientId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "clientSecret": "your-client-secret",
  "subscriptionId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "tenantId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

#### 3.2 Add Repository Variables
Go to: **Settings** → **Secrets and Variables** → **Actions** → **Variables**

Add these variables:

| Variable Name | Example Value | Description |
|---------------|---------------|-------------|
| `RESOURCE_GROUP` | `RG-BH_HPC_Cloud_Azure-NP-SUB-000005-EastUS-dev` | Target resource group for deployments |
| `AZURE_REGION` | `eastus` | Azure region for deployments |
| `VIRTUAL_NETWORK_NAME` | `hpc-vnet` | VNet name for private networking |
| `VIRTUAL_NETWORK_RESOURCE_GROUP_NAME` | `RG-BH_HPC_Cloud_Azure-NP-SUB-000005-EastUS-dev` | VNet resource group name |
| `VIRTUAL_NETWORK_SUBNET_NAME` | `cyclecloud-subnet` | Subnet name for CycleCloud container |

### Step 4: Setup Network Access

Since all resources use **private IPs only**, you need one of these access methods:

#### Option A: VPN Gateway (Recommended for Production)
```bash
# Create VPN gateway subnet
az network vnet subnet create \
  --resource-group "RG-BH_HPC_Cloud_Azure-NP-SUB-000005-EastUS-dev" \
  --vnet-name "hpc-vnet" \
  --name "GatewaySubnet" \
  --address-prefix 10.0.255.0/27

# Create VPN gateway (takes 20-45 minutes)
az network vnet-gateway create \
  --resource-group "RG-BH_HPC_Cloud_Azure-NP-SUB-000005-EastUS-dev" \
  --name "hpc-vpn-gateway" \
  --vnet "hpc-vnet" \
  --public-ip-addresses "vpn-gateway-ip" \
  --gateway-type Vpn \
  --sku VpnGw1 \
  --vpn-type RouteBased
```

#### Option B: Azure Bastion (Good for Occasional Access)  
```bash
# Create Bastion subnet
az network vnet subnet create \
  --resource-group "RG-BH_HPC_Cloud_Azure-NP-SUB-000005-EastUS-dev" \
  --vnet-name "hpc-vnet" \
  --name "AzureBastionSubnet" \
  --address-prefix 10.0.3.0/24

# Create Bastion host
az network bastion create \
  --resource-group "RG-BH_HPC_Cloud_Azure-NP-SUB-000005-EastUS-dev" \
  --name "hpc-bastion" \
  --vnet-name "hpc-vnet" \
  --public-ip-address "bastion-ip"
```

#### Option C: Jump Box (Simple for Testing)
```bash
# Create Jump Box VM with public IP
az vm create \
  --resource-group "RG-BH_HPC_Cloud_Azure-NP-SUB-000005-EastUS-dev" \
  --name "jump-box" \
  --image "Ubuntu2204" \
  --size "Standard_B2s" \
  --vnet-name "hpc-vnet" \
  --subnet "cyclecloud-subnet" \
  --public-ip-address "jump-box-ip" \
  --admin-username "azureuser" \
  --generate-ssh-keys
```

---

## 🚀 Running the Workflows

### Step 1: Deploy CycleCloud Container

1. Go to **Actions** → **Workflow 1 - Deploy CycleCloud**
2. Click **Run workflow**
3. Configure parameters:
   - **Environment**: Select your resource group environment
   - **Image tag**: `latest` or specific version
   - **Container name**: `cyclecloud-mcr` (default)
   - **CPU/Memory**: Adjust as needed  
   - **Deployment mode**: `forced` for first run

**Expected time**: ~15 minutes

### Step 2: Get Subnet Resource ID

You'll need the full subnet resource ID for Workflow 2:

```bash
az network vnet subnet show \
  --resource-group "RG-BH_HPC_Cloud_Azure-NP-SUB-000005-EastUS-dev" \
  --vnet-name "hpc-vnet" \
  --name "compute-subnet" \
  --query id -o tsv
```

Copy this full resource ID.

### Step 3: Create PBS Pro Cluster

1. Go to **Actions** → **Workflow 2 - Create PBS Pro Cluster**  
2. Click **Run workflow**
3. Configure parameters:
   - **Environment**: Same as Workflow 1
   - **Container name**: `cyclecloud-mcr` (match Workflow 1)
   - **Cluster name**: `pbspro-cluster` (or your preferred name)
   - **Subnet ID**: **PASTE THE FULL RESOURCE ID** from Step 2
   - **VM sizes**: Adjust as needed
   - **Max nodes**: Set limits for your quota
   - **Auto start master**: `true` (recommended)

**Expected time**: ~25 minutes

---

## ✅ Verification Steps

### After Workflow 1 Completes:
```bash
# Check container status
az container show \
  --resource-group "RG-BH_HPC_Cloud_Azure-NP-SUB-000005-EastUS-dev" \
  --name "cyclecloud-mcr" \
  --query "{State: instanceView.state, IP: ipAddress.ip}"

# Get private IP for access
PRIVATE_IP=$(az container show \
  --resource-group "RG-BH_HPC_Cloud_Azure-NP-SUB-000005-EastUS-dev" \
  --name "cyclecloud-mcr" \
  --query "ipAddress.ip" -o tsv)

echo "CycleCloud Web UI: http://$PRIVATE_IP:8080"
```

### After Workflow 2 Completes:
```bash
# Check cluster status via container CLI
az container exec \
  --resource-group "RG-BH_HPC_Cloud_Azure-NP-SUB-000005-EastUS-dev" \
  --name "cyclecloud-mcr" \
  --exec-command "/bin/bash -c 'cyclecloud show_cluster pbspro-cluster'"
```

### Access CycleCloud Web UI:
1. Connect to VPN or use Bastion/Jump Box
2. Browse to `http://<PRIVATE_IP>:8080`
3. Login with your admin credentials
4. Verify cluster appears in UI

---

## 🔧 Troubleshooting

### Common Issues:

#### "Repository variable RESOURCE_GROUP is required"
**Fix**: Go to repository **Settings** → **Secrets and Variables** → **Actions** → **Variables** and add missing variables.

#### "Virtual network 'X' not found"  
**Fix**: Create VNet first using Azure CLI commands above, or verify names match exactly.

#### "Azure Container Instance deployment failed"
**Possible causes**:
- Service principal lacks permissions
- Subnet not delegated for containers  
- Network security group blocking traffic
- Azure quota limits reached

**Fix steps**:
```bash
# Check service principal permissions
az role assignment list --assignee <CLIENT_ID> --output table

# Check subnet delegation
az network vnet subnet show --query delegations \
  --resource-group "RG-BH_HPC_Cloud_Azure-NP-SUB-000005-EastUS-dev" \
  --vnet-name "hpc-vnet" \
  --name "cyclecloud-subnet"

# Check quota usage  
az vm list-usage --location eastus --output table
```

#### "Cannot access CycleCloud web UI"
**Fix**: Ensure you're connected to the VNet via VPN, Bastion, or Jump Box. The container only has a private IP.

#### "Workflow 2 fails with 'Container not running'"
**Fix**: Ensure Workflow 1 completed successfully and container is in "Running" state before starting Workflow 2.

### Get Support:
- **Workflows**: Open issue in this repository
- **Azure CycleCloud**: https://github.com/Azure/cyclecloud-pbspro  
- **PBS Professional**: https://github.com/pbspro/pbspro

---

## 🎯 What You Get

After successful deployment:
- ✅ **CycleCloud container** running on private IP
- ✅ **PBS Pro master node** with configured queues  
- ✅ **Auto-scaling integration** (azpbs) ready to scale 0-100s of nodes
- ✅ **Private networking** for maximum security
- ✅ **Production-ready HPC environment** 

**Total time**: ~45 minutes from start to production cluster! 🚀

The cluster will automatically scale compute nodes up when you submit jobs, and scale them down when idle. Perfect for cost-effective HPC in the cloud.