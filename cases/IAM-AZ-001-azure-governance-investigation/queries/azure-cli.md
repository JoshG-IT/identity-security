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

**Question answered:** What rule is Azure evaluating?

---

## 8. Attempt Direct Assignment Read

```powershell
az policy assignment show `
  --name <POLICY_ASSIGNMENT_NAME>
```

Observed training-tenant result:

```text
AuthorizationFailed
Microsoft.Authorization/policyAssignments/read
```

**Question answered:** Can the Reader identity directly inspect the subscription-level assignment object?

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

az policy assignment show
= assignment configuration
```