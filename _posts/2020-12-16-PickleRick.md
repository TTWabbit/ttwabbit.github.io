---
layout: post
title: THM Pickle Rick Write Up
---

This is a write up for the [tryhackme](https://www.tryhackme.com "Home page of TryHackMe") room [Pickle Rick](https://tryhackme.com/room/picklerick "Pickle Rick Room")

<!-- MarkdownTOC -->

- [Enumeration](#enumeration)
- [Exploitation](#exploitation)

<!-- /MarkdownTOC -->

<blockquote cite="https://tryhackme.com/room/picklerick">This Rick and Morty themed challenge requires you to exploit a webserver to find 3 ingredients that will help Rick make his potion to transform himself back into a human from a pickle.</blockquote>

## Enumeration

First step, to enumerate the target machine and see what ports and service is running we use `nmap` .

![Nmap Results](https://raw.githubusercontent.com/TTWabbit/ttwabbit.github.io/master/static/img/_posts/picklerick/pr1.png "Nmap Results")

So there are an http service running in port 80 and a ssh service running in port 22. The information given in the introduction tells us to focus on the webserver.

We navigate to the given IP and see a static webpage. If we look at the source code

![Source Code](https://raw.githubusercontent.com/TTWabbit/ttwabbit.github.io/master/static/img/_posts/picklerick/pr2.png "Username")

We got a username. Next we are going to use `gobuster` to scan the web site and see if there is some place we can use this username, or get more info.

![Gobuster Results](https://raw.githubusercontent.com/TTWabbit/ttwabbit.github.io/master/static/img/_posts/picklerick/pr3.png "Gobuster Results")

We have some interesting pages in this results. Let's visit them.

![Assets](https://raw.githubusercontent.com/TTWabbit/ttwabbit.github.io/master/static/img/_posts/picklerick/pr4.png "Assets")

Nothing seems to be out of the ordinary here.

![Login](https://raw.githubusercontent.com/TTWabbit/ttwabbit.github.io/master/static/img/_posts/picklerick/pr5.png "Login")

We have a possible username but no password yet.

In the `robots.txt` page we find a word. We try to login with the previously found username and this word as password, access is granted to a Command Panel

![Command Panel](https://raw.githubusercontent.com/TTWabbit/ttwabbit.github.io/master/static/img/_posts/picklerick/pr6.png "Command Pannel")

The links Potions, Creatures, and Beth Clone redirect to a `/denied.php` page where we can see Pickle Rick.

## Exploitation

Let's see if we can execute commands in this panel, first we try `ls`

![ls results](https://raw.githubusercontent.com/TTWabbit/ttwabbit.github.io/master/static/img/_posts/picklerick/pr7.png "ls results")

The first file seems promising, we try to open it with the command `cat`
![cat results](https://raw.githubusercontent.com/TTWabbit/ttwabbit.github.io/master/static/img/_posts/picklerick/pr8.png "cat results")

The command is disabled. We try with the command `less`

![less results](https://raw.githubusercontent.com/TTWabbit/ttwabbit.github.io/master/static/img/_posts/picklerick/pr9.png "less results")

We got our first ingredient.

Then we use `less` to read the content of the `clue.txt` file, and it says, <q>Look around the file system for the other ingredient.</q>

Now, to see the directory we are in the system we use `pwd`

![pwd results](https://raw.githubusercontent.com/TTWabbit/ttwabbit.github.io/master/static/img/_posts/picklerick/pr10.png "pwd results")

Then we list the `root` directory

![ls root](https://raw.githubusercontent.com/TTWabbit/ttwabbit.github.io/master/static/img/_posts/picklerick/pr11.png "ls root")

the `home` directory has two subfolders, we list the one named `rick`, inside there is a file with the name `second ingredients`. We use `less` to view the content

![second ingredients](https://raw.githubusercontent.com/TTWabbit/ttwabbit.github.io/master/static/img/_posts/picklerick/pr12.png "second ingredients")

If we try to list the `/root` directory the command doesn't work. Let's verify the permissions for that folder

![root permissions](https://raw.githubusercontent.com/TTWabbit/ttwabbit.github.io/master/static/img/_posts/picklerick/pr13.png "root permissions")

As expected, we don't have the permissions to access that directory. To check what commands we can issue as ´root´ we can use `sudo -l`

![sudo permissions](https://raw.githubusercontent.com/TTWabbit/ttwabbit.github.io/master/static/img/_posts/picklerick/pr14.png "sudo permission")

We have full access!! Let's see what's in the `/root` directory

![ls /root](https://raw.githubusercontent.com/TTWabbit/ttwabbit.github.io/master/static/img/_posts/picklerick/pr15.png "ls /root")

With `less` we see the content of the `3rd.txt` file

![3rd.txt](https://raw.githubusercontent.com/TTWabbit/ttwabbit.github.io/master/static/img/_posts/picklerick/pr16.png "3rd.txt")

That's all, we have the three ingredients needed to turn Rick back into his human form.

![gif pickle rick](https://thumbs.gfycat.com/CelebratedBreakableEquestrian-size_restricted.gif)