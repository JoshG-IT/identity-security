<!--
  Conventions
  Case IDs  IAM-<AZ|AWS|GCP|ONP>-NNN — numbering restarts per platform
  Folders   cases/<CASE-ID>-slug/ — README + evidence always; diagrams, docs, queries as needed
  Guides    Guides/<platform>/ — one file per interface
  Table     completed cases only; Key Finding = the result, not the topic
  Type      shows how far the arc went: Investigation, or Investigation → Implementation → Validation
  Scope     identity and access only — detection goes in security-operations
-->

# Identity Security

Security casework covering identity, access, and governance across cloud and on-premises environments — Microsoft Entra ID, Active Directory, AWS IAM, Google Cloud IAM, and the RBAC and policy controls that govern them.

Each case includes sanitized evidence, methodology, technical analysis, the commands used, findings, root-cause analysis, and recommendations.

> **How to read the Type column.** Cases in training environments are read-only, so they cover investigation and recommendation — the scope a SOC or IAM analyst actually works in. Cases in my own lab carry the full arc: investigate, implement, validate. Environment and access level are stated in every case README.

---

## Cases

| Case | Investigation | Type | Environment | Key Finding |
|---|---|---|---|---|
| **IAM-AZ-001** | [Operation Dead Deploy](cases/IAM-AZ-001-operation-dead-deploy/) | Investigation | Azure · training tenant · read-only | Naming policy correctly detected the violation but ran in Audit mode, recording non-compliance without preventing deployment |

---

## Skills Demonstrated

`Azure CLI` · `Azure PowerShell` · `Microsoft Graph` · `ARM Deployment History` · `Azure Policy` · `Azure RBAC` · `JMESPath` · `Root Cause Analysis`

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

I use each case as an opportunity to learn which interface suits the resource, identity, configuration, log source, or activity being examined. The goal is not to force every case through every interface.

### Microsoft Azure

Primary focus is **Azure CLI**; the others are used where relevant.

| Interface | Best For | Guide |
|---|---|---|
| Azure CLI | Resources, RBAC, Policy, networking, tags, locks, reconnaissance | [Azure CLI](investigations/IAM-001-privileged-access-investigation/queries/azure-cli.md) |
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
        +-- Visual exploration or validation
                --> Azure Portal
```

<!-- new platform sections mirror the Azure block above -->

Platform guides for AWS, Google Cloud, and on-premises environments are added as cases in those environments are completed.

---

## Data Handling

These cases document **method and reasoning**, not training answer keys. Challenge values, environment-specific identifiers, tenant and subscription IDs, object and group IDs, usernames, and resource names are redacted from public evidence.

---

Scenario sources are credited within each case. All analysis, evidence collection, findings, and documentation are my own work.
