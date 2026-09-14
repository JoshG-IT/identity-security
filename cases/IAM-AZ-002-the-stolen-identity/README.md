# Azure Identity Investigation
## The Stolen Identity

> Reconstructed a five-stage OAuth consent-phishing and application-persistence chain in a **live multi-user Azure training tenant**, tracing the attack across two linked Microsoft Entra app registrations with Azure CLI.

![Azure Identity Investigation Architecture](diagrams/entra-id-oauth-consent-kill-chain.png)

> **Scope note:** The architecture diagram represents only the user identity, application registrations, credentials, permissions, OAuth relationships, and redirect infrastructure relevant to this investigation. Other identities and resources in the shared training tenant are intentionally omitted.

---

## Executive Summary

This project documents a read-only Microsoft Entra ID investigation performed in a live multi-user Azure training tenant.

The lab itself was designed around the Azure Portal. I extended the investigation by using **Azure CLI as the primary investigation interface** to enumerate app registrations, identify the two applications involved in the incident, and inspect the Entra application objects associated with each stage of the attack.

The investigation reconstructed five stages:

1. **Entry** - a phished user completed MFA and the resulting authenticated session was stolen.
2. **Escalate** - ownership of a legacy application was abused to create a long-lived client secret.
3. **Pivot** - a second attacker-created application was introduced into the trust chain while the legacy application retained powerful Microsoft Graph application permissions.
4. **Persist** - a custom delegated API scope created an additional OAuth access path.
5. **Loot** - an attacker-controlled redirect URI was used in an OAuth consent flow to capture an authorization response and obtain access through the victim's already-authenticated session.

The investigation showed that the incident was no longer just a compromised-user problem once the attacker began abusing application identities. Password resets, session revocation, and stronger MFA would not automatically remove application credentials, ownership relationships, exposed API scopes, redirect URIs, or OAuth consent grants.

The root cause was **identity and application governance drift**: stale ownership, excessive application permissions, long-lived credentials, and unreviewed OAuth configuration allowed a user-session compromise to become durable application-level persistence.

> **Environment disclosure:** This was a live multi-user Azure **training tenant**, not a production environment. Challenge answers and environment-specific identifiers are intentionally excluded from the public write-up.

---

## Scenario

The incident briefing stated that an attacker had entered the tenant within the previous 24 hours without exploiting a software vulnerability. The initial foothold came through the identity plane.

A user was phished by a fraudulent sign-in page and completed MFA. The attacker obtained the authenticated session created after MFA had been satisfied. The compromised user had also been left as an owner of a legacy internal connector application, `Mad-Hat-Legacy-Sync-Service`.

That stale ownership relationship gave the attacker a path from a compromised human identity into an application identity with significantly greater capability.

My task was to determine:

- **Who** provided the initial foothold and which application identities became involved?
- **What** application configuration enabled escalation, persistence, and token harvesting?
- **When** did the attack occur relative to the incident window?
- **Where** did the attack artifacts exist across the legacy and rogue app registrations?
- **Why** would normal user-focused containment fail to completely remove the attacker's access?

The investigation was performed in **observe mode**. No application registrations, credentials, permissions, scopes, owners, or redirect URIs were modified or deleted.

---

## Environment

| Component | Details |
|---|---|
| Cloud platform | Microsoft Azure |
| Identity platform | Microsoft Entra ID |
| Environment | Live multi-user Azure training tenant |
| Investigation access | Read-only directory application access |
| Primary investigation tool | Azure CLI |
| Query/filter language | JMESPath |
| Primary evidence source | Microsoft Entra app registration objects |
| Supporting validation | Microsoft OAuth consent flow |
| URL decoding | CyberChef |
| Applications investigated | `Mad-Hat-Legacy-Sync-Service` and `Mad-Hat-Labs-App` |
| Investigation mode | Read-only |

---

# Investigation

## 1. Entry - Compromised Session and Legacy Application

I first enumerated the tenant's app registrations to locate the legacy application identified in the incident briefing.

```powershell
az ad app list -o table
```

The application inventory exposed `Mad-Hat-Legacy-Sync-Service` and provided the application identifier needed for deeper inspection.

> ![Application Inventory - Legacy App](evidence/01-app-inventory-legacy.png)

I then inspected the legacy application object and reviewed its internal notes metadata.

```powershell
az ad app show `
  --id <LEGACY-APP-ID> `
  -o json
```

The notes documented that the initial compromise began with a phished user and an authenticated session obtained after MFA had already been satisfied.

> ![Legacy Application Entry Evidence](evidence/02-legacy-app-entry.png)

**What I concluded:** the attacker did not begin by compromising an Azure resource. The foothold originated from a human identity, and stale ownership of the legacy application turned that user-session compromise into an application-security incident.

---

## 2. Escalate - Long-Lived Client Secret

I inspected the legacy application's password credentials.

```powershell
az ad app show `
  --id <LEGACY-APP-ID> `
  --query passwordCredentials
```

The credential metadata showed a client secret with an expiration date set near the end of the century.

> ![Legacy Application Client Secret](evidence/03-legacy-client-secret.png)

Creating a client secret changed the nature of the compromise. The attacker no longer needed to repeatedly authenticate as the phished user. The application could authenticate programmatically as its service principal through the client credentials flow.

**What I concluded:** the attacker converted temporary access derived from a compromised user session into durable application-level authentication.

---

## 3. Pivot - Rogue Application and Application Privilege

A single client secret could eventually be discovered and rotated, so the attacker introduced a second application registration.

I enumerated the app registrations again to identify the attacker-created application, `Mad-Hat-Labs-App`.

```powershell
az ad app list -o table
```

> ![Application Inventory - Rogue App](evidence/04-app-inventory-rogue.png)

I then inspected the rogue application object and its metadata.

```powershell
az ad app show `
  --id <ROGUE-APP-ID> `
  -o json
```

The application metadata tied the rogue registration to the persistence chain.

> ![Rogue Application Metadata](evidence/05-rogue-app-metadata.png)

### Supporting Permission Evidence

I also inspected the legacy application's requested resource access to understand why control of the legacy app mattered.

```powershell
az ad app show `
  --id <LEGACY-APP-ID> `
  --query requiredResourceAccess
```

> ![Legacy Application Microsoft Graph Permissions](evidence/06-legacy-graph-permissions.png)

The output contained two `resourceAccess` entries with:

```json
"type": "Role"
```

and a `resourceAppId` corresponding to Microsoft Graph.

These are **not two users** and they are not two Entra directory-role assignments. Each `Role` entry represents a Microsoft Graph **application permission (app role)** requested by the legacy application. In this scenario, the two permissions were:

- `Directory.Read.All`
- `User.ReadWrite.All`

Those permissions increased the blast radius because the application could operate with directory-level privileges independently of the originally compromised user's interactive session.

**What I concluded:** the attacker created a second application identity to strengthen persistence, while the legacy application's Microsoft Graph application permissions made continued control of that application especially valuable.

---

## 4. Persist - Custom Exposed API Scope

I returned to the legacy application and inspected its API configuration.

```powershell
az ad app show `
  --id <LEGACY-APP-ID> `
  --query api
```

The `oauth2PermissionScopes` collection contained a custom delegated scope published by the legacy application.

> ![Legacy Application Exposed API Scope](evidence/07-legacy-api-scope.png)

Publishing an API scope allows another application to request delegated access to the legacy application as a protected resource.

That created a second persistence path that did not depend entirely on the original client secret.

**What I concluded:** the attacker established an OAuth-based backup path that could remain useful even if the original application credential was discovered and removed.

---

## 5. Loot - OAuth Consent and Token Capture

The final stage centered on the rogue application's redirect URI configuration.

I inspected the rogue application's web settings.

```powershell
az ad app show `
  --id <ROGUE-APP-ID> `
  --query web
```

The output showed multiple redirect URIs, including a normal local-development callback and a second URI associated with the attack flow.

> ![Rogue Application Redirect URIs](evidence/08-rogue-web-redirect-uris.png)

I then followed the OAuth consent flow used by the scenario.

The first consent screen showed the rogue application requesting access associated with the legacy application's exposed API.

> ![OAuth Consent Flow - Step 1](evidence/09-oauth-consent-step-1.png)

The second step required the user to explicitly accept the requested permissions.

> ![OAuth Consent Flow - Step 2](evidence/10-oauth-consent-step-2.png)

After consent, the authorization response was redirected to the configured callback and the scenario demonstrated successful token capture.

> ![Token Capture Demonstration](evidence/11-token-captured.png)

Finally, I used CyberChef to URL-decode the redirect data and validate the encoded value carried in the callback.

> ![CyberChef URL Decode](evidence/12-cyberchef-url-decode.png)

The attack path can be summarized as:

```text
Victim already authenticated
        ↓
MFA already satisfied
        ↓
Rogue application consent request
        ↓
Legacy application's exposed API scope
        ↓
User accepts consent
        ↓
Authorization response
        ↓
Attacker-controlled redirect URI
        ↓
Token access through the OAuth flow
```

**What I concluded:** OAuth consent allowed the attacker to leverage a victim who was already authenticated on a trusted session instead of forcing the attacker to perform another suspicious interactive sign-in.

---

# What Broke / What Surprised Me

What surprised me most was how quickly the attack stopped depending on the original phished user.

The initial compromise was a stolen authenticated session, but once the attacker could abuse ownership of the legacy application, they could create a client secret and begin operating through the application identity instead. At that point, the incident was no longer solved by treating the user account as the only compromised identity.

The second major surprise was the persistence chain. A defender could reset the user's password, revoke every active user session, and strengthen MFA, yet still leave behind:

- the application client secret,
- the rogue application relationship,
- the custom exposed API scope,
- the attacker-controlled redirect URI,
- and the OAuth consent grant.

The most important lesson was that **identity containment has to include application identities and OAuth trust relationships**. A standard user-account containment playbook can succeed against the human account while leaving the attacker's application persistence intact.

I was also surprised by how easy it would be to miss the privilege path during a normal administrative review. Looking only at highly privileged directory roles would not tell the whole story. An application owner with the ability to modify credentials and configuration can represent a shadow administrative path even though that privilege does not look like a traditional Global Administrator assignment.

---

# Findings and Recommendations

## Root Cause

> **A stale application ownership relationship and excessive trust in a legacy Entra application allowed a compromised user session to expand into durable application-level persistence and OAuth token harvesting.**

The attack succeeded because several individually manageable identity risks had accumulated:

- a compromised authenticated user session,
- stale ownership of a legacy application,
- powerful Microsoft Graph application permissions,
- a long-lived client secret,
- a second attacker-created application,
- a custom exposed API scope,
- and an attacker-controlled redirect URI.

No single control explained the entire incident. The attack existed in the **relationship between identities, applications, credentials, permissions, and OAuth consent**.

| Condition / Control | Result |
|---|---|
| MFA completed by the victim | The stolen session already represented an MFA-satisfied authentication |
| Stale legacy-app ownership | User compromise expanded into control over application configuration |
| Long-lived client secret | Created durable application authentication |
| Powerful Graph application permissions | Increased the directory-level blast radius of the legacy app |
| Rogue application | Added a second attacker-controlled identity to the persistence chain |
| Custom exposed API scope | Created a delegated OAuth access path |
| Attacker-controlled redirect URI | Directed authorization responses to attacker infrastructure |
| OAuth consent grant | Could remain after password reset, session revocation, and stronger MFA |

## Recommendations

| Priority | Recommendation | Reason |
|---|---|---|
| High | Revoke and remove unauthorized application credentials | Removes the client-secret persistence mechanism |
| High | Remove unauthorized application and service principal owners | Breaks the attacker's ability to maintain or recreate application access |
| High | Explicitly revoke malicious OAuth consent grants | User password and session containment do not automatically remove OAuth grants |
| High | Remove attacker-controlled redirect URIs | Prevents authorization responses from reaching attacker infrastructure |
| High | Remove unauthorized exposed API scopes | Eliminates the delegated access path created for persistence |
| High | Review and reduce Microsoft Graph application permissions | Limits the blast radius of a compromised application |
| Medium | Audit app-registration Owners lists regularly | Application ownership can function as a shadow administrative path |
| Medium | Restrict who can register applications where business requirements allow | Reduces opportunities to create rogue applications |
| Medium | Alert on new client secrets and certificates | Detects new application credentials used for persistence |
| Medium | Alert on redirect URI and API-scope changes | Detects OAuth configuration changes associated with token theft |
| Medium | Enforce credential-expiration standards | Prevents effectively permanent application secrets |

> Identity incident response should contain both the compromised **human identity** and any affected **application identities, credentials, ownership relationships, consent grants, scopes, and redirect URIs**.

---

# 5 W's at a Glance

| Question | Answer |
|---|---|
| **Who** | A phished user provided the initial foothold; the attacker then operated through a legacy application and a rogue app registration |
| **What** | A five-stage OAuth consent-phishing and application-persistence chain |
| **When** | The incident briefing placed the compromise within the previous 24 hours |
| **Where** | Microsoft Entra ID across `Mad-Hat-Legacy-Sync-Service` and `Mad-Hat-Labs-App` |
| **Why** | Stale ownership, excessive application permissions, long-lived credentials, and unreviewed OAuth configuration created a durable trust path |

---

# What I Learned

- A compromised user session can become an application-identity compromise if the user owns an app registration.
- MFA does not remove risk from an already-authenticated stolen session.
- Client secrets allow an attacker to move from human interactive access to programmatic application authentication.
- Application owners can represent a shadow administrative path that will not appear in a simple review of privileged directory roles.
- `requiredResourceAccess` exposes the permissions an application requests from resource APIs such as Microsoft Graph.
- A `resourceAccess` entry with `"type": "Role"` represents an application permission/app role, not a user or directory-role assignment.
- Exposed API scopes and redirect URIs are security-sensitive identity configuration, not just developer settings.
- OAuth consent grants require explicit investigation and revocation during identity incident response.
- Password resets, session revocation, and MFA enforcement do not automatically remove application credentials or OAuth persistence.
- `az ad app list` and `az ad app show` can expose most of the app-registration evidence needed to reconstruct an Entra application attack.
- JMESPath is useful for isolating security-relevant properties from large application JSON objects.
- The overall attack demonstrates a **confused deputy** pattern in which a trusted application can be used to perform actions through access that should not have been authorized.

---

# Technical Drill-Down

For the deeper technical material:

- [Technical Analysis](docs/technical-analysis.md)
- [Azure CLI Commands](queries/azure-cli.md)

---

# Tools and Services

- Microsoft Azure
- Microsoft Entra ID
- Azure CLI
- App registrations
- Service principals
- Microsoft Graph application permissions
- OAuth 2.0
- JMESPath
- PowerShell
- Microsoft OAuth consent flow
- CyberChef

---

# Data Handling

This repository intentionally documents the **investigation method and reasoning**, not the course answer key.

The following are redacted from public screenshots and command output:

- `MadHat{...}` challenge values
- usernames and email addresses
- operative identifiers
- tenant IDs
- application/client IDs where environment-specific
- object IDs and service principal IDs
- credential key IDs
- authorization codes and tokens
- client secret values
- challenge-specific redirect URI values
- encoded challenge answers
- any other value that functions as a lab answer

For this case specifically:

- Objective 1 screenshots should redact the challenge value contained in the legacy application's notes.
- Objective 2 should preserve the suspicious expiration date while redacting the challenge-bearing credential description.
- Objective 3 should redact challenge values from the rogue app metadata. The Microsoft Graph permission-ID screenshot may remain if it contains no challenge values or environment-specific identifiers that need removal.
- Objective 4 should show that a custom API scope exists while redacting the challenge-bearing consent display name.
- Objective 5 redirect-URI and CyberChef screenshots should redact the challenge value while preserving the structure of the OAuth flow.
- The token-capture demonstration should not expose reusable tokens, authorization codes, or other live credentials.

---

## Resume Line

> Reconstructed a five-stage OAuth consent-phishing kill chain in a live Azure training tenant using Azure CLI; traced the attack across two linked Entra app registrations through a stolen MFA-satisfied session, a long-lived client secret, powerful Microsoft Graph application permissions, a rogue application pivot, a custom exposed API scope, and an attacker-controlled redirect URI, then delivered identity- and OAuth-focused remediation recommendations.
