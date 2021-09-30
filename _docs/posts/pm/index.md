---
title: 기획
permalink: /pm/
description: 기획 포스팅
tags:
  - pm
---

<!-- @format -->

# 기획

<ul>
  {% assign devPosts = site.categories.pm | sort: "date" | reverse %}
  {% for post in devPosts %}
      <li>
        <a href="{{ post.url }}">{{ post.title }}</a>{% if post.author %} | {{ post.author }} {% endif %} | {{ post.date | date: "%Y/%m/%e" }}
      </li>    
  {% endfor %}
</ul>
