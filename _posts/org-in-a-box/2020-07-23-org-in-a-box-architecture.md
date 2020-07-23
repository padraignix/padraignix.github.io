---
layout:     post
title:      Org-in-a-Box - Project Architecture - padraignix.github.io
title2:     Org-in-a-Box - Architecture
date:       2020-7-23 10:00:00 -0400
summary:    Initial post covering my personal project of an "organization in a box". Using fundamental open source Identity and Access Management components of an organization in a self-contained box to explore and expand knowledge of those various components. This article will cover the overall architecture and help establish the high level plan moving forward.
categories: org-in-a-box
thumbnail:  cogs
keywords:   org-in-a-box,identity management,access management,IAM,architecture,organizational planning,personal project,raspberry pi,pi
thumbnail:  https://padraignix.github.io/assets/org-in-a-box/infocard-architecture.png
canon:      https://padraignix.github.io/org-in-a-box/2020/07/23/org-in-a-box-architecture/
tags:
 - org-in-a-box
 - architecture
 - IAM
 - raspberry pi
 - project
---

<p align="center">
<img width="70%" height="70%" src="{{ '/assets/org-in-a-box/infocard-architecture.png' | relative_url }}">
</p>

<h1>Introduction & Plans</h1>

<p>Years ago when I first started working with Identity and Access Management technology it was not originally by choice. I started my InfoSec career as a founding member of a local security team and we had to spread coverage across several security domains. As a result of all other founding members being more interested in network security I ended up being the main individual on the team focused on IAM components. Over the years I kept learning and growing in the role gaining an appreciation for IAM as a business enabler, cost reducer, and an underlying core component of most security and regulatory processes.</p>

<p>As an effort to further my overall knowledge I have worked with various parts of IAM technology in my home lab. From installing OpenLDAP to explore schema creation, instance replication and general data structures, to standing up an MIT Kerberos instance to learn more about authentication flows over the wire, these experiments were always done in a standalone approach. While they were good in isolation for learning the specific technologies, there were missed opportunities to expand that knowledge from an overall infrastructure perspective.</p>

<p>Over the last few months I have been toying with the idea of furthering these technology experiments into a single, functioning infrastructure that could emulate a small company or entity. In addition to expanding my knowledge of IAM component interoperability I also wanted to have fun with the concept. The overall structure would be limited to a single "box" and instead of the usual virtual server x86 architecture solution, I wanted to see if I could make this work on an arm based solution - Raspberry PIs. The initial hardware was acquired and the idea of "Org-in-a-Box" was born!</p>

<p align="center">
<img width="80%" height="80%" src="{{ '/assets/org-in-a-box/physical_hardware_02.jpg' | relative_url }}">
</p>

<h1>So Why Do This?</h1>

<p>If I had to boil it down to a single word - learning. There is no underlying business driver or particular problem I am trying to solve in my home lab. That freedom is both a pro and a con as without a definitive end-goal I need to be very clear with my objectives otherwise this may become an endless loop through scope creep. That in itself is not necessarily a bad thing, however I would rather define a specific objective, complete it, review the implementation, evaluate, and create a new scope moving forward. You know, proper process and such...</p>

<p>As has become common with my personal projects there is one additional reason for doing this - I expect to fail. It may sound weird however I fully expect to fail implementing components and the overall vision... at first. I tend to have a slightly sadistic nature where I will not walk away from the computer for essential activities, like sleeping, until I reach a logical stopping point where it makes sense and "clicks". This approach while not necessarily the healthiest in longer runs, allows me to struggle against a particular problem and learn the technology in-depth. With this project I am establishing an initial set of acceptance criteria that I fully expect to evolve as I work through the independent parts, struggle, learn, fail, and eventually solve the specific problem. As much as it can be frustrating that "eureka" moment of seeing those first successful logs is a thing of beauty and worth every second.</p>

<h1>Goals / Acceptance Criteria</h1>

<p>I originally listed out this section with fleshed out details of "how". In the interest of attempting to keep it simple and able to evolve as I run into failure I pulled back from that approach and focused more on the "what", leaving the specific implementation changeable.</p>

*  Centralized authentication
*  Centralized identity management
*  Proper logging system and dashboards
*  Proper monitoring and alerting
*  Some end user functionality platforms

<h1>Initial Architecture</h1>

<p>The next step in the journey was to start putting an architecture plan together. I spent quite a bit of time reading various administrator guides, technical manuals, and other security blogs doing research on what could feasibly work and if folks had attempted various combinations with or without any complications along the way.</p>

<p><u>Network Architecture</u></p>

<p align="center">
<img src="{{ '/assets/org-in-a-box/org-architecture-00.jpg' | relative_url }}">
</p>

<p><u>Process Plan</u></p>

<p align="center">
<img src="{{ '/assets/org-in-a-box/org-architecture-01.jpg' | relative_url }}">
</p>

<p><u>Logging Flow</u></p>

<p align="center">
<img src="{{ '/assets/org-in-a-box/org-architecture-02.jpg' | relative_url }}">
</p>

<h1>Standalone Proof-of-Concept</h1>

<p>I'll leave the complete installation and configuration of the Kerberos and LDAP instances to the next part of this series, however I wanted to add a sneak peak of standing up a POC test MIT Kerberos instance and configuring an Open LDAP instance to leverage Kerberos authentication.</p>

<h2>MIT Kerberos</h2>

<p>I set the initial Kerberos configuration to create the initial home directory and request Kerberos tickets on successful login.</p>

<p align="center">
<img src="{{ '/assets/org-in-a-box/architecture-kerberos-login.png' | relative_url }}">
</p>

<p>I was then able to successfully authenticate and acquire tickets from my Windows machine. The corresponding logs on the server are also included at the bottom of the request.</p>

<p align="center">
<img src="{{ '/assets/org-in-a-box/architecture-kerberos-windows.png' | relative_url }}">
</p>

<h2>OpenLDAP</h2>

<p>As a final preview let's quickly show OpenLDAP Kerberos authentication. Similar to above I'll leave the OpenLDAP installation and configuration for the next post.</p>

<p>While I'll skip over how it was created for this post, here is a the result of querying my People record using an anonymous bind to the server.</p>

<p align="center">
<img src="{{ '/assets/org-in-a-box/architecture-ldap-anonymous.png' | relative_url }}">
</p>

<p>Now let us demonstrate a successful GSSAPI bind using ldapwhoami followed by repeating the query above, this time using Kerberos authentication instead of anonymous access.</p>

<p align="center">
<img src="{{ '/assets/org-in-a-box/architecture-ldap-query.png' | relative_url }}">
</p>

<p align="center">
<img src="{{ '/assets/org-in-a-box/architecture-ldap-query2.png' | relative_url }}">
</p>

<h1>Summary</h1>

<p>I hope people reading this have had their interest peaked and follow along as I struggle through building up this environment. I will do my best to cover the installation, configuration, and interoperability between components, all while covering complications I run into throughout the process.</p>

<p>As always thanks everyone, until next time!</p>