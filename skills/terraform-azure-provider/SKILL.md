---
name: terraform-azure-provider
description: >-
  Azure Terraform provider (azurerm) v4.x issues, breaking changes,
  best practices, and troubleshooting. Use when working with Azure Terraform resources.
---

# Azure Terraform Provider (azurerm) v4.x Guide

## 1. Breaking Changes from v3.x to v4.x

### 1.1 Subscription ID Now Required

**Issue**: `subscription_id` is now a **required** provider property when using Azure CLI authentication.

**Error Message**:

```text
Error: `subscription_id` is a required provider property when performing a plan/apply operation
```

**Solution**: Explicitly specify subscription ID in provider block or environment variable:

```hcl
provider "azurerm" {
  features {}
  subscription_id = "00000000-0000-0000-0000-000000000000"
}
```

Or via environment variable:

```bash
export ARM_SUBSCRIPTION_ID="00000000-0000-0000-0000-000000000000"
```

### 1.2 Resource Provider Registration Changes

**Issue**: `skip_provider_registration` argument removed.

**Solution**: Use new granular control arguments:

```hcl
provider "azurerm" {
  features {}

  # Options: "core", "extended", "all", "none", "legacy"
  resource_provider_registrations = "core"

  resource_providers_to_register = [
    "Microsoft.ContainerService",
    "Microsoft.KeyVault"
  ]
}
```

**Migration**:

- `skip_provider_registration = true` → `resource_provider_registrations = "none"`
- Default behavior changed from registering all RPs to registering only core RPs

### 1.3 Removed Deprecated Resources

**Major Resource Replacements**:

| Old Resource (v3.x) | New Resource (v4.x) | Notes |
| --- | --- | --- |
| `azurerm_app_service` | `azurerm_windows_web_app` / `azurerm_linux_web_app` | OS-specific split |
| `azurerm_app_service_plan` | `azurerm_service_plan` | Requires `os_type` and `sku_name` |
| `azurerm_function_app` | `azurerm_windows_function_app` / `azurerm_linux_function_app` | OS-specific split |
| `azurerm_sql_database` | `azurerm_mssql_database` | SQL Server rename |
| `azurerm_sql_server` | `azurerm_mssql_server` | SQL Server rename |
| `azurerm_mysql_server` | `azurerm_mysql_flexible_server` | Flexible server only |
| `azurerm_mariadb_server` | `azurerm_mariadb_flexible_server` | Flexible server only |

**Migration Strategy** (for deprecated resources):

```hcl
# Step 1: Remove from state (without destroying)
removed {
  from = azurerm_app_service_plan.main
  lifecycle {
    destroy = false
  }
}

# Step 2: Add new resource
resource "azurerm_service_plan" "main" {
  name                = "asp-example"
  resource_group_name = azurerm_resource_group.example.name
  location            = azurerm_resource_group.example.location
  os_type             = "Windows"
  sku_name            = "S1"
}

# Step 3: Import
# terraform import azurerm_service_plan.main /subscriptions/.../serverFarms/asp-example
```

### 1.4 AKS (Kubernetes Cluster) Changes

**Breaking Changes**:

1. **Preview API removed**: Only stable AKS API supported
   - Preview features must use `azapi` provider instead

2. **Removed properties**:
   - `default_node_pool.node_taints` - removed
   - `default_node_pool.type` - "AvailabilitySet" option removed
   - Various `enable_*` properties renamed to `*_enabled` pattern

**Solution for Preview Features**:

```hcl
# Use azapi provider for preview AKS features
resource "azapi_resource" "aks_preview" {
  type      = "Microsoft.ContainerService/managedClusters@2023-10-01-preview"
  name      = "aks-cluster"
  location  = azurerm_resource_group.example.location
  parent_id = azurerm_resource_group.example.id

  body = jsonencode({
    properties = {
      # Preview features here
    }
  })
}
```

### 1.5 Storage Account Changes

**Breaking Changes**:

- `storage_account_name` → `storage_account_id` for `azurerm_storage_container` and `azurerm_storage_share`

**Solution**: Use migration support added in v4.20.0+

```hcl
resource "azurerm_storage_container" "example" {
  name                  = "example"
  storage_account_id    = azurerm_storage_account.example.id  # Use ID not name
  container_access_type = "private"
}
```

### 1.6 Virtual Network and Subnet Changes

**Breaking Changes**:

- `address_space` type change: `list(string)` → `set(string)`
- Subnet `address_prefixes` type change: `list(string)` → `set(string)`

**Impact**: Cannot reference by index, must use `toset()` function

```hcl
# Before (v3.x)
resource "azurerm_virtual_network" "example" {
  address_space = ["10.0.0.0/16", "10.1.0.0/16"]
}

# After (v4.x)
resource "azurerm_virtual_network" "example" {
  address_space = toset(["10.0.0.0/16", "10.1.0.0/16"])
}
```

---

## 2. Provider-Defined Functions (New in v4.x)

### 2.1 `normalise_resource_id()`

Normalizes resource ID casing:

```hcl
locals {
  normalized_id = provider::azurerm::normalise_resource_id(
    "/Subscriptions/12345678-.../ResourceGroups/resGroup1/PROVIDERS/..."
  )
  # Result: /subscriptions/12345678-.../resourceGroups/resGroup1/providers/...
}
```

### 2.2 `parse_resource_id()`

Parses resource ID into components:

```hcl
locals {
  parsed = provider::azurerm::parse_resource_id(
    "/subscriptions/12345678-.../resourceGroups/resGroup1/providers/..."
  )
}

output "resource_group" {
  value = local.parsed["resource_group_name"]  # "resGroup1"
}
```

---

## 3. Common Issues and Solutions

### 3.1 Multi-Subscription Deployments

**Problem**: v4.x requires explicit subscription ID per provider alias.

**Solution**:

```hcl
provider "azurerm" {
  features {}
  subscription_id = var.subscription_id_core
  alias           = "core"
}

provider "azurerm" {
  features {}
  subscription_id = var.subscription_id_connectivity
  alias           = "connectivity"
}

resource "azurerm_resource_group" "core" {
  provider = azurerm.core
  name     = "core-rg"
  location = "eastus"
}
```

### 3.2 Resource Group Deletion Prevention

**Problem**: Accidental deletion of resource groups with resources.

**Solution**: Use `features` block:

```hcl
provider "azurerm" {
  features {
    resource_group {
      prevent_deletion_if_contains_resources = true
    }
    key_vault {
      purge_soft_delete_on_destroy    = false
      recover_soft_deleted_key_vaults = true
    }
  }
}
```

### 3.3 Managed Identity Authentication

```hcl
provider "azurerm" {
  features {}
  use_msi          = true
  subscription_id  = "00000000-0000-0000-0000-000000000000"
}
```

**For AKS Workload Identity**:

```hcl
provider "azurerm" {
  features {}
  use_aks_workload_identity = true
  use_cli                   = false
  tenant_id                 = var.tenant_id
  subscription_id           = var.subscription_id
}
```

### 3.4 Storage Account Cross-Subscription Access

```hcl
provider "azurerm" {
  features {}
  alias           = "source"
  subscription_id = var.source_subscription_id
}

data "azurerm_storage_account" "source" {
  provider            = azurerm.source
  name                = "sourcestorage"
  resource_group_name = "source-rg"
}
```

---

## 4. Best Practices

### 4.1 Provider Configuration Template

```hcl
terraform {
  required_version = "~> 1.14"

  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 4.0"
    }
  }
}

provider "azurerm" {
  features {
    resource_group {
      prevent_deletion_if_contains_resources = true
    }
    key_vault {
      purge_soft_delete_on_destroy    = false
      recover_soft_deleted_key_vaults = true
    }
  }

  subscription_id = var.subscription_id
  tenant_id       = var.tenant_id

  resource_provider_registrations = "core"
  resource_providers_to_register = [
    "Microsoft.ContainerService",
    "Microsoft.KeyVault",
    "Microsoft.Storage"
  ]
}
```

### 4.2 Upgrade Path Recommendation

1. **Pin current version**: `version = "= 3.116.0"` (last v3.x bridge version)
2. **Upgrade to v3.116.0** and fix deprecation warnings
3. **Migrate deprecated resources** using remove/import pattern
4. **Upgrade to v4.x**: `version = "~> 4.0"`
5. **Fix breaking changes**:
   - Add `subscription_id` to provider block
   - Update resource provider registration
   - Fix type changes (list → set)
   - Update renamed properties

---

## 5. Known Issues (2025-2026)

1. **AKS node pool compatibility**: v3.x and v4.x state files not compatible
   - Issue: [#27209](https://github.com/hashicorp/terraform-provider-azurerm/issues/27209)
   - Workaround: Complete upgrade in single apply, no rollback

2. **Storage container migration**: Forces replacement in v4.0-v4.19
   - Issue: [#27942](https://github.com/hashicorp/terraform-provider-azurerm/issues/27942)
   - Fixed: v4.20.0+ with migration support

3. **Virtual network perpetual diff**: With `ip_address_pool` feature
   - Fixed: v4.37.0

---

## 6. Quick Reference: Property Renames

| Old Property (v3.x) | New Property (v4.x) | Resource |
| --- | --- | --- |
| `enable_auto_scaling` | `auto_scaling_enabled` | `azurerm_kubernetes_cluster` |
| `enable_host_encryption` | `host_encryption_enabled` | `azurerm_kubernetes_cluster` |
| `enable_node_public_ip` | `node_public_ip_enabled` | `azurerm_kubernetes_cluster_node_pool` |
| `storage_account_name` | `storage_account_id` | `azurerm_storage_container` |
| `sku` (block) | `sku_name` (string) | `azurerm_service_plan` |
| `kind` | `os_type` | `azurerm_service_plan` |

**Pattern**: Boolean properties renamed from `enable_*` to `*_enabled` for consistency.

---

## 7. Version Reference

| Version | Release Date | Key Changes |
| --- | --- | --- |
| 4.0.0 | 2024-08 | Subscription ID required, resource renames, provider functions |
| 4.20.0+ | 2025 | Storage container migration support |
| 4.37.0+ | 2025 | Virtual network perpetual diff fix |

**Official Documentation**:

- [Terraform Registry - Azure Provider](https://registry.terraform.io/providers/hashicorp/azurerm/latest)
- [4.0 Upgrade Guide](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/guides/4.0-upgrade-guide)
- [GitHub Issues](https://github.com/hashicorp/terraform-provider-azurerm/issues)
