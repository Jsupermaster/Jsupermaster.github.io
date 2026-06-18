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
  - title: 敏捷硬件开发语言Chisel与数字系统设计
    slug: chisel-and-scala-study
    description: 从零开始学习Chisel所需的Scala编程基础，覆盖函数式编程、面向对象、集合、模式匹配、泛型与隐式转换等核心主题。
    accent_class: collection-fpga
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
    <article class="collection-card {{ collection.accent_class }}" data-collection-column="{% cycle 'left', 'right' %}">
      <button
        class="collection-toggle"
        type="button"
        aria-expanded="false"
        data-collection-title="{{ collection.title | escape }}"
        data-collection-description="{{ collection.description | escape }}"
        data-collection-count="{{ collection_posts | size }}"
        data-collection-accent="{{ collection.accent_class }}"
        data-collection-side="{% cycle 'right', 'left' %}"
      >
        <div class="collection-card-head">
          <strong>{{ collection.title }}</strong>
          <span>{{ collection_posts | size }} 篇文章</span>
        </div>
        <p>{{ collection.description }}</p>
      </button>
      <template>
        {% if collection_posts.size > 0 %}
          {% for post in collection_posts %}
          <a class="collection-post-link" href="{{ post.url | relative_url }}">
            <strong>{{ post.title }}</strong>
            <span>{{ post.date | date: "%Y-%m-%d" }}</span>
          </a>
          {% endfor %}
        {% else %}
          <div class="collection-empty">这个合集暂时还没有匹配到文章。</div>
        {% endif %}
      </template>
    </article>
    {% endfor %}
  </div>
</section>

<div id="collection-overlay" class="collection-overlay" hidden>
  <button class="collection-overlay-backdrop" type="button" aria-label="关闭合集目录"></button>
  <section id="collection-panel" class="collection-panel" aria-hidden="true">
    <div class="collection-panel-inner">
      <div class="collection-panel-header">
        <div class="collection-panel-heading">
          <p class="section-kicker collection-panel-kicker">COLLECTION</p>
          <h2 id="collection-panel-title"></h2>
          <p id="collection-panel-description" class="collection-panel-copy"></p>
        </div>
        <div class="collection-panel-actions">
          <button id="collection-panel-close" class="collection-panel-close" type="button" aria-label="关闭合集目录">×</button>
        </div>
      </div>
      <div id="collection-panel-list" class="collection-panel-list"></div>
    </div>
  </section>
</div>

<script>
  (() => {
    const toggles = Array.from(document.querySelectorAll('.collection-toggle'));
    const overlay = document.getElementById('collection-overlay');
    const panel = document.getElementById('collection-panel');
    const title = document.getElementById('collection-panel-title');
    const description = document.getElementById('collection-panel-description');
    const list = document.getElementById('collection-panel-list');
    const closeButton = document.getElementById('collection-panel-close');
    const backdrop = document.querySelector('.collection-overlay-backdrop');
    if (!toggles.length || !overlay || !panel || !title || !description || !list || !closeButton || !backdrop) return;

    let activeToggle = null;

    const closePanel = () => {
      overlay.classList.remove('is-open');
      panel.classList.remove('is-open');
      panel.setAttribute('aria-hidden', 'true');
      toggles.forEach((toggle) => toggle.setAttribute('aria-expanded', 'false'));
      document.body.classList.remove('collection-open');
      activeToggle = null;

      window.setTimeout(() => {
        if (!overlay.classList.contains('is-open')) {
          panel.classList.remove('panel-left', 'panel-right', 'collection-pim', 'collection-fpga', 'collection-gem5', 'collection-memory');
          overlay.hidden = true;
          list.innerHTML = '';
        }
      }, 260);
    };

    const openPanel = (toggle) => {
      const template = toggle.parentElement.querySelector('template');
      if (!template) return;

      activeToggle = toggle;
      toggles.forEach((item) => item.setAttribute('aria-expanded', item === toggle ? 'true' : 'false'));

      title.textContent = toggle.dataset.collectionTitle || '';
      description.textContent = toggle.dataset.collectionDescription || '';
      list.innerHTML = template.innerHTML;

      panel.className = 'collection-panel';
      panel.classList.add(toggle.dataset.collectionAccent || '', toggle.dataset.collectionSide === 'left' ? 'panel-left' : 'panel-right');

      overlay.hidden = false;
      requestAnimationFrame(() => {
        document.body.classList.add('collection-open');
        overlay.classList.add('is-open');
        panel.classList.add('is-open');
        panel.setAttribute('aria-hidden', 'false');
      });
    };

    toggles.forEach((toggle) => {
      toggle.addEventListener('click', () => {
        if (activeToggle === toggle && overlay.classList.contains('is-open')) {
          closePanel();
          return;
        }

        openPanel(toggle);
      });
    });

    closeButton.addEventListener('click', closePanel);
    backdrop.addEventListener('click', closePanel);

    document.addEventListener('keydown', (event) => {
      if (event.key === 'Escape' && overlay.classList.contains('is-open')) {
        closePanel();
      }
    });
  })();
</script>
