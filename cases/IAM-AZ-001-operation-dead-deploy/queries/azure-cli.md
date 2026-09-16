# Azure CLI Commands
## Operation Dead Deploy

Every command used during this case, in the order it was run. Environment-specific values are replaced
with placeholders.

> **Data handling:** resource group and deployment names, subscription and tenant IDs, object and group
> IDs, policy definition and assignment GUIDs, account names, and tag and parameter values carrying
> environment-specific content are replaced with placeholders. Azure service names, resource types,
> policy effect names, and API field names are retained, because they are identical in every tenant and
> the commands cannot be read without them.

---

## 1. Discover the resource group command family

```powershell
az group --help
```

**Question answered:** Which verb under `az group` returns a list of resource groups rather than a
single one?

`list` and `show` are the two candidates and they address different things: `show` returns one named
group, `list` returns the set. Enumeration across the subscription requires `list`.

---

## 2. Enumerate resource groups

```powershell
az group list -o table
```

**Question answered:** What resource groups exist in this subscription, and does any one of them break
the naming pattern?

Table output is correct here. The comparison being made is across names, and JSON would bury the
pattern in per-object noise.

---

## 3. Inspect current resource state and tags

```powershell
az resource list `
  -g <RESOURCE_GROUP> `
  -o json
```

**Question answered:** What resources exist in this group right now, and what metadata is attached to
them?

JSON rather than table, deliberately. Table output suppresses the `tags` block, which is where the
metadata attached to this resource at creation time lived.

---

## 4. List ARM deployment history

```powershell
az deployment group list `
  -g <RESOURCE_GROUP> `
  -o table
```

**Question answered:** What deployment records exist for this resource group, and what is the name of
the deployment that created its contents?

This is the pivot from current state to control-plane history. It also supplies the deployment name
required by the next command.

---

## 5. Inspect deployment parameters

```powershell
az deployment group show `
  -g <RESOURCE_GROUP> `
  -n <DEPLOYMENT_NAME> `
  --query properties.parameters
```

**Question answered:** What input values were supplied to this specific deployment when it was
submitted?

`show` is correct here because the identifier came from the preceding `list`. The projection to
`properties.parameters` skips the template, outputs, and provider blocks, which are large and not
relevant to the question.

---

## 6. Query policy evaluation state

```powershell
az policy state list `
  -g <RESOURCE_GROUP> `
  -o table `
  --query "[].{PolicyReference:policyDefinitionReferenceId, PolicyName:policyDefinitionName, Compliance:complianceState, ActionPerPolicy:policyDefinitionAction, Location:resourceLocation}"
```

**Question answered:** Which policies evaluated this resource group, what compliance state did each
return, and what action was effective?

`policyDefinitionAction` is the field that carries the effect actually applied. Projecting compliance
state without it produces a result that names a violation and omits whether anything blocked it.

Note on the output: `policyDefinitionReferenceId` is populated only for definitions evaluated as part
of an initiative. A standalone assignment returns a blank reference ID and a bare GUID under
`PolicyName`, which is exactly what the relevant record in this case did. The projection could not name
it, and the GUID had to be resolved separately.

---

## 7. Resolve the definition GUID to a policy definition

```powershell
az policy definition show `
  --name <POLICY_DEFINITION_ID>
```

**Question answered:** What rule does this definition express, and which effects does it permit?

`--name` accepts the definition GUID returned by policy state. `show` rather than `list`, because the
identifier was already known.

---

## 8. List the assignments in force at the scope

```powershell
az policy assignment list `
  -g <RESOURCE_GROUP> `
  --filter "atScope()" `
  --query "[].{DisplayName:displayName,Description:description}" `
  -o table
```

**Question answered:** Which policy assignments are actually applied over this resource group,
including any inherited from a higher scope?

**Why `list` and not `show`:** `show` addresses a single assignment at one exact scope. `--filter
"atScope()"` returns every assignment in force at the supplied scope, including assignments created at
a parent scope and inherited downward. The assignment governing this resource group was applied above
it and would not have appeared otherwise.

**What this projection does not return:** display name and description only. It confirms the assignment
is in force; it does not return the assignment's parameter values, so the effect in force was not read
from this command. It was established from `policyDefinitionAction` on the state record in section 6.

---

## 9. Read the effect directly from the assignment

```powershell
az policy assignment list `
  -g <RESOURCE_GROUP> `
  --filter "atScope()" `
  --query "[].{DisplayName:displayName, Effect:parameters.effect.value, Enforcement:enforcementMode}" `
  -o table
```

**Question answered:** Which effect value and enforcement mode is each assignment configured with at
this scope?

**Not run during this case.** Recorded here because it is the direct way to read the fact the
investigation established indirectly. `enforcementMode` is included because `DoNotEnforce` suppresses
the effect regardless of whether `Deny` is selected.

---

# JMESPath Quick Reference

```text
[0]                     first array item
[0].property            property from the first item
[].property             property from all items
[?property=='value']    filter
[].{Name:property}      custom projection
```

Projection used in this case:

```powershell
--query "[].{Compliance:complianceState, ActionPerPolicy:policyDefinitionAction}"
```

A projection is a decision about what you can see. A field that is empty for the object you are hunting
will hide it among the objects that populate it.

---

# Mental Model

```text
az resource list
= what exists in this scope now

az deployment group list
= what deployments were submitted here

az deployment group show
= what inputs one deployment received

az policy state list
= what evaluation results exist, and which effect ran

az policy definition show
= what the rule says and which effects it permits

az policy assignment list --filter "atScope()"
= which assignments are in force here, including inherited
```

Each command addresses a different object. Matching the command to the object, and the object to its
scope, is what determines whether a request returns what you expect.
