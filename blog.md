---
layout: default
title: "Notes — Vitalii Marchenko"
description: "Engineering notes on infrastructure, reliability, and automation."
permalink: /blog/
---

## 📝 Notes

Short write-ups on infrastructure, reliability, and the automation behind them.

<div class="post-list">
{% for post in site.posts %}
  <div class="post-item">
    <a class="post-title" href="{{ post.url | relative_url }}">{{ post.title }}</a>
    <div class="post-meta">
      <time datetime="{{ post.date | date_to_xmlschema }}">{{ post.date | date: "%d %b %Y" }}</time>
      {% if post.tags.size > 0 %}
        · {% for tag in post.tags %}<span class="post-tag">{{ tag }}</span>{% endfor %}
      {% endif %}
    </div>
    {% if post.description %}<p class="post-excerpt">{{ post.description }}</p>{% endif %}
  </div>
{% endfor %}
</div>

<p><a href="{{ '/' | relative_url }}">← Back to profile</a></p>
