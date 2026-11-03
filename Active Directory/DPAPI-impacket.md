# DPAPI-impacket

👉 **decrypt data protected by Windows DPAPI**

Windows uses DPAPI to encrypt sensitive stuff like:

- saved credentials (Credential Manager)
- browser passwords (Chrome/Edge old versions)
- RDP saved creds
- WiFi passwords
- private keys / certificates
- vault files

we can use mimikatz for same but it need system privs but impacket doesnt.

### 🔴 Mimikatz

- Runs **on the target**
- Pulls keys from:
    - LSASS memory
    - SYSTEM secrets
- Usually needs:
👉 **admin / SYSTEM**

✔ Pros:

- fast
- automatic
- minimal setup

❌ Cons:

- requires high privileges

---

### 🟢 Impacket (dpapi.py)

- Runs **off the target (on Kali)**
- Uses:
    - password / NT hash
    - exported masterkeys
    - or SYSTEM hives

✔ Pros:

- can work **without admin on target**
- good for offline decryption

❌ Cons:

- YOU must supply the keys manually

---

# 🧠 DPAPI — What it is (1-liner for notes)

**DPAPI**

→ Windows encryption system that protects user/system secrets using keys derived from:

- user password (user DPAPI)
- machine secrets (system DPAPI)

---

# 🔑 DPAPI Attack Workflow (REAL)

## 1️⃣ Find interesting files

Typical locations:

```
AppData\Roaming\Microsoft\Credentials\
AppData\Roaming\Microsoft\Protect\
AppData\Local\Microsoft\Vault\
```

---

## 2️⃣ Identify blob → extract MasterKey GUID

```
dpapi.py credential -file credential.bin

```

Output:

```
Guid MasterKey : XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX
```

---

## 3️⃣ Get matching masterkey file

From target:

```
dir"$env:APPDATA\Microsoft\Protect\<SID>\"
```

Match filename with GUID.

---

## 4️⃣ Decrypt masterkey

```
dpapi.py masterkey \
-file <MASTERKEY_FILE> \
-sid S-1-5-21-XXXX \
-password'Password123!'
```

Output:

```
[+] MasterKey decrypted: 0xABCDEF...
```

---

## 5️⃣ Decrypt credential blob

```
dpapi.py credential \
-file credential.bin \
-key 0xABCDEF...
```

👉 Output:

- username
- password / secret
- target

---

# 🔥 SYSTEM DPAPI (your case)

If you see:

```
CRYPTPROTECT_SYSTEM
```

👉 user password is NOT enough

---

## Required:

Dump SYSTEM + SECURITY hives:

```
regsaveHKLM\SYSTEMsystem.hive
regsaveHKLM\SECURITYsecurity.hive
```

---

## Extract machine DPAPI keys:

```
dpapi.py system-system system.hive-security security.hive
```

---

Then use extracted keys to decrypt blob.

---

# ⚙️ Alternative with Mimikatz

Using Mimikatz:

```
privilege::debug
dpapi::cred /in:credential.bin
```

OR:

```
sekurlsa::dpapi
```

👉 Requires admin/SYSTEM

---

# 📦 Common DPAPI Targets (IMPORTANT)

| Location | What you get |
| --- | --- |
| Credentials | saved creds |
| Vault | Windows Vault passwords |
| Chrome | browser creds |
| WiFi | network keys |
| RDP | saved sessions |

---

# 🧠 Key Concepts (put this in notes)

## 1. DPAPI Types

- User DPAPI → needs password/hash
- System DPAPI → needs SYSTEM secrets

---

## 2. Core Components

- **Blob** → encrypted data
- **MasterKey** → decrypts blob
- **SID** → ties key to user
- **Password / SYSTEM keys** → decrypt masterkey

---

## 3. Attack Logic

```
Blob → MasterKey GUID → MasterKey → Decryption Key → Secret
```

---

## 4. What tools actually do

| Tool | Role |
| --- | --- |
| Impacket dpapi.py | offline decryption |
| Mimikatz | live memory extraction |
| smbclient | file exfiltration |

---

# ⚠️ Common Mistakes (you made some of these)

- ❌ using wrong file as masterkey
- ❌ ignoring GUID matching
- ❌ trying DPAPI without key material
- ❌ confusing user vs system DPAPI

---

# 🚀 Minimal Example (CPTS-ready)

```
# 1. Identify GUID
dpapi.py credential-file cred.bin

# 2. Decrypt masterkey
dpapi.py masterkey-file masterkey-sid S-1-5-21-XXX-password'pass'

# 3. Decrypt credential
dpapi.py credential-file cred.bin-key 0xKEY
```

---

# 🧭 When to use DPAPI (exam mindset)

Use DPAPI when you have:

✔ user password or NTLM

✔ access to AppData

✔ credential/vault files

---

Avoid DPAPI when:

❌ no creds

❌ no masterkeys

❌ no SYSTEM access (for system blobs)

---

# 🔥 Final brutal truth

DPAPI is NOT:

- a cracking technique
- a shortcut

👉 It’s a **post-exploitation multiplier**

You already need access — DPAPI just **turns it into more creds**.

---

# we've a id_rsa file how can we find it's user from the file only??

```jsx
 ssh-keygen -y -f ./id_rsa                                                                                                                                             
```