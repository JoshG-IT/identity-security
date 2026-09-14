# Azure CLI Commands
## The Stolen Identity

This file documents the Azure CLI commands used during the investigation.

> **Data handling:** Application IDs, object IDs, service principal IDs, tenant identifiers, grant IDs, and challenge values are replaced with placeholders or omitted.

---

## 1. Enumerate App Registrations

Used to identify the legacy application and the attacker-created application.

```powershell
az ad app list -o table
```

### Purpose

- Enumerate app registrations visible to the operative account
- Identify:
  - `Mad-Hat-Legacy-Sync-Service`
  - `Mad-Hat-Labs-App`
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

The returned owner object identified the `Mad-Hat-Labs-App` **service principal** as an owner of the legacy application.

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
        ↓
Service principal
        ↓
Owner of legacy application
```

Application ownership created a persistence path that could survive rotation of the original client secret.

---

## 6. Review Requested API Permissions

```powershell
az ad app show `
  --id <LEGACY-APP-ID> `
  --query requiredResourceAccess
```

### Purpose

Inspect the APIs and permission types requested by the legacy application.

The Microsoft Graph resource entry contained two `resourceAccess` objects with:

```json
"type": "Role"
```

`Role` in this context represents a Microsoft Graph **application permission**, not a user or Entra directory-role assignment.

The investigation identified:

- `Directory.Read.All`
- `User.ReadWrite.All`

These permissions increased the potential directory-level blast radius of the compromised application.

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
resourceDisplayName: Mad-Hat-Legacy-Sync-Service
scope: Legacy.Sync
```

### Investigation Significance

This was stronger than simply observing a consent screen.

It proved that consent resulted in a persistent delegated grant:

```text
Rogue application / service principal
        ↓
OAuth2PermissionGrant
        ↓
Legacy application
        ↓
Legacy.Sync
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
        ↓
Identify relevant applications
        ↓
az ad app show
        ↓
Inspect application objects
        ↓
Query specific properties
        ↓
Validate ownership relationship
        ↓
Review requested permissions
        ↓
Inspect exposed API scope
        ↓
Inspect redirect URI configuration
        ↓
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
