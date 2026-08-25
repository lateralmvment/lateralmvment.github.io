---
layout: default
---

I been learning about computers and all of that from 
COMPLETELY ZERO, before that i had no idea what was a kernel or how the 
network really works. I started investigating by myself and from now i 
think theres no way back, so im here to show you all my process.

Starting Point
Ok this are the things that i know in this EXACT moment (4/08/26)

Basic Networking: When i started i didnt knew NOTHING 
about networking so after a few weeks of seeing videos, free courses and
 the help of AI and my notes. Now i know almost all of the basics of 
networking, TCP, UDP, HTTP, HTTPS, DNS, CG-NAT, DDNS, IPv4, IPv6, and 
yeah u will see on my notes hahaha.
Good level of Linux: I have a funny history with linux, i 
literally switched to linux IN THE EXACT DAY that i was learning about 
computers. The video that i was seeing said that if you wanna understand
 how computers work switch to linux, and then i just switched to debian,
 at the end im very curious and i guess that helped me to get a "good" 
level of linux.
bandit (OverTheWire):
I passed ALL the goddamn levels, the guys who recommend this to an 
absolutely new guy
ARE EVIL, ik that is literally made for newies but DAMN it was getting 
harder and harder on each level. But i made it and im so happy about it,
 now im even proud of
myself for stay on the levels and actually learn. But yeah idk if im 
dumb or why the
advanced levels are so hard.
HTB (starting point):
This is another good thing that im very happy about it, i passed all the
 starting point
levels, and i have alot of things to say, but im goona say just some 
things. It was such a nice experience, kinda challenging, i learned alot
 in that few boxes, because at the end i think HTB is a really great 
tool to learn about cybersecurity, how to use the most famous tools, In 
this levels i realized that if you think about it. The levels are very 
very easy, the boxes barely have security, are misconfigurated but hey 
its nice to learn what you can do in each situation. Its literally like a
 puzzle without alot of help (unless you see the walktroughs) and thats 
great because if its complicated and you stay there studying you will 
get better than just seeing a Youtube video or a walktrough, at least 
for me i felt a big difference before starting with hack the box than 
after.

Next tiny goals:

PortSwigger Web Academy: I wanna start getting better on 
web hacking because i feel so attracted to it, so im gonna use this 
resource to improve my skills.
This website: I wanna fill of content this blog because i 
wanna remember the things that i did and even if theres a curious guy 
out there they can see my things too, thats a win win.

final goals:

-Do bugbounties
-Become a pentester
-Being a red team member
-Having an IT/cybersecurity related job


Writeups


## Easy

<ul>
{% assign sorted_writeups = site.writeups | sort: "path" %}
{% for post in sorted_writeups %}
  {% if post.path contains 'writeups/Easy/' %}
    <li><a href="{{  https://lateralmvment.github.io/}}{{  https://lateralmvment.github.io/}}">{{ Busqueda }}</a></li>
  {% endif %}
{% endfor %}
</ul>

## Medium

<ul>
{% for post in sorted_writeups %}
  {% if post.path contains 'writeups/Medium/' %}
    <li><a href="{{ site.baseurl }}{{ post.url }}">{{ post.title }}</a></li>
  {% endif %}
{% endfor %}
</ul>
