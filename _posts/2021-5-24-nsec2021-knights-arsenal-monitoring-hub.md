---
layout:     post
title2:      NSEC 2021 - Knight's Siege Arsenal Monitoring Hub
title:     NSEC 2021 - Knight's Siege Arsenal Monitoring Hub
date:       2021-5-24 08:00:00 -0400
summary:    Walkthrough and approach of the Knight's Siege Arsenal track of NSEC2021. An infrastructure based tracked that focused around ossec, a host intrusion detection system, we were tasked by the Knight Defender of North Sectoria to help test the castle's defences and report any weaknesses. 
categories: [ctf]
thumbnail:  cogs
keywords:   northsec,nsec,nsec2021,ctf,writeup,ossec,rce
thumbnail:  https://blog.quantumlyconfused.com/assets/nsec2021/ctf/knights-arsenal/infocard.PNG
canon:      https://blog.quantumlyconfused.com/ctf/2021/05/24/nsec2021-knights-arsenal-monitoring-hub/
tags:
 - nsec2021
 - ctf
 - ossec
 - rsa
 - rce 
---

<h1>Introduction</h1>
<p>
<img width="50%" height="50%" src="{{ 'assets/nsec2021/ctf/knights-arsenal/infocard.PNG' | relative_url }}">
</p>

<p>The Knight's Siege Arsenal ended up being my favorite track during the CTF event. This particular track followed an infrastructure approach, with multiple nodes, pivots, privilege escalations, all while integrating the overall medieval storyline of North Sectoria. Personally that immersion, not to mention quality of the designed challenge, is what elevated this particular track to favorite status and I am looking forward to further challenges by the designer - Simon Décosse.</p>

<p>Time to dive into the challenge itself!</p>

<p>
<a href="/assets/nsec2021/ctf/knights-arsenal/infocard2.PNG" data-lightbox="image1"><img src="{{ '/assets/nsec2021/ctf/knights-arsenal/infocard2.PNG' | relative_url }}"></a>
</p>

<h1>Flag 1 - Brute</h1>

<p>Right off the bat we are presented with the layout of the infrastructure we will be working with. There are three separate instances: a central management server, and two agent nodes.</p>

<p>
<a href="/assets/nsec2021/ctf/knights-arsenal/mainpage.PNG" data-lightbox="image2"><img src="{{ '/assets/nsec2021/ctf/knights-arsenal/mainpage.PNG' | relative_url }}"></a>
</p>

<p>Additionally we are able to get a copy of the instance configurations. Without knowing more about the Manager/Agent interaction this seems like a good area to start on.</p>

<p>Immediately it becomes evident that we are working with an <a href="https://www.ossec.net/">OSSEC</a> Host Intrusion Detection system. Looking at the central manager configuration we can get a good idea of what to expect.</p>

<p>
<a href="/assets/nsec2021/ctf/knights-arsenal/central-rules.PNG" data-lightbox="image3"><img src="{{ '/assets/nsec2021/ctf/knights-arsenal/central-rules.PNG' | relative_url }}"></a>
</p>

<p>
<a href="/assets/nsec2021/ctf/knights-arsenal/central-rules2.PNG" data-lightbox="image4"><img src="{{ '/assets/nsec2021/ctf/knights-arsenal/central-rules2.PNG' | relative_url }}"></a>
</p>

<p>
<a href="/assets/nsec2021/ctf/knights-arsenal/central-rules3.PNG" data-lightbox="image5"><img src="{{ '/assets/nsec2021/ctf/knights-arsenal/central-rules3.PNG' | relative_url }}"></a>
</p>

<p>Based on the main page and this configuration we can see that <code>eastern-trebuchet.ctf</code> should have a <code>getflag</code> endpoint. Additionally under the ossec active responses it seems that we should be able to execute the write-file.sh action by triggering rule 102. To do so we will need to trigger rule 101, which is requesting the getflag url (rule 101), with a frequency of 3 within a short timeframe.</p>

<p>Alright let's spam ctrl+f5 a few times and see if this works out.</p>

<p>
<a href="/assets/nsec2021/ctf/knights-arsenal/main-flag1.PNG" data-lightbox="image6"><img src="{{ '/assets/nsec2021/ctf/knights-arsenal/main-flag1.PNG' | relative_url }}"></a>
</p>

<p>
<a href="/assets/nsec2021/ctf/knights-arsenal/eastern-flag1.PNG" data-lightbox="image7"><img src="{{ '/assets/nsec2021/ctf/knights-arsenal/eastern-flag1.PNG' | relative_url }}"></a>
</p>

<h1>Flag 2 - RCE</h1>

<p>Looking back the ossec configuration we notice another rule set created for agent 002, or <code>southern-ballista.ctf</code>. This time we see from the central configuration that exec-logger is triggered by rule 201, which looks for invalid ssh users and leverages a syslog subprocess to write them down to users.txt. </p>

<p>
<a href="/assets/nsec2021/ctf/knights-arsenal/south-ssh.PNG" data-lightbox="image12"><img src="{{ '/assets/nsec2021/ctf/knights-arsenal/south-ssh.PNG' | relative_url }}"></a>
</p>

<p>
<a href="/assets/nsec2021/ctf/knights-arsenal/south-ssh2.PNG" data-lightbox="image13"><img src="{{ '/assets/nsec2021/ctf/knights-arsenal/south-ssh2.PNG' | relative_url }}"></a>
</p>

<p>
<a href="/assets/nsec2021/ctf/knights-arsenal/south-ssh3.PNG" data-lightbox="image14"><img src="{{ '/assets/nsec2021/ctf/knights-arsenal/south-ssh3.PNG' | relative_url }}"></a>
</p>

<p>First let's see it in action by SSH'ing over as a user that shouldn't exist. Sure enough this gets captured by the rule set.</p>

<p>
<a href="/assets/nsec2021/ctf/knights-arsenal/ssh-user.PNG" data-lightbox="image8"><img src="{{ '/assets/nsec2021/ctf/knights-arsenal/ssh-user.PNG' | relative_url }}"></a>
</p>

<p>Excellent! Now in theory we should be able to reach the user.txt endpoint since according to logger.py it is storing it at <code>/opt/user.txt</code>. Let's take a look (keep in mind this was a previous screenshot when using kali as the username).</p>

<p>
<a href="/assets/nsec2021/ctf/knights-arsenal/ssh-usertxt.PNG" data-lightbox="image9"><img src="{{ '/assets/nsec2021/ctf/knights-arsenal/ssh-usertxt.PNG' | relative_url }}"></a>
</p>

<p>At this point in theory if we are able to include command execution into the user portion of our SSH request, which gets logged to <code>/var/log/auth.log</code> it should be run by subprocess and return the output to our <code>/opt/user.txt</code>. Let's try that out with a simple ls and see if it works.</p>

<p>
<a href="/assets/nsec2021/ctf/knights-arsenal/ssh-ls.PNG" data-lightbox="image10"><img src="{{ '/assets/nsec2021/ctf/knights-arsenal/ssh-ls.PNG' | relative_url }}"></a>
</p>

<p>
<a href="/assets/nsec2021/ctf/knights-arsenal/ssh-ls2.PNG" data-lightbox="image11"><img src="{{ '/assets/nsec2021/ctf/knights-arsenal/ssh-ls2.PNG' | relative_url }}"></a>
</p>

<p>Great, we should be able to make some progress from here.</p>

<blockquote>
Note - this is where we ended going off rails. Our initial attempts of geting a reverse shell we're not working and we ended finishing this flag and flag 3 leveraging "hardmode" of not having a shell. While not the most efficient nor speediest of solution I rather enjoyed taking this approach and making it work.
</blockquote>

<p>After a bit of poking and experimenting a few facts were gathered.</p>

* Slashes were being stripped in the commands
* Spaces were hit or miss, sometimes working, sometimes being stripped
* Environment variables were leveragable

<p>With the third point in mind we were able to find a nice alternative to using whitespaces by using <code>${IFS}</code> and using <code>${PWD}</code> as an alternative to slashes in our user's environment variables.</p>

<p>
<a href="/assets/nsec2021/ctf/knights-arsenal/user-env.PNG" data-lightbox="image15"><img src="{{ '/assets/nsec2021/ctf/knights-arsenal/user-env.PNG' | relative_url }}"></a>
</p>

<p>Now with more freedom to explore around we found the location of flag.txt, however cat seemed to also be stripped in the command execution. Instead we leveraged head and were able to acquire the second flag.</p>

<p>
<a href="/assets/nsec2021/ctf/knights-arsenal/flag2.PNG" data-lightbox="image16"><img src="{{ '/assets/nsec2021/ctf/knights-arsenal/flag2.PNG' | relative_url }}"></a>
</p>

<h1>Flag 3 - RSA</h1>

<p>In theory by this point we should have had shell access, but as mentioned above we liked inflicting even more pain on ourselves and do this entire next section without remote shell access!</p>

<p>In our sysadmin user's homedirectory we find an interesting <code>notepad</code> folder. By checking the leftover bash history we can see that there is something  happening with this notepad folder worth investigating.</p>

<p>
<a href="/assets/nsec2021/ctf/knights-arsenal/sysadminhome.PNG" data-lightbox="image17"><img src="{{ '/assets/nsec2021/ctf/knights-arsenal/sysadminhome.PNG' | relative_url }}"></a>
</p>

<p>
<a href="/assets/nsec2021/ctf/knights-arsenal/sysadmin-notepad.PNG" data-lightbox="image18"><img src="{{ '/assets/nsec2021/ctf/knights-arsenal/sysadmin-notepad.PNG' | relative_url }}"></a>
</p>

<p>
<a href="/assets/nsec2021/ctf/knights-arsenal/sysadmin-bashhistory.PNG" data-lightbox="image19"><img src="{{ '/assets/nsec2021/ctf/knights-arsenal/sysadmin-bashhistory.PNG' | relative_url }}"></a>
</p>

<p>Alright let's get all of this over to our local machine for further analysis. The encrypt/decrypt files and the public key were easy enough to list out as we had previously done, however the .enc files were binary/hex based and weren't playing nice. In this case we base64 encoded them all and brought them back over to convert back to original format.</p>

<p>
<a href="/assets/nsec2021/ctf/knights-arsenal/sysadmin-prefix1.PNG" data-lightbox="image20"><img src="{{ '/assets/nsec2021/ctf/knights-arsenal/sysadmin-prefix1.PNG' | relative_url }}"></a>
</p>

<p>
<a href="/assets/nsec2021/ctf/knights-arsenal/sysadmin-prefix2.PNG" data-lightbox="image21"><img src="{{ '/assets/nsec2021/ctf/knights-arsenal/sysadmin-prefix2.PNG' | relative_url }}"></a>
</p>

<p>If we combine this with the files and public key we brought over:</p>

<p>
<a href="/assets/nsec2021/ctf/knights-arsenal/sysadmin-files.PNG" data-lightbox="image22"><img src="{{ '/assets/nsec2021/ctf/knights-arsenal/sysadmin-files.PNG' | relative_url }}"></a>
</p>

<p>It starts to become clear we will be factoring our 256 bit RSA public key.</p>

<p>
<a href="/assets/nsec2021/ctf/knights-arsenal/sysadmin-public.PNG" data-lightbox="image23"><img src="{{ '/assets/nsec2021/ctf/knights-arsenal/sysadmin-public.PNG' | relative_url }}"></a>
</p>

<p>A quick search later and I found <a href="https://0day.work/how-i-recovered-your-private-key-or-why-small-keys-are-bad/">this blog</a> that would ultimately give us everything we need to recover the private key. Without spending too much time going over the RSA details we are looking for four factors: p, q, n, e. We have n (modulus) and e (exponent) from the public key and our challenge is to factor n into two prime numbers: p, q. This is the security premise of RSA, that factoring p and q from n should be too computationally complex to be broken "quick enough". However in this case since our key is a small 256 bit this factoring should be feasible.</p>

<p>Getting a copy of yafu installed I put in our modulus and exponent then kicked it off. It took less than 100 seconds to properly factor our private key information.</p>

<p>
<a href="/assets/nsec2021/ctf/knights-arsenal/yafu.PNG" data-lightbox="image23"><img src="{{ '/assets/nsec2021/ctf/knights-arsenal/yafu.PNG' | relative_url }}"></a>
</p>

<p>With the information in place I put together a small script based on the blog post to generate our private key file.</p>

<p>
<a href="/assets/nsec2021/ctf/knights-arsenal/rsa-script.PNG" data-lightbox="image24"><img src="{{ '/assets/nsec2021/ctf/knights-arsenal/rsa-script.PNG' | relative_url }}"></a>
</p>

<p>Let's check to see if our new private key is valid.</p>

<p>
<a href="/assets/nsec2021/ctf/knights-arsenal/rsa-priv.PNG" data-lightbox="image25"><img src="{{ '/assets/nsec2021/ctf/knights-arsenal/rsa-priv.PNG' | relative_url }}"></a>
</p>

<p>Looking good! As an extra validation we also can see our modulus is correct. All that is left is to run the decrypt script using our newly minted private key.</p>

<p>
<a href="/assets/nsec2021/ctf/knights-arsenal/sysadmin-decode.PNG" data-lightbox="image26"><img src="{{ '/assets/nsec2021/ctf/knights-arsenal/sysadmin-decode.PNG' | relative_url }}"></a>
</p>

<p>Drat, not a flag! Oh well this is still a step forward and gives us a hint with where to look next - the SSH keys.</p>

<p>While the private key wasn't directly present on sysadmin's home, there was a nice backup file that was definitely enticing.</p>

<p>Sure enough extracting the contents of the file gives a nice, juicy private key.</p>

<p>
<a href="/assets/nsec2021/ctf/knights-arsenal/sysadmin-backup1.PNG" data-lightbox="image27"><img src="{{ '/assets/nsec2021/ctf/knights-arsenal/sysadmin-backup1.PNG' | relative_url }}"></a>
</p>

<p>
<a href="/assets/nsec2021/ctf/knights-arsenal/sysadmin-backup2.PNG" data-lightbox="image27"><img src="{{ '/assets/nsec2021/ctf/knights-arsenal/sysadmin-backup2.PNG' | relative_url }}"></a>
</p>

<p>All that was left was to use it and SSH over to <code>central-dovecot.ctf</code> and claim our flag!</p>

<p>
<a href="/assets/nsec2021/ctf/knights-arsenal/flag3.PNG" data-lightbox="image28"><img src="{{ '/assets/nsec2021/ctf/knights-arsenal/flag3.PNG' | relative_url }}"></a>
</p>

<p>As an extra perk this also means we have a shell connection... no more user.txt abuse here!</p>

<h1>Flag Bonus - Cred Spray</h1>

<p>This one was definitely a gimme, but could easily have been overlooked. With our SSH credentials from the previous flag, if we took a step back and logged on to <code>eastern-trebuchet.ctf</code> there was an additional flag.txt right in the home directory for us.</p>

<p>
<a href="/assets/nsec2021/ctf/knights-arsenal/flagbonus.PNG" data-lightbox="image29"><img src="{{ '/assets/nsec2021/ctf/knights-arsenal/flagbonus.PNG' | relative_url }}"></a>
</p>

<h1>Flag 5 - PHP</h1>

<p>Ah the final flag... this one ended up being a pain and really shouldn't have been. Full disclosure I was only able to finish it after the submissions closed however I still wanted to finish it up out of principal at this point.</p>

<p>Initially I believed the escalation path would be related to <a href="https://github.com/ossec/ossec-hids/releases/tag/2.8.1">CVE-2014-5284</a> and I ended up losing a lot of time seeing how I could make this work.</p>

<p>What I should have realized quickly while running through the usual priv-escalation enumeration scripts was a world-writeable PHP file in <code>/var/www/html/download</code>. Owned by www-data this would allow us a lateral pivot on the host. Why does this matter? As sysadmin we were not able to access the main ossec folders like <code>/var/ossec</code> however on further investigation our good old www-data user was part of the ossec group. Hum... that should have been more evident to me.</p>

<p>By modifying index.php we can arbitrarily run anything as www-data. Apparently I have something against reverse shells as I went with a direct flag capture approach - by reading the file directly.</p>

<p>
<a href="/assets/nsec2021/ctf/knights-arsenal/flag4-1.PNG" data-lightbox="image30"><img src="{{ '/assets/nsec2021/ctf/knights-arsenal/flag4-1.PNG' | relative_url }}"></a>
</p>

<p>
<a href="/assets/nsec2021/ctf/knights-arsenal/flag4-2.PNG" data-lightbox="image30"><img src="{{ '/assets/nsec2021/ctf/knights-arsenal/flag4-2.PNG' | relative_url }}"></a>
</p>

<p>And thus brings to an end the track!</p>

<h1>Summary</h1>

<p>As I mentioned at the start this was by far my favorite track during this year's NSEC CTF. It is possible that the infrastructure based scenario tugged at my infra background, and I really hope to see similar style challenges in future events.</p>

<p>I want to thank Simon Décosse for an excellent challenge, and the entire NSEC team for making this event a huge success. I'm already looking forward to next year's edition!</p>

<p>As always thanks folks, until next time!</p>