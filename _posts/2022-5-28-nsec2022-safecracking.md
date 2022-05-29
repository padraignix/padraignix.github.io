---
layout:     post
title2:     NSEC 2022 - Safe Cracking
title:      NSEC 2022 - Safe Cracking
date:       2022-05-28 08:00:00 -0400
summary:    Walkthrough of NSEC 2022's Safe Cracking Challenge. Originally planned as a physical track the challenge was adapted to a computer variant. This post will cover the basics of safe cracking theory, how it translates to our provided challenge material, and ultimately the solution.
categories: [ctf]
thumbnail:  microchip
math: false
keywords:   safe cracking,nsec22,ctf,northsec,safe,lockpicking,capture the flag
thumbnail:  https://blog.quantumlyconfused.com/assets/nsec2022/safecracking/logo.png
canon:      https://blog.quantumlyconfused.com/ctf/2022/05/28/nsec2022-safecracking/
tags:
 - safe cracking
 - nsec2022
 - ctf
---

<h1>Introduction</h1>

<p>
<img width="70%" height="70%" src="{{ '/assets/nsec2022/safecracking/logo.png' | relative_url }}">
</p>

Over the last couple years I've started dabbling in locksport. Learning the basics of how a lock works has shown interesting parallels to the cybersecurity world. From identifying weaknesses in the locking mechanism, how people abuse these weaknesses to bypass security, security mechanisms added in to address these weaknesses, and ultimately the continuous loop of cat and mouse; of offensive and defensive security escalations.

Participants at Northsec 2022 CTF this year we were presented with an interesting safe cracking challenge. Talking with the organizers after the CTF they mentioned that this was originally planned as part of the physical track of the CTF, however due to circumstances had to adjust the offering to a digital variant. While I was a bit dejected this wouldn't be a hands-on challenge, since I had separately already started looking into practice safes to start playing with, this was a great opportunity to start learning the theory and understand how to approach this physically in the future.


<h1>Challenge Description</h1>

<p><blockquote>MikhailVolkov:<br>
I saw my office painting slightly crooked than usual. Since my safe is behind the painting, it immediately raised my suspicion.<br>
I found this <code>strange piece of paper</code> on my safe containing highly sensitive documents.<br>
I am not going to open my safe with you guys close by.<br>
Since you claim to be good at security, validate for me if someone had an access to my safe.
<br><br>
Note: flag is in FLAG-\d{2}-\d{2}-\d{2} format
</blockquote></p>

The challenge starts out with a missive - the Ouyaya company member believes someone could have broken into their safe and accessed sensitive documents. There was some cryptic letter left on top the safe and they provided us a copy of it. Let's start this challenge off and take a look at the mysterious note.

<p>
<a href="/assets/nsec2022/safecracking/input.png" data-lightbox="image1"><img src="{{ '/assets/nsec2022/safecracking/input.png' | relative_url }}"></a>
</p>

Before going into the challenge I had not looked worked with safe cracking. Someone with that background should be able to immediately identify what this means, however I had to start at the basics and start looking online with "where do I start?!".

<h1>Safe Cracking Theory</h1>

I quickly found a <a href="https://www.youtube.com/watch?v=f4IPsd9MDGM&list=PL1mdjQBV_-Jv2lf9QZ4chNtW656EOVXzl">video playlist</a> that really helped clarify not only what to look for, but even went into the underlying theory, explaining how safes work mechanically, where the weaknesses are introduced, and ultimately how to extract the right numbers from an analysis.

[![Safe Cracking Theory](https://img.youtube.com/vi/f4IPsd9MDGM/0.jpg)](https://www.youtube.com/watch?v=f4IPsd9MDGM&list=PL1mdjQBV_-Jv2lf9QZ4chNtW656EOVXzl "Safe Cracking Theory")

In addition to the series above there was a <a href="https://www.youtube.com/watch?v=biNog4QctAw">video review</a> of a Sparrow Challenge vault that provided a good view at the internals of how a safe works.

<p>
<a href="/assets/nsec2022/safecracking/mechanics.png" data-lightbox="image2"><img src="{{ '/assets/nsec2022/safecracking/mechanics.png' | relative_url }}"></a>
</p>

With a little bit more digging, I read through the the Sparrow Manual on learning how to crack their practice vault. This page specifically jumped out at me for a few reasons.

<p>
<a href="/assets/nsec2022/safecracking/sparrowgraph.png" data-lightbox="image3"><img width="70%" height="70%" src="{{ '/assets/nsec2022/safecracking/sparrowgraph.png' | relative_url }}"></a>
</p>

It really seems like the provided file are the notes some master safe cracker left behind. We see the 0-100 starting index, the contact points, and the wheel numbers at the top. Looks to me like a lot of the hard work has been done for us already and we just need to understand and interpret the data left behind.

<h1>Mapping Data</h1>

Similar to how we would approach a real safe to crack let's start with wheel number three. We can interpret the columns as the left and right contact points being mapped. To bring it into a similar format to what we viewed in the videos I graphed it out in excel.

<p>
<a href="/assets/nsec2022/safecracking/wheel3-1.png" data-lightbox="image4"><img src="{{ '/assets/nsec2022/safecracking/wheel3-1.png' | relative_url }}"></a>
</p>

Great, that looks pretty solid. The next step is to try and see where the contact points have a convergence from the basis, towards each other. There is one specific point where that seems more clear than anywhere else.

<p>
<a href="/assets/nsec2022/safecracking/wheel3-2.png" data-lightbox="image5"><img src="{{ '/assets/nsec2022/safecracking/wheel3-2.png' | relative_url }}"></a>
</p>

Based off of this since the contact point is 42.5 I believe the third wheel's number is either <code>42 or 43</code>. In reality we could be more granular on the safe itself and go by individual numbers instead of by 2.5, however we have to work with the data provided, so let's hope it ends up being one of the two.

<h1>Victory</h1>

If we continue the approach for the second wheel, we graph out the data and check for the convergence.

<p>
<a href="/assets/nsec2022/safecracking/wheel2.png" data-lightbox="image6"><img src="{{ '/assets/nsec2022/safecracking/wheel2.png' | relative_url }}"></a>
</p>

Excellent, so the second wheel should be <code>70</code>.

For the first wheel, we weren't given any contact point data. We just see an <code>x</code> in the first four columns. If you think about it this makes sense, as we have the first two numbers, this should in theory be a matter of finding the final numbers by process of elimination. With that assumption - our first wheel should somewhere between <code>8, 9, 10</code>.

We don't have an exact number based of this, but the permutations we need to try are reduced to a handful!

<h2>Side Note on Headache</h2>

In reality, I didn't get the numbers 42.5 and 70 initially. The reason? I had forgotten to change the axis label from index to value. So while I had the same graphs, and the same correct "convergence", I was trying to use 18 and 29 instead...

I was certain I had the right numbers, and while trying the various permutations I decidedly tripped one of the monitors on the NSEC admin team. Thankfully they were understanding and since it wasn't an automated bruteforcing left me off with a nice "hey, stop it".

<p>
<a href="/assets/nsec2022/safecracking/brute.png" data-lightbox="image7"><img src="{{ '/assets/nsec2022/safecracking/brute.png' | relative_url }}"></a>
</p>

<p>
<a href="/assets/nsec2022/safecracking/stop.png" data-lightbox="image8"><img width="50%" height="50%" src="{{ '/assets/nsec2022/safecracking/stop.png' | relative_url }}"></a>
</p>

<h2>Submission</h2>

After way too much time, I had an "aha" moment and realized what I had done wrong with the index vs. value blunder while graphing. With that corrected, and the right numbers gathered I was at:

* Wheel 3 - <code>[42,43]</code>
* Wheel 2 - <code>70</code>
* Wheel 1 - <code>[8,9,10]</code>

With a smaller number of permutations I hoped not to invoke the wrath of the NSEC admins and tried them out. Thankfully within a few different tries I was rewarded!

<p>
<a href="/assets/nsec2022/safecracking/victory.png" data-lightbox="image9"><img src="{{ '/assets/nsec2022/safecracking/victory.png' | relative_url }}"></a>
</p>

<h1>Summary</h1>

Considering learning Safe Cracking was something on my "to do" list, I was pleasantly surprised to see this challenge during the NSEC CTF. This offered me a good opportunity to learn the basics, understand how it works, and if nothing else has reinvigorated my interest. Looking closer at the Sparrow practice vault I would say it is quite probable that it will end up in my arsenal in the near future. This would help reinforce what I learned during the CTF, and move it from the theoretical to the practical on how to recognize the contact points.

As always, I am thankful to the NSEC team for not only the event, and quality of challenges offered - that alone is a massive testament to the effort the team put into preparing and running the event, but I am also thankful for their understanding and willingness to reach out to clarify my "bruteforcing" attempts with the incorrect number. Where I initially felt defeated as I was certain it was the right number, I was able to continue, realize the mistake, and correct it successfully. Thank you (and again, sorry!) for making that a possibility.

Thanks folks, until next time!