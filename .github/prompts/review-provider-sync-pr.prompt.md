---
description: "Review a provider-sync PR generated automatically by the azurerm sync workflow. Validates code quality, convention compliance, and correctness of generated Terraform changes."
agent: "agent"
tools: ["search", "read_file"]
---

Review the open provider-sync PR in this repository. Perform the following checks and report findings grouped by severity (❌ blocker, ⚠️ warning, ✅ pass):

## 1. Convention compliance

Check each modified file against the module conventions in `AGENTS.md` and the org-level `copilot-instructions.md`:

- **variables.tf**: Optional fields use `optional()` inside `object({})`. Optional top-level variables have `default = null` or `default = []`. Every variable has `type` and `description`. Validation blocks use `contains()` for enums and `can(regex())` for patterns. Cross-field validations are on the dependent variable.
- **main.tf**: Optional single blocks use `dynamic` + `for_each = var.x != null ? [""] : []`. Repeatable sub-resources use `for_each = { for item in var.list : item.name => item }`. No `count` on named resources. No `provider` block. No hardcoded variable values — references go through `each.value.*`.
- **outputs.tf**: Every output has `value` and `description`. Collection outputs are maps keyed by name.
- **CHANGELOG.md**: New entry appears under the correct section (`FEATURES:`, `ENHANCEMENTS:`, or `BREAKING CHANGES:`). Format matches `* **New Parameter:** \`variable_name\``.

## 2. Correctness

- Do the new variables match the provider documentation attribute types exactly?
- Are all required sub-attributes of a new block included (not just the optional ones)?
- If a feature requires coordinated changes across multiple resources (e.g., `azurerm_virtual_network` + `azurerm_subnet`), are all resources updated consistently?
- Are there validation constraints that enforce Azure-level restrictions (e.g., a parameter only valid for a specific SKU)?

## 3. Test coverage

- Does `tests/testing.tftest.hcl` include at least one new `assert` block for the added parameter in the `run "plan"` block?
- Does the assert message follow the format: `"The <resource> <attribute> input variable is being modified."`?
- Is the assert condition actually verifiable at plan time (not an apply-time attribute)?

## 4. Scope

- Does the PR implement ONLY what the title/description says? Flag any unrelated changes.
- Are there files modified that should NOT have been changed (e.g., `versions.tf`, `tests/environment/*`)?

## Summary

End your review with a concise summary table and a recommendation: **approve**, **request changes**, or **needs discussion**.
