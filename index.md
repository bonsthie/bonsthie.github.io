
---
layout: default
title: Home
---

# Welcome to Bonsthie's Blog!

This is a simple blog where I share my thoughts, ideas, and tutorials. Below are my latest posts:

<ul>
  {% for post in site.posts %}
    <li>
      <a href="{{ post.url }}">{{ post.title }}</a> - {{ post.date | date: "%B %d, %Y" }}
    </li>
  {% endfor %}
</ul>
