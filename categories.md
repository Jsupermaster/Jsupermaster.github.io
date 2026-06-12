---
layout: default
nav: categories
title: 合集
description: 浏览围绕同一主题持续整理的系列内容。
permalink: /categories/index.html
collections:
  - title: PIM论文阅读
    slug: pim-paper-reading
    description: 聚合 PIM/PNM 方向的论文精读、综述整理与趋势观察。
    accent_class: collection-pim
  - title: FPGA 8051软核处理器设计实战
    slug: fpga-8051-softcore-design
    description: 记录 8051 软核处理器从基础理解到 Verilog 设计与验证的完整过程。
    accent_class: collection-fpga
  - title: Gem5全系统仿真运行实录
    slug: gem5-full-system-notes
    description: 覆盖简易 SoC 搭建、Linux 启动与系统级调试的 Gem5 实践记录。
    accent_class: collection-gem5
  - title: 内存与系统
    slug: memory-and-systems
    description: 聚焦 DRAM、系统集成与内存层次相关的学习整理。
    accent_class: collection-memory
---
<section class="archive-header">
  <p class="section-kicker">COLLECTIONS</p>
  <h1>合集</h1>
  <p class="archive-copy">把围绕同一条主线持续更新的文章组织在一起，方便成体系地阅读。</p>
</section>

<section class="panel-block">
  <div class="panel-heading">
    <h3>全部合集</h3>
    <span>{{ page.collections | size }} 个主题系列</span>
  </div>
  <div class="collection-grid">
    {% for collection in page.collections %}
    {% assign collection_posts = site.posts | where: "series", collection.slug %}
    <article class="collection-card {{ collection.accent_class }}">
      <div class="collection-card-head">
        <strong><a class="collection-title-link" href="#{{ collection.slug }}">{{ collection.title }}</a></strong>
        <span>{{ collection_posts | size }} 篇文章</span>
      </div>
      <p>{{ collection.description }}</p>
      <div id="{{ collection.slug }}" class="collection-list">
        {% for post in collection_posts %}
        <a class="collection-post-link" href="{{ post.url | relative_url }}">
          <strong>{{ post.title }}</strong>
          <span>{{ post.date | date: "%Y-%m-%d" }}</span>
        </a>
        {% endfor %}
        {% if collection_posts.size == 0 %}
        <div class="collection-empty">这个合集暂时还没有匹配到文章。</div>
        {% endif %}
      </div>
    </article>
    {% endfor %}
  </div>
</section>
