---
layout:     post
title2:     HVAC and InfoSec - What a Pair!
title:      HVAC and InfoSec - What a Pair!
date:       2022-1-25 09:00:00 -0400
summary:    HVACs, InfoSec, recruitment practices - what do these topics have in common? As we shall see surprisingly alot! Follow along with me as I walk through a troubleshoot of my HVAC system that had me pause and realize just how much crossover between my day-to-day work and everyday life.  
categories: [trench-talk]
thumbnail:  microchip
math: false
keywords:   hvac,real-life,investigation,troubleshoot,incident management,problem management,sre,
thumbnail:  https://blog.quantumlyconfused.com/assets/misc/hvac-2021/logo.png
canon:      https://blog.quantumlyconfused.com/miscellaneous/2022/1/28/real-life-parallels-hvac/
tags:
 - SRE
 - troubleshooting
 - real-life
---

<h1>Introduction</h1>

<p>
<a href="/assets/misc/hvac-2021/logo.png" data-lightbox="image1"><img width="60%" height="60%" src="{{ '/assets//misc/hvac-2021/logo.png' | relative_url }}"></a>
</p>

<p>HVACs, InfoSec, recruitment practices - what do these topics have in common? As we shall see surprisingly a lot! I recently had a chance to take a small vacation recharge mentally and physically. Unfortunately near the end I ran into a slightly uncomfortable situation. In the middle of a heat wave my HVAC system decided to stop working effectively! With temperatures reaching over 40C with humidex it quickly became an "outage" situation and despite my lack of HVAC knowledge my troubleshooting instincts kicked in and I dove right into figuring out what was wrong. What I started to notice was that regardless of the domain, if we approach it with an inquisitive mind our established processes and methodologies can directly translate to helping us organize a logical troubleshoot & resolution.</p>

<h1>System Components</h1>

<blockquote>First thing first - a disclaimer. I am by no means a professional HVAC engineer. I highly suggest you engage a professional to ensure everything is done properly. In the following situation I ended up doing surface level investigations and actions, however if I would have had to go deeper I would have engaged someone who understood the underlying systems better (<code>engaging the vendor SME</code>).</blockquote>

<p>With that out of the way - the next thing we need is a cursory understanding of how HVAC systems work. A high level diagram explaining the main components is below.</p>

<p>
<a href="/assets/misc/hvac-2021/thermo-pump.jpg" data-lightbox="image2"><img src="{{ '/assets//misc/hvac-2021/thermo-pump.jpg' | relative_url }}"></a>
</p>

<p>Where the indoor coil is part of the furnace, and the outdoor coil is the unit outside on the side of the house. The beauty of an HVAC system is that the reversing valve essentially changes the direction of the flow allowing the cooling to be done on the indoor coil (AC in the summer) or have the cooling exhausted outdoor in the winter (heating on the indoor coils). Supplement this with an electric furnace for those cold Canadian winters and we are setup for anything... when it works.</p>

<h1>Incident Start</h1>

<p>Alright so how did this all start? Essentially one more I noticed two things. First, that the temperature in the house was creeping up slightly. Secondly that the air coming out of the bedroom vent was not as strong as it usually felt. Sure enough as I took a look at the thermostat it confirmed my suspicions.</p>

<p>
<a  href="/assets/misc/hvac-2021/temp2.jpg" data-lightbox="image3"><img width="60%" height="60%" src="{{ '/assets/misc/hvac-2021/temp2.jpg' | relative_url }}"></a>
</p>

<p>This only got worst throughout the day as the outside temperature also rose. Clearly the AC was not able to keep up with the overall environment (<code>service degradation</code> in being able to process heat backlog) and I kicked into troubleshooting mode. I may not know the ins and outs of cooling systems but with a methodic approach I should at least be able to deduce something!

<h1>Investigation</h1>

<p>Alright so what first? I ended up doping a visual inspection of the thermopump outside. Unrelated to _this_ situation I had previous issues with the external unit and I figured I would start there (<code>Previous post-mortem lessons learned</code>). Immediately I noticed some freezing on the refrigerant line.</p>

<p>
<a href="/assets/misc/hvac-2021/outdoorcoil1.jpg" data-lightbox="image4"><img width="60%" height="60%" src="{{ '/assets/misc/hvac-2021/outdoorcoil1.jpg' | relative_url }}"></a>
</p>

<p>Considering by this point it was 28C outside there really should not have been ice building up - that's not a good sign. I stopped the entire system, cleared the ice, restarted the cooling, and hoped that this was a one time issue. <b>*Morgan Freeman voice*</b> Surprising no one, it was not. Within 30 seconds of the cooling having started I noticed new frost building up.

<p>
<a href="/assets/misc/hvac-2021/outdoorcoil2.jpg" data-lightbox="image5"><img width="60%" height="60%" src="{{ '/assets/misc/hvac-2021/outdoorcoil2.jpg' | relative_url }}"></a>
</p>

<p>Ok that wasn't it. Back to the drawing board. What else can I check? To rule out whether the cooling was working at all I took the only thermometer available at the moment (trusty old meat thermometer!) and very inaccurately tested the ambient temperature and the air coming out of the vents. This was accurate enough to at least confirm there was a ~23F difference between the two - so clearly the air is at least being cooled.</p>

<p>
<a href="/assets/misc/hvac-2021/livingroom1.jpg" data-lightbox="image6"><img width="60%" height="60%" src="{{ '/assets/misc/hvac-2021/livingroom1.jpg' | relative_url }}"></a>
</p>

<p>
<a href="/assets/misc/hvac-2021/kitchentemp.jpg" data-lightbox="image7"><img width="60%" height="60%" src="{{ '/assets/misc/hvac-2021/kitchentemp.jpg' | relative_url }}"></a>
</p>

<p>So what do we know at this point?</p>

<ul>
    <li>The air is actually being cooled as measured</li>
    <li>There is ice build up on the external unit</li>
    <li>The air flow seems reduced</li>
</ul>

<p>Since the air is being cooled the air flow being reduced could explain why the cooling isn't permeating throughout the house. What can cause that? Hum...the air filter! Time to go check it out.<p>

<p>Even though I was within the recommended 6 month lifespan (<code>expiry</code>) of the filter, it was visibly dirty (<code>degraded</code>). This could definitely lead to clogging the overall air flow. Quickly off to the hardware store I grabbed a replacement and swapped in a fresh filter. Sadly this did not seem to increase the air flow by any measurable difference. At this point I felt I needed to engage a professional. Thankfully I had one more card to play before getting to that point.</p>

<p>One of my good friends has a depth of knowledge in the field (<code>Internal SME</code>). I gave him a call and explained everything I had seen or done up to this point. It was at this point he explained how the fan strengths are generally set for furnace/HVACs. The blower fans have an internal wiring circuit that generally has settings 1 through 5 for strength. A similar 3 speed diagram is below - just imagine there are an additional 2 speeds for my unit.</p>

<p>
<a href="/assets/misc/hvac-2021/wiringdiagram.png" data-lightbox="image8"><img width="60%" height="60%" src="{{ '/assets/misc/hvac-2021/wiringdiagram.png' | relative_url }}"></a>
</p>

<p>While in general you should see something like "low" being 2, and "high" being 4, my friend explained there was a possibility that the wiring was done all to a single speed. This could explain why the overall airflow is lower, however since the reduced airflow was a symptom observed it wouldn't match up to being badly configured initially. Either way it was an easy area to confirm so I took off the furnace panel and checked.</p>

<p>
<a href="/assets/misc/hvac-2021/wiring1.png" data-lightbox="image9"><img width="60%" height="60%" src="{{ '/assets/misc/hvac-2021/wiring1.png' | relative_url }}"></a>
</p>

<p>
<a href="/assets/misc/hvac-2021/fanpins.png" data-lightbox="image10"><img width="60%" height="60%" src="{{ '/assets/misc/hvac-2021/fanpins.png' | relative_url }}"></a>
</p>

<p>Sure enough as you can see they are set to 2 and 4. So nothing odd there. The keen observer might notice something interesting however. We saw some ice build up outside right? Well we also see some ice build up on the coils itself in the furnace section!</p>

<p>
<a href="/assets/misc/hvac-2021/coils3.png" data-lightbox="image11"><img width="60%" height="60%" src="{{ '/assets/misc/hvac-2021/coils3.png' | relative_url }}"></a>
</p>

<p>Wow, ok! That layer of ice would definitely block airflow. As I talked with my friend we established a few more points that lead to the actual <code>root cause</code>.</p>

<ul>
    <li>Increasing fan speed increases the back pressure in the overall system which can lead to freezing.</li>
    <li>Dirty air filters will slow down the flow of air, leading to a similar back pressure build up.</li>
    <li>Closing various vents in the house (in an attempt to "focus" the cooling) will also lead to a pressure build up.</li>
</ul>

<p>With all of these points culminating together, some self-inflicted as attempted workarounds, the system degraded enough to reach the point where condensation was freezing. At this point the system was caught in a degrading loop and would most likely not get better by itself.</p>

<h1>Resolution</h1>

<p>Resolution was simple enough once the root cause was identified. Turn off the system, clear the ice off the coils. I ended up flipping the reversing valve (turned on the heat instead of AC) for a few minutes despite the outdoor temp being north of 30C to clear the remaining ice. Once everything was clear, the air filter was changed, and the closed vents throughout the hour were opened up - I restarted the system. After a few minutes of run time the coils stayed clear!</p>

<p>
<a href="/assets/misc/hvac-2021/coils1.png" data-lightbox="image12"><img width="60%" height="60%" src="{{ '/assets/misc/hvac-2021/coils1.png' | relative_url }}"></a>
</p>

<p>I ended up letting the system run at this point. After another 30 minutes the thermostat registered a half degree drop while the external temp rose by a full degree. Success! Time to let the system do its thing and catch up (<code>drain the backlog</code> of heat built up).

<h1>Summary</h1>

<p>It wasn't until after everything was done did I really appreciate how much of my day-to-day behavior kicked in during this situation. At initial glance there was a problem, and everything in my nature wanted to understand why this problem was happening. I could have easily thrown in the towel right off the bat and called an HVAC specialist, however I would have robbed myself of the chance to learn more about how the system worked.</p>

<p>To finish off the "process" part - I also learned through a root cause analysis and post-mortem to make sure I had proper visibility to the health of the system. I've implemented a monthly check-in (<code>monitoring</code>) to hopefully catch early degradation of the system and prevent this level of impact in the future.</p>

<h2>Candidate Mentality</h2>

<p>Another aspect I wanted to touch on from this situation is the parallels to hiring experiences. As a hiring manager you see all types of candidates - some technically deep, some a little less deep but more broad, some who don't have a technical background necessarily. One of the things I look for when reading over resumes or talking with potential candidates is to understand how they think and approach a problem. A lot of the time I'm asking a question I'm less interested in the answer itself and more interested in how someone approaches it. Did they follow the standard path? Did they come up with an innovative approach? Did they ask clarifying questions or make assumptions? If they did make assumptions, what was the base for those?</p>

<p><blockquote>How we define 'qualified' must change. Sticking to old-school criteria and strict educational requirements will result in missing great team members, and make bridging the resource gap harder than it already is.</blockquote></p>

<p>Thanks folks, until next time!</p>