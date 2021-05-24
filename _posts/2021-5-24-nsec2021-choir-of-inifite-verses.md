---
layout:     post
title2:      NSEC 2021 - Choir of Infinite Verses
title:     NSEC 2021 - Choir of Infinite Verses
date:       2021-5-24 08:01:00 -0400
summary:    Walkthrough and approach of NSEC2021's the Choir of Infinite Verses challenge. By leveraging an insecure nonce reuse we are able to leverage RC4 Keystream reuse and craft our own modified cookie values.
categories: [ctf]
thumbnail:  cogs
math: true
keywords:   northsec,nsec,nsec2021,ctf,writeup,cryptography,nonce reuse,ctr,RC4,captcha
thumbnail:  https://padraignix.github.io/assets/nsec2021/ctf/choir/infocard.png
canon:      https://padraignix.github.io/ctf/2021/05/24/nsec2021-choir-of-infinite-verses/
tags:
 - nsec2021
 - ctf
 - cryptography
 - captcha
---

<h1>Introduction</h1>
<p>
<img width="65%" height="65%" src="{{ '/assets/nsec2021/ctf/choir/infocard.png' | relative_url }}">
</p>

<p>Choir of "Infinite" Verses was an interesting cryptography challenge part of NSEC 2021's CTF. Once we gathered an initial understanding of how the code work it was a matter of realizing the insecure implementation of RC4, specifically with nonce reuse, then abusing this configuration to craft and encrypt our own malicious cookie.</p>

<p>Challenges like this are nice in that they very quickly highlight the criticality of properly configuring cryptographic implemantions as the smallest mistake can completely break down the system.</p>

<p>Let's jump right in to the challenge details.</p>

<p>
<a href="/assets/nsec2021/ctf/choir/infocard2.png" data-lightbox="image1"><img src="{{ '/assets/nsec2021/ctf/choir/infocard2.png' | relative_url }}"></a>
</p>

<h1>Code Analysis</h1>

<p>Going the main page we are presented with captcha style challenge page. Apparently the monks of North Sectoria believe that if they solve enough captchas (seemingly over <code>10000000000000000</code>!) they can achieve sainthood.</p>

<p>
<a href="/assets/nsec2021/ctf/choir/mainpage.png" data-lightbox="image2"><img src="{{ '/assets/nsec2021/ctf/choir/mainpage.png' | relative_url }}"></a>
</p>

<p>We also notice an interesting cookie value. The first portion is Base64 encoded, delimited by a <code>|</code> and then some form of counter. Unfortunately Base64 decoding the first portion did not provide anything immediately usable so went back to the page and explored the functions.</p>

<p>Sure enough entering a proper captcha increases the counter by 1. The other interesting aspect was that regardless of good or bad submission the second portion of the cookie would increase by 1. Even when refreshing the page (which also generated a new captcha) the counter increases by 1.</p>

<p>A thought crossed my mind that we would need to automate some form of captcha solver to finish this challenge. Thankfully before I started down this route I noticed at the bottom of the page a link to an interesting zip file.</p>

<p>
<a href="/assets/nsec2021/ctf/choir/backup.png" data-lightbox="image3"><img src="{{ '/assets/nsec2021/ctf/choir/backup.png' | relative_url }}"></a>
</p>

<p>Sure enough this ended up being the source code of the app! Excellent let's take a poke at how this works.</p>

<p>
<a href="/assets/nsec2021/ctf/choir/code1.png" data-lightbox="image4"><img src="{{ '/assets/nsec2021/ctf/choir/code1.png' | relative_url }}"></a>
</p>

<p>
<a href="/assets/nsec2021/ctf/choir/code2.png" data-lightbox="image5"><img src="{{ '/assets/nsec2021/ctf/choir/code2.png' | relative_url }}"></a>
</p>

<p>I had to sit down and go over the logic a few times to make sure I understood what was happening here.</p>

<p>Essentially when you load the page it is checking if you have a cookie set. If you don't have one it goes through <code>make_struct</code> to create a cookie and pass it back to you. The code passes three pieces of information to this function:</p>

* <code>letters</code>, which seems to be a set of 10 random letters (captcha anyone?)
* <code>0</code>, which gets zero filled to 24 character length, and finally
* <code>cache['foo']</code>, an internal variable representing the counter

<p>The cookie has a form of:</p>

{% highlight python %}
base64(encrypted(hex(<letters>|<zerofilled-tries>|cache['foo']))|cache['foo']
{% endhighlight %}

<p>Which means our initial cookie would have a value of:</p>

<p>
<a href="/assets/nsec2021/ctf/choir/initialcookie.png" data-lightbox="image6"><img src="{{ '/assets/nsec2021/ctf/choir/initialcookie.png' | relative_url }}"></a>
</p>

{% highlight python %}
base64(encrypted(hex(ettyrqtamc|000000000000000000000000|1))|1
{% endhighlight %}

<p>When a cookie is set the app breaks down the values and attempts to check whether we have acheived sainthood yet or not. If so, it should gives us our flag recognizing our saintly status.</p>

<h1>RC4 Theory</h1>

<p>Now that we have a better understanding of what the app is doing, we need to dive a bit further into the encryption/decryption components. Unfortunately because this encryption component we will not be able to arbitrarily modify the values of the cookie... or so it currently seems...</p>

<p>
<a href="/assets/nsec2021/ctf/choir/crypto.png" data-lightbox="image7"><img src="{{ '/assets/nsec2021/ctf/choir/crypto.png' | relative_url }}"></a>
</p>

<p>Alright we are working with <a href="https://en.wikipedia.org/wiki/RC4">RC4</a> encryption.</p>

<p>Stepping through the configuration we assume (although I never actually tested!) that the key has been changed to something more appropriate for the event, and generates a nonce for the RC4 keystream by taking the current counter, modulus 2048. Oh ho... so essentially a Keystream reuse every 2048 values of the counter. Interesting...</p>

<p>Before we go straight into the solution we need to understand how a keystream reuse is bad news in RC4. In this case we know the plaintext structure and value, and we know the ciphertext.</p>

<p>This <a href="https://crypto.stackexchange.com/questions/45021/rc4-finding-key-if-we-know-plain-text-and-ciphertext">stackexchange post</a> does a great job going over how keystream reuse can be leveraged.</p>

<p>The RC4 keystream is generated by a "key" - \(KS = RC4(k)\)</p>

<p>Where in this case our key is both the encryption key and nonce concatenated together and SHA'd. Once the keystream is generated our ciphertext is created by XOR'ing our original message with the keystream.</p>

<p>\(C = M ⊕ KS\)</p>

<p>Since we already know the full plaintext and the final ciphertext we can XOR ciphertext by our plaintext and get our Keystream for this specific nonce.</p>

<p>\(KS = C ⊕ M\)</p>

<p>We do not have a way to extract the exact key from this setup, but do we need to?</p>

<p>In theory for any given page refresh we can take presented captcha text, our current tries amount, and the ctr and craft our own malicious cookie. Since we have the keystream that would be used at the ctr+2048 attempt we can craft the plaintext to include a value of tries higher than the threshold, pass along the provided captcha letters at that point, and encrypt the result using the previously captured keystream.</p>

<h1>Injecting Crafted Cookie</h1>

<p>By this point we were around the 3577 range of counters. With the plan ready I captured the information required, computed the keystream, and prepped the malicious cookie structure. All that remained was refreshing the page 2048 times to get us back to the same keystream.</p>

<p>
<a href="/assets/nsec2021/ctf/choir/keystream.png" data-lightbox="image8"><img src="{{ '/assets/nsec2021/ctf/choir/keystream.png' | relative_url }}"></a>
</p>

<p>The most stressful part of this process was making sure I didn't accidentally go over 2048 refreshes and have to start the entire process over. Carefully I refreshed until the 5625'th counter and took note of the new letters that were generated - qavlgsmniv. With this I went through the cookie generation using our keystream.</p>

<p>
<a href="/assets/nsec2021/ctf/choir/keystream2.png" data-lightbox="image9"><img src="{{ '/assets/nsec2021/ctf/choir/keystream2.png' | relative_url }}"></a>
</p>

<p>All that is left at this point is to modify our cookie in Firefox and either refresh or submit a captcha.</p>

<p>
<a href="/assets/nsec2021/ctf/choir/flag.png" data-lightbox="image10"><img src="{{ '/assets/nsec2021/ctf/choir/flag.png' | relative_url }}"></a>
</p>

<p>
<a href="/assets/nsec2021/ctf/choir/flag2.png" data-lightbox="image11"><img src="{{ '/assets/nsec2021/ctf/choir/flag2.png' | relative_url }}"></a>
</p>

<p>
<a href="/assets/nsec2021/ctf/choir/flag3.png" data-lightbox="image12"><img src="{{ '/assets/nsec2021/ctf/choir/flag3.png' | relative_url }}"></a>
</p>

<h1>Summary</h1>

<p>This was definitely a fun challenge that illustrates the sensitivity of cryptographic solution implementation. By incorrectly configuring the nonce, and ultimately reusing the same keystream every 2048 iterations we did not even need the encryption key to arbitrarily control cookie values. Maybe next time they can hire use North Sectoria magicians to configure their systems!</p> 

<p>As always thanks folks, until next time!</p>