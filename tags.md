---
layout: default
nav: tags
title: Tags
description: Browse all tags used across the blog.
permalink: /tags/index.html
---
{% assign sorted_tags = site.tags | sort %}

<section class="archive-header">
  <p class="section-kicker">TAGS</p>
  <h1>Tags</h1>
  <p class="archive-copy">Automatically grouped from all current posts. Click a tag to open its article list.</p>
</section>

<section class="panel-block">
  <div class="panel-heading">
    <h3>All Tags</h3>
    <span>{{ sorted_tags.size }} Labels</span>
  </div>
  {% if sorted_tags.size > 0 %}
  <div class="topic-grid">
    {% for tag in sorted_tags %}
    {% assign tag_name = tag[0] %}
    {% assign tag_posts = tag[1] %}
    {% capture tag_path %}/tags/result.html?tag={{ tag_name | uri_escape }}{% endcapture %}
    <a class="topic-card" href="{{ tag_path | relative_url }}">
      <strong>#{{ tag_name }}</strong>
      <span>{{ tag_posts | size }} 篇文章{% if tag_posts.first %} · 最新 {{ tag_posts.first.date | date: "%Y-%m-%d" }}{% endif %}</span>
    </a>
    {% endfor %}
  </div>
  {% else %}
  <p class="feed-excerpt">当前还没有检测到标签。只要文章 front matter 中带有 <code>tags</code> 字段，这里就会自动显示。</p>
  {% endif %}
</section>
