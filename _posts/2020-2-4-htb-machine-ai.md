---
layout:     post
title2:      HacktheBox Writeup - AI - padraignix.github.io
title:     Hack the Box - AI
date:       2020-2-4 17:00:00 -0400
summary:    HTB AI machine walkthrough. Initial portions were more frustrating than complicated, reminiscent of daily struggles dealing with various home assistants. Once foothold was established priviledged escalation to root involved abusing a java debugging process running locally.
categories: [hack-the-box]
thumbnail:  cogs
keywords:   hackthebox,htb,pentest,redteam,writeup,walkthrough,ai,ttv,text to voice,jdwp,java
thumbnail:  https://padraignix.github.io/assets/htb-ai/infocard.PNG
canon:      https://padraignix.github.io/hack-the-box/2020/02/04/htb-machine-ai/
tags:
 - htb 
 - ai
 - text-to-voice
 - jdwp
---

<h1>Introduction</h1>
<p>
<img width="75%" height="75%" src="{{ '/assets/htb-ai/infocard.PNG' | relative_url }}">
</p>

<p>AI was an interesting machine recently retired by HTB. The initial portions were more frustrating than complicated and reminded me of the daily struggle of dealing with various home assistants. Once a foothold was established privesc to root involved abusing a java debugging process running on localhost. As usual this was another HTB machine that gave me an opportunity to work with technology that I had not touched before and I overall enjoyed the journey.</p>

<h1>Initial Recon</h1>

<p>As we always do, let's kick off with an nmap scan of the host to see what we can find.</p>

{% highlight shell %}
root@kali:~/Desktop/HTB/AI# nmap -sV -sV -p- 10.10.10.163
Starting Nmap 7.70 ( https://nmap.org ) at 2019-11-13 21:19 EST
Nmap scan report for 10.10.10.163
Host is up (0.029s latency).
Not shown: 65533 closed ports
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.29 ((Ubuntu))
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
{% endhighlight %}

<p>Alright not all that much. There does seem to be an Apache instance running on port 80, so let's take a look at what is running on there.</p>

<img  src="{{ '/assets/htb-ai/recon-mainpage.png' | relative_url }}">

<img  src="{{ '/assets/htb-ai/recon-aboutpage.png' | relative_url }}">

<p>The most interesting page so far is the <code>db.php</code> page.</p>

<img  src="{{ '/assets/htb-ai/recon-db.png' | relative_url }}">

<p>For a little bit of further recon I also ran a quick scan of the site to see if we could get any extra hits. Sure enough we got some interesting results.</p>

{% highlight shell %}
root@kali:~/Desktop/HTB/AI# dirb http://10.10.10.163 /usr/share/dirb/wordlists/common.txt -X .php

-----------------
DIRB v2.22    
By The Dark Raver
-----------------

START_TIME: Wed Nov 13 21:46:17 2019
URL_BASE: http://10.10.10.163/
WORDLIST_FILES: /usr/share/dirb/wordlists/common.txt
EXTENSIONS_LIST: (.php) | (.php) [NUM = 1]

-----------------

GENERATED WORDS: 4612                                                          

---- Scanning URL: http://10.10.10.163/ ----
+ http://10.10.10.163/about.php (CODE:200|SIZE:37503)                                                                            
+ http://10.10.10.163/contact.php (CODE:200|SIZE:37371)                                                                          
+ http://10.10.10.163/db.php (CODE:200|SIZE:0)                                                                                   
+ http://10.10.10.163/index.php (CODE:200|SIZE:37347)                                                                            
+ http://10.10.10.163/intelligence.php (CODE:200|SIZE:38674)                                                                     
                                                                                                                                 
-----------------
END_TIME: Wed Nov 13 21:48:22 2019
DOWNLOADED: 4612 - FOUND: 5
{% endhighlight %}

<img  src="{{ '/assets/htb-ai/recon-intelligence.png' | relative_url }}">

<p>Very interesting. Looks like it is giving us a general syntax for a voice-to-text usage. Let's try and leverage a local utility, <code>flite</code>. I assumed at this point that we would need to leverage voice-to-text to perform some form of SQL injection. Let's see if we can get any useful output. </p>

<h1>User exploitation</h1>

<p>Funny enough I accidentally stumbled on to an error message based on one of my initial attempts.</p>

<img  src="{{ '/assets/htb-ai/user-flite6.png' | relative_url }}">

<img  src="{{ '/assets/htb-ai/user-flite6-result.png' | relative_url }}">

<p>Ok so we know it is running MySQL under the hood. Unfortunately I was having quite a bit of issues with getting the proper text to be understood properly. It was only once I came across this <a href="https://support.microsoft.com/en-us/help/4042244/windows-10-use-dictation">Microsoft article</a> that I had the proper tips to get the text translated correctly. Funny enough "username" was not being translated properly so I had to take artistic liberties on the spelling and it finally worked.</p>

{% highlight shell %}
root@kali:~/Desktop/HTB/AI# echo "open single quote union select Yusername from users Comment Database" > test6.txt
root@kali:~/Desktop/HTB/AI# flite -voice rms -f test6.txt -o test6.wav
{% endhighlight %}

<img  src="{{ '/assets/htb-ai/user-username.png' | relative_url }}">

<p>Excellent! Alright now all we need is the password. Let's assume there is a password entry in the same user table.</p>

{% highlight shell %}
root@kali:~/Desktop/HTB/AI# echo "Open single quote; union select; password FROM users Comment Database" > test7.txt
root@kali:~/Desktop/HTB/AI# flite -voice rms -f test7.txt -o test7.wav
{% endhighlight %}

<img  src="{{ '/assets/htb-ai/user-password.png' | relative_url }}">

<p>Bam! Alright we should have a proper set of credentials now - <code>alexa / H,Sq9t6}a<)?q93_</code>. Let's try getting on to the host.</p>

<img  src="{{ '/assets/htb-ai/user-pwned.png' | relative_url }}">

<p>And just like that user is owned. On to root!</p>

<h1>Root exploitation</h1>

<p>Once on the box I moved over a few enumeration scripts - namely <code>lse</code> and <code>pspy</code>. After some poking around I found an interesting process running on the host.</p>

<img  src="{{ '/assets/htb-ai/root-pspy.png' | relative_url }}">

<p>Based on the <code>jdpa</code> mention I was able to deduce it was related to <a href="https://docs.oracle.com/javase/8/docs/technotes/guides/jpda/index.html">Java Platform Debugger Architecture</a>. Following the process calls and we can see that it is running on port 8000 and is related to <code>JDWP</code>. A quick search later and we have more details on the process running from the <a href="https://docs.oracle.com/javase/7/docs/technotes/guides/jpda/conninv.html">Oracle JPDA</a> documentation.</p>

<p>
<blockquote>
-agentlib:jdwp=transport=dt_socket, server=y,address=localhost:8000, timeout=5000
<br><br>
Listen for a socket connection on port 8000 on the loopback address only. Terminate if the debugger does not attach within 5 seconds. Suspend this VM before main class loads (suspend=y by default). Once the debugger application connects, it can send a JDWP command to resume the VM.
</blockquote>
</p>

<p>Based on the documentation and the process output the only difference is that our host does not have suspend set <code>suspend=n</code>.</p>

<p>Alright now that we have a better understanding of what is happening how will that help us? A quick look around for any low hanging fruit and we find an existing <a href="https://github.com/IOActive/jdwp-shellifier/blob/master/jdwp-shellifier.py">exploit</a>.</p>

<p>I got the code over to the host and kicked it off. I'm not entirely sure _why_ I decided to go straight for the flag instead of initiating a root shell, however that decision ended up requiring triggering the exploit twice with different payloads due to an entitlements oversight of copying the flag. Hindsight is always 20/20 they say.</p>

<img  src="{{ '/assets/htb-ai/root-own1.png' | relative_url }}">

<p>Well that's annoying... let's change those entitlements.</p>

<img  src="{{ '/assets/htb-ai/root-own2.png' | relative_url }}">

<p>And just like that AI is in the books. Thanks folks, until next time!</p>
