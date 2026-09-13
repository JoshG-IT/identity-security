# Technical Analysis
## Azure CLI, ARM, Azure Policy, JMESPath, and RBAC

## 1. Current State vs Deployment History

One of the most important distinctions in this investigation was:

```text
az resource list
= What exists now?

az deployment group show
= What inputs were used by a specific deployment?

az deployment group list
= What deployment records exist?
```

This prevented current-state evidence from being confused with deployment-time evidence.

---

## 2. Azure CLI Discovery Pattern

```text
--help
↓
learn available commands/options

list
↓
discover objects

show
↓
inspect one known object

--query
↓
reduce JSON to relevant evidence
```

A practical mental model:

```text
list = I do not know the object yet
show = I know the exact object I want
```

---

## 3. Resource Inventory

```powershell
az resource list `
  -g <RESOURCE_GROUP> `
  -o json
```

Useful current-state properties observed in this investigation included:

```text
createdTime
changedTime
kind
location
name
provisioningState
resourceGroup
sku
tags
type
```

---

## 4. ARM Deployment Parameters

```powershell
az deployment group show `
  -g <RESOURCE_GROUP> `
  -n <DEPLOYMENT_NAME> `
  --query properties.parameters
```

Observed parameter names:

```text
internFlag
location
operativesGroupId
```

---

## 5. ARM Deployment History

```powershell
az deployment group list `
  -g <RESOURCE_GROUP> `
  -o table
```

This exposed:

```text
Name
State
Timestamp
Mode
ResourceGroup
```

---

## 6. JMESPath

JMESPath was used to reduce Azure CLI JSON results.

```text
[0]                     first object
[0].property            property from first object
[].property             property from all objects
[?property=='value']    filter
[].{Name:property}      custom projection
```

Policy-state projection used during the investigation:

```powershell
--query "[].{PolicyReference:policyDefinitionReferenceId, PolicyName:policyDefinitionName, Compliance:complianceState, ActionPerPolicy:policyDefinitionAction, Location:resourceLocation}"
```

---

## 7. Azure Policy Object Model

```text
Policy Definition
Defines the rule
        ↓
Policy Assignment
Applies the rule to a scope and supplies configuration
        ↓
Policy State
Records the evaluation result
```

---

## 8. Policy State

```powershell
az policy state list -g <RESOURCE_GROUP>
```

The relevant record was:

```text
Compliance        = NonCompliant
ActionPerPolicy   = audit
Location          = eastus
```

---

## 9. Policy Definition

```powershell
az policy definition show `
  --name <POLICY_DEFINITION_ID>
```

Observed policy logic:

```text
displayName = Naming Convention
mode        = All
policyType  = Custom
```

Effect parameter:

```text
Audit
Deny
Disabled
```

Resource-group condition:

```text
type == Microsoft.Resources/subscriptions/resourceGroups
name notLike rg-*
```

---

## 10. Azure PowerShell Validation

```powershell
Get-AzPolicyDefinition -Name <POLICY_DEFINITION_ID>
```

The PowerShell result confirmed the same custom definition and exposed:

```text
DisplayName
Mode
PolicyType
Version
```

---

## 11. RBAC Boundary

```powershell
az policy assignment show `
  --name <POLICY_ASSIGNMENT_NAME>
```

Observed failure:

```text
AuthorizationFailed
Microsoft.Authorization/policyAssignments/read
```

This demonstrated an authorization boundary rather than a CLI syntax error.

The Reader identity could:

```text
read policy state
read the custom policy definition
identify the assignment relationship
```

but could not:

```text
directly read the subscription-level assignment object
```

---

## 12. Policy Effect Interpretation

| Effect | Behavior |
|---|---|
| `Audit` | Detects and records non-compliance while allowing the request |
| `Deny` | Rejects a non-compliant request |
| `Disabled` | Disables the policy's effect |

---

## 13. Technical Takeaway

The investigation required correlation of:

```text
Resource inventory
        +
Resource tags
        +
ARM deployment parameters
        +
ARM deployment history
        +
Policy state
        +
Policy definition
        +
Policy assignment
        +
RBAC
```

The policy engine behaved as configured. The governance weakness was the use of a detective effect where preventive enforcement would have required `Deny`.