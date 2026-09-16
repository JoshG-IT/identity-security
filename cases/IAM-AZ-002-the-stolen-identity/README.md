# Azure Identity Investigation
## The Stolen Identity

> Reconstructed a five-stage OAuth consent-phishing chain across two linked Microsoft Entra app registrations, tracing a stolen session into durable application-level persistence that standard containment would not remove.

![Azure Identity Investigation Architecture](diagrams/entra-id-oauth-consent-kill-chain.png)

<p align="center">
<img src="https://img.shields.io/badge/IDENTITY_SECURITY-2B5D8C?style=for-the-badge" alt="Identity Security"/>
<img src="https://img.shields.io/badge/ENTRA_ID-2B5D8C?style=for-the-badge" alt="Entra ID"/>
<img src="https://img.shields.io/badge/AZURE_CLI-2B5D8C?style=for-the-badge" alt="Azure CLI"/>
<img src="https://img.shields.io/badge/OAUTH_2.0-2B5D8C?style=for-the-badge" alt="OAuth 2.0"/>
<img src="https://img.shields.io/badge/READ--ONLY-6E7681?style=for-the-badge" alt="Read-only"/>
</p>

<p align="center">
<img src="https://img.shields.io/badge/App_Registrations-2B5D8C?style=flat-square" alt="App Registrations"/>
<img src="https://img.shields.io/badge/Service_Principals-2B5D8C?style=flat-square" alt="Service Principals"/>
<img src="https://img.shields.io/badge/Graph_App_Permissions-2B5D8C?style=flat-square" alt="Graph Application Permissions"/>
<img src="https://img.shields.io/badge/OAuth2_Permission_Grants-2B5D8C?style=flat-square" alt="OAuth2 Permission Grants"/>
<img src="https://img.shields.io/badge/JMESPath-2B5D8C?style=flat-square" alt="JMESPath"/>
<img src="https://img.shields.io/badge/CyberChef-2B5D8C?style=flat-square" alt="CyberChef"/>
</p>

> **Reading this:** the Executive Summary and Findings cover the outcome. The Investigation walks the evidence in the order it was found. Underlying mechanics are in the Technical Drill-Down at the end.

---

## Executive Summary

An attacker reached an Entra tenant through a stolen, MFA-satisfied user session and converted it into application-level access that no longer depended on the compromised user. Working read-only through Azure CLI, I reconstructed five stages on a flagged legacy app registration: a credential set to expire near the end of the century, a second application whose service principal had been added as an owner, a custom exposed API scope, an attacker-controlled redirect URI, and a persisted OAuth consent grant.

The root cause was identity and application governance drift. Ownership, credentials, permissions, and OAuth configuration had gone unreviewed long enough that a single user compromise could anchor itself in application identities.

Containment aimed at the user would not have closed it. Resetting the password, revoking sessions, and enforcing MFA leave the credential, the ownership relationship, the exposed scope, the redirect URI, and the consent grant untouched.

---

## Briefing

An attacker entered the tenant within the previous 24 hours. No exploit was used against a software vulnerability; access came through the identity plane, and no alerts were raised. The logs showed a sequence of ordinary sign-ins.

The task was to reconstruct what the attacker did, stage by stage, using only the available read access. Every step was reported to have left evidence on a single app registration the team had flagged as tampered with: `Mad-Hat-Legacy-Sync-Service`, a legacy internal connector application.

Read-only. No application registrations, credentials, permissions, scopes, owners, or redirect URIs were modified or deleted.

---

## Environment

| Component | Details |
|---|---|
| Cloud platform | Microsoft Azure |
| Identity platform | Microsoft Entra ID |
| Environment | Live multi-user Azure tenant |
| Access level | Read-only directory application access |
| Investigation interface | Azure CLI |
| Query/filter language | JMESPath |
| Evidence source | Entra app registration and service principal objects |

---

# Investigation

## 1. Entry

The briefing named the flagged application but gave no identifier, so I enumerated the tenant's app registrations to obtain it.

```powershell
az ad app list -o table
```

The inventory returned `Mad-Hat-Legacy-Sync-Service` and its application identifier.

> ![Application Inventory - Legacy App](evidence/01-app-inventory-legacy.png)
> *Context: Application enumeration identified `Mad-Hat-Legacy-Sync-Service` and provided the `AppId` needed for deeper inspection.*

I inspected the application object for configuration and any recorded context.

```powershell
az ad app show `
  --id <LEGACY-APP-ID> `
  --query "{isDeviceOnlyAuthSupported:isDeviceOnlyAuthSupported,isDisabled:isDisabled,isFallbackPublicClient:isFallbackPublicClient,keyCredentials:keyCredentials,nativeAuthenticationApisEnabled:nativeAuthenticationApisEnabled,notes:notes,optionalClaims:optionalClaims}" `
```

The `notes` property carried a value written during the incident, confirming the application had been touched.

> ![Legacy Application Entry Evidence](evidence/02-legacy-app-entry.png)
> *Highlighted: the `notes` property on the legacy application, redacted. Its presence confirms the application was modified during the incident.*

**Finding:** the flagged application was confirmed and its identifier obtained. The `notes` property established that it had been modified, but not how. Per the incident briefing, entry came through a phished user whose session was stolen after MFA had been satisfied, which is consistent with the absence of alerts in the sign-in logs.

---

## 2. Escalate

A stolen session expires. If the attacker intended to persist, a credential on the application is the mechanism that would outlast it, so I inspected the legacy application's password credentials.

```powershell
az ad app show `
  --id <LEGACY-APP-ID> `
  --query passwordCredentials
```

One client secret was present, with an expiration set near the end of the century.

> ![Legacy Application Client Secret](evidence/03-legacy-client-secret.png)
> *Highlighted: the `credential metadata` identifies the suspicious client secret, while the 2099 expiration date indicates an effectively long-lived application credential.*

**Finding:** access derived from a compromised user session had been converted into application authentication. The application can authenticate as itself, with no human in the flow.

---

## 3. Pivot

A secret can be rotated. That raised the question of whether anything else held control over the legacy application, so I re-enumerated the tenant's app registrations.

```powershell
az ad app list -o table
```

The inventory returned a second registration, `Mad-Hat-Labs-App`.

> ![Application Inventory - Rogue App](evidence/04-app-inventory-rogue.png)
> *Context: Application enumeration identified `Mad-Hat-Labs-App` and provided the `AppId` needed for deeper inspection.*

I inspected that application object for recorded context.

```powershell
az ad app show `
  --id <ROGUE-APP-ID> `
  --query "{isDeviceOnlyAuthSupported:isDeviceOnlyAuthSupported,isDisabled:isDisabled,isFallbackPublicClient:isFallbackPublicClient,keyCredentials:keyCredentials,nativeAuthenticationApisEnabled:nativeAuthenticationApisEnabled,notes:notes,optionalClaims:optionalClaims}" `
```

Its `notes` property also carried a value written during the incident.

> ![Rogue Application Metadata](evidence/05-rogue-app-metadata.png)
> *Highlighted: the `notes` property on the second registration, redacted. Its presence places this application inside the incident timeline.*

A second application with incident-related metadata is suggestive, not conclusive. A relationship to the legacy application would be, so I queried its Owners collection.

```powershell
az ad app owner list `
  --id <LEGACY-APP-ID> `
  --query "[].{Name:displayName,Id:id,CreatedDateTime:createdDateTime}" `
  -o table
```

The owners list returned the `Mad-Hat-Labs-App` service principal.

> ![Rogue Service Principal Owns Legacy App](evidence/05a-rogue-owner-relationship.png)
> *Evidence: The legacy application's Owners collection identified `Mad-Hat-Labs-App` as an owner, establishing a direct ownership relationship between the rogue and legacy applications.*

Ownership matters in proportion to what the owned application can do, so I inspected the legacy application's requested resource access.

```powershell
az ad app show `
  --id <LEGACY-APP-ID> `
  --query requiredResourceAccess
```

Two Microsoft Graph entries of `"type": "Role"` were returned, identifying them as application permissions rather than delegated user scopes.

> ![Legacy Application Microsoft Graph Permissions](evidence/06-legacy-graph-permissions.png)
> *Evidence: The legacy application's requested resource access contained Microsoft Graph entries of type `Role`, indicating application permissions rather than delegated user scopes.*

The entries carried permission GUIDs rather than names, so I resolved each against the Microsoft Graph service principal's `appRoles` collection.

```powershell
az ad sp show `
  --id <MS-GRAPH-APP-ID> `
  --query "appRoles[?id=='<PERMISSION-ID>']" `
  -o table
```

> ![Directory.Read.All Application Permission](evidence/06a-directory-read-all.png)
> *Validation: Resolving the first Microsoft Graph app-role identifier confirmed the `Directory.Read.All` application permission.*

> ![User.Read.All Application Permission](evidence/06b-user-read-all.png)
> *Validation: Resolving the second Microsoft Graph app-role identifier confirmed the `User.Read.All` application permission.*

**Finding:** a second application's service principal was an owner of the legacy application. An owner can create new credentials, so control of the legacy application no longer depended on any single secret surviving. Because that application held broad Graph directory permissions, the control carried directory-level reach.

---

## 4. Persist

Two paths were established: the credential and the ownership relationship. Both are removable by an administrator who finds them. I checked whether the legacy application exposed anything that would create a third.

```powershell
az ad app show `
  --id <LEGACY-APP-ID> `
  --query api
```

The `oauth2PermissionScopes` collection contained a custom delegated scope published by the legacy application.

> ![Legacy Application Exposed API Scope](evidence/07-legacy-api-scope.png)
> *Highlighted: the legacy application's API configuration contains an enabled custom delegated scope, `Legacy.Sync`, establishing an additional OAuth access path.*

**Finding:** a third path existed, built on user consent rather than a credential. Credential rotation does not touch it.

---

## 5. Loot

A published scope is only useful if something requests it and the authorization response reaches infrastructure the attacker controls. I inspected the second application's web configuration.

```powershell
az ad app show `
  --id <ROGUE-APP-ID> `
  --query web
```

Multiple redirect URIs were returned: a standard local-development callback, and a second that did not match that pattern.

> ![Rogue Application Redirect URIs](evidence/08-rogue-web-redirect-uris.png)
> *Highlighted: the suspicious `redirect URI` identifies the callback destination associated with the investigated OAuth authorization flow.*

To establish what that configuration produced in practice, I followed the consent flow.

> ![OAuth Consent Flow - Step 1](evidence/09-oauth-consent-step-1.png)
>
> *Context: The consent flow showed the `rogue application` requesting delegated access to the `legacy application's` exposed API.*

> ![OAuth Consent Flow - Step 2](evidence/10-oauth-consent-step-2.png)
>
> *Context: The authorization flow required `explicit user consent` before the requested delegated permission could be granted.*

A consent screen is a browser event. Whether it wrote a durable object was a separate question, so I queried the grants held by the second application.

```powershell
az ad app permission list-grants `
  --id <ROGUE-APP-ID> `
  --show-resource-name true `
  -o json
```

The grant returned `consentType: Principal`, resource `Mad-Hat-Legacy-Sync-Service`, scope `Legacy.Sync`.

> ![OAuth2 Permission Grant](evidence/10a-oauth2-permission-grant.png)
> *Validation: The OAuth permission grant confirms that `Mad-Hat-Labs-App` received delegated access to `Mad-Hat-Legacy-Sync-Service` through the `Legacy.Sync` scope.*

The configured callback then received the authorization response.

> ![Token Capture Demonstration](evidence/11-token-captured.png)
> *Validation: The configured `OAuth callback` successfully received the authorization response, demonstrating that the `redirect path` was operational.*

I URL-decoded the redirect data to inspect the value carried in the callback.

> ![CyberChef URL Decode](evidence/12-cyberchef-url-decode.png)
> *Validation: `URL decoding` confirmed the value carried in the `callback` and allowed the authorization response to be inspected in its decoded form.*

**Finding:** consent wrote a persistent delegated grant into the directory, and the redirect URI sent the resulting authorization response outside the tenant. The flow required no authentication from the victim, only approval, which is why it produced no suspicious sign-in. This is a **confused deputy** pattern: a trusted application acting on a request it should never have authorized. Nothing in the chain is a vulnerability. Every component behaves as designed.

---

# What Surprised Me

How quickly the attack stopped depending on the person who was phished. The stolen session lasted one stage. By the third, an ownership relationship existed that could mint fresh credentials on demand, and the original account was incidental. Containing the human identity would have contained the least durable part of the intrusion.

The ownership relationship is the finding most likely to be missed. A privileged-access review that enumerates directory roles returns nothing here, because nobody was granted a role. A service principal was made an owner of an application, which is administrative capability sitting in a place administrative capability is not normally looked for. Ownership is also enumerated per application rather than per identity, so there is no single view that answers what a given service principal owns.

The consent grant was the second. A consent prompt presents as a user-interface event and is easy to treat as one. It writes a durable authorization object that outlives the browser session, the password, and the MFA method that authorized it. Five artifacts persist independently of the user account, and a containment playbook aimed at the account closes none of them.

The third was a default. In Entra ID, any standard user can register an application, and whoever registers it becomes its owner automatically. The attacker did not need elevated privilege to create the second application or to own it. That capability was already granted to every user in the tenant, and it is the precondition for the entire pivot stage.

---

# Findings and Recommendations

## Root Cause

> **A stale application ownership relationship and excessive trust in a legacy Entra application allowed a compromised user session to expand into durable application-level persistence and OAuth token harvesting.**

No single control explains the incident. It exists in the relationship between identities, applications, credentials, permissions, ownership, and consent.

| Condition | Result |
|---|---|
| MFA completed by the victim | The stolen session carried an MFA-satisfied claim |
| Stale application ownership | User compromise expanded into control over application configuration |
| Long-lived client secret | Created durable application authentication |
| Broad Graph application permissions | Increased the directory-level blast radius |
| Service principal added as owner | Created a control path that survives credential rotation |
| Custom exposed API scope | Created a consent-based access path |
| Attacker-controlled redirect URI | Directed authorization responses outside the tenant |
| OAuth permission grant | Persisted delegated authorization beyond the consent interaction |

## Recommendations

| Priority | Recommendation | Reason |
|---|---|---|
| High | Revoke unauthorized application credentials | Removes the client-secret persistence mechanism |
| High | Remove unauthorized application and service principal owners | Breaks the ability to recreate application access |
| High | Explicitly revoke malicious OAuth consent grants | User containment does not remove delegated grants |
| High | Remove attacker-controlled redirect URIs | Prevents authorization responses reaching external infrastructure |
| High | Remove unauthorized exposed API scopes | Eliminates the consent-based access path |
| High | Review and reduce Graph application permissions | Limits the blast radius of a compromised application |
| Medium | Audit app-registration Owners lists on a schedule | Ownership functions as a shadow administrative path |
| High | Disable default user application registration | Any standard user can register an application and becomes its owner automatically. This is the precondition for the pivot stage |
| Medium | Audit application ownership the way directory role membership is audited | Ownership is an unlogged privilege path outside the usual review scope |
| Medium | Alert on new application credentials and ownership changes | Detects persistence through credentials or control relationships |
| Medium | Alert on redirect URI, exposed-scope, and consent changes | Detects OAuth configuration associated with token theft |
| Medium | Enforce credential-expiration standards | Prevents effectively permanent application secrets |

> Identity incident response has to contain the compromised human identity **and** the affected application identities, credentials, ownership relationships, consent grants, scopes, and redirect URIs. Closing the account and stopping there reports the incident contained while four access paths remain open.

---

# 5 W's at a Glance

| Question | Answer |
|---|---|
| **Who** | A phished user provided the foothold; the attacker then operated through a legacy application and a second application registration |
| **What** | A five-stage OAuth consent-phishing chain ending in a persisted delegated grant |
| **When** | Within the 24-hour window stated in the briefing |
| **Where** | Microsoft Entra ID, across `Mad-Hat-Legacy-Sync-Service` and `Mad-Hat-Labs-App` |
| **Why** | Stale ownership, broad application permissions, long-lived credentials, and unreviewed OAuth configuration created durable persistence |

---

# What I Learned

- Application ownership is a shadow administrative path. It will not appear in a review of privileged directory roles, because no role was assigned.
- Application permissions are exercised by the application identity, so they survive containment aimed at the user and are not subject to Conditional Access.
- Exposed API scopes and redirect URIs are security configuration, not developer settings. The redirect URI determines where authorization codes are delivered.
- Consent grants persist as directory objects and require explicit revocation. Password resets, session revocation, and MFA enforcement do not remove them.
- Any standard user can register an application by default and becomes its owner. Application creation is not a privileged action, but application ownership is a privileged position.

---

# Technical Drill-Down

- [Technical Analysis](docs/technical-analysis.md) - Entra and OAuth mechanics, permission resolution, attack chain, detection opportunities
- [Azure CLI Commands](queries/azure-cli.md) - every command used, with purpose

---

# Data Handling

This repository documents method and reasoning. Challenge values, tenant and object identifiers, credentials, tokens, and environment-specific redirect values are redacted from public evidence. Full redaction detail is in the Technical Analysis.
