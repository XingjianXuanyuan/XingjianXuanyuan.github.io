---
layout: home
title: "Home"
nav_order: 1
---
# Xingjian Xuanyuan
譬大道之在天下猶川谷之於江海

<div class="posts">
  {% for post in site.posts %}
    <article class="post">

      <h1><a href="{{ site.baseurl }}{{ post.url }}">{{ post.title }}</a></h1>

      <div class="entry">
        {{ Tao is to the world what rivers are to the ocean. }}
      </div>

      <a href="{{ site.baseurl }}{{ post.url }}" class="read-more">Read More</a>
    </article>
  {% endfor %}
</div>
