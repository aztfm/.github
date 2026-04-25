# aztfm Terraform Module Conventions

All repositories in this organization are Terraform modules for Azure (`hashicorp/azurerm` provider).
Each module wraps one or more related `azurerm_*` resources that together implement a single Azure concept.

## File structure

Every module contains exactly these files (add `locals.tf` only when computed lookup maps are needed):

```
main.tf          # resource blocks only — no provider, no data sources, no locals
variables.tf     # all input variables
outputs.tf       # all outputs
versions.tf      # terraform + provider version constraints
locals.tf        # optional: lookup maps (e.g. service_delegation_actions)
CHANGELOG.md
README.md
tests/
  testing.tftest.hcl
  environment/
    main.tf      # azurerm_resource_group with uuid()-based name
    outputs.tf
    locals.tf
    versions.tf
examples/
  <descriptive-name>/
    main.tf
```

## variables.tf conventions

```hcl
# Simple scalar — always include description
variable "name" {
  type        = string
  description = "The name of the ..."
}

# Optional scalar — use default = null
variable "optional_setting" {
  type        = string
  description = "..."
  default     = null
}

# Complex optional object — use optional() for every field that has a sensible default
variable "blob_properties" {
  type = object({
    versioning_enabled  = optional(bool, false)
    delete_retention_days = optional(number, 7)
  })
  description = "..."
  default     = null
}

# Repeatable sub-resources — list of objects with optional() fields
variable "containers" {
  type = list(object({
    name                  = string
    container_access_type = optional(string, "private")
  }))
  description = "..."
  default     = []
}

# Validation — use contains() for enums, can(regex()) for patterns
variable "account_tier" {
  validation {
    condition     = contains(["Standard", "Premium"], var.account_tier)
    error_message = "The account tier must be either Standard or Premium."
  }
}

# Cross-field validation — add as second validation block on the dependent variable
variable "account_kind" {
  validation {
    condition     = contains(["FileStorage", "BlockBlobStorage"], var.account_kind) ? var.account_tier == "Premium" : true
    error_message = "FileStorage and BlockBlobStorage account kinds require Premium tier."
  }
}
```

## main.tf conventions

```hcl
# Optional single block — null-guard idiom
resource "azurerm_storage_account" "main" {
  dynamic "blob_properties" {
    for_each = var.blob_properties != null ? [""] : []
    content {
      versioning_enabled    = blob_properties.value.versioning_enabled
      delete_retention_policy {
        days = blob_properties.value.delete_retention_days
      }
    }
  }
}

# Repeatable sub-resources — for_each with name-keyed map
resource "azurerm_storage_container" "containers" {
  for_each = { for container in var.containers : container.name => container }
  name                  = each.key
  storage_account_id    = azurerm_storage_account.main.id
  container_access_type = each.value.container_access_type
}
```

Rules:
- Never use `count` for resources that have a `name` attribute
- Never declare a `provider` block in a module
- Reference child resources via the primary resource: `azurerm_storage_account.main.name`, not `var.name`
- Hardcode values that are always correct (e.g. `type = "UserAssigned"` when only UserAssigned is valid)

## outputs.tf conventions

```hcl
# Every output has value + description
output "id" {
  value       = azurerm_storage_account.main.id
  description = "The ID of the Storage Account."
}

# Collection outputs are maps keyed by name
output "containers" {
  value       = { for container in azurerm_storage_container.containers : container.name => container }
  description = "A map of Storage Containers. Access: module.NAME.containers[\"CONTAINER_NAME\"].id"
}
```

## CHANGELOG.md format

```markdown
<!-- markdownlint-disable MD041 -->

## UNRELEASED

FEATURES:

* **New Parameter:** `blob_properties` ([#N](link))

ENHANCEMENTS:

* `account_replication_type`: added `Premium_ZRS` as valid value ([#N](link))

BREAKING CHANGES:

* `access_tier` default changed from `Hot` to `null` ([#N](link))

---

## 1.2.0 (May 04, 2024)
...
```

## tests/testing.tftest.hcl conventions

```hcl
provider "azurerm" { features {} }

run "setup" {
  module { source = "./tests/environment" }
}

variables {
  # shared values used across runs
  location = run.setup.resource_group_location
  tags     = run.setup.resource_group_tags
}

run "plan" {
  command = plan
  variables {
    name                = run.setup.workspace_id
    resource_group_name = run.setup.resource_group_name
  }
  assert {
    condition     = azurerm_storage_account.main.account_tier == "Standard"
    error_message = "The storage account account_tier input variable is being modified."
  }
}

run "apply" {
  command = apply
  # apply-time asserts go here
}
```

Assert message format: `"The <resource_type_short> <attribute> input variable is being modified."`
Only add asserts for attributes verifiable at plan time.
For resources with name length constraints, use: `substr(replace(run.setup.workspace_id, "-", ""), 0, 24)`
