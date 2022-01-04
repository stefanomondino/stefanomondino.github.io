---
layout: archive
title: "Public Speaking"
permalink: /speaking/
collection: speaking
author_profile: false
header:
    image: /assets/images/droidcon_pano.jpeg
---

<div class="grid__wrapper">
  {% for post in site.speaking %}
    {% include archive-single.html type="grid" %}
  {% endfor %}
</div>