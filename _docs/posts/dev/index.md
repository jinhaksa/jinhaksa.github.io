---
title: Dev
description: 개발 포스팅
tags:
  - dev
---

# 개발

<ul>
  {% assign devPosts = site.categories.dev | sort: "date" | reverse %}
  {% for post in devPosts %}
      <li>
        <a href="{{ post.url }}">{{ post.title }}</a>{% if post.author %} | {{ post.author }} {% endif %} | {{ post.date | date: "%Y/%m/%e" }}
      </li>    
  {% endfor %}
</ul>
