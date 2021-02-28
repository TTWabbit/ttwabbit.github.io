---
layout: post
title: THM Madeye's Castle
---

This is a write up for the room [Madeye's Castle](https://tryhackme.com/room/madeyescastle) from [tryhackme](https://tryhackme.com)

<!-- MarkdownTOC -->

- [Service Enumeration](#service-enumeration)
  - [Nmap Scan Results](#nmap-scan-results)
- [Initial Shell Vulnerability Exploited](#initial-shell-vulnerability-exploited)
  - [Vulnerability Explanation](#vulnerability-explanation)
  - [Vulnerability Fix](#vulnerability-fix)
  - [Severity](#severity)
  - [Proof of Concept Code Here](#proof-of-concept-code-here)
- [Privilege Escalation](#privilege-escalation)
  - [Vulnerability Exploited](#vulnerability-exploited)
  - [Vulnerability Explanation](#vulnerability-explanation-1)
  - [Vulnerability Fix](#vulnerability-fix-1)
  - [Severity](#severity-1)
  - [Exploit Code](#exploit-code)

<!-- /MarkdownTOC -->

## Service Enumeration

| Port | Service |
|---|---|
| 22| OpenSSH 7.6p1|
| 80| Apache http 2.4.69|
| 139/445| Samba smb| 

### Nmap Scan Results

![nmap](../images/madeyes/mdy1.png)

## Initial Shell Vulnerability Exploited

We used [enum4linux-ng](https://github.com/cddmp/enum4linux-ng) to enumerate the `smb` service.

![smb](../images/madeyes/mdy2.png)
![smb2](../images/madeyes/mdy3.png)

The `sambashare` is accessible, we find two files

![smbfiles](../images/madeyes/mdy4.png)

The first one, `spellnames.txt` seems like a list of passwords. The content of `.notes.txt` is

> Hagrid told me that spells names are not good since they will not "rock you"
Hermonine loves historical text editors along with reading old books.

The server in port 80 shows the default `apache` web page. Looking at the source code we find

![source](../images/madeyes/mdy5.png)

If we add that host to our `/etc/hosts` file we can see this other web page:

![hogwartz](../images/madeyes/mdy6.png)

Trying to brute force `ssh` or the login form with the `spellnames.txt` files doesn't gives any results.

Trying `' or 1=1 --` as username in the login form produces this result:

![sqli](../images/madeyes/mdy7.png)

So `sql injection` is possible in this login form.

### Vulnerability Explanation

SQL injection is a common attack vector. A user can introduce a malicious payload that is interpreted for the SQL database on the backend of the web server, allowing the user to access information that was not intended to be displayed.

### Vulnerability Fix

To prevent SQL injection attacks any user input has to be sanitized. The application should never use user input directly. Input validation and parametrized queries are also useful to prevent this kind of vulnerability. It is also a good idea to turn off the visibility of database errors in the production site, as they can be used to gain information about the database.  

### Severity

High (CVSS3 8.6)

### Proof of Concept Code Here

Using `sqlmap` we can't dump the database but at least we obtain the information that the database management system is SQLite.

Following the steps depicted [here](https://book.hacktricks.xyz/pentesting-web/sql-injection) we determine the number of columns of the database.

![columns](../images/madeyes/mdy8.png)
![columns](../images/madeyes/mdy9.png)

Now with the help of [this other](https://www.exploit-db.com/docs/english/41397-injecting-sqlite-database-based-applications.pdf) article we extract the table name.

![table](../images/madeyes/mdy10.png)
![table](../images/madeyes/mdy11.png)

We use this other payload to extract the name of the columns.

![columns](../images/madeyes/mdy12.png)
![columns](../images/madeyes/mdy13.png)

We can see the name of the columns are `name`, `password`, `admin` and `notes`.

With the following payload we can dump all the columns content changing the argument of the `group_concat()` for the name of the column we want to dump.

![dump](../images/madeyes/mdy14.png)
![dump](../images/madeyes/mdy15.png)

The values on the `admin` column are all '0' so it is not interesting. Using a little of bash magic we group the users with their notes and passwords in one document obtaining:

![columns](../images/madeyes/mdy16.png)

The most interesting user is `Harry Turner`, giving the information we have in his note. The `best 64` seems to be a reference of a `hashcat` rule. The password is encrypted using `SHA-512` according to this analyzer.

![columns](../images/madeyes/mdy17.png)

Using the mentioned rule and the `spellnames.txt` file as dictionary we crack the hash with the command

````bash
hashcat -m 1700 -r /usr/share/hashcat/rules/best64.rule harryhash spellnames.txt
````
With the obtained credentials we can SSH to the machine and grab the user flag.

![banner](../images/madeyes/mdy18.png)

## Privilege Escalation

The only other user in the system is `hermonine`. The files with `SUID` doesn't seem to be exploitable. Nothing interesting in the `crontab` file. We find this email in `/var/www/html/backup`

![email](../images/madeyes/mdy19.png)

Doesn't give us any new information.

The sudo version of the machine is vulnerable to the Baron Samedit exploit

![sudo exploit](../images/madeyes/mdy20.png)

But this is clearly not the intended method to root, so we keep looking.

The `sudo -l` command returns

![sudo -l](../images/madeyes/mdy21.png)

Running the command we can enter the `nano` text editor version 2.9.3. As we can run this command as `hermonine` and `nano` can be used to spawn a shell we do

````bash
sudo -u hermonine '/usr/bin/pico'
````

And once in nano with 

````bash
^R^X
reset; bash 1>&0 2>&0
````
We gain a shell as `hermonine`

![user shell](../images/madeyes/mdy22.png)

We can grab the second flag, and we incorporate an `ssh` public key to the `/.ssh/authorized_keys` of the user to improve the shell connecting remotely via `ssh` to the machine as the user `hermonine`.

The hint for the root flag is

>Time is tricky. Can you trick time

Using `linpeas` we have this finding

![linpeas](../images/madeyes/mdy23.png)

The hint and that last directory seem related. Inside it we find an executable file called `swagger` with the SUID bit activated. Running it we obtain

![swagger](../images/madeyes/mdy24.png)

We transfer the file to our local machine and analyze it with `ghidra`.

![ghidra main](../images/madeyes/mdy25.png)

As we can see this program takes a value (`local_18`) and compares it with a random number (`local_14`) generated with the `rand()` function taking the time of the system as seed. If the numbers are not equal the program displays the message we have seen previously. If the numbers are equal we execute the function `impressive()`. Here we have the decompiled version of the function `impressive()`

![ghidra impressive](../images/madeyes/mdy26.png)

In this function we have a call to the system executable `uname` without using the full path.

### Vulnerability Exploited

To elevate privileges we exploit two vulnerabilities.

First, bad generated random numbers.

Second, relative path vulnerability.

### Vulnerability Explanation

The executable file uses `srand (time)` and the `rand()` function to generate a random number. The system time is a bad seed because it is predictable within a small range. And the ANSI C `rand()` function does not generate good random numbers.

We invoke a system binary without specifying the full path, inside a program with `SUID` on. This gives any user the possibility of spoof the path and craft a binary to be executed instead of the one intended, and with elevated privileges. 

### Vulnerability Fix

Use `/dev/random` as the device to generate a seed for random numbers.

Use the full path of the binaries when invoking them inside an executable file.

### Severity

Critical

### Exploit Code

First we spoof the path and create our own `uname` version.

![path spoof](../images/madeyes/mdy27.png)

Then we try to pipe calls to the executable so we successfully guess the random number.

![ghidra impressive](../images/madeyes/mdy28.png)

However this method doesn't work because the shell is spawned but it closes immediately, even using the `-i` option for `bash`.

![ghidra impressive](../images/madeyes/mdy29.png)

Instead we use the executable to generate another executable file, copy of `/bin/bash` with `SUID` set so we can use it to spawn a shell with root privileges.

![ghidra impressive](../images/madeyes/mdy30.png)
