<!--
  Conventions
  Case IDs  IAM-<AZ|AWS|GCP|ONP>-NNN; numbering restarts per platform
  Folders   cases/<CASE-ID>-slug/ holds README + evidence always; diagrams, docs, queries as needed
  Guides    Guides/<platform>/ holds one file per interface
  Table     completed cases only; Key Finding = the result, not the topic
  Type      how far the arc went: Investigation, or Investigation → Implementation → Validation
  Scope     identity and access only; detection goes in security-operations
-->

# Identity Security

Security casework covering identity, access, and governance across cloud and on-premises environments: Microsoft Entra ID, Active Directory, AWS IAM, Google Cloud IAM, and the RBAC and policy controls that govern them.

Each case includes sanitized evidence, methodology, technical analysis, the commands used, findings, root-cause analysis, and recommendations.

> **How to read the Type column.** Cases in training environments are read-only, so they cover investigation and recommendation: the scope an IAM or SOC analyst actually works in. Cases in my own lab carry the full arc: investigate, implement, validate. Environment and access level are stated in every case README.

---

## Cases

| Case | Investigation | Type | Environment | Key Finding |
|---|---|---|---|---|
| **IAM-AZ-001** | [Operation Dead Deploy](cases/IAM-AZ-001-operation-dead-deploy/) | Investigation | Azure · training tenant · read-only | Naming policy correctly detected the violation but ran in Audit mode, recording non-compliance without preventing deployment |

---

## Skills Demonstrated

`Azure CLI` · `Azure PowerShell` · `Microsoft Graph` · `ARM Deployment History` · `Azure Policy` · `Azure RBAC` · `JMESPath` · `Active Directory` · `Group Policy` · `LDAP` · `Root Cause Analysis`

---

## Approach

The same process applies regardless of platform or access level:

1. Understand the objective.
2. Identify the object, service, identity, or data source involved.
3. Determine which interface is appropriate for retrieving or analyzing the information.
4. Inspect the available information before heavily filtering it.
5. Understand the object structure, IDs, scope, and relationships.
6. Filter or query the information needed.
7. Correlate findings with additional evidence.
8. Document the commands, evidence, findings, and conclusions.

Where I have write access, the case continues: implement the recommendation, validate that it works, and confirm it does not break legitimate activity.

---

## Investigation Interfaces

I use each case as an opportunity to learn which interface suits the resource, identity, configuration, or activity being examined. The goal is not to force every case through every interface.

### Microsoft Azure

Primary focus is **Azure CLI**; the others are used where relevant.

| Interface | Best For | Guide |
|---|---|---|
| Azure CLI | Resources, RBAC, Policy, deployment history, reconnaissance | [Azure CLI](Guides/azure/azure-cli.md) |
| Azure PowerShell | Scripting, loops, cross-subscription queries, independent validation | [Azure PowerShell](Guides/azure/powershell.md) |
| Microsoft Graph | Entra ID, users, groups, applications, service principals, sign-ins, audit | [Microsoft Graph](Guides/azure/microsoft-graph.md) |
| Azure Portal | Visual validation, aggregated views, blocked API paths | [Azure Portal](Guides/azure/azure-portal.md) |

```text
What am I investigating?
        |
        +-- Azure resource, RBAC, Policy, lock, tag, or network
        |       --> Azure CLI
        |
        +-- Repeated task, scripting, or cross-subscription query
        |       --> Azure PowerShell
        |
        +-- Entra ID, identity, sign-in, or directory data
        |       --> Microsoft Graph
        |
        +-- Visual validation, or an API path that is blocked
                --> Azure Portal
```

### On-Premises

| Interface | Best For | Guide |
|---|---|---|
| Active Directory PowerShell | Users, groups, computers, delegation, privilege, replication | [Active Directory PowerShell](Guides/on-prem/active-directory-powershell.md) |
| Group Policy | Policy inspection, RSoP, security baseline settings | [Group Policy](Guides/on-prem/group-policy.md) |
| LDAP and dsquery | Precise filters, userAccountControl bit matching, module-free queries | [LDAP and dsquery](Guides/on-prem/ldap-dsquery.md) |

```text
What am I investigating?
        |
        +-- Users, groups, computers, delegation, or privilege
        |       --> Active Directory PowerShell
        |
        +-- Applied configuration, security baseline, or audit settings
        |       --> Group Policy
        |
        +-- Precise attribute filtering, or no AD module available
                --> LDAP / dsquery
```

<!-- new platform sections mirror the blocks above -->

Guides for AWS and Google Cloud are added alongside the first case in those environments.

---

## Data Handling

These cases document method and reasoning. Tenant and subscription IDs, object and group IDs, policy and assignment GUIDs, usernames, email addresses, and resource names that expose environment-specific information are redacted from public evidence.
