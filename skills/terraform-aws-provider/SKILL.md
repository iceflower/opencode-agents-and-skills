---
name: terraform-aws-provider
description: >-
  AWS Terraform provider v6.x issues, breaking changes, best practices,
  and troubleshooting. Use when working with AWS Terraform resources.
---

# AWS Terraform Provider v6.x Guide

## 1. Breaking Changes from v5.x

### 1.1 Removed Resources

**OpsWorks Resources (All 16 resources removed)**

AWS OpsWorks service was discontinued, and all related Terraform resources were removed in v6.0:

- `aws_opsworks_stack`, `aws_opsworks_instance`, `aws_opsworks_layer`
- `aws_opsworks_application`, `aws_opsworks_custom_layer`
- `aws_opsworks_ganglia_layer`, `aws_opsworks_haproxy_layer`
- `aws_opsworks_java_app_layer`, `aws_opsworks_memcached_layer`
- `aws_opsworks_mysql_layer`, `aws_opsworks_nodejs_app_layer`
- `aws_opsworks_permission`, `aws_opsworks_php_app_layer`
- `aws_opsworks_rails_app_layer`, `aws_opsworks_rds_db_instance`
- `aws_opsworks_static_web_layer`

**Migration Action**: Remove these resources from configuration before upgrading. If resources still exist in state, use `terraform state rm` to remove them.

**Other Deprecated Resources**:

- `aws_simpledb_*` resources (SimpleDB discontinued)
- `aws_worklink_*` resources (WorkLink discontinued)
- `aws_evidently_*` resources (CloudWatch Evidently ending 10/2025)

### 1.2 Boolean Strictness Changes

v6.0 enforces strict boolean types. String representations like `"1"`, `"0"`, `"true"`, `"false"` are no longer accepted:

```hcl
# ❌ v5.x (worked but incorrect)
resource "aws_launch_template" "example" {
  block_device_mappings {
    ebs {
      delete_on_termination = "1"  # Rejected in v6
      encrypted             = "false"
    }
  }
}

# ✅ v6.x (correct)
resource "aws_launch_template" "example" {
  block_device_mappings {
    ebs {
      delete_on_termination = true
      encrypted             = false
    }
  }
}
```

**Common affected attributes**:

- `delete_on_termination`, `encrypted`, `ebs_optimized`
- `monitoring`, `associate_public_ip_address`

### 1.3 Default Value Changes

**Security-Related Defaults**:

- `aws_redshift_cluster.publicly_accessible`: Now defaults to `false` (was `true`)
- `aws_redshift_cluster.encrypted`: Now defaults to `true` (was `false`)

### 1.4 Parameter Changes

**Elastic IP (aws_eip)**:

```hcl
# ❌ v5.x
resource "aws_eip" "example" {
  vpc = true
}

# ✅ v6.x
resource "aws_eip" "example" {
  domain = "vpc"
}
```

### 1.5 New Features in v6.x

**Per-Resource Region Override**: v6.0 introduces the ability to specify region at the resource level:

```hcl
provider "aws" {
  region = "us-east-1"
}

# Override region inline (NEW in v6.x)
resource "aws_s3_bucket" "backup" {
  bucket = "my-backup-bucket"
  region = "eu-west-1"
}
```

---

## 2. Common Provider Issues

### 2.1 State Locking Issues

**Problem**: `ConditionalCheckFailedException` when using DynamoDB for state locking

**Root Causes**:

1. Previous Terraform process crashed without releasing lock
2. Concurrent Terraform runs (CI/CD pipeline race conditions)
3. DynamoDB table misconfiguration
4. Network timeout during lock release

**Solutions**:

```bash
# 1. Verify no active Terraform process
ps aux | grep terraform

# 2. Force unlock (use with caution)
terraform force-unlock f8e5b9a3-7c42-11ed-a1eb-0242ac120002

# 3. Manual DynamoDB cleanup
aws dynamodb delete-item \
  --table-name terraform-state-locks \
  --key '{"LockID": {"S": "my-terraform-state/prod/terraform.tfstate"}}'
```

**Prevention - S3 Native Locking (Terraform v1.11+)**:

```hcl
terraform {
  backend "s3" {
    bucket       = "my-terraform-state"
    key          = "prod/terraform.tfstate"
    region       = "us-east-1"
    encrypt      = true
    use_lockfile = true  # Native S3 locking - no DynamoDB needed
  }
}
```

### 2.2 Eventual Consistency Issues

AWS APIs have eventual consistency, causing "resource not found" errors immediately after creation.

**Solution**: Provider includes automatic retries, but explicit `depends_on` can help:

```hcl
resource "aws_iam_role_policy_attachment" "example" {
  depends_on = [aws_iam_role.example]
  role       = aws_iam_role.example.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess"
}
```

**Known Resources with Eventual Consistency Protection**:

- `aws_ses_receipt_rule` (v5.84+)
- `aws_sqs_queue_policy` (v5.38+)
- `aws_s3_bucket_lifecycle_configuration` (v5.95+)
- `aws_iot_policy` (v5.26+)
- `aws_route_table_association` (v5.0+)

### 2.3 API Quota and Throttling Issues

**Error Pattern**:

```text
Error: throttlingException: Rate exceeded
Error: RequestLimitExceeded: Request limit exceeded
```

**Solutions**:

```hcl
# Provider-Level Retry Configuration
provider "aws" {
  region      = "us-east-1"
  max_retries = 25
}
```

```bash
# Environment Variables
export AWS_MAX_ATTEMPTS=25
export AWS_RETRY_MODE=adaptive

# Parallelism Control
terraform apply -parallelism=5  # Default is 10
```

---

## 3. Provider Configuration Best Practices

### 3.1 Assume Role Configuration

**Multi-Account Setup**:

```hcl
provider "aws" {
  region = "us-east-1"

  assume_role {
    role_arn         = "arn:aws:iam::123456789012:role/terraform-execution"
    session_name     = "terraform-session"
    duration_seconds = 3600
    external_id      = "unique-external-id"
  }

  default_tags {
    tags = {
      ManagedBy   = "Terraform"
      Environment = var.environment
    }
  }
}
```

**Cross-Account with OIDC (GitHub Actions)**:

```hcl
provider "aws" {
  region = "us-east-1"

  assume_role_with_web_identity {
    role_arn           = var.role_arn
    session_name       = "github-actions"
    web_identity_token = file("/var/run/secrets/tokens/actions-token")
  }
}
```

### 3.2 Region Handling

**Multi-Region (v6.x+)**:

```hcl
provider "aws" {
  region = "us-east-1"
}

# Override region inline (v6.x feature)
resource "aws_s3_bucket" "west" {
  bucket = "west-bucket"
  region = "us-west-2"
}
```

---

## 4. Known Issues with Specific Resources

### 4.1 RDS Resources

**Issue 1: `aws_rds_cluster` IOPS Drift (v6.3+)**

- **Symptom**: After provisioning RDS cluster with `iops = null`, subsequent plans show drift
- **Affected**: Non-Aurora RDS clusters with `storage_type = "gp3"` and allocated storage < 400GB
- **Workaround**: Explicitly set `iops = 3000` (default) instead of `null`

```hcl
resource "aws_rds_cluster" "example" {
  cluster_identifier = "my-cluster"
  engine             = "postgres"
  storage_type       = "gp3"
  allocated_storage  = 250
  iops               = 3000  # Explicit default
}
```

**Issue 2: Resource Identity Mismatch**

- **Symptom**: `Error: Unexpected Identity Change` during read operation
- **Resolution**:
  1. Fix duplicate rules in configuration
  2. `terraform state rm aws_security_group.example`
  3. `terraform import aws_security_group.example sg-12345678`

### 4.2 EKS Resources

**Issue: EKS v1.30+ IMDSv2 Credential Retrieval**

- **Symptom**: `no EC2 IMDS role found`
- **Root Cause**: EKS v1.30+ requires IMDSv2
- **Resolution**: Update Terraform Enterprise to v202601-1 or configure IMDSv2

```hcl
resource "aws_launch_template" "eks_nodes" {
  metadata_options {
    http_endpoint               = "enabled"
    http_tokens                 = "required"  # IMDSv2 required
    http_put_response_hop_limit = 2
  }
}
```

### 4.3 S3 Resources

**Issue: Bucket ACL Deprecation**

```hcl
# ❌ Deprecated
resource "aws_s3_bucket" "example" {
  bucket = "my-bucket"
  acl    = "private"
}

# ✅ Current approach
resource "aws_s3_bucket" "example" {
  bucket = "my-bucket"
}

resource "aws_s3_bucket_acl" "example" {
  bucket = aws_s3_bucket.example.id
  acl    = "private"
}
```

---

## 5. Pre-Migration Checklist for v6.x

```bash
# 1. Find deprecated resources
grep -r "aws_opsworks_\|aws_simpledb_\|aws_worklink_\|aws_evidently_" . --include="*.tf"

# 2. Find lazy boolean usage
grep -r '"[01]"' . --include="*.tf" | grep -E "delete_on_termination|encrypted|ebs_optimized"

# 3. Backup state
terraform state pull > backup.tfstate.backup

# 4. Run TFLint
tflint --enable-rule=terraform_deprecated_lookup
```

---

## 6. Migration Strategy

**Recommended Approach**: Module-by-Module Migration

1. Development utilities (lowest risk)
2. Shared networking (VPCs, subnets)
3. Data resources (S3, RDS)
4. Compute resources (EC2, ECS)
5. Production critical (only after thorough testing)

**Version Pinning**:

```hcl
# Development
version = "~> 6.0"

# Production
version = "= 6.36.0"
```

---

## 7. Rollback Procedures

```bash
# Quick Rollback
rm -rf .terraform/
rm .terraform.lock.hcl
terraform init

# State Surgery
terraform state rm aws_instance.borked
terraform import aws_instance.borked i-1234567890abcdef0
```

---

## 8. Version Reference

| Version | Release Date | Key Changes |
| --- | --- | --- |
| 6.0.0 | 2025-04 | OpsWorks removal, boolean strictness, EIP domain |
| 6.36.0 | 2026-03-11 | Latest stable |

**Official Documentation**:

- [Terraform Registry - AWS Provider](https://registry.terraform.io/providers/hashicorp/aws/latest)
- [AWS Prescriptive Guidance](https://docs.aws.amazon.com/prescriptive-guidance/latest/terraform-aws-provider-best-practices/)
- [GitHub Issues](https://github.com/hashicorp/terraform-provider-aws/issues)
