# Azure PowerShell

Used for scripting, loops, and reusable workflows, and as an independent check on Azure CLI findings.

## Setup

```powershell
Connect-AzAccount
Get-AzContext
Set-AzContext -Subscription <SUBSCRIPTION_ID>
Get-AzSubscription
```

## Discovery

```powershell
Get-AzResourceGroup | Format-Table ResourceGroupName, Location
Get-AzResource -ResourceGroupName <RG> | Format-Table Name, ResourceType, Location
Get-AzResource -ResourceId <RESOURCE_ID>
```

## RBAC

```powershell
Get-AzRoleAssignment | Format-Table DisplayName, RoleDefinitionName, Scope
Get-AzRoleAssignment -ObjectId <OBJECT_ID>
Get-AzRoleAssignment -Scope "/subscriptions/<SUB_ID>/resourceGroups/<RG>"
Get-AzRoleDefinition -Name "Contributor" | Select-Object -ExpandProperty Actions
```

## Azure Policy

```powershell
Get-AzPolicyDefinition -Name <POLICY_DEFINITION_ID>
Get-AzPolicyAssignment -Scope "/subscriptions/<SUB_ID>"
Get-AzPolicyState -ResourceGroupName <RG>
```

## Deployment history

```powershell
Get-AzResourceGroupDeployment -ResourceGroupName <RG>
Get-AzResourceGroupDeployment -ResourceGroupName <RG> -Name <DEPLOYMENT> |
  Select-Object -ExpandProperty Parameters
```

## Where PowerShell wins

Objects, not text. You can filter and iterate without parsing.

```powershell
# every Owner assignment across every subscription
Get-AzSubscription | ForEach-Object {
    Set-AzContext -Subscription $_.Id | Out-Null
    Get-AzRoleAssignment |
      Where-Object RoleDefinitionName -eq 'Owner' |
      Select-Object @{n='Subscription';e={$_.Name}}, DisplayName, Scope
}

# service principal credentials older than 365 days
Get-AzADServicePrincipal | ForEach-Object {
    $sp = $_
    Get-AzADSpCredential -ObjectId $sp.Id -ErrorAction SilentlyContinue |
      Where-Object { $_.StartDateTime -lt (Get-Date).AddDays(-365) } |
      Select-Object @{n='SP';e={$sp.DisplayName}}, StartDateTime, EndDateTime
}
```

## Notes

- `Select-Object -ExpandProperty` reaches nested values. `Format-Table` is display only, never pipe it into further processing.
- `Get-Member` on any result shows what you can actually filter on.
- The `Az` module replaced `AzureRM`. Ignore `AzureRM` examples found online.
- Running the same query in both CLI and PowerShell is a legitimate way to confirm a finding is real and not a tooling artifact.
