# Azure CLI

Primary interface for Azure control-plane investigation: resources, RBAC, Policy, networking, tags, and locks.

## Setup

```bash
az login
az account show
az account set --subscription <SUBSCRIPTION_ID>
az account list -o table
```

## Discovery

```bash
az group list -o table
az resource list -g <RG> -o table
az resource list --resource-type Microsoft.Storage/storageAccounts -o table
az resource show --ids <RESOURCE_ID>
```

## RBAC

```bash
# every assignment visible at this scope, including inherited
az role assignment list --all --include-inherited -o table

# by principal
az role assignment list --assignee <OBJECT_ID> --all -o table

# by scope
az role assignment list --scope /subscriptions/<SUB_ID>/resourceGroups/<RG> -o table

# what a role actually grants
az role definition list --name "Contributor" --query "[].permissions"

# custom roles only
az role definition list --custom-role-only true -o table
```

## Azure Policy

Three separate objects. Investigating one does not give you the others.

```bash
# state: what happened when Azure evaluated resources
az policy state list -g <RG> -o table

# definition: what is the rule
az policy definition show --name <POLICY_DEFINITION_ID>

# assignment: where and how is it applied
az policy assignment show --name <ASSIGNMENT_NAME>
az policy assignment list --scope /subscriptions/<SUB_ID> -o table
```

## Deployment history

Current state and deployment-time inputs are different things.

```bash
az deployment group list -g <RG> -o table
az deployment group show -g <RG> -n <DEPLOYMENT> --query properties.parameters
az deployment group show -g <RG> -n <DEPLOYMENT> --query properties.outputs
```

## Service principals and applications

```bash
az ad sp list --display-name <NAME> -o table
az ad sp show --id <APP_ID>
az ad sp credential list --id <APP_ID> -o table    # credential age
az ad app permission list --id <APP_ID>
```

## Activity Log

```bash
az monitor activity-log list --resource-group <RG> --start-time 2026-01-01 -o table
az monitor activity-log list --caller <UPN> --start-time 2026-01-01
```

## JMESPath

`--query` runs client-side on the returned JSON. Inspect raw output first, filter second.

```bash
# project specific fields into a table
az policy state list -g <RG> -o table \
  --query "[].{Policy:policyDefinitionName, Compliance:complianceState, Action:policyDefinitionAction}"

# filter
az resource list --query "[?location=='eastus']" -o table
az role assignment list --all --query "[?roleDefinitionName=='Owner']" -o table

# nested
az ad sp show --id <APP_ID> --query "appRoles[].{Role:displayName, Enabled:isEnabled}"
```

## Notes

- `-o json` for structure, `-o table` for reading, `-o tsv` for scripting.
- `--all` matters on `role assignment list`. Without it you see only the current scope.
- Reader can read policy *state* but not always the policy *assignment*, `Microsoft.Authorization/policyAssignments/read` is a separate permission. An `AuthorizationFailed` here is an RBAC boundary, not a bad command.
- `az <group> --help` lists subcommands before you guess.
