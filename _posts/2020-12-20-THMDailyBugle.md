---
layout: post
title: Daily Bugle Write Up
categories: [cybersecurity, writeUps, tryHackMe]
---

This is a write-up for the machine [Daily Bugle](https://tryhackme.com/room/dailybugle) from [tryhackme](https://tryhackme.com).

<!-- MarkdownTOC -->

- [Initial Enumeration](#initial-enumeration)
- [Exploiting SQLI to access Joomla](#exploiting-sqli-to-access-joomla)
- [Gaining a Shell](#gaining-a-shell)
- [Local Enumeration](#local-enumeration)
- [Privilege Escalation](#privilege-escalation)

<!-- /MarkdownTOC -->

## Initial Enumeration

`Nmap` scan to check for open ports and services.

![nmap](https://raw.githubusercontent.com/TTWabbit/ttwabbit.github.io/master/static/img/_posts/dailybugle/dai1.png)

The host seems to be running `CentOS` as OS.

- Port 22 OpenSSH 7.4
- Port 80 Apache 2.4.6
- Port 3306 Maria DB

The `nmap` scan already have found 15 disallowed entries in a `robots.txt` file. Running `gobuster` against the server yields this results.

![gobuster](https://raw.githubusercontent.com/TTWabbit/ttwabbit.github.io/master/static/img/_posts/dailybugle/dai2.png)

The web server shows this page.

![web](https://raw.githubusercontent.com/TTWabbit/ttwabbit.github.io/master/static/img/_posts/dailybugle/dai3.png)

In the `/administrator` folder we have the `Joomla` login page. A vulnerability scan using `nmap` tells us that this version of `Joomla` is vulnerable to `SQLI`. We follow the information from [this exploit](https://www.exploit-db.com/exploits/42033).

## Exploiting SQLI to access Joomla

We use this commands to dump the tables, the `user` table columns and the `user` and `password` columns. 

````shell
sqlmap -u "http://10.10.173.19/index.php?option=com_fields&view=fields&layout=modal&list[fullordering]=updatexml" --risk=3 --level=5 --random-agent --dbs -p list[fullordering] --batch --tables

sqlmap -u "http://10.10.173.19/index.php?option=com_fields&view=fields&layout=modal&list[fullordering]=updatexml" --risk=3 --level=5 --random-agent --dbs -p list[fullordering] --batch --columns -D mysql -T user

sqlmap -u "http://10.10.173.19/index.php?option=com_fields&view=fields&layout=modal&list[fullordering]=updatexml" --risk=3 --level=5 --random-agent --dbs -p list[fullordering] --batch --dump -D mysql -T user -C User,Password
````

![user](https://raw.githubusercontent.com/TTWabbit/ttwabbit.github.io/master/static/img/_posts/dailybugle/dai4.png)

But the result is not useful, we are looking for a `Jonah` user. Finally we dump the table `#__users` and then we find the user credentials we were looking for.

````shell
sqlmap -u "http://10.10.173.19/index.php?option=com_fields&view=fields&layout=modal&list[fullordering]=updatexml" --risk=3 --level=5 --random-agent --dbs -p list[fullordering] -D joomla -T "#__users" --dump -C name,username,email,password
````

![table](https://raw.githubusercontent.com/TTWabbit/ttwabbit.github.io/master/static/img/_posts/dailybugle/dai5.png)

With `john` we can crack this password.

![john](https://raw.githubusercontent.com/TTWabbit/ttwabbit.github.io/master/static/img/_posts/dailybugle/dai6.png)

And with this credentials we can access the `joomla` control panel.

## Gaining a Shell

In [this](https://www.hackingarticles.in/joomla-reverse-shell/) article we can find the way to use Joomla to gain a shell into the system.

The steps we have to follow are

1. Go to extensions/templates.
2. Choose a template.
3. Go to index.php.
4. Edit the template adding a php reverse shell code.
5. Start a listener to the port selected.
6. Navigate to index.php.

Following those steps we gain a shell as the user `apache`.

![shell](https://raw.githubusercontent.com/TTWabbit/ttwabbit.github.io/master/static/img/_posts/dailybugle/dai7.png)

## Local Enumeration

The only user on the machine is `jjameson` and we can't access his files. Enumerating the `/var/www` directory we find some credentials in the `configuration.php` file.

![credentials](https://raw.githubusercontent.com/TTWabbit/ttwabbit.github.io/master/static/img/_posts/dailybugle/dai8.png)

If we try to `ssh` to the machine as `jjameson` with that password we have access and can now read the user flag.

![ssh](https://raw.githubusercontent.com/TTWabbit/ttwabbit.github.io/master/static/img/_posts/dailybugle/dai9.png)

## Privilege Escalation

Running `sudo -l` as `jjameson` yields this result.

![sudo -l](https://raw.githubusercontent.com/TTWabbit/ttwabbit.github.io/master/static/img/_posts/dailybugle/dai10.png)

According to [this entry](https://gtfobins.github.io/gtfobins/yum/#sudo) on `gtfobins` this can be used to escalate privileges. 

![root](https://raw.githubusercontent.com/TTWabbit/ttwabbit.github.io/master/static/img/_posts/dailybugle/dai11.png)

