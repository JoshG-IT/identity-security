# Azure Portal

Used for visual validation, for views that aggregate several backend objects, and when an API path is blocked.

## When the Portal is the right tool

- Confirming a CLI or PowerShell finding visually before documenting it
- Policy assignment views, which combine definition, assignment, and parameters on one page
- Effective access blades that resolve inheritance for you
- Any request the API refuses but the Portal surfaces through a different backend call

## Useful locations

| Finding | Where |
|---|---|
| Role assignments and inheritance | Resource → Access control (IAM) → Role assignments |
| What a principal can actually do | Access control (IAM) → Check access |
| Policy compliance | Policy → Compliance |
| Policy assignment parameters and effect | Policy → Assignments → *(select)* → Parameters |
| Deployment history | Resource group → Deployments |
| Sign-in activity | Entra ID → Sign-in logs |
| Consent grants | Entra ID → Enterprise applications → *(app)* → Permissions |
| Directory changes | Entra ID → Audit logs |

## Portal vs API

The Portal shows one workflow where the API exposes separate objects. Policy is the clearest example:

```text
Policy Definition     "what is the rule?"
Policy Assignment     "where and how is it applied?"
Policy State          "what happened when Azure evaluated the resource?"
```

The Portal renders all three as a single compliance view. Reproducing that picture through CLI takes three
calls and correlating the results yourself, which is why the CLI version is the better demonstration of
understanding.

## Evidence

- Capture the full blade including the breadcrumb, so scope is visible.
- Annotate the specific field being cited.
- Redact tenant, subscription, and object IDs before publishing.
- Note when you moved to the Portal and why, particularly if it was because an API call was denied.

## Notes

- Portal views may aggregate several backend calls, so a value here may not map to a single API field.
- Portal permissions can differ from direct API permissions, reading an object here does not mean the equivalent API call will succeed.
