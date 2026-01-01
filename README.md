# 🛡️ Azure Kubernetes Service (AKS) - Disaster Recovery Strategy

> **Enterprise-grade DR implementation for Full-Stack Applications on Azure**  
> Angular Frontend • Spring Boot Backend • Multi-Region Resilience

[![Azure](https://img.shields.io/badge/Azure-AKS-0078D4?logo=microsoft-azure)](https://azure.microsoft.com/en-us/services/kubernetes-service/)
[![Terraform](https://img.shields.io/badge/IaC-Terraform-7B42BC?logo=terraform)](https://www.terraform.io/)
[![DR Ready](https://img.shields.io/badge/DR-Production%20Ready-success)](/)
[![RTO](https://img.shields.io/badge/RTO-%E2%89%A4%2030min-green)]()
[![RPO](https://img.shields.io/badge/RPO-%E2%89%A4%205min-green)]()

---

## 📚 Table of Contents

- [Overview](#-overview)
- [DR Objectives](#-dr-objectives)
- [Architecture](#-architecture)
- [DR Strategy Explained](#-dr-strategy-explained)
- [Infrastructure Components](#-infrastructure-components)
- [Terraform Implementation](#-terraform-implementation)
- [Failover Procedures](#-failover-procedures)
- [Testing & Validation](#-testing--validation)
- [Cost Optimization](#-cost-optimization)
- [Security Considerations](#-security-considerations)
- [Monitoring & Alerts](#-monitoring--alerts)
- [Best Practices](#-best-practices)
- [FAQ](#-faq)

---

## 🎯 Overview

This repository provides a **production-ready Disaster Recovery (DR) plan** for full-stack applications running on Azure Kubernetes Service (AKS). The solution is designed for enterprises requiring high availability, minimal data loss, and rapid recovery from regional failures.

### 🏗️ Application Stack

```
┌─────────────────────────────────────────┐
│           Frontend Layer                 │
│  Angular SPA → Nginx → Docker Container │
└─────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────┐
│           Backend Layer                  │
│    Spring Boot REST API → Stateless     │
└─────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────┐
│           Data Layer                     │
│  Azure SQL / PostgreSQL / Cosmos DB     │
└─────────────────────────────────────────┘
```

### 🎯 Who This Is For

- **DevOps Engineers** implementing DR strategies
- **SREs** managing mission-critical workloads
- **Cloud Architects** designing resilient systems
- **Platform Teams** ensuring business continuity

---

## 📊 DR Objectives

### Key Metrics

| Metric | Target | Description |
|--------|--------|-------------|
| **RTO** (Recovery Time Objective) | **≤ 30 minutes** | Maximum acceptable downtime |
| **RPO** (Recovery Point Objective) | **≤ 5 minutes** | Maximum acceptable data loss |
| **Availability SLA** | **99.9%** | System uptime guarantee |
| **MTTR** (Mean Time To Recovery) | **15-30 minutes** | Average recovery time |

### 🛡️ Failure Scenarios Covered

- ✅ **Availability Zone (AZ) Failure** - Single datacenter outage
- ✅ **Regional Failure** - Entire Azure region unavailable
- ✅ **Application Failure** - Pod crashes, deployment issues
- ✅ **Database Failure** - Primary database corruption
- ✅ **Network Failure** - Connectivity issues
- ✅ **Infrastructure Failure** - Node pool degradation

---

## 🏛️ Architecture

### 🌍 Multi-Region Strategy

```
                    ┌─────────────────────────────┐
                    │     Global Traffic Layer     │
                    │   Azure Traffic Manager      │
                    │   (DNS-based Failover)       │
                    └──────────┬──────────────────┘
                               │
              ┌────────────────┴────────────────┐
              │                                 │
    ┌─────────▼─────────┐           ┌──────────▼────────┐
    │   PRIMARY REGION  │           │    DR REGION       │
    │     (East US)     │           │   (Central US)     │
    │                   │           │                    │
    │  ┌─────────────┐  │           │  ┌─────────────┐  │
    │  │  AKS Cluster │  │           │  │ AKS Cluster  │  │
    │  │   (Active)   │  │           │  │ (Warm Stdby) │  │
    │  │              │  │           │  │              │  │
    │  │ • 3 Nodes    │  │           │  │ • 1-2 Nodes  │  │
    │  │ • Autoscale  │  │           │  │ • Scaled Down│  │
    │  └──────┬───────┘  │           │  └──────┬───────┘  │
    │         │          │           │         │          │
    │  ┌──────▼───────┐  │           │  ┌──────▼───────┐  │
    │  │ Ingress-NGINX│  │           │  │Ingress-NGINX │  │
    │  │    + TLS     │  │◄─Sync────►│  │   + TLS      │  │
    │  └──────┬───────┘  │           │  └──────┬───────┘  │
    │         │          │           │         │          │
    │  ┌──────▼───────┐  │           │  ┌──────▼───────┐  │
    │  │   Frontend   │  │           │  │  Frontend    │  │
    │  │   (Angular)  │  │           │  │  (Angular)   │  │
    │  └──────────────┘  │           │  └──────────────┘  │
    │  ┌──────────────┐  │           │  ┌──────────────┐  │
    │  │   Backend    │  │           │  │   Backend    │  │
    │  │(Spring Boot) │  │           │  │(Spring Boot) │  │
    │  └──────┬───────┘  │           │  └──────┬───────┘  │
    └─────────┼──────────┘           └─────────┼─────────┘
              │                                 │
              └────────┬────────────────────────┘
                       │
           ┌───────────▼──────────────┐
           │   Azure SQL Database      │
           │   (Geo-Replicated)        │
           │                           │
           │  Primary ──Async──► DR   │
           │  (Read-Write)  (Read-Only)│
           └───────────────────────────┘
```

### 🔄 Component Distribution

| Component | Primary Region | DR Region | Sync Method |
|-----------|----------------|-----------|-------------|
| **AKS Cluster** | Active (3 nodes) | Warm Standby (1-2 nodes) | GitOps |
| **Container Registry** | Active | Geo-replicated | Auto-sync |
| **Database** | Primary (RW) | Geo-replica (RO) | Async replication |
| **Storage** | GRS | Read-access replica | Azure Storage |
| **DNS/Traffic** | Traffic Manager | Priority-based routing | Health probes |
| **Secrets** | Key Vault Primary | Key Vault DR | Manual sync |

---

## 💡 DR Strategy Explained

### Why This Architecture Works

Our DR strategy is built on **three core principles**:

#### 1️⃣ **Stateless Design** 
All application components (Angular + Spring Boot) are stateless. This enables:
- Fast pod recreation in DR region
- No session data loss
- Horizontal scaling without complications
- Quick rollback capabilities

#### 2️⃣ **DNS-Based Failover**
Azure Traffic Manager uses DNS to route users:
- Health probes monitor `/actuator/health` endpoint
- Automatic failover when primary fails
- Sub-60 second detection
- No code changes required

#### 3️⃣ **Data Layer Resilience**
Database geo-replication ensures:
- Continuous data synchronization
- Minimal RPO (≤5 minutes)
- Automatic promotion in DR scenarios
- Point-in-time recovery options

### 🎨 Failover Flow Visualization

```
Normal Operation:
User → DNS → Primary AKS → Primary DB
                 ↓
              (Healthy)

During Failure:
User → DNS → Primary AKS (DOWN!)
         ↓
    Traffic Manager detects failure
         ↓
User → DNS → DR AKS → DR DB
                ↓
           (Activated)

Time to failover: 15-30 minutes
```

---

## 🏗️ Infrastructure Components

### 1. **Network Architecture**

#### Primary Region (East US)
```
VNET-Primary: 10.0.0.0/16
  ├── AKS Subnet: 10.0.1.0/24
  ├── Application Gateway Subnet: 10.0.2.0/24
  └── Private Link Subnet: 10.0.3.0/24
```

#### DR Region (Central US)
```
VNET-DR: 10.1.0.0/16
  ├── AKS Subnet: 10.1.1.0/24
  ├── Application Gateway Subnet: 10.1.2.0/24
  └── Private Link Subnet: 10.1.3.0/24
```

**Design Principles:**
- ✅ No overlapping IP ranges
- ✅ Identical subnet structure for consistency
- ✅ Reserved capacity for scaling
- ✅ VNet peering for cross-region communication (optional)

### 2. **AKS Cluster Configuration**

#### Primary Cluster
```yaml
Name: aks-prod-eastus
Location: East US
Node Pools:
  - System Pool: 3 nodes (Standard_DS2_v2)
  - User Pool: 3-10 nodes (autoscale)
Features:
  - Azure CNI networking
  - Managed identity
  - Azure Monitor integration
  - Ingress-NGINX controller
```

#### DR Cluster
```yaml
Name: aks-dr-centralus
Location: Central US
Node Pools:
  - System Pool: 1 node (cost-optimized)
  - User Pool: 1-2 nodes (ready to scale)
Features:
  - Identical configuration to primary
  - Pre-deployed applications
  - Same Kubernetes version
  - GitOps sync enabled
```

### 3. **Container Registry**

```
Azure Container Registry (ACR)
├── SKU: Premium (required for geo-replication)
├── Primary: East US
├── Replica: Central US
└── Features:
    ├── Automated geo-replication
    ├── Webhook triggers
    ├── Image scanning (Azure Defender)
    └── Content trust
```

### 4. **Database Strategy**

#### Option A: Azure SQL Database (Recommended for RDBMS)
```
Primary: East US (Read-Write)
├── Geo-Replica: Central US (Read-Only)
├── Replication Lag: <5 seconds
├── Automatic failover groups
└── RPO: ~5 minutes
```

#### Option B: Azure Database for PostgreSQL
```
Primary Server: East US
├── Read Replica: Central US
├── Async replication
└── Manual promotion required
```

#### Option C: Azure Cosmos DB (Recommended for NoSQL)
```
Multi-region write enabled
├── East US (Primary)
├── Central US (Secondary)
├── Automatic failover
└── RPO: Near-zero
```

### 5. **Traffic Management**

```
Azure Traffic Manager Profile
├── Routing Method: Priority
├── DNS TTL: 30 seconds
├── Health Check:
│   ├── Protocol: HTTPS
│   ├── Port: 443
│   ├── Path: /actuator/health
│   └── Interval: 10 seconds
├── Endpoints:
│   ├── Primary: Priority 1
│   └── DR: Priority 2
```

---

## 🔧 Terraform Implementation

### 📁 Repository Structure

```
aks-dr-terraform/
├── README.md
├── .gitignore
├── main.tf                    # Root module orchestration
├── variables.tf               # Input variables
├── outputs.tf                 # Output values
├── terraform.tfvars          # Default values
├── modules/
│   ├── network/              # VNet, Subnets, NSGs
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── aks/                  # AKS cluster configuration
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── acr/                  # Container registry
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── database/             # Azure SQL/PostgreSQL
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── traffic-manager/      # DNS failover
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   └── keyvault/             # Secrets management
│       ├── main.tf
│       ├── variables.tf
│       └── outputs.tf
├── environments/
│   ├── prod/                 # Primary region config
│   │   ├── main.tf
│   │   └── terraform.tfvars
│   └── dr/                   # DR region config
│       ├── main.tf
│       └── terraform.tfvars
└── scripts/
    ├── deploy.sh             # Deployment automation
    ├── failover.sh           # Manual failover script
    └── validate.sh           # Infrastructure validation
```

### 🌐 Module 1: Network Infrastructure

**File:** `modules/network/main.tf`

```hcl
# Virtual Network
resource "azurerm_virtual_network" "vnet" {
  name                = var.vnet_name
  location            = var.location
  resource_group_name = var.resource_group_name
  address_space       = [var.vnet_address_space]

  tags = var.tags
}

# AKS Subnet
resource "azurerm_subnet" "aks" {
  name                 = "aks-subnet"
  resource_group_name  = var.resource_group_name
  virtual_network_name = azurerm_virtual_network.vnet.name
  address_prefixes     = [var.aks_subnet_cidr]
}

# Application Gateway Subnet
resource "azurerm_subnet" "appgw" {
  name                 = "appgw-subnet"
  resource_group_name  = var.resource_group_name
  virtual_network_name = azurerm_virtual_network.vnet.name
  address_prefixes     = [var.appgw_subnet_cidr]
}

# Network Security Group
resource "azurerm_network_security_group" "aks_nsg" {
  name                = "${var.vnet_name}-aks-nsg"
  location            = var.location
  resource_group_name = var.resource_group_name

  security_rule {
    name                       = "AllowHTTPS"
    priority                   = 100
    direction                  = "Inbound"
    access                     = "Allow"
    protocol                   = "Tcp"
    source_port_range          = "*"
    destination_port_range     = "443"
    source_address_prefix      = "*"
    destination_address_prefix = "*"
  }

  tags = var.tags
}

# Associate NSG with AKS Subnet
resource "azurerm_subnet_network_security_group_association" "aks" {
  subnet_id                 = azurerm_subnet.aks.id
  network_security_group_id = azurerm_network_security_group.aks_nsg.id
}
```

**File:** `modules/network/variables.tf`

```hcl
variable "vnet_name" {
  description = "Name of the virtual network"
  type        = string
}

variable "location" {
  description = "Azure region"
  type        = string
}

variable "resource_group_name" {
  description = "Resource group name"
  type        = string
}

variable "vnet_address_space" {
  description = "Address space for VNet"
  type        = string
  default     = "10.0.0.0/16"
}

variable "aks_subnet_cidr" {
  description = "CIDR for AKS subnet"
  type        = string
  default     = "10.0.1.0/24"
}

variable "appgw_subnet_cidr" {
  description = "CIDR for Application Gateway subnet"
  type        = string
  default     = "10.0.2.0/24"
}

variable "tags" {
  description = "Tags to apply to resources"
  type        = map(string)
  default     = {}
}
```

**File:** `modules/network/outputs.tf`

```hcl
output "vnet_id" {
  description = "ID of the virtual network"
  value       = azurerm_virtual_network.vnet.id
}

output "aks_subnet_id" {
  description = "ID of the AKS subnet"
  value       = azurerm_subnet.aks.id
}

output "appgw_subnet_id" {
  description = "ID of the Application Gateway subnet"
  value       = azurerm_subnet.appgw.id
}
```

---

### ☸️ Module 2: AKS Cluster

**File:** `modules/aks/main.tf`

```hcl
# AKS Cluster
resource "azurerm_kubernetes_cluster" "aks" {
  name                = var.cluster_name
  location            = var.location
  resource_group_name = var.resource_group_name
  dns_prefix          = var.cluster_name
  kubernetes_version  = var.kubernetes_version

  # System Node Pool
  default_node_pool {
    name                = "system"
    node_count          = var.system_node_count
    vm_size             = var.system_node_size
    vnet_subnet_id      = var.subnet_id
    type                = "VirtualMachineScaleSets"
    enable_auto_scaling = var.enable_auto_scaling
    min_count           = var.enable_auto_scaling ? var.min_node_count : null
    max_count           = var.enable_auto_scaling ? var.max_node_count : null
    
    upgrade_settings {
      max_surge = "10%"
    }
  }

  # Managed Identity
  identity {
    type = "SystemAssigned"
  }

  # Network Profile
  network_profile {
    network_plugin     = "azure"
    network_policy     = "azure"
    load_balancer_sku  = "standard"
    service_cidr       = var.service_cidr
    dns_service_ip     = var.dns_service_ip
  }

  # Azure Monitor Integration
  azure_monitor {
    log_analytics_workspace_id = var.log_analytics_workspace_id
  }

  # Azure AD Integration
  azure_active_directory_role_based_access_control {
    managed                = true
    azure_rbac_enabled     = true
  }

  # Key Vault Secrets Provider
  key_vault_secrets_provider {
    secret_rotation_enabled = true
  }

  tags = merge(
    var.tags,
    {
      Environment = var.environment
      ManagedBy   = "Terraform"
    }
  )
}

# User Node Pool (for application workloads)
resource "azurerm_kubernetes_cluster_node_pool" "user" {
  count                 = var.enable_user_node_pool ? 1 : 0
  name                  = "user"
  kubernetes_cluster_id = azurerm_kubernetes_cluster.aks.id
  vm_size               = var.user_node_size
  node_count            = var.user_node_count
  vnet_subnet_id        = var.subnet_id
  enable_auto_scaling   = true
  min_count             = var.user_min_nodes
  max_count             = var.user_max_nodes

  node_labels = {
    "workload" = "application"
  }

  tags = var.tags
}
```

**File:** `modules/aks/variables.tf`

```hcl
variable "cluster_name" {
  description = "Name of the AKS cluster"
  type        = string
}

variable "location" {
  description = "Azure region"
  type        = string
}

variable "resource_group_name" {
  description = "Resource group name"
  type        = string
}

variable "kubernetes_version" {
  description = "Kubernetes version"
  type        = string
  default     = "1.28"
}

variable "subnet_id" {
  description = "Subnet ID for AKS nodes"
  type        = string
}

variable "system_node_count" {
  description = "Number of nodes in system pool"
  type        = number
  default     = 3
}

variable "system_node_size" {
  description = "VM size for system nodes"
  type        = string
  default     = "Standard_DS2_v2"
}

variable "enable_auto_scaling" {
  description = "Enable autoscaling"
  type        = bool
  default     = true
}

variable "min_node_count" {
  description = "Minimum node count for autoscaling"
  type        = number
  default     = 2
}

variable "max_node_count" {
  description = "Maximum node count for autoscaling"
  type        = number
  default     = 10
}

variable "service_cidr" {
  description = "Service CIDR for Kubernetes"
  type        = string
  default     = "10.2.0.0/16"
}

variable "dns_service_ip" {
  description = "DNS service IP"
  type        = string
  default     = "10.2.0.10"
}

variable "log_analytics_workspace_id" {
  description = "Log Analytics Workspace ID"
  type        = string
}

variable "enable_user_node_pool" {
  description = "Enable user node pool"
  type        = bool
  default     = true
}

variable "user_node_count" {
  description = "Initial user node count"
  type        = number
  default     = 2
}

variable "user_node_size" {
  description = "VM size for user nodes"
  type        = string
  default     = "Standard_DS3_v2"
}

variable "user_min_nodes" {
  description = "Min nodes in user pool"
  type        = number
  default     = 1
}

variable "user_max_nodes" {
  description = "Max nodes in user pool"
  type        = number
  default     = 10
}

variable "environment" {
  description = "Environment name"
  type        = string
}

variable "tags" {
  description = "Tags"
  type        = map(string)
  default     = {}
}
```

---

### 📦 Module 3: Azure Container Registry

**File:** `modules/acr/main.tf`

```hcl
# Container Registry
resource "azurerm_container_registry" "acr" {
  name                = var.acr_name
  resource_group_name = var.resource_group_name
  location            = var.primary_location
  sku                 = "Premium"  # Required for geo-replication
  admin_enabled       = false

  # Enable image quarantine
  quarantine_policy_enabled = true

  # Retention policy
  retention_policy {
    days    = var.retention_days
    enabled = true
  }

  # Trust policy
  trust_policy {
    enabled = true
  }

  # Network rules
  network_rule_set {
    default_action = "Allow"
  }

  tags = var.tags
}

# Geo-Replication to DR Region
resource "azurerm_container_registry_replication" "dr" {
  name                  = "dr-replication"
  container_registry_id = azurerm_container_registry.acr.id
  location              = var.dr_location
  zone_redundancy_enabled = true

  tags = var.tags
}

# Role assignment for AKS to pull images
resource "azurerm_role_assignment" "aks_acr_pull" {
  scope                = azurerm_container_registry.acr.id
  role_definition_name = "AcrPull"
  principal_id         = var.aks_principal_id
}
```

---

### 🌐 Module 4: Traffic Manager

**File:** `modules/traffic-manager/main.tf`

```hcl
# Traffic Manager Profile
resource "azurerm_traffic_manager_profile" "tm" {
  name                   = var.profile_name
  resource_group_name    = var.resource_group_name
  traffic_routing_method = "Priority"

  dns_config {
    relative_name = var.dns_name
    ttl           = 30
  }

  monitor_config {
    protocol                     = "HTTPS"
    port                         = 443
    path                         = var.health_check_path
    interval_in_seconds          = 10
    timeout_in_seconds           = 5
    tolerated_number_of_failures = 2
  }

  tags = var.tags
}

# Primary Endpoint
resource "azurerm_traffic_manager_external_endpoint" "primary" {
  name       = "primary-endpoint"
  profile_id = azurerm_traffic_manager_profile.tm.id
  target     = var.primary_endpoint
  priority   = 1
  weight     = 100
}

# DR Endpoint
resource "azurerm_traffic_manager_external_endpoint" "dr" {
  name       = "dr-endpoint"
  profile_id = azurerm_traffic_manager_profile.tm.id
  target     = var.dr_endpoint
  priority   = 2
  weight     = 100
}
```

---

### 🔐 Module 5: Key Vault

**File:** `modules/keyvault/main.tf`

```hcl
data "azurerm_client_config" "current" {}

# Key Vault
resource "azurerm_key_vault" "kv" {
  name                       = var.keyvault_name
  location                   = var.location
  resource_group_name        = var.resource_group_name
  tenant_id                  = data.azurerm_client_config.current.tenant_id
  sku_name                   = "premium"
  soft_delete_retention_days = 90
  purge_protection_enabled   = true

  enabled_for_deployment          = true
  enabled_for_disk_encryption     = true
  enabled_for_template_deployment = true

  network_acls {
    default_action = "Allow"
    bypass         = "AzureServices"
  }

  tags = var.tags
}

# Access Policy for AKS
resource "azurerm_key_vault_access_policy" "aks" {
  key_vault_id = azurerm_key_vault.kv.id
  tenant_id    = data.azurerm_client_config.current.tenant_id
  object_id    = var.aks_identity_object_id

  secret_permissions = [
    "Get",
    "List"
  ]

  certificate_permissions = [
    "Get",
    "List"
  ]
}
```

---

### 🌍 Environment Configuration

**File:** `environments/prod/terraform.tfvars`

```hcl
# Primary Region Configuration
environment = "production"
location    = "eastus"

# Network
vnet_address_space = "10.0.0.0/16"
aks_subnet_cidr    = "10.0.1.0/24"

# AKS Configuration
cluster_name       = "aks-prod-eastus"
kubernetes_version = "1.28"
system_node_count  = 3
enable_auto_scaling = true
min_node_count     = 2
max_node_count     = 10

# Tags
tags = {
  Environment = "Production"
  Region      = "Primary"
  ManagedBy   = "Terraform"
  CostCenter  = "Platform"
}
```

**File:** `environments/dr/terraform.tfvars`

```hcl
# DR Region Configuration
environment = "disaster-recovery"
location    = "centralus"

# Network
vnet_address_space = "10.1.0.0/16"
aks_subnet_cidr    = "10.1.1.0/24"

# AKS Configuration (Cost-Optimized)
cluster_name       = "aks-dr-centralus"
kubernetes_version = "1.28"
system_node_count  = 1
enable_auto_scaling = true
min_node_count     = 1
max_node_count     = 5

# Tags
tags = {
  Environment = "Disaster Recovery"
  Region      = "Secondary"
  ManagedBy   = "Terraform"
  CostCenter  = "Platform"
}
```

---

## 🔄 Failover Procedures

### 🚨 Automatic Failover (Recommended)

**Trigger:** Health check failures detected by Traffic Manager

```
Step 1: Traffic Manager Health Probe
├── Checks: /actuator/health (every 10s)
├── Timeout: 5 seconds
└── Failures: 2 consecutive failures

Step 2: Automatic Detection
├── Primary endpoint marked unhealthy
├── DNS records updated (TTL: 30s)
└── Traffic routed to DR endpoint

Step 3: DR Cluster Activation
├── Node pool auto-scales (1→3 nodes)
├── Pods already running (warm standby)
└── Load balancer distributes traffic

Step 4: Database Failover
├── Azure SQL failover group activates
├── DR database promoted to primary
└── Connection strings remain unchanged

Total Time: 15-30 minutes
```

### 🔧 Manual Failover (Planned Maintenance)

**Use Case:** Planned region migration, testing, or maintenance

**Script:** `scripts/failover.sh`

```bash
#!/bin/bash
set -e

echo "🚨 Starting Manual DR Failover..."

# 1. Scale up DR cluster
echo "📈 Scaling DR AKS cluster..."
az aks scale \
  --resource-group rg-dr \
  --name aks-dr-centralus \
  --node-count 3 \
  --nodepool-name system

# 2. Verify DR pods are healthy
echo "✅ Verifying DR pods..."
kubectl config use-context aks-dr-centralus
kubectl get pods --all-namespaces

# 3. Update Traffic Manager priority
echo "🌐 Updating Traffic Manager..."
az network traffic-manager endpoint update \
  --resource-group rg-shared \
  --profile-name tm-aks-dr \
  --name dr-endpoint \
  --type externalEndpoints \
  --priority 1

az network traffic-manager endpoint update \
  --resource-group rg-shared \
  --profile-name tm-aks-dr \
  --name primary-endpoint \
  --type externalEndpoints \
  --priority 2

# 4. Failover database
echo "💾 Initiating database failover..."
az sql failover-group set-primary \
  --resource-group rg-dr \
  --server sql-dr-server \
  --name sql-failover-group

# 5. Verify application health
echo "🔍 Health check verification..."
curl -f https://myapp.trafficmanager.net/actuator/health

echo "✅ Failover completed successfully!"
echo "⏱️  Elapsed time: $(date)"
```

### 📊 Failover Validation Checklist

```bash
# Validation Script
#!/bin/bash

echo "🔍 DR Failover Validation"
echo "========================"

# 1. DNS Resolution
echo "1️⃣ Testing DNS..."
nslookup myapp.trafficmanager.net

# 2. Health Endpoint
echo "2️⃣ Testing application health..."
curl -s https://myapp.trafficmanager.net/actuator/health | jq

# 3. Database Connectivity
echo "3️⃣ Testing database..."
kubectl exec -n production deployment/backend -- \
  curl -s localhost:8080/api/health/db

# 4. Pod Status
echo "4️⃣ Checking pod status..."
kubectl get pods -n production

# 5. Node Status
echo "5️⃣ Checking node status..."
kubectl get nodes

# 6. Ingress Status
echo "6️⃣ Checking ingress..."
kubectl get ingress -n production

echo "✅ Validation complete!"
```

---

### 🔙 Failback Procedure

**When primary region is restored:**

```bash
#!/bin/bash
# scripts/failback.sh

echo "🔄 Starting Failback to Primary Region..."

# 1. Verify primary region is healthy
echo "1️⃣ Verifying primary region..."
az aks show \
  --resource-group rg-prod \
  --name aks-prod-eastus \
  --query "powerState.code" -o tsv

# 2. Sync data back to primary database
echo "2️⃣ Syncing database..."
# Database automatically syncs via geo-replication

# 3. Validate primary application
echo "3️⃣ Testing primary endpoints..."
kubectl config use-context aks-prod-eastus
kubectl get pods --all-namespaces

# 4. Update Traffic Manager (gradual shift)
echo "4️⃣ Gradual traffic shift..."
# Week 1: 10% to primary
az network traffic-manager endpoint update \
  --name primary-endpoint --weight 10
# Week 2: 50% to primary
az network traffic-manager endpoint update \
  --name primary-endpoint --weight 50
# Week 3: 100% to primary
az network traffic-manager endpoint update \
  --name primary-endpoint --priority 1

# 5. Scale down DR cluster
echo "5️⃣ Scaling down DR cluster..."
az aks scale \
  --resource-group rg-dr \
  --name aks-dr-centralus \
  --node-count 1

echo "✅ Failback completed!"
```

---

## 🧪 Testing & Validation

### 📅 DR Testing Schedule

| Test Type | Frequency | Duration | Scope |
|-----------|-----------|----------|-------|
| **Health Check Verification** | Daily | 5 min | Automated |
| **Backup Validation** | Weekly | 15 min | Automated |
| **Partial Failover Test** | Monthly | 1 hour | Single service |
| **Full DR Drill** | Quarterly | 4 hours | Complete failover |
| **Chaos Engineering** | Bi-annually | 1 day | Random failures |

### 🎯 Quarterly DR Drill Procedure

**Objective:** Validate entire DR process end-to-end

#### Phase 1: Pre-Drill Preparation (1 week before)

```bash
# 1. Review and update runbooks
# 2. Notify stakeholders
# 3. Schedule maintenance window
# 4. Prepare monitoring dashboards
# 5. Document baseline metrics

# Create drill checklist
cat > dr-drill-checklist.md <<EOF
## DR Drill Checklist - $(date +%Y-%m-%d)

### Pre-Drill
- [ ] Stakeholders notified
- [ ] Monitoring enabled
- [ ] Backup verified
- [ ] Team assembled

### During Drill
- [ ] Primary region disabled
- [ ] Traffic Manager failover
- [ ] Database failover
- [ ] Application validation
- [ ] Performance metrics

### Post-Drill
- [ ] Failback completed
- [ ] RTO/RPO measured
- [ ] Issues documented
- [ ] Runbook updated
EOF
```

#### Phase 2: Drill Execution

```bash
#!/bin/bash
# scripts/dr-drill.sh

START_TIME=$(date +%s)
echo "🎯 DR Drill Started: $(date)"

# 1. Simulate primary region failure
echo "Step 1: Simulating primary failure..."
az network nsg rule create \
  --resource-group rg-prod \
  --nsg-name aks-nsg \
  --name BlockAllTraffic \
  --priority 100 \
  --direction Inbound \
  --access Deny \
  --protocol '*' \
  --source-address-prefix '*' \
  --destination-address-prefix '*'

# 2. Monitor failover time
echo "Step 2: Monitoring failover..."
while true; do
  STATUS=$(curl -s -o /dev/null -w "%{http_code}" https://myapp.trafficmanager.net/health)
  if [ "$STATUS" == "200" ]; then
    END_TIME=$(date +%s)
    RTO=$((END_TIME - START_TIME))
    echo "✅ Application recovered! RTO: ${RTO} seconds"
    break
  fi
  echo "Waiting for recovery... ($STATUS)"
  sleep 10
done

# 3. Validate all components
echo "Step 3: Validation..."
./scripts/validate.sh

# 4. Measure data loss (RPO)
echo "Step 4: Measuring RPO..."
# Query last transaction timestamp
kubectl exec -n production deployment/backend -- \
  curl -s localhost:8080/api/metrics/last-transaction

# 5. Restore primary region
echo "Step 5: Cleanup..."
az network nsg rule delete \
  --resource-group rg-prod \
  --nsg-name aks-nsg \
  --name BlockAllTraffic

echo "🎯 DR Drill Completed!"
echo "📊 Results:"
echo "   RTO: ${RTO} seconds (Target: 1800s)"
echo "   RPO: Check database logs"
```

#### Phase 3: Post-Drill Analysis

```markdown
## DR Drill Report - Q1 2024

### Executive Summary
- **RTO Achieved:** 24 minutes (Target: ≤30 min) ✅
- **RPO Measured:** 2 minutes (Target: ≤5 min) ✅
- **Success Rate:** 100%

### Metrics Collected
| Metric | Target | Actual | Status |
|--------|--------|--------|--------|
| Detection Time | <1 min | 45 sec | ✅ |
| DNS Propagation | <2 min | 1.5 min | ✅ |
| Cluster Scale-up | <5 min | 4 min | ✅ |
| DB Failover | <2 min | 90 sec | ✅ |
| App Recovery | <30 min | 24 min | ✅ |

### Issues Identified
1. ⚠️ Manual intervention needed for database promotion
2. ⚠️ Some monitoring alerts didn't trigger
3. ✅ All pods recovered successfully

### Action Items
- [ ] Automate database failover
- [ ] Fix alert configuration
- [ ] Update runbook with new learnings
- [ ] Schedule next drill: Q2 2024
```

---

## 💰 Cost Optimization

### 📊 Cost Breakdown (Monthly Estimate)

| Component | Primary Region | DR Region | Total |
|-----------|----------------|-----------|-------|
| **AKS Cluster** | $300 (3 nodes) | $100 (1 node) | $400 |
| **Load Balancer** | $20 | $20 | $40 |
| **Container Registry** | $167 (Premium) | $0 (replicated) | $167 |
| **Azure SQL** | $200 | $200 (replica) | $400 |
| **Storage** | $50 | $25 | $75 |
| **Traffic Manager** | $1 | $0 | $1 |
| **Key Vault** | $3 | $3 | $6 |
| **Monitoring** | $50 | $25 | $75 |
| **Total** | ~$791 | ~$373 | **~$1,164/mo** |

### 💡 Cost Optimization Strategies

#### 1. **Right-Size DR Cluster**

```hcl
# DR cluster configuration
resource "azurerm_kubernetes_cluster" "dr" {
  # Use smaller node size in DR
  default_node_pool {
    vm_size    = "Standard_D2s_v3"  # vs D4s_v3 in prod
    node_count = 1                   # Minimum viable
  }
}
```

**Savings:** ~40% on compute costs

#### 2. **Use Reserved Instances**

```bash
# Purchase 1-year reserved capacity
az reservations reservation-order purchase \
  --reservation-order-id xxx \
  --sku Standard_DS2_v2 \
  --quantity 3 \
  --term P1Y
```

**Savings:** ~30-40% on VM costs

#### 3. **Implement Auto-Shutdown**

```yaml
# Auto-scale DR cluster during non-business hours
apiVersion: v1
kind: ConfigMap
metadata:
  name: cluster-autoscaler-schedule
data:
  scale-down-schedule: "0 22 * * *"  # 10 PM
  scale-up-schedule: "0 6 * * *"     # 6 AM
```

**Savings:** ~25% on DR compute

#### 4. **Optimize Database Tier**

```bash
# Use Basic tier for DR database
az sql db update \
  --resource-group rg-dr \
  --server sql-dr \
  --name myapp-db \
  --service-objective Basic
```

**Savings:** ~50% on database costs

#### 5. **Storage Lifecycle Policies**

```hcl
resource "azurerm_storage_management_policy" "lifecycle" {
  storage_account_id = azurerm_storage_account.dr.id

  rule {
    name    = "archive-old-backups"
    enabled = true
    
    filters {
      blob_types   = ["blockBlob"]
      prefix_match = ["backups/"]
    }
    
    actions {
      base_blob {
        tier_to_cool_after_days    = 30
        tier_to_archive_after_days = 90
        delete_after_days          = 365
      }
    }
  }
}
```

**Savings:** ~60% on backup storage

### 📈 Cost vs. Availability Trade-offs

```
┌─────────────────────────────────────────────┐
│         Cost Optimization Matrix            │
├─────────────────────────────────────────────┤
│                                             │
│  High Cost  ────────────────  High Avail    │
│      │                            │         │
│      │  • Active-Active          │         │
│      │  • Multi-region write     │         │
│      │  • Full redundancy        │         │
│      │                            │         │
│      ▼                            ▼         │
│  Mid Cost   ────────────────  Good Avail    │
│      │                            │         │
│      │  • Active-Warm (Current)  │         │
│      │  • Geo-replication        │         │
│      │  • Auto-scaling           │         │
│      │                            │         │
│      ▼                            ▼         │
│  Low Cost   ────────────────  Basic Avail   │
│      │                            │         │
│      │  • Cold standby           │         │
│      │  • Manual failover        │         │
│      │  • Backup/Restore         │         │
│                                             │
└─────────────────────────────────────────────┘
```

**Recommended:** Active-Warm (Current Implementation)
- Balances cost and recovery time
- Meets 99.9% SLA requirements
- RTO: 15-30 minutes
- Monthly cost: ~$1,200

---

## 🔒 Security Considerations

### 🛡️ Security Layers

```
┌─────────────────────────────────────────┐
│        Network Security                  │
│  • Private endpoints                    │
│  • NSG rules                            │
│  • Azure Firewall                       │
└─────────────────────────────────────────┘
                  ↓
┌─────────────────────────────────────────┐
│        Identity & Access                │
│  • Azure AD integration                 │
│  • RBAC policies                        │
│  • Managed identities                   │
└─────────────────────────────────────────┘
                  ↓
┌─────────────────────────────────────────┐
│        Application Security             │
│  • Pod security policies                │
│  • Network policies                     │
│  • Image scanning                       │
└─────────────────────────────────────────┘
                  ↓
┌─────────────────────────────────────────┐
│        Data Security                    │
│  • Encryption at rest                   │
│  • Encryption in transit                │
│  • Key Vault integration                │
└─────────────────────────────────────────┘
```

### 🔐 Security Implementation

#### 1. Network Isolation

```hcl
# Private AKS cluster
resource "azurerm_kubernetes_cluster" "aks" {
  private_cluster_enabled = true
  
  api_server_access_profile {
    authorized_ip_ranges = [
      "10.0.0.0/8",  # Internal only
    ]
  }
}
```

#### 2. Pod Security

```yaml
# Pod Security Policy
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: restricted
spec:
  privileged: false
  allowPrivilegeEscalation: false
  requiredDropCapabilities:
    - ALL
  volumes:
    - 'configMap'
    - 'secret'
    - 'persistentVolumeClaim'
  runAsUser:
    rule: 'MustRunAsNonRoot'
  seLinux:
    rule: 'RunAsAny'
  fsGroup:
    rule: 'RunAsAny'
```

#### 3. Network Policies

```yaml
# Deny all by default
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
---
# Allow frontend to backend
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: frontend-to-backend
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: backend
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8080
```

#### 4. Secrets Management

```yaml
# Use Azure Key Vault CSI Driver
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: azure-keyvault
spec:
  provider: azure
  parameters:
    keyvaultName: "myapp-kv"
    objects: |
      array:
        - |
          objectName: db-password
          objectType: secret
        - |
          objectName: api-key
          objectType: secret
---
# Use in deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
spec:
  template:
    spec:
      containers:
      - name: app
        volumeMounts:
        - name: secrets
          mountPath: "/mnt/secrets"
          readOnly: true
      volumes:
      - name: secrets
        csi:
          driver: secrets-store.csi.k8s.io
          readOnly: true
          volumeAttributes:
            secretProviderClass: "azure-keyvault"
```

#### 5. Image Security

```yaml
# Azure Container Registry webhook for scanning
resource "azurerm_container_registry_webhook" "scan" {
  name                = "security-scan"
  registry_name       = azurerm_container_registry.acr.name
  resource_group_name = azurerm_resource_group.rg.name
  
  service_uri = "https://defender.azure.com/webhook"
  status      = "enabled"
  scope       = "*"
  actions     = ["push"]
}
```

### 🔍 Security Checklist

- ✅ **Network**: Private endpoints, NSGs, Azure Firewall
- ✅ **Identity**: Azure AD, RBAC, managed identities
- ✅ **Pods**: Security policies, non-root containers
- ✅ **Network Policies**: Default deny, explicit allow
- ✅ **Secrets**: Key Vault, CSI driver, no plain text
- ✅ **Images**: Vulnerability scanning, signed images
- ✅ **Encryption**: TLS in transit, encryption at rest
- ✅ **Compliance**: Azure Policy, regulatory requirements
- ✅ **Monitoring**: Security alerts, audit logs
- ✅ **DR**: Same security posture in both regions

---

## 📊 Monitoring & Alerts

### 🎯 Monitoring Architecture

```
┌─────────────────────────────────────────────┐
│         Application Layer                    │
│  • Application Insights                      │
│  • Custom metrics                            │
│  • Distributed tracing                       │
└──────────────┬──────────────────────────────┘
               │
┌──────────────▼──────────────────────────────┐
│         Kubernetes Layer                     │
│  • Azure Monitor for containers              │
│  • Prometheus metrics                        │
│  • Pod/Node metrics                          │
└──────────────┬──────────────────────────────┘
               │
┌──────────────▼──────────────────────────────┐
│         Infrastructure Layer                 │
│  • Azure Monitor                             │
│  • Log Analytics                             │
│  • Resource health                           │
└──────────────┬──────────────────────────────┘
               │
               ▼
         Alert Rules → PagerDuty/Teams
```

### 📈 Key Metrics Dashboard

```yaml
# Prometheus queries for Grafana dashboard

# 1. Application Health
up{job="backend"} == 0

# 2. Response Time (95th percentile)
histogram_quantile(0.95, 
  rate(http_request_duration_seconds_bucket[5m]))

# 3. Error Rate
rate(http_requests_total{status=~"5.."}[5m])

# 4. Pod Restart Count
kube_pod_container_status_restarts_total

# 5. Node CPU Usage
100 - (avg by (instance) 
  (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)

# 6. Database Connection Pool
hikaricp_connections_active

# 7. Traffic Manager Health
trafficmanager_endpoint_status
```

### 🚨 Alert Configuration

```yaml
# alerts.yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: dr-alerts
spec:
  groups:
  - name: availability
    interval: 30s
    rules:
    - alert: PrimaryClusterDown
      expr: up{cluster="primary"} == 0
      for: 2m
      labels:
        severity: critical
        team: platform
      annotations:
        summary: "Primary cluster is down"
        description: "Primary AKS cluster unreachable for 2 minutes"
        runbook: "https://wiki.company.com/dr-failover"
    
    - alert: HighErrorRate
      expr: |
        rate(http_requests_total{status=~"5.."}[5m]) > 0.05
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "High error rate detected"
        description: "Error rate above 5% for 5 minutes"
    
    - alert: DatabaseReplicationLag
      expr: |
        azure_sql_replication_lag_seconds > 300
      for: 10m
      labels:
        severity: warning
      annotations:
        summary: "Database replication lag high"
        description: "Replication lag exceeds 5 minutes"
```

### 📊 Grafana Dashboard JSON

Create comprehensive dashboard:

```json
{
  "dashboard": {
    "title": "AKS DR Monitoring",
    "panels": [
      {
        "title": "Cluster Health",
        "targets": [
          {
            "expr": "up{job=\"kubernetes-nodes\"}"
          }
        ]
      },
      {
        "title": "Application Response Time",
        "targets": [
          {
            "expr": "histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))"
          }
        ]
      },
      {
        "title": "Traffic Distribution",
        "targets": [
          {
            "expr": "sum(rate(http_requests_total[5m])) by (region)"
          }
        ]
      }
    ]
  }
}
```

---

## 🎓 Best Practices

### ✅ DR Best Practices Checklist

#### Design Principles
- ✅ **Stateless Applications**: No local state, externalize everything
- ✅ **Idempotent Operations**: Safe to retry without side effects
- ✅ **Circuit Breakers**: Fail fast, prevent cascading failures
- ✅ **Health Checks**: Meaningful liveness and readiness probes
- ✅ **Graceful Shutdown**: Handle SIGTERM properly
- ✅ **Configuration Externalization**: Use ConfigMaps and Secrets

#### Infrastructure
- ✅ **Infrastructure as Code**: Everything in Terraform/Bicep
- ✅ **GitOps**: Declarative configs in Git (ArgoCD/Flux)
- ✅ **Immutable Infrastructure**: Never modify, always replace
- ✅ **Version Control**: Tag everything (images, charts, configs)
- ✅ **Automated Testing**: Test DR procedures regularly

#### Data Management
- ✅ **Automated Backups**: Daily backups with retention policies
- ✅ **Backup Testing**: Regularly restore and validate
- ✅ **Point-in-Time Recovery**: Enable for databases
- ✅ **Data Classification**: Know what needs protection
- ✅ **Encryption**: At rest and in transit

#### Operations
- ✅ **Runbooks**: Documented procedures for all scenarios
- ✅ **On-Call Rotation**: 24/7 coverage for critical systems
- ✅ **Incident Management**: Clear escalation paths
- ✅ **Post-Mortems**: Learn from every incident
- ✅ **Chaos Engineering**: Proactively test failure scenarios

### 📝 Deployment Best Practices

```yaml
# Recommended deployment configuration
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0  # Zero-downtime deployments
  template:
    spec:
      containers:
      - name: app
        image: myacr.azurecr.io/backend:v1.2.3
        resources:
          requests:
            cpu: 100m
            memory: 256Mi
          limits:
            cpu: 500m
            memory: 512Mi
        livenessProbe:
          httpGet:
            path: /actuator/health/liveness
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /actuator/health/readiness
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 15"]
      terminationGracePeriodSeconds: 30
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - backend
              topologyKey: kubernetes.io/hostname
```

---

## ❓ FAQ

### General Questions

**Q: What's the difference between RTO and RPO?**

**A:** 
- **RTO (Recovery Time Objective)**: Maximum acceptable downtime. How long can your business tolerate being offline?
- **RPO (Recovery Point Objective)**: Maximum acceptable data loss. How much data can you afford to lose?

Example: RTO of 30 minutes means your app must be back online within 30 minutes. RPO of 5 minutes means you can lose at most 5 minutes of data.

**Q: Why warm standby instead of hot standby?**

**A:** Cost optimization. Warm standby (scaled-down DR cluster) provides excellent RTO while reducing costs by ~60% compared to hot standby (full-size DR cluster running actively).

**Q: Can I use this for on-premises Kubernetes?**

**A:** The concepts apply, but you'll need to replace Azure-specific services:
- Traffic Manager → HAProxy/Consul
- ACR → Harbor/Docker Registry
- Azure SQL → PostgreSQL with streaming replication
- Key Vault → HashiCorp Vault

### Technical Questions

**Q: How does database failover work?**

**A:** Azure SQL Failover Groups provide automatic failover:
1. Primary database replicates to secondary asynchronously
2. On failure, secondary is promoted to primary
3. Connection string remains unchanged (uses listener endpoint)
4. Applications reconnect automatically

**Q: What happens to in-flight requests during failover?**

**A:** 
- Requests to primary: Will fail (503 errors)
- Client retry logic: Should retry with exponential backoff
- New requests: Routed to DR region automatically
- Session data: Lost if not externalized (use Redis/Cosmos)

**Q: How do you handle DNS caching?**

**A:** Traffic Manager uses a 30-second TTL. Most browsers/apps respect this, ensuring quick failover. For critical paths, implement client-side health checks.

**Q: What about certificates during DR?**

**A:** Certificates are stored in Key Vault (replicated to DR). Ingress controllers fetch certs via CSI driver. Same certs work in both regions.

### Cost Questions

**Q: How much does this DR setup cost?**

**A:** Approximately $1,200/month for the reference architecture:
- Primary: ~$800/month
- DR: ~$400/month
- Can be optimized to ~$900/month with reserved instances

**Q: Can I reduce DR costs further?**

**A:** Yes, options include:
- Use spot instances for non-critical workloads
- Implement auto-shutdown for DR during off-hours
- Use backup/restore instead of warm standby (increases RTO to 2-4 hours)
- Reduce database tier in DR region

### Operational Questions

**Q: How often should we test DR?**

**A:** Recommended schedule:
- Monthly: Automated health checks
- Quarterly: Full DR drill
- Bi-annually: Chaos engineering exercises
- Annually: Complete disaster simulation

**Q: What if both regions fail?**

**A:** This is an extremely rare Azure-wide outage. Mitigation:
- Multi-cloud strategy (Azure + AWS/GCP)
- On-premises backup
- Accept the risk (99.99% availability is costly)

**Q: How do you handle database schema changes?**

**A:** Use blue-green or canary deployments:
1. Deploy backward-compatible schema to both regions
2. Deploy application to DR first (validation)
3. Deploy to primary
4. Remove old schema after validation

---

## 📚 Additional Resources

### Documentation
- [Azure AKS Best Practices](https://docs.microsoft.com/en-us/azure/aks/best-practices)
- [Kubernetes Disaster Recovery](https://kubernetes.io/docs/concepts/cluster-administration/disaster-recovery/)
- [Azure Well-Architected Framework](https://docs.microsoft.com/en-us/azure/architecture/framework/)

### Tools
- [Velero](https://velero.io/) - Kubernetes backup and restore
- [KubeDR](https://github.com/catalogicsoftware/kubedr) - DR orchestration
- [Chaos Mesh](https://chaos-mesh.org/) - Chaos engineering

### Community
- [AKS GitHub](https://github.com/Azure/AKS)
- [CNCF Slack](https://cloud-native.slack.com)
- [r/kubernetes](https://reddit.com/r/kubernetes)

---

## 🤝 Contributing

Contributions are welcome! Please:

1. Fork the repository
2. Create a feature branch
3. Commit your changes
4. Push to the branch
5. Create a Pull Request

---

## 📄 License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

---

## 👥 Authors

**Platform Engineering Team**
- Initial work and architecture design
- Contact: platform-team@company.com

---

## 🙏 Acknowledgments

- Azure AKS Team for platform capabilities
- CNCF community for Kubernetes ecosystem
- HashiCorp for Terraform
- All contributors and reviewers

---

<p align="center">
  <strong>Built with ❤️ for resilient cloud-native applications</strong>
</p>

<p align="center">
  <a href="#-overview">Back to Top ↑</a>
</p>
