# 46 - Standard Agent Setup with BYO VNet and GSA Proxy (Preview)

> **⚠️ PREVIEW FEATURE**
>
> The GSA (Global Secure Access) AI Connector proxy used in this setup is currently in **preview**. Features, APIs, and behavior may change before general availability.
>
> **To enroll in the GSA AI Connector preview**, fill out the onboarding form: [Securing Foundry Agents in Private Network Deployment with GSA](https://forms.cloud.microsoft/r/Qvn0cSWb42)

> **IMPORTANT — Subnet Address Range Availability**
>
> Class A subnet support (e.g., `10.x.x.x`) is available only in select regions: **Australia East, Brazil South, Canada East, East US, East US 2, France Central, Germany West Central, Italy North, Japan East, South Africa North, South Central US, South India, Spain Central, Sweden Central, UAE North, UK South, West Europe, West US, West US 3.**
>
> Class B (`172.16.x.x` – `172.31.x.x`) and Class C (`192.168.x.x`) subnet support is GA and available in all regions supported by Azure AI Foundry Agent Service. This template defaults to Class B addresses (`172.16.0.0/16`).
>
> For more on supported regions, see [Models supported by Azure AI Foundry Agent Service](https://learn.microsoft.com/en-us/azure/ai-foundry/agents/concepts/model-region-support?tabs=global-standard).

---

## What is the GSA AI Connector?

The **Global Secure Access (GSA) AI Connector** is an Azure Marketplace virtual appliance that acts as a transparent forward proxy for AI Foundry agent egress traffic. It enables organizations to:

- **Inspect and control outbound traffic** from AI agents to external services
- **Observe and audit egress requests** all AI agent network communications

The GSA proxy is deployed as a VM in your VNet, and a UDR (User Defined Route) on the agent subnet routes all default (`0.0.0.0/0`) traffic through it. Critical Azure service traffic is exempted via service tag routes.

---

This sample deploys an Azure AI Foundry agent setup with:

- **BYO Virtual Network** with agent subnet delegation (no private endpoints)
- **GSA AI Connector proxy** VM for egress traffic control
- **UDR (User Defined Route)** that routes `0.0.0.0/0` traffic from the agent subnet through the GSA proxy
- **Service tag exceptions** in the UDR so critical Azure traffic bypasses the proxy

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│  Virtual Network (172.16.0.0/16)                                │
│                                                                 │
│  ┌───────────────────────────┐  ┌─────────────────────────────┐ │
│  │ agent-subnet              │  │ gsa-proxy-subnet            │ │
│  │ (172.16.0.0/24)           │  │ (172.16.1.0/24)             │ │
│  │  Delegated to             │  │                             │ │
│  │  Microsoft.App/environments│  │  ┌──────────────────────┐  │ │
│  │                           │  │  │ GSA Proxy VM          │  │ │
│  │  UDR: 0/0 → Proxy IP     │  │  │ - Managed Identity    │  │ │
│  │  (with service tag        │──│──│ - IP Forwarding: ON   │  │ │
│  │   exceptions)             │  │  │ - NSG: VNet allowed   │  │ │
│  │                           │  │  └──────────────────────┘  │ │
│  └───────────────────────────┘  └─────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
         │
         ├── AI Services (publicNetworkAccess: Enabled)
         │     └── networkInjections: agent subnet
         ├── AI Project
         ├── Cosmos DB
         ├── AI Search
         └── Storage Account
```

## Key Differences from Other Setups

| Feature | 41-standard-agent | 15-private-network | **46-byovnet-gsa-proxy** |
|---|---|---|---|
| VNet | ❌ | ✅ | ✅ |
| Private Endpoints | ❌ | ✅ | ❌ |
| DNS Zones | ❌ | ✅ | ❌ |
| Public Network Access | Enabled | Disabled | **Enabled** |
| Agent Subnet Delegation | ❌ | ✅ | ✅ |
| GSA Proxy | ❌ | ❌ | ✅ |
| UDR (egress control) | ❌ | ❌ | ✅ |

## GSA Proxy Details

The GSA AI Connector is deployed from the Azure Marketplace:
- **Publisher**: `microsoftcorporation1687208452115`
- **Offer**: `gsaaiconnector1-preview`
- **Plan**: `gsaaiconnectorplan1`

### VM Configuration
- **Managed Identity**: System-assigned (enabled)
- **IP Forwarding**: Enabled on the NIC (required for routing)
- **NSG Rules**:
  - Inbound: Allow `VirtualNetwork` and `AzureLoadBalancer`; deny all other
  - Outbound: Allow `VirtualNetwork` and `Internet`

## UDR (User Defined Routes)

The agent subnet has a route table that sends all default (`0.0.0.0/0`) traffic through the GSA proxy VM as a virtual appliance. The following Azure service tags are **excepted** (routed directly to Internet):

| Service Tag | Purpose |
|---|---|
| `AzureActiveDirectory` | Entra ID authentication |
| `AzureResourceManager` | ARM API calls |
| `AzureMonitor` | Monitoring and diagnostics |
| `GuestAndHybridManagement` | VM guest agent management |
| `AzureContainerRegistry` | Container image pulls |
| `AzureKeyVault` | Key Vault access |
| `Storage` | Azure Storage access |
| `AzureFrontDoor.FirstParty` | Azure Front Door first-party services |
| `ContainerAppsManagement` | Container Apps management plane |

## Prerequisites

1. **Accept the marketplace terms** for the GSA AI Connector image before deploying:
   ```bash
   az vm image terms accept \
     --publisher microsoftcorporation1687208452115 \
     --offer gsaaiconnector1-preview \
     --plan gsaaiconnectorplan1
   ```

2. **Generate an SSH key pair** for the proxy VM:
   ```bash
   ssh-keygen -t rsa -b 4096 -f ~/.ssh/gsa-proxy-key -N ""
   ```

## Deployment

### Using Azure CLI with Bicep parameter file

```bash
# Set your SSH public key
export GSA_PROXY_SSH_PUBLIC_KEY=$(cat ~/.ssh/gsa-proxy-key.pub)

# Create resource group
az group create --name rg-ai-agent-gsa-proxy --location eastus

# Deploy
az deployment group create \
  --resource-group rg-ai-agent-gsa-proxy \
  --template-file main.bicep \
  --parameters main.bicepparam
```

### Using inline parameters

```bash
az deployment group create \
  --resource-group rg-ai-agent-gsa-proxy \
  --template-file main.bicep \
  --parameters \
    location=eastus \
    aiServices=foundry \
    gsaProxySshPublicKey="$(cat ~/.ssh/gsa-proxy-key.pub)"
```

## Module Structure

```
46-standard-agent-byovnet-gsa-proxy-setup/
├── main.bicep                              # Orchestrator - deploys everything
├── main.bicepparam                         # Parameter file
├── README.md                               # This file
└── modules/
    ├── vnet.bicep                           # VNet with agent + proxy subnets
    ├── gsa-proxy.bicep                      # GSA proxy VM, NIC, NSG, UDR
    ├── agent-subnet-udr-association.bicep   # Associates UDR to agent subnet
    ├── ai-account-identity.bicep            # AI Services with subnet injection
    ├── ai-project-identity.bicep            # AI Project
    ├── standard-dependent-resources.bicep   # Cosmos DB, AI Search, Storage
    ├── add-project-capability-host.bicep    # Capability host configuration
    ├── validate-existing-resources.bicep    # Validates BYO resources
    ├── format-project-workspace-id.bicep    # Workspace ID formatting
    ├── ai-search-role-assignments.bicep     # AI Search RBAC
    ├── azure-storage-account-role-assignment.bicep  # Storage RBAC
    ├── blob-storage-container-role-assignments.bicep # Blob container RBAC
    ├── cosmos-container-role-assignments.bicep       # Cosmos container RBAC
    └── cosmosdb-account-role-assignment.bicep        # Cosmos account RBAC
```

## Parameters

| Parameter | Type | Default | Description |
|---|---|---|---|
| `location` | string | `eastus` | Azure region |
| `aiServices` | string | `foundry` | AI Services resource name (max 9 chars) |
| `firstProjectName` | string | `project` | Project resource name |
| `vnetName` | string | `agent-vnet` | Virtual network name |
| `vnetAddressPrefix` | string | `172.16.0.0/16` | VNet CIDR |
| `agentSubnetPrefix` | string | `172.16.0.0/24` | Agent subnet CIDR |
| `gsaProxySubnetPrefix` | string | `172.16.1.0/24` | GSA proxy subnet CIDR |
| `gsaProxyVmSize` | string | `Standard_D2s_v3` | Proxy VM size |
| `gsaProxySshPublicKey` | secure string | *(required)* | SSH public key for VM |
| `modelName` | string | `gpt-4.1` | Model to deploy |
| `modelCapacity` | int | `30` | TPM for model deployment |
| `aiSearchResourceId` | string | `''` | Optional existing AI Search ARM ID |
| `azureStorageAccountResourceId` | string | `''` | Optional existing Storage ARM ID |
| `azureCosmosDBAccountResourceId` | string | `''` | Optional existing Cosmos DB ARM ID |

## Post-Deployment Steps

### 1. Assign Azure AI User Role

After deployment, users who need to build agents must be assigned the **Azure AI User** role on the AI Services account:

```bash
az role assignment create \
  --assignee <USER_EMAIL_OR_OBJECT_ID> \
  --role "Azure AI User" \
  --scope "/subscriptions/<SUBSCRIPTION_ID>/resourceGroups/<RESOURCE_GROUP>/providers/Microsoft.CognitiveServices/accounts/<AI_ACCOUNT_NAME>"
```

Without this role, you'll see: *"You don't have permission to build agents in this project."*

### 2. Ensure VNet Traffic Is Allowed on Subnet-Level NSGs

The GSA proxy VM requires inbound traffic from the virtual network on **all ports** (including ports 80 and 443). While this template attaches an NSG to the NIC that allows VNet traffic, additional NSGs may be attached to the **subnet** by organizational policies, Azure Policy, or compliance tooling (e.g., NRMS in Microsoft internal subscriptions).

If agent traffic is not reaching the proxy after deployment, check for any additional NSGs on the GSA proxy subnet:

```bash
# Check what NSG is attached to the gsa-proxy-subnet
az network vnet subnet show \
  --resource-group <RESOURCE_GROUP> \
  --vnet-name <VNET_NAME> \
  --name gsa-proxy-subnet \
  --query "networkSecurityGroup.id" -o tsv

# List all NSGs in the resource group
az network nsg list --resource-group <RESOURCE_GROUP> --query "[].name" -o table
```

If a subnet-level NSG exists and does not allow VNet inbound traffic on all ports, add an allow rule:

```bash
az network nsg rule create \
  --resource-group <RESOURCE_GROUP> \
  --nsg-name <SUBNET_NSG_NAME> \
  --name AllowVNetInbound \
  --priority 100 \
  --direction Inbound \
  --access Allow \
  --protocol '*' \
  --source-address-prefixes VirtualNetwork \
  --destination-address-prefixes '*' \
  --destination-port-ranges '*'
```

> **Why is this needed?** Azure evaluates NSG rules at both the subnet and NIC level. Traffic must be allowed by **both** NSGs. If a subnet-level NSG blocks port 80 or 443 from the VNet, agent traffic will be dropped before reaching the proxy VM — even though the NIC-level NSG allows it.
