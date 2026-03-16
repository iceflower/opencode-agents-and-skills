---
name: terraform-workflow
description: >-
  Terraform core workflow rules, state management, module design patterns,
  and version considerations. Use for general Terraform operations and best practices.
---
# Terraform Workflow Rules

## 1. Terraform Code Analysis Rules

### Code-First Analysis Principle

- ALWAYS read Terraform code before making assumptions about infrastructure state
- NEVER assume missing resources need manual creation without verifying Terraform code
- Start analysis with `modules/` and `environments/` directory structure
- When encountering unclear or ambiguous aspects of Terraform or cloud provider resources, consult official documentation to verify behavior, command syntax, and resource attributes

### Resource Creation vs Reference

- `resource` blocks CREATE new resources (Terraform manages these)
- `data` blocks REFERENCE existing resources (must exist before Terraform run)
- Proxy subnets, firewall rules, and compute resources are typically CREATED by Terraform modules

### Declarative Nature Understanding

- Terraform is DECLARATIVE: it defines WHAT should exist, not HOW to create it
- Missing resources in target environment likely means Terraform hasn't been run yet, not that they need manual creation
- Pre-creating resources that Terraform will create causes resource conflicts and errors

### Validation Workflow

1. Read module code to understand what resources are created
2. Check variable flow from environments to modules
3. Verify if resources are created (`resource`) or referenced (`data`)
4. Only then assess current infrastructure state with CLI tools
5. Never skip step 1-3 even if infrastructure state seems obvious

### Common Anti-Patterns

- ❌ "I see missing subnet, so I must create it manually first"
- ❌ "Firewall rules don't exist, so they're prerequisites"
- ✅ "Let me check if the module creates these resources automatically"
- ✅ "What does the Terraform code actually do?"

### Variable Wiring Verification

- When adding a new variable to an environment's `variables.tf`, always verify the complete wiring chain:
  1. `terraform.tfvars` (actual value) → `variables.tf` (variable declaration) → `main.tf` (module call parameter) → `modules/*/variables.tf` (module variable)
  2. Missing any link in this chain causes the variable to silently use its default value instead of the intended value
- A variable defined in `variables.tf` but NOT passed in the `module {}` block of `main.tf` will be ignored — the module will use its own default
- After adding or modifying variables, run `terraform plan` to confirm the expected values are applied
- When wiring a variable across multiple environments (dev, stage, prod), verify all environments individually — do not assume one environment's wiring is replicated in others

---

## 2. Plan Output Interpretation

### Symbol Reference

| Symbol | Meaning | Risk Level |
| --- | --- | --- |
| `+` | Resource will be created | Low |
| `~` | Resource will be updated in-place | Medium |
| `-` | Resource will be destroyed | High |
| `-/+` | Resource will be destroyed and recreated | High |
| `<=` | Data source will be read | Low |

### Review Guidelines

- Always review the full plan output before applying
- Pay special attention to `-` and `-/+` changes — these indicate potential downtime or data loss
- When `-/+` appears, check if the triggering attribute change is intentional (e.g., `name` change forces replacement in many resources)
- Verify the total count of changes matches expectations: `Plan: X to add, Y to change, Z to destroy`

---

## 3. Resource Design Best Practices

### `count` vs `for_each` Selection

- Prefer `for_each` over `count` for resource creation
- `count` identifies resources by index — adding/removing items shifts all subsequent indices, causing unnecessary destroy/recreate
- `for_each` identifies resources by key — adding/removing items only affects the specific resource
- Use `count` only for simple conditional creation (`count = var.enabled ? 1 : 0`)

### `depends_on` Usage

- Prefer implicit dependencies (resource references) over explicit `depends_on`
- `depends_on` should be a last resort — it creates a hard dependency that forces serial execution
- Common valid use case: when a resource depends on a side effect (e.g., IAM policy must exist before a resource can use the role, but there is no direct attribute reference)

### `lifecycle` Block

- `prevent_destroy`: Use for critical resources (databases, storage) that should never be accidentally deleted
- `create_before_destroy`: Use when replacement must avoid downtime (e.g., SSL certificates, load balancer backends)
- `ignore_changes`: Use for attributes managed outside Terraform (e.g., auto-scaling group size, tags managed by external systems)
- Avoid overusing `ignore_changes` — it hides drift and can mask real configuration issues

---

## 4. Module Change Impact Rules

### Scope Awareness

- Module changes (`modules/`) affect ALL environments that reference the module
- Before modifying a module, verify impact across all environments
- Environment-specific changes should go in `environments/{env}/` only

### Verification Checklist

1. Identify which environments use the modified module
2. Check if variable defaults or required inputs changed
3. Run `terraform plan` in each affected environment
4. Review plan output for unintended resource changes (especially destroy/recreate)

---

## 5. Terraform Workflow Rules

### Code Quality

- Run `terraform fmt` after every code change
- Run `terraform validate` before committing

### State Management

- State files (`.tfstate`, `.tfstate.backup`, `.tfstate.*.backup`) must NEVER be committed to version control
- When resources are created outside Terraform or need to be removed from state, use `terraform state rm` and `terraform import` commands
- Always run `terraform init` when switching between service directories or after modifying provider/backend configurations
- For team collaboration, use remote backend (e.g., GCS, S3, Terraform Cloud) with state locking enabled to prevent concurrent modifications

### Apply Failure Recovery

- When `terraform apply` fails mid-way (e.g., resource already exists, timeout, quota exceeded):
  1. Identify which resources were successfully created and which failed
  2. For resources that exist in the cloud but not in Terraform state: use `terraform import` to bring them under management
  3. For resources in Terraform state that were not actually created: use `terraform state rm` to remove the stale reference
  4. For orphaned resources (created but not needed): delete via provider CLI, then clean state with `terraform state rm`
- After recovery, always run `terraform plan` to verify the state is consistent before attempting `terraform apply` again
- Common failure pattern: resource already exists error (e.g., GCP `409 alreadyExists`, AWS `AlreadyExistsException`) — resource was previously created manually or by a prior failed apply. Resolve by importing the existing resource or deleting it and re-applying

### Production Safety Protocol

- ALWAYS verify changes in dev/stage environments before applying to prod
- NEVER assume prod configuration is identical to dev/stage — verify variables and values explicitly
- Use `terraform plan` with detailed output review before any prod apply
- Enable deletion protection for critical prod resources (databases, storage, networking)
- Maintain separate state files and backend configurations for each environment
- For prod environments, add explicit confirmation steps before destructive operations

---

## 6. Deprecated Commands and Modern Alternatives

### `terraform taint` → `terraform apply -replace`

- `terraform taint` is **deprecated** since v0.15.2
- Use `terraform apply -replace="<resource_address>"` instead
- `-replace` is safer because it shows the full plan before execution, whereas `taint` creates a race condition where other team members could generate plans against the tainted resource before review
- ❌ `terraform taint aws_instance.example`
- ✅ `terraform apply -replace="aws_instance.example"`

### `terraform state mv` → `moved` block

- For resource refactoring (renaming, moving into/out of modules), prefer the `moved` block (v1.1+) over `terraform state mv`
- `moved` block is tracked in version control and applied automatically during `terraform apply`
- `terraform state mv` is a one-off CLI operation with no audit trail in code

```hcl
moved {
  from = aws_instance.old_name
  to   = aws_instance.new_name
}
```hcl

### `terraform import` CLI → `import` block

- For importing existing resources, prefer the `import` block (v1.5+) over `terraform import` CLI
- `import` block integrates into the standard `terraform apply` workflow and is tracked in code
- Use `terraform plan -generate-config-out=generated.tf` to auto-generate the corresponding `resource` block for the imported resource
- `terraform import` CLI is a one-off operation that modifies state directly without plan review

```hcl
import {
  to = aws_instance.example
  id = "i-1234567890abcdef0"
}
```hcl

### Provisioners (`provisioner` block)

- Provisioners (`local-exec`, `remote-exec`, `file`) are **discouraged** by HashiCorp
- Use cloud-init, Packer, or configuration management tools (Ansible, etc.) instead
- If unavoidable, treat provisioners as a last resort and document the reason

---

## 7. Version Support and Features (2026-03-14)

### Current Versions

| Version | Status | Latest Patch | Notes |
| --- | --- | --- | --- |
| 1.14 | Supported | 1.14.7 | Current stable |
| 1.13 | Supported | 1.13.5 | |
| 1.12 | EOL | 1.12.2 | Ended 2025-11-19 |
| 1.11 | EOL | 1.11.4 | Ended 2025-08-20 |
| 1.10 | EOL | 1.10.5 | Ended 2025-05-14 |

### Terraform 1.15 (Upcoming)

Terraform 1.15 (currently alpha) introduces significant new features:

#### New Features

- **Windows ARM64 support**: Native builds for Windows on ARM
- **`deprecated` attribute**: Mark variables and outputs as deprecated

  ```hcl
  variable "old_variable" {
    type        = string
    deprecated  = "Use new_variable instead"
  }

  output "legacy_output" {
    value       = module.example.result
    deprecated  = "This output will be removed in v2.0"
  }
  ```

- **`convert` function**: Precise inline type conversions

  ```hcl
  variable "config" {
    type = convert(var.raw_config, object({
      name = string
      tags = list(string)
    }))
  }
  ```

- **Variables in module source**: Dynamic module sources

  ```hcl
  module "app" {
    source  = "${var.module_registry}/${var.module_name}"
    version = var.module_version
  }
  ```

- **Backend validation**: `terraform validate` now checks backend blocks
- **S3 backend**: Support for `aws login` authentication

#### Migration Planning

- Review deprecation warnings in current code
- Plan variable/output migrations before 1.15 upgrade
- Test `convert` function for complex type scenarios

---

## 8. Provider Version Considerations

### AWS Provider (v6.x)

- Latest: v6.36.0 (2026-03-11)
- Requires Terraform >= 1.0
- Major version upgrades require explicit `version = "~> 6.0"` in `required_providers`

### Azure Provider (azurerm v4.x)

- Latest: v4.x series
- Breaking changes from v3.x: resource renames, property changes
- Review [upgrade guide](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs/guides/4.0-upgrade-guide) before major version upgrade

### GCP Provider (google v6.x)

- Latest: v6.x series
- Beta provider (`google-beta`) tracks main provider versioning

### Provider Version Pinning

```hcl
terraform {
  required_version = "~> 1.14"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 6.0"
    }
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 4.0"
    }
  }
}
```hcl

---

## 9. Testing and Validation

### Pre-Commit Hooks

Recommended `.pre-commit-config.yaml` for Terraform:

```yaml
repos:
  - repo: https://github.com/antonbabenko/pre-commit-terraform
    rev: v1.96.0
    hooks:
      - id: terraform_fmt
      - id: terraform_validate
      - id: terraform_tflint
      - id: terraform_tfsec
```hcl

### Validation Commands

```bash
# Format check
terraform fmt -check -recursive

# Validate configuration
terraform validate

# Security scan (tfsec)
tfsec .

# Lint (tflint)
tflint --init && tflint
```hcl

### CI/CD Pipeline Stages

1. `terraform fmt -check` — Code style
2. `terraform validate` — Syntax and schema
3. `tflint` — Best practices
4. `tfsec` — Security analysis
5. `terraform plan` — Preview changes
6. Manual approval (for prod)
7. `terraform apply` — Apply changes

---

## 10. Provider-Specific Skills

For provider-specific issues and best practices, see dedicated skills:

- **AWS Provider**: `terraform-aws-provider` skill
- **Azure Provider**: `terraform-azure-provider` skill
- **GCP Provider**: `terraform-gcp-provider` skill

