# PR Review Guide: Fix Spurious Plan Changes in Pod Resource

This document explains the changes made in the `fix-pod-read-state` branch to help reviewers understand the problem and solution, even without prior Terraform provider development experience.

## Table of Contents

1. [Background: How Terraform Providers Work](#background-how-terraform-providers-work)
2. [The Problem](#the-problem)
3. [Root Cause Analysis](#root-cause-analysis)
4. [The Solution](#the-solution)
5. [Code Changes Explained](#code-changes-explained)
6. [Testing](#testing)
7. [References](#references)

---

## Background: How Terraform Providers Work

### The CRUD Lifecycle

Terraform providers implement a [CRUD](https://developer.hashicorp.com/terraform/plugin/framework/resources) (Create, Read, Update, Delete) lifecycle for each resource type. The key operations are:

- **Create**: Called when `terraform apply` creates a new resource
- **Read**: Called during `terraform plan` and `terraform apply` to fetch the current state of a resource from the API
- **Update**: Called when `terraform apply` needs to modify an existing resource
- **Delete**: Called when `terraform apply` needs to destroy a resource

### State Management

Terraform maintains a [state file](https://developer.hashicorp.com/terraform/language/state) that stores the last-known configuration of all managed resources. During `plan`, Terraform:

1. Calls the provider's **Read** function to get the current state from the API
2. Compares the API state with the desired configuration (your `.tf` files)
3. Shows a diff of what needs to change

### Schema Attributes

Each resource has a [schema](https://developer.hashicorp.com/terraform/plugin/framework/handling-data/schemas) defining its attributes. Key properties include:

| Property | Meaning |
|----------|---------|
| `Required` | User must provide this value |
| `Optional` | User may provide this value |
| `Computed` | Provider sets this value (e.g., from API response) |
| `Optional + Computed` | User may provide, or provider computes a default |
| `Default` | Value used when user doesn't specify one |

### Plan Modifiers

[Plan modifiers](https://developer.hashicorp.com/terraform/plugin/framework/resources/plan-modification) alter how Terraform handles attribute values during planning. The key one for this fix is:

- **`UseStateForUnknown()`**: During planning, if the planned value would be "unknown" (computed at apply time), use the current state value instead. This prevents spurious "(known after apply)" diffs.

---

## The Problem

When running `tofu plan` (or `terraform plan`) on infrastructure with RunPod pods, users saw spurious "update in-place" changes even when nothing had actually changed:

```hcl
# runpod_pod.my_pod will be updated in-place
~ resource "runpod_pod" "my_pod" {
    + cloud_type           = "SECURE"
    + compute_type         = "GPU"
    ~ cost_per_hr          = 0.4 -> (known after apply)
    ~ desired_status       = "RUNNING" -> (known after apply)
    + gpu_count            = 1
    + gpu_type_priority    = "availability"
    ~ machine_id           = "abc123" -> (known after apply)
    ~ memory_in_gb         = 55 -> (known after apply)
    + min_ram_per_gpu      = 8
    + min_vcpu_per_gpu     = 2
    ~ public_ip            = "1.2.3.4" -> (known after apply)
      # ... etc
  }
```

This happened on **every single plan**, making it impossible to:
- Know if real changes were pending
- Safely run `terraform apply` without unintended modifications
- Trust the infrastructure state

---

## Root Cause Analysis

The issue had three distinct causes:

### 1. Incomplete `Read` Function

The `updateStateFromPod()` helper function (called by `Read`) was only setting a few fields from the API response:

```go
// BEFORE: Only these fields were being set
data.DesiredStatus = types.StringValue(pod.DesiredStatus)
data.PublicIp = types.StringValue(pod.PublicIp)
data.MachineId = types.StringValue(pod.MachineId)
data.CostPerHr = types.Float64Value(pod.CostPerHr)
// ... a few more

// Missing: ComputeType, CloudType, GPUCount, Interruptible, Locked, etc.
```

**Result**: Many attributes remained `null` in state, causing Terraform to think they needed to be added.

### 2. Missing `UseStateForUnknown()` Plan Modifiers

Computed-only fields (like `public_ip`, `cost_per_hr`) were defined without plan modifiers:

```go
// BEFORE
"cost_per_hr": schema.Float64Attribute{
    Computed: true,  // No plan modifier!
},
```

**Result**: Terraform showed `(known after apply)` for these fields on every plan, even though the values hadn't changed.

### 3. Empty State + Default Values = Perpetual Diff

For `Optional + Computed` fields with defaults (like `cloud_type = "SECURE"`):

1. The API doesn't return these fields
2. The Read function didn't set them (cause #1)
3. State remained empty/null
4. During plan, Terraform applied the schema default
5. Plan showed: `+ cloud_type = "SECURE"` (adding the default)
6. But Apply never ran, so state stayed empty
7. Next plan showed the same diff again

This created an infinite loop of spurious diffs.

---

## The Solution

### Fix 1: Populate All Fields in `updateStateFromPod()`

The Read function now sets all relevant fields from the API response:

```go
// Computed status fields - always set from API
data.DesiredStatus = types.StringValue(pod.DesiredStatus)
data.PublicIp = types.StringValue(pod.PublicIp)
data.CostPerHr = types.Float64Value(pod.CostPerHr)
// ... etc

// Boolean fields - always present in API response
data.Interruptible = types.BoolValue(pod.Interruptible)
data.Locked = types.BoolValue(pod.Locked)
data.GlobalNetworking = types.BoolValue(pod.GlobalNetworking)
```

### Fix 2: Add `UseStateForUnknown()` to Computed Fields

All computed-only fields now have plan modifiers:

```go
// AFTER
"cost_per_hr": schema.Float64Attribute{
    Computed: true,
    PlanModifiers: []planmodifier.Float64{
        float64planmodifier.UseStateForUnknown(),
    },
},
```

This tells Terraform: "If you don't know the new value yet, assume it won't change from the current state."

### Fix 3: Apply Defaults When State is Empty

For fields the API doesn't return, we now check if state is empty and apply the default:

```go
// Use API value if available, otherwise preserve state, otherwise use default
if pod.ComputeType != "" {
    data.ComputeType = types.StringValue(pod.ComputeType)
} else if data.ComputeType.IsNull() || data.ComputeType.IsUnknown() {
    data.ComputeType = types.StringValue("GPU") // default
}
// (if state has a value, it's preserved automatically)
```

This "heals" existing state that was corrupted by the previous buggy provider version.

---

## Code Changes Explained

### File: `internal/provider/pod_resource.go`

#### New Imports

```go
import (
    // ... existing imports ...
    "github.com/hashicorp/terraform-plugin-framework/resource/schema/float64planmodifier"
    "github.com/hashicorp/terraform-plugin-framework/resource/schema/int64planmodifier"
)
```

These provide the `UseStateForUnknown()` modifiers for `Float64` and `Int64` attribute types.

#### Schema Changes

Added `PlanModifiers` to 17 attributes:

| Attribute | Type | Why |
|-----------|------|-----|
| `compute_type` | String (Optional+Computed) | API doesn't return; preserve state |
| `cloud_type` | String (Optional+Computed) | API doesn't return; preserve state |
| `gpu_count` | Int64 (Optional+Computed) | API doesn't return; preserve state |
| `vcpu_count` | Int64 (Optional+Computed) | API doesn't return; preserve state |
| `min_vcpu_per_gpu` | Int64 (Optional+Computed) | API doesn't return; preserve state |
| `min_ram_per_gpu` | Int64 (Optional+Computed) | API doesn't return; preserve state |
| `gpu_type_priority` | String (Optional+Computed) | API doesn't return; preserve state |
| `cpu_flavor_priority` | String (Optional+Computed) | API doesn't return; preserve state |
| `data_center_priority` | String (Optional+Computed) | API doesn't return; preserve state |
| `desired_status` | String (Computed) | Prevent "(known after apply)" |
| `public_ip` | String (Computed) | Prevent "(known after apply)" |
| `machine_id` | String (Computed) | Prevent "(known after apply)" |
| `actual_data_center` | String (Computed) | Prevent "(known after apply)" |
| `cost_per_hr` | Float64 (Computed) | Prevent "(known after apply)" |
| `adjusted_cost_per_hr` | Float64 (Computed) | Prevent "(known after apply)" |
| `memory_in_gb` | Float64 (Computed) | Prevent "(known after apply)" |
| `last_started_at` | String (Computed) | Prevent "(known after apply)" |

#### `updateStateFromPod()` Changes

**Before**: ~15 lines, only set a handful of fields

**After**: ~90 lines, comprehensive field population with three patterns:

1. **Always set from API** (for fields API always returns):
   ```go
   data.DesiredStatus = types.StringValue(pod.DesiredStatus)
   ```

2. **Set if API returns, else preserve state or use default**:
   ```go
   if pod.ComputeType != "" {
       data.ComputeType = types.StringValue(pod.ComputeType)
   } else if data.ComputeType.IsNull() || data.ComputeType.IsUnknown() {
       data.ComputeType = types.StringValue("GPU")
   }
   ```

3. **Set from nested object** (for `actual_data_center`):
   ```go
   if pod.Machine != nil {
       if dataCenterId, ok := pod.Machine["dataCenterId"].(string); ok {
           data.ActualDataCenter = types.StringValue(dataCenterId)
       }
   }
   ```

---

## Testing

### Before Fix

```bash
$ tofu plan
# runpod_pod.my_pod will be updated in-place
# ... 15+ spurious changes shown ...
Plan: 0 to add, 4 to change, 0 to destroy.
```

### After Fix

```bash
$ tofu plan
No changes. Your infrastructure matches the configuration.
```

### How to Test Locally

1. Build the provider:
   ```bash
   cd ~/work/neena/terraform-provider-runpod
   go build -o terraform-provider-runpod
   ```

2. Create a dev override config (`~/.terraform.d/dev.tfrc`):
   ```hcl
   provider_installation {
     dev_overrides {
       "decentralized-infrastructure/runpod" = "/path/to/terraform-provider-runpod"
     }
     direct {}
   }
   ```

3. Run plan with the override:
   ```bash
   TF_CLI_CONFIG_FILE=~/.terraform.d/dev.tfrc tofu plan
   ```

---

## References

### Terraform Plugin Framework Documentation

- [Resources Overview](https://developer.hashicorp.com/terraform/plugin/framework/resources)
- [Schemas and Attributes](https://developer.hashicorp.com/terraform/plugin/framework/handling-data/schemas)
- [Plan Modification](https://developer.hashicorp.com/terraform/plugin/framework/resources/plan-modification)
- [UseStateForUnknown](https://pkg.go.dev/github.com/hashicorp/terraform-plugin-framework/resource/schema/stringplanmodifier#UseStateForUnknown)

### Terraform Core Concepts

- [State](https://developer.hashicorp.com/terraform/language/state)
- [Resource Lifecycle](https://developer.hashicorp.com/terraform/language/resources/behavior)

### RunPod API

- [RunPod REST API Docs](https://docs.runpod.io/reference/get-pod)
- [RunPod GraphQL API](https://graphql-spec.runpod.io/)

### Related Issues/Patterns

- [Handling Computed Attributes](https://developer.hashicorp.com/terraform/plugin/framework/handling-data/attributes#computed)
- [Common Provider Bugs](https://developer.hashicorp.com/terraform/plugin/best-practices/testing)

---

## Summary

| Issue | Symptom | Fix |
|-------|---------|-----|
| Incomplete Read | `+ field = "value"` on every plan | Populate all fields in `updateStateFromPod()` |
| Missing plan modifiers | `~ field = X -> (known after apply)` | Add `UseStateForUnknown()` to computed fields |
| Empty state + defaults | `+ field = "default"` forever | Apply defaults when API and state are both empty |
