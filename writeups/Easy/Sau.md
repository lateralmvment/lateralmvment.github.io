---
title: "Sau"
date: 20-08-26
---
Sau - HackTheBox

OS: Linux
Difficulty: Easy
Pawn Date: 20/08/26

---

Recognition
 nmap
```` nmap -sVC -p- -v IP ````

Ports found: 22,80 (filtered),8338 (filtered),55555

Enumeration
On the previous scan we find that Openssh is running on the defaul port, also the ports
80 and 8338 are open but filtered, but there another open port wich responds to HTTP
request. So we begin our enumeration by browsing the ip with the port 55555.

Initial Access
Touching and trying all the page i discovered that this server is vulnerable to SSRF
(CVE-2023-27163), i tested it with nc to see if it really works, and yep it works so
i decide to connect to the server but now on the port 80 to see whats in there.

Then i saw a Maltrail instance running, if you look the page you will notice that in
there you can see the version of Maltrail that is running (v.053). If you check on google
you will find this version is vulnerable to "unantheticated OS Command Injection". This is
another CVE to finally do the explotation. 

Explotation
After that i just checked on internet the exploit for maltrail, i downloaded the first
one. Readed how to use it, it was pretty simple so i just ran it after puting a port
in listening mode (how the instructions said) and suddently i had a shell as the user 
"puma"

Privilege Escalation
from:puma
to:root
Vector: sudo -l
Method: After checking sudo permissions for the user that we have, we discover that he
can run ```` /usr/bin/systemctl status trail.service ```` as root without a password. Doing another
quick research on gtfobins, we now that systemctl runs with less, so we just need to execute
```` :!/bin/bash ```` or ```` !sh ```` to gain a full root shell

Lessons Learned:
- Always check the version of the software that is running and their CVEs.
- SSRF can be used to pivot to internal services not directly accessible
- Always check GTFOBins when you find a binary in sudo -l
- systemctl status uses less as pager → !sh for shell escape
