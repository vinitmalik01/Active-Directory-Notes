# situational awareness- bof

# CS Situational Awareness BOF — Cobalt Strike Cheatsheet

> TrustedSec’s [CS-Situational-Awareness-BOF](https://github.com/trustedsec/CS-Situational-Awareness-BOF). Run all commands from an active beacon after loading `SA.cna`.
> 

---

## Host Info

### `whoami`

Full `whoami /all` — user, groups, privileges, logon session.

```
whoami
```

### `get_session_info`

Current user logon session details.

```
get_session_info
```

### `ipconfig`

IPv4 address, hostname, and DNS server.

```
ipconfig
```

### `env`

List all process environment variables.

```
env
```

### `uptime`

System boot time and total uptime.

```
uptime
```

### `locale`

System locale, language, locale ID, date/time format, country.

```
locale
```

### `resources`

Memory usage and available disk space on the primary drive.

```
resources
```

### `useridletime`

How long the current user has been idle (seconds, minutes, hours, days).

```
useridletime
```

---

## Process & Module Enumeration

### `tasklist`

Running processes with PID, PPID, and full command line (via WMI).

```
tasklist
tasklist [server]
```

### `listmods`

List loaded DLLs in the current process or a target PID. Useful for detecting AV/EDR injection.

```
listmods
listmods [pid]
```

### `findLoadedModule`

Find all processes a module is loaded into. Optionally filter by process name.

```
findLoadedModule [modulepart]
findLoadedModule [modulepart] [procnamepart]
```

> Example: `findLoadedModule mssec` — find what’s loaded with AV/EDR modules.
> 

### `driversigs`

Enumerate installed service ImagePaths and check signing certs against known AV/EDR vendors.

```
driversigs
```

> Pair with `listmods` to check if your process has been injected.
> 

### `enum_filter_driver`

Enumerate filter drivers (filesystem, network minifilter).

```
enum_filter_driver
enum_filter_driver [computer]
```

---

## Network

### `netstat`

TCP and UDP IPv4 listening/connected ports.

```
netstat
```

### `arp`

ARP table.

```
arp
```

### `routeprint`

IPv4 routing table.

```
routeprint
```

### `listdns`

List DNS cache entries and attempt to resolve each.

```
listdns
```

### `nslookup`

DNS query with optional server and record type.

```
nslookup [hostname]
nslookup [hostname] [dns_server] [record_type]
nslookup dc01.corp.local 0 A
nslookup corp.local 8.8.8.8 ANY
```

> Use `0` for default DNS server. Record types: `A`, `AAAA`, `ANY`, `MX`, `TXT`, etc.
> 

### `probe`

Check if a specific port is open on a host.

```
probe [host] [port]
probe 10.10.10.5 445
probe dc01.corp.local 88
```

### `list_firewall_rules`

List all Windows Firewall rules.

```
list_firewall_rules
```

---

## Network Shares & Sessions

### `netshares`

List shares on local or remote computer.

```
netshares [hostname]
netshares dc01.corp.local
```

### `netview`

List all reachable computers in the current domain.

```
netview
```

### `netsession`

Enumerate sessions on the local or specified computer.

```
netsession
netsession [computer]
```

### `netsession2`

BOFHound-compatible version. Supports custom resolution method and DNS server.

```
netsession2
netsession2 [computer] [resolution_method] [dns_server]
```

### `netloggedon`

Return users currently logged on locally or remotely.

```
netloggedon [hostname]
```

### `netloggedon2`

BOFHound-compatible version.

```
netloggedon2
netloggedon2 [hostname]
```

### `enumLocalSessions`

Enumerate currently attached user sessions — local and RDP.

```
enumLocalSessions
```

### `regsession`

Enumerate logged-on user SIDs via `HKEY_USERS`. BOFHound compatible.

```
regsession
regsession [hostname]
```

### `nettime`

Display time on a remote computer.

```
nettime [hostname]
```

### `netuptime`

Return boot time info on local or remote computer.

```
netuptime [hostname]
```

### `netuse_list`

List all bound share resources or info about a specific local resource.

```
netuse_list
netuse_list [target]
```

### `netuse_add`

Bind a new connection to a remote share.

```
netuse_add [sharename] [opt:username] [opt:password]
netuse_add \\dc01\C$ Administrator P@ssw0rd /DEVICE:Z:
netuse_add \\dc01\SYSVOL "" "" /PERSIST
```

### `netuse_delete`

Delete a bound device or share.

```
netuse_delete [device||sharename]
netuse_delete Z: /FORCE
netuse_delete \\dc01\C$ /PERSIST
```

---

## User & Group Enumeration

### `netuser`

Get detailed info about a specific user, local or domain.

```
netuser [username]
netuser [username] [domain]
netuser Administrator corp.local
```

### `get_password_policy`

Get password policy and lockout settings on a server or domain.

```
get_password_policy [hostname]
get_password_policy dc01.corp.local
```

### `netGroupList`

List groups from the default or a specified domain.

```
netGroupList
netGroupList [domain]
```

### `netGroupListMembers`

List members of a domain group.

```
netGroupListMembers [groupname]
netGroupListMembers [groupname] [domain]
netGroupListMembers "Domain Admins"
netGroupListMembers "Domain Admins" corp.local
```

### `netLocalGroupList`

List local groups on local or remote computer.

```
netLocalGroupList
netLocalGroupList [server]
```

### `netLocalGroupListMembers`

List members of a local group.

```
netLocalGroupListMembers [groupname]
netLocalGroupListMembers [groupname] [server]
netLocalGroupListMembers Administrators
netLocalGroupListMembers Administrators srv01.corp.local
```

### `netLocalGroupListMembers2`

BOFHound-compatible version. Group name and server are both optional.

```
netLocalGroupListMembers2
netLocalGroupListMembers2 [groupname] [server]
```

---

## Active Directory & LDAP

### `ldapsearch`

Execute arbitrary LDAP queries against the domain.

```
ldapsearch --dn "DC=corp,DC=local" --attributes "samaccountname,memberof"
ldapsearch --dn "DC=corp,DC=local" --attributes "*,ntsecuritydescriptor"
ldapsearch --dn "DC=corp,DC=local" --attributes "samaccountname" --count 50
ldapsearch --hostname dc01.corp.local --dn "DC=corp,DC=local" --ldaps
```

> Use `*,ntsecuritydescriptor` to get all attributes + base64 ACLs for BOFHound.
> 

### `ldapsecuritycheck`

Check LDAP signing and LDAPS channel binding requirements on DCs. Auto-discovers DC if omitted.

```
ldapsecuritycheck
ldapsecuritycheck [dc]
ldapsecuritycheck dc01.corp.local
```

> Generates Event ID 2889 when LDAP signing is required but not enforced.
> 

### `adcs_enum`

Enumerate Certificate Authorities and templates using Win32 functions.

```
adcs_enum
```

### `adcs_enum_com`

Enumerate CAs and templates via ICertConfig COM object.

```
adcs_enum_com
```

### `adcs_enum_com2`

Enumerate CAs and templates via IX509PolicyServerListManager COM object.

```
adcs_enum_com2
```

---

## Registry

### `reg_query`

Query a registry value or enumerate a single key. Supports remote hosts.

```
reg_query [hive] [path]
reg_query [hive] [path] [value]
reg_query [hostname] [hive] [path] [value]

reg_query HKLM "SOFTWARE\Microsoft\Windows NT\CurrentVersion" ProductName
reg_query dc01.corp.local HKLM "SYSTEM\CurrentControlSet\Control\Lsa" LmCompatibilityLevel
```

### `reg_query_recursive`

Recursively enumerate a registry key. Supports remote hosts.

```
reg_query_recursive [hive] [path]
reg_query_recursive [hostname] [hive] [path]

reg_query_recursive HKLM "SOFTWARE\Microsoft\Windows\CurrentVersion\Run"
```

---

## Services & Scheduled Tasks

### `sc_enum`

Enumerate all services — qc, query, qfailure, and qtriggers info.

```
sc_enum
sc_enum [server]
```

### `sc_query`

Query service status (like `sc query`).

```
sc_query
sc_query [service_name]
sc_query [service_name] [server]
sc_query wuauserv
```

### `sc_qc`

Query service configuration (like `sc qc`).

```
sc_qc [service_name]
sc_qc [service_name] [server]
sc_qc wuauserv
```

### `sc_qdescription`

Query service description.

```
sc_qdescription [service_name]
sc_qdescription [service_name] [server]
```

### `sc_qfailure`

Query service failure/recovery conditions.

```
sc_qfailure [service_name]
sc_qfailure [service_name] [server]
```

### `sc_qtriggerinfo`

Query service trigger conditions.

```
sc_qtriggerinfo [service_name]
sc_qtriggerinfo [service_name] [server]
```

### `schtasksenum`

Enumerate all scheduled tasks on local or remote computer.

```
schtasksenum
schtasksenum [server]
```

### `schtasksquery`

Query a specific scheduled task by path.

```
schtasksquery [taskpath]
schtasksquery [server] [taskpath]
schtasksquery \Microsoft\Windows\WindowsUpdate\Scheduled Start
```

---

## File System & Hashing

### `dir`

List files in a directory. Supports wildcards. Better than CS built-in `ls` (no wildcard support).

```
dir [directory]
dir [directory] /s
dir C:\Users
dir C:\Windows\S* /s
```

### `cacls`

List user permissions for a file. Wildcards supported.

```
cacls [filepath]
cacls C:\Windows\System32\lsass.exe
cacls "C:\Program Files\*"
```

### `md5`

Hash a file with MD5.

```
md5 [filename]
md5 C:\Windows\System32\lsass.exe
```

### `sha1`

Hash a file with SHA1.

```
sha1 [filename]
```

### `sha256`

Hash a file with SHA256.

```
sha256 [filename]
```

---

## Security & Audit

### `adv_audit_policies`

Retrieve advanced security audit policies configured on the host.

```
adv_audit_policies
```

### `get_dpapi_system`

Get the `DPAPI_SYSTEM` key and bootkey (requires SYSTEM).

```
get_dpapi_system
```

### `listpipes`

List all named pipes on the system.

```
listpipes
```

### `windowlist`

List visible windows in the current user session.

```
windowlist
windowlist all
```

---

## Misc / WMI

### `wmi_query`

Run a WMI query and return results as CSV.

```
wmi_query [query]
wmi_query [query] [server]
wmi_query [query] [server] [namespace]

wmi_query "SELECT * FROM Win32_Product"
wmi_query "SELECT Name,ProcessId FROM Win32_Process" srv01.corp.local
```

### `vssenum`

Enumerate Volume Shadow Copies (Server 2012+).

```
vssenum [hostname]
vssenum [hostname] [sharename]
vssenum dc01.corp.local
```

---

## Common Chains

### First-look checklist on a new beacon

```
whoami
ipconfig
get_session_info
uptime
netstat
arp
routeprint
tasklist
driversigs
listmods
listpipes
windowlist
```

### Domain recon from beacon

```
netview
netGroupList
netGroupListMembers "Domain Admins"
netGroupListMembers "Enterprise Admins"
ldapsearch --dn "DC=corp,DC=local" --attributes "samaccountname,memberof,description"
adcs_enum
ldapsecuritycheck
```

### Identify AV/EDR presence

```
driversigs
listmods
enum_filter_driver
sc_enum
tasklist
```

### Enumerate a remote target before moving

```
probe [host] 445
probe [host] 5985
netsession [host]
netshares [host]
netloggedon [host]
sc_enum [host]
schtasksenum [host]
reg_query [host] HKLM "SOFTWARE\Microsoft\Windows NT\CurrentVersion" ProductName
```

### LDAP full dump for BOFHound

```
ldapsearch --dn "DC=corp,DC=local" --attributes "*,ntsecuritydescriptor"
netLocalGroupListMembers2
netloggedon2
netsession2
regsession
```