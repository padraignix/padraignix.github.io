---
layout:     post
title2:      NSEC 2020 - Dreamcast Writeup - padraignix.github.io
title:     NSEC 2020 - Dreamcast
date:       2020-5-19 10:00:00 -0400
summary:    Walkthrough of the two Dreamcast challenges from NorthSec 2020. Starting with a quick overview of the Dreamcast architecture then quickly pivoting into analyzing the provided roms follow along as I cover my approach and solution to the challenges.
categories: [reverse-engineering]
thumbnail:  cogs
keywords:   northsec,nsec2020,ctf,writeup,binary,reverse engineering,dreamcast,re,ghidra,gdb,gef,emulator,rom
thumbnail:  https://padraignix.github.io/assets/nsec2020/infocard-dreamcast.PNG
canon:      https://padraignix.github.io/reverse-engineering/2020/05/19/nsec2020-dreamcast/
tags:
 - nsec2020
 - ctf
 - reverse-engineering
 - ghidra
 - gdb
 - dreamcast
 - emulation
---

<h1>Introduction</h1>
<p>
<img width="85%" height="85%" src="{{ '/assets/nsec2020/infocard-dreamcast.PNG' | relative_url }}">
</p>

<p>As mentioned in my previous <a href="{{ '_posts/reverse-engineering/2020-05-18-nsec2020-crackme.md' | relative_url }}">post</a> I will be covering the two <a href="https://www.nsec.io/">NSEC 2020 CTF</a> Dreamcast challenges, my approaches, and solutions. As a fan of both reverse engineering and emulation in general as soon as I saw these challenges I immediately knew I wanted to tackle them. While I had a bit of background with earlier Chip8 and GameBoy architecture and basic emulation loops I was previously not familiar with the Dreamcast's SH-4 CPU architecture. As I will cover below I was initially confused by some of the decompilation output of the challenge ROMs, however after some further reading and analyzing the machine code directly rather than the incorrectly decompiled output I was able to make sense and finish the first challenge. The second challenge was similarly created and with the previous leg work done on the first challenge it took much less time to complete.</p>

<h1>Dreamcast Background</h1>

<p>The first thing to do was to familiarize myself with the architecture and how it all worked. The challenge provided 4 files, two .cdi files and two .elf files, one for both of the challenges, and a good 500 page document covering the Dreamcast's SH-4 CPU architecture. This included opcodes and generally any and all information one could want regarding the platform CPU from a hardware perspective. The challenge text also _strongly_ suggested using the reicast emulation software to run the provided ROMs.</p>

<p>
<img src="{{ '/assets/nsec2020/nsec2020-dc1-dev.PNG' | relative_url }}">
</p>

<p>Unfortunately one the many complications involved in these challenges was simply getting the emulation working properly.</p>

<h1>Emulation</h1>

<p>Sadly the packaged reicast 20.04 release did not end up working for me.</p>

<p>
<img src="{{ '/assets/nsec2020/nsec2020-dc1-emufail.PNG' | relative_url }}">
</p>

<p>While there were quite a few forum posts and general troubleshooting articles out there none of them seemed to solve my particular issue. Determined to make this run on my Kali instance I opted to git clone the latest branch and compile my own version of the emulator. Thankfully after only a few minor dependency issues that I needed to resolve I was able to get the emulator started.</p>

<p>
<img src="{{ '/assets/nsec2020/nsec2020-dc1-success.PNG' | relative_url }}">
</p>

<p>One last configuration point before we get into actual solutions. Once I was able to get the emulator running I made sure to toggle the dump serial output to stdout. It is debatable whether this helped or not but I like to believe it helped understand a few parts later on.</p>

<p>
<img src="{{ '/assets/nsec2020/nsec2020-dc1-config.PNG' | relative_url }}">
</p>

<h1>Challenge 1</h1>

<p>With everything ready let's get that ROM finally loaded!</p>

<p>
<img src="{{ '/assets/nsec2020/nsec2020-dc1-romboot.gif' | relative_url }}">
</p>

<p>Woot! Now that the pre-challenge is done we can finally start the real challenge!</p>

<p>Alright so right off the bat this seems like a combination lock of sorts. If we enter any random old code to see how it behaves we get the following.</p>

<p>
<img src="{{ '/assets/nsec2020/nsec2020-dc1-intrusion.PNG' | relative_url }}">
</p>

<p>Drat my completely logical "031337" did not work. It seems like we will need to dig a bit deeper.</p>

<p>I found it suspicious that we were provided both .cdi and .elf files for the two challenges. It is at this point that I learned the difference between the two through <a href="https://www.retroreversing.com/sega-dreamcast-game-debug-symbols">this article</a>. It seems that the elf file includes all the debugging symbols whereas the cdi files do not. This will be very important during the next few steps to understand what is going on at the code level. Let us open up ghidra and pop in that elf file to see what we can find.</p>

<p>
<img src="{{ '/assets/nsec2020/nsec2020-dc1-functions.PNG' | relative_url }}">
</p>

<p>Immediately when looking at the function list we see some nice output. Definitely looks like we should be able to piece together the logic. Let's start with main.</p>

<p>
<img src="{{ '/assets/nsec2020/nsec2020-dc1-main.PNG' | relative_url }}">
</p>

<p>Looks like it may possibly be setting up the input digits and entering a graphics loop. Standard emulation workflow, got it. Let's dive into that graphics loop.</p>

<p>
<img src="{{ '/assets/nsec2020/nsec2020-dc1-graphicloop.PNG' | relative_url }}">
</p>

<p>Hum, not sure if this useful or not, but let's go one layer deeper into process_code.</p>

<p>
<img src="{{ '/assets/nsec2020/nsec2020-dc1-processcode.PNG' | relative_url }}">
</p>

<p>Now we're talking. Looks like it is pulling our <code>digit_box_array</code>, arguably the numbers we provide, and seems to compare them to some logic flow. If successful, we trigger <code>_draw_flag</code> however if any of the input comparisons fail we get redirected to <code>_failure</code>.</p>

<p>Before we can start going through the logic check we need to step back and get one more vital piece of information - how the digitbox array stores value in memory. If we understand how that works we will be able to step through the processing logic.</p>

<p>
<img src="{{ '/assets/nsec2020/nsec2020-dc1-selector.PNG' | relative_url }}">
</p>

<p>Hum, while it helps understand our left/right and up/down, no mention to memory itself. Again, let's go a level deeper.</p>

<p>
<img src="{{ '/assets/nsec2020/nsec2020-dc1-increase.PNG' | relative_url }}">
</p>

<p>Excellent! So every each digit takes up 0x14 space. We now know the offset to work with.</p>

<p>Unfortunately at this point is where I lost the majority of my time. Understanding the offset from above I tried to start going through the process check based on the decompilation. If we follow the logic provided we should have something along the lines of.</p>

<blockquote>
0x0  [0] - must be 1<br>
0x14 [1] - must be 8<br>
0x28 [2] - must be 0<br>
0x3c [3] - must be less than 3<br>
0x50 [4] - must not be 7<br>
0x64 [5] - must not be 0<br>
</blockquote>

<p>Try as I might, this combination, forwards, backwards was not working. After beating my head against the keyboard a bit I took a step back and looked at the machine code directly rather than decompilation. This was broken into two code chunks representing that first [0] = 1 then the subsequent logic seen in the decompiled while loop.</p>

<p>
<img src="{{ '/assets/nsec2020/nsec2020-dc1-machine1.PNG' | relative_url }}">
</p>

<p>
<img src="{{ '/assets/nsec2020/nsec2020-dc1-machine2.PNG' | relative_url }}">
</p>

<p>Thanks to the handy SH-4 documentation provided we can actually understand what this all means! The first part to understand is the SH-4 T-bit. This is the register used to store results of comparisons and other operands and will help us determine our conditional branching logic.</p>

<p>
<img src="{{ '/assets/nsec2020/nsec2020-dc1-t.PNG' | relative_url }}">
</p>

<p>Going through the remainder of the code, the following were the relevant opcodes we need to understand.</p>

<p>
<img src="{{ '/assets/nsec2020/nsec2020-dc1-mov.PNG' | relative_url }}">
</p>
<p>
<img src="{{ '/assets/nsec2020/nsec2020-dc1-cmpgt.PNG' | relative_url }}">
</p>
<p>
<img src="{{ '/assets/nsec2020/nsec2020-dc1-cmpeq.PNG' | relative_url }}">
</p>
<p>
<img src="{{ '/assets/nsec2020/nsec2020-dc1-tst.PNG' | relative_url }}">
</p>
<p>
<img src="{{ '/assets/nsec2020/nsec2020-dc1-bt.PNG' | relative_url }}">
</p>
<p>
<img src="{{ '/assets/nsec2020/nsec2020-dc1-bts.PNG' | relative_url }}">
</p>
<p>
<img src="{{ '/assets/nsec2020/nsec2020-dc1-bf.PNG' | relative_url }}">
</p>

<p>Putting it all together we come up with the following logic.</p>

<p>
<img src="{{ '/assets/nsec2020/nsec2020-dc1-newprocess.PNG' | relative_url }}">
</p>

<p>Alright we can see why our previous attempts were not working. Seems the decompilation wasn't exactly accurate and gave us bad advice! With our proper understanding let's give "171270" a shot. That should satisfy all our parameters.</p>

<p>
<img src="{{ '/assets/nsec2020/nsec2020-dc1-flag.gif' | relative_url }}">
</p>

<p>Bam! There we go! Nothing more satisfying than your research and reading coming to fruition! We also get the flag dumped to console as an added bonus.</p>

<p>
<img src="{{ '/assets/nsec2020/nsec2020-dc1-flag.PNG' | relative_url }}">
</p>

<p>Alright five second happy dance over, we still have another challenge to complete!</p>

<h1>Challenge 2</h1>

<p>With everything primed and ready from the first challenge let's boot up the second ROM and see what we can find.</p>

<p>
<img src="{{ '/assets/nsec2020/nsec2020-dc2-romboot.gif' | relative_url }}">
</p>

<p>Ok so very similar to previous challenge, just 5 digits instead of 6. With everything we just learned, let's pop the elf file into ghidra and see what we can find!</p>

<p>
<img src="{{ '/assets/nsec2020/nsec2020-dc2-functions.PNG' | relative_url }}">
</p>

<p>Again, very similar the first challenge. We can skip forward a few steps and take a look at <code>process_code</code> as this will most likely be our interesting function.</p>

<p>
<img src="{{ '/assets/nsec2020/nsec2020-dc2-process.PNG' | relative_url }}">
</p>

<p>Hum...we learned our lesson previously with taking the decompilation at face value. Let's cut to the chase with the machine code.</p>

<p>
<img src="{{ '/assets/nsec2020/nsec2020-dc2-machine1.PNG' | relative_url }}">
</p>

<p>
<img src="{{ '/assets/nsec2020/nsec2020-dc2-machine2.PNG' | relative_url }}">
</p>

<p>To understand the first three digits we need add a few new opcodes from before.</p>

<p>
<img src="{{ '/assets/nsec2020/nsec2020-dc2-xor.PNG' | relative_url }}">
</p>

<p>
<img src="{{ '/assets/nsec2020/nsec2020-dc2-extn.PNG' | relative_url }}">
</p>

<p>If we follow along the first three digits need to be "439".</p>

<p>
<img src="{{ '/assets/nsec2020/nsec2020-dc2-3digits.PNG' | relative_url }}">
</p>

<p>The last two digits are a little more complex to understand as they ultimately interact with each other. At this point we need to introduce our last required opcode.</p>

<p>
<img src="{{ '/assets/nsec2020/nsec2020-dc2-shld.PNG' | relative_url }}">
</p>

<p>Essentially, we either shift left or shift right on Rn based on Rm. This also explains why the decompilation provided an if/else statement with either shift left or right. At least the decompile got something right!</p>

<p>In the context of our situation, the value of [3] is to be shifted by the value of [4] and the result must equal 8. I found it easier to visualize at this point.</p>

<p>
<img src="{{ '/assets/nsec2020/nsec2020-dc2-shld-calc.PNG' | relative_url }}">
</p>

<p>So if [3] = 1 and we shift it by the value of [4] = 3, which since is a positive value ends up performing a left shift, we end up with 1000 in binary, or 8. That brings our final solution to "43913". Let's see if our work pays off!</p>

<p>
<img src="{{ '/assets/nsec2020/nsec2020-dc2-flag.gif' | relative_url }}">
</p>

<p>Excellent! And again similar to the previous challenge our flag is also dumped to stdout.</p>

<p>
<img src="{{ '/assets/nsec2020/nsec2020-dc2-flag.PNG' | relative_url }}">
</p>

<h1>Summary</h1>

<p>What a ride! I learned about the SH-4 architecture, Dreamcast emulation in general, ROM debug symbols, and most importantly to never trust the decompiled code at face value! I really enjoyed every second of these challenges and look forward to dealing with future emulation based efforts!</p>

<p>As always thanks folks, until next time!</p>