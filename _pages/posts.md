---
layout: page
title: Blog Posts
permalink: /posts/
nav_order: 3
---

## Financial Engineering

<ul>
  {% for post in site.posts %}
    <li>
      <a href="{{ post.url }}">{{ post.title }}</a>
    </li>
  {% endfor %}
</ul>