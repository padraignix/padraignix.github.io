---
layout:     post
title2:     NSEC 2023 - N64 - Document Dropper 2 Reverse Engineering
title:      NSEC 2023 - N64 - Document Dropper 2 Reverse Engineering
date:       2023-5-22 14:00:00 -0400
summary:    Walkthrough of NSEC 2023's two N64 ROM challenges. This post will cover the ROM analysis, including loading into Ghidra, extracting debug symbols from a separately provided file, and then stepping through the logic loop to determine the right input. Ultimately the challenge was solved using a memory buffer overflow arbitrarily calling the printflag() function.
categories: [ctf]
thumbnail:  microchip
math: false
keywords:   northsec,nsec,nsec2023,ctf,writeup,binary,reverse engineering,n64,re,ghidra,rom,symbols
thumbnail:  https://blog.quantumlyconfused.com/assets/nsec2023/docdropper/logo.png
canon:      https://blog.quantumlyconfused.com/ctf/2023/05/22/nsec2023-gamestation/
tags:
 - nsec2023
 - ctf
 - reverse-engineering
 - ghidra
 - n64 
---

<h1>Introduction</h1>

<p>
<img width="80%" height="80%" src="{{ '/assets/nsec2023/docdropper/logo.png' | relative_url }}">
</p>

One of my favorite times of the year - NSEC's annual CTF. I have participated in various formats over the years, from hybrid, to virtual during the pandemic, back to a mostly in person event, to this year's "in-person first", and while no year has disappointed, it was really something to see close to 700 participants alongside me competing over the weekend. 

Similar to previous years, there was a gaming track that focused on reverse engineering. The author of this specific challenge, <a href="https://www.linkedin.com/in/zack-deveau-18328b64/">Zack Deveau</a>, also authored last year's N64-based challenge I loved so much, and I can't thank him enough for continuing the tradition. Not only did this year's track offer another opportunity to dive into the low-level architecture of N64 architecture, but this year built up on the previous edition and where we still had to understand the flow and spend time understanding what was happening logically, as you will shortly see, this year's version expanded into exploiting that understanding and creating a buffer overflow to take over arbitrary execution. Last, but certainly not least, since this was in-person-first, while we could work on the general proof-of-concept using emulation, the final exploit had to be done in-person, using a physical "Game Station" (N64) with the controller I grew up holding in my hands. I won't lie and say it brought back some nostalgia while executing.

Happy to say I also stepped up my own abilities this year - not only was I was successful in the Document Dropper 2 exploitation, but I was quick enough to snag first blood on both challenges!

Now, let's dive into the details.

<h1>Initial Setup & Analysis</h1>

<p>
<a href="/assets/nsec2023/docdropper/intro.png" data-lightbox="image1"><img src="{{ '/assets/nsec2023/docdropper/intro.png' | relative_url }}"></a>
</p>

Having gone through some initial setup in last year's NSEC CTF (<a href="https://blog.quantumlyconfused.com/ctf/2022/05/22/nsec2022-n64/">NSEC 2022 - N64 RE</a>) I had a few tools already available to start diving into the challenge. Before we go too far however, I booted up Doc Dropper 2 and took a look at what we would be doing.

<p>
<a href="/assets/nsec2023/docdropper/p64-reg.gif" data-lightbox="image2"><img src="{{ '/assets/nsec2023/docdropper/p64-reg.gif' | relative_url }}"></a>
</p>

Ok, simple enough; directional arrows, Z to drop off a doc, and then R basically submits our previous entries. The only extra thing I did at this point was to see how many documents we could drop off; it would not increment past 64.

My first thought was that we would need to find the right combination, similar to 2022, that would print out our flag. With that presumption I moved over to Ghidra analysis.

<h1>Memory Debugging</h1>

There were two parts I did to make my life easier here. Both were previously covered in the 2022 writeup, however I'll summarize them here:

* <a href="https://github.com/zeroKilo/N64LoaderWV">N64LoaderWV</a> - Allows for better quality of life when analyzing N64 roms in Ghidra. Fun fact, I actually had to drop down to Ghidra 10.2.3 to use the plugin since it hasn't been updated to 10.3 yet.

* <code>ImportSymbolsScript.py</code> - Ghidra script allowing us to import the debug symbol files (the *.out files) that were also provided. Required leveraging <code>readelf</code> and reversing the data in the line, but nothing crazy and we now have symbols available.

With the prep work done, loaded the first rom and took a look at what we had.

The first thing I found was the <code>updateGame</code> functions. The first was relatively boring, transitioning from the main menu to the actual game. The second, at least initially, controlled the input mapping and logic. That seems a bit more interesting.

<p>
<a href="/assets/nsec2023/docdropper/ghidra1.png" data-lightbox="image3"><img src="{{ '/assets/nsec2023/docdropper/ghidra1.png' | relative_url }}"></a>
</p>

Let's keep going. What about that <code>prepareHistory</code> function?

<p>
<a href="/assets/nsec2023/docdropper/ghidra2.png" data-lightbox="image4"><img src="{{ '/assets/nsec2023/docdropper/ghidra2.png' | relative_url }}"></a>
</p>

Did you see it yet? If not, no worries we will get back to this later. Lastly, let's take a look at the <code>printHistory</code>.

<p>
<a href="/assets/nsec2023/docdropper/ghidra3.png" data-lightbox="image5"><img src="{{ '/assets/nsec2023/docdropper/ghidra3.png' | relative_url }}"></a>
</p>

Hovering over the memory we can see that this is what is actually printing our document location history (the symbol line showing what is actually printed). Ok, great, we have a bit better understanding how the game works, but there is one last thing that I found, and this was VERY interesting.

<p>
<a href="/assets/nsec2023/docdropper/ghidra4.png" data-lightbox="image6"><img src="{{ '/assets/nsec2023/docdropper/ghidra4.png' | relative_url }}"></a>
</p>

Oh, ho ho... what do we have here?! A <code>printFlag</code> function, that isn't referenced anywhere in code. The text was ominous, but also really cool, since it wasn't initially mentioned (or if it was, I missed it), that we would need to execute the exploit in person. This just made me want to complete it all the more!

We should have enough to get started, let's get cracking on a proof of concept.

<h1>Challenge 1 POC</h1>

Enter Project64 debug mode! While poking around I was keeping an eye on memory. The commands pane was less useful here in challenge 1, however made an appearance for the next one.

First thing I wanted to do was see what happens, both at runtime and in memory, when I fill up the document buffer.

<p>
<a href="/assets/nsec2023/docdropper/p64-debug1.gif" data-lightbox="image7"><img src="{{ '/assets/nsec2023/docdropper/p64-debug1.gif' | relative_url }}"></a>
</p>

Now that's what I'm talking about! We have the ability to control execution path! For anyone confused as to why this happens? Let's go back to our <code>prepareHistory</code> function, this time with a few highlights.

<p>
<a href="/assets/nsec2023/docdropper/ghidra2-anotate.png" data-lightbox="image8"><img src="{{ '/assets/nsec2023/docdropper/ghidra2-anotate.png' | relative_url }}"></a>
</p>

So remember that our index goes up to 64 right? Well the prepareHistory function iterates through our entire document history (up to the 64 byte index) and tries to cram it into <code>auStack_18</code>, which is only 24 bytes long.

<h1>Challenge 1 - Completion</h1>

With an approach, time to see if overwriting the memory directly to the <code>printFlag</code> function works. While I initially did do it by hand, I was very happy that Project64 allows for direct modification of memory during runtime. Copy-pasting that memory test saved a lot of time.

<p>
<a href="/assets/nsec2023/docdropper/p64-poc1.gif" data-lightbox="image9"><img src="{{ '/assets/nsec2023/docdropper/p64-poc1.gif' | relative_url }}"></a>
</p>

Bam! There we go! 

Alright, alright, now time to cash in hopefully and execute the payload physically. I have to admit, having to wait an hour to be able to do it added to anxiety.

<p>
<a width="60%" height="60%" href="/assets/nsec2023/docdropper/discord.png" data-lightbox="image10"><img src="{{ '/assets/nsec2023/docdropper/discord.png' | relative_url }}"></a>
</p>

With the Game Station prepped, I had to PAINSTAKINGLY enter the code address by address. In all I think it took close to 10 mins to enter it. However, when I pressed print I was relieved to see part of a flag!

<p>
<a href="/assets/nsec2023/docdropper/victory1.png" data-lightbox="image11"><img src="{{ '/assets/nsec2023/docdropper/victory1.png' | relative_url }}"></a>
</p>

Fistbumps all around with a few of the NSEC mods for the first successful attempt of the CTF and it was quick to go back and work on the second part!

<h1>Challenge 2 - Analysis</h1>

Booting up the second rom, I was initially a little confused - it looked like the first. 

<p>
<a href="/assets/nsec2023/docdropper/p64-memcorrupt.png" data-lightbox="image12"><img src="{{ '/assets/nsec2023/docdropper/p64-memcorrupt.png' | relative_url }}"></a>
</p>

Analyzing through Ghidra I was also seeing the same functions. Until that is, I got to the <code>printFlag</code> function.

<p>
<a href="/assets/nsec2023/docdropper/ghidra5.png" data-lightbox="image13"><img src="{{ '/assets/nsec2023/docdropper/ghidra5.png' | relative_url }}"></a>
</p>

Aha, there are a few logical checks. We can't just spray and pray like the first challenge and we will need to be a bit more tactical. Off to the debugger!

<h1>Project64 Debugging</h1>

In this case I ended up using the memory addresses for the comparisons I got from Ghidra, and adding breakpoints in Project64's Commands pane.

<p>
<a href="/assets/nsec2023/docdropper/p64-debugger.png" data-lightbox="image14"><img src="{{ '/assets/nsec2023/docdropper/p64-debugger.png' | relative_url }}"></a>
</p>

This allowed me to step through the execution using the exploit from the first challenge and figuring out exactly which memory address corresponded to each compare in <code>printFlag</code>, adjusting the exploit code as needed. I did a best guess first pass of the comparisons and I initially messed up <code>local_f</code> however this provided a good example of how to step through the debugger to compare using the breakpoints previously set. As I stepped through and identified the proper memory address I also added a symbol within Project64.

First compare - success

<p>
<a href="/assets/nsec2023/docdropper/mem1.png" data-lightbox="image15"><img src="{{ '/assets/nsec2023/docdropper/mem1.png' | relative_url }}"></a>
</p>

Second compare - failure

<p>
<a href="/assets/nsec2023/docdropper/mem2.png" data-lightbox="image16"><img src="{{ '/assets/nsec2023/docdropper/mem2.png' | relative_url }}"></a>
</p>

Third compare - success

<p>
<a href="/assets/nsec2023/docdropper/mem3.png" data-lightbox="image17"><img src="{{ '/assets/nsec2023/docdropper/mem3.png' | relative_url }}"></a>
</p>

Fourth compare - success

<p>
<a href="/assets/nsec2023/docdropper/mem4.png" data-lightbox="image18"><img src="{{ '/assets/nsec2023/docdropper/mem4.png' | relative_url }}"></a>
</p>

Fifth compare - success

<p>
<a href="/assets/nsec2023/docdropper/mem5.png" data-lightbox="image19"><img src="{{ '/assets/nsec2023/docdropper/mem5.png' | relative_url }}"></a>
</p>

Sixth compare - success

<p>
<a href="/assets/nsec2023/docdropper/mem6.png" data-lightbox="image20"><img src="{{ '/assets/nsec2023/docdropper/mem6.png' | relative_url }}"></a>
</p>

Seventh compare - success

<p>
<a href="/assets/nsec2023/docdropper/mem7.png" data-lightbox="image21"><img src="{{ '/assets/nsec2023/docdropper/mem7.png' | relative_url }}"></a>
</p>

The overall fail compare since the failure register had been incremented from the second compare.

<p>
<a href="/assets/nsec2023/docdropper/mem-fail.png" data-lightbox="image22"><img src="{{ '/assets/nsec2023/docdropper/mem-fail.png' | relative_url }}"></a>
</p>

<h1>Challenge 2 - POC</h1>

Adjusting for that one failure, the updated exploit code from the first challenge then became:

<code>
58F519AB 200179EC 790279EC 800279EC 800279EC 800279EC
800279EC 800279EC 800279EC 800279EC 800279EC 800279EC
800279EC 800279EC 800279EC 800279EC 40
</code>

All that was left was to confirm it worked properly on the emulator.

<p>
<a href="/assets/nsec2023/docdropper/p64-poc2.gif" data-lightbox="image23"><img src="{{ '/assets/nsec2023/docdropper/p64-poc2.gif' | relative_url }}"></a>
</p>

Bam! Alright, one more round at the Game Station to do it in person annnnnnndddddd... I typo'd an entry somewhere and it didn't work. After a painstaking extra 10 mins or so to enter it a second time... thankfully the second submission was good to finish off first blood on the track!

<p>
<a href="/assets/nsec2023/docdropper/victory2.png" data-lightbox="image24"><img src="{{ '/assets/nsec2023/docdropper/victory2.png' | relative_url }}"></a>
</p>

Phew... what a journey. 

<h1>Wrap-Up Conclusion</h1>

As with previous years I thoroughly enjoyed the "Game Station" track. What really added to the overall experience was that this year was not just about "breaking the code" as it was with 2022. This year's edition went a step further with executing a buffer overflow in the game code, calling the unreferenced function. 

Then, having to do the actual exploit in-person, physically, was magical. The stress levels of putting in the addresses using an N64 controller (which I hadn't mentioned but was inverted to the emulator, since you can remap an emulator...) was only exceeded by the level of excitement and sheer "f' yeah!" I felt having completed it sitting next to Zack & Phil - two legends who's challenges I have been attempting (both successfully and...otherwise) over the years. The experience is something I will never forget. Here's to prepping for NSEC 2024 already!

Thanks folks, until next time!