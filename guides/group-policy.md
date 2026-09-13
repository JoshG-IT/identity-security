# Group Policy

Interface for inspecting policy-based configuration and security settings across a domain.

## Setup

```powershell
Import-Module GroupPolicy
Get-GPO -All | Format-Table DisplayName, GpoStatus, ModificationTime
```

## Inspecting policy

```powershell
Get-GPO -Name "<GPO_NAME>"
Get-GPOReport -Name "<GPO_NAME>" -ReportType Html -Path .\gpo-report.html
Get-GPOReport -All -ReportType Xml -Path .\all-gpos.xml
```

## Links and scope

```powershell
Get-GPInheritance -Target "OU=Workstations,DC=example,DC=com"

# every OU a GPO is linked to
Get-GPO -All | ForEach-Object {
    $gpo = $_
    ([xml](Get-GPOReport -Guid $gpo.Id -ReportType Xml)).GPO.LinksTo |
      Select-Object @{n='GPO';e={$gpo.DisplayName}}, SOMPath, Enabled
}
```

## Resultant set of policy

What a machine or user actually receives after inheritance, blocking, and enforcement.

```powershell
gpresult /h gpresult.html /f
gpresult /r /scope:computer
Get-GPResultantSetOfPolicy -ReportType Html -Path .\rsop.html
```

## Security-relevant settings

| Setting | Why it matters |
|---|---|
| Password policy, length, complexity, age | Baseline credential strength |
| Account lockout threshold | Brute-force resistance vs denial-of-service risk |
| Audit policy, logon, object access, process creation | Determines whether detection is possible at all |
| PowerShell logging, script block, module, transcription | Highest-value telemetry for hunting |
| LLMNR and NetBIOS | Enabled by default, widely abused for credential relay |
| SMB signing | Prevents relay attacks |
| Restricted Groups | Controls local administrator membership |
| User Rights Assignment | Who may log on locally, as a service, or as a batch job |

```powershell
# find every GPO configuring a given setting
Get-GPO -All | ForEach-Object {
    $r = Get-GPOReport -Guid $_.Id -ReportType Xml
    if ($r -match "ScriptBlockLogging") { $_.DisplayName }
}
```

## Notes

- Link order matters: the GPO closest to the object wins, unless Enforced is set.
- Block Inheritance is a common source of "the policy exists but isn't applying."
- Delegation on a GPO is itself a finding, whoever can edit it can execute code on every machine it applies to.
- `GpoStatus` of `AllSettingsDisabled` means the GPO exists but does nothing. Easy to miss.
