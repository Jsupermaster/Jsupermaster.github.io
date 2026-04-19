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
  <p class="archive-copy">Automatically grouped from all current posts. Click a tag to jump to its article list.</p>
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
    {% capture tag_id %}tag-{{ tag_name | replace: ' ', '-' | replace: '/', '-' | replace: '#', '' | replace: '.', '-' | replace: ',', '-' | downcase }}{% endcapture %}
    <a class="topic-card" href="#{{ tag_id }}">
      <strong>#{{ tag_name }}</strong>
      <span>{{ tag_posts | size }} 篇文章{% if tag_posts.first %} · 最新 {{ tag_posts.first.date | date: "%Y-%m-%d" }}{% endif %}</span>
    </a>
    {% endfor %}
  </div>
  {% else %}
  <p class="feed-excerpt">当前还没有检测到标签。只要文章 front matter 中带有 <code>tags</code> 字段，这里就会自动显示。</p>
  {% endif %}
</section>

<div class="tag-section-list">
  {% for tag in sorted_tags %}
  {% assign tag_name = tag[0] %}
  {% assign tag_posts = tag[1] %}
  {% capture tag_id %}tag-{{ tag_name | replace: ' ', '-' | replace: '/', '-' | replace: '#', '' | replace: '.', '-' | replace: ',', '-' | downcase }}{% endcapture %}
  <section class="panel-block tag-section" id="{{ tag_id }}">
    <div class="panel-heading">
      <h3>#{{ tag_name }}</h3>
      <span>{{ tag_posts | size }} 篇文章</span>
    </div>
    <div class="tag-post-list">
      {% for post in tag_posts %}
      <article class="feed-card">
        <div class="feed-meta">
          <span>{{ post.date | date: "%Y-%m-%d" }}</span>
          {% if post.categories.size > 0 %}
          <span>{{ post.categories | join: " / " }}</span>
          {% endif %}
        </div>
        <h2><a href="{{ post.url | relative_url }}">{{ post.title }}</a></h2>
        {% if post.cover_image %}
        <a class="feed-cover-link" href="{{ post.url | relative_url }}">
          <img class="feed-cover" src="{{ post.cover_image | relative_url }}" alt="{{ post.title }}">
        </a>
        {% endif %}
        <p class="feed-excerpt">{{ post.excerpt | strip_html | strip_newlines }}</p>
        <div class="feed-footer">
          <div class="chip-row">
            {% for item in post.tags %}
            {% capture item_id %}tag-{{ item | replace: ' ', '-' | replace: '/', '-' | replace: '#', '' | replace: '.', '-' | replace: ',', '-' | downcase }}{% endcapture %}
            <a class="chip" href="#{{ item_id }}">#{{ item }}</a>
            {% endfor %}
          </div>
          <a class="read-link" href="{{ post.url | relative_url }}">阅读全文</a>
        </div>
      </article>
      {% endfor %}
    </div>
  </section>
  {% endfor %}
</div>
