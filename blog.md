---
layout: default
title: Blog 
---

# Blog

Writing about blockchain security, machine learning, data engineering, and software development.

<div class="blog-list">
  {% for post in site.posts %}
    <article class="blog-list-item">
      <h2 class="blog-list-title">
        <a href="{{ post.url }}">{{ post.title }}</a>
      </h2>
      <div class="blog-list-meta">
        <time datetime="{{ post.date | date_to_xmlschema }}">
          {{ post.date | date: "%B %-d, %Y" }}
        </time>
        {% if post.categories %}
          <span class="blog-list-separator">•</span>
          <span class="blog-list-categories">
            {% for category in post.categories %}
              {{ category }}{% unless forloop.last %}, {% endunless %}
            {% endfor %}
          </span>
        {% endif %}
      </div>
      {% if post.excerpt %}
        <div class="blog-list-excerpt">
          {{ post.excerpt | strip_html | truncatewords: 40 }}
        </div>
      {% endif %}
    </article>
  {% endfor %}
</div>
