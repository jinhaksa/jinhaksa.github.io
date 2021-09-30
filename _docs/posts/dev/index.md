---
title: Dev
description: 개발 포스팅
tags:
  - dev
---

<ul>
  {% for post in site.posts %}
      <li>
        <a href="{{ post.url }}">{{ post.title }}</a>
      </li>    
  {% endfor %}
</ul>
