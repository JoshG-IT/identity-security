# LDAP and dsquery

Lower-level directory queries. Useful when the ActiveDirectory module is unavailable, and for precise filter control.

## LDAP filter syntax

Prefix notation, operators come before operands.

```text
(objectClass=user)
(&(objectClass=user)(objectCategory=person))
(|(department=IT)(department=Security))
(!(userAccountControl:1.2.840.113556.1.4.803:=2))     account NOT disabled
```

## userAccountControl bit matching

The most useful filter in AD investigation.

```text
1.2.840.113556.1.4.803    bitwise AND, flag is set

2        disabled
32       password not required
64       password cannot change
65536    password never expires
262144   smartcard required
524288   trusted for delegation
8388608  password expired
```

```text
(userAccountControl:1.2.840.113556.1.4.803:=65536)     password never expires
(userAccountControl:1.2.840.113556.1.4.803:=524288)    unconstrained delegation
(!(userAccountControl:1.2.840.113556.1.4.803:=2))      enabled accounts only
```

## PowerShell without the AD module

```powershell
$searcher = New-Object DirectoryServices.DirectorySearcher
$searcher.Filter = "(&(objectClass=user)(userAccountControl:1.2.840.113556.1.4.803:=65536))"
$searcher.PropertiesToLoad.AddRange(@("samaccountname","distinguishedname"))
$searcher.PageSize = 1000
$searcher.FindAll() | ForEach-Object { $_.Properties.samaccountname }
```

`PageSize` matters. Without it results cap at 1000 silently.

## dsquery

```cmd
dsquery user -limit 0
dsquery user -inactive 12
dsquery user -disabled
dsquery group -name "Domain Admins"
dsquery * -filter "(&(objectClass=user)(servicePrincipalName=*))" -attr samaccountname servicePrincipalName
```

Pipe into `dsget` to expand:

```cmd
dsquery group -name "Domain Admins" | dsget group -members -expand
```

## ldapsearch

```bash
ldapsearch -x -H ldap://dc.example.com -D "user@example.com" -W \
  -b "DC=example,DC=com" "(objectClass=user)" sAMAccountName

ldapsearch -x -H ldap://dc.example.com -D "user@example.com" -W \
  -b "DC=example,DC=com" \
  "(userAccountControl:1.2.840.113556.1.4.803:=524288)" sAMAccountName
```

## Useful queries

```text
SPN-bearing accounts   (&(objectClass=user)(servicePrincipalName=*)(!(objectClass=computer)))
Pre-auth not required  (userAccountControl:1.2.840.113556.1.4.803:=4194304)
Unconstrained deleg.   (userAccountControl:1.2.840.113556.1.4.803:=524288)
Protected Users        (memberOf=CN=Protected Users,CN=Users,DC=example,DC=com)
Admin-count set        (adminCount=1)
```

## Notes

- `adminCount=1` marks objects that were once in a protected group. It persists after removal, so it surfaces historical privilege.
- LDAP returns raw attribute values, timestamps arrive as Windows FILETIME and need converting.
- Anonymous binds are usually disabled; expect to authenticate.
- Port 389 is plaintext, 636 is LDAPS. Note which you used when documenting.
