# Azure PowerShell Validation

Azure PowerShell was used as a secondary validation interface. Azure CLI remained the primary command-line investigation tool.

---

## Policy Definition

```powershell
Get-AzPolicyDefinition -Name <POLICY_DEFINITION_ID>
```

The screenshot supplied for this investigation exposed:

- DisplayName: `Naming Convention`
- Mode: `All`
- PolicyType: `Custom`
- Version: `1.0.0`

---

## Policy State

A related validation command used during the investigation was:

```powershell
Get-AzPolicyState -ResourceGroupName <RESOURCE_GROUP>
```

Useful fields include:

```text
PolicyAssignmentName
PolicyAssignmentId
PolicyAssignmentScope
PolicyDefinitionName
PolicyDefinitionAction
ComplianceState
ResourceId
ResourceType
Timestamp
```

---

## Policy Assignment

The Azure PowerShell equivalent of the assignment read is:

```powershell
Get-AzPolicyAssignment `
  -Name <POLICY_ASSIGNMENT_NAME> `
  -Scope <POLICY_ASSIGNMENT_SCOPE>
```

The same Azure RBAC model applies regardless of whether the request comes from Azure CLI or Azure PowerShell.

---

## Interface Mapping

| Investigation Question | Azure CLI | Azure PowerShell |
|---|---|---|
| What policy evaluations exist? | `az policy state list` | `Get-AzPolicyState` |
| What is the rule? | `az policy definition show` | `Get-AzPolicyDefinition` |
| How is it assigned? | `az policy assignment show` | `Get-AzPolicyAssignment` |