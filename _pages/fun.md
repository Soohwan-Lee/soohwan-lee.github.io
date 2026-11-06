---
layout: archive
permalink: /fun/
title: "Fun"
author_profile: true
---

Side projects and toys: things I built because I was curious, outside of papers.

<!-- Content lives in _data/fun.yml; cards reuse _includes/project-card.html. -->

<div class="proj-page">
<div class="proj-grid">
{%- for item in site.data.fun.toys %}
{% include project-card.html card=item %}
{%- endfor %}
</div>
</div>
