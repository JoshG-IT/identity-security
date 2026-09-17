# Technical Analysis
## The Stolen Identity

The Entra ID and OAuth mechanics behind the investigation. Environment-specific values and Microsoft's published identifiers alike are replaced with placeholders. The identifiers are documented by Microsoft and can be looked up by anyone working through the same technique; substituting them keeps that step as work the reader does rather than a value this file hands over.

---

## 1. App Registration vs Service Principal

An **app registration** is the application definition:

- application/client ID
- credentials
- requested API permissions
- exposed API scopes
- redirect URIs
- owners

A **service principal** is the application's tenant-local identity.

```text
App Registration
"What is this application?"
        |
        v
Service Principal
"What identity represents it in this tenant?"
```

The same `appId` correlates the two, but each is a separate directory object with its own object ID.

This mattered in the Pivot stage. The owner returned on the legacy application was not another app registration. It was a **service principal**, which is what made the relationship an active control path rather than a naming artifact.

---

## 2. Why a Stolen Session Defeats MFA Without Defeating It

```text
User enters credentials
        |
        v
User completes MFA
        |
        v
Authenticated session established
        |
        v
Session stolen
        |
        v
Attacker inherits MFA-satisfied session context
```

MFA authenticated the user correctly. The attacker did not bypass it, race it, or phish a code for reuse. They took the artifact MFA produces.

The consequence is that MFA does not need to be defeated again. An already-authenticated session carries its own proof of authentication, which is why the later consent flow produced no suspicious sign-in event.

---

## 3. Application Ownership as Privilege

An application owner can modify the configuration of the application they own, including adding credentials.

That is administrative capability, but it is not a directory role. A privileged-access review that enumerates:

- Global Administrator
- Privileged Role Administrator
- Application Administrator

returns nothing for an owner relationship, because no role was assigned.

The relationship confirmed here:

```text
<ROGUE-APP-NAME>
        |
        v
Service Principal
        |
        v
Owner of
        |
        v
<LEGACY-APP-NAME>
```

Ownership is enumerated per application, not per identity. There is no single view that answers "what does this service principal own," which is part of why the path is easy to miss.

---

## 4. Client Secret Persistence

A client secret enables the OAuth 2.0 **client credentials flow**:

```text
Client ID
    +
Client Secret
        |
        v
Token request
        |
        v
Service principal authentication
        |
        v
Application permissions
```

No user is involved. The application authenticates as itself and exercises its own permissions.

The compromise therefore shifts from:

```text
Human identity
```

to:

```text
Application identity
```

User-focused containment does not reach the second one. Resetting a password has no effect on a credential held by an application.

---

## 5. Microsoft Graph Application Permissions

The legacy application's `requiredResourceAccess` contained two Microsoft Graph entries with:

```json
"type": "Role"
```

In `requiredResourceAccess`, `Role` means an **application permission / app role**.

It does not mean:

- a human user
- an Entra directory-role assignment
- two separate users

### Delegated vs application permissions

| | Delegated | Application |
|---|---|---|
| Acts as | The signed-in user | The application itself |
| Requires a user session | Yes | No |
| Effective permission | Intersection of user rights and scope | The permission as granted |
| Subject to Conditional Access | Yes | No |
| Consent | User or admin | Admin only |

Application permissions are the higher-risk grant, and they are exactly what survives a session revocation.

### Resolving permission GUIDs

`requiredResourceAccess` returns identifiers, not names. The names live on the resource's service principal.

Microsoft Graph has a well-known application ID that is the same in every tenant and is published in Microsoft's documentation. It is represented here as `<MS-GRAPH-APP-ID>`.

```powershell
az ad sp show `
  --id <MS-GRAPH-APP-ID> `
  --query "appRoles[?id=='<PERMISSION-ID>']" `
  -o table
```

Resolved in this investigation:

```text
<DIRECTORY-READ-ALL-PERMISSION-ID>  ->  Directory.Read.All
<USER-READ-ALL-PERMISSION-ID>       ->  User.Read.All
```

The resolution chain:

```text
requiredResourceAccess
        |
        v
resourceAppId          which API is being requested
        |
        v
Microsoft Graph
        |
        v
resourceAccess[].id    which permission, as a GUID
        |
        v
appRoles[] on the Graph service principal
        |
        v
Human-readable application permission
```

Skipping this step leaves the permission set unreadable, and an unreadable permission set is usually recorded as "some Graph permissions" rather than as a finding.

---

## 6. The Ownership Pivot

A single credential is fragile persistence:

```text
One credential
        |
        v
If rotated, access ends
```

Ownership is not:

```text
Service principal
        |
        v
Owns the application
        |
        v
Can modify its configuration
        |
        v
Can create fresh credentials
```

Rotating the secret removes one credential. It does not remove the ability to create another.

Confirmed with:

```powershell
az ad app owner list `
  --id <LEGACY-APP-ID> `
  --query "[].{Name:displayName,Id:id,CreatedDateTime:createdDateTime}" `
  -o table
```

---

## 7. Exposed API Scopes

Publishing an `oauth2PermissionScope` turns an application into a protected API resource that other applications can request delegated access to.

```text
Legacy application
        |
        v
Exposes delegated scope
        |
        v
Second application requests the scope
        |
        v
User sees a consent prompt
        |
        v
User grants consent
```

This path runs on consent, not on a credential. Credential hygiene does not touch it.

---

## 8. Redirect URIs and the Authorization Code Flow

The redirect URI tells Microsoft where to send the authorization response.

```text
Victim already signed in
        |
        v
Application requests delegated scope
        |
        v
Victim accepts consent
        |
        v
Authorization code issued
        |
        v
Browser redirected
        |
        v
Configured redirect URI receives the code
        |
        v
Backend exchanges code for token
```

If the redirect URI resolves to infrastructure outside the tenant, the authorization response leaves with it. Microsoft delivers the code to the address the application registration specifies.

This is why a redirect URI is security configuration. It determines where credentials are delivered.

---

## 9. OAuth2PermissionGrant

Consent writes a durable object into the directory. It is not browser state.

```powershell
az ad app permission list-grants `
  --id <ROGUE-APP-ID> `
  --show-resource-name true `
  -o json
```

Confirmed:

```text
consentType: Principal
resource:    <LEGACY-APP-NAME>
scope:       <CUSTOM-SCOPE-NAME>
```

```text
User consent
        |
        v
Service principal
        |
        v
OAuth2PermissionGrant
        |
        v
Legacy application
        |
        v
<CUSTOM-SCOPE-NAME>
```

`consentType: Principal` means one user consented for themselves. `AllPrincipals` would mean tenant-wide admin consent, which is the more severe variant of the same finding.

The grant survives the browser session, the password, and the MFA method that authorized it. It ends when it is explicitly revoked.

---

## 10. Why User Containment Is Incomplete

A standard identity response:

```text
Reset password
Revoke sessions
Require MFA
```

addresses the human identity and leaves:

```text
Application client secret
Application owner relationship
Custom exposed API scope
Redirect URI
OAuth2PermissionGrant
```

Each persists independently.

### Human identity

- reset credentials
- revoke active sessions
- review MFA methods
- investigate suspicious sign-ins

### Application identity

- revoke client secrets and certificates
- review app-registration owners
- review service-principal owners
- remove unauthorized redirect URIs
- review exposed API scopes
- review application permissions
- revoke malicious OAuth grants

An incident response that runs the first list and stops will report the incident contained while four access paths remain open.

---

## 11. Why This Path Rather Than Phishing Credentials Again

Credential phishing was already proven to work here. The attacker used it to get in. The question is why they built four stages of application infrastructure instead of simply doing it again when they needed access.

Because credential phishing has to survive the controls that sit between a stolen password and a usable session. A reused credential arrives from an unmanaged device, from an unfamiliar location, and has to satisfy MFA. Every one of those is a place Conditional Access can refuse the sign-in, and every one of them is a control this tenant would plausibly tighten immediately after an incident.

Consent phishing does not encounter any of them. The victim is already signed in, on a corporate device, with MFA already satisfied. They are not asked to authenticate. They are asked to approve, and approval is a button, not a credential. The authorization code is issued against a session the tenant has already accepted as legitimate, then delivered to the address the application registration specifies.

The output of that flow is also more durable than a password. An `OAuth2PermissionGrant` is a directory object. Resetting the password does not delete it. Revoking sessions does not delete it. Enforcing MFA does not delete it. It persists until someone explicitly revokes the grant, and standard containment does not include that step.

---

## 12. Confused Deputy

A confused deputy is a trusted component with legitimate authority, induced to act on behalf of a party that should not hold that authority.

```text
Legacy application
        |
        v
Trusted, privileged application identity
        |
        v
Second application obtains a delegated access path
        |
        v
Victim consents
        |
        v
The trusted relationship is used as the attacker intended
```

Nothing in the chain is a vulnerability. Every component behaves as designed. The attack is assembled from configuration.

---

## 13. Attack Chain

```text
1. ENTRY
Phished user completes MFA
        |
        v
Authenticated session stolen

2. ESCALATE
Application ownership abused
        |
        v
Long-lived client secret established
        |
        v
Application-level authentication

3. PIVOT
Second application registered
        |
        v
Its service principal added as an owner
        |
        v
Durable control path into the legacy application

4. PERSIST
Legacy application exposes a custom delegated scope
        |
        v
Second application can request delegated access

5. LOOT
Victim accepts OAuth consent
        |
        v
OAuth2PermissionGrant created
        |
        v
Authorization response sent to the configured redirect URI
        |
        v
Token access
```

---

## 14. Takeaways

### Applications are identities

App registrations and service principals need the same governance as user accounts: ownership review, credential lifecycle, permission review, and access recertification.

### Ownership is privilege

An application owner holds administrative capability over that application without holding a directory role. Reviews that enumerate roles will not find it.

### MFA is not containment

MFA protects authentication. It does not invalidate a stolen session, an application credential, an ownership relationship, or a consent grant.

### OAuth configuration is security configuration

Owners, credentials, Graph permissions, exposed scopes, redirect URIs, and consent grants are all security controls, and all are editable by anyone with sufficient influence over the application.

### The attack surface is a graph

```text
User
  |
  v
App registration
  |
  v
Service principal
  |
  v
Owner relationship
  |
  v
Application permissions
  |
  v
OAuth scope
  |
  v
Consent grant
  |
  v
Redirect URI
```

Every edge is a relationship someone configured. Reviewing objects individually misses the paths between them.

---

## 15. Data Handling

This repository documents method and reasoning.

Redacted from public evidence and command output:

- values seeded into object fields that function as assessment answers
- usernames and email addresses
- operative identifiers
- tenant IDs
- application and client IDs
- object IDs and service principal IDs
- credential key IDs
- authorization codes and tokens
- client secret values
- environment-specific redirect URI values

Also replaced with placeholders, though published by Microsoft and identical in every tenant:

- the Microsoft Graph application ID
- Microsoft Graph permission IDs

These carry no tenant-specific information and redacting them protects nothing. They are substituted because resolving an identifier to a permission name is the technique this file explains, and a reader who looks the values up has performed that technique rather than read its answer. The method is stated in full; the lookup is left to the reader.

Application display names and the custom scope name are retained. They carry no tenant-specific value, and removing them would make the ownership and consent relationships unreadable.

Applied per stage:

| Stage | Preserved | Redacted |
|---|---|---|
| Entry | Initial access method recorded in notes | Value inside the notes field |
| Escalate | Expiration date showing the credential lifetime | Credential description and key ID |
| Pivot | Owner relationship and permission names | Object IDs, service principal IDs, metadata values, permission and resource GUIDs |
| Persist | Existence and name of the custom scope | Consent display value |
| Loot | `consentType`, resource name, scope name, OAuth flow structure | Grant IDs, authorization codes, tokens, environment-specific redirect values |

The callback evidence does not expose reusable tokens, authorization codes, or other live credentials.

---

## 16. Detection Opportunities

Derived from this investigation, as candidates for a detection-engineering effort:

- new client secret added to an application
- credential expiration beyond policy
- new application owner added
- service principal added as an application owner
- new broad Microsoft Graph application permission
- new exposed API scope
- new or modified redirect URI
- new OAuth consent grant, particularly `consentType: AllPrincipals`

Not implemented here. Recorded as follow-on work.
