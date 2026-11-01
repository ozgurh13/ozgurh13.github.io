---
layout: page
title: Projects
permalink: /projects/
---

Here are some of the projects I have built:

<ul>
  {% for project in site.projects %}
    <li>
      <a href="{{ project.url | relative_url }}">{{ project.title }}</a>
      <br> {{ project.description }}
    </li>
  {% endfor %}
</ul>
