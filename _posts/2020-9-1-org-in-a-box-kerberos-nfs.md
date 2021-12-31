---
layout:     post
title:      Org-in-a-Box - Kerberos & NFS
title2:     Org-in-a-Box - Kerberos & NFS - QuantumlyConfused
date:       2020-9-1 14:00:00 -0400
summary:    Second post covering my personal project of an organization in a box. Based on the initial architecture this article goes over partially setting up the first two authentication servers in the project leveraging MIT Kerberos and NFSv4. These hosts will serve as the core base for the rest of the Org-in-a-Box project and a subsequent post will cover the LDAP and DBIS configuration on these hosts.
categories: [org-in-a-box]
thumbnail:  cogs
keywords:   org-in-a-box,identity management,access management,IAM,Kerberos,MIT Kerberos,replication,nfs,nfsv4,lessonslearned,authentication
thumbnail:  https://blog.quantumlyconfused.com/assets/org-in-a-box/infocard-architecture.png
canon:      https://blog.quantumlyconfused.com/org-in-a-box/2020/09/01/org-in-a-box-kerberos-nfs/
tags:
 - org-in-a-box
 - authentication
 - kerberos
 - nfs 
---

<p>
<img width="70%" height="70%" src="{{ '/assets/org-in-a-box/infocard-architecture.png' | relative_url }}">
</p>

<h1>Introduction</h1>

<p>The second post covering my personal project of an "organization in a box". Based on the <a href="{{ site.baseurl }}{% link _posts/2020-7-23-org-in-a-box-architecture.md %}">initial architecture</a> this article goes over partially setting up the first two authentication servers in the project leveraging MIT Kerberos and NFSv4. These hosts will serve as the base for the rest of the Org-in-a-Box project and a subsequent post will cover the LDAP and DBIS configuration on these hosts.</p>

<h1>Host Prep Work</h1>

<p>Before we start down the Kerberos and NFS path we have a bit of prep work to get done. The very first thing is setting a copy of <a href="https://www.raspberrypi.org/downloads/raspberry-pi-os/">Raspberry Pi OS</a> installed on our Pis following the installation guide provided. Secondly, since I'm running with the headless version to cut back on resource usage we need to make sure to include a file on the SD card, aptly called "ssh", to ensure that SSH is enabled by default. In recent versions of the OS they disabled this service by default and in a headless setup becomes a pain to configure without having to play with HDMI cables.</p>

<p>Since I opted for a proper Pi case with built in fans, we also need to make sure that the pin setup is attached to the board properly.</p>

<p>
<img width="80%" height="80%" src="{{ '/assets/org-in-a-box/kerberos-nfs/pihost-fan-pin.jpg' | relative_url }}">
</p>

<p>Last but not least, once we have logged on the host for the first time we should go ahead and do an initial vanilla update to ensure we apply any available patches or new versions.</p>

{% highlight shell %}
sudo apt-get update && sudo apt-get upgrade
{% endhighlight %}

<p>With that we can move on to the good stuff - getting Kerberos up and running.</p>

<h1>MIT Kerberos</h1>

<p>Most of the installation process is well documented under MIT's <a href="https://web.mit.edu/kerberos/krb5-1.17/doc/admin/install.html">Kerberos 1.17 guide</a>. While many sources of information were used MIT's guide served as the base.</p>

<p>Let us start by getting the relevant 1.17 binaries installed.</p>

<p>
<img src="{{ '/assets/org-in-a-box/kerberos-nfs/kerberos-kadm-binaries.png' | relative_url }}">
</p>

<p>With no cold feet in sight, let's dive on in and create our <code>PADRAIGNIX.COM</code> Kerberos realm.</p>

<p>
<img src="{{ '/assets/org-in-a-box/kerberos-nfs/kerberos-newrealm.png' | relative_url }}">
</p>

<p>Before we start creating new principals we also want to ensure we enable admin entitlements for our <code>/admin</code> accounts. We do this by uncommenting the line in kadm5.acl.</p>

<p>
<img src="{{ '/assets/org-in-a-box/kerberos-nfs/kerberos-kadm-acl.png' | relative_url }}">
</p>

<p>Lastly we need to set up our kdc.conf file which serves as the main configuration for the service. Encryption types, ticket options such as renewable and forwadable, and KDC servers in our environment are all defined in this file. You'll also notice I cheated and put <code>auth02</code> (which will be our replica KDC) as I knew I was going to be building that one shortly as well.</p>

<p>
<img src="{{ '/assets/org-in-a-box/kerberos-nfs/kerberos-kdc-conf.png' | relative_url }}">
</p>

<p>With a quick service restart we confirm everything is up and running and we can start populating the Kerberos database.</p>

{% highlight shell %}
root@auth01:/etc# systemctl status krb5-kdc.service
● krb5-kdc.service - Kerberos 5 Key Distribution Center
   Loaded: loaded (/lib/systemd/system/krb5-kdc.service; enabled; vendor preset: enabled)
   Active: active (running) since Sun 2020-xx-xx xx:xx:xx EDT; xh xxmin ago
 Main PID: 472 (krb5kdc)
    Tasks: 1 (limit: 4915)
   CGroup: /system.slice/krb5-kdc.service
           └─472 /usr/sbin/krb5kdc -P /var/run/krb5-kdc.pid
{% endhighlight %}

<p>Alright, time to add myself, my /admin variant, and the auth01 host principal to our database. I should have configured policies prior to doing so, however I plan on going back later on and creating those to enforce password complexity and various other nuances. Since we don't have any principals at the moment we will use kadmin.local instead of kadmin to interact with the database. The difference between the two is that kadmin.local directly accesses the KDC database, while kadmin performs operations using kadmind - a kerberizes service. We will use kadmin to create the host key once our user principals are created.</p>

<p>
<img src="{{ '/assets/org-in-a-box/kerberos-nfs/kerberos-kadmin-local.png' | relative_url }}">
</p>

<p>
<img src="{{ '/assets/org-in-a-box/kerberos-nfs/kerberos-kadmin.png' | relative_url }}">
</p>

<p>
<img src="{{ '/assets/org-in-a-box/kerberos-nfs/kerberos-hostkey.png' | relative_url }}">
</p>

<p>Excellent we now have our user, admin, and host principals created, so now what? Let's tie this in to the login authentication. The plan is to base our host login on principal/password combinations held within Kerberos, rather than the usual passwd/shadow method. Let's see if we can add a few quality of life items while we are at it as well.</p>

<p>For our initial approach to work we will need to manually add our user to the host and the various groups required. This will not be the long term way of doing so as we will be leveraging LDAP and DBIS in the future. However, to get things started, let us get it done real quick.</p>

<p>
<img src="{{ '/assets/org-in-a-box/kerberos-nfs/kerberos-local-account.png' | relative_url }}">
</p>

<p>Now we have our local user padraignix, and we have a Kerberos principal padraignix, however if we tried to use the Kerberos password to assume the identity (su, ssh, etc.) we would see it fails with incorrect password. In fact, the local user does not even have a password set at the moment. What we will need to do is get an additional set of PAM libaries to help bridge that authentication for us.</p>

{% highlight shell %}
apt-get install libpam-krb5
{% endhighlight %}

<p>
<img src="{{ '/assets/org-in-a-box/kerberos-nfs/kerberos-pam.png' | relative_url }}">
</p>

<p>What we aiming for is to start with Kerberos authentication and if that is successful, permit the login as well as get an initial ticket issued by the KDC. If it is not successful, then we default back to local Unix authentication without any Kerberos tickets. As an added bonus with the PAM configuration we have the ability to create a homedirectory on first login if it does not exist already - so let's select that option as well.</p>


<blockquote>
<h3>Lessons Learned Section</h3>
<p>Unless you like to live dangerously make sure you are also leaving Unix authentication enabled. Without it, should you have any corruption with your Kerberos database or service, or if you're like me and you try to fit a square peg into a round hole and brick your host, you will not have the ability to get back on to fix whatever you broke.</p>

<p>I have an out-of-band (OOB) account that I added to the hosts without a corresponding Kerberos principal. The plan was per the above if anything went wrong, I could log in without going through Kerberos and fix anything needed. However with that PAM option disabled this account became as useless as the host.</p>
</blockquote>

<p>With both Kerberos and Unix options selected the PAM configuration looks like the following - first attempting Kerberos and falling back to local auth if unsuccessful.</p>

<p>
<img src="{{ '/assets/org-in-a-box/kerberos-nfs/kerberos-pam-common-auth.png' | relative_url }}">
</p>

<p>Now all that is left is to log in as padraignix and see if it works.</p>

<p>
<img src="{{ '/assets/org-in-a-box/kerberos-nfs/kerberos-pam-auto-homedir.png' | relative_url }}">
</p>

<p>So far so good. Our password was accepted and the homedirectory was created on initial login. Now how about Kerberos tickets?</p>

<p>
<img src="{{ '/assets/org-in-a-box/kerberos-nfs/kerberos-pam-auto-kinit.png' | relative_url }}">
</p>

<p>Excellent! At this point we earned a quick breather... alright breather done. Time to move on to NFS!</p>

<h1>NFS</h1>

<p>With our Kerberos service running time to see if we can get NFSv4 working and secure it. Since this is generally a test infrastructure I choose a small form factor USB dongle as the eventual NFS share. The plan is to tie it together by not only leveraging Kerberos for the communication, but use the NFS share to create a shared <code>/home</code> space that our hosts will leverage.</p>

<p>The first step was to get more information on our USB stick.</p>

<p>
<img src="{{ '/assets/org-in-a-box/kerberos-nfs/nfs-vanilla.png' | relative_url }}">
</p>

<p>Alright time to get it mounted normally.</p>

<p>
<img src="{{ '/assets/org-in-a-box/kerberos-nfs/nfs-vanilla-mount.png' | relative_url }}">
</p>

<p>Unfortunately the stick was formatted FAT32 by default. To leverage it for our NFS purposes we will need to format it to ext4 so let's get that done.</p>

<p>
<img src="{{ '/assets/org-in-a-box/kerberos-nfs/nfs-ext4.png' | relative_url }}">
</p>

<p>If we add the uuid to <code>/etc/fstab</code> we should be able to access it locally on the mounted host. Before we get fancy with our second host it would be a good idea to confirm everything is working so far post reboot.</p>

<p>
<img src="{{ '/assets/org-in-a-box/kerberos-nfs/nfs-ext4-fstab.png' | relative_url }}">
</p>

<p>
<img src="{{ '/assets/org-in-a-box/kerberos-nfs/nfs-ext4-localhost.png' | relative_url }}">
</p>

<p>Great, time to bump up the difficulty! Starting off, we should ensure that only V4 is running as we want to enforce Kerberized security. Debian has a good <a href="https://wiki.debian.org/NFSServerSetup">NFS Server Guide</a> that helped understand the various options and configurations we would need.</p>

<p>
<img src="{{ '/assets/org-in-a-box/kerberos-nfs/nfs-guide.png' | relative_url }}">
</p>

<p>To which we updated.</p>

<p>
<img src="{{ '/assets/org-in-a-box/kerberos-nfs/nfs-ext4-common.png' | relative_url }}">
</p>

<p>
<img src="{{ '/assets/org-in-a-box/kerberos-nfs/nfs-ext4-kernel-server.png' | relative_url }}">
</p>

<p>To ensure owner and group information is available from client hosts the imapd service Mapping needs to be configured and running on the NFS server.</p>

<p>
<img src="{{ '/assets/org-in-a-box/kerberos-nfs/nfs-ext4-imapd.png' | relative_url }}">
</p>

<p>Lastly on our first host, we need to create a new nfs Kerberos principal and add it to our host keytab.</p>

<p>
<img src="{{ '/assets/org-in-a-box/kerberos-nfs/nfs-principal.png' | relative_url }}">
</p>

<p>
<img src="{{ '/assets/org-in-a-box/kerberos-nfs/nfs-keytab.png' | relative_url }}">
</p>

<p>At this point we are automatically mounting /home from our USB stick locally on <code>auth01</code>. With the configuration work done above all that is left is to serve the share using NFS onthe LAN by configuring <code>/etc/exports</code>.</p>

<p>
<img src="{{ '/assets/org-in-a-box/kerberos-nfs/nfs-ext4-exports.png' | relative_url }}">
</p>

<p>There are three different Kerberos options for serving NFS. Debian has a good <a href="https://wiki.debian.org/NFS/Kerberos">NFS Kerberos</a> page that breaks down those options:</p>

* krb5 Use Kerberos for authentication only.
* krb5i Use Kerberos for authentication, and include a hash with each transaction to ensure integrity. 
* krb5p Use Kerberos for authentication, and encrypt all traffic between the client and server.
<br>

<h2>Second Host</h2>

<p>I will skip over the initial setup as it was covered above. The only difference is that no new Kerberos realms are initialized on the second host, we only make sure the client utilities are available for now.</p>

<p>From <code>auth01</code> we will go ahead and create a new set of principals - a host principal and nfs principal for <code>auth02</code>. We will add them to a new keytab and send that keytab over to the host.</p>

<p>
<img src="{{ '/assets/org-in-a-box/kerberos-nfs/nfs2-keytab.png' | relative_url }}">
</p>

<p>With the keytab in place we should have everything needed to be able to mount our share. Let's try doing that manually.</p>

<p>
<img src="{{ '/assets/org-in-a-box/kerberos-nfs/nfs2-manual-mount.png' | relative_url }}">
</p>

<p>It took entirely too long to realize I was using sec=krb5 instead of sec=krb5p as described in the previous section. Once that hiccup was cleared I was able to get it mounted properly.</p>

{% highlight shell %}
pi@auth02:/$ sudo mount -t nfs4 -o sec=krb5 auth01.padraignix.com:/home /home
mount.nfs4: Connection timed out
pi@auth02:/$ sudo mount -t nfs4 -o sec=krb5p auth01.padraignix.com:/home /home
pi@auth02:/$
{% endhighlight %}

<p>
<img src="{{ '/assets/org-in-a-box/kerberos-nfs/nfs2-bad-imapd.png' | relative_url }}">
</p>

<p>... well almost properly. The share IS mounted and the the location is R/W, however it seems there is something wrong with the ownership and groups. It took me a bit of reading to understand what was going on. Ultimately two things failed me here. First, I needed to have the same user and group information on <code>auth02</code> as I did on <code>auth01</code>. This was simple, yet annoying, to fix and I was able to create my uid=1001 padraignix account locally on <code>auth02</code>. Manually creating accounts on various hosts sounds like it will be a messy, error-prone situation however and we will be solving this requirement further on with LDAP/DBIS. However for the moment, the second issue we had to fix once local accounts were created was the imapd service mapping. If you remember above when I covered the imapd configuration there was a [Mapping] section. Turns out we needed to make sure this was commented out to allow the service to serve the existing owners and groups. A quick modification and refresh on auth01... </p>

<p>
<img src="{{ '/assets/org-in-a-box/kerberos-nfs/nfs2-imapd.png' | relative_url }}">
</p>

<p>and then it looks much better on auth02!</p>

<p>
<img src="{{ '/assets/org-in-a-box/kerberos-nfs/nfs2-good.png' | relative_url }}">
</p>

<p>One more configuration successful. No time stop now however as there is one more aspect we want to address in this post - Kerberos replication.</p>

<h1>Kerberos Replication</h1>

<p>Alright last topic for this post. You would think that setting up replication between two Kerberos instances should be trivial compared to getting NFSv4 setup, however this proved to not be accurate. It was during this portion that my "bricking" situation happened.</p>

<p>The first requirement is getting <code>kpropd</code> installed on both hosts.</p>

{% highlight shell %}
apt-get install krb5-kpropd
{% endhighlight %}

<p>Continuing with the <a href="https://web.mit.edu/kerberos/krb5-1.17/doc/admin/install.html">Kerberos 1.17 guide</a> process we create/add the replica KDC host principal to the Keytabs and subsequently add the primary KDC host principal to the replica's keytab.</p>

<p>
<img src="{{ '/assets/org-in-a-box/kerberos-nfs/replication-keytab1.png' | relative_url }}">
</p>

<p>
<img src="{{ '/assets/org-in-a-box/kerberos-nfs/replication-keytab2.png' | relative_url }}">
</p>

<p>Then we follow up by copying over the following configuration files to <code>auth02</code>.</p>

{% highlight shell %}
krb5.conf
kdc.conf
kadm5.acl
master key stash file
{% endhighlight %}

<p>Having manually started <code>kpropd</code> on the replica KDC I then attempted manually triggering a replication. This is the moment that started the unfortunate bricking situation started.</p>

{% highlight shell %}
root@auth01:/etc/krb5kdc# kdb5_util dump /var/tmp/replica_datatrans
root@auth01:/etc/krb5kdc# ls -l /var/tmp/replica_datatrans
-rw------- 1 root root 5460 Aug 30 12:10 /var/tmp/replica_datatrans
root@auth01:/etc/krb5kdc# kprop -f /var/tmp/replica_datatrans auth02.padraignix.com
kprop: Preauthentication failed while getting initial credentials
{% endhighlight %}

<p>That doesn't seem right... I took a look at the logs to see if I could make sense of it.</p>

<p>
<img src="{{ '/assets/org-in-a-box/kerberos-nfs/replication-fail-log.png' | relative_url }}">
</p>

<p>What I came to understand was that there was a KVNO (Key version number) difference in my host keytabs. Unfortunately I did not figure this out before attempting to reboot <code>auth01</code>, which combined with the removal of the PAM Unix local auth configuration, essentially locked me out of the host. The irony was that the host was able to boot successfully and serve Kerberos requests from <code>auth02</code> which made the decision to blow away 01 and start from scratch all the more painful. Thankfully my notes were detailed enough to go back through the entire process under an hour, so I will chalk it up as a valuable learning experience.</p>

<p>With keytabs sorted out and everything back in working order it is time to re-attempt the manual sync.</p>

{% highlight shell %}
root@auth01:/etc# kprop -f /var/tmp/replica_datatrans auth02.padraignix.com
Database propagation to auth02.padraignix.com: SUCCEEDED
{% endhighlight %}

<p>There we go! A quick test by commenting out the primary KDC from kdc.conf on <code>auth02</code> and trigerring a kdestroy/kinit, confirmed by local logs on 02, was performed and successful. All that is left is to automate the process and make sure everything starts on boot properly.</p>

<p>Step 1 is easy enough. Script up the replication process and add it as a cron job on <code>auth01</code>.</p>

{% highlight shell %}
root@auth01:/etc/krb5kdc# cat secondary-replication.sh
#!/bin/sh

kdb5_util dump /var/tmp/replica_datatrans

kprop -f /var/tmp/replica_datatrans auth02.padraignix.com
{% endhighlight %}

<p>Another quick manual test to confirm it's working.</p>

<p>
<img src="{{ '/assets/org-in-a-box/kerberos-nfs/replication-script.png' | relative_url }}">
</p>

<p>Then adding the proper crontab entry to run this every 10 minutes.</p>

{% highlight shell %}
*/10 * * * * /etc/krb5kdc/secondary-replication.sh
{% endhighlight %}

<p>Unfortunately there was one final complication before we close the book on this portion of the build. For a reason unknown at the time when <code>auth02</code> was rebooted, the kpropd service would start in a failed state. It would try and start up, but end up in a failed stated. It took a bit of digging but I was able to narrow it down.</p> 

{% highlight shell %}
Aug 30 15:34:19 auth02 kpropd[375]: ready
Aug 30 15:34:19 auth02 systemd[1]: Started RPC security service for NFS server.
Aug 30 15:34:19 auth02 systemd[1]: Started RPC security service for NFS client and server.
Aug 30 15:34:19 auth02 systemd[1]: Reached target NFS client services.
Aug 30 15:34:19 auth02 systemd[1]: Reached target Remote File Systems (Pre).
Aug 30 15:34:19 auth02 kpropd[375]: getaddrinfo: Name or service not known
Aug 30 15:34:19 auth02 systemd[1]: krb5-kpropd.service: Main process exited, code=exited, status=1/FAILURE
Aug 30 15:34:19 auth02 systemd[1]: krb5-kpropd.service: Failed with result 'exit-code'.
{% endhighlight %}

<p>What ended up happening was kpropd attempted to start before the network and services were fully ready. I tried a few variations under Unit->After for the systemctl configuration, however defaulting back to tried and true ExecStartPre=sleep ended up working.</p>

<p>
<img src="{{ '/assets/org-in-a-box/kerberos-nfs/replication-systemctl.png' | relative_url }}">
</p>

<p>And if we check systemctl we see that that the service did wait the 10 seconds and that it ultimately started successfully. Problem solved. 

<p>
<img src="{{ '/assets/org-in-a-box/kerberos-nfs/replication-systemctl-start.png' | relative_url }}">
</p>

<h1>Summary & Next Steps</h1>

<p>There we go. Kerberos primary and secondaries, with replication, and NFSv4 leveraging krb5p done and dusted. Next step is to get LDAP and DBIS setup which will remove our manual user creation and propagation requirements.</p> 

<p>As always thanks folks, until next time!</p>