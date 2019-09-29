---
layout: post
title:  "Exploring Buffer Overflows - A CTF Preparation Case Study"
date:   2019-09-28 17:46:48 -0400
categories: presentation, ctf training
---
<h3>Introduction: Necessity is the Mother of Education</h3>
<p>As part of our Red Team preparations with the then upcoming 2019 Google CTF we held an excercise to take stock of our strengths and weaknesses to know where we would need to improve. It became evident that while we had some Web-based expertise among the group there was a lack in lower level exploitation experience. In the interest of getting folks started and since I was one of the only individuals with prior experience, I was nominated and went through a simplistic, practical BOF example.</p>

<p>Once the preverbial "tag" was issued the first question that came to mind was - "Ok, now what do I cover?". I tried to break it down into specific requirement items to help plan for the presentation.</p>

* <b>Most of the team used Kali</b> - I wanted to make sure they would have a similar platform to retrace the steps themselves.
* <b>Most of the team wanted to work with GDB</b> - There are many Reverse Engineering tools available however the consensus was that folks wanted to work with GDB.
* <b>The purpose was only to explain the theory at a cursory level</b> - All countermeasures were explicitly turned off to allow a beginner discussion.
* <b>I wanted to present the steps in a repeatable fashion</b> - While there would be a presentation and walkthrough, I wanted to allow individuals the ability to go through hands-on themselves after the fact. This meant including the commands and screenshots of the example for reference.

<p>With the plan mapped out I started prepping the deck. Thankfully, there were several resources available that had covered the topic similarly. I was able to cherry-pick between the various sources then add my own explanations and hands-on walkthroughs.</p>

<h3>Presentation</h3>

<embed src="{{ '/assets/pdfs/BufferOverflow-Presentation.pdf' | relative_url }}" type="application/pdf" style="width:100%; height:500px;" frameborder="0" />

<h3>Reception</h3>

<p>Overall it was well received. My measure of success for this presentation was by group engagement and the distinct lack of snores throughout the 1 hour session. By the second criteria it was thankfully a complete success. For group engagement, I had left 5-10 minutes at the end of the presentation for questions and discussions and we ended up going over by more than 20. The questions were a mixture of digging deeper into specifics that were covered as well as what they should focus on to continue learning.

<h3>Next Steps</h3>

<p>As was expected when we first formed the Red Team group most of the planned activities slowed down for the summer. With that said we are looking at various events in the coming months and a continuation of this series may be beneficial. </p>