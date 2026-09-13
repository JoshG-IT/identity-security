# MHI | Mad Hat Investigations

### Security Investigation Portfolio

Hands-on security investigations completed through **Mad Hat** using live, multi-user training environments.

MHI is my case-based portfolio for documenting practical security investigations across infrastructure, Microsoft Azure, AWS, and Google Cloud.

Each completed case includes sanitized evidence, investigation methodology, technical analysis, commands or queries used, findings, root-cause analysis, and security recommendations.

> **Training Environment:** These investigations are performed in hands-on training environments and are not presented as production customer incidents.

---

## Investigation Tracks

| Track | Focus | Progress | Status |
|---|---|:---:|:---:|
| [**Infrastructure Security Operations**](Infrastructure%20Security%20Operations/) | Windows, networking, forensics, Active Directory, Linux, scripting | 0 / 8 | ⏳ Pending |
| [**Azure Security Investigations**](Azure%20Security%20Investigations/) | Governance, identity, RBAC, compute, networking, storage, detection and cloud security | 1 / 10 | 🟢 Active |
| [**AWS Security Investigations**](AWS%20Security%20Investigations/) | AWS security investigation scenarios | - | ⏳ Pending |
| [**GCP Security Investigations**](GCP%20Security%20Investigations/) | Google Cloud security investigation scenarios | - | ⏳ Pending |

---

## Completed Investigations

| Case | Investigation | Track | Focus |
|---|---|---|---|
| **MHI-AZ-001** | [**Operation Dead Deploy**](Azure%20Security%20Investigations/MHI-AZ-001-operation-dead-deploy/) | Azure | Governance, ARM deployment tracing, Azure Policy and RBAC |

---

**Skills demonstrated:**

`Azure CLI` · `ARM` · `Azure Policy` · `RBAC` · `JMESPath` · `Azure PowerShell` · `Root Cause Analysis`

---

## Portfolio Progress

**1 documented investigation completed**

```text
Infrastructure Security Operations    0 / 8
Azure Security Investigations         1 / 10
AWS Security Investigations           Pending
GCP Security Investigations           Pending
```

---

## About the Environment

The investigations documented here originate from hands-on training through **Mad Hat**.

The portfolio reorganizes the concluding practical exercises into independent security investigation cases.

## Investigation Interfaces and Training Guides

I use the Mad Hat investigation scenarios as opportunities to become familiar with different command-line, scripting, API, query, and graphical interfaces across each platform.

The objective is not to use every available interface during every investigation. Instead, I use the scenarios to learn which tools are appropriate for the resource, service, identity, log source, or other data being investigated.

The interfaces used vary by platform and investigation. Detailed learning notes are maintained within each investigation track.

| Track | Interface and Training Guides |
|---|---|
| Microsoft Azure | [Azure Investigation Guides](Azure%20Security%20Investigations/Guides/) |
| Amazon Web Services | [AWS Investigation Guides](AWS%20Security%20Investigations/Guides/) |
| Google Cloud | [GCP Investigation Guides](GCP%20Security%20Investigations/Guides/) |
| Infrastructure Security Operations | [Infrastructure Security Operations](Infrastructure%20Security%20Operations/) |

### Learning Approach

Across each platform, I focus on understanding:

1. What object, service, or data source I am investigating.
2. Which interface is appropriate for retrieving or analyzing the information.
3. What service or API exists underneath that interface.
4. How to inspect and understand the raw information before filtering it.
5. How different resources, identities, permissions, and events relate to one another.
6. How to document the investigation in a repeatable and understandable way.

> **Learning Goal:** Use each investigation as an opportunity to improve both security investigation skills and familiarity with the administrative and investigative interfaces available within each platform.
