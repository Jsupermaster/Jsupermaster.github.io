---
layout: default
nav: tags
title: 标签
description: 浏览博客中的全部标签。
permalink: /tags/index.html
---
{% assign sorted_tags = site.tags | sort %}

<section class="archive-header">
  <p class="section-kicker">标签</p>
  <h1>标签</h1>
  <p class="archive-copy">全部标签会根据现有文章自动归类，点击标签可查看相关文章。</p>
</section>

<section class="panel-block">
  <div class="panel-heading">
    <h3>全部标签</h3>
    <span>{{ sorted_tags.size }} 个标签</span>
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
  <p class="feed-excerpt">当前还没有检测到标签。只要文章的 front matter 中包含 <code>tags</code> 字段，这里就会自动显示。</p>
  {% endif %}
</section>
