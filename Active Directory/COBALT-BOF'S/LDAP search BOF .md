# LDAP search BOF

All queries use the TrustedSec CS-Situational-Awareness BOF `ldapsearch`, not OpenLDAP `ldapsearch`. Syntax: `ldapsearch <filter> [--attributes] [--count] [--scope] [--hostname] [--dn] [--ldaps]`. Default scope is subtree, `--dn` sets the search base, `--hostname` targets a specific DC.

## Domains

| Generic | Role |
| --- | --- |
| megacorp.com | Local domain |
| foreign.com | Foreign domain |

## Machines

| Generic | Role |
| --- | --- |
| DC-1 | Domain controller (megacorp.com) |
| WEB-1 | Web server |
| SQL-1 | SQL server |
| WS-1 | Workstation |
| DC-1.foreign.com | Foreign domain controller |
| FS-1 | File server (foreign.com) |

## Discovery - Users

```
# List all users
beacon> ldapsearch (samAccountType=805306368)

# All users with minimal attributes for BloodHound
beacon> ldapsearch (samAccountType=805306368) --attributes samaccounttype,distinguishedname,objectsid,serviceprincipalname,useraccountcontrol,ntsecuritydescriptor

# Admin-protected accounts (adminCount=1)
beacon> ldapsearch (&(samAccountType=805306368)(adminCount=1)) --attributes name,memberof

# Admin-protected excluding krbtgt
beacon> ldapsearch (&(samAccountType=805306368)(adminCount=1)(!(name=krbtgt)))

# Users with "admin" in description or username
beacon> ldapsearch (&(samAccountType=805306368)(|(description=*admin*)(samaccountname=*adm*)))

# Specific user by samAccountName (check UAC, group membership, delegation flags)
beacon> ldapsearch (samAccountName=USERNAME) --attributes userAccountControl,memberOf

# Specific user by SID
beacon> ldapsearch (objectsid=S-1-5-21-DOMAIN_SID-1108) --attributes *,ntsecuritydescriptor

# AS-REP roastable (no preauth required)
beacon> ldapsearch (&(samAccountType=805306368)(userAccountControl:1.2.840.113556.1.4.803:=4194304)) --attributes samaccountname

# Users with SPNs (kerberoastable) - WARNING: check for honeypots before roasting
beacon> ldapsearch (&(samAccountType=805306368)(servicePrincipalName=*)) --attributes samaccountname,serviceprincipalname

# MSSQL service accounts (use wildcards to broaden, e.g. *SQL*)
beacon> ldapsearch (&(samAccountType=805306368)(servicePrincipalName=MSSQLSvc*)) --attributes name,samAccountName,servicePrincipalName

# Domain Admins (local domain)
beacon> ldapsearch "(&(samAccountType=805306368)(memberOf=CN=Domain Admins,CN=Users,DC=MEGACORP,DC=COM))" --attributes samAccountName,userAccountControl

# Domain Admins in foreign domain
beacon> ldapsearch "(&(samAccountType=805306368)(memberOf=CN=Domain Admins,CN=Users,DC=FOREIGN,DC=COM))" --hostname dc-1.FOREIGN.COM --dn DC=FOREIGN,DC=COM --attributes samAccountName

# Enterprise Admins SID (for golden/diamond ticket /sids parameter)
beacon> ldapsearch "(&(samAccountType=268435456)(samAccountName=Enterprise Admins))" --hostname dc-1.FOREIGN.COM --dn DC=FOREIGN,DC=COM --attributes objectSid
```

## Discovery - Computers

```
# Computer objects in current domain / context
beacon> ldapsearch (samAccountType=805306369) --attributes samAccountName,dNSHostName

# Computers in foreign domain / context
beacon> ldapsearch (samAccountType=805306369) --hostname dc-1.FOREIGN.COM --dn DC=FOREIGN,DC=COM --attributes samAccountName,dNSHostName

# Unconstrained delegation computers (DCs always have unconstrained delegation - ignore)
beacon> ldapsearch (&(samAccountType=805306369)(userAccountControl:1.2.840.113556.1.4.803:=524288)) --attributes samaccountname

# Constrained delegation computer accounts
beacon> ldapsearch (&(samAccountType=805306369)(msDS-AllowedToDelegateTo=*)) --attributes samAccountName,msDS-AllowedToDelegateTo,userAccountControl

# Constrained delegation user/service accounts
beacon> ldapsearch (&(samAccountType=805306368)(msDS-AllowedToDelegateTo=*)) --attributes samAccountName,msDS-AllowedToDelegateTo,userAccountControl

# Protocol transition check for a single host (returns row if enabled, 0 results if not)
beacon> ldapsearch (&(samAccountName=WEB-1$)(userAccountControl:1.2.840.113556.1.4.803:=16777216)) --attributes samAccountName

# Check UAC flags for a specific computer
beacon> ldapsearch (&(samAccountType=805306369)(samaccountname=WEB-1$)) --attributes userAccountControl

# Protocol transition enabled computers (TRUSTED_TO_AUTH_FOR_DELEGATION)
beacon> ldapsearch (&(samAccountType=805306369)(userAccountControl:1.2.840.113556.1.4.803:=16777216)) --attributes samAccountName,msDS-AllowedToDelegateTo

# RBCD configured computers
beacon> ldapsearch (msDS-AllowedToActOnBehalfOfOtherIdentity=*) --attributes samAccountName,msDS-AllowedToActOnBehalfOfOtherIdentity

# Get SID for a principal in foreign domain
beacon> ldapsearch (samAccountName=FS-1$) --hostname dc-1.FOREIGN.COM --dn DC=FOREIGN,DC=COM --attributes objectSid

# Get their DN (find which OU it's in, check GPO scope)
beacon> ldapsearch (samAccountName=FS-1$) --hostname dc-1.FOREIGN.COM --dn DC=FOREIGN,DC=COM --attributes distinguishedName

# Check ACL (check WriteProperty for RBCD abuse)
beacon> ldapsearch (samAccountName=FS-1$) --hostname dc-1.FOREIGN.COM --dn DC=FOREIGN,DC=COM --attributes ntsecuritydescriptor
```

## Discovery - Groups

```
# All groups
beacon> ldapsearch (samAccountType=268435456)

# Specific group by name (spaces need quotes)
beacon> ldapsearch "(cn=SQL Admins)" --attributes objectSid,member
beacon> ldapsearch "(samAccountName=SQL Admins)" --attributes objectSid

# SQL/DB related groups
beacon> ldapsearch (&(samAccountType=268435456)(|(name=*SQL*)(name=*DB*)(name=*Database*))) --attributes distinguishedName,member

# Group members by memberOf (direct, non-transitive)
beacon> ldapsearch (memberOf=CN=Cross Domain Access,CN=Users,DC=MEGACORP,DC=COM) --attributes samAccountName

# Transitive group membership (unrolls nested groups via IN_CHAIN)
beacon> ldapsearch "(memberOf:1.2.840.113556.1.4.1941:=CN=Domain Admins,CN=Users,DC=MEGACORP,DC=COM)" --attributes samaccountname

# Trust accounts
beacon> ldapsearch (samAccountType=805306370) --attributes samAccountName

# Resolve any SID to its object (group, user, computer)
beacon> ldapsearch (objectSid=S-1-5-21-DOMAIN_SID-3103) --attributes samAccountName,objectClass,member
beacon> ldapsearch (objectSid=S-1-5-21-FOREIGN_SID-2103) --hostname dc-1.FOREIGN.COM --dn DC=FOREIGN,DC=COM --attributes samAccountName,objectClass,member
```

## Discovery - Domain / OUs / GPOs

```
# Domain object (get domain SID)
beacon> ldapsearch (objectClass=domain) --attributes objectSid

# Foreign domain SID via domain name (DNS resolves to available DC, no need to know DC hostname)
beacon> ldapsearch (objectClass=domain) --dn DC=FOREIGN,DC=COM --attributes name,objectSid --hostname FOREIGN.COM

# Foreign domain SID
beacon> ldapsearch (objectClass=domain) --hostname dc-1.FOREIGN.COM --dn DC=FOREIGN,DC=COM --attributes objectSid

# All GPOs (names and SYSVOL paths)
beacon> ldapsearch (objectClass=groupPolicyContainer) --attributes displayName,gPCFileSysPath

# All GPOs (names only, quick reference)
beacon> ldapsearch (objectClass=groupPolicyContainer) --attributes name,displayName

# GPOs with WMI filter links (which GPOs have OS-based targeting applied)
beacon> ldapsearch (objectClass=groupPolicyContainer) --attributes displayName,gPCWQLFilter

# GPOs in foreign domain
beacon> ldapsearch (objectClass=groupPolicyContainer) --hostname dc-1.FOREIGN.COM --dn DC=FOREIGN,DC=COM --attributes displayName,gPCFileSysPath

# GPOs with specific name
beacon> ldapsearch (displayName=*SQL*) --dn "CN=Policies,CN=System,DC=MEGACORP,DC=COM" --attributes displayName,gPCFileSysPath

# Where is a GPO linked, ie which OU (gPLink attribute value isn't just the GUID, wildacards match that withing that longer string)
beacon> ldapsearch (&(|(objectClass=organizationalUnit)(objectClass=domain))(gPLink=*ABCD1234*)) --attributes objectClass,name
beacon> ldapsearch (&(|(objectClass=organizationalUnit)(objectClass=domain))(gPLink=*ABCD1234*)) --hostname dc-1.FOREIGN.COM --dn DC=FOREIGN,DC=COM --attributes name,distinguishedName

# Check gPLink on a specific OU (does this OU have any GPOs?)
beacon> ldapsearch (&(objectClass=organizationalUnit)(name=FS)) --hostname dc-1.FOREIGN.COM --dn DC=FOREIGN,DC=COM --attributes gPLink
beacon> ldapsearch (distinguishedName=OU=Servers,DC=FOREIGN,DC=COM) --hostname dc-1.FOREIGN.COM --dn DC=FOREIGN,DC=COM --attributes gPLink

# Enumerate WMI filters (OS version targeting used by AppLocker/GPO scoping)
beacon> ldapsearch (objectClass=msWMI-Som) --attributes name,msWMI-Name,msWMI-Parm2 --dn "CN=SOM,CN=WMIPolicy,CN=System,DC=FOREIGN,DC=COM"
```

## Discovery - Trusts

```
# All trusts (from current domain)
beacon> ldapsearch (objectClass=trustedDomain) --attributes name,trustDirection,trustAttributes,trustPartner,flatName

# Trusts from parent domain
beacon> ldapsearch (objectClass=trustedDomain) --hostname dc-1.FOREIGN.COM --dn DC=FOREIGN,DC=COM --attributes trustPartner,trustDirection,trustAttributes,flatName

# Resolve trust NetBIOS name to objectGUID (needed for GUID-based DCSync of trust key)
beacon> ldapsearch (flatName=FOREIGN) --attributes objectGUID
```

## Discovery - ADCS

```
# Certificate Authorities
beacon> ldapsearch "(objectClass=pKIEnrollmentService)" --dn "CN=Enrollment Services,CN=Public Key Services,CN=Services,CN=Configuration,DC=MEGACORP,DC=COM" --attributes name,dNSHostName,certificateTemplates

# Certificate Authorities in foreign domain
beacon> ldapsearch "(objectClass=pKIEnrollmentService)" --hostname dc-1.FOREIGN.COM --dn "CN=Enrollment Services,CN=Public Key Services,CN=Services,CN=Configuration,DC=FOREIGN,DC=COM" --attributes name,dNSHostName,certificateTemplates

# Certificate Templates
beacon> ldapsearch "(objectClass=pKICertificateTemplate)" --dn "CN=Certificate Templates,CN=Public Key Services,CN=Services,CN=Configuration,DC=MEGACORP,DC=COM" --attributes name,msPKI-Certificate-Name-Flag,msPKI-Enrollment-Flag,pKIExtendedKeyUsage,nTSecurityDescriptor
```

## Discovery - Foreign Security Principals (FSPs)

```
# FSPs in a foreign domain (find cross-forest access principals)
# Ignore well-known: S-1-5-4, S-1-5-9, S-1-5-11, S-1-5-17
beacon> ldapsearch (objectClass=foreignSecurityPrincipal) --hostname dc-1.FOREIGN.COM --dn DC=FOREIGN,DC=COM --attributes objectSid,memberOf

# FSP full attributes (when memberOf is blank, check all attributes)
beacon> ldapsearch (objectSid=S-1-5-21-FOREIGN_SID-2601) --hostname dc-1.FOREIGN.COM --dn DC=FOREIGN,DC=COM --attributes *

# Resolve FSP SID back to source domain object
beacon> ldapsearch (objectSid=S-1-5-21-DOMAIN_SID-2601) --attributes samAccountName,objectClass,memberOf
```

## Bulk Collection for BOFHound

```
beacon> ldapsearch (|(objectClass=domain)(objectClass=organizationalUnit)(objectClass=groupPolicyContainer)) --attributes *,ntsecuritydescriptor
beacon> ldapsearch (|(samAccountType=805306368)(samAccountType=805306369)(samAccountType=268435456)) --attributes *,ntsecuritydescriptor
```