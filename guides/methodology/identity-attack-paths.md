# Identity Attack Paths

Common paths an attacker takes through identity infrastructure, what each requires, and what evidence each
leaves. Used to decide what to look for and where.

---

## Why map paths rather than techniques

A single misconfiguration is rarely the finding. The finding is the chain it enables. "Service account has an
SPN" is a fact; "service account has an SPN, a weak password, and membership in a group with write access to a
Tier 0 asset" is an attack path.

When reviewing identity, trace the path forward: **what does this let someone reach next?**

---

## On-premises Active Directory

### Credential access to privilege

```text
SPN on a user account with a weak password
    -> service ticket requested and cracked offline
        -> service account credentials
            -> whatever that account can reach
```

| Look for | Where |
|---|---|
| User accounts with `servicePrincipalName` set | AD PowerShell, LDAP |
| Password age on those accounts | `PasswordLastSet` |
| Group memberships of those accounts | `Get-ADPrincipalGroupMembership` |
| Abnormal service ticket request volume | Event 4769 |

Service accounts with SPNs, old passwords, and privileged group membership are the highest-value finding in
most AD environments.

### Delegation to impersonation

```text
Unconstrained delegation on a host
    -> any account authenticating to it leaves a usable TGT in memory
        -> compromise of that host yields those identities
```

| Look for | Where |
|---|---|
| `TrustedForDelegation` on computers or users | AD PowerShell, LDAP |
| `msDS-AllowedToDelegateTo` populated | Constrained delegation targets |
| Domain controllers authenticating to non-DC hosts | Event 4624 |

Unconstrained delegation on anything that is not a domain controller warrants explanation.

### ACL to control

```text
Write permission on a privileged group or user object
    -> add self to the group, or reset the password
        -> privilege without ever touching a credential
```

| Look for | Where |
|---|---|
| `GenericAll`, `WriteDacl`, `WriteOwner`, `WriteProperty` on privileged objects | Object ACLs |
| Non-admin principals with rights on Tier 0 objects | Same |
| `adminCount=1` on accounts no longer in a protected group | LDAP |

ACL findings are invisible to group membership review, which is why membership review alone is insufficient.

### Nested group to unintended privilege

```text
Low-privilege group nested into a privileged group
    -> everyone in the outer group inherits the privilege
        -> often nobody realizes
```

Always enumerate recursively. `Get-ADGroupMember -Recursive` exists because the non-recursive answer is
routinely wrong.

### Tier violation

```text
Domain Admin logs on interactively to a workstation
    -> credential material lands on a lower-tier machine
        -> workstation compromise becomes domain compromise
```

| Look for | Where |
|---|---|
| Tier 0 accounts with logon type 2 or 10 on non-Tier 0 hosts | Event 4624 |
| Absence of Protected Users membership for admin accounts | AD PowerShell |
| No authentication policy silos configured | AD |

---

## Entra ID and cloud identity

### Consent to data access

```text
User grants consent to an application
    -> the application holds delegated permissions to that user's data
        -> access persists independently of the user's password or MFA
```

| Look for | Where |
|---|---|
| Consent grant events | Audit logs, `Consent to application` |
| Applications with mail, file, or directory scopes | `Get-MgOauth2PermissionGrant` |
| Unverified publishers | Service principal properties |
| Admin consent granted tenant-wide | Audit logs |

**Revoking a password does not revoke a consent grant.** This is why consent phishing works and why it is
frequently missed during incident response.

### Service principal to standing privilege

```text
Service principal created for automation
    -> assigned a broad role for convenience
        -> pipeline decommissioned, identity remains
            -> standing privilege with no owner and no rotation
```

| Look for | Where |
|---|---|
| Service principals with Owner or Contributor at subscription scope | `az role assignment list --all` |
| Credential age on those principals | `az ad sp credential list` |
| Application permissions rather than delegated | Graph |
| No owner recorded | Application properties |

Application permissions act without a signed-in user. They are the higher-risk grant and are exempt from
Conditional Access.

### Hybrid sync as a bridge

```text
On-premises compromise
    -> Entra Connect sync account
        -> cloud directory write access
```

| Look for | Where |
|---|---|
| Sync account privileges in both directories | AD and Entra |
| Password hash sync or pass-through auth configuration | Entra Connect |
| Sync server treated as Tier 0 | Host configuration |

The sync server is a Tier 0 asset. It is regularly managed like a utility server.

### Conditional Access gaps

```text
Legacy authentication protocol still permitted
    -> MFA cannot be enforced on that path
        -> credential stuffing succeeds against a fully "MFA-enforced" tenant
```

| Look for | Where |
|---|---|
| Sign-ins using legacy auth clients | `SigninLogs`, client app field |
| Policies scoped with broad exclusions | Conditional Access |
| Break-glass accounts excluded from all policies | Conditional Access |
| Guest and external identities outside policy scope | Conditional Access |

Exclusions are where Conditional Access breaks. Enumerate them explicitly; a policy report showing "enabled"
says nothing about who it does not apply to.

---

## Review order

Working outside in finds the highest-impact issues first.

```text
1. Who holds the highest privilege, and is each one justified
2. What can reach that privilege, directly or by nesting
3. What credentials protect those paths, and how old are they
4. What authentication paths bypass the controls in place
5. What would be logged if any of this were exploited
```

Step 5 is the one usually skipped. A finding that could not be detected if exploited deserves its own entry.

---

## Recording a path

```markdown
## Attack Path

Source:     Standard user account with write access to `SVC-Backup`
Enabling:   `SVC-Backup` is a member of `Backup Operators`
Reaches:    Backup Operators can read `NTDS.dit` on domain controllers
Impact:     Full credential database exposure from a standard user account

Evidence:   [ACL output], [group membership], [privilege enumeration]
Detection:  Would generate Event 4728 on group change, no alert configured
```

Source, enabling condition, what it reaches, impact, evidence, detectability. That format turns a list of
misconfigurations into something a reader can act on.
