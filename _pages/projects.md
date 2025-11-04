---
layout: page
title: projects
permalink: /projects/
description: A growing collection of your cool projects.
nav: true
nav_order: 3
---

{% assign sorted_projects = site.projects | sort: "importance" %}
<div class="projects">
  {% include projects.liquid projects=sorted_projects limit=nil %}
</div>