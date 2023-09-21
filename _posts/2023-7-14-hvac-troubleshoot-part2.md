---
layout:     post
title2:     HVAC Troubleshooting - Journey to SME Status
title:      HVAC Troubleshooting - Journey to SME Status
date:       2023-7-14 15:00:00 -0400
summary:    Have you ever cleaned a heavy piece of machinery with a hand-held vacuum? What is a Canadian summer without another HVAC issue! Let's dive into a new error that popped up recently leaving me two weeks without cooling, and what I learned from it! 
categories: [trench-talk]
thumbnail:  microchip
math: false
keywords:   hvac,real-life,investigation,troubleshoot,incident management,problem management,sre,learning
thumbnail:  https://blog.quantumlyconfused.com/assets/misc/hvac-2023/logo.png
canon:      https://blog.quantumlyconfused.com/miscellaneous/2023/07/14/hvac-troubleshoot-part2/
tags:
 - SRE
 - troubleshooting
 - real-life
 - hvac
---

<h1>Introduction</h1>

<p>
<a href="/assets/misc/hvac-2023/logo.png" data-lightbox="image1"><img width="70%" height="70%" src="{{ '/assets//misc/hvac-2023/logo.png' | relative_url }}"></a>
</p>

<p>Have you ever cleaned a piece of heavy machinery with a hand-held vacuum to remove pollen? Not a question I thought I would need to ask myself before this summer. Let's dive into my continued journey to becoming a HVAC subject matter expert (SME), learning way more than I ever expected in cooling technology. As with my <a href="https://blog.quantumlyconfused.com/trench-talk/2022/01/25/real-life-parallels-hvac/">first HVAC troubleshooting post</a> I'll bridge some concepts to cybersecurity and ultimately how some of the lessons learned from the first incident were incorporated to reduce the impact.</p>

<h1>Background Reminder</h1>

<p>As a quick reminder let's overview how my HVAC system works. There is a furnace unit in the basement and a heat pump unit which has a set of coils, one located inside with the furnace unit, while the outdoor coil is housed on a unit on the side of the home. If we look at the system from a diagram perspective, you can see how the heat pump can be "reversed" to provide either heat or cooling inside.</p>

<p>
<a href="/assets/misc/hvac-2023/hvac.png" data-lightbox="image2"><img width="60%" height="60%" src="{{ '/assets/misc/hvac-2023/hvac.png' | relative_url }}"></a>
</p>

<p>A heat pump is an efficient way to manage the heating or cooling in the house, and the furnace is available for cold Canadian winters where the outside falls below the efficient heat pump operating temperature (which varies by model but once we reach -15C that is a perceived threshold).</p>

<p>As a quick recap, the previous year's issues were related to two separate situations</p>

* Restrictive air filters leading to back-pressure and eventual frost build-up created a vicious cycle
* A refrigerant leak in the copper pipes between the external/internal coils that ended up requiring a new line to be installed

<p>The last HVAC issue was fixed in Fall 2021. The next summer period of 2022 went off without any problems. Finally, the dreaded issues were solved and I could enjoy my home without having to worry about internal temperatures reaching dangerous levels! Right...? Right?!</p>

<h1>Investigation & Troubleshooting</h1>

<p>Well as it turns out, this summer ended that streak. Late June 2023, just as the humidity and heat was starting to creep up in the area, the house started feeling warm. Uh-oh... time to break out the trusty meat thermometer (yes, I still haven't gotten a proper one) and found the first troubling data point. The temperature coming out of the vents was the same as outside. While this isn't generally good, the previous issues with back-pressure and low refrigerant levels led to diminished cooling, but cooling all the same. This time, something else is up.</p>


<p>
<center>
<video width="500" height="350" controls>
<source src="/assets/misc/hvac-2023/snapshot.mp4">
Your browser does not support the video tag.
</video>
</center>
</p>

<p>Well that's definitely not a good sound... so with a new set of symptoms means a new troubleshooting round. There has just been a large influx of pollen earlier in the week, worse than any year we experienced previously. As a result I noticed a new potential symptom that while different _could_ maybe explain a different way for the backpressure to have been generated.</p>

<p>
<a href="/assets/misc/hvac-2023/pollen.jpg" data-lightbox="image3"><img width="60%" height="60%" src="{{ '/assets/misc/hvac-2023/pollen.jpg' | relative_url }}"></a>
</p>

<p>
<a href="/assets/misc/hvac-2023/pollen2.jpg" data-lightbox="image4"><img width="60%" height="60%" src="{{ '/assets/misc/hvac-2023/pollen2.jpg' | relative_url }}"></a>
</p>

<p>That is some significant congestion. I tried clearing it out with a shop-vac (just picture someone vacuuming their HVAC unit outside, it was a sight). Sadly, even with a reboot there was no relief. The knocking sound was still happening periodically.</p>

<p>The next step was to check if there was another leak in the copper wiring. Last time a new route was run from the external unit to the furnace, from the outside. Checking that new run, I did notice something I thought was off - residue on the pipes making me think of corrosion.</p>

<p>
<a href="/assets/misc/hvac-2023/copper.jpg" data-lightbox="image5"><img width="60%" height="60%" src="{{ '/assets/misc/hvac-2023/copper.jpg' | relative_url }}"></a>
</p>

<p>However, after some quick searching this was deemed "normal" for outdoor copper piping, especially when it is exposed to the elements, or in this case rain. Ok, one more angle eliminated.</p>

<p>With the more cursory elements eliminated as potential break points, I started diving more into the documentation. This time I focused a bit more on the electrical. While still scary, and please if you end up working with your own systems do be cautious and consult a professional, I felt a bit more confident trying to trace down the potential issue. Starting with what are we working with:</p>

<p>
<a href="/assets/misc/hvac-2023/wiring1.jpg" data-lightbox="image6"><img width="60%" height="60%" src="{{ '/assets/misc/hvac-2023/wiring1.jpg' | relative_url }}"></a>
</p>

<p>After a few iterations of trying to pinpoint exactly where the noise was being generated from (which I can say was not easy!) I was able to deduce the sound seemed to be coming from the compressor. For anyone who has had to replace that part you feel me when I say "ouch". A replacement compressor costs upwards of 40% of replacing the entire system. But all is not yet lost. While I think it may be the compressor, we have to think about the full picture.</p> 

<p>So what is happening when that sound is generated? Based on my reading it seems that as the system is spinning up to cool, the switch is flipped, and the compressor is meant to engage, starting the flow of refrigerant. Based on the lack of cooling and knocking sound I can assume that it is not happening. So could that be a faulty compressor? Sure, but that's not all it could be.</p> 

<p>At this point is where I decided to call in the professionals. I had already pushed boundaries of my knowledge and if we were going into more expensive/complicated parts to replace I did not want to damage anything further. As I'll cover in the next section however, that engagement lead to even more knowledge, and should this exact situation happen again (hopefully not!) I'll be better positioned to fix it myself.</p>

<h1>Resolution - Enter Professionals</h1>

<p>After a few weeks of contingencies in terms of breaking out the standing unit in the living room, the pros were able to come take a look. They did the similar initial triage such as checking refrigerant levels but after showing them the knocking sound they immediately zoned in on the capacitor. According to them it was either the capacitor and/or the compressor. So as an initial test they would replace the capacitor and see if it fixed the situation.</p>

<p>
<a href="/assets/misc/hvac-2023/fix2.jpg" data-lightbox="image7"><img width="60%" height="60%" src="{{ '/assets/misc/hvac-2023/fix2.jpg' | relative_url }}"></a>
</p>

<p>
<a href="/assets/misc/hvac-2023/fix.jpg" data-lightbox="image8"><img width="60%" height="60%" src="{{ '/assets/misc/hvac-2023/fix.jpg' | relative_url }}"></a>
</p>

<p>With a hope and a prayer I turned on the system. After about 10-15 seconds the system engaged, the compressor kicked in, and... it kept running! Victory!</p>

<p>Thankfully, a capacitor was a fraction of the cost of if I would have had to change the compressor. In all, aside from the inconvenience of having to deal with the contingency systems for a few weeks, I was able to learn a bit more about how the system worked, what else to check, and hopefully if something were to happen again in the future, I'm one step further into being able to handle the situation myself. </p>

<p>
<a href="/assets/misc/hvac-2023/wiring.jpg" data-lightbox="image9"><img width="60%" height="60%" src="{{ '/assets/misc/hvac-2023/wiring.jpg' | relative_url }}"></a>
</p>

<p>Here's to slowly, unwillingly, becoming a pseudo-SME for my HVAC system!</p>



<p>Thanks folks, until next time!</p>