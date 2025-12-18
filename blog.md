---
layout: default
title: Blog 
---

# Blog

Writing about blockchain security, machine learning, data engineering, and software development.

<div>
  <ul>
    {% for post in site.posts %}
      <li><a href="{{ post.url }}">{{ post.title }}</a></li>
    {% endfor %}
  </ul>
</div>
