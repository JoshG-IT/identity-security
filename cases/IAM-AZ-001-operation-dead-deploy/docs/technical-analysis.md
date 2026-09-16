# Technical Analysis
## Operation Dead Deploy

This file covers the general mechanics the case README relies on: the Azure CLI discovery pattern, the
separation between current state and deployment history, the Azure Policy object model, and how
JMESPath projections shape what an investigation is able to see. Environment-specific values are
replaced with placeholders; Azure service names, resource types, policy effect names, and API field
names are retained because they are identical in every tenant and the technique cannot be explained
without them.

---

## 1. Current State and Deployment History Are Different Objects

The most consequential distinction in this case was between what exists now and what was supplied when
it was created.

```text
az resource list
= What exists in this scope right now?

az deployment group list
= What deployment records exist for this scope?

az deployment group show
= What inputs were supplied to one named deployment?
```

Current state carries properties and tags as they stand today. A deployment record carries the
parameters as they were at submission, which survive later edits to the resource. Treating one as
evidence for the other is how a provisioning timeline gets reconstructed incorrectly.

---

## 2. The Azure CLI Discovery Pattern

```text
--help
        |
        v
learn the available verbs and options

list
        |
        v
discover objects when the target is unknown

show
        |
        v
inspect one object whose identifier is known

--query
        |
        v
reduce the returned JSON to the relevant fields
```

The practical rule:

```text
list = I do not know which object I want yet
show = I have the identifier
```

Each step in this case alternated between the two, and the identifier for each `show` came from the
preceding `list`.

---

## 3. Resource Inventory Fields

```powershell
az resource list `
  -g <RESOURCE_GROUP> `
  -o json
```

Current-state properties returned for the storage account:

```text
changedTime
createdTime
kind
location
name
provisioningState
resourceGroup
sku
tags
type
```

Table output suppresses the tag block. Requesting JSON is what made the metadata attached at creation
time visible alongside the resource properties.

---

## 4. ARM Deployment History

```powershell
az deployment group list `
  -g <RESOURCE_GROUP> `
  -o table
```

Fields returned:

```text
Name
State
Timestamp
Mode
ResourceGroup
```

`Mode` distinguishes `Incremental` from `Complete`. An `Incremental` deployment adds to whatever is
already in the resource group and leaves unlisted resources in place, so it says nothing about what
the group contained beforehand.

---

## 5. ARM Deployment Parameters

```powershell
az deployment group show `
  -g <RESOURCE_GROUP> `
  -n <DEPLOYMENT_NAME> `
  --query properties.parameters
```

Parameter names observed:

```text
location
operativesGroupId
<THIRD_PARAMETER>
```

`operativesGroupId` holds a directory group object ID. A group object ID identifies a container of
principals, not a principal. Resolving it to a membership list requires a directory read that was not
performed in this case, which is why attribution in the README rests on the `owner` tag rather than on
a conclusion drawn from this parameter.

---

## 6. JMESPath and the Cost of a Projection

```text
[0]                     first object
[0].property            property from the first object
[].property             property from all objects
[?property=='value']    filter
[].{Name:property}      custom projection
```

The projection used against policy state:

```powershell
--query "[].{PolicyReference:policyDefinitionReferenceId, PolicyName:policyDefinitionName, Compliance:complianceState, ActionPerPolicy:policyDefinitionAction, Location:resourceLocation}"
```

`policyDefinitionReferenceId` is populated only for definitions evaluated as members of an initiative.
A standalone assignment has no reference ID, so that column returns blank and `policyDefinitionName`
returns the bare definition GUID. The record that mattered most in this case was therefore the least
readable one in the output. A projection is a decision about what the investigation is able to see, and
a field that is blank for the target object is a field that hides it.

---

## 7. The Azure Policy Object Model

```text
Policy Definition
"What is the rule, and which effects may it use?"
        |
        v
Policy Assignment
"Where does the rule apply, and with which parameter values?"
        |
        v
Policy State
"What happened when this scope was evaluated?"
```

Each object is addressed differently. A definition is addressed by name or ID, an assignment exists at
a specific scope, and state is queried by scope. Matching the command to the object, and the object to
its scope, determines whether a request returns what you expect.

---

## 8. Policy State

```powershell
az policy state list -g <RESOURCE_GROUP>
```

The record for the naming control:

```text
policyDefinitionReferenceId = (empty)
policyDefinitionName        = <POLICY_DEFINITION_ID>
complianceState             = NonCompliant
policyDefinitionAction      = audit
resourceLocation            = eastus
```

Two fields on this record answer two different questions. `complianceState` says the resource was found
in breach. `policyDefinitionAction` says what the control did about it. A report that carries the first
without the second describes a violation while omitting whether anything stood in its way.

---

## 9. Policy Definition

```powershell
az policy definition show `
  --name <POLICY_DEFINITION_ID>
```

Definition properties:

```text
displayName = Naming Convention
mode        = All
policyType  = Custom
```

Effect parameter:

```text
allowedValues = Audit, Deny, Disabled
defaultValue  = Audit
```

Rule conditions:

```text
type    equals   Microsoft.Resources/subscriptions/resourceGroups
name    notLike  rg-*
then    effect   [parameters('effect')]
```

`mode` of `All` is required here. `Indexed` mode evaluates only resource types that support tags and
location, and would not evaluate a resource group at all.

The `then` block resolves the effect from a parameter rather than hard-coding one. The definition
therefore declares which effects are permitted and which is used when no value is supplied. It does not
determine which effect is in force at any given scope.

---

## 10. Policy Assignment

```powershell
az policy assignment list `
  -g <RESOURCE_GROUP> `
  --filter "atScope()" `
  --query "[].{DisplayName:displayName,Description:description}" `
  -o table
```

`show` addresses a single assignment at one exact scope. `list` with the `atScope()` filter returns
every assignment in force at the supplied scope, including assignments created at a parent scope and
inherited downward. An assignment applied above the resource group does not appear without it.

The projection used here returns display name and description only. It establishes that the
`Naming Convention` assignment is in force over the scope. It does not return the assignment's
parameter values, so the effect in force was not read from this command.

That value was established from the evaluation record instead, where `policyDefinitionAction` returned
`audit`. Reading the assignment's own parameters requires a different projection:

```powershell
az policy assignment list `
  -g <RESOURCE_GROUP> `
  --filter "atScope()" `
  --query "[].{DisplayName:displayName, Effect:parameters.effect.value, Enforcement:enforcementMode}" `
  -o table
```

That command was not run during this case. It is recorded here as the direct way to read the same fact,
rather than inferring it from the effective action on a state record.

The chain, with the source of each statement:

```text
Definition
"Audit, Deny, and Disabled are permitted; Audit is the default"
        |
        v
Assignment
"This definition is in force over this resource group"
        |
        v
State
"This resource group was evaluated NonCompliant, effective action audit"
```

Reading only the definition would have shown that `Deny` was available and left the impression that
enforcement was possible. Reading only the state would have shown a violation attributed to an unnamed
GUID. The assignment is what connected them.

---

## 11. Policy Effect Interpretation

| Effect | Behaviour | Category |
|---|---|---|
| `Audit` | Records non-compliance and allows the request | Detective |
| `Deny` | Rejects the non-compliant request | Preventive |
| `Disabled` | The rule is not evaluated | Inactive |

`enforcementMode` is a separate control from the effect. An assignment set to `DoNotEnforce` suppresses
the effect regardless of whether `Deny` is selected, which is a second way a control that appears
preventive does not act. It was not the mechanism in this case.

---

## 12. The Pattern: A Control That Reports Instead of Acting

```text
Control exists          -> yes
Control is assigned     -> yes
Control evaluates       -> yes
Control records breach  -> yes
Control blocks anything -> no
```

Every indicator a compliance review normally checks returns a positive answer. The negative sits in a
single parameter value that a policy inventory does not surface. The danger is not that the control is
absent, it is that its presence is read as enforcement.

---

## 13. Self-Reported Metadata Is Not Attribution

Two objects in this case looked like they identified a person and did not.

```text
Deployment parameter  operativesGroupId
= a directory group object ID
= a container of principals, not a principal

Resource tag          owner
= a string the provisioner typed
= no directory validation, no authentication behind it
```

Tags carry no enforcement. A tag naming an account is a claim made by whoever created the resource, and
in this case two of the four tags were populated with placeholder values, which demonstrates how little
weight the set as a whole can bear. Authoritative attribution for a control-plane write comes from
Azure Activity Log, which records the authenticated caller. That source was not queried in this case,
and the README states attribution as corroboration rather than as proof.

---

## 14. Takeaways

### Detection and enforcement are not distinguishable from the outside

`Audit` and `Deny` produce identical dashboards for a control that is present, assigned, and
evaluating. Only the effect in force at the scope separates them, and it has to be read deliberately.

### A finding requires correlation, not a single lookup

Resource inventory, tags, deployment history, deployment parameters, policy state, policy definition,
and policy assignment each answered part of the question and none answered it alone. The conclusion
came from the relationships between them.

---

## 15. Data Handling

This repository documents method and reasoning.

Redacted from public evidence and command output:

- resource group and deployment names
- resource names and resource IDs
- tenant and subscription IDs
- object and group IDs
- policy definition and policy assignment GUIDs
- usernames, service account names, and email addresses
- tag values, deployment parameter values, and policy assignment description values that carry
  environment-specific content

Retained deliberately: Azure service names, resource types, SKU and kind values, region, deployment
mode, provisioning state, policy effect names, `enforcementMode` values, and the API field names used
in projections. These are identical in every Azure tenant, are published by Microsoft, and the
technique cannot be explained without them.

Applied per stage:

| Stage | Preserved | Redacted |
|---|---|---|
| Resource group discovery | Naming pattern, region, status column | All resource group names |
| Resource inspection | Type, kind, SKU, location, provisioning state, tag keys, placeholder tag values | Resource name and ID, subscription ID, owner tag value, fourth tag value |
| Deployment history | State, mode | Deployment name, resource group name |
| Deployment parameters | Parameter names, `location` value | Third parameter value, group object ID, deployment name |
| Policy state | Compliance state, effective action, region | Definition GUIDs, resource group name |
| Policy definition | Display name, mode, policy type, effect values, rule conditions | Subscription ID, definition GUID, creator account identifiers |
| Policy assignment | Display names of built-in and Defender assignments | Assignment description value, subscription IDs in display names |

---

## 16. Detection Opportunities

Derived from this case, as candidates for a detection-engineering effort:

- A policy evaluation returning `complianceState` of `NonCompliant` where `policyDefinitionAction` is
  `audit`, which identifies violations that nothing will block
- A resource group created that does not match the subscription naming standard
- An assignment whose effect parameter is changed from `Deny` to `Audit`, or whose `enforcementMode` is
  set to `DoNotEnforce`
- A successful ARM deployment submitted by a principal holding a time-bound Contributor grant
- A resource created with required tags present but set to placeholder values

Not implemented here. Recorded as follow-on work.
