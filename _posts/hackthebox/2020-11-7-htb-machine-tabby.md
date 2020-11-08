---
layout:     post
title:      HacktheBox Writeup - Tabby - padraignix.github.io
title2:     Hack the Box - Tabby
date:       2020-11-8 15:00:00 -0400
summary:    HTB Tabby machine walkthrough. Tabby starts oof with careful recon enumeration leveraging local file inclusion to harvest credentials then using those credentials to establish a foothold through Apache manager script usage. User escalation then came through a backup zip file encrypted with the user's system password, gathered by using zip2john. Lastly root privesc was achieved by leveraging LXD system container manager as the regular user was part of the lxd control group. Overall this was a fun box to get back into the swing of things after a couple month hiatus and I learned a few tips and tricks.
categories: hack-the-box
thumbnail:  cogs
keywords:   hackthebox,htb,pentest,redteam,infosec,cybersecurity,lxd,apache,lfi
thumbnail:  https://padraignix.github.io/assets/htb-tabby/infocard.PNG
canon:      https://padraignix.github.io/hack-the-box/2020/11/08/htb-machine-tabby/
tags:
 - htb 
 - walkthrough
 - writeup
 - apache
 - lxd
 - lfi
---

<h1>Introduction</h1>
<p align="center">
<img width="75%" height="75%" src="{{ '/assets/htb-tabby/infocard.PNG' | relative_url }}">
</p>

<p>Tabby starts off with careful recon enumeration leveraging local file inclusion to harvest credentials then using those credentials to establish a foothold through Apache manager script usage. User escalation then came through a backup zip file encrypted with the user's system password, gathered by using zip2john. Lastly root privesc was achieved by leveraging LXD system container manager as the regular user was part of the lxd control group. Overall this was a fun box to get back into the swing of things after a couple months of hiatus and I learned a few tips and tricks along the way.</p>

<h1>Initial Recon</h1>

<p>Let's kick it off with an NMAP scan.</p>

{% highlight shell %}
kali@kali:~/Desktop/HTB$ nmap -sV -sC -p- 10.10.10.194
Starting Nmap 7.91 ( https://nmap.org ) at 2020-11-06 15:35 EST                
Nmap scan report for 10.10.10.194                                              
Host is up (0.036s latency).                                                        
Not shown: 65531 closed ports                                                       
PORT     STATE SERVICE  VERSION                                                     
22/tcp   open  ssh      OpenSSH 8.2p1 Ubuntu 4 (Ubuntu Linux; protocol 2.0)              
| ssh-hostkey:                                                                           
|   3072 45:3c:34:14:35:56:23:95:d6:83:4e:26:de:c6:5b:d9 (RSA)                           
|   256 89:79:3a:9c:88:b0:5c:ce:4b:79:b1:02:23:4b:44:a6 (ECDSA)                              
|_  256 1e:e7:b9:55:dd:25:8f:72:56:e8:8e:65:d5:19:b0:8d (ED25519)                            
80/tcp   open  http     Apache httpd 2.4.41 ((Ubuntu))                                       
|_http-server-header: Apache/2.4.41 (Ubuntu)                                                    
|_http-title: Mega Hosting                                                                       
8080/tcp open  http     Apache Tomcat                                                              
|_http-open-proxy: Proxy might be redirecting requests                                             
|_http-title: Apache Tomcat                                                                           
8443/tcp open  ssl/http LXD container manager REST API                                                   
|_http-title: Site doesn't have a title (application/json).                                              
| ssl-cert: Subject: commonName=root@ghost/organizationName=linuxcontainers.org                             
| Subject Alternative Name: DNS:ghost, IP Address:127.0.0.1, IP Address:0:0:0:0:0:0:0:1                        
| Not valid before: 2020-06-16T13:34:28                                                                             
|_Not valid after:  2030-06-14T13:34:28                                                                                  
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel                      
{% endhighlight %}

<p>A couple Apache instances, SSH, and an interesting REST API port for LXD. Let's start off with taking a look at the first Apache instance.</p>

<p align="center">
<img src="{{ '/assets/htb-tabby/recon-mainpage.PNG' | relative_url }}">
</p>

<p>Nothing too exciting on the page until we hit one of the tab redirects...</p>

<p align="center">
<img src="{{ '/assets/htb-tabby/recon-megahost.PNG' | relative_url }}">
</p>

<p>Alright time to add the domain to our hosts file and retry.</p>

{% highlight shell %}
kali@kali:~/Desktop/HTB$ cat /etc/hosts
127.0.0.1       localhost
127.0.1.1       kali
10.10.10.194    megahosting.htb
{% endhighlight %}

<p align="center">
<img src="{{ '/assets/htb-tabby/recon-megahost2.PNG' | relative_url }}">
</p>

<p>Well that's sure an interesting message. It also looks like it pulling a local file using <code>file=</code>. Seems like a prime LFI candidate. Let's see if we can pop out some information.</p>

<p align="center">
<img src="{{ '/assets/htb-tabby/recon-lfi.PNG' | relative_url }}">
</p>

<p>Excellent, good progress so far. We learn that there is a user named <code>ash</code>. Unfortunately without a hint as to where to look there isn't much more progress that can be done here. Let's table the LFI for use later and move on to the other Apache instance under port 8080.</p>

<p align="center">
<img src="{{ '/assets/htb-tabby/recon-apache2.PNG' | relative_url }}">
</p>

<p>Oh ho ho the pieces are starting to come together! By clicking on manager-webapp and host-manager webapp were are presented with Basic auth. With the additional information provided we also know the root structure is under <code>/usr/share/tomcat9</code> and are also given the location of the credential file at <code>/etc/tomcat9/tomcat-users.xml</code>. We should be able to leverage our previous LFI capability to access the credentials! One thing to note here is the final location is <u>not</u> <code>/etc/tomcat9/tomcat-users.xml</code> as that location is relative to the base of the instance, which is under <code>/usr/share/tomcat9</code> giving us an ultimate final location of <code>/usr/share/tomcat9/etc/tomcat9/tomcat-users.xml</code>. Sure enough we see the results come through our BURP listener when triggered.</p>

<p align="center">
<img src="{{ '/assets/htb-tabby/recon-tomcatcreds.PNG' | relative_url }}">
</p>

<p>Other than the obvious user creds the interesting note is just above in the comments - specifically talking about the <code>manager-gui</code> role, which unfortunately our credentials don't possess. This may limit us, but one step at a time, let's log on and see what we can do.</p>

<p align="center">
<img src="{{ '/assets/htb-tabby/recon-apachemanger.PNG' | relative_url }}">
</p>

<p>Sure enough as soon as we start going past the main page we start getting hit with the denial stick. We do have manager-script entitlements however, so we should in theory be able to run all these admin commands using curl calls. Let's take a quick look at the documentation and give that a shot.</p>

<p align="center">
<img src="{{ '/assets/htb-tabby/recon-apachemanger-curl.PNG' | relative_url }}">
</p>

<p align="center">
<img src="{{ '/assets/htb-tabby/recon-apachemanger-curl2.PNG' | relative_url }}">
</p>

<p>Perfect now if we keep reading through the documentation we see there is a possibility to upload our own WAR applications. We can create a malicious file using msvenom, upload it using our manager-script entitlements, and then all things running smoothly be able to trigger it at will. Only one way to find out if it works.</p>

<p align="center">
<img src="{{ '/assets/htb-tabby/recon-apachemanger-deploy.PNG' | relative_url }}">
</p>

<p align="center">
<img src="{{ '/assets/htb-tabby/recon-apachemanger-msvenom.PNG' | relative_url }}">
</p>

<p align="center">
<img src="{{ '/assets/htb-tabby/recon-apachemanger-deploy2.PNG' | relative_url }}">
</p>

<p>Now let's see if it is actually there.</p>

<p align="center">
<img src="{{ '/assets/htb-tabby/recon-apachemanger-curl3.PNG' | relative_url }}">
</p>

<p>All that is left is to trigger it with a local listener on 4444.</p>

<p align="center">
<img src="{{ '/assets/htb-tabby/recon-apachemanger-curl4.PNG' | relative_url }}">
</p>

<p align="center">
<img src="{{ '/assets/htb-tabby/recon-foothold.PNG' | relative_url }}">
</p>

<h1>User exploitation</h1>

<p>Now unfortunately we only have a foothold as the tomcat user and as such do not have access to the user flag (which we now confirm is <code>ash</code> by checking /home/ash). After some poking around we managed to find a backup zip file owned by ash. </p>

<p align="center">
<img src="{{ '/assets/htb-tabby/user-backup.PNG' | relative_url }}">
</p>

<p>Popping that over to our host we see it is password protected. Seems as good an avenue as any to crack.</p>

<p align="center">
<img src="{{ '/assets/htb-tabby/user-backup2.PNG' | relative_url }}">
</p>

<p>We prep the hash file using zip2john and run that through our trusty rockyou list. Sure enough within a few seconds we find the password - <code>admin@it</code>.</p>

<p align="center">
<img src="{{ '/assets/htb-tabby/user-john.PNG' | relative_url }}">
</p>

<p align="center">
<img src="{{ '/assets/htb-tabby/user-john2.PNG' | relative_url }}">
</p>

<p>At this point the assumption was that ash used their own password to encrypt the zip. Let us see if we can pivot over using the new creds.</p>

<p align="center">
<img src="{{ '/assets/htb-tabby/user-owned.PNG' | relative_url }}">
</p>

<p>Sure enough, user down!</p>

<h1>Root exploitation</h1>

<p>Time to bring all the remaining pieces together and finish this machine off. If we take a look at the original recon we remember there was an LXD Rest API running. You'll notice above that our user ash is also part of the lxd group. A quick online search and we find a <a href="https://book.hacktricks.xyz/linux-unix/privilege-escalation/interesting-groups-linux-pe/lxd-privilege-escalation">privesc method using LXD</a> and seems like a probable path. The theory behind this approach is essentially that since we have root access to containers created we can mount the host file system and leverage the container's root access to access any file we want.</p>

<p>Following along with the instructions we get ourselves an Alpine image. We quickly pop that over to our local Apache instance and get it over to the host as ash. I had to go through an <code>lxd init</code> to get the environment on the host ready, however once completed everything worked like a charm.</p>

<p align="center">
<img src="{{ '/assets/htb-tabby/root-lxd.PNG' | relative_url }}">
</p>

<p align="center">
<img src="{{ '/assets/htb-tabby/root-lxd2.PNG' | relative_url }}">
</p>

<p>All that is left is to navigate to our mounted root location (/mnt/root) and proceed to the usual /root/root.txt location.</p>

<p align="center">
<img src="{{ '/assets/htb-tabby/root-owned.PNG' | relative_url }}">
</p>

<p>And with that our journey with tabby comes to an end. After taking a few months off for various things ongoing this was a nice box to shake the rust off and get back into it.</p>

<p>Thanks folks, until next time! </p>