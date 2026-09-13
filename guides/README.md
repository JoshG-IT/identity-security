# Guides

References used across the cases in this repository.

## Methodology

| Guide | Covers |
|---|---|
| [Identity Attack Paths](identity-attack-paths.md) | Common paths through identity infrastructure, what each requires, and what evidence each leaves |

## Microsoft Azure

| Guide | Best For |
|---|---|
| [Azure CLI](azure/azure-cli.md) | Resources, RBAC, Policy, deployment history, reconnaissance |
| [Azure PowerShell](azure/powershell.md) | Scripting, loops, cross-subscription queries, independent validation |
| [Microsoft Graph](azure/microsoft-graph.md) | Entra ID, users, groups, applications, service principals, sign-ins, audit |
| [Azure Portal](azure/azure-portal.md) | Visual validation, aggregated views, blocked API paths |

## On-Premises

| Guide | Best For |
|---|---|
| [Active Directory PowerShell](on-prem/active-directory-powershell.md) | Users, groups, computers, delegation, privilege, replication |
| [Group Policy](on-prem/group-policy.md) | Policy inspection, RSoP, security baseline settings |
| [LDAP and dsquery](on-prem/ldap-dsquery.md) | Precise filters, userAccountControl bit matching, module-free queries |

Guides for AWS and Google Cloud are added alongside the first case in those environments.
