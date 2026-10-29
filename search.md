---
layout: default
nav: search
title: 搜索
description: 搜索博客中的文章和页面。
permalink: /search/index.html
---
<section class="archive-header">
  <p class="section-kicker">搜索</p>
  <h1>站内搜索</h1>
  <p class="archive-copy">搜索博客中的文章与重要页面。</p>
</section>

<section class="panel-block search-panel">
  <div class="panel-heading">
    <h3>搜索内容</h3>
    <span>文章与页面</span>
  </div>
  <label class="search-label" for="search-input">搜索标题、标签、分类或摘要</label>
  <input id="search-input" class="search-input" type="search" placeholder="输入关键词..." autocomplete="off">
  <div id="search-results" class="search-results" role="status" aria-live="polite" aria-atomic="true"></div>
</section>

<script>
  (() => {
    const input = document.getElementById('search-input');
    const results = document.getElementById('search-results');
    if (!input || !results) return;
    let items = [];
    const render = (matches, query) => {
      if (!query) {
        results.innerHTML = '<p class="search-empty">请输入关键词开始搜索。</p>';
        return;
      }
      if (!matches.length) {
        results.innerHTML = '<p class="search-empty">未找到匹配内容。</p>';
        return;
      }
      results.innerHTML = matches.map(item => `
        <a class="search-result-card" href="${item.url}">
          <strong>${item.title}</strong>
          <span>${item.kind}</span>
          <p>${item.meta}</p>
        </a>
      `).join('');
    };
    results.innerHTML = '<p class="search-empty" role="status">正在加载搜索内容…</p>';
    fetch('{{ "/search.json" | relative_url }}')
      .then(res => {
        if (!res.ok) throw new Error('Search index request failed');
        return res.json();
      })
      .then(data => {
        items = data;
        render([], '');
      })
      .catch(() => {
        results.innerHTML = '<p class="search-empty" role="alert">搜索内容加载失败，请刷新页面后重试。</p>';
      });
    input.addEventListener('input', () => {
      const query = input.value.trim().toLowerCase();
      const matches = items.filter(item =>
        item.title.toLowerCase().includes(query) ||
        item.meta.toLowerCase().includes(query)
      );
      render(matches, query);
    });
  })();
</script>
