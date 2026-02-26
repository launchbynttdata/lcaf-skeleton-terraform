<!-- version: 1.3 -->

---
name: Terraform Reference Architecture Creator
description: Agent that creates a Terraform Reference Architecture from a skeleton repository to meet our standards.
---

# AI Agent Guide for Reference Architecture Modules

> **This guide is for reference architecture modules** that compose multiple primitive modules.
> For primitive modules (single resource wrappers), see [agents-primitive.md](./agents-primitive.md)

## Changelog

- **1.3** – Added agent header, migrated to agents folder, added skeleton cleanup checklist.
- **1.2** – Fixed resource naming module usage: `for_each = var.resource_names_map` (not a module input), correct variable name `class_env` (not `environment`), added required `cloud_resource_type`/`maximum_length` params, corrected output reference syntax to `module.resource_names["key"].format`, noted hyphens-stripping for AWS regions, replaced incorrect `resource_names_strategy` variable pattern with correct per-resource output format selection
- **1.1** – Added cloud provider API verification patterns (Azure, AWS, GCP) to Terratest guidance; tests must now verify real resource state via provider SDKs, not just Terraform outputs; reference architecture tests must also cover optional features enabled in the example
- **1.0** – Initial release

> **For agents working in the skeleton repo (`lcaf-skeleton-terraform`):** If you modify this file, update the `<!-- version -->` comment at the top and add a changelog entry here. Bump the minor version (e.g. 1.1 → 1.2) for new guidance or clarifications; bump the major version (e.g. 1.x → 2.0) for changes that would require significant rework of existing modules.

## Overview

Reference architecture modules compose multiple primitive modules to implement complete infrastructure patterns with opinionated configurations, best practices, and additional capabilities like monitoring and private networking.

**Repository naming:** `tf-<provider>-module_reference-<architecture>`

**Examples:** 
- `tf-azurerm-module_reference-postgresql_server`
- `tf-aws-module_reference-lambda_function`
- `tf-google-module_reference-gke_cluster`

## Cloud Providers Supported

- **Azure** (`azurerm` provider) - Primary platform
- **AWS** (`aws` provider) - Large number of modules
- **Google Cloud** (`google` provider) - Growing library

**This guide applies to all cloud providers.** Provider-specific differences are noted where relevant.

## What Makes a Reference Architecture?

Unlike primitives which wrap a single resource with no opinions, reference architectures:
- **Compose multiple primitives** - orchestrate several primitive modules
- **Implement patterns** - encode best practices and organizational standards
- **Add capabilities** - monitoring, alerting, private endpoints, identity integration
- **Provide opinions** - sensible defaults for security and compliance
- **Abstract complexity** - hide implementation details from consumers

## Key Differences from Primitives

| Aspect | Primitives | Reference Architectures |
|--------|-----------|------------------------|
| Purpose | Wrap single resource | Implement complete pattern |
| Resources | ONE resource type | Multiple primitives composed |
| Dependencies | Minimal (just provider) | Many primitives from registry |
| Opinions | None - maximum flexibility | Opinionated - enforce standards |
| Business Logic | No | Yes - implement conventions |
| Variables | Mirror resource arguments | Higher-level abstractions |
| Consumers | Other modules | End users / applications |

## Required File Structure
```
tf-<provider>-module_reference-<architecture>/
├── .github/workflows/      # CI/CD with pre-commit, tests
├── examples/
│   └── complete/           # REQUIRED: Full working example
│       ├── main.tf
│       ├── variables.tf
│       ├── outputs.tf
│       ├── providers.tf
│       └── README.md
├── tests/                  # Note: "tests" not "test"
│   └── <architecture>_test.go
├── main.tf                 # REQUIRED: Compose primitives
├── variables.tf            # REQUIRED: High-level inputs
├── outputs.tf              # REQUIRED: Aggregated outputs
├── versions.tf             # REQUIRED: Version constraints
├── locals.tf               # OPTIONAL: Computed values
├── README.md               # REQUIRED: Auto-generated docs
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

## Composition Patterns

### Always Use the Resource Naming Module

Every reference architecture should start with the resource naming module.

`resource_names_map` is used as `for_each` — the module is invoked once per resource type. The map key (e.g. `"resource_group"`, `"postgresql_server"`) becomes the instance key used to retrieve the generated name. Each instance produces multiple output formats (`standard`, `minimal_random_suffix`, `dns_compliant_standard`, etc.) — choose the appropriate format per resource in your locals or inline.

**Azure pattern:**
```hcl
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

  # Azure region names have no hyphens (e.g. "eastus"), so pass directly
  region                = var.location
  use_azure_region_abbr = var.use_azure_region_abbr
}
```

**AWS pattern:**
```hcl
data "aws_region" "current" {}

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

  # AWS regions have hyphens (e.g. "us-east-1") — strip them so they don't
  # appear as extra separators in generated names
  region = join("", split("-", data.aws_region.current.name))
}
```

**Referencing generated names:**
```hcl
# Syntax: module.resource_names["<map_key>"].<output_format>
module.resource_names["resource_group"].standard          # e.g. "launch-database-eus-dev-rg-000"
module.resource_names["postgresql_server"].standard       # e.g. "launch-database-eus-dev-psql-000"
module.resource_names["s3_bucket"].minimal_random_suffix  # e.g. "launch-bkt-5823947201" (for globally unique names)
module.resource_names["lambda_function"].dns_compliant_standard  # for DNS-constrained names
```

**Why:** Ensures consistent naming across all resources in the architecture.

### Create Resource Group (Azure Only)

**Azure pattern:**
```hcl
module "resource_group" {
  source  = "terraform.registry.launch.nttdata.com/module_primitive/resource_group/azurerm"
  version = "~> 1.0"

  name     = module.resource_names["resource_group"].standard
  location = var.location
  tags     = var.tags
}
```

**AWS pattern:**
```hcl
# No resource group in AWS
# Use tags for grouping and organization
```

**Pattern:** All subsequent Azure resources reference `module.resource_group.name` and `var.location`. AWS uses tags instead.

### Compose Primitives from Registry

**Pattern 1: Using Internal Primitives**

**Azure example:**
```hcl
module "postgresql_server" {
  source  = "terraform.registry.launch.nttdata.com/module_primitive/postgresql_server/azurerm"
  version = "~> 1.1"

  name                = module.resource_names["postgresql_server"].standard
  resource_group_name = module.resource_group.name
  location            = var.location

  # Pass through core configuration
  sku_name                      = var.sku_name
  postgres_version              = var.postgres_version
  storage_mb                    = var.storage_mb
  
  # Networking
  delegated_subnet_id           = var.delegated_subnet_id
  private_dns_zone_id           = var.private_dns_zone_id
  public_network_access_enabled = var.public_network_access_enabled

  # Authentication
  authentication         = var.authentication
  administrator_login    = var.administrator_login
  administrator_password = var.administrator_password

  tags = var.tags
}
```

**AWS example:**
```hcl
module "s3_bucket" {
  source  = "terraform.registry.launch.nttdata.com/module_primitive/s3_bucket/aws"
  version = "~> 1.0"

  bucket = module.resource_names["s3_bucket"].minimal_random_suffix  # S3 bucket names are globally unique
  
  # Pass through configuration
  versioning_enabled = var.versioning_enabled
  encryption_enabled = var.encryption_enabled
  
  tags = var.tags
}
```

**Pattern 2: Using Community Modules**

AWS reference architectures may use well-established community modules:
```hcl
module "lambda_function" {
  source  = "terraform-aws-modules/lambda/aws"
  version = "~> 7.4"

  function_name = module.resource_names["lambda_function"].minimal_random_suffix
  description   = var.description
  handler       = var.handler
  runtime       = var.runtime
  
  # ... extensive configuration
  
  tags = var.tags
}
```

**When to use community modules:**
- Well-maintained, popular modules with good track records
- Complex resources that would duplicate significant effort
- AWS Lambda, VPC, ECS - where terraform-aws-modules are standard

**When to use internal primitives:**
- Simpler resources
- When you need strict control over implementation
- When organizational standards differ from community patterns

### Add Configuration Primitives (for_each pattern)
```hcl
module "postgresql_server_configuration" {
  source   = "terraform.registry.launch.nttdata.com/module_primitive/postgresql_server_configuration/azurerm"
  version  = "~> 1.0"
  for_each = var.server_configuration

  name      = each.key
  server_id = module.postgresql_server.id
  value     = each.value
}
```

**Pattern:** Use `for_each` to create multiple configuration resources from a map variable.

### Add Optional Features (count pattern)

**Azure example - Active Directory Administrator:**
```hcl
module "postgresql_server_ad_administrator" {
  source  = "terraform.registry.launch.nttdata.com/module_primitive/postgresql_server_ad_administrator/azurerm"
  version = "~> 1.0"
  count   = var.ad_administrator != null ? 1 : 0

  server_id      = module.postgresql_server.id
  tenant_id      = var.ad_administrator.tenant_id
  object_id      = var.ad_administrator.object_id
  principal_name = var.ad_administrator.principal_name
  principal_type = var.ad_administrator.principal_type
}
```

**AWS example - IAM policies:**
```hcl
module "lambda_iam_role" {
  source  = "terraform.registry.launch.nttdata.com/module_primitive/iam_role/aws"
  version = "~> 1.0"
  count   = var.create_iam_role ? 1 : 0

  name               = module.resource_names["iam_role"].standard
  assume_role_policy = data.aws_iam_policy_document.lambda_assume_role.json
  
  tags = var.tags
}
```

**Pattern:** Use `count` for optional single-instance features based on variable presence or boolean flags.

### Add Private Endpoint (Azure pattern)
```hcl
module "private_endpoint" {
  source  = "terraform.registry.launch.nttdata.com/module_primitive/private_endpoint/azurerm"
  version = "~> 1.0"
  count   = var.create_private_endpoint ? 1 : 0

  name                = module.resource_names["private_endpoint"].standard
  resource_group_name = module.resource_group.name
  location            = var.location
  subnet_id           = var.private_endpoint_subnet_id

  private_service_connection_name = module.resource_names["private_service_connection"].standard
  private_connection_resource_id  = module.postgresql_server.id
  is_manual_connection            = var.private_endpoint_is_manual_connection
  subresource_names               = var.private_endpoint_subresource_names
  request_message                 = var.private_endpoint_request_message

  private_dns_zone_group_name = var.private_endpoint_dns_zone_group_name
  private_dns_zone_ids        = var.private_endpoint_dns_zone_ids

  tags = var.tags
}
```

**AWS equivalent - VPC endpoints:**
```hcl
module "vpc_endpoint" {
  source  = "terraform.registry.launch.nttdata.com/module_primitive/vpc_endpoint/aws"
  version = "~> 1.0"
  count   = var.create_vpc_endpoint ? 1 : 0

  vpc_id             = var.vpc_id
  service_name       = var.service_name
  subnet_ids         = var.subnet_ids
  security_group_ids = var.security_group_ids

  tags = var.tags
}
```

### Add Monitoring (conditional)

**Azure example:**
```hcl
module "monitor_action_group" {
  source  = "terraform.registry.launch.nttdata.com/module_primitive/monitor_action_group/azurerm"
  version = "~> 1.0.0"
  count   = var.action_group != null ? 1 : 0

  name                = var.action_group.name
  resource_group_name = var.resource_group_name != "" ? var.resource_group_name : module.resource_group.name
  short_name          = var.action_group.short_name

  email_receivers    = var.action_group.email_receivers
  arm_role_receivers = var.action_group.arm_role_receivers

  tags = var.tags
}

module "monitor_metric_alert" {
  source   = "terraform.registry.launch.nttdata.com/module_primitive/monitor_metric_alert/azurerm"
  version  = "~> 2.0"
  for_each = var.metric_alerts

  name                = "${module.resource_names["postgresql_server"].standard}-${each.key}"
  resource_group_name = var.resource_group_name != "" ? var.resource_group_name : module.resource_group.name
  scopes              = [module.postgresql_server.id]

  description = each.value.description
  severity    = each.value.severity
  frequency   = each.value.frequency
  enabled     = each.value.enabled

  action_group_ids = concat(
    var.action_group != null ? [module.monitor_action_group[0].id] : [],
    var.action_group_ids
  )

  criteria         = each.value.criteria
  dynamic_criteria = each.value.dynamic_criteria

  tags = var.tags
}
```

**AWS example - CloudWatch:**
```hcl
module "cloudwatch_log_group" {
  source  = "terraform.registry.launch.nttdata.com/module_primitive/cloudwatch_log_group/aws"
  version = "~> 1.0"
  count   = var.create_cloudwatch_logs ? 1 : 0

  name              = "/aws/lambda/${module.lambda_function.function_name}"
  retention_in_days = var.cloudwatch_logs_retention_in_days
  kms_key_id        = var.cloudwatch_logs_kms_key_id

  tags = var.tags
}

module "cloudwatch_metric_alarm" {
  source   = "terraform.registry.launch.nttdata.com/module_primitive/cloudwatch_metric_alarm/aws"
  version  = "~> 1.0"
  for_each = var.metric_alarms

  alarm_name          = "${module.lambda_function.function_name}-${each.key}"
  comparison_operator = each.value.comparison_operator
  evaluation_periods  = each.value.evaluation_periods
  metric_name         = each.value.metric_name
  namespace           = each.value.namespace
  period              = each.value.period
  statistic           = each.value.statistic
  threshold           = each.value.threshold
  alarm_description   = each.value.description
  alarm_actions       = each.value.alarm_actions

  dimensions = {
    FunctionName = module.lambda_function.function_name
  }

  tags = var.tags
}
```

## Variables Pattern

### Resource Naming Map

**Standard for all reference architectures:**
```hcl
variable "resource_names_map" {
  description = "A map of key to resource_name that will be used by tf-launch-module_library-resource_name to generate resource names"
  type = map(object({
    name       = string
    max_length = optional(number, 60)
  }))
  
  # Azure example
  default = {
    resource_group = {
      name       = "rg"
      max_length = 60
    }
    postgresql_server = {
      name       = "psql"
      max_length = 60
    }
    private_endpoint = {
      name       = "pe"
      max_length = 80
    }
  }
  
  # AWS example
  # default = {
  #   lambda_function = {
  #     name       = "fn"
  #     max_length = 80
  #   }
  #   iam_role = {
  #     name       = "role"
  #     max_length = 64
  #   }
  #   cloudwatch_log_group = {
  #     name       = "lg"
  #     max_length = 512
  #   }
  # }
}
```

**Choosing the right output format per resource:**

Each `module.resource_names` instance exposes multiple output formats. Choose the right one per resource in locals:
```hcl
locals {
  # Most resources: use standard
  iam_role_name    = module.resource_names["iam_role"].standard

  # Globally unique resources (S3 buckets, etc.): use minimal_random_suffix
  s3_bucket_name   = module.resource_names["s3_bucket"].minimal_random_suffix

  # Resources with DNS naming constraints: use dns_compliant_* variants
  lambda_name      = module.resource_names["lambda_function"].dns_compliant_minimal_random_suffix
}
```

Available output formats: `standard`, `lower_case`, `upper_case`, `minimal`, `minimal_random_suffix`, `dns_compliant_standard`, `dns_compliant_minimal`, `dns_compliant_minimal_random_suffix`, `camel_case`, `recommended_per_length_restriction`.

### Naming Context Variables

**Standard for all reference architectures:**
```hcl
variable "logical_product_family" {
  description = "(Required) Name of the product family for which the resource is created. Example: org_name, department_name."
  type        = string
  default     = "launch"
}

variable "logical_product_service" {
  description = "(Required) Name of the product service for which the resource is created. For example, backend, frontend, middleware etc."
  type        = string
  default     = "database"  # or "lambda", "storage", etc.
}

variable "class_env" {
  description = "(Required) Environment where resource is going to be deployed. For example. dev, qa, uat"
  type        = string
  default     = "dev"
}

variable "instance_env" {
  description = "Number that represents the instance of the environment."
  type        = number
  default     = 0
}

variable "instance_resource" {
  description = "Number that represents the instance of the resource."
  type        = number
  default     = 0
}

# Azure-specific
variable "use_azure_region_abbr" {
  description = "Abbreviate the region in the resource names"
  type        = bool
  default     = true
}

variable "location" {
  description = "Azure region where resources will be created"
  type        = string
  default     = "eastus"
}

# AWS doesn't need location variable - uses data source
# data "aws_region" "current" {}

variable "tags" {
  description = "A mapping of tags to assign to the resource."
  type        = map(string)
  default     = {}
}
```

### Core Service Variables

**Pass through from primitive, but provide better defaults:**

**Azure example:**
```hcl
variable "sku_name" {
  description = "The name of the SKU used by this Postgres Flexible Server"
  type        = string
  default     = "B_Standard_B1ms"
}

variable "postgres_version" {
  description = "Version of the Postgres Flexible Server"
  type        = string
  default     = "16"
}

variable "storage_mb" {
  description = "The storage capacity in megabytes"
  type        = number
  default     = 32768  # Reference architecture provides default
}
```

**AWS example:**
```hcl
variable "runtime" {
  description = "Lambda Function runtime"
  type        = string
  default     = "python3.9"
}

variable "handler" {
  description = "Lambda Function entrypoint in your code"
  type        = string
  default     = "index.lambda_handler"
}

variable "memory_size" {
  description = "Amount of memory in MB your Lambda Function can use at runtime"
  type        = number
  default     = 128
}

variable "timeout" {
  description = "The amount of time your Lambda Function has to run in seconds"
  type        = number
  default     = 3
}
```

### Feature Flag Variables

**Enable/disable optional capabilities:**
```hcl
# Azure example
variable "create_private_endpoint" {
  description = "Whether or not to create a Private Endpoint"
  type        = bool
  default     = false
}

# AWS example
variable "create_lambda_function_url" {
  description = "Whether the Lambda Function URL resource should be created"
  type        = bool
  default     = true
}

variable "create" {
  description = "Controls whether resources should be created"
  type        = bool
  default     = false  # Common in AWS modules for safety
}
```

### Complex Object Variables

**For optional features with multiple fields:**
```hcl
# Azure example
variable "ad_administrator" {
  description = <<-EOT
    tenant_id      = The tenant ID of the AD administrator
    object_id      = The object ID of the AD administrator
    principal_name = The name of the principal to assign as AD administrator
    principal_type = The type of principal to assign as AD administrator
  EOT
  type = object({
    tenant_id      = string
    object_id      = string
    principal_name = string
    principal_type = string
  })
  default = null
}

# AWS example
variable "cors" {
  description = "CORS settings to be used by the Lambda Function URL"
  type = object({
    allow_credentials = optional(bool, false)
    allow_headers     = optional(list(string), null)
    allow_methods     = optional(list(string), null)
    allow_origins     = optional(list(string), null)
    expose_headers    = optional(list(string), null)
    max_age           = optional(number, 0)
  })
  default = {}
}
```

## Outputs Pattern

### Aggregate Key Information
```hcl
# Azure example
output "id" {
  description = "The ID of the PostgreSQL server"
  value       = module.postgresql_server.id
}

output "name" {
  description = "The name of the PostgreSQL server"
  value       = module.postgresql_server.name
}

output "fqdn" {
  description = "The FQDN of the PostgreSQL server"
  value       = module.postgresql_server.fqdn
}

output "resource_group_name" {
  description = "The name of the resource group"
  value       = module.resource_group.name
}

# AWS example
output "lambda_function_arn" {
  description = "The ARN of the Lambda function"
  value       = module.lambda_function.lambda_function_arn
}

output "lambda_function_name" {
  description = "The name of the Lambda function"
  value       = module.lambda_function.lambda_function_name
}

output "lambda_function_url" {
  description = "The URL of the Lambda function"
  value       = module.lambda_function.lambda_function_url
}

output "lambda_role_arn" {
  description = "The ARN of the IAM role for the Lambda function"
  value       = module.lambda_function.lambda_role_arn
}
```

### Expose Optional Feature Outputs
```hcl
# Azure example
output "admin_tenant_id" {
  description = "The tenant ID of the AD administrator"
  value       = var.ad_administrator != null ? module.postgresql_server_ad_administrator[0].tenant_id : null
}

# AWS example
output "cloudwatch_log_group_arn" {
  description = "The ARN of the CloudWatch log group"
  value       = var.create_cloudwatch_logs ? module.cloudwatch_log_group[0].arn : null
}
```

### Expose Configuration State
```hcl
# Azure example
output "server_configuration" {
  description = "Map of server configurations applied"
  value       = { for k, v in module.postgresql_server_configuration : k => v.value }
}

# AWS example
output "lambda_iam_policies" {
  description = "List of IAM policies attached to Lambda role"
  value       = var.attach_policies ? var.policies : []
}
```

## versions.tf Pattern

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
  required_version = "~> 1.5"  # Newer - Azure modules should update

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.14"
    }
  }
}
```

**NOTE:** AWS modules using `~> 1.5` represents a newer standard. Azure modules still using `~> 1.0` should be updated to match.

## locals.tf Pattern

Use locals for computed values, complex logic, or to avoid repetition:

**Azure example:**
```hcl
locals {
  # Determine resource group for monitoring resources
  monitor_resource_group = var.resource_group_name != "" ? var.resource_group_name : module.resource_group.name

  # Combine action group IDs
  all_action_group_ids = concat(
    var.action_group != null ? [module.monitor_action_group[0].id] : [],
    var.action_group_ids
  )

  # Compute complex conditionals
  private_endpoint_enabled = var.create_private_endpoint && var.private_endpoint_subnet_id != null
}
```

**AWS example:**
```hcl
locals {
  # Pick the appropriate output format per resource type
  function_name   = module.resource_names["lambda_function"].dns_compliant_minimal_random_suffix
  iam_role_name   = module.resource_names["iam_role"].standard

  # Determine IAM role ARN
  lambda_role_arn = var.create_iam_role ? module.lambda_iam_role[0].arn : var.existing_iam_role_arn
}
```

**When to use locals:**
- Complex conditionals used multiple times
- Data transformations
- Combining lists or maps
- Default value computation
- Selecting the right resource name output format per resource

## Provider-Specific Patterns

### Azure Patterns

**Resource Group Management:**
```hcl
variable "resource_group_name" {
  description = "Optional resource group name. If empty, a new one is created."
  type        = string
  default     = ""
}

module "resource_group" {
  source  = "..."
  version = "~> 1.0"
  count   = var.resource_group_name == "" ? 1 : 0

  name     = module.resource_names["resource_group"].standard
  location = var.location
  tags     = var.tags
}

locals {
  resource_group_name = var.resource_group_name != "" ? var.resource_group_name : module.resource_group[0].name
}
```

**Private Networking:**
- Private endpoints for PaaS services
- VNet integration for secure access
- Private DNS zones for name resolution

**Monitoring:**
- Azure Monitor action groups
- Metric alerts
- Log Analytics workspaces

### AWS Patterns

**Region from Data Source:**
```hcl
data "aws_region" "current" {}

# Use data.aws_region.current.name in modules
```

**Selecting naming output format in locals:**
```hcl
locals {
  # Use the output format appropriate to each resource's constraints:
  # - .standard for most resources
  # - .minimal_random_suffix for globally unique resources (S3, ECR, etc.)
  # - .dns_compliant_* for DNS-constrained resources
  function_name = module.resource_names["lambda_function"].dns_compliant_minimal_random_suffix
  bucket_name   = module.resource_names["s3_bucket"].minimal_random_suffix
}
```

**Using Community Modules:**
```hcl
module "lambda_function" {
  source  = "terraform-aws-modules/lambda/aws"
  version = "~> 7.4"
  
  # Extensive configuration options
  # Well-maintained, widely used
}
```

**IAM Policy Flexibility:**
```hcl
variable "attach_policy_statements" {
  type    = bool
  default = false
}

variable "policy_statements" {
  type    = map(string)
  default = {}
}

variable "attach_policy" {
  type    = bool
  default = false
}

variable "policy" {
  type    = string
  default = null
}

# Multiple ways to attach policies
```

**VPC Integration:**
```hcl
variable "vpc_subnet_ids" {
  description = "List of subnet ids when Lambda should run in VPC"
  type        = list(string)
  default     = null
}

variable "vpc_security_group_ids" {
  description = "List of security group ids when Lambda should run in VPC"
  type        = list(string)
  default     = null
}
```

**CloudWatch Logs:**
```hcl
variable "attach_cloudwatch_logs_policy" {
  type    = bool
  default = true
}

variable "cloudwatch_logs_retention_in_days" {
  type    = number
  default = 30
}

variable "cloudwatch_logs_kms_key_id" {
  type    = string
  default = null
}
```

## Testing Pattern

### Terratest for Reference Architectures

Tests must verify **both** Terraform outputs **and** actual resource state via the cloud provider API. Terraform outputs are generated by Terraform itself and do not prove the cloud resources were actually created or configured correctly. Always use provider SDK helpers to confirm real resource state for each significant resource in the architecture.

Reference architecture tests should cover:
- The primary resource (existence + key configuration)
- Optional features that were enabled in the example (private endpoint, monitoring, AD integration, etc.)
- Outputs from composed primitives (resource group name, FQDNs, ARNs, etc.)

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

func TestArchitectureComplete(t *testing.T) {
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

    fqdn := terraform.Output(t, terraformOptions, "fqdn")
    assert.NotEmpty(t, fqdn)

    resourceGroupName := terraform.Output(t, terraformOptions, "resource_group_name")
    assert.NotEmpty(t, resourceGroupName)

    // Verify primary resource via Azure API
    // Use terratest azure helpers where available (github.com/gruntwork-io/terratest/modules/azure)
    // Fall back to the Azure SDK for resources not covered by terratest helpers
    // (github.com/Azure/azure-sdk-for-go/sdk/resourcemanager/...)

    // Example: verify the PostgreSQL server exists and is correctly configured
    // cred, _ := azidentity.NewDefaultAzureCredential(nil)
    // client, _ := armpostgresqlflexibleservers.NewServersClient(subscriptionID, cred, nil)
    // server, _ := client.Get(context.Background(), resourceGroupName, name, nil)
    // require.NotNil(t, server.Properties)
    // assert.Equal(t, armpostgresqlflexibleservers.ServerVersion("16"), *server.Properties.Version)
    // assert.Equal(t, "B_Standard_B1ms", *server.Properties.SKU.Name)
    // assert.False(t, *server.Properties.Network.PublicNetworkAccess == armpostgresqlflexibleservers.ServerPublicNetworkAccessStateEnabled)

    // Verify optional features that are enabled in the example
    // Example: confirm private endpoint was created
    // privateEndpointName := terraform.Output(t, terraformOptions, "private_endpoint_name")
    // assert.NotEmpty(t, privateEndpointName)
    // pe := azure.GetPrivateEndpoint(t, resourceGroupName, privateEndpointName, subscriptionID)
    // require.NotNil(t, pe)
    // assert.Equal(t, "Approved", *pe.Properties.PrivateLinkServiceConnections[0].Properties.PrivateLinkServiceConnectionState.Status)
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

func TestArchitectureComplete(t *testing.T) {
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
    functionName := terraform.Output(t, terraformOptions, "lambda_function_name")
    assert.NotEmpty(t, functionName)

    functionArn := terraform.Output(t, terraformOptions, "lambda_function_arn")
    assert.NotEmpty(t, functionArn)

    roleArn := terraform.Output(t, terraformOptions, "lambda_role_arn")
    assert.NotEmpty(t, roleArn)

    // Verify primary resource via AWS API
    // Use terratest aws helpers where available (github.com/gruntwork-io/terratest/modules/aws)
    // Fall back to the AWS SDK for resources not covered by terratest helpers
    // (github.com/aws/aws-sdk-go-v2/service/...)

    // Example: verify the Lambda function exists and is correctly configured
    // cfg, _ := config.LoadDefaultConfig(context.Background(), config.WithRegion(awsRegion))
    // client := lambda.NewFromConfig(cfg)
    // result, _ := client.GetFunction(context.Background(), &lambda.GetFunctionInput{FunctionName: &functionName})
    // require.NotNil(t, result.Configuration)
    // assert.Equal(t, "python3.9", string(result.Configuration.Runtime))
    // assert.EqualValues(t, 256, *result.Configuration.MemorySize)
    // assert.Equal(t, types.StateActive, result.Configuration.State)

    // Verify optional features enabled in the example
    // Example: confirm CloudWatch log group was created
    // logGroupName := "/aws/lambda/" + functionName
    // aws.AssertCloudWatchLogGroupExists(t, awsRegion, logGroupName)

    // Example: confirm IAM role has the expected policies
    // iamClient := iam.NewFromConfig(cfg)
    // policies, _ := iamClient.ListAttachedRolePolicies(context.Background(), &iam.ListAttachedRolePoliciesInput{RoleName: &roleName})
    // assert.NotEmpty(t, policies.AttachedPolicies)
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

func TestArchitectureComplete(t *testing.T) {
    t.Parallel()

    projectID := gcp.GetGoogleProjectIDFromEnvVar(t)

    terraformOptions := &terraform.Options{
        TerraformDir: "../examples/complete",
    }

    defer terraform.Destroy(t, terraformOptions)
    terraform.InitAndApply(t, terraformOptions)

    // Verify Terraform outputs
    name := terraform.Output(t, terraformOptions, "name")
    assert.NotEmpty(t, name)

    // Verify primary resource via GCP API
    // Use terratest gcp helpers where available (github.com/gruntwork-io/terratest/modules/gcp)
    // Fall back to GCP client libraries for resources not covered by terratest helpers
    // (google.golang.org/api/...)

    // Example: verify a GKE cluster exists and is correctly configured
    // cluster := gcp.GetGkeCluster(t, projectID, "us-east1", name)
    // require.NotNil(t, cluster)
    // assert.Equal(t, "RUNNING", cluster.Status)
    // assert.True(t, cluster.AddonsConfig.HttpLoadBalancing.Disabled == false)

    // Example: verify a storage bucket
    // bucket := gcp.GetStorageBucketE(t, projectID, name)
    // require.NotNil(t, bucket)
    // assert.Equal(t, "US-EAST1", bucket.Location)
    // assert.Equal(t, "STANDARD", bucket.StorageClass)
}
```

**Key patterns:**
- Use `t.Parallel()` for concurrent testing
- Defer cleanup with `terraform.Destroy`
- **Always verify both Terraform outputs and cloud provider API state**
- Use terratest provider helpers (`modules/azure`, `modules/aws`, `modules/gcp`) as the first choice
- Use the provider SDK directly for resources not covered by terratest helpers
- Test the primary resource's existence **and** specific configuration values (SKU, version, runtime, encryption, access settings)
- Test each optional feature that is enabled in the `examples/complete` example
- Read credentials/region from environment variables; fail clearly if required env vars are missing
- Assert on specific values, not just non-emptiness — this catches misconfiguration that Terraform would still report as a successful output

## Example Usage in examples/complete/

**Azure example:**
```hcl
module "postgresql_reference" {
  source = "../.."

  # Naming context
  logical_product_family  = "launch"
  logical_product_service = "database"
  class_env               = "dev"
  instance_env            = 0
  instance_resource       = 0
  location                = "eastus"

  # Core configuration
  sku_name         = "B_Standard_B1ms"
  postgres_version = "16"
  storage_mb       = 32768

  # Authentication
  administrator_login    = "postgresadmin"
  administrator_password = random_password.postgres.result

  # Networking
  delegated_subnet_id = module.subnet.id
  private_dns_zone_id = module.private_dns_zone.id

  # Private endpoint
  create_private_endpoint       = true
  private_endpoint_subnet_id    = module.pe_subnet.id
  private_endpoint_dns_zone_ids = [module.private_dns_zone.id]

  # Monitoring
  action_group = {
    name       = "postgresql-action-group"
    short_name = "pgsql-ag"
    email_receivers = [{
      name          = "ops-team"
      email_address = "ops@example.com"
    }]
  }

  tags = var.tags
}
```

**AWS example:**
```hcl
module "lambda_reference" {
  source = "../.."

  # Naming context
  logical_product_family  = "launch"
  logical_product_service = "lambda"
  class_env               = "demo"
  instance_env            = 0
  instance_resource       = 0

  # Core Lambda configuration
  runtime      = "python3.9"
  handler      = "index.lambda_handler"
  memory_size  = 256
  timeout      = 10

  # Source code
  create_package = true
  source_path    = "${path.module}/lambda_code"

  # IAM permissions
  attach_policy_statements = true
  policy_statements = {
    s3_read = {
      effect    = "Allow"
      actions   = ["s3:GetObject"]
      resources = ["arn:aws:s3:::my-bucket/*"]
    }
  }

  # CloudWatch Logs
  attach_cloudwatch_logs_policy     = true
  cloudwatch_logs_retention_in_days = 7

  # Function URL
  create_lambda_function_url = true
  authorization_type         = "NONE"

  # Enable creation
  create = true

  tags = var.tags
}
```

## Best Practices

### 1. Composition Over Configuration

**Good:** Compose multiple primitives
```hcl
module "postgresql_server" { ... }
module "postgresql_server_configuration" { ... }
module "private_endpoint" { ... }
module "monitor_action_group" { ... }
```

**Bad:** Creating resources directly
```hcl
resource "azurerm_postgresql_flexible_server" "this" { ... }
```

### 2. Make Features Optional

Use boolean flags or null checks:
```hcl
count = var.create_private_endpoint ? 1 : 0
count = var.ad_administrator != null ? 1 : 0
count = var.create ? 1 : 0  # AWS pattern
```

### 3. Provide Sensible Defaults

Reference architectures should have opinions:
```hcl
variable "public_network_access_enabled" {
  type    = bool
  default = false  # Security best practice
}

variable "backup_retention_days" {
  type    = number
  default = 7  # Minimum recommended
}
```

### 4. Use Community Modules Wisely (AWS)
```hcl
# Good: Use for complex, well-maintained resources
module "lambda" {
  source = "terraform-aws-modules/lambda/aws"
  # ...
}

# Consider: For simpler resources, use internal primitives
module "s3_bucket" {
  source = "terraform.registry.launch.nttdata.com/module_primitive/s3_bucket/aws"
  # ...
}
```

### 5. Handle Provider Differences
```hcl
# Azure - explicit location
variable "location" {
  type = string
}

# AWS - from data source
data "aws_region" "current" {}
```

### 6. Version Constraints

Pin to minor versions of primitives:
```hcl
# Internal primitives
source  = "terraform.registry.launch.nttdata.com/module_primitive/postgresql_server/azurerm"
version = "~> 1.1"  # Allows 1.1.x, not 1.2.0

# Community modules
source  = "terraform-aws-modules/lambda/aws"
version = "~> 7.4"  # Allows 7.4.x, not 7.5.0
```

## Common Patterns

### Pattern: Optional Resource Group (Azure)
```hcl
variable "resource_group_name" {
  description = "Optional resource group name. If empty, a new one is created."
  type        = string
  default     = ""
}

module "resource_group" {
  source  = "..."
  count   = var.resource_group_name == "" ? 1 : 0
  # ...
}

locals {
  resource_group_name = var.resource_group_name != "" ? var.resource_group_name : module.resource_group[0].name
}
```

### Pattern: Resource Names Output Format (AWS)

The `resource_names` module outputs multiple name formats per resource. Select the right one per resource in locals:
```hcl
locals {
  # Standard: most Azure and AWS resources
  rg_name       = module.resource_names["resource_group"].standard

  # Minimal with random suffix: globally unique resources (S3, ECR, etc.)
  bucket_name   = module.resource_names["s3_bucket"].minimal_random_suffix

  # DNS-compliant: resources with DNS naming constraints (Lambda, ECS, etc.)
  lambda_name   = module.resource_names["lambda_function"].dns_compliant_minimal_random_suffix
}
```

### Pattern: Combining Lists/IDs
```hcl
# Azure - action groups
locals {
  all_action_group_ids = concat(
    var.action_group != null ? [module.monitor_action_group[0].id] : [],
    var.action_group_ids
  )
}

# AWS - IAM policies
locals {
  all_policy_arns = concat(
    var.attach_policy ? [var.policy] : [],
    var.attach_policies ? var.policies : []
  )
}
```

### Pattern: For_each Over Maps
```hcl
variable "server_configuration" {
  type    = map(string)
  default = {}
}

module "configuration" {
  source   = "..."
  for_each = var.server_configuration

  name  = each.key
  value = each.value
}
```

## Anti-Patterns to Avoid

**Don't:**
- Create resources directly (use primitives)
- Hardcode resource names (use resource naming module)
- Make everything required (provide sensible defaults)
- Ignore monitoring and observability
- Skip private networking options (Azure)
- Skip VPC integration options (AWS)
- Omit IAM/AD integration
- Create monolithic architectures (keep focused)
- Duplicate primitive logic (compose, don't copy)
- Mix cloud provider patterns

**Do:**
- Compose primitives
- Use generated names
- Make features optional with good defaults
- Include monitoring capabilities
- Support secure networking (private endpoints/VPC)
- Enable identity integration where applicable
- Keep architectures focused on one pattern
- Trust and use primitives as building blocks
- Follow provider-specific best practices

## Creating a New Reference Architecture

When asked to create a new reference architecture:

1. **Identify the pattern and provider**
   - What problem does this solve?
   - Which cloud provider? (Azure, AWS, GCP)
   - What primitives are needed?
   - What additional capabilities?

2. **Plan the composition**
   - Core primitive(s) for main resources
   - Configuration primitives
   - Networking primitives (private endpoints for Azure, VPC for AWS)
   - Monitoring primitives (Azure Monitor, CloudWatch)
   - Identity primitives (AD for Azure, IAM for AWS)

3. **Design the interface**
   - What should users provide?
   - What should have sensible defaults?
   - What should be optional?
   - Provider-specific requirements?

4. **Implement**
   - Start with resource naming
   - Add resource group (Azure only)
   - Get region from data source (AWS)
   - Compose core primitive or community module
   - Add optional features with conditionals
   - Add monitoring last

5. **Test**
   - Create complete example
   - Write Terratest
   - Verify all optional features can be enabled/disabled

6. **Document**
   - Explain the pattern being implemented
   - Document all optional features
   - Provide usage examples
   - Note provider-specific considerations

## Updates Needed for Older Modules

Based on comparing Azure and AWS modules, older modules may need:

1. **Terraform version update**
   - Old: `required_version = "~> 1.0"`
   - New: `required_version = "~> 1.5"`

2. **Resource naming module version**
   - Ensure using `version = "~> 2.0"`

3. **AWS-specific additions**
   - Use `locals` to select the appropriate output format per resource (`.standard`, `.minimal_random_suffix`, `.dns_compliant_*`)

4. **Consistent patterns**
   - Use `count` for optional features
   - Use `for_each` for multiple similar resources
   - Support pre-existing resources (resource groups, IAM roles)

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

For primitive module patterns, see [agents-primitive.md](./agents-primitive.md)

These shared standards apply to both primitives and references:
- Commit message formats
- Pre-commit hooks
- Testing approaches
- Makefile targets
- Documentation standards