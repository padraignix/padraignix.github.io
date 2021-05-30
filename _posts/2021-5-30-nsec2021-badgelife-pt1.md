---
layout:     post
title2:      NSEC 2021 - Badgelife - Main Flags
title:     NSEC 2021 - Badgelife - Main Flags
date:       2021-5-30 12:02:00 -0400
summary:    Walkthrough of the first 9 badge flags for NSEC 2021's hardware badge challenge. The writeup will go through the various in-game solutions as well as the more esoteric ways the flags were discovered. Additionally this will have a first introduction to the ESP32 architecture that while useful in these flag captures was essentially an introduction for the 10th and final flag which required reverse engineering the badge's firmware.
categories: [ctf]
thumbnail:  cogs
keywords:   northsec,nsec,nsec2021,ctf,writeup,binary,reverse engineering,esp32,re,ghidra,firmware,badgelife
thumbnail:  https://padraignix.github.io/assets/nsec2021/badgelife/horse.png
canon:      https://padraignix.github.io/ctf/2021/05/30/nsec2021-badegelife-pt1/
tags:
 - nsec2021
 - ctf
 - reverse-engineering
 - ghidra
 - esp32
 - hardware 
---

<h1>Introduction</h1>
<p>
<img width="50%" height="50%" src="{{ '/assets/nsec2021/badgelife/horse.png' | relative_url }}">
</p>

<p>Ah the horse...</p>

<p>As a preamble to the annual <a href="https://nsec.io/competition/">NSEC CTF</a> the team released a hardware badge challenge that could be ordered before the event. Since this year's edition would be remote due to a small thing like a global pandemic and that I had never taken part in hardware hacking of this nature I decided to snag one up.</p>

<p>What follows over the next two articles illustrates the curiosity, joy, frustration, elation, rage, and finally relief of this journey. Having never stepped foot into the hardware arena I learned a ridiculous amount and this will not be the last <code>#badgelife</code> interaction I have.</p>

<p>I've broken down the writeup into two parts. The first, this article, will cover the first 9 badges which could be captured by direct interaction with the badge whether it be the game, the cli, or to my chagrin listening carefully on the network. I will present them in the numerical order that NSEC's FLAGBOT presented them rather than the order I captured them.</p>

<p>The second article will cover a deeper analysis of the firmware, the game's memory structure, and the eventual modification and flashing to the board to capture the elusive final 10th flag.</p>

<h1>Initial Information</h1>

<p>First thing's first let us figure out what we are working with. If we turn the badge around we get a bit of a better idea.</p>

<p>
<a href="/assets/nsec2021/badgelife/horse-back.png" data-lightbox="image1"><img src="{{ '/assets/nsec2021/badgelife/horse-back.png' | relative_url }}"></a>
</p>

<p>A quick search and we find out the badge is running on an <a href="https://www.espressif.com/en/products/socs/esp32">ESP32</a> chip. Tiny but powerful it seems this badge should be capable of wifi and bluetooth connectivity. Let's keep that in mind as we progress.</p>

<p>Knowing what I do now I would have probably tackled the badge a bit differently, however being completely clueless and excited, I plugged it in and my journey to North Sectoria had commenced.</p>

<p>
<a href="/assets/nsec2021/badgelife/badge1.png" data-lightbox="image2"><img src="{{ '/assets/nsec2021/badgelife/badge1.png' | relative_url }}"></a>
</p>

<p>
<a href="/assets/nsec2021/badgelife/badge2.png" data-lightbox="image3"><img src="{{ '/assets/nsec2021/badgelife/badge2.png' | relative_url }}"></a>
</p>

<h1>Serial Connection</h1>

<p>Before we move on to first flag we will connect to the badge's serial port. I initially ended up using minicom but ended up switching to picocom as there was a frustrating issue with flag 6.</p>

<p>
<a href="/assets/nsec2021/badgelife/serial1.png" data-lightbox="image5"><img src="{{ '/assets/nsec2021/badgelife/serial1.png' | relative_url }}"></a>
</p>

{% highlight shell %}
sudo minicom --device /dev/ttyUSB0
{% endhighlight %}

<p>And once properly connected I quickly took a poke at the options before continuing with the game.</p>

<p>
<a href="/assets/nsec2021/badgelife/serial2.png" data-lightbox="image6"><img src="{{ '/assets/nsec2021/badgelife/serial2.png' | relative_url }}"></a>
</p>

<p>
<a href="/assets/nsec2021/badgelife/serial3.png" data-lightbox="image7"><img src="{{ '/assets/nsec2021/badgelife/serial3.png' | relative_url }}"></a>
</p>

<h1>Flag 1 - Gimme</h1>

<p>The first flag was a gimme. It was meant to introduce you to the format of the badge, how to submit, and what exactly happens when you discover a flag in-game - namely having the flag icon go from grey to red. This information will be important shortly as we will see.</p>

<p>
<a href="/assets/nsec2021/badgelife/flag1-1.png" data-lightbox="image3"><img src="{{ '/assets/nsec2021/badgelife/flag1-1.png' | relative_url }}"></a>
</p>

<p>
<a href="/assets/nsec2021/badgelife/flag1-2.png" data-lightbox="image4"><img src="{{ '/assets/nsec2021/badgelife/flag1-2.png' | relative_url }}"></a>
</p>

<h1>Flag 2 - RE101</h1>

<p>Ah to explore North Sectoria! The wonderful Information building, the mysterious Tom's hut, Marcus Madison bakery, our castle entrance seemingly blocked off... where to first? Let's go with Tom!</p>

<h2>Wifi connection</h2>
<p>
<a href="/assets/nsec2021/badgelife/flag2-1.png" data-lightbox="image8"><img src="{{ '/assets/nsec2021/badgelife/flag2-1.png' | relative_url }}"></a>
</p>

<p>
<a href="/assets/nsec2021/badgelife/flag2-2.png" data-lightbox="image9"><img src="{{ '/assets/nsec2021/badgelife/flag2-2.png' | relative_url }}"></a>
</p>

<p>Well that's rude Tom... fine let's go get connected. There was a building near the start with a wifi symbol so that seems like a good target. Unfortunately it only allowed to toggle the wifi connection on and off, so I ended up having to connect using the serial connection.</p>

<p>
<a href="/assets/nsec2021/badgelife/flag2-5.png" data-lightbox="image10"><img src="{{ '/assets/nsec2021/badgelife/flag2-5.png' | relative_url }}"></a>
</p>

<p>At this point the wifi shack shows us connected and allows us to toggle the status in game.</p>

<p>
<a href="/assets/nsec2021/badgelife/flag2-3.png" data-lightbox="image11"><img src="{{ '/assets/nsec2021/badgelife/flag2-3.png' | relative_url }}"></a>
</p>

<p>Let's go back to Tom's hut and see what have!</p>

<p>
<a href="/assets/nsec2021/badgelife/flag2-4.png" data-lightbox="image12"><img src="{{ '/assets/nsec2021/badgelife/flag2-4.png' | relative_url }}"></a>
</p>

<p>Alright that is something. Also explains why we needed a connection. They want us to download two zip files that should correspond to two RE challenges. Bring it on Tom!</p>

<h2>Xtensa</h2>

<p>Extracting the contents of both zip files we get two ELF files. Running some initial recon we can see that they are <a href="https://0x04.net/~mwk/doc/xtensa.pdf">Tensilica Xtensa</a> format. The first thing I wanted to do at this point was get the first elf imported to Ghidra. Unfortunately Ghidra did not have an understanding of what this format was and the result was less that useful.</p>

<p>
<a href="/assets/nsec2021/badgelife/flag2-6.png" data-lightbox="image13"><img src="{{ '/assets/nsec2021/badgelife/flag2-6.png' | relative_url }}"></a>
</p>

<p>A quick poke around the net and stumbled on to this <a href="https://github.com/yath/ghidra-xtensa">Ghidra extension</a>. Once installed we were good to go and the import went much smoother.</p>

<p>The first thing we needed to do was find the entry point and understand what the code was doing. Sure enough <code>app_main</code> was the intro function and the logic seems simple enough to follow even without fully understanding the Xtensa opcodes fully.</p>

* Take user input
* Call verify function
* If verify works, load flag into register and print
* Else, jump to "nope_try_again" function

<p>
<a href="/assets/nsec2021/badgelife/flag2-7.png" data-lightbox="image14"><img src="{{ '/assets/nsec2021/badgelife/flag2-7.png' | relative_url }}"></a>
</p>

<p>The verify function at first seems confusing, but ultimately the logic was straightforward.</p>

<p>
<a href="/assets/nsec2021/badgelife/flag2-8.png" data-lightbox="image15"><img src="{{ '/assets/nsec2021/badgelife/flag2-8.png' | relative_url }}"></a>
</p>

<p>The stack offsets represented our final flag values. Starting at 0x0 and inscreasing by 0x4 each character we can see the function is validating our user input, a1, against values loaded to the various a8-a15 registers. This should be as simple as taking the values loaded into the registers and building the input one character at a time. Once we've stepped through the full iteration all that was left was to validate using the badge's CLI.</p> 

<p>
<a href="/assets/nsec2021/badgelife/flag2-9.png" data-lightbox="image16"><img src="{{ '/assets/nsec2021/badgelife/flag2-9.png' | relative_url }}"></a>
</p>

<h1>Flag 3 - RE102</h1>

<p>The second RE challenge was much of the same. ELF in Xtensa format, app_main function and verify function.</p>

<p>
<a href="/assets/nsec2021/badgelife/flag3-1.png" data-lightbox="image17"><img src="{{ '/assets/nsec2021/badgelife/flag3-1.png' | relative_url }}"></a>
</p>

<p>
<a href="/assets/nsec2021/badgelife/flag3-2.png" data-lightbox="image18"><img src="{{ '/assets/nsec2021/badgelife/flag3-2.png' | relative_url }}"></a>
</p>

<p>In this case we are starting at 0x0 with a9 and using a similar approach as flag2 of incrementing by 0x1 for each of the flag's characters. We start by comparing a9[0x0] with 0x66, or f in ascii, before continuing down the line.</p>

<p>
<a href="/assets/nsec2021/badgelife/flag3-3.png" data-lightbox="image19"><img src="{{ '/assets/nsec2021/badgelife/flag3-3.png' | relative_url }}"></a>
</p>

<p>
<a href="/assets/nsec2021/badgelife/flag3-4.png" data-lightbox="image20"><img src="{{ '/assets/nsec2021/badgelife/flag3-4.png' | relative_url }}"></a>
</p>

<p>
<a href="/assets/nsec2021/badgelife/flag3-5.png" data-lightbox="image21"><img src="{{ '/assets/nsec2021/badgelife/flag3-5.png' | relative_url }}"></a>
</p>

<p>
<a href="/assets/nsec2021/badgelife/flag3-6.png" data-lightbox="image22"><img src="{{ '/assets/nsec2021/badgelife/flag3-6.png' | relative_url }}"></a>
</p>

<p>With a final flag being submitted through the CLI as well.</p>

<p>
<a href="/assets/nsec2021/badgelife/flag3-7.png" data-lightbox="image23"><img src="{{ '/assets/nsec2021/badgelife/flag3-7.png' | relative_url }}"></a>
</p>

<h1>Flag 4 - Konami</h1>

<p>Ah <a href="https://en.wikipedia.org/wiki/Konami_Code">Konami code</a>. I peppered this through various parts of the game - main screen, in buildings, in conversations, and it finally paid off. There was a flag chest on the top left area of the map that seemingly was blocked off.</p>

<p>
<a href="/assets/nsec2021/badgelife/flag4-1.png" data-lightbox="image24"><img src="{{ '/assets/nsec2021/badgelife/flag4-1.png' | relative_url }}"></a>
</p>

<p>Angela dropped an interesting hint about how we could reach it when talking with her. This was a nice touch throughout the badge where the NPC conversations may have appeared random but in reality were related to the various flags</p>

<p>
<a href="/assets/nsec2021/badgelife/flag4-2.png" data-lightbox="image25"><img src="{{ '/assets/nsec2021/badgelife/flag4-2.png' | relative_url }}"></a>
</p>

<p>Suspicious indeed! When putting in the Konami code a magical door opens up for us!</p>

<p>
<a href="/assets/nsec2021/badgelife/flag4-3.png" data-lightbox="image26"><img src="{{ '/assets/nsec2021/badgelife/flag4-3.png' | relative_url }}"></a>
</p>

<p>
<a href="/assets/nsec2021/badgelife/flag4-4.png" data-lightbox="image27"><img src="{{ '/assets/nsec2021/badgelife/flag4-4.png' | relative_url }}"></a>
</p>

<h1>Flag 5 - Blocked Map</h1>

<p>The fifth flag is where things started going off the deep end for me. In reality you can solve this flag quite easily by just playing the game and paying attention to small hints. I however have a tendency of complicating things when approaching these problems.</p>

<h2>Intended Solution</h2>

<p>The first indication comes from seeing a tiny land mass at the bottom right of the map. Seemingly unreachable it stuck out as "why would that be there"? This was reinforced by Vicious' initial hint of perceiving that the landmass is there.</p>

<p>
<a href="/assets/nsec2021/badgelife/flag5-1.png" data-lightbox="image28"><img src="{{ '/assets/nsec2021/badgelife/flag5-1.png' | relative_url }}"></a>
</p>

<p>To add on to this one of the Fisherman's conversation points seemingly alluded to the land mass as well.</p>

<p>
<a href="/assets/nsec2021/badgelife/flag5-2.png" data-lightbox="image29"><img src="{{ '/assets/nsec2021/badgelife/flag5-2.png' | relative_url }}"></a>
</p>

<p>At this point I started looking in the most random places for a way to that landmass. I was thinking similar to flag4 there could be a secret tunnel and way to get to the island. After all another one of the villagers did mention a "secret underground"!</p>

<p>Since at this point in my adventure I had already dumped the firmware and explored that a bit I ended up taking a tangent in solving flag 5 by approached it with the alternate approach below. This was absolutely not required however as I realized I actually ran into the "secret" area by mistake early on and dismissed it...</p>

<p>
<a href="/assets/nsec2021/badgelife/flag5-3.png" data-lightbox="image30"><img src="{{ '/assets/nsec2021/badgelife/flag5-3.png' | relative_url }}"></a>
</p>

<p>Then by proceeding off screen you could go the entire way down the map until you reached a hidden chest. Flag 5 complete!</p>

<p>
<a href="/assets/nsec2021/badgelife/flag5-4.png" data-lightbox="image31"><img src="{{ '/assets/nsec2021/badgelife/flag5-4.png' | relative_url }}"></a>
</p>

<p>
<a href="/assets/nsec2021/badgelife/flag5-5.png" data-lightbox="image32"><img src="{{ '/assets/nsec2021/badgelife/flag5-5.png' | relative_url }}"></a>
</p>

<h2>Alternative Solution</h2>

<p>This followed dumping the firmware and looking through the SPIFFS storage partition. I'll keep the how I dumped the firmware for the second article, but all you need to know is that the firmware had a specific storage partition. </p>

<p>
<a href="/assets/nsec2021/badgelife/flag5-6.png" data-lightbox="image33"><img src="{{ '/assets/nsec2021/badgelife/flag5-6.png' | relative_url }}"></a>
</p>

<p>After extracting the binary portion of the partition I took a poke at it using binwalk. At first I thought I had hit jackpot but quickly realized it wouldn't be that easy.</p>

<p>
<a href="/assets/nsec2021/badgelife/flag5-7.png" data-lightbox="image34"><img src="{{ '/assets/nsec2021/badgelife/flag5-7.png' | relative_url }}"></a>
</p>

<p>When trying to extract the files using binwalk I was running into weird corruption issues.</p>

<p>
<a href="/assets/nsec2021/badgelife/flag5-8.png" data-lightbox="image35"><img src="{{ '/assets/nsec2021/badgelife/flag5-8.png' | relative_url }}"></a>
</p>

<p>I spent a bit of time reading on how the storage partitions for ESP32 chips works and learned that we were working with a <a href="https://docs.espressif.com/projects/esp-idf/en/latest/esp32/api-reference/storage/spiffs.html">SPIFFS Filesystem</a>. This makes sense why binwalk could recognize some of the file headers and list them out, but due to the SPIFFS block and page sizes it wasn't able to extract properly. To do so we would need a utility like <a href="https://github.com/igrr/mkspiffs">mkspiffs</a>.</p>

<p>Unfortunately there were many configuration option possibilities during both the mkspiffs build and when trying to extract the filesystem - neither of which had obvious tells off the bat. One of the intial investigations involved going through the hexdump of the partition. At this point I did notice an interesting pattern.</p>

<p>
<a href="/assets/nsec2021/badgelife/flag5-9.png" data-lightbox="image36"><img src="{{ '/assets/nsec2021/badgelife/flag5-9.png' | relative_url }}"></a>
</p>

<p>It seems like there is a repeating pattern every 0x400 bytes where it is 0xFF padded. This ended up being the page size of the page - 1024 bytes. Great one step closer!</p>

<p>After a bit of discussion with other <code>#badgelifers</code> I had the right build options for mkspiffs. I built the tool locally and passed it a page size of 1024. Lo and behold the extraction worked!</p>

<p>
<a href="/assets/nsec2021/badgelife/flag5-10.png" data-lightbox="image37"><img src="{{ '/assets/nsec2021/badgelife/flag5-10.png' | relative_url }}"></a>
</p>

<p>At this point it was a matter of going through the material and trying to understand if there was anything useful. Maybe an image file indicating a door could be opened, or a secret flower pot?! It didn't seem like that was the case, but there was an interesting folder under rpg that had a <code>main.scene</code> and <code>main.blocked</code> file.</p>

<p>It took awhile to understand but after I loosely calculated the amount of tiles in game I started realizing that the blocked file was being used to determine enterable and "blocked" areas of the map!</p>

<p>The blocked file was exactly 5000 bytes. If we look at it in bits - 5000*8 = 40000, that comes out to a nice round number of 200 by 200 tiles, which came out close to our manual estimations!</p>

<p>What was left was to try and print out the map and see what we could find.</p>

<p>
<a href="/assets/nsec2021/badgelife/flag5-11.png" data-lightbox="image38"><img src="{{ '/assets/nsec2021/badgelife/flag5-11.png' | relative_url }}"></a>
</p>

<p>Yes! We can see our island area at the bottom right of the map, and even better seemingly a path leading north. If we follow it top the top of the map we can find the secret entrance.</p>

<p>
<a href="/assets/nsec2021/badgelife/flag5-12.png" data-lightbox="image39"><img src="{{ '/assets/nsec2021/badgelife/flag5-12.png' | relative_url }}"></a>
</p>

<p>
<a href="/assets/nsec2021/badgelife/flag5-3.png" data-lightbox="image40"><img src="{{ '/assets/nsec2021/badgelife/flag5-3.png' | relative_url }}"></a>
</p>

<h1>Flag 6 - Quack</h1>

<p>This flag tested how perceptive you were while working with the badge. Seemingly randomly participants would realize a new badge popped up on the UI. But while the flag was seen as captured, there had been no flag text shown, nothing to submit to flagbot in the NSEC discord...</p>

<p>Thankfully I would be working on the badge while I had the serial connection in the screen in front of me. This ended up being paramount to seeing flag 6.</p>

<p>When the mysterious flag appearance occured for me I was able to narrow it down to about 5 actions I had taken. I had walked through the middle market area, talked with a few villagers, and visited "lac du quack".</p>

<p>
<a href="/assets/nsec2021/badgelife/flag6-1.png" data-lightbox="image41"><img src="{{ '/assets/nsec2021/badgelife/flag6-1.png' | relative_url }}"></a>
</p>

<p>Retracing my steps carefully I noticed when I tried talking with the duck my serial console flickered. This seemed weird and after confirming the behavior happened every time I talked with the duck I knew I should investigate further. I setup my minicom terminal to dump all output to file and repeated the process. This time I managed to catch that ellusive duck quack!</p>

<p>
<a href="/assets/nsec2021/badgelife/flag6-2.png" data-lightbox="image42"><img src="{{ '/assets/nsec2021/badgelife/flag6-2.png' | relative_url }}"></a>
</p>

<p>
<a href="/assets/nsec2021/badgelife/flag6-3.png" data-lightbox="image43"><img src="{{ '/assets/nsec2021/badgelife/flag6-3.png' | relative_url }}"></a>
</p>

<p>Looks like a cipher to me. I ended up going through the most obvious possibilities, but nothing was making sense. The fact that "quick" was used also made me think of potential morse code usage, but without spaces this wouldn't really work.</p>

<p>
<a href="/assets/nsec2021/badgelife/flag6-4.png" data-lightbox="image44"><img src="{{ '/assets/nsec2021/badgelife/flag6-4.png' | relative_url }}"></a>
</p>

<p>As it turns out my hunch was right, it was just an issue with minicom and how it interpreted data. After a quick discussion with the mods they suggested I try a different serial connection - enter picocom. Repeating the cycle with picocom we see a slight, but definitive, difference.</p>

<p>
<a href="/assets/nsec2021/badgelife/flag6-5.png" data-lightbox="image45"><img src="{{ '/assets/nsec2021/badgelife/flag6-5.png' | relative_url }}"></a>
</p>

<p>My hunch was right! Using <code>^[[H^[[2J^[[3J</code> (which is the "clear" control sequence, explaining the seemingly empty serial output) as the morse code delimiter we are able to finish up the decoding.</p>

<p>
<a href="/assets/nsec2021/badgelife/flag6-6.png" data-lightbox="image46"><img src="{{ '/assets/nsec2021/badgelife/flag6-6.png' | relative_url }}"></a>
</p>

<h1>Flag 7 - Punk</h1>

<p>I solved this one accidentally while spamming the konami code almost everywhere. When talking with the Punk and getting half way through the konami code I was instantly presented with the flag.</p>

<p>
<a href="/assets/nsec2021/badgelife/flag7-1.png" data-lightbox="image47"><img src="{{ '/assets/nsec2021/badgelife/flag7-1.png' | relative_url }}"></a>
</p>

<h2>Alternative Solution</h2>

<p>Once you had the firmware dumped there were a few extra ways of finding this flag. When looking at the NPC conversations you notice that the Punk's conversation doesn't include multiple options as everyone else, and there are newlines included before eventually printing the flag. This explains how when going through the Konami code and getting to "down" that I was able to see the flag.</p>

<p>
<a href="/assets/nsec2021/badgelife/flag7-2.png" data-lightbox="image48"><img src="{{ '/assets/nsec2021/badgelife/flag7-2.png' | relative_url }}"></a>
</p>

<p>
<a href="/assets/nsec2021/badgelife/flag7-3.png" data-lightbox="image49"><img src="{{ '/assets/nsec2021/badgelife/flag7-3.png' | relative_url }}"></a>
</p>

<h1>Flag 8 - Secret CLI</h1>

<p>Another flag where I was probably my own worst enemy. I solved this one only after having dumped the firmware, and I definitely went on some crazy tangents trying to piece together the secret CLI call from the code.</p>

<p>When looking at the serial command options we noticed there was a "???".</p>

<p>
<a href="/assets/nsec2021/badgelife/flag8-1.png" data-lightbox="image50"><img src="{{ '/assets/nsec2021/badgelife/flag8-1.png' | relative_url }}"></a>
</p>

<h2>Intended Solution</h2>

<p>Now this was actually incorrect as it's a sub-option for the RE option... but it set me off the path to dig into the code and shed some light. I'll share in the alternate section that hunt.</p>

<p>The obvious hint I missed was something presented when you were connected through the serial port during a badge reboot. When that was the case you would be presented with some additional information. One being a big glaring hint.</p>

<p>
<a href="/assets/nsec2021/badgelife/flag8-2.png" data-lightbox="image51"><img src="{{ '/assets/nsec2021/badgelife/flag8-2.png' | relative_url }}"></a>
</p>

<p>Auto-complete you say? Should have seen that sooner... going through each character and pressing tab we are eventually rewarded at 't'.</p>

<p>
<a href="/assets/nsec2021/badgelife/flag8-3.png" data-lightbox="image52"><img src="{{ '/assets/nsec2021/badgelife/flag8-3.png' | relative_url }}"></a>
</p>

<h2>Alternative Solution</h2>

<p>I lost a lot of time digging into the firmware trying to find the reference to this mythical secret command. I was able to initially find where our CLI commands were being registered.</p>

<p>
<a href="/assets/nsec2021/badgelife/flag8-4.png" data-lightbox="image53"><img src="{{ '/assets/nsec2021/badgelife/flag8-4.png' | relative_url }}"></a>
</p>

<p>Even finding exactly where the <code>hidden_cmd</code> was registered.</p>

<p>
<a href="/assets/nsec2021/badgelife/flag8-5.png" data-lightbox="image54"><img src="{{ '/assets/nsec2021/badgelife/flag8-5.png' | relative_url }}"></a>
</p>

<p>It turns out that the <code>hint_cmd</code> was something that slipped through QA and shouldn't have made it into the badge. An interesting easter egg left to confuse those easily confused like me.</p>

<p>Unfortunately my poking wasn't returning anything further and I ended up solving the flag the intended way with auto-completion.</p>

<h1>Flag 9 - Noisy Network</h1>

<p>Ah flag 9... my nemesis. This was actually the very last flag I capture during my time with <code>the_horse</code>. I had already reverse engineered the NVS partition and modified the flag save states giving myself all 10 flags (essentially solving the 10th flag, tacking on the 9th while we were at it) and yet the proper way of doing this was eluding me.</p>

<p>The frustrating part is that the solution to this flag was something I identified and dismissed on the very first day.</p>

<p>When I first got the badge and realized I had to connect it to wifi I got curious and popped open wireshark to see if anything interesting was being broadcasted. In my naivity I dismissed noise I was seeing and moved on with the rest of the badge challenges.</p>

<p>My mistake was forgetting that I had used a second wifi card in promiscuous mode to do the sniffing. Of course the 802.11 traffic I was seeing in wireshark was gibberish I wasn't on the same network and the traffic was encrypted!</p>

<p>After several days of trying to turn over every rock (and hints from Vicious) I went back to the network angle. This time realizing what my mistake was during the early days I connected the_horse to my machine directly. Sure enough this changed everything.</p>

<p>
<a href="/assets/nsec2021/badgelife/flag9-1.png" data-lightbox="image55"><img src="{{ '/assets/nsec2021/badgelife/flag9-1.png' | relative_url }}"></a>
</p>

<p>For reference my badge's IP was <code>192.168.12.171</code> at this point. This output clearly shows an interesting behavior. I wonder why it is trying to reach out to <code>198.51.100.42</code> and especially on port 4444... reverse shell listener anyone?</p>

<p>Since I was using create_ap to get the badge connected to my instance it was trivial to pass it a specific IP configuration. If the badge wants to connect to this address let's give it a path!</p>

<p>
<a href="/assets/nsec2021/badgelife/flag9-2.png" data-lightbox="image56"><img src="{{ '/assets/nsec2021/badgelife/flag9-2.png' | relative_url }}"></a>
</p>

<p>We also need to setup a listener locally on port 4444 to see if the badge does anything interesting once it reaches the destination. Sure enough within a few seconds both wireshark and our listener get a hit.</p>

<p>
<a href="/assets/nsec2021/badgelife/flag9-3.png" data-lightbox="image57"><img src="{{ '/assets/nsec2021/badgelife/flag9-3.png' | relative_url }}"></a>
</p>

<p>
<a href="/assets/nsec2021/badgelife/flag9-4.png" data-lightbox="image58"><img src="{{ '/assets/nsec2021/badgelife/flag9-4.png' | relative_url }}"></a>
</p>

<p>Relief! I think I actually jumped in my seat a little bit when this flag popped. Several days of thinking I'm going crazy finally paying off.</p>

<h1>Bonus - Castle Area</h1>

<p>There was an entire castle area that seemingly we could not get into. While solving flag 5 the "alternate" way I noticed something interesting in the blocked output.</p>

<p>
<a href="/assets/nsec2021/badgelife/castle-1.png" data-lightbox="image59"><img src="{{ '/assets/nsec2021/badgelife/castle-1.png' | relative_url }}"></a>
</p>

<p>Is that a walkable path into the castle?!</p>

<p>
<a href="/assets/nsec2021/badgelife/castle-2.png" data-lightbox="image60"><img src="{{ '/assets/nsec2021/badgelife/castle-2.png' | relative_url }}"></a>
</p>

<p>
<a href="/assets/nsec2021/badgelife/castle-3.png" data-lightbox="image61"><img src="{{ '/assets/nsec2021/badgelife/castle-3.png' | relative_url }}"></a>
</p>

<p>Sure is! Unfortunately at the time I was still looking for the final flags and there didn't end up being any in the area. There was however a very interesting spiral door. When trying to enter it I had a minor heart attack thinking I had messed up my badge.</p>

<p>
<a href="/assets/nsec2021/badgelife/castle-4.png" data-lightbox="image62"><img src="{{ '/assets/nsec2021/badgelife/castle-4.png' | relative_url }}"></a>
</p>

<p>Thankfully this ended up being an intended easter egg by the devs. A quick badge reset and I was back to normal. While not part of the flag capturing journey directly this was a nice little side addition if for nothing else to keep us on our toes.</p>

<h1>Summary</h1>

<p>Wow what a time. As a first time <code>#badgelifer</code> I really enjoyed the entire journey - frustration, excitement, relief, and confusion included. I ended up learning a lot about hardware and badge hacking in general and as the second article will hopefully convey, a small understanding in the ESP32 architecture and IoT in general.</p>

<p>I can't thank Vicious, Tom, and the entire team enough for putting this together. The entire adventure was high quality and even tied in directly to overall NSEC 2021 North Sectoria theme, making it all the more special. They set the bar high with this edition and I can't wait to see what's in store next year.</p>

<p>As always thanks folks, until next time!</p>
