# Active Directory PowerShell

Primary interface for on-premises identity investigation.

## Setup

```powershell
Import-Module ActiveDirectory
Get-ADDomain
Get-ADForest
Get-ADDomainController -Filter * | Format-Table Name, Site, IsGlobalCatalog
```

## Users

```powershell
Get-ADUser -Filter * -Properties LastLogonDate, PasswordLastSet, Enabled |
  Select-Object Name, SamAccountName, Enabled, LastLogonDate, PasswordLastSet

Get-ADUser -Filter { Enabled -eq $false } -Properties WhenChanged
Get-ADUser -Filter { PasswordNeverExpires -eq $true } -Properties PasswordNeverExpires

# stale but still enabled
$cutoff = (Get-Date).AddDays(-90)
Get-ADUser -Filter { LastLogonDate -lt $cutoff -and Enabled -eq $true } -Properties LastLogonDate
```

## Groups and privilege

```powershell
Get-ADGroupMember -Identity "Domain Admins" -Recursive
Get-ADGroupMember -Identity "Enterprise Admins" -Recursive
Get-ADPrincipalGroupMembership -Identity <SAM> | Select-Object Name

# nested expansion: where unintended privilege hides
Get-ADGroup -Identity "Domain Admins" -Properties Members |
  Select-Object -ExpandProperty Members
```

## Computers

```powershell
Get-ADComputer -Filter * -Properties OperatingSystem, LastLogonDate |
  Select-Object Name, OperatingSystem, LastLogonDate

Get-ADComputer -Filter { OperatingSystem -like "*2012*" } -Properties OperatingSystem
```

## Delegation: high-value findings

```powershell
# unconstrained delegation
Get-ADComputer -Filter { TrustedForDelegation -eq $true } -Properties TrustedForDelegation
Get-ADUser -Filter { TrustedForDelegation -eq $true } -Properties TrustedForDelegation

# constrained delegation targets
Get-ADObject -Filter { msDS-AllowedToDelegateTo -like "*" } -Properties msDS-AllowedToDelegateTo
```

## Service accounts

```powershell
# accounts carrying an SPN
Get-ADUser -Filter { ServicePrincipalName -like "*" } `
  -Properties ServicePrincipalName, PasswordLastSet
```

## Object ACLs

```powershell
(Get-Acl "AD:$((Get-ADUser <SAM>).DistinguishedName)").Access |
  Where-Object { $_.ActiveDirectoryRights -match "WriteDacl|GenericAll|WriteOwner" }
```

## Replication health

```powershell
repadmin /replsummary
repadmin /showrepl
dcdiag /v
```

## Notes

- `LastLogonDate` derives from `lastLogonTimestamp`, replicated with up to a 14-day lag. `lastLogon` is accurate but per-DC and unreplicated, query every DC if precision matters.
- `-Properties` is required for anything outside the default attribute set. A missing property returns nothing, not an error.
- `-Recursive` on `Get-ADGroupMember` expands nested groups. Without it you miss inherited privilege.
- Run from a domain-joined host with RSAT installed.
