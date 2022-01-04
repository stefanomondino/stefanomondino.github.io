---
layout: archive
title: "Posts"
permalink: /posts/
collection: posts
author_profile: false
---

<div class="grid__wrapper">
  {% for post in site.posts %}
    {% include archive-single.html %}
  {% endfor %}
</div>