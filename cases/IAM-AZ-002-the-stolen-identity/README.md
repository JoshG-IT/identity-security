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

> **Scope note:** The architecture diagram represents only the identities, application registrations, credentials, permissions, OAuth relationships, and redirect infrastructure relevant to this investigation. Other identities and resources in the shared tenant are intentionally omitted.

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

### Enumerating app registrations

The briefing named the flagged application but gave no identifier, so I enumerated the tenant's app registrations to obtain it.

```powershell
az ad app list -o table
```

> ![Application Inventory - Legacy App](evidence/01-app-inventory-legacy.png)
> *Highlighted: `Mad-Hat-Legacy-Sync-Service` and its `AppId`.*

The inventory returned the flagged application and the identifier needed to inspect it.

### Inspecting the legacy application object

With the identifier in hand, I inspected the application object for configuration and any metadata written during the incident.

```powershell
az ad app show `
  --id <LEGACY-APP-ID> `
  --query "{isDeviceOnlyAuthSupported:isDeviceOnlyAuthSupported,isDisabled:isDisabled,isFallbackPublicClient:isFallbackPublicClient,keyCredentials:keyCredentials,nativeAuthenticationApisEnabled:nativeAuthenticationApisEnabled,notes:notes,optionalClaims:optionalClaims}" `
```

> ![Legacy Application Entry Evidence](evidence/02-legacy-app-entry.png)
> *Highlighted: the `notes` property, redacted. Its presence confirms the application was modified during the incident.*

The `notes` property carried a value written during the incident, confirming the application had been touched. It established that the application was modified, not how. Per the briefing, entry came through a phished user whose session was stolen after MFA had been satisfied, which is consistent with the absence of alerts in the sign-in logs.

---

## 2. Escalate

### Reviewing application credentials

A stolen session expires. If the attacker intended to persist, a credential on the application is the mechanism that would outlast it, so I inspected the legacy application's password credentials.

```powershell
az ad app show `
  --id <LEGACY-APP-ID> `
  --query passwordCredentials
```

> ![Legacy Application Client Secret](evidence/03-legacy-client-secret.png)
> *Highlighted: the client secret's credential metadata, with an expiration set near the end of the century.*

One client secret was present, expiring near the end of the century. The attacker no longer needed the stolen session; the application could authenticate as itself.

---

## 3. Pivot

### Re-enumerating app registrations

A secret can be rotated. That raised the question of whether anything else held control over the legacy application, so I re-enumerated the tenant's app registrations.

```powershell
az ad app list -o table
```

> ![Application Inventory - Rogue App](evidence/04-app-inventory-rogue.png)
> *Highlighted: `Mad-Hat-Labs-App` and its `AppId`.*

The inventory returned a second registration alongside the legacy application.

### Inspecting the second registration

I inspected that application object for metadata written during the incident.

```powershell
az ad app show `
  --id <ROGUE-APP-ID> `
  --query "{isDeviceOnlyAuthSupported:isDeviceOnlyAuthSupported,isDisabled:isDisabled,isFallbackPublicClient:isFallbackPublicClient,keyCredentials:keyCredentials,nativeAuthenticationApisEnabled:nativeAuthenticationApisEnabled,notes:notes,optionalClaims:optionalClaims}" `
```

> ![Rogue Application Metadata](evidence/05-rogue-app-metadata.png)
> *Highlighted: the `notes` property on the second registration, redacted. Its presence places this application inside the incident timeline.*

The second registration also carried incident-related metadata. That is suggestive, not conclusive. A relationship to the legacy application would be.

### Querying application owners

An owner can modify the application it owns, including adding credentials. I queried the legacy application's Owners collection.

```powershell
az ad app owner list `
  --id <LEGACY-APP-ID> `
  --query "[].{Name:displayName,Id:id,CreatedDateTime:createdDateTime}" `
  -o table
```

> ![Rogue Service Principal Owns Legacy App](evidence/05a-rogue-owner-relationship.png)
> *Highlighted: the `Mad-Hat-Labs-App` service principal listed as an owner of the legacy application.*

The owners list returned the second application's service principal. That connects the two applications, and it means the attacker can mint a new secret on the legacy app whenever the current one is removed.

### Reviewing requested Graph permissions

Ownership matters in proportion to what the owned application can do, so I inspected the legacy application's requested resource access.

```powershell
az ad app show `
  --id <LEGACY-APP-ID> `
  --query requiredResourceAccess
```

> ![Legacy Application Microsoft Graph Permissions](evidence/06-legacy-graph-permissions.png)
> *Highlighted: two Microsoft Graph `resourceAccess` entries of `"type": "Role"`.*

Both entries were `"type": "Role"`, which identifies them as application permissions rather than delegated user scopes. They carried GUIDs rather than names.

### Resolving the first permission identifier

Permission names live on the resource's service principal, so I resolved each GUID against Microsoft Graph's `appRoles` collection.

```powershell
az ad sp show `
  --id <MS-GRAPH-APP-ID> `
  --query "appRoles[?id=='<PERMISSION-ID>']" `
  -o table
```

> ![Directory.Read.All Application Permission](evidence/06a-directory-read-all.png)
> *Highlighted: the first identifier resolves to `Directory.Read.All`.*

The first entry resolved to a directory-wide read permission.

### Resolving the second permission identifier

I repeated the resolution for the second identifier.

```powershell
az ad sp show `
  --id <MS-GRAPH-APP-ID> `
  --query "appRoles[?id=='<PERMISSION-ID>']" `
  -o table
```

> ![User.Read.All Application Permission](evidence/06b-user-read-all.png)
> *Highlighted: the second identifier resolves to a directory-wide user permission.*

Both permissions belong to the application identity and run without a user session. With ownership of this application, the attacker held directory-level reach.

---

## 4. Persist

### Reviewing exposed API configuration

Two access paths were established: the credential and the ownership relationship. Both are removable by an administrator who finds them. I checked whether the legacy application exposed anything that would create a third.

```powershell
az ad app show `
  --id <LEGACY-APP-ID> `
  --query api
```

> ![Legacy Application Exposed API Scope](evidence/07-legacy-api-scope.png)
> *Highlighted: an enabled custom delegated scope, `Legacy.Sync`, published under the legacy application's API configuration.*

The legacy application publishes a custom scope. Another application can request delegated access to it through user consent, giving the attacker a third path that credential rotation does not touch.

---

## 5. Loot

### Reviewing redirect URI configuration

A published scope is only useful if something requests it and the authorization response reaches infrastructure the attacker controls. I inspected the rogue application's web configuration.

```powershell
az ad app show `
  --id <ROGUE-APP-ID> `
  --query web
```

> ![Rogue Application Redirect URIs](evidence/08-rogue-web-redirect-uris.png)
> *Highlighted: a redirect URI that does not match the standard local-development pattern present alongside it.*

Two redirect URIs were configured. One was a normal local-development callback. The other pointed outside the tenant, and that is where authorization codes issued to this application land.

### Executing the consent flow

Configuration alone does not prove what the flow produces. I executed the authorization request against my own operative account to demonstrate the path end to end.

> ![OAuth Consent Flow - Step 1](evidence/09-oauth-consent-step-1.png)
> *Highlighted: the rogue application requesting delegated access to the legacy application's exposed API.*

The request surfaced as a consent prompt naming the rogue application and the scope it was asking for.

### Approving the consent request

The flow required explicit approval before the delegated permission could be issued.

> ![OAuth Consent Flow - Step 2](evidence/10-oauth-consent-step-2.png)
> *Highlighted: the approval step required before the requested delegated permission is granted.*

No authentication was requested at any point. The account was already signed in, and the only action required was approval.

### Querying OAuth permission grants

A consent screen is a browser event. Whether it wrote a durable object was a separate question, so I queried the grants held by the rogue application.

```powershell
az ad app permission list-grants `
  --id <ROGUE-APP-ID> `
  --show-resource-name true `
  -o json
```

> ![OAuth2 Permission Grant](evidence/10a-oauth2-permission-grant.png)
> *Highlighted: `consentType: Principal`, resource `Mad-Hat-Legacy-Sync-Service`, scope `Legacy.Sync`.*

The consent wrote an `OAuth2PermissionGrant` into the directory. It outlived the browser session that created it.

### Capturing the authorization response

With the grant in place, I confirmed where the resulting authorization response was delivered.

> ![Token Capture Demonstration](evidence/11-token-captured.png)
> *Highlighted: the configured callback receiving the authorization response.*

The configured redirect URI received the response. The delivery path works.

### Decoding the callback value

The response carried URL-encoded data, so I decoded it to inspect the value in readable form.

> ![CyberChef URL Decode](evidence/12-cyberchef-url-decode.png)
> *Highlighted: the decoded value carried in the callback, redacted.*

Consent wrote a persistent delegated grant into the directory, and the redirect URI sent the resulting authorization response outside the tenant. The flow required no authentication from the victim, only approval, which is why it produces no suspicious sign-in. This is a **confused deputy** pattern: a trusted application acting on a request it should never have authorized. Nothing in the chain is a vulnerability. Every component behaves as designed.

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
