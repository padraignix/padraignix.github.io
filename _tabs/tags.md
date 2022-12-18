---
title: Tags
title2: Padraignix's InfoSec Blog - Tags
type: tags
summary: Personal blog covering CTF events, Security Infrastructure, Cryptography, Emulator development, Quantum Technology and related adventures
keywords: hacking,quantumly confused,blog,information security,infosec,hackthebox,quantum computing,quantum technology,emulation,emulators,reverse engineering
thumbnail:  https://github.com/padraignix.png
canon:      https://blog.quantumlyconfused.com/tabs/tags/
---

{% comment %}
  'site.tags' looks like a Map, e.g. site.tags.MyTag.[ Post0, Post1, ... ]
  Print the {{ site.tags }} will help you to understand it.
{% endcomment %}
<div id="tags" class="d-flex flex-wrap ml-xl-2 mr-xl-2">
{% assign tags = "" | split: "" %}
{% for t in site.tags %}
  {% assign tags = tags | push: t[0] %}
{% endfor %}

{% assign sorted_tags = tags | sort_natural %}

{% for t in sorted_tags %}
  <div>
    <a class="tag" href="{{ site.baseurl }}/tags/{{ t | replace: ' ', '-' | downcase | url_encode }}/">{{ t }}<span class="text-muted">{{ site.tags[t].size }}</span></a>
  </div>
{% endfor %}

</div>
