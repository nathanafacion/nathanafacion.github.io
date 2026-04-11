---
layout: default
title: Home
---

<h1 style="font-size: 2.2rem; animation: fadeUp 0.5s ease both;">
  <span style="background: linear-gradient(135deg, #a78bfa, #818cf8, #6366f1); -webkit-background-clip: text; -webkit-text-fill-color: transparent; background-clip: text;">Blog</span>
</h1>
<p style="color: var(--text-muted); margin-bottom: 2rem; animation: fadeUp 0.5s ease 0.05s both;">Pensamentos sobre frontend, código e tecnologia.</p>

<ul class="post-list">
{% for post in site.posts %}
  <li>
    <div class="post-card">
      {% if post.image %}
      <a href="{{ post.url }}" class="post-card-image-link">
        <img src="{{ post.image }}" alt="{{ post.title }}" class="post-card-image">
      </a>
      {% endif %}
      <a href="{{ post.url }}" class="post-card-title-link">
        <span class="post-card-title">{{ post.title }}</span>
      </a>
      <span class="post-card-date">{{ post.date | date: "%d/%m/%Y" }}</span>
      {% if post.description %}
      <p class="post-card-description">{{ post.description }}</p>
      {% endif %}
      {% if post.tags.size > 0 %}
      <div class="tags">
        {% for tag in post.tags %}
        <span class="tag">{{ tag }}</span>
        {% endfor %}
      </div>
      {% endif %}
      <a href="{{ post.url }}" class="post-card-readmore">Veja mais →</a>
    </div>
  </li>
{% endfor %}
</ul>
