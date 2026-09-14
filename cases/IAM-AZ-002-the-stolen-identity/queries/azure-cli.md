# Azure CLI Commands
## The Stolen Identity

This file documents the Azure CLI commands used during the investigation.

> **Data handling:** Application display names, custom scope names, application and client IDs, object IDs, service principal IDs, tenant identifiers, grant IDs, and challenge values are replaced with placeholders or omitted. Publicly documented Microsoft identifiers, such as the Microsoft Graph application ID and Microsoft Graph permission IDs, are retained because they are published by Microsoft and are required to explain the resolution technique.

---

## 1. Enumerate App Registrations

Used to identify the legacy application and the attacker-created application.

```powershell
az ad app list -o table
```

### Purpose

- Enumerate app registrations visible to the operative account
- Identify the legacy application and the attacker-created application
- Obtain the application/client ID needed for deeper inspection

---

## 2. Inspect an App Registration

Used to inspect the complete Microsoft Entra application object.

```powershell
az ad app show `
  --id <APP-ID> `
  -o json
```

### Purpose

The raw application object exposed security-relevant properties including:

- `notes`
- `passwordCredentials`
- `requiredResourceAccess`
- `api`
- `web`

This command was used against both the legacy and rogue applications.

---

## 3. Review the Legacy Application Notes

```powershell
az ad app show `
  --id <LEGACY-APP-ID> `
  --query notes `
  -o tsv
```

### Purpose

Read the legacy application's internal notes field documenting the initial compromise context.

---

## 4. Review Client Credentials

```powershell
az ad app show `
  --id <LEGACY-APP-ID> `
  --query passwordCredentials
```

### Purpose

Inspect application credential metadata, including:

- credential display name
- start time
- expiration time
- hint
- key ID

The investigation identified an unusually long-lived client secret.

### Alternate Credential View

Azure CLI also provides a dedicated credential command:

```powershell
az ad app credential list `
  --id <LEGACY-APP-ID> `
  -o table
```

This was useful for validating the same credential metadata in a condensed format.

---

## 5. Review Application Owners

```powershell
az ad app owner list `
  --id <LEGACY-APP-ID> `
  -o json
```

### Purpose

Inspect the owner relationship on the legacy app registration.

The returned owner object identified the rogue application's **service principal** as an owner of the legacy application.

### Condensed View

```powershell
az ad app owner list `
  --id <LEGACY-APP-ID> `
  --query "[].{Owner:displayName,Type:servicePrincipalType}" `
  -o table
```

### Investigation Significance

This relationship directly demonstrated the Pivot stage:

```text
Rogue application
        |
        v
Service principal
        |
        v
Owner of legacy application
```

Application ownership created a persistence path that could survive rotation of the original client secret.

---

## 6. Review and Resolve Microsoft Graph Application Permissions

First, inspect the API permissions requested by the legacy application.

```powershell
az ad app show `
  --id <LEGACY-APP-ID> `
  --query requiredResourceAccess
```

### Purpose

Identify:

- the resource API being requested,
- each permission GUID,
- and whether each permission is an application permission or delegated permission.

The Microsoft Graph resource entry contained two `resourceAccess` objects with:

```json
"type": "Role"
```

In `requiredResourceAccess`, `Role` represents an **application permission / app role**. It does not represent a user or an Entra directory-role assignment.

### Resolve `Directory.Read.All`

The raw permission GUID can be resolved against the Microsoft Graph service principal's `appRoles` collection:

```powershell
az ad sp show `
  --id <MS-GRAPH-APP-ID> `
  --query "appRoles[?id=='<DIRECTORY-READ-ALL-PERMISSION-ID>']" `
  -o table
```

Result:

```text
Directory.Read.All
```

Microsoft Graph identifiers (publicly documented, redacted here):

```text
Microsoft Graph App ID:
<MS-GRAPH-APP-ID>

Directory.Read.All application permission ID:
<DIRECTORY-READ-ALL-PERMISSION-ID>
```

### Resolve `User.Read.All`

```powershell
az ad sp show `
  --id <MS-GRAPH-APP-ID> `
  --query "appRoles[?id=='<USER-READ-ALL-PERMISSION-ID>']" `
  -o table
```

Result:

```text
User.Read.All
```

Microsoft Graph identifiers (publicly documented, redacted here):

```text
Microsoft Graph App ID:
<MS-GRAPH-APP-ID>

User.Read.All application permission ID:
<USER-READ-ALL-PERMISSION-ID>
```

### Investigation Significance

The legacy application requested these Microsoft Graph **application permissions**:

- `Directory.Read.All`
- `User.Read.All`

Resolving the permission GUIDs directly against Microsoft Graph confirmed what the otherwise opaque `resourceAccess[].id` values represented.

Because both entries were application permissions, the effective permission belonged to the application identity/service principal rather than depending on the permissions of the originally compromised user's interactive session.

---

## 7. Review Exposed API Configuration

```powershell
az ad app show `
  --id <LEGACY-APP-ID> `
  --query api
```

### Purpose

Inspect the application's API configuration and the `oauth2PermissionScopes` collection.

The investigation found a custom delegated scope used as part of the OAuth consent path.

---

## 8. Review Rogue Application Web Configuration

```powershell
az ad app show `
  --id <ROGUE-APP-ID> `
  --query web
```

### Purpose

Inspect redirect URI configuration associated with the rogue application.

The output exposed:

- a normal local-development callback
- a second redirect URI associated with the attack flow

Challenge-specific values were redacted from the public evidence.

---

## 9. Review OAuth Permission Grants

```powershell
az ad app permission list-grants `
  --id <ROGUE-APP-ID> `
  --show-resource-name true `
  -o json
```

### Purpose

Confirm that the consent flow created an actual delegated OAuth authorization relationship.

The result showed:

```text
consentType: Principal
resourceDisplayName: <LEGACY-APP-NAME>
scope: <CUSTOM-SCOPE-NAME>
```

### Investigation Significance

This was stronger than simply observing a consent screen.

It proved that consent resulted in a persistent delegated grant:

```text
Rogue application / service principal
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

---

# JMESPath Exploration Notes

Azure CLI uses JMESPath for `--query`.

The following patterns were useful while inspecting unfamiliar application objects.

## Show Root Properties

```powershell
--query "keys(@)"
```

## Check a Property Type

```powershell
--query "type(passwordCredentials)"
```

## Count Items in an Array

```powershell
--query "length(passwordCredentials)"
```

## Show Keys Inside the First Array Object

```powershell
--query "keys(passwordCredentials[0])"
```

## Select a Single Property

```powershell
--query notes
```

## Select a Nested Object

```powershell
--query api.oauth2PermissionScopes
```

## Select a Property from Every Array Item

```powershell
--query "appRoles[].displayName"
```

## Build a Custom Output Object

```powershell
--query "{Name:displayName,AppId:appId,ObjectId:id}"
```

---

# Investigation Workflow

```text
az ad app list
        |
        v
Identify relevant applications
        |
        v
az ad app show
        |
        v
Inspect application objects
        |
        v
Query specific properties
        |
        v
Validate ownership relationship
        |
        v
Review requested permissions
        |
        v
Inspect exposed API scope
        |
        v
Inspect redirect URI configuration
        |
        v
Confirm OAuth2PermissionGrant
```

---

# Key Azure CLI Lessons

- `az ad app list` is useful for tenant-level application discovery.
- `az ad app show` exposes the full application registration object.
- `az ad app owner list` can reveal users or service principals that own an application.
- `requiredResourceAccess` shows requested API permissions, but GUIDs may require additional resolution to human-readable permission names.
- `az ad app permission list-grants` exposes delegated OAuth grants created through consent.
- JMESPath is useful for reducing large JSON objects into investigation-relevant fields.
