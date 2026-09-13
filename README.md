<!--
  Conventions
  Case IDs  IAM-<AZ|AWS|GCP|ONP>-NNN — numbering restarts per repo/env
  Folders   investigations/IAM-XX-NNN-slug/ — README, diagrams, docs, evidence, queries
  Guides    Guides/<platform>/ — one file per interface
  Table     completed cases only; Key Finding = the result, not the topic
  Scope     identity and access only — detection goes in security-operations
-->

# Identity Security

Security investigations covering identity, access, and governance across cloud and on-premises environments — Microsoft Entra ID, Active Directory, AWS IAM, Google Cloud IAM, and the policy and RBAC controls that govern them.

Each case includes sanitized evidence, investigation methodology, technical analysis, the commands used, findings, root-cause analysis, and security recommendations.

> **Training environments.** These investigations are performed in hands-on, multi-user training tenants and are not presented as production customer incidents. Scenario sources are credited within each case.

---

## Investigations

| Case | Environment | Investigation | Focus | Key Finding |
|---|---|---|---|---|
| **IAM-AZ-001** | Azure | [Operation Dead Deploy](investigations/IAM-AZ-001-operation-dead-deploy/) | Governance · ARM deployment tracing · Azure Policy · RBAC | Naming policy correctly detected the violation but was configured in Audit mode, recording non-compliance without preventing deployment |

---

## Skills Demonstrated

`Azure CLI` · `Azure PowerShell` · `ARM Deployment History` · `Azure Policy` · `Azure RBAC` · `JMESPath` · `Root Cause Analysis`

---

## Investigation Approach

The same process applies regardless of platform:

1. Understand the investigation objective.
2. Identify the object, service, identity, or data source involved.
3. Determine which interface is appropriate for retrieving or analyzing the information.
4. Inspect the available information before heavily filtering it.
5. Understand the object structure, IDs, scope, and relationships.
6. Filter or query the information needed for the investigation.
7. Correlate findings with additional evidence when appropriate.
8. Document the commands, evidence, findings, and conclusions.

---

## Investigation Interfaces

I use each investigation as an opportunity to learn which interface suits the resource, identity, configuration, log source, or activity being examined. The goal is not to force every case through every interface.

### Microsoft Azure

Primary focus is **Azure CLI**; the others are used where relevant.

| Interface | Best For | Guide |
|---|---|---|
| Azure CLI | Resources, RBAC, Policy, networking, tags, locks, reconnaissance | [Azure CLI](IAM-001-privileged-access-investigation/queries/azure-cli.md) |
| Azure PowerShell | Scripting, automation, loops, reusable workflows | [PowerShell](Guides/azure/powershell.md) |
| Microsoft Graph | Entra ID, users, groups, applications, service principals, sign-ins, audit data | [Microsoft Graph](Guides/azure/microsoft-graph.md) |
| KQL | Logs, telemetry, Log Analytics, Sentinel, Defender, event investigation | [KQL](Guides/azure/kql.md) |
| Azure Resource Graph | Large-scale resource discovery, inventory, filtering | [Azure Resource Graph](Guides/azure/azure-resource-graph.md) |
| Azure Portal | Visual exploration, validation, tasks better suited to a graphical interface | [Azure Portal](Guides/azure/azure-portal.md) |

#### Interface Selection — Azure

```text
What am I investigating?
        |
        +-- Azure resource, RBAC, Policy, lock, tag, or network
        |       --> Azure CLI
        |
        +-- Repeated task, scripting, or automation
        |       --> Azure PowerShell
        |
        +-- Entra ID, identity, sign-in, or directory data
        |       --> Microsoft Graph
        |
        +-- Logs, events, telemetry, or security activity
        |       --> KQL
        |
        +-- Large-scale Azure resource inventory
        |       --> Azure Resource Graph
        |
        +-- Visual exploration or validation
                --> Azure Portal
```

---

## Data Handling

These investigations document **method and reasoning**. Environment-specific identifiers, tenant and subscription IDs, object and group IDs, usernames, and resource names are redacted from public evidence.
