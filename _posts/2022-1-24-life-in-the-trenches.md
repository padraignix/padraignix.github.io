---
layout:     post
title2:     Trench Talk - An Infosec Ops Team Series
title:      Trench Talk - An Infosec Ops Team Series
date:       2022-1-24 15:00:00 -0400
summary:    Kick off to a new post series covering "the narrative" of the last decade of being in an Infosec Operations team. This post will introduce the topics to be covered and give an introduction to the overall reason for doing this - sharing my hard earned knowledge and hopefully inspiring future generations that follow similar paths. 
categories: [trench-talk]
thumbnail:  microchip
math: false
keywords:   ops,war stories,operations,identity,iam,identity and access management, devops, sre
thumbnail:  https://blog.quantumlyconfused.com/assets/ops-life/infosec-banner1.png
canon:      https://blog.quantumlyconfused.com/ops-life/2022/01/24/life-in-the-trenches/
tags:
 - operations
 - iam
 - musings
---

<h1>Introduction</h1>

<p>
<img width="70%" height="70%" src="{{ '/assets/ops-life/infosec-banner1.png' | relative_url }}">
</p>

<figure>
<blockquote>But the materials were the least significant aspect of our training. The relevant bits, the ones I would recall two years later, were the war stories...
</blockquote>
<figcaption>&emsp;<i>- Michael Lewis - Liar's Poker</i></figcaption>
</figure>

A new year, a new series. Looking at my previous posts there is a trend of diving, deeply, into technical topics. While 2022 is lining up with several events that will help continue that trend, I wanted to also explore a different angle - discussing some hard learned lessons over the years and sharing some of how I think, or more importantly how I learn. 

With 2022 I'm rounding off my first decade in the industry and I have to say... what a ride so far! With global scale issues like shellshock, more recently log4j, and the overall evolution of my role, team, and responsibilities through situations like a global pandemic, I've had the opportunity to see and adapt to many "new" things over the years. Even as such, I don't consider myself an expert in any specific field. If anything, the last decade has shown me that while I learned a lot, just how much I don't actually know. With that came an important realization that it's less important what I know and how quickly I can learn something. 

<p>
<a href="/assets/ops-life/problemsolving.jpg" data-lightbox="image4"><img width="50%" height="50%" src="{{ '/assets/ops-life/problemsolving.jpg' | relative_url }}"></a>
</p>

I realized that I really enjoy solving problems. Regardless whether it is pre-built academic challenges, or real-life issues, I love the challenge of understanding why something works (or doesn't work) the way it does. I've built up some intuition and general "gut feelings" when approaching certain situations and this curiosity is what I attribute partially to some of success I've had. Some of this is teachable, depending on how you best learn - in terms of concepts, patterns, or thinking approaches. Others, you really need to go through first-hand to appreciate and solidify the knowledge or process. I'm hoping with this series to share a bit of both!

As I'm sure anyone who has read my posts is not surprised, I learn best when I am hands-on. The first tidbit of knowledge on how my learning method works is not only to do it directly, but then turn around an make myself have to explain it terms that almost anyone can understand. This can be explained simply using the [Feynman Technique](https://collegeinfogeek.com/feynman-technique/) (see what I did there?). Partially the reason for starting this blog in the first place, if I am not able to explain something in simple terms, I tell myself I don't know it well enough and I keep the iterative approach. 

<p>
<a href="/assets/ops-life/feynmanntechnique.jpg" data-lightbox="image1"><img width="50%" height="50%" src="{{ '/assets/ops-life/feynmanntechnique.jpg' | relative_url }}"></a>
</p>

By forcing myself to look back at the last decade and condense topics down to an article or two, I'm embracing this even further. While I've been able to do this from a concrete, technical perspective, approaching this with more conceptual topics will present a new set of challenges for me to tackle.

<h1>So what am I covering?</h1>

I want to break down the topics of this series into four core topics - IAM, Operations, Team Evolution, and Up-Skilling. All with an overarching theme of hard lessons lessons-learned "in the trenches" of having to deal with it first-hand. 

<h2>Demystify Identity and Access Management concepts</h2>

Is an IAM team in the realm of Infosec, infrastructure, both? What is the difference between an Identity and an Account? What is the conceptual difference between Data Owner and Data Custodian? As part of an attempt to evangelize a topic I feel can use more clarity I'll cover a few of the pain points I've had to deal with in one way or another over the years.

<h2>Give some insight from the trenches</h2>

<p>
<a href="/assets/ops-life/log4j.png" data-lightbox="image2"><img width="75%" height="75%" src="{{ '/assets/ops-life/log4j.png' | relative_url }}"></a>
</p>

Outages and incidents are part of life as an operations team. One of the first things I had to really learn  was that it's not about if, but when, and being prepared for that situation. Over the last decade I have been part of, and occasionally have caused, outages of all sizes. Outside of the focus of getting back up and running quickly, which can be its own unique challenge, the main takeaway from these situations is how we learn from them - the post-mortem. Analyzing what went well, what could have been better, and ultimately - how do we ensure that we don't end up in a similar situation in the future.

<h2>Talk about my Operations team's evolutions</h2>

The industry has changed a lot in the last several years. This should not be a surprise to many, as the nature of the field itself is dynamic and ever-changing. I wanted to go over some specific trends I've experienced, from a separate Operations and Engineering structure, to a consolidated Identity and Access Management approach, to newly adopted DevOps and SRE methodologies. I'll also talk about what I wish I would have had when starting, from technical training and process documentation, to building a 8 week "mini-capstone" onboarding program and efforts to improve future onboardings experiences.

<h2>Talk about the challenges of up-skilling</h2> 

<p>
<a href="/assets/ops-life/upskill.png" data-lightbox="image3"><img width="50%" height="50%" src="{{ '/assets/ops-life/upskill.png' | relative_url }}"></a>
</p>

One of my favorite topics is training and development. I legitimately enjoy learning a new concept, getting hands-on in something new, and enabling an environment that helps my teams and myself an opportunity to grow. Over the years I've lead and been part of various Training committees, helped run innovation style events, including CTF events with custom built challenges, and helped form and run grassroots efforts on emerging technologies such as Quantum Computing. The commonality to all of this - continuously improving ourselves. 

In these posts I'll cover how I got started, and some of the challenges and what worked to make these efforts a success over the years. Additionally to what worked, I'll cover what I wish I could have done with some of these efforts and where I'd like to see these move towards in the upcoming years.

<h1>Summary</h1>

A lot has happened in the last 10 years. Join me as I go down the rabbit whole of aspects both good and painful that I've ran into during this time. I'll also demonstrate some of the skills I use during my day job also translate to my personal life, and more specifically how this type of "analytical thinking" is something I look for in potential candidates. Technical competences are great, but if someone has an ability to be taught how to fish and then having them turn around and automate an entire fishing process, that is the environment I want to find myself in and am working towards fostering.

Thanks folks, until next time!