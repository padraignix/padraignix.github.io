---
layout:     post
title2:     Ballad for a Badge - NSEC 2026 Team Edition
title:      Ballad for a Badge - NSEC 2026
date:       2026-08-30 15:00:00 -0400
summary:    A behind-the-scenes journey through the design, development, and delivery of the NSEC 2026 badge. This post follows an amazing team from the first ideas in October 2025 through hardware bring-up, firmware development, production, a last-minute shipping crisis, and a successful launch at NorthSec.
categories: [ctf]
math:       false
keywords:   northsec,nsec,nsec2026,badge,badgelife,hardware,pcb,firmware,nfc,embedded,team
thumbnail:  https://blog.quantumlyconfused.com/assets/nsec2026/badge-journey/header.png
permalink:  /ctf/2026/09/01/nsec2026-badge-journey/
canon:      https://blog.quantumlyconfused.com/ctf/2026/09/01/nsec2026-badge-journey/
tags:
 - nsec2026
 - badgelife
 - hardware
 - pcb
 - firmware
 - nfc
---

<h1>Introduction</h1>

<!-- IMAGE: /assets/nsec2026/badge-journey/team-kickoff.jpg (team working session) -->
<p>
<a href="/assets/nsec2026/badge-journey/header.png" data-lightbox="header"><img width="100%" height="100%" src="{{ '/assets/nsec2026/badge-journey/header.png' | relative_url }}"></a>
</p>

Every year, NorthSec brings thousands of people together for a few intense days of security, competition, learning, and community. Making that happen is a large undertaking in its own right. Building custom electronic hardware for those attendees adds another project inside the project: a product has to be imagined, engineered, manufactured, programmed, assembled, and delivered against a deadline that cannot move.

The conference badge is a highlight of the conference according to attendees. It is something to wear, explore, hack, and keep after the event. To the team behind it, the finished board represents months of design reviews, pull requests, component orders, prototype tests, firmware debugging, manufacturing decisions, and late-night conversations. No single person could carry all of that. The badge only happens because an amazing group of people brought different skills to the table and repeatedly stepped in wherever the project needs them.

Our work on the NSEC 2026 badge began in October 2025. Over the next seven months, an idea became artwork and schematics, then a slightly-working prototype, a complete prototype, and eventually a complete hardware platform. The final weeks leading to the conference could be described as chaos. Not because of the team, they were amazing individuals to work with. Component availability shifted, a shipment became trapped in an administrative maze, and the margin between delivery and the start of the conference disappeared entirely.

Let's dive into how a NSEC conference badge is made!

<p>
<a href="/assets/nsec2026/badge-journey/project-timeline.svg" data-lightbox="project-timeline"><img width="100%" src="{{ '/assets/nsec2026/badge-journey/project-timeline.svg' | relative_url }}" alt="Timeline of NSEC 2026 badge milestones from the October 2025 kick-off through the May 14, 2026 conference launch"></a>
</p>
<p class="text-center"><em>Seven months of parallel work and a very steep climb from confidence to chaos.</em></p>

<h1>October to January - Finding the Badge's Soul</h1>

The first conversations in October were deliberately broad. We knew the theme was going to be Solarpunk, and we talked through some of the high-level aspects - which components we would want to target, certain features we wanted to introduce, and overall the user experience with the badges. At that point the badge was more possibility than product.

<!-- IMAGE: /assets/nsec2026/badge-journey/team-kickoff.jpg (team working session) -->
<p>
<a href="/assets/nsec2026/badge-journey/kickoff.jpg" data-lightbox="image1"><img width="100%" height="100%" src="{{ '/assets/nsec2026/badge-journey/kickoff.jpg' | relative_url }}"></a>
</p>

That breadth is what makes a badge project so interesting, but also what makes it difficult. It is artwork, electronics, a firmware target, a communications platform, and a manufactured product at the same time. Even the framework around the firmware was something we decided to try differently this year. A small feature request can affect board routing, component count, power consumption, firmware complexity, cost, and lead time. A beautiful shape can make panelization difficult, as we found out with some of the shifting art direction. A perfect component can become unusable if the supplier cannot deliver enough of them, as we also found out with regards to the screens. Every decision crosses borders (technical, and geopolitical), so every discipline has to remain in conversation with the others.

The early concepts went in a very different visual direction, including a tall crystalline form with interconnected elements. It was a useful beginning, but not yet the badge. The overall art direction for the conference evolved, and the badge design had to adjust with it.

<!-- IMAGE: /assets/nsec2026/badge-journey/design-concept-early.png (purple crystal concept) -->
<p>
<a href="/assets/nsec2026/badge-journey/crystal1-pcb.png" data-lightbox="image2"><img width="80%" height="80%" src="{{ '/assets/nsec2026/badge-journey/crystal1-pcb.png' | relative_url }}"></a>
</p>

An intermediate design introduced the hexagonal outline, organic elements, a prominent centrepiece, and the large NSEC wordmark.

<!-- IMAGE: /assets/nsec2026/badge-journey/design-concept-intermediate.png (hexagonal NSEC concept) -->
<p>
<a href="/assets/nsec2026/badge-journey/crystal2.png" data-lightbox="image3"><img width="80%" height="80%" src="{{ '/assets/nsec2026/badge-journey/crystal2.png' | relative_url }}"></a>
</p>

By the end of January, the badge started to find its identity and the design direction was mostly finalized. 

<!-- IMAGE: /assets/nsec2026/badge-journey/project-timeline.png -->
<p>
<a href="/assets/nsec2026/badge-journey/crystal3.png" data-lightbox="image5"><img width="60%" height="60%" src="{{ '/assets/nsec2026/badge-journey/crystal3.png' | relative_url }}"></a>
</p>

The schedule already showed the scale of the undertaking. Long component lead times forced parts to be pre-ordered before the hardware was fully tested. PCB manufacturing and shipping consumed weeks, not days, and the boards would still require assembly, flashing, and validation after they arrived. The timeline looked sequential on paper, but the team could not work sequentially. Hardware, firmware, procurement, artwork, production tooling, and CTF development all had to move in parallel if we were going to be ready for May.

<h1>February and March - Making It Real</h1>


Prototype V1 arrived on February 16. Holding the first assembled board changed the project immediately. We could start to evaluate the outline, artwork, component placement, connectors, and electrical behaviour as one system. More importantly, the question changed from "should this work?" to the much more useful "what works, what does not, and who can help move it forward?". A few fun details was that the V1 prototype accidentally didn't have the components included on the front panel, and the original NFC built-in antenna was too small to work reliably. So buttons, SAOs, and light sensors were omitted, but with a DIY antenna fix, some manually soldered front buttons, thankfully most of the functionality lived on the back and we were able to get coding.

<!-- IMAGE: /assets/nsec2026/badge-journey/prototype-v1.jpg -->
<p>
<a href="/assets/nsec2026/badge-journey/protov1-1.jpg" data-lightbox="image6"><img width="60%" height="60%" src="{{ '/assets/nsec2026/badge-journey/protov1-1.jpg' | relative_url }}"></a>
</p>

<!-- IMAGE: /assets/nsec2026/badge-journey/prototype-v1.jpg -->
<p>
<a href="/assets/nsec2026/badge-journey/protov1-2.jpg" data-lightbox="image4"><img width="60%" height="60%" src="{{ '/assets/nsec2026/badge-journey/protov1-2.jpg' | relative_url }}"></a>
</p>

The first boot target was intentionally simple: make the LEDs respond. When the test firmware lit them, we had our first visible confirmation that the board could be programmed and controlled. It was a small technical milestone and a large morale boost. From there, the work accelerated and spread across the team.

{% include embed/video.html
  src="/assets/nsec2026/badge-journey/protov1-Ledtest.mp4"
  autoplay=false
  muted=false
  loop=true
%}

Then a quick test suite to validate the manually soldered buttons.

<!-- IMAGE: /assets/nsec2026/badge-journey/prototype-v1.jpg -->
<p>
<a href="/assets/nsec2026/badge-journey/protov1-button.png" data-lightbox="image6"><img width="60%" height="60%" src="{{ '/assets/nsec2026/badge-journey/protov1-button.png' | relative_url }}"></a>
</p>

```
Waiting for button presses...
(Press Ctrl+C or wait 30s to exit early)

  [PRESS]   A (IO17)  ✓ OK
  [RELEASE] A
  [PRESS]   B (IO18)  ✓ OK
  [RELEASE] B
  [PRESS]   Right (IO16)  ✓ OK
  [RELEASE] Right
  [PRESS]   Up (IO13)  ✓ OK
  [RELEASE] Up
  [PRESS]   Left (IO15)  ✓ OK
  [RELEASE] Left
  [PRESS]   Down (IO14)  ✓ OK

--- Results ---
  A       IO17  PASS
  B       IO18  PASS
  Left    IO15  PASS
  Right   IO16  PASS
  Up      IO13  PASS
  Down    IO14  PASS

6/6 buttons verified.
```

A GitHub project board turned the larger vision into work that could be owned, discussed, and reviewed. Hardware tasks sat beside core firmware, CTF functionality, module integrations, and docking ideas. That visibility mattered, and was a new structure put in place this year. It allowed people to pick up work, identify dependencies, review each other's decisions, and contribute in multiple areas. The badge was no longer just a PCB being designed by a few people, it had become a platform being built by a team.

<!-- IMAGE: /assets/nsec2026/badge-journey/github-project-march-8.png -->
<p>
<a href="/assets/nsec2026/badge-journey/devboard1.png" data-lightbox="image7"><img width="100%" height="100%" src="{{ '/assets/nsec2026/badge-journey/devboard1.png' | relative_url }}"></a>
</p>

The beginning of March brought a steady series of incremental technical vistories. Test firmware exercised the memory functions and established a known-good base for later features. Pre-ordered parts were sent to the fabricators making earlier component decisions tangible and increasingly expensive to change. By March 5, NFC card emulation and reading were both operational. Concepts that had existed in planning documents were becoming real interactions between badges, cards, modules, and CTF systems.

{% include embed/video.html
  src="/assets/nsec2026/badge-journey/protov1-nvs.mp4"
  autoplay=false
  muted=false
  loop=true
%}

{% include embed/video.html
  src="/assets/nsec2026/badge-journey/protov1-nfc-pair.mp4"
  autoplay=false
  muted=false
  loop=true
%}

By March 8, the project board showed how much the effort had expanded. Some features were complete, others were in review, and the backlog stretched across firmware, integrations, hardware files, CTF ideas, and docking. Progress was happening because people were working on different problems simultaneously while still converging on a common system.

The hardware itself continued to evolve. We discussed adding another solder mask layer to Prototype V2. Solder mask is functional, but on an artistic PCB it is also part of the colour palette. The change could improve protection and contrast while making the physical badge feel substantially different without altering its outline.

<!-- IMAGE: /assets/nsec2026/badge-journey/final-design.png (final board render) -->
<p>
<a href="/assets/nsec2026/badge-journey/protov2-solder.png" data-lightbox="image8"><img width="80%" height="80%" src="{{ '/assets/nsec2026/badge-journey/protov2-solder.png' | relative_url }}"></a>
</p>

The revised design retained the form developed in January while adding far more visual depth. Copper, mask, silkscreen, components, and traces all became part of the illustration. The badge was no longer artwork placed on top of electronics; the electronics were the artwork.

<!-- IMAGE: /assets/nsec2026/badge-journey/manufacturing-panel.png -->
<p>
<a href="/assets/nsec2026/badge-journey/protov2-panel.png" data-lightbox="image9"><img width="80%" height="80%" src="{{ '/assets/nsec2026/badge-journey/protov2-panel.png' | relative_url }}"></a>
</p>


On March 13, we finally showed the outside world a glimpse of what the team had been building. Had we known the upcoming shipping and border delays, we may not have been so enthusiastic, however we were living in blissful ignorance at this point. The NSEC newsletter reveal leaned into the Solarpunk theme, exposing enough of the assembled prototype to make it real without giving away the complete board. Behind that public milestone, peer-to-peer pairing was also complete. The following day, a Wi-Fi portal proof of concept was demonstrated. Within two weeks, memory, NFC, badge-to-badge pairing, and a browser-based wireless interface had all moved from plans into working code.

<!-- IMAGE: /assets/nsec2026/badge-journey/newsletter-reveal.png -->
<p>
<a href="/assets/nsec2026/badge-journey/newsletter.jpg" data-lightbox="image10"><img width="70%" height="70%" src="{{ '/assets/nsec2026/badge-journey/newsletter.jpg' | relative_url }}"></a>
</p>

The value of the captive portal was to give participants an easy way to integrate with the badge internals, especially since we were not longer going with a screen due to lack of inventory, through a fmailiar wireless interface as something a user could operate in a browser. With that path working, the polish of user experience was expanded further than what originally would have been capable with only an onboard screen.

<!-- MEDIA: /assets/nsec2026/badge-journey/wifi-portal-poc.mp4 (Wi-Fi portal demonstration) -->
{% include embed/video.html
  src="/assets/nsec2026/badge-journey/captive-wifi-portal.mp4"
  autoplay=false
  muted=false
  loop=true
%}

Prototype V2 arrived on March 24, carrying the lessons from the first board into physical layout fixes, such as an extended NFC antenna, manufacturing adjustments, and visual refinements. NFC Pairing worked out of the box with existing code. That result said more than a passing test alone: the work completed against Prototype V1 had transferred cleanly to the revised hardware, and the different parts of the project were integrating as intended. Four days later the e-ink display was validated, confirming that the screen, electrical interface, and firmware driver all behaved together.

<!-- IMAGE: /assets/nsec2026/badge-journey/eink-validation.jpg -->
<p>
<a href="/assets/nsec2026/badge-journey/protov2-eink.jpg" data-lightbox="eink"><img width="55%" height="55%" src="{{ '/assets/nsec2026/badge-journey/protov2-eink.jpg' | relative_url }}"></a>
</p>

On March 29, one day after validating the display, we ordered the production badges. To give numbers, we ordered around 1500 assembled badges. A prototype always leaves room for one more revision; a production order turns decisions into hundreds of physical boards and starts a clock that is much harder to move. After five months of design and bring-up, the project crossed from development into delivery.

<h1>April - The Race to Be Ready</h1>

April became a race along several parallel tracks. Even if the production badges were ordered, we hadn't finalized every detail in code. We had enough confidence that all the components would work, but we needed to integrate everything into a single conference badge framework. Early in the month, the light sensor was calibrated and integrated into the firmware. Raw colour channels became useful lux and colour-temperature readings, thresholds behaved predictably, and detected events could be retained in non-volatile storage. It was another example of the detailed work hidden behind a finished feature, placing a sensor on the schematic was only the beginning, calibration made it useful.

<!-- IMAGE: /assets/nsec2026/badge-journey/light-sensor-calibration.png -->
<p>
<a href="/assets/nsec2026/badge-journey/light-sensor.png" data-lightbox="image12"><img width="55%" height="55%" src="{{ '/assets/nsec2026/badge-journey/light-sensor.png' | relative_url }}"></a>
</p>

Alongside the main badge, the SAOs were designed by the badge team to add an interactive tie in to the Solarpunk theme. Preview renders arrived on April 12 with a planned physical journey as part of the soldering village activities of the conference.

<!-- IMAGE: /assets/nsec2026/badge-journey/sao-terrarium-render.png -->
<p>
<a href="/assets/nsec2026/badge-journey/sao1.png" data-lightbox="image13"><img width="65%" height="65%" src="{{ '/assets/nsec2026/badge-journey/sao1.png' | relative_url }}"></a>
</p>

<!-- IMAGE: /assets/nsec2026/badge-journey/sao-sun-render.png -->
<p>
<a href="/assets/nsec2026/badge-journey/sao2.png" data-lightbox="image14"><img width="65%" height="65%" src="{{ '/assets/nsec2026/badge-journey/sao2.png' | relative_url }}"></a>
</p>

The SAOs were ordered on April 20, leaving little slack before the conference. However little did we know this wasn't going to be the biggest stressor regarding delivery timelines. 

Supply-chain pressure was already beginning to show. The batteries arrived on April 14, resolving one dependency just as stock of the validated e-ink screen disappeared. We had proven the display only weeks earlier, but validation does not reserve vendor inventory. A component can work perfectly and still become a critical risk when it is unavailable in the quantity required. The team worked the problem as it had every other complication, adapting the plan. Still, it was the supply-chain equivalent of two steps forward and one step back, and the room to recover was shrinking.

At the same time, we had to prepare for scale. By April 19 the multi-flasher script was running. Flashing one prototype by hand is development; flashing an entire production run requires repeatability, parallelism, useful error handling, and a workflow that many people can follow. The script was as much a part of the production system as the docks, fixtures, and cables. This was an improvement on the 2025 development process, and honestly is one of the  reasons we were able to recover the way we did on delivery.

```
python tools/flash.py --mode dual --bin-dir release/dual --all
=== NorthSec Badge 2026 Flasher ===
Mode:      dual
Bin dir:   ..\badge-2026\release\dual       
Files to flash:
  0x0  bootloader.bin  (14.8 KB)
  0x8000  partitions.bin  (3.0 KB)
  0x10000  badge-conference.bin  (414.9 KB)
  0x150000  badge-ctf.bin  (440.8 KB)

Flashing 2 badges in parallel (dual mode)...
  [1/2] COM11
  [2/2] COM4

============================================================
Results (7.1s):
============================================================
  [+] OK  COM11
  [+] OK  COM4

Total: 2 succeeded, 0 failed out of 2
```

Firmware reached version 1.0.0 on April 25. Development did not stop, but the team now had a stable production image and a known baseline for final integration. The SAOs arrived two days later, and on April 28 the production badges shipped. It felt as though months of parallel effort were finally converging: hardware was in transit, add-ons were in hand, firmware was tagged, and the multi-flasher was ready.

For the first time, the remaining path looked almost straightforward. Or so we thought...

<h1>May - From Confidence to Panic</h1>

While the production boards travelled, the team continued testing with Prototype V2. On May 3, the dock code was exercised with multiple prototype badges, validating not only the software but the physical workflow we expected to use once the shipment arrived.

Then, on May 7, the first signs of trouble appeared. A shipping invoice needed correction. By May 11, we were navigating CARM, duties, and the paperwork required to release the shipment. What should have been a routine final delivery became a chain of administrative blockers at exactly the wrong time. The boards were manufactured and close enough to track, but not close enough to flash. Every day spent resolving paperwork was another day removed from assembly and testing.

At first, there was room for cautious optimism. The team kept working through the process, providing documents, following up, and preparing everything that could be prepared without the production hardware. Then the dates became impossible to ignore. By May 13, NSEC was beginning and the badges still had not arrived. Remember, the conference kick off was the next day, May 14th, and we didn't physically have the badges, nor were they assembled, nor have the firmware flashed. It was chaos.

That was when concern became panic. Our NSEC president escalated the issue as far as the COO and CEO of DHL Canada while, in parallel, the team decided to work through contingencies based on previous years' badges. The question became did we have enough badges from the previous year to give one per CTF team, and could we retrofit the challenge components of the firmware into these older badges. Surprisingly, I was able to make the firmwares work, but thankfully that was ultimately not required.

Months of engineering had succeeded, but the entire attendee experience was at risk because the completed boards were beyond our reach.

A new estimate promised delivery on May 14, the day of the conference kick-off. It was encouraging, but by then an estimate was not something we could build the conference around. We had to be ready for both outcomes, the new badges arriving at the last possible moment, or not arriving at all.

<h1>The Conference Catches Up with the Badges</h1>

<!-- IMAGE: /assets/nsec2026/badge-journey/opening-ceremony.jpg -->
<p>
<a href="/assets/nsec2026/badge-journey/intro1.jpg" data-lightbox="intro1"><img width="100%" height="100%" src="{{ '/assets/nsec2026/badge-journey/intro1.jpg' | relative_url }}"></a>
</p>


The opening ceremony included a presentation about the badges even though the production boards were not yet there to hand out. It was an appropriately surreal moment. I stepped on to the stage and could explain the design, demonstrate the ideas, and describe the months of work behind them, but the objects themselves remained somewhere between customs, delivery, and the venue. The conference opened while we waited for the physical ending of the story to catch up.

<!-- IMAGE: /assets/nsec2026/badge-journey/flashing-party.jpg -->
<p>
<a href="/assets/nsec2026/badge-journey/delivery1.jpg" data-lightbox="delivery1"><img width="100%" height="100%" src="{{ '/assets/nsec2026/badge-journey/delivery1.jpg' | relative_url }}"></a>
</p>

The shipment finally reached Marche Bonsecours a few hours after I introduced them to the crowd. The boxes showing up brought an enormous sense of relief, followed almost immediately by the realization that arrival was not the finish line. The boards still had to be assembled, flashed, validated, and prepared for attendees. Months of carefully scheduled work had collapsed into the opening hours of NSEC.

Once the badges reached us, the development project transformed into a flashing and assembly party in the middle of the conference. Laptops, cables, docks, components, and boxes took over the room. The group of NSEC volunteers got together and made it a reality.

<!-- IMAGE: /assets/nsec2026/badge-journey/flashing-party.jpg -->
<p>
<a href="/assets/nsec2026/badge-journey/prodline1.jpg" data-lightbox="prod1"><img width="100%" height="100%" src="{{ '/assets/nsec2026/badge-journey/prodline1.jpg' | relative_url }}"></a>
</p>

<!-- IMAGE: /assets/nsec2026/badge-journey/flashing-party.jpg -->
<p>
<a href="/assets/nsec2026/badge-journey/prodline2.jpg" data-lightbox="prod2"><img width="100%" height="100%" src="{{ '/assets/nsec2026/badge-journey/prodline2.jpg' | relative_url }}"></a>
</p>

<!-- IMAGE: /assets/nsec2026/badge-journey/flashing-party.jpg -->
<p>
<a href="/assets/nsec2026/badge-journey/prodline3.jpg" data-lightbox="prod3"><img width="100%" height="100%" src="{{ '/assets/nsec2026/badge-journey/prodline3.jpg' | relative_url }}"></a>
</p>

More than any individual technical milestone, this was the part of the project that showed how good the badge team, and overall NSEC volunteer community was. People banded together without worrying about whose task something had originally been. The room became an improvised production line, and the conference community met the delay with patience and encouragement. What could have been remembered only as a shipping failure instead became a shared effort and, ultimately, a massive success. Ultimately we managed to get production, conference-ready badges, in attendee hands a mere few hours delayed from the kick-off, and the overall sentiment from individuals was positive. 

On top of it all, we had a stellar defect rate this year. Coming off of last year's ~11% defect rate (stemming from the LED screen issues), we were happy to see only 14 badges defective, for an overall defect rate of 0.9%!

The scene was chaotic, technical, collaborative, and perhaps the most honest conclusion possible. Seven months earlier, the badge had been sketches and conversations. Now the team was sitting together at NSEC, assembling, flashing, and hadning out real hardware while the event unfolded around us. The badges made it to attendees, the features we had spent months building came to life, and the project became part of the conference experience we had imagined back in October.

<h1>Summary</h1>

Building the NSEC 2026 badge was a large undertaking from the beginning. Between October and May, the team took it through several visual concepts, two hardware prototypes, production, and a broad firmware stack covering memory, NFC, peer-to-peer communication, Wi-Fi, e-ink, sensing, docking, and add-ons. Around that technical work sat procurement, manufacturing, testing infrastructure, documentation, logistics, and all the coordination required to make those efforts converge.

There were complications throughout, but the project kept moving because the team kept adapting. The final shipping crisis pushed that ability to its limit. Paperwork and customs held the completed boards until NSEC had already begun, turning the final days into a mixture of escalation, contingency planning, cautious hope, and urgent on-site production.

It was not the ending shown on our original timeline, but it may have been the ending that best represented the project. The badge succeeded because a remarkable team stayed collaborative under pressure, trusted the preparation already completed, and kept solving the next problem until the boards reached attendees' hands.

Finally, I'd love to share the actualy ballad of the badge that one of the team members generate, capturing our overall struggles, journey, and triumph. 

{% include embed/audio.html
  src="/assets/nsec2026/badge-journey/ballad-for-a-badge-v1.mp3"
  title="Ballad for a Badge - NSEC 2026 Edition"
%}

<h1>Looking Ahead to NSEC 2027</h1>

If anyone is interested in the code, schematics, and lower level details - we open-sourced all the code for the 2026 edition in [NSEC Badge 2026 repo](https://github.com/nsec/badge-2026).

Useful outcomes of a project like this is not only the finished code and hardware. It is also the set of lessons that can make the next project better. We now have direct experience with the places where the 2026 schedule held, where it had too little margin, which technical workflows scaled well, and which procurement and shipping assumptions need to change.

Planning for NSEC 2027 is already beginning! The goal is to build on what we have learned over the last few years, take some of the more administrative improvements we learned from 2026, and see how the team can elevate themselves yet again for an amazing conference experience. It's a tall order, but it's also an amazing team.

There will be more to share as those plans take shape. For now, the 2026 journey gives us both something to celebrate and a valuable engineering baseline for what comes next.

Thanks folks, until next time!