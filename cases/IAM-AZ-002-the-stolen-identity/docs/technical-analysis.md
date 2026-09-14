# Technical Analysis
## The Stolen Identity

This document explains the identity and OAuth mechanics behind the investigation without reproducing challenge answers.

---

## 1. Application Registration vs. Service Principal

A Microsoft Entra **app registration** is the application definition.

It contains configuration such as:

- application/client ID
- credentials
- requested API permissions
- exposed API scopes
- redirect URIs
- owners

A **service principal** is the application's tenant-local identity.

A useful mental model is:

```text
App Registration
"What is this application?"
        ↓
Service Principal
"What identity represents it in this tenant?"
```

The same `appId` can be used to correlate an app registration with its corresponding service principal, while each directory object has its own object ID.

This distinction became important during the Pivot stage because the owner of the legacy application was not merely another app-registration object. The returned owner object was a **service principal** associated with `Mad-Hat-Labs-App`.

---

## 2. Initial Access: MFA-Satisfied Session Theft

The attack began with a phished user who completed MFA on a fraudulent sign-in page.

The important security point is not that MFA "failed."

MFA successfully authenticated the user.

The attacker instead obtained the resulting authenticated session.

Conceptually:

```text
User enters credentials
        ↓
User completes MFA
        ↓
Authenticated session established
        ↓
Session stolen
        ↓
Attacker inherits MFA-satisfied session context
```

This demonstrates why MFA alone does not eliminate session theft risk.

The attacker did not need to defeat MFA again if they could reuse an already-authenticated session.

---

## 3. Why Application Ownership Changed the Incident

The compromised user had retained ownership of a legacy application.

Application ownership is security-sensitive because owners can influence the configuration of the application they control.

That creates a different privilege path than traditional Entra directory roles.

A review focused only on roles such as:

- Global Administrator
- Privileged Role Administrator

could miss a dangerous application owner.

The investigation later confirmed a second ownership relationship:

```text
Mad-Hat-Labs-App
        ↓
Service Principal
        ↓
Owner of
        ↓
Mad-Hat-Legacy-Sync-Service
```

This is why application ownership can act like a **shadow administrative path**.

---

## 4. Client Secret Persistence

The legacy application contained a long-lived client secret.

A client secret allows an application to authenticate programmatically through the OAuth 2.0 client credentials flow.

Conceptually:

```text
Client ID
+
Client Secret
        ↓
Token request
        ↓
Service principal authentication
        ↓
Application permissions
```

The important change is that the attacker no longer has to operate as the original human user.

The compromise shifts from:

```text
Human identity
```

to:

```text
Application identity
```

This matters because user-focused containment actions do not automatically remove application credentials.

---

## 5. Microsoft Graph Application Permissions

The legacy application's `requiredResourceAccess` configuration referenced Microsoft Graph and contained two permission entries where:

```json
"type": "Role"
```

In this context, `Role` means an **application permission / app role**.

It does not mean:

- a human user,
- an Entra directory-role assignment,
- or two separate users.

The raw `requiredResourceAccess` data did not initially expose friendly permission names. It exposed the Microsoft Graph resource App ID and permission GUIDs.

The investigation therefore resolved those GUIDs against the Microsoft Graph service principal's `appRoles` collection.

### Microsoft Graph Resource

Microsoft Graph uses the well-known application ID:

```text
00000003-0000-0000-c000-000000000000
```

### `Directory.Read.All`

Permission ID:

```text
7ab1d382-f21e-4acd-a863-ba3e13f7da61
```

Resolved through Azure CLI:

```powershell
az ad sp show `
  --id 00000003-0000-0000-c000-000000000000 `
  --query "appRoles[?id=='7ab1d382-f21e-4acd-a863-ba3e13f7da61']" `
  -o table
```

Result:

```text
Directory.Read.All
```

This application permission allows the application identity to read broad directory data.

### `User.Read.All`

Permission ID:

```text
df021288-bdef-4463-88db-98f22de89214
```

Resolved through Azure CLI:

```powershell
az ad sp show `
  --id 00000003-0000-0000-c000-000000000000 `
  --query "appRoles[?id=='df021288-bdef-4463-88db-98f22de89214']" `
  -o table
```

Result:

```text
User.Read.All
```

This application permission allows the application identity to read user profile information across the directory.

### Why This Matters

The confirmed Microsoft Graph application permissions were:

- `Directory.Read.All`
- `User.Read.All`

These permissions are exercised by the **service principal/application identity**, not by borrowing the permissions of an interactive user.

That is important because the attack had already shifted from:

```text
Compromised human session
```

to:

```text
Application identity
```

Once the attacker established durable control over the legacy application's credentials and ownership, the application's own permissions became part of the attacker's potential blast radius.

The permission-resolution process also demonstrated an important investigation technique:

```text
requiredResourceAccess
        ↓
resourceAppId
        ↓
Microsoft Graph
        ↓
resourceAccess[].id
        ↓
Microsoft Graph service principal appRoles[]
        ↓
Human-readable application permission
```

---

## 6. Ownership Pivot

A single client secret is fragile persistence.

If defenders find and rotate it, the credential is gone.

The attacker therefore established a second control path by adding the rogue application's service principal as an owner of the legacy application.

That changes the persistence model from:

```text
One credential
        ↓
If rotated, access dies
```

to:

```text
Rogue service principal
        ↓
Owns legacy application
        ↓
Can influence application configuration
        ↓
Can potentially establish fresh credentials
```

The owner relationship was directly validated with:

```powershell
az ad app owner list
```

This was one of the strongest findings in the investigation because it demonstrated a durable trust relationship rather than only a challenge artifact.

---

## 7. Exposed API Scope

The legacy application published a custom delegated OAuth scope under its API configuration.

An exposed scope allows an application to become a protected API resource that another client application can request access to.

Conceptually:

```text
Legacy application
        ↓
Exposes delegated scope
        ↓
Rogue application requests scope
        ↓
User sees consent prompt
        ↓
User grants consent
```

The custom scope created another access path independent of the original client secret.

---

## 8. Redirect URI and Authorization Code Flow

The rogue application contained redirect URI configuration.

In an OAuth authorization-code flow, the redirect URI tells Microsoft where to send the authorization response after the user authenticates and grants consent.

Simplified flow:

```text
Victim already signed in
        ↓
Rogue app requests delegated scope
        ↓
Victim accepts consent
        ↓
Authorization code issued
        ↓
Browser redirected
        ↓
Configured redirect URI receives code
        ↓
Backend exchanges code for token
```

If the redirect URI points to attacker-controlled infrastructure, the attacker can receive the authorization response.

This is why redirect URIs are security-sensitive identity configuration, not merely developer metadata.

---

## 9. OAuth2PermissionGrant

The consent flow created a delegated OAuth authorization object.

Azure CLI exposed the relationship with:

```powershell
az ad app permission list-grants
```

The investigation confirmed:

```text
consentType: Principal
resource: Mad-Hat-Legacy-Sync-Service
scope: Legacy.Sync
```

This relationship can be visualized as:

```text
User consent
        ↓
Rogue service principal
        ↓
OAuth2PermissionGrant
        ↓
Legacy application
        ↓
Legacy.Sync
```

The significant point is that consent creates a persistent authorization relationship.

It is not merely a temporary browser prompt.

---

## 10. Why Normal User Containment Is Incomplete

A standard identity incident response may include:

```text
Reset password
Revoke sessions
Require MFA
```

Those controls address the human identity.

They do not automatically remove:

```text
Application client secret
Application owner relationship
Custom exposed API scope
Redirect URI
OAuth2PermissionGrant
```

That creates a containment gap.

A more complete identity response must examine both:

```text
Human identity containment
+
Application identity containment
```

### Human Identity

- reset credentials
- revoke active sessions
- review MFA methods
- investigate suspicious sign-ins

### Application Identity

- revoke client secrets and certificates
- review app-registration owners
- review service-principal owners
- remove unauthorized redirect URIs
- review exposed API scopes
- review application permissions
- revoke malicious OAuth grants

---

## 11. Confused Deputy Pattern

The scenario demonstrates a **confused deputy** pattern.

A confused deputy occurs when a trusted component with legitimate authority is induced to perform an action on behalf of another party that should not have received that authority.

In this investigation:

```text
Legacy application
        ↓
Trusted / privileged application identity
        ↓
Rogue application obtains delegated access path
        ↓
Victim consents
        ↓
Trusted application relationship is abused
```

The danger comes from the trusted application's existing authority and the attacker's ability to manipulate how that authority is invoked.

---

## 12. Attack Chain Summary

```text
1. ENTRY
Phished user completes MFA
        ↓
Authenticated session stolen

2. ESCALATE
Legacy app ownership abused
        ↓
Long-lived client secret established
        ↓
Application-level authentication

3. PIVOT
Rogue app created
        ↓
Rogue service principal added as owner
        ↓
Durable administrative path to legacy app

4. PERSIST
Legacy app exposes custom delegated scope
        ↓
Rogue app can request delegated access

5. LOOT
Victim accepts OAuth consent
        ↓
OAuth2PermissionGrant created
        ↓
Authorization response sent to configured redirect URI
        ↓
Token access
```

---

## 13. Security Takeaways

### Applications Are Identities

App registrations and service principals should be governed with the same seriousness as human identities.

### Ownership Is Privilege

An application owner can represent meaningful administrative capability even when no privileged Entra directory role is assigned.

### MFA Is Not Complete Containment

MFA helps protect authentication, but it does not automatically invalidate:

- stolen authenticated sessions
- application credentials
- application ownership
- OAuth grants

### OAuth Configuration Is Security Configuration

The following should be treated as security controls:

- app owners
- client credentials
- Graph permissions
- exposed API scopes
- redirect URIs
- OAuth consent grants

### Investigate Permission Chains

The effective attack surface is often a relationship graph:

```text
User
↓
App registration
↓
Service principal
↓
Owner relationship
↓
Application permissions
↓
OAuth scope
↓
Consent grant
↓
Redirect URI
```

Reviewing only direct directory-role assignments can miss these indirect trust paths.

---

## 14. Recommended Detection Opportunities

This investigation suggests several useful detection targets for a future security-operations project:

- new client secret added to an application
- unusually long credential expiration
- new application owner added
- service principal added as an application owner
- new broad Microsoft Graph application permission
- new exposed API scope
- new or modified redirect URI
- new OAuth consent grant

These detections are not implemented in this case repository; they are potential follow-on work for a dedicated detection-engineering or SecOps project.
