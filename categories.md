---
layout: default
nav: categories
title: 分类
description: 浏览博客的主要主题方向。
permalink: /categories/index.html
topics:
  - title: 最新工作
    description: 我最近的论文、项目与开源工作。
    href: /blog/Latest_Work/index.html
  - title: 计算机体系结构
    description: 体系结构基础、理解与工具链实践。
    href: /blog/Computer_Architecture/index.html
  - title: PIM/PNM
    description: 近存计算相关论文、工具链与架构设计。
    href: /blog/PIM/index.html
  - title: AI 加速器
    description: AI 加速器的最新进展与开源工作。
    href: /blog/AI_Accelerator/index.html
---
<section class="archive-header">
  <p class="section-kicker">分类</p>
  <h1>分类</h1>
  <p class="archive-copy">浏览这个博客当前覆盖的主要主题方向。</p>
</section>

<section class="panel-block">
  <div class="panel-heading">
    <h3>当前方向</h3>
    <span>主要主题</span>
  </div>
  <div class="topic-grid">
    {% for topic in page.topics %}
    <a class="topic-card" href="{{ topic.href | relative_url }}">
      <strong>{{ topic.title }}</strong>
      <span>{{ topic.description }}</span>
    </a>
    {% endfor %}
  </div>
</section>
