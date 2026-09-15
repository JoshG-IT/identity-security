# Technical Analysis
## Azure CLI, ARM, Azure Policy, and JMESPath

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
  |
  v
learn available commands/options

list
  |
  v
discover objects

show
  |
  v
inspect one known object

--query
  |
  v
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
        |
        v
Policy Assignment
Applies the rule to a scope and supplies configuration
        |
        v
Policy State
Records the evaluation result
```

Each object is addressed differently. A definition is addressed by name, state is queried by scope, and an assignment exists at a specific scope. Matching the command to the object, and the object to its scope, determines whether a request returns what you expect.

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

Policy state answers what happened during evaluation. It does not by itself explain what the control was configured to do about it.

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

The definition declares which effects are permitted and which is the default. It does not determine which effect is actually in force at a given scope.

---

## 10. Policy Assignment

```powershell
az policy assignment list `
  -g <RESOURCE_GROUP> `
  --filter "atScope()" `
  -o json
```

`show` addresses a single assignment at one exact scope. `list` with the `atScope()` filter returns every assignment in force at the supplied scope, including assignments created at a parent scope and inherited downward. Because the assignment governing this resource group was applied above it, the inherited view was the one that returned it.

Result:

```text
displayName        = Naming Convention
enforcementMode    = Default
parameters.Effect  = Audit
```

The assignment is where the definition meets a scope and receives its parameter values. It is the object that answers the question the investigation was actually asking.

```text
Definition
"Audit, Deny, and Disabled are permitted; Audit is the default"
        |
        v
Assignment
"At this scope, Effect = Audit"
        |
        v
State
"This resource group was evaluated and found NonCompliant"
```

Reading only the definition would have shown that `Deny` was available and left the impression that enforcement was possible. Reading only the state would have shown the violation without explaining the outcome. The assignment supplied the missing link.

---

## 11. Policy Effect Interpretation

| Effect | Behavior |
|---|---|
| `Audit` | Detects and records non-compliance while allowing the request |
| `Deny` | Rejects a non-compliant request |
| `Disabled` | Disables the policy's effect |

---

## 12. Technical Takeaway

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
```

The policy engine behaved as configured. The governance weakness was the use of a detective effect where preventive enforcement would have required `Deny`.
