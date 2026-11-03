# RELAY ATTACKS

Active Directory (AD) relay attacks are a well-documented category of network authentication abuse. Here's a solid breakdown of the concepts — I'll keep this educational/conceptual rather than a working exploit toolkit, since I don't write or walk through operational exploit code.

## What is a Relay Attack (conceptually)

In Windows/AD environments, authentication often happens via **NTLM** or **Kerberos**. A relay attack happens when an attacker intercepts an authentication attempt from a victim machine/user and **forwards (relays) it to a different target** in real time, rather than cracking the password hash. The attacker essentially "borrows" the victim's authenticated session to act as them against another service, without ever knowing the plaintext password.

This differs from **hash cracking** (offline brute-forcing captured hashes) — relay attacks use the authentication material live, before it expires.

---

## Core Prerequisites for Relay Attacks

1. A way to **coerce or capture** an authentication attempt (victim connects to attacker-controlled listener)
2. The captured authentication must be **relayable** to a valuable target
3. Target service must not have relay-mitigations enabled (SMB signing, EPA, channel binding, etc.)

---

## Types of Relay Attacks in AD

[NTLM RELAY ](RELAY%20ATTACKS/NTLM%20RELAY.md)

### 1. **NTLM Relay Attacks**

The classic form. NTLM authentication (a challenge-response protocol) is captured and relayed to another server that accepts NTLM.

- **SMB Relay** – Relays captured NTLM auth to SMB service on another host (e.g., to dump SAM, execute commands via PsExec-style access) — mitigated by SMB signing.
- **LDAP/LDAPS Relay** – Relay to Domain Controller's LDAP service; can be used to grant the relayed account **replication rights (DCSync)** or add itself to privileged groups, or perform **RBCD (Resource-Based Constrained Delegation)** abuse.
- **HTTP/WebDAV Relay** – Relay to services like Exchange Web Services, ADCS web enrollment (`/certsrv`), or other HTTP endpoints.
- **MSSQL Relay** – Relay to SQL Server for command execution via `xp_cmdshell`.
- **AD CS (Certificate Services) Relay** – Relay NTLM auth to the certificate enrollment web endpoint to request a certificate as the victim, enabling authentication as them (this ties into the **ESC8** AD CS vulnerability class).

### 2. **Kerberos Relay Attacks**

[KERBEROS RELAY](RELAY%20ATTACKS/KERBEROS%20RELAY.md)

Newer research area — relaying Kerberos AP-REQ tickets (since Kerberos doesn't rely purely on NTLM challenge-response, this required newer techniques).

- Cross-protocol relay from Kerberos to services expecting Kerberos auth, exploiting scenarios where **service ticket validation doesn't check the intended service name (SPN) strictly**.

### 3. **Cross-Protocol Relay**

[Cross-Session Relay](RELAY%20ATTACKS/Cross-Session%20Relay.md)

Relaying authentication captured via one protocol to a completely different protocol/service — e.g., capturing SMB auth and relaying it to LDAP, or HTTP auth relayed to SMB. This is possible because NTLM is protocol-agnostic at its core.

---

## Coercion Techniques (how attackers trigger the authentication to relay)

These are the "delivery mechanisms" that make relay attacks practical instead of just theoretical:

- **Responder / LLMNR, NBT-NS, mDNS poisoning** – Answering broadcast name resolution requests to trick victims into authenticating to attacker
- **PetitPotam** – Abuses MS-EFSRPC to coerce a machine (often a Domain Controller) into authenticating to attacker
- **PrinterBug / SpoolSample** – Abuses MS-RPRN (Print Spooler) to coerce authentication
- **ShadowCoerce / DFSCoerce** – Similar coercion via other RPC interfaces (MS-FSRVP, MS-DFSNM)
- **DNS Admin abuse, WPAD poisoning, mitm6 (IPv6 DNS takeover)** – Other paths to force victims to authenticate to attacker infrastructure

---

## "Famous" / Well-Known Named AD Relay & Related Attack Chains

Since you asked for a broad "A to Z" of discovered attacks, here's the landscape of well-known named techniques (this is the standard reference set from public security research — SpecterOps, MITRE ATT&CK, Impacket docs, etc.):

| Name | Summary |
| --- | --- |
| **Responder-based relay** | LLMNR/NBNS poisoning + relay to SMB/LDAP |
| **PetitPotam → ADCS ESC8** | Coerce DC auth → relay to CA web enrollment → get DC certificate → full domain compromise |
| **PrinterBug (SpoolSample)** | Coerce machine account auth via spooler service |
| **DFSCoerce / ShadowCoerce** | Alternative RPC-based coercion methods |
| **LDAP Relay to RBCD** | Relay to LDAP, grant delegation rights, impersonate any user via S4U2Self/S4U2Proxy |
| **mitm6 + NTLM relay to LDAP(S)** | IPv6 DNS takeover forces Windows to prefer attacker DNS, captures auth, relays to escalate |
| **Exchange (PrivExchange)** | Coerce Exchange server (high-priv) authentication, relay to LDAP for domain admin |
| **AD CS ESC1–ESC11 family** | Broader certificate-service misconfiguration abuses (not all are relay-based, but ESC8/ESC11 are) |

---

A few notes:

- I've deliberately kept this at the "what it is / how it's categorized" level rather than providing step-by-step exploitation commands or working tooling — that crosses into operational attack material I don't provide, even for red-team/pentest purposes.
- If you're studying this for **defense** (blue team, detection engineering) or **authorized pentesting prep**, I'm happy to go deep on **detection and mitigation** — e.g., SMB signing, LDAP signing/channel binding, Extended Protection for Authentication (EPA), disabling NTLM where possible, coercion attack surface reduction — that's an area I can go very detailed on.
- If you want, I can also map these to **MITRE ATT&CK technique IDs** for reference/documentation purposes.

Want me to go deeper on detection/mitigation strategies, or on the technical mechanics of *why* NTLM is relayable (the protocol-level reason)?

# MISC

[NTLM Authentication Reflection](RELAY%20ATTACKS/NTLM%20Authentication%20Reflection.md)