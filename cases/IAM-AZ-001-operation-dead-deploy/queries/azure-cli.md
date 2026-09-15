# Azure CLI Investigation Commands

These are the Azure CLI command patterns used during the investigation. Environment-specific answers are replaced with placeholders.

---

## 1. Discover Resource-Group Commands

```powershell
az group --help
```

---

## 2. Enumerate Resource Groups

```powershell
az group list -o table
```

**Question answered:** What resource groups exist, and which one breaks the naming pattern?

---

## 3. Inspect Current Resource State and Tags

```powershell
az resource list `
  -g <RESOURCE_GROUP> `
  -o json
```

**Question answered:** What resources exist in this group **right now**, and what metadata/tags are attached?

---

## 4. Inspect Deployment Parameters

```powershell
az deployment group show `
  -g <RESOURCE_GROUP> `
  -n <DEPLOYMENT_NAME> `
  --query properties.parameters
```

**Question answered:** What values were passed into this specific deployment?

---

## 5. List ARM Deployment History

```powershell
az deployment group list `
  -g <RESOURCE_GROUP> `
  -o table
```

**Question answered:** What deployment records exist for this resource group?

---

## 6. Query Policy State

```powershell
az policy state list `
  -g <RESOURCE_GROUP> `
  -o table `
  --query "[].{PolicyReference:policyDefinitionReferenceId, PolicyName:policyDefinitionName, Compliance:complianceState, ActionPerPolicy:policyDefinitionAction, Location:resourceLocation}"
```

**Question answered:** What policy evaluations apply, what is their compliance state, and what action was effective?

---

## 7. Inspect the Policy Definition

```powershell
az policy definition show `
  --name <POLICY_DEFINITION_ID>
```

**Question answered:** What rule is Azure evaluating, and which effects does it permit?

---

## 8. Read the Policy Assignment

```powershell
az policy assignment list `
  -g <RESOURCE_GROUP> `
  --filter "atScope()" `
  -o json
```

**Question answered:** At what scope is the rule applied, and which effect is supplied there?

**Why `list` and not `show`:** `show` addresses a single assignment at one exact scope. `--filter "atScope()"` returns every assignment in force at the supplied scope, including assignments created at a parent scope and inherited downward. An assignment applied above the resource group will not appear without it.

---

# JMESPath Quick Reference

```text
[0]                     first array item
[0].property            property from first item
[].property             property from all items
[?property=='value']    filter
[].{Name:property}      custom projection
```

Example:

```powershell
--query "[].{Compliance:complianceState,Effect:policyDefinitionAction}"
```

---

# Investigation Mental Model

```text
az resource list
= current resource state

az deployment group list
= deployment history

az deployment group show
= one deployment's details/inputs

az policy state list
= policy evaluation results

az policy definition show
= policy rule

az policy assignment list --filter "atScope()"
= assignments in force at a scope, including inherited
```

Each of these addresses a different object. Matching the command to the object, and the object to its scope, is what determines whether a request returns what you expect.
