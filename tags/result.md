---
layout: default
nav: tags
title: 标签结果
description: 按标签浏览文章。
permalink: /tags/result.html
---
<section class="archive-header">
  <p class="section-kicker">标签</p>
  <h1 id="tag-result-title">标签结果</h1>
  <p id="tag-result-copy" class="archive-copy">正在加载所选标签的文章...</p>
</section>

<section class="panel-block search-panel">
  <div class="panel-heading">
    <h3>标签概览</h3>
    <span id="tag-result-count">准备中...</span>
  </div>
  <div id="tag-result-state" class="search-results"></div>
</section>

<div id="tag-result-list" class="archive-list"></div>

<script>
  (() => {
    const title = document.getElementById('tag-result-title');
    const copy = document.getElementById('tag-result-copy');
    const count = document.getElementById('tag-result-count');
    const state = document.getElementById('tag-result-state');
    const list = document.getElementById('tag-result-list');
    if (!title || !copy || !count || !state || !list) return;

    const params = new URLSearchParams(window.location.search);
    const selectedTag = params.get('tag');

    const tagHref = (tag) => `{{ '/tags/result.html' | relative_url }}?tag=${encodeURIComponent(tag)}`;

    const renderEmpty = (message, summary) => {
      title.textContent = '标签结果';
      copy.textContent = summary;
      count.textContent = '0 篇文章';
      state.innerHTML = `<p class="search-empty">${message}</p>`;
      list.innerHTML = '';
    };

    const renderPosts = (tagInfo) => {
      document.title = `标签：${tagInfo.name} | {{ site.title }}`;
      title.textContent = `#${tagInfo.name}`;
      copy.textContent = `当前展示标签 ${tagInfo.name} 下的全部文章。`;
      count.textContent = `${tagInfo.count} 篇文章`;
      state.innerHTML = `
        <div class="panel-heading">
          <h3>当前标签</h3>
          <span><a class="read-link" href="{{ '/tags/index.html' | relative_url }}">返回标签页</a></span>
        </div>
      `;

      list.innerHTML = tagInfo.posts.map(post => {
        const categories = Array.isArray(post.categories) && post.categories.length ? `<span>${post.categories.join(' / ')}</span>` : '';
        const cover = post.cover_image ? `
          <a class="feed-cover-link" href="${post.url}">
            <img class="feed-cover" src="${post.cover_image}" alt="${post.title}">
          </a>
        ` : '';
        const tags = Array.isArray(post.tags) ? post.tags.map(tag => `
          <a class="chip" href="${tagHref(tag)}">#${tag}</a>
        `).join('') : '';

        return `
          <article class="feed-card">
            <div class="feed-meta">
              <span>${post.date}</span>
              ${categories}
            </div>
            <h2><a href="${post.url}">${post.title}</a></h2>
            ${cover}
            <p class="feed-excerpt">${post.excerpt}</p>
            <div class="feed-footer">
              <div class="chip-row">${tags}</div>
              <a class="read-link" href="${post.url}">阅读全文</a>
            </div>
          </article>
        `;
      }).join('');
    };

    if (!selectedTag) {
      renderEmpty('请选择一个标签后再查看文章列表。', '打开标签页并选择一个标签查看相关文章。');
      return;
    }

    fetch('{{ "/tag-index.json" | relative_url }}')
      .then(res => res.json())
      .then(data => {
        const match = data.find(item => item.name === selectedTag);
        if (!match) {
          renderEmpty(`没有找到标签“${selectedTag}”对应的文章。`, `当前文章集中不存在标签“${selectedTag}”。`);
          return;
        }
        renderPosts(match);
      })
      .catch(() => {
        renderEmpty('标签数据加载失败，请稍后刷新重试。', '当前无法加载标签索引。');
      });
  })();
</script>
