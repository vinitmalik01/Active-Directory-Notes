# cross-forest

# 27.Attacking Domain Trusts - Cross-Forest Trust Abuse - from Windows

**I. Cross-Forest Kerberoasting**

**Concept:** Performing Kerberoasting across domain trusts to obtain credentials for accounts in other domains.

**Steps:**

1. **Enumerate SPNs:**
    - Use PowerView:
- 
    - `Get-DomainUser SPN Domain FREIGHTLOGISTICS.LOCAL | select SamAccountName`
- **Verify account privileges:**
    - Use PowerView:
- 
    - `Get-DomainUser Domain FREIGHTLOGISTICS.LOCAL Identity mssqlsvc | select samaccountname,memberof`
- **Perform Kerberoasting:**
    - Use Rubeus with the `/domain:` flag:
1. 
    - `.\Rubeus.exe kerberoast /domain:FREIGHTLOGISTICS.LOCAL /user:mssqlsvc /nowrap`
2. **Crack the hash:**
    - Use Hashcat to crack the obtained hash.

**Key Takeaway:**

- Exploiting trusts to gain access to privileged accounts in other domains.

---

**II. Admin Password Re-Use & Group Membership**

**Password Re-Use:**

- Checking for password reuse across domains with bidirectional trusts.
- If an attacker gains credentials in one domain, they might work in another.

**Group Membership:**

- Enumerating foreign group memberships to find accounts with cross-domain privileges.

**Commands:**

`Get-DomainForeignGroupMember -Domain FREIGHTLOGISTICS.LOCAL
Convert-SidToName S-1-5-21-3842939050-3880317879-2865463114-500`

**Verify Access:**

`Enter-PSSession -ComputerName ACADEMY-EA-DC03.FREIGHTLOGISTICS.LOCAL -Credential INLANEFREIGHT\administrator`

**Key Takeaway:**

- Exploiting weak password management and misconfigured group memberships.

---

**III. SID History Abuse - Cross Forest**

**Concept:**

- Abusing SID history across forest trusts when SID filtering is not enabled.
- Allows accounts to retain privileges from their original domain after migration.

**Mechanism:**

- Adding SIDs from one forest to the `sidHistory` attribute of accounts in another.
- The user retains the rights of the added SID in their token.

**Key Takeaway:**

- Exploiting misconfigurations in trust relationships to maintain or gain privileges.

---

**IV. Important Considerations:**

- Cross-forest attacks can significantly expand an attacker's reach.
- Enumeration and verification are crucial steps.
- Password reuse is a very common issue.
- SID filtering is very important to have enabled.

# 28.Attacking Domain Trusts - Cross-Forest Trust Abuse - from Linux

**I. Cross-Forest Kerberoasting**

**Concept:** Leveraging Kerberoasting across domain trusts to extract and crack credentials from accounts in other domains.

**Steps:**

1. **Enumerate SPNs:**
    - Use PowerView to list Service Principal Names (SPNs):
- 
    - `Get-DomainUser SPN Domain FREIGHTLOGISTICS.LOCAL | select SamAccountName`
- **Verify Account Privileges:**
    - Check if the target account has privileged group memberships:
- 
    - `Get-DomainUser Domain FREIGHTLOGISTICS.LOCAL Identity mssqlsvc | select samaccountname,memberof`
- **Perform Kerberoasting:**
    - Request and extract service tickets for cracking:
1. 
    - `.\Rubeus.exe kerberoast /domain:FREIGHTLOGISTICS.LOCAL /user:mssqlsvc /nowrap`
2. **Crack the Hash:**
    - Use Hashcat (mode 13100) to crack the extracted ticket hash.

**Key Takeaways:**

- Cross-forest Kerberoasting exploits domain trust relationships.
- Privileged accounts with SPNs can be targeted for credential extraction.

---

**II. Admin Password Reuse & Group Membership**

**Password Reuse:**

- Evaluating password reuse across trusted domains with bidirectional trusts.
- If an attacker compromises credentials in one domain, they may gain access to another.

**Group Membership Enumeration:**

- Identifying cross-domain privileges through foreign group memberships.

**Commands:**

`Get-DomainForeignGroupMember -Domain FREIGHTLOGISTICS.LOCAL
Convert-SidToName S-1-5-21-3842939050-3880317879-2865463114-500`

**Verifying Access:**

- Test access using PowerShell remoting:

`Enter-PSSession -ComputerName ACADEMY-EA-DC03.FREIGHTLOGISTICS.LOCAL -Credential INLANEFREIGHT\administrator`

**Key Takeaways:**

- Weak password policies and reused credentials increase risk.
- Identifying privileged accounts across domains is crucial for security assessments.

---

**III. SID History Abuse in Cross-Forest Attacks**

**Concept:**

- Exploiting SID history in inter-forest trusts when SID filtering is not enforced.
- Migrated accounts can retain privileges from their original domain.

**Mechanism:**

- Injecting SIDs from one forest into the `sidHistory` attribute of accounts in another.
- The modified account inherits the permissions associated with the added SID.

**Key Takeaways:**

- Misconfigurations in trust relationships can lead to privilege escalation.
- Proper SID filtering must be enabled to prevent unauthorized privilege retention.

---

**IV. Security Considerations:**

- Cross-forest attacks significantly expand an attacker’s ability to escalate privileges.
- Continuous monitoring and auditing of trust relationships are essential.
- Enforcing strong password policies and SID filtering can mitigate risk.
- Properly securing privileged accounts with limited SPN exposure reduces attack surface.