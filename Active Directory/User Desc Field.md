# User Desc Field

![image.png](User%20Desc%20Field/image.png)

```bash
nxc ldap <DC_IP> -u <USER> -p '<PASSWORD>' -M get-desc-users
```

```bash
ldapsearch -H ldap://<DC_IP> -x -D "<USER>" -w '<PASSWORD>' -b "DC=<DOMAIN>,DC=<TLD>" "(objectClass=user)" description samAccountName
```