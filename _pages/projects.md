---
layout: page
title: projects
permalink: /projects/
description: Collection of my past and ongoing projects.
nav: true
nav_order: 3
---

{% assign sorted_projects = site.projects | sort: "importance" %}
<div class="projects">
  {% include projects.liquid projects=sorted_projects limit=nil %}
</div>