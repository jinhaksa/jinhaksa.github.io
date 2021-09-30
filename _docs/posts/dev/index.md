---
title: Dev
description: 개발 포스팅
tags:
  - dev
---

# 개발

<ul>
  {% for post in site.categories.dev %}
      <li>
        <a href="{{ post.url }}">{{ post.title }}</a>{% if post.author %} | {{ post.author }} {% endif %} | {{ post.date | date: "%Y/%m/%e" }}
      </li>    
  {% endfor %}
</ul>
