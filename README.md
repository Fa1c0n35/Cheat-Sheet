# Active Directory Cheat Sheet

## General Process
- Recon
- Domain Enum
- Local Privilege Escalation
- Local Account Stealing
- Monitor Potential Incomming account
- Local Account Stealing
- Admin Recon
- Lateral Mouvement
- Domain Admin Privileges
- Cross Trust Attacks
- Persistance and Exfiltrate

### Users,Computer,Domain Admins, Enterprise Administrators, Shares
Tool : PowerView
> . .\PowerView.ps1
Import-Module PowerView.ps1

- Enumerate Users
```Powershell
Get-NetUser
Get-NetUser | select -ExpandProperty samaccountname
```

- Enumerate Computers
```Powershell
Get-NetComputer
```

- Enumerate Domain Administrators
```Powershell
Get-NetGroup -GroupName "Domain Admins" -FullData
Get-NetGroupMember -GroupName "Domain Admins"
```

- Enumerate Enterprise Administrators
```Powershell
Get-NetGroupMember -GroupName "Enterprise Admins"
Get-NetGroupMember -GroupName "Enterprise Admins" -Domain target.local
```

- Enumerate Interesting Shares
```Powershell
Invoke-ShareFinder -ExcludeStandard -ExcludePrint -ExcludeIPC -Verbose
```

### OU, Computer in target OU, GPO, GPO on target OU
Tool : PowerView
> . .\PowerView.ps1
Import-Module PowerView.ps1

- Enumerate Restricted Groups from GPO
```Powershell
Get-NetGPOGroup -Verbose
```

- Membership of the Group "RDPUsers”
```Powershell
Get-NetGroupMember -GroupName RDPUsers
```

- List all the computers in the target OU
```powershell
Get-NetOU targetcomputer | %{Get-NetComputer -ADSPath $_}
```

- List GPOs
```powershell
Get-NetGPO
```
- GPO applied on the target OU
```powershell
(Get-NetOU target -FullData).gplink[LDAP://cn={x-x-x-x-x},cn=policies,cn=system,DC=target,DC=domain,DC=local;0]
Get-NetGPO -ADSpath 'LDAP://cn={x-x-x-x-x},cn=policies,cn=system,DC=target,DC=domain,DC=local'
```

### ACL for a Users group, ACL for Domain Admins group, right/permissions for target user
Tool : PowerView
> . .\PowerView.ps1
Import-Module PowerView.ps1

- Enumerate ACLs
```powershell
Get-ObjectAcl -SamAccountName "users" -ResolveGUIDs -Verbose
Get-ObjectAcl -SamAccountName "Domain Admins" -ResolveGUIDs -Verbose
```

- Enumerate ACLs for all the GPOs
```powershell
Get-NetGPO | %{Get-ObjectAcl -ResolveGUIDs -Name $_.Name}
```

- Enumerate GPOs where target user or group have interesting permissions
```powershell
Get-NetGPO | %{Get-ObjectAcl -ResolveGUIDs -Name $_.Name} | ?{$_.IdentityReference -match "target"}
```

- Check for modify rights/permissions for the target
```powershell
Invoke-ACLScanner -ResolveGUIDs | ?{$_.IdentityReference -match "target"}
Invoke-ACLScanner -ResolveGUIDs | ?{$_.IdentityReference -match "targetgroup"}
```

### Domain, trust, external trust
Tool : PowerView
> . .\PowerView.ps1
Import-Module PowerView.ps1

- All domains
```powershell
Get-NetForestDomain -Verbose
```
- Map the trusts
```powershell
Get-NetDomainTrust
```

- Map all the trusts of the domain.local forest
```powershell
Get-NetForestDomain -Verbose | Get-NetDomainTrust
```

- List external trusts
```powershell
Get-NetForestDomain -Verbose | Get-NetDomainTrust | ?{$_.TrustType -eq 'External'}
```
if Bi-Directional trust we can extract information


