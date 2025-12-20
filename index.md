---
layout: default
title: Home
---

<div class="home-intro">
  <h1>Hi, I'm Cyprian 👋</h1>
  <p class="home-subtitle">Software developer and researcher exploring the intersection of blockchain security, machine learning, and data engineering.</p>
</div>

## What I Write About

<div class="topics-grid">
  <div class="topic-card">
    <h3>🔒 Smart Contract Security</h3>
    <p>Vulnerability detection, Solidity analysis, and blockchain security research</p>
  </div>
  
  <div class="topic-card">
    <h3>🧠 Machine Learning</h3>
    <p>Graph neural networks, deep learning, and AI applications in code analysis</p>
  </div>
  
  <div class="topic-card">
    <h3>⚙️ Data Engineering</h3>
    <p>Building ETL pipelines, data processing, and business intelligence</p>
  </div>
  
  <div class="topic-card">
    <h3>💻 Software Development</h3>
    <p>Full-stack development, hackathon projects, and technical experiments</p>
  </div>
</div>

## Recent Posts

<div class="recent-posts">
  {% for post in site.posts limit:5 %}
    <div class="recent-post-item">
      <h3 class="recent-post-title">
        <a href="{{ post.url }}">{{ post.title }}</a>
      </h3>
      <div class="recent-post-meta">
        {{ post.date | date: "%B %-d, %Y" }}
      </div>
    </div>
  {% endfor %}
</div>

<div class="home-cta">
  <a href="/blog.html" class="cta-button">View All Posts →</a>
</div>

## Connect

<div class="connect-links">
  <a href="https://github.com/CDNamchu" class="connect-link">GitHub</a>
  <span class="connect-separator">•</span>
  <a href="https://www.linkedin.com/in/Cyprian%20Dyenree%20Namchu" class="connect-link">LinkedIn</a>
  <span class="connect-separator">•</span>
  <a href="mailto:cypriandnamchu@gmail.com" class="connect-link">Email</a>
</div>
