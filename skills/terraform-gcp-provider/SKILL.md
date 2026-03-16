---
name: terraform-gcp-provider
description: >-
  Google Cloud Platform Terraform provider v6.x issues, breaking changes,
  best practices, and troubleshooting. Use when working with GCP Terraform resources.
---

# GCP Terraform Provider v6.x Guide

## 1. Breaking Changes from v5.x

### 1.1 Default Label Behavior Change

**Breaking Change**: The `goog-terraform-provisioned` label changed from opt-in (v5.16+) to **opt-out** in v6.0.

**Impact**: All newly created resources with `labels` fields automatically receive this label.

**Solution**: To opt-out, explicitly disable in provider configuration:

```hcl
provider "google" {
  add_terraform_attribution_label = false
}
```

**Evidence** ([HashiCorp Blog](https://www.hashicorp.com/en/blog/terraform-provider-for-google-cloud-6-0-is-now-ga)):

> "In version 6.0, this attribution label is now enabled by default, and will be added to all newly created resources that support labels."

### 1.2 Deletion Protection Defaults

**Breaking Change**: Several resources now have deletion protection enabled by default:

- `google_project` - `deletion_policy = "PREVENT"` (default)
- `google_folder` - `deletion_protection = true` (default)
- `google_cloud_run_v2_service` - `deletion_protection = true` (default)
- `google_cloud_run_v2_job` - `deletion_protection = true` (default)
- `google_domain` - `deletion_protection = true` (default)

**Solution**: Explicitly set to `false` if you need Terraform to destroy these resources:

```hcl
resource "google_project" "my_project" {
  project_id          = "my-project"
  name                = "My Project"
  deletion_policy     = "DELETE"  # Override default PREVENT
}

resource "google_folder" "my_folder" {
  display_name        = "My Folder"
  parent              = "organizations/123456789"
  deletion_protection = false  # Override default true
}
```

**Evidence** ([Google Cloud Blog](https://cloud.google.com/blog/products/management-tools/announcing-terraform-google-provider-6-0-0)):

> "In order to prevent the accidental deletion of important resources, many resources now have a form of deletion protection enabled by default."

### 1.3 Name Prefix Suffix Length

**Change**: `name_prefix` resources now allow longer user-defined prefixes (54 chars vs 37 chars) by using shorter appended suffixes.

**Impact**: Existing resources with `name_prefix` may show diffs on upgrade, but no replacement occurs.

**Evidence** ([GitHub Issue #15374](https://github.com/hashicorp/terraform-provider-google/issues/15374)):

> "The max length of the user-defined name_prefix has increased from 37 characters to 54."

---

## 2. Common Provider Issues

### 2.1 Project Handling Issues

**Issue**: Project ID vs Project Number confusion

**Symptom**: Resources fail with "Project not found" or permission errors when using project number instead of ID.

**Solution**: Always use project ID in resource definitions, use `google_project` data source to resolve:

```hcl
data "google_project" "current" {
  project_id = var.project_id  # Use ID, not number
}

resource "google_compute_instance" "vm" {
  project      = data.google_project.current.project_id
  name         = "my-vm"
  machine_type = "e2-medium"
  zone         = "us-central1-a"
}
```

### 2.2 Service Account Impersonation

**Issue**: Permission denied errors when impersonating service accounts across projects.

**Root Cause**: Missing `roles/iam.serviceAccountTokenCreator` role on the target service account.

**Required IAM Permissions**:

| Role | Granted To | Resource |
| --- | --- | --- |
| `roles/iam.serviceAccountTokenCreator` | User/SA doing impersonation | Target Service Account |
| `roles/iam.serviceAccountUser` | User/SA doing impersonation | Target Service Account |
| Required resource roles (e.g., `roles/compute.admin`) | Target Service Account | Target project/resources |

**Provider Configuration**:

```hcl
provider "google" {
  project = var.project_id

  # Service account impersonation
  impersonate_service_account = "terraform-sa@target-project.iam.gserviceaccount.com"

  # For cross-project impersonation, also specify:
  impersonate_service_account_delegates = [
    "user:${data.google_client_config.current.email}",  # End user
    "serviceAccount:ci-sa@ci-project.iam.gserviceaccount.com"  # CI service account
  ]
}

data "google_client_config" "current" {}
```

**Common Pitfall**: Impersonation uses wrong project for API enablement.

**Solution**: Enable required APIs in the **target project** (where resources are created), not the impersonating SA's project:

```hcl
resource "google_project_service" "compute" {
  project            = var.target_project_id  # Target, not CI project
  service            = "compute.googleapis.com"
  disable_on_destroy = false
}
```

### 2.3 API Enablement Timing Issues

**Issue**: Resource creation fails because required API is not yet enabled, even with `google_project_service` resource.

**Root Cause**: API enablement is asynchronous; Terraform may proceed before API is fully active.

**Solution**: Use `time_sleep` to add delay after API enablement:

```hcl
resource "google_project_service" "compute" {
  project            = var.project_id
  service            = "compute.googleapis.com"
  disable_on_destroy = false
}

# Wait for API to be fully enabled
resource "time_sleep" "wait_for_compute_api" {
  depends_on      = [google_project_service.compute]
  create_duration = "30s"
}

resource "google_compute_network" "vpc" {
  project    = var.project_id
  name       = "my-vpc"
  depends_on = [time_sleep.wait_for_compute_api]
}
```

---

## 3. Provider Configuration Best Practices

### 3.1 Multi-Project/Region Pattern

**Recommended Pattern**: Use separate provider blocks for different projects/regions with aliases.

```hcl
# Primary provider (default project)
provider "google" {
  project = var.primary_project_id
  region  = var.primary_region
}

# Secondary project provider
provider "google" {
  alias   = "secondary"
  project = var.secondary_project_id
  region  = var.secondary_region
}

# Multi-region deployment with same project
provider "google" {
  alias   = "us"
  project = var.project_id
  region  = "us-central1"
}

provider "google" {
  alias   = "eu"
  project = var.project_id
  region  = "europe-west1"
}
```

**Usage**:

```hcl
# Uses default provider
resource "google_compute_network" "primary" {
  name = "primary-vpc"
}

# Uses secondary project provider
resource "google_compute_network" "secondary" {
  provider = google.secondary
  name     = "secondary-vpc"
}

# Uses US region provider
resource "google_compute_instance" "us_vm" {
  provider     = google.us
  name         = "us-vm"
  machine_type = "e2-medium"
  zone         = "us-central1-a"
}
```

### 3.2 Regional Resource Patterns

**Best Practice**: Define region/zone at provider level, override only when necessary.

```hcl
provider "google" {
  project = var.project_id
  region  = var.region  # e.g., "us-central1"
}

# Resource inherits region from provider
resource "google_compute_network" "vpc" {
  name = "my-vpc"
}

# Override zone for specific resource
resource "google_compute_instance" "vm" {
  name         = "my-vm"
  machine_type = "e2-medium"
  zone         = "us-central1-f"  # Explicit zone override
}
```

### 3.3 Authentication Best Practices

**Local Development**: Use Application Default Credentials (ADC)

```bash
gcloud auth application-default login
```

```hcl
provider "google" {
  project = var.project_id
  region  = var.region
  # Uses ADC automatically
}
```

**CI/CD**: Use Workload Identity Federation (preferred) or service account keys (with caution).

---

## 4. Known Issues with Specific Resources

### 4.1 GKE (Google Kubernetes Engine)

**Issue**: Cluster node pool updates cause unexpected recreation.

**Symptom**: Changing `node_config` attributes triggers `-/+` (destroy/create) instead of in-place update.

**Attributes that force recreation**:

- `node_config.machine_type`
- `node_config.disk_size_gb`
- `node_config.disk_type`

**Workaround**: Use `node_pool` resource separately from cluster:

```hcl
resource "google_container_cluster" "primary" {
  name     = "my-cluster"
  location = var.region

  remove_default_node_pool = true
  initial_node_count       = 1
}

resource "google_container_node_pool" "primary_nodes" {
  name       = "primary-pool"
  location   = var.region
  cluster    = google_container_cluster.primary.name
  node_count = 3

  node_config {
    machine_type = "e2-standard-4"
    disk_size_gb = 100
  }

  lifecycle {
    create_before_destroy = true
  }
}
```

### 4.2 Cloud SQL

**Issue 1**: `google_sql_user` revokes non-Terraform managed roles.

**Symptom**: PostgreSQL roles granted outside Terraform are revoked on every apply.

**Workaround**: Manage all roles through Terraform or use `lifecycle.ignore_changes`.

**Issue 2**: Query string length validation error.

**Symptom**: `Invalid query string length: 50240. Query String length should be 256-4500.`

**Root Cause**: Provider validation doesn't match GCP API limits for Enterprise Plus instances.

**Issue 3**: Cannot recreate DB instance deleted outside Terraform.

**Solution**: Remove from state and re-import:

```bash
terraform state rm google_sql_database_instance.main
terraform apply
```

### 4.3 VPC / Compute

**Issue**: Perpetual diff on `oauth2_client_id` after disabling IAP.

**Workaround**: Use `lifecycle.ignore_changes`:

```hcl
resource "google_compute_backend_service" "backend" {
  name = "my-backend"

  iap {
    oauth2_client_id     = var.oauth_client_id
    oauth2_client_secret = var.oauth_client_secret
  }

  lifecycle {
    ignore_changes = [
      iap[0].oauth2_client_id
    ]
  }
}
```

---

## 5. Troubleshooting Checklist

1. **Check API Enablement**

   ```bash
   gcloud services list --project=PROJECT_ID --filter="state:ENABLED"
   ```

2. **Verify Service Account Permissions**

   ```bash
   gcloud projects get-iam-policy PROJECT_ID \
     --flatten="bindings[].members" \
     --format="table(bindings.role)" \
     --filter="bindings.members:serviceAccount:SA_EMAIL"
   ```

3. **Check Provider Version Compatibility**

   ```hcl
   terraform {
     required_providers {
       google = {
         source  = "hashicorp/google"
         version = "~> 6.0"
       }
     }
   }
   ```

4. **Enable Debug Logging**

   ```bash
   export TF_LOG=DEBUG
   export TF_LOG_PATH=./terraform.log
   terraform apply
   ```

5. **Check Known Issues**
   - Search [GitHub Issues](https://github.com/hashicorp/terraform-provider-google/issues)
   - Filter by service label (e.g., `service/sqladmin-cp`, `service/compute-vpc`)

---

## 6. Version Reference

| Version | Release Date | Key Changes |
| --- | --- | --- |
| 6.0.0 | 2024-08-26 | Default labels (opt-out), deletion protection, name_prefix changes |
| 6.x (latest) | 2026-03 | Continuous feature additions, bug fixes |

**Official Documentation**:

- [Terraform Registry - Google Provider](https://registry.terraform.io/providers/hashicorp/google/latest)
- [Version 6.0 Upgrade Guide](https://registry.terraform.io/providers/hashicorp/google/latest/docs/guides/version_6_upgrade)
- [Google Cloud Terraform Best Practices](https://cloud.google.com/docs/terraform/best-practices)
- [CHANGELOG](https://github.com/hashicorp/terraform-provider-google/blob/main/CHANGELOG.md)
