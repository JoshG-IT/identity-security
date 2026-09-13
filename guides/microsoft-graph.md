# Microsoft Graph

The interface for Entra ID: users, groups, applications, service principals, sign-ins, and audit logs.

## Setup

```powershell
Connect-MgGraph -Scopes "User.Read.All","Directory.Read.All","AuditLog.Read.All","Application.Read.All"
Get-MgContext
```

Scopes are requested per session. A missing scope produces an authorization error, not an empty result, read the error before assuming there is no data.

## Users

```powershell
Get-MgUser -All | Select-Object DisplayName, UserPrincipalName, AccountEnabled
Get-MgUser -Filter "accountEnabled eq false" -All
Get-MgUser -UserId <UPN> -Property "signInActivity" |
  Select-Object -ExpandProperty SignInActivity
```

## Groups and membership

```powershell
Get-MgGroup -All | Select-Object DisplayName, Id, GroupTypes
Get-MgGroupMember -GroupId <GROUP_ID> -All
Get-MgUserMemberOf -UserId <UPN>
```

## Applications and service principals

Where consent-phishing and standing-privilege findings live.

```powershell
Get-MgServicePrincipal -All | Select-Object DisplayName, AppId, PublisherName

# delegated permissions granted to an app
Get-MgOauth2PermissionGrant -All | Where-Object ClientId -eq <SP_OBJECT_ID>

# application permissions (app-only, no signed-in user)
Get-MgServicePrincipalAppRoleAssignment -ServicePrincipalId <SP_OBJECT_ID>

# credential age
Get-MgServicePrincipal -ServicePrincipalId <SP_OBJECT_ID> |
  Select-Object -ExpandProperty PasswordCredentials
```

## Directory roles

```powershell
Get-MgDirectoryRole -All | Select-Object DisplayName, Id
Get-MgDirectoryRoleMember -DirectoryRoleId <ROLE_ID>
```

## Sign-in and audit logs

```powershell
Get-MgAuditLogSignIn -Top 50 |
  Select-Object CreatedDateTime, UserPrincipalName, AppDisplayName, IpAddress,
                @{n='Status';e={$_.Status.ErrorCode}}

Get-MgAuditLogSignIn -Filter "userPrincipalName eq '<UPN>'" -Top 100

# consent grants recorded in audit
Get-MgAuditLogDirectoryAudit -Filter "activityDisplayName eq 'Consent to application'" -Top 50
```

## Raw requests

Some properties are not exposed by a cmdlet. Call the endpoint directly.

```powershell
Invoke-MgGraphRequest -Method GET `
  -Uri "https://graph.microsoft.com/v1.0/servicePrincipals/<ID>/oauth2PermissionGrants"
```

## Notes

- `-All` or the result is paged and silently truncated.
- `v1.0` is stable; `beta` has more properties and no guarantees. State which you used.
- Delegated permissions act as a signed-in user. Application permissions act without one and are the higher-risk grant.
- An unverified publisher on an app holding mail permissions is a consent-phishing indicator worth documenting.
