# Azure CLI Commands
## The Stolen Identity

This file documents the Azure CLI commands used during the investigation.

> **Data handling:** Application display names, custom scope names, application and client IDs, object IDs, service principal IDs, tenant identifiers, grant IDs, and any value seeded into an object field that functions as an assessment answer are replaced with placeholders or omitted. Microsoft's published identifiers, including the Microsoft Graph application ID and Microsoft Graph permission IDs, are also represented as placeholders. They carry no tenant-specific information, but resolving an identifier to a permission name is the technique this file documents, and a reader who looks the values up has performed that technique rather than read its answer.

---

## 1. Enumerate App Registrations

Used to identify the legacy application and the attacker-created application.

```powershell
az ad app list -o table
```

**Question answered:** What application registrations exist in the tenant?

---

## 2. Inspect the Legacy Application Object

Used to inspect the complete Microsoft Entra application object for configuration and metadata.

```powershell
az ad app show `
  --id <LEGACY-APP-ID> `
  -o json
```

**Question answered:** What is the full configuration of the legacy application?

The raw application object exposes security-relevant properties including:

- `notes`
- `passwordCredentials`
- `requiredResourceAccess`
- `api`
- `web`

---

## 3. Review the Legacy Application Notes

```powershell
az ad app show `
  --id <LEGACY-APP-ID> `
  --query notes `
  -o tsv
```

**Question answered:** What metadata was recorded in the legacy application's notes field?

---

## 4. Review Client Credentials

```powershell
az ad app show `
  --id <LEGACY-APP-ID> `
  --query passwordCredentials
```

**Question answered:** What password credentials are configured on the legacy application?

Inspect application credential metadata, including:

- credential display name
- start time
- expiration time
- hint
- key ID

The investigation identified an unusually long-lived client secret set to expire near the end of the century.

### Alternate Credential View

Azure CLI also provides a dedicated credential command:

```powershell
az ad app credential list `
  --id <LEGACY-APP-ID> `
  -o table
```

**Question answered:** What are the credentials configured on this application, in condensed format?

This view was useful for validating the same credential metadata in a table format.

---

## 5. Review Application Owners

```powershell
az ad app owner list `
  --id <LEGACY-APP-ID> `
  -o json
```

**Question answered:** Who owns the legacy application?

Inspect the owner relationship on the legacy app registration. The returned owner object identified the rogue application's **service principal** as an owner of the legacy application.

### Condensed View

```powershell
az ad app owner list `
  --id <LEGACY-APP-ID> `
  --query "[].{Owner:displayName,Type:servicePrincipalType}" `
  -o table
```

**Question answered:** What are the owners of this application, and what type is each?

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

**Question answered:** What API permissions does the legacy application request?

Identify:

- the resource API being requested
- each permission GUID
- whether each permission is an application permission or delegated permission

The Microsoft Graph resource entry contained two `resourceAccess` objects with:

```json
"type": "Role"
```

In `requiredResourceAccess`, `Role` represents an **application permission / app role**. It does not represent a user or an Entra directory-role assignment.

### Resolve Directory.Read.All

The raw permission GUID can be resolved against the Microsoft Graph service principal's `appRoles` collection:

```powershell
az ad sp show `
  --id <MS-GRAPH-APP-ID> `
  --query "appRoles[?id=='<DIRECTORY-READ-ALL-PERMISSION-ID>']" `
  -o table
```

**Question answered:** What permission does this GUID represent?

Result:

```text
Directory.Read.All
```

### Resolve User.Read.All

```powershell
az ad sp show `
  --id <MS-GRAPH-APP-ID> `
  --query "appRoles[?id=='<USER-READ-ALL-PERMISSION-ID>']" `
  -o table
```

**Question answered:** What permission does this second GUID represent?

Result:

```text
User.Read.All
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

**Question answered:** What APIs does the legacy application expose?

Inspect the application's API configuration and the `oauth2PermissionScopes` collection. The investigation found a custom delegated scope used as part of the OAuth consent path.

---

## 8. Review Rogue Application Web Configuration

```powershell
az ad app show `
  --id <ROGUE-APP-ID> `
  --query web
```

**Question answered:** What redirect URIs are configured on the rogue application?

Inspect redirect URI configuration associated with the rogue application. The output exposed:

- a normal local-development callback
- a second redirect URI associated with the attack flow

The second redirect URI carried a query string. It was redacted in full from the public evidence rather than partially, because a query string can carry values that are not visible as sensitive at a glance.

---

## 9. Review OAuth Permission Grants

```powershell
az ad app permission list-grants `
  --id <ROGUE-APP-ID> `
  --show-resource-name true `
  -o json
```

**Question answered:** What OAuth permission grants exist for the rogue application?

Confirm that the consent flow created an actual delegated OAuth authorization relationship.

The result showed:

```text
consentType: Principal
resourceDisplayName: <LEGACY-APP-NAME>
scope: <CUSTOM-SCOPE-NAME>
```

### Investigation Significance

This was stronger than simply observing a consent screen. It proved that consent resulted in a persistent delegated grant:

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

**Result:** List of all top-level keys in the returned object.

## Check a Property Type

```powershell
--query "type(passwordCredentials)"
```

**Result:** The data type of the specified property.

## Count Items in an Array

```powershell
--query "length(passwordCredentials)"
```

**Result:** The number of items in an array.

## Show Keys Inside the First Array Object

```powershell
--query "keys(passwordCredentials[0])"
```

**Result:** All keys in the first item of an array.

## Select a Single Property

```powershell
--query notes
```

**Result:** The value of a single property.

## Select a Nested Object

```powershell
--query api.oauth2PermissionScopes
```

**Result:** The nested object specified.

## Select a Property from Every Array Item

```powershell
--query "appRoles[].displayName"
```

**Result:** All display names from every item in the array.

## Filter an Array by Field Value

```powershell
--query "appRoles[?id=='<PERMISSION-ID>']"
```

**Result:** Only the array items matching the condition. This is what makes permission resolution practical, since the Microsoft Graph service principal returns several hundred app roles.

## Build a Custom Output Object

```powershell
--query "{Name:displayName,AppId:appId,ObjectId:id}"
```

**Result:** A custom object with renamed fields.

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
- Reading an object unprojected before narrowing to a projection is what makes the projection defensible. It is also a redaction decision: narrowing the output keeps values off the screen that would otherwise have to be blurred out of evidence afterwards.
