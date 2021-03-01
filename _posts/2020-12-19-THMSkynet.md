---
layout: post
title: Skynet Write Up
categories: [cybersecurity, writeUps, tryHackMe]
---

This is a write-up for the room [Skynet](https://tryhackme.com/room/skynet) from [tryhackme](https://tryhackme.com).

<!-- MarkdownTOC -->

- [Initial Enumeration](#initial-enumeration)
- [Exploitation](#exploitation)
- [Privilege Escalation](#privilege-escalation)

<!-- /MarkdownTOC -->

## Initial Enumeration

We run `nmap` to enumerate open ports and services in the machine.

![nmap](https://raw.githubusercontent.com/TTWabbit/ttwabbit.github.io/master/static/img/_posts/skynet/sky1.png)

The host OS seems to be `Ubuntu`.

- Port 22, OpenSSH 7.2p2
- Port 80, Apache 2.4.18
- Port 110, pop3.
- Port 139/445, samba 4.3.11
- Port 143, imap.

The server on port 80 is this page

![port80](https://raw.githubusercontent.com/TTWabbit/ttwabbit.github.io/master/static/img/_posts/skynet/sky2.png)

Running `gobuster` against the server gives us some other interesting folders but the only one we can access is `/squirrelmail`

![squirrel](https://raw.githubusercontent.com/TTWabbit/ttwabbit.github.io/master/static/img/_posts/skynet/sky3.png)

We don't have any credentials and the version doesn't seem to be vulnerable.

Next we try to enumerate the `samba` service using `enum4linux`

![samba](https://raw.githubusercontent.com/TTWabbit/ttwabbit.github.io/master/static/img/_posts/skynet/sky4.png)

We find a username `milesdyson`. The shares are:

![samba](https://raw.githubusercontent.com/TTWabbit/ttwabbit.github.io/master/static/img/_posts/skynet/sky5.png)

We connect to the `anonymous` share and have access to some files

![files](https://raw.githubusercontent.com/TTWabbit/ttwabbit.github.io/master/static/img/_posts/skynet/sky6.png)

The content of the files is

![attention.txt](https://raw.githubusercontent.com/TTWabbit/ttwabbit.github.io/master/static/img/_posts/skynet/sky7.png)

![log1.txt](https://raw.githubusercontent.com/TTWabbit/ttwabbit.github.io/master/static/img/_posts/skynet/sky8.png)

Using `hydra` to brute-force the squirrel mail login page we obtain some credentials.

![hydra](https://raw.githubusercontent.com/TTWabbit/ttwabbit.github.io/master/static/img/_posts/skynet/sky9.png)

The only useful email seems to be this one

![email](https://raw.githubusercontent.com/TTWabbit/ttwabbit.github.io/master/static/img/_posts/skynet/sky10.png)

With this new credentials we connect to the share, under the `/notes` directory we find a file `important.txt` with the content

![important.txt](https://raw.githubusercontent.com/TTWabbit/ttwabbit.github.io/master/static/img/_posts/skynet/sky11.png)

If we navigate to this directory we can see this page

![secret directory](https://raw.githubusercontent.com/TTWabbit/ttwabbit.github.io/master/static/img/_posts/skynet/sky12.png)

## Exploitation

Using `gobuster` we find the directory `/administrator`, we have the login page of `Cuppa CMS`. According to [this entry](https://www.exploit-db.com/exploits/25971) this `cms` is vulnerable to local and remote file inclusion.

To test it we try to read `/etc/passwd` having success.

![passwd](https://raw.githubusercontent.com/TTWabbit/ttwabbit.github.io/master/static/img/_posts/skynet/sky13.png)

Now we use a `php-reverse-shell` and upload it obtaining a shell into the system.

![payload](https://raw.githubusercontent.com/TTWabbit/ttwabbit.github.io/master/static/img/_posts/skynet/sky14.png)

![shell](https://raw.githubusercontent.com/TTWabbit/ttwabbit.github.io/master/static/img/_posts/skynet/sky15.png)

We can now access the user flag.

## Privilege Escalation

In my case I used a kernel exploit to escalate privileges in this machine. I downloaded [this exploit](https://www.exploit-db.com/exploits/43418), compiled it and run it to root.

![root](https://raw.githubusercontent.com/TTWabbit/ttwabbit.github.io/master/static/img/_posts/skynet/sky16.png)

The intended method on the other hand is to exploit a `cronjob` this machine has running every minute. The program executed is named `backup.sh` and is vulnerable to a wildcard expansion of the `tar program`. This method is explained in [this](https://blog.tryhackme.com/skynet-writeup/) other write-up.