---
name: Terraform Primitive Module Creator
description: Agent that creates a Terraform Primitive Module from a skeleton repository to meet our standards.
---

<!-- version: 1.5 -->

# AI Agent Guide for Terraform Primitive Modules

This document provides context and instructions for AI coding assistants working with the launchbynttdata Terraform module library.

## Changelog

- **1.5** – Added notes about security-first defaults and checking files for references to skeleton and templates during cleanup.
- **1.4** – Fixed version header (block must come first to be recognized as an agent)
- **1.3** – Added agent header, migrated to agents folder, added skeleton cleanup checklist.
- **1.2** – Fixed resource naming module usage: `for_each = var.resource_names_map` (not a module input), correct variable name `class_env` (not `environment`), added required `cloud_resource_type`/`maximum_length` params, corrected output reference syntax to `module.resource_names["key"].format`, noted hyphens-stripping for AWS regions
- **1.1** – Added cloud provider API verification patterns (Azure, AWS, GCP) to Terratest guidance; tests must now verify real resource state via provider SDKs, not just Terraform outputs
- **1.0** – Initial release

> **For agents working in the skeleton repo (`lcaf-skeleton-terraform`):** If you modify this file, update the `<!-- version -->` comment at the top and add a changelog entry here. Bump the minor version (e.g. 1.1 → 1.2) for new guidance or clarifications; bump the major version (e.g. 1.x → 2.0) for changes that would require significant rework of existing modules.

## Overview

This organization maintains 250+ Terraform modules following a strict **composition model**:
- **Primitive modules** (~90%): Wrap a single cloud resource type
  - Repository naming: `tf-<provider>-module_primitive-<resource>`
  - Example: `tf-azurerm-module_primitive-postgresql_server`, `tf-aws-module_primitive-s3_bucket`
- **Reference architecture modules** (~10%): Compose multiple primitives
  - Repository naming: `tf-<provider>-module_reference-<architecture>`
  - Example: `tf-azurerm-module_reference-postgresql_server`, `tf-aws-module_reference-lambda_function`

You are most likely helping create or modify a **primitive module**.

## Cloud Providers Supported

- **Azure** (`azurerm` provider) - Primary platform
- **AWS** (`aws` provider) - Large number of modules
- **Google Cloud** (`google` provider) - Growing library

**This guide applies to all cloud providers.** Provider-specific differences are noted where relevant.

## Module Architecture Principles

### Primitive Module Pattern
```
Single Resource → Single Module → Maximum Reusability
```

**Rules:**
- ONE resource type per primitive module
- Resource block named based on resource type (e.g., `postgres`, `redis`, `lambda`, not `this`)
- Export ALL useful resource attributes as outputs
- Comprehensive input validation where appropriate
- Working example with automated Terratest
- No business logic - pure resource wrapper

**Example:**
A primitive module for `azurerm_storage_account` contains ONLY that resource. A primitive for `aws_s3_bucket` contains ONLY that resource. A reference architecture module for "secure data lake" would compose multiple primitives (storage + networking + encryption + monitoring, etc.).

### Why This Matters for AI Assistance
- Look for similar primitive modules as templates
- Don't add multiple resources to a primitive module
- Don't assume business logic belongs in primitives
- Keep it simple, complete, and composable

## Required File Structure

Every primitive module must have this structure:
```
tf-<provider>-module_primitive-<resource>/
├── .github/workflows/      # CI/CD (optional but recommended)
├── examples/
│   └── complete/           # REQUIRED: Full working example
│       ├── main.tf
│       ├── variables.tf
│       ├── outputs.tf
│       ├── providers.tf    # Provider configuration
│       └── README.md
├── tests/                  # Note: "tests" not "test"
│   ├── complete_test.go    # REQUIRED: Terratest
│   └── fixtures/           # Test data if needed
├── main.tf                 # REQUIRED: Resource definition
├── variables.tf            # REQUIRED: Input variables
├── outputs.tf              # REQUIRED: Output values
├── versions.tf             # REQUIRED: Version constraints
├── README.md               # REQUIRED: Documentation
├── Makefile                # REQUIRED: Standard targets
├── go.mod                  # Go dependencies for Terratest
├── go.sum                  # Go dependency checksums
├── LICENSE                 # Apache 2.0
├── NOTICE                  # Copyright notice
├── CODEOWNERS              # GitHub code owners
├── .gitignore              # Standard ignores
├── .tool-versions          # asdf tool versions
├── .lcafenv                # Launch CAF environment config
└── .secrets.baseline       # Detect-secrets baseline
```

## Naming Conventions

### Repository Naming
```
tf-<provider>-module_primitive-<resource>
```
Examples:
- `tf-azurerm-module_primitive-postgresql_server`
- `tf-azurerm-module_primitive-redis_cache`
- `tf-aws-module_primitive-s3_bucket`
- `tf-aws-module_primitive-lambda_function`
- `tf-google-module_primitive-storage_bucket`

**Note:** Use underscores in "module_primitive", use hyphens between words

### Resource Naming
```hcl
# Azure example
resource "azurerm_postgresql_flexible_server" "postgres" {
  # Resource name matches the resource type
}

# AWS example
resource "aws_s3_bucket" "bucket" {
  # Resource name matches the resource type
}

# GCP example
resource "google_storage_bucket" "bucket" {
  # Resource name matches the resource type
}
```

**Pattern:** Use short, descriptive name that matches the resource type
- `azurerm_postgresql_flexible_server` → `postgres`
- `azurerm_redis_cache` → `redis`
- `aws_s3_bucket` → `bucket`
- `aws_lambda_function` → `lambda` or `function`
- `google_storage_bucket` → `bucket`

**Do NOT use "this"** - use descriptive names instead

### Variable Naming
- Use `snake_case`
- Be descriptive but concise
- Match provider argument names where possible
- Group related variables with comment headers

## Code Standards

### variables.tf

**Required elements:**
- Explicit type declarations
- Comprehensive descriptions
- Validation blocks for constrained inputs (where beneficial)
- Sensible defaults for optional inputs OR no default if required

**Template pattern:**
```hcl
variable "name" {
  description = "Name of the [resource]. Must be unique within [scope]."
  type        = string
  # No default - this is required
}

variable "resource_group_name" {  # Azure
# OR
variable "tags" {                  # AWS/GCP
  description = "[Description of what this configures]"
  type        = string # or appropriate type
  # Provider-specific required fields typically have no default
}

variable "location" {  # Azure
# OR
variable "region" {    # AWS/GCP - often computed from data source
  description = "Azure region / AWS region / GCP location where resource will be created"
  type        = string
  # May or may not have default depending on pattern
}

variable "sku_name" {              # Example optional with default
  description = "The SKU/tier for this resource"
  type        = string
  default     = "Standard"         # Sensible default for optional

  validation {
    condition     = contains(["Basic", "Standard", "Premium"], var.sku_name)
    error_message = "SKU must be Basic, Standard, or Premium."
  }
}

variable "enable_feature" {
  description = "Whether to enable [specific feature]"
  type        = bool
  default     = false              # Security-first default
}

variable "complex_config" {
  description = <<-EOT
    field1 = Description of field1
    field2 = Description of field2
  EOT
  type = object({
    field1 = string
    field2 = optional(string)
  })
  default = null                   # Optional complex objects default to null
}

variable "tags" {
  description = "Map of tags to assign to the resource"
  type        = map(string)
  default     = {}                 # Always include tags, default to empty
}
```

**Key patterns:**
- Required infrastructure inputs: no defaults (name, resource_group_name/region, location)
- Optional feature flags: default to `false` or most secure option
- Tags: always include, default to empty map `{}`
- Complex objects: use `object()` with `optional()` fields
- Multi-line descriptions: use heredoc `<<-EOT ... EOT`

**Provider-specific common variables:**

**Azure:**
```hcl
variable "resource_group_name" {
  description = "Resource group name"
  type        = string
}

variable "location" {
  description = "Azure region"
  type        = string
}
```

**AWS:**
```hcl
# Region is typically obtained from data source, not variable
# Tags are the primary grouping mechanism

variable "tags" {
  description = "Map of tags"
  type        = map(string)
  default     = {}
}
```

**GCP:**
```hcl
variable "project" {
  description = "GCP project ID"
  type        = string
}

variable "location" {  # or region, depending on resource
  description = "GCP location"
  type        = string
}
```

### main.tf

**Template pattern:**
```hcl
# Azure example
resource "azurerm_postgresql_flexible_server" "postgres" {
  name                = var.name
  resource_group_name = var.resource_group_name
  location            = var.location
  
  version     = var.postgres_version
  sku_name    = var.sku_name
  storage_mb  = var.storage_mb
  
  # Use dynamic blocks for optional nested configuration
  dynamic "authentication" {
    for_each = var.authentication != null ? [var.authentication] : []
    content {
      active_directory_auth_enabled = authentication.value.active_directory_auth_enabled
      password_auth_enabled         = authentication.value.password_auth_enabled
      tenant_id                     = authentication.value.tenant_id
    }
  }

  tags = var.tags
}

# AWS example
resource "aws_s3_bucket" "bucket" {
  bucket = var.bucket_name

  tags = var.tags
}

# Separate resources for AWS (vs nested blocks in Azure)
resource "aws_s3_bucket_versioning" "bucket" {
  count  = var.enable_versioning ? 1 : 0
  bucket = aws_s3_bucket.bucket.id

  versioning_configuration {
    status = "Enabled"
  }
}
```

**Key patterns:**
- Single resource block with descriptive name
- Direct variable mapping (no transformations)
- Dynamic blocks for optional nested blocks: `for_each = var.x != null ? [var.x] : []`
- No lifecycle blocks unless absolutely necessary
- Tags at the end
- No data sources unless essential
- **AWS-specific:** Often uses separate resources instead of nested blocks

### outputs.tf

**Your actual pattern:**
```hcl
output "id" {
  value = azurerm_postgresql_flexible_server.postgres.id
  # OR
  value = aws_s3_bucket.bucket.id
}

output "name" {
  value = azurerm_postgresql_flexible_server.postgres.name
  # OR
  value = aws_s3_bucket.bucket.bucket
}

output "arn" {  # AWS-specific
  value = aws_s3_bucket.bucket.arn
}

output "fqdn" {  # Common for databases
  value = azurerm_postgresql_flexible_server.postgres.fqdn
}

# Export important attributes individually
output "primary_endpoint" {
  value = azurerm_storage_account.storage.primary_blob_endpoint
}
```

**Key patterns:**
- Export critical attributes individually
- Simple value references, no `description` fields
- No `sensitive = true` flags (handle in calling code)
- Export enough for composition, but not necessarily everything
- No complete resource object output
- **Provider-specific outputs:** ARNs (AWS), FQDNs (Azure), etc.

### versions.tf

**Your actual patterns:**

**Azure:**
```hcl
terraform {
  required_version = "~> 1.0"

  required_providers {
    azurerm = {
      source  = "hashicorp/azurerm"
      version = "~> 3.113"
    }
  }
}
```

**AWS:**
```hcl
terraform {
  required_version = "~> 1.5"  # Note: Newer than Azure modules

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.14"
    }
  }
}
```

**Pattern:**
- Terraform version: `~> 1.0` (Azure) or `~> 1.5` (AWS) - use latest stable
- Provider version: Pin to specific minor version with `~>`
- No provider configuration block (handled in examples)

**Note:** AWS modules using `~> 1.5` is newer - Azure modules should be updated to match.

## Provider-Specific Patterns

### Azure Patterns

**Resource Groups:**
```hcl
variable "resource_group_name" {
  type = string
}

variable "location" {
  type = string
}
```

**Networking:**
- VNets, subnets, NSGs are common dependencies
- Private endpoints for secure access
- Delegated subnets for managed services

### AWS Patterns

**Region Data Source:**
```hcl
data "aws_region" "current" {}

# Use data.aws_region.current.name when needed
```

**Separate Resources:**
AWS often uses separate resources for configuration vs Azure's nested blocks:
```hcl
resource "aws_s3_bucket" "bucket" {
  bucket = var.bucket_name
}

# Separate resource for versioning
resource "aws_s3_bucket_versioning" "bucket" {
  bucket = aws_s3_bucket.bucket.id
  
  versioning_configuration {
    status = "Enabled"
  }
}

# Separate resource for encryption
resource "aws_s3_bucket_server_side_encryption_configuration" "bucket" {
  bucket = aws_s3_bucket.bucket.id
  
  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}
```

**Tags as Primary Grouping:**
AWS doesn't have resource groups like Azure, so tags are critical for organization.

### GCP Patterns

**Project Context:**
```hcl
variable "project" {
  description = "GCP project ID"
  type        = string
}
```

**Labels vs Tags:**
GCP uses `labels` instead of `tags` in most resources.

## Testing Standards

### examples/complete/

**Standard structure (all providers):**
```
examples/complete/
├── main.tf          # Module usage
├── variables.tf     # Example variables
├── outputs.tf       # Pass-through outputs
├── providers.tf     # Provider configuration
└── README.md        # Example documentation
```

**Example main.tf pattern:**
```hcl
# All providers use resource naming module
# resource_names_map is used as for_each - the module is called once per resource type.
# The map key becomes the instance key (e.g. "resource_group", "postgresql_server").
module "resource_names" {
  source   = "terraform.registry.launch.nttdata.com/module_library/resource_name/launch"
  version  = "~> 2.0"

  for_each = var.resource_names_map

  logical_product_family  = var.logical_product_family
  logical_product_service = var.logical_product_service
  class_env               = var.class_env
  instance_env            = var.instance_env
  instance_resource       = var.instance_resource
  cloud_resource_type     = each.value.name
  maximum_length          = each.value.max_length

  # Azure: pass location directly (no hyphens in Azure region names)
  region                = var.location
  use_azure_region_abbr = var.use_azure_region_abbr

  # AWS/GCP: strip hyphens from region (e.g. "us-east-1" -> "useast1")
  # region = join("", split("-", data.aws_region.current.name))
}

# Reference names by map key and desired output format:
#   module.resource_names["resource_group"].standard
#   module.resource_names["postgresql_server"].standard
#   module.resource_names["s3_bucket"].minimal_random_suffix  (AWS - globally unique)

# Azure example - create resource group
module "resource_group" {
  source  = "terraform.registry.launch.nttdata.com/module_primitive/resource_group/azurerm"
  version = "~> 1.0"

  name     = module.resource_names["resource_group"].standard
  location = var.location
  tags     = var.tags
}

# Use the primitive module
module "postgres" {  # or "bucket", "lambda", etc.
  source = "../.."

  name                = module.resource_names["postgresql_server"].standard
  resource_group_name = module.resource_group.name  # Azure
  location            = var.location                  # Azure
  # OR for AWS:
  # No resource group, tags for grouping
  
  # Pass through configuration variables
  sku_name         = var.sku_name
  postgres_version = var.postgres_version
  
  tags = var.tags
}
```

**Key points:**
- Use resource naming module for consistent naming
- Create required dependencies (resource group for Azure)
- Use `source = "../.."` to reference parent module
- Pass through variables, minimal hardcoding

### Terratest (tests/)

Tests must verify **both** Terraform outputs **and** actual resource state via the cloud provider API. Terraform outputs are generated by Terraform itself and do not prove the cloud resource was actually created or configured correctly. Always use provider SDK helpers to confirm real resource state.

**Azure pattern:**
```go
package tests

import (
    "os"
    "testing"

    "github.com/gruntwork-io/terratest/modules/azure"
    "github.com/gruntwork-io/terratest/modules/terraform"
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"
)

func TestResourceComplete(t *testing.T) {
    t.Parallel()

    subscriptionID := os.Getenv("ARM_SUBSCRIPTION_ID")
    require.NotEmpty(t, subscriptionID, "ARM_SUBSCRIPTION_ID must be set")

    terraformOptions := &terraform.Options{
        TerraformDir: "../examples/complete",
    }

    defer terraform.Destroy(t, terraformOptions)
    terraform.InitAndApply(t, terraformOptions)

    // Verify Terraform outputs
    id := terraform.Output(t, terraformOptions, "id")
    assert.NotEmpty(t, id)

    name := terraform.Output(t, terraformOptions, "name")
    assert.NotEmpty(t, name)

    resourceGroupName := terraform.Output(t, terraformOptions, "resource_group_name")
    assert.NotEmpty(t, resourceGroupName)

    // Verify via Azure API - confirm the resource actually exists and is configured correctly
    // Use terratest azure helpers where available:
    //   github.com/gruntwork-io/terratest/modules/azure
    // Fall back to the Azure SDK for resources not covered by terratest helpers:
    //   github.com/Azure/azure-sdk-for-go/sdk/resourcemanager/...

    // Example for a storage account:
    storageAccount := azure.GetStorageAccount(t, resourceGroupName, name, subscriptionID)
    require.NotNil(t, storageAccount)
    assert.Equal(t, "Standard_LRS", string(storageAccount.SKU.Name))
    assert.Equal(t, "eastus", *storageAccount.Location)

    // Example using the Azure SDK directly for a resource not in terratest helpers:
    // cred, _ := azidentity.NewDefaultAzureCredential(nil)
    // client, _ := armpostgresqlflexibleservers.NewServersClient(subscriptionID, cred, nil)
    // server, _ := client.Get(context.Background(), resourceGroupName, name, nil)
    // assert.Equal(t, "16", *server.Properties.Version)
    // assert.Equal(t, "B_Standard_B1ms", *server.Properties.SKU.Name)
}
```

**AWS pattern:**
```go
package tests

import (
    "os"
    "testing"

    "github.com/gruntwork-io/terratest/modules/aws"
    "github.com/gruntwork-io/terratest/modules/terraform"
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"
)

func TestResourceComplete(t *testing.T) {
    t.Parallel()

    awsRegion := os.Getenv("AWS_DEFAULT_REGION")
    if awsRegion == "" {
        awsRegion = "us-east-1"
    }

    terraformOptions := &terraform.Options{
        TerraformDir: "../examples/complete",
    }

    defer terraform.Destroy(t, terraformOptions)
    terraform.InitAndApply(t, terraformOptions)

    // Verify Terraform outputs
    bucketName := terraform.Output(t, terraformOptions, "name")
    assert.NotEmpty(t, bucketName)

    arn := terraform.Output(t, terraformOptions, "arn")
    assert.NotEmpty(t, arn)

    // Verify via AWS API - confirm the resource actually exists and is configured correctly
    // Use terratest aws helpers where available:
    //   github.com/gruntwork-io/terratest/modules/aws
    // Fall back to the AWS SDK for resources not covered by terratest helpers:
    //   github.com/aws/aws-sdk-go-v2/service/...

    // Example for an S3 bucket:
    aws.AssertS3BucketExists(t, awsRegion, bucketName)

    versioningStatus := aws.GetS3BucketVersioning(t, awsRegion, bucketName)
    assert.Equal(t, "Enabled", versioningStatus)

    // Example using AWS SDK directly:
    // cfg, _ := config.LoadDefaultConfig(context.Background(), config.WithRegion(awsRegion))
    // client := s3.NewFromConfig(cfg)
    // result, _ := client.GetBucketEncryption(context.Background(), &s3.GetBucketEncryptionInput{Bucket: &bucketName})
    // rule := result.ServerSideEncryptionConfiguration.Rules[0]
    // assert.Equal(t, types.ServerSideEncryptionAes256, rule.ApplyServerSideEncryptionByDefault.SSEAlgorithm)
}
```

**GCP pattern:**
```go
package tests

import (
    "testing"

    "github.com/gruntwork-io/terratest/modules/gcp"
    "github.com/gruntwork-io/terratest/modules/terraform"
    "github.com/stretchr/testify/assert"
    "github.com/stretchr/testify/require"
)

func TestResourceComplete(t *testing.T) {
    t.Parallel()

    projectID := gcp.GetGoogleProjectIDFromEnvVar(t)

    terraformOptions := &terraform.Options{
        TerraformDir: "../examples/complete",
    }

    defer terraform.Destroy(t, terraformOptions)
    terraform.InitAndApply(t, terraformOptions)

    // Verify Terraform outputs
    bucketName := terraform.Output(t, terraformOptions, "name")
    assert.NotEmpty(t, bucketName)

    // Verify via GCP API - confirm the resource actually exists and is configured correctly
    // Use terratest gcp helpers where available:
    //   github.com/gruntwork-io/terratest/modules/gcp
    // Fall back to GCP client libraries for resources not covered by terratest helpers:
    //   google.golang.org/api/...

    // Example for a storage bucket:
    bucket := gcp.GetStorageBucketE(t, projectID, bucketName)
    require.NotNil(t, bucket)
    assert.Equal(t, "US-EAST1", bucket.Location)
    assert.Equal(t, "STANDARD", bucket.StorageClass)
}
```

**Key patterns:**
- File in `tests/` directory (plural)
- Test function named `Test<Resource>Complete`
- Use `t.Parallel()` for concurrent testing
- Defer cleanup with `terraform.Destroy`
- **Always verify both Terraform outputs and cloud provider API state**
- Use terratest provider helpers (`modules/azure`, `modules/aws`, `modules/gcp`) as the first choice
- Use the provider SDK directly for resources not covered by terratest helpers
- Assert on specific configuration values (SKU, version, location, encryption settings), not just non-emptiness
- Read credentials/region from environment variables; fail clearly if required env vars are missing

## Makefile Standards

Every module must have a Makefile with these targets:
```makefile
.PHONY: configure
configure: ## Download and configure shared makefiles and tools
    # Downloads common makefile includes and tooling

.PHONY: env
env: ## Set environment variables (cloud-provider specific)
    # Sources provider_env.sh for authentication

.PHONY: check
check: ## Run all validation checks
    # Combines lint, validate, plan, conftests, terratest, opa

.PHONY: lint
lint: ## Run tflint
    tflint --init
    tflint

.PHONY: validate
validate: ## Validate Terraform code
    terraform validate

.PHONY: test
test: ## Run Terratest
    cd tests && go test -v -timeout 30m

.PHONY: docs
docs: ## Generate documentation
    terraform-docs markdown table --output-file README.md --output-mode inject .
```

**Provider-specific notes:**
- Azure modules reference `azure_env.sh`
- AWS modules would use AWS credentials/profile
- GCP modules would use service account

## Common Anti-Patterns to Avoid

**Don't:**
- Use "this" as resource name (use descriptive names)
- Add multiple resource types to a primitive
- Include business logic or conventions
- Hardcode values (use variables)
- Create abstractions over provider resources
- Make assumptions about use cases
- Mix provider resource patterns (e.g., AWS nested blocks like Azure)

**Do:**
- Wrap one resource type per primitive
- Use a security-first approach to defaults
- Expose all useful functionality via variables
- Use dynamic blocks for optional config (where provider supports it)
- Follow existing module patterns for your provider
- Keep it simple and composable
- Let reference architectures handle opinions
- Follow provider best practices for resource structure

## Creating a New Primitive Module

When asked to create a new primitive module, follow this process:

1. **Identify provider and resource**
   - Which cloud provider? (Azure, AWS, GCP)
   - Which specific resource type?
   - Review provider documentation

2. **Find similar primitives**
   - Look at existing primitives for the same provider
   - Identify common patterns
   - Note provider-specific conventions

3. **Implement core files**
   - `versions.tf` - Set Terraform and provider versions
   - `variables.tf` - All resource arguments as variables
   - `main.tf` - Single resource with dynamic blocks
   - `outputs.tf` - Key resource attributes

4. **Create working example**
   - `examples/complete/` with all required files
   - Use resource naming module
   - Create dependencies (resource group for Azure, etc.)
   - Make it deployable

5. **Write Terratest**
   - `tests/<resource>_test.go`
   - Deploy example, verify outputs, cleanup

6. **Add supporting files**
   - Standard files (LICENSE, NOTICE, etc.)
   - Provider-specific test setup

7. **Validate**
```bash
   make configure
   make env  # Set cloud provider credentials
   make check  # Run all validation
```

8. Carefully check through files and clean up references to the skeleton and templates.


## Skeleton Cleanup Checklist

When transforming the skeleton into a new primitive module, complete ALL of these steps:

### Files to Remove or Transform
- [ ] **TEMPLATED_README.md** → Delete after incorporating relevant content into README.md
- [ ] **Only one Terraform Check CI workflow can be present** (`.github/workflows/pull-request-terraform-check-*.yml`) → Remove Azure for AWS module, remove AWS for Azure module, etc. Keep the one that matches your provider.
- [ ] **`examples/with_cake/`** → Delete skeleton example directory

### Files to Update
- [ ] **`go.mod`** → Update the `lcaf-skeleton-terraform` portion of the `github.com/launchbynttdata/lcaf-skeleton-terraform` header to your module name
- [ ] **Test imports** → Update all Go import paths to match new `go.mod` module path
- [ ] **CI workflow skeleton guard** → Remove the `if: github.repository != 'launchbynttdata/lcaf-skeleton-terraform'` condition from all workflow files
- [ ] **README.md** → Replace Azure-specific references (ARM_CLIENT_ID, azure_env.sh, azurerm provider) with provider-appropriate content

## Cross-Reference

For reference architecture patterns, see [reference-architecture-creator.agent.md](./reference-architecture-creator.agent.md)

These shared standards apply to both primitives and references:
- Commit message formats
- Pre-commit hooks
- Testing approaches
- Makefile targets
- Documentation standards