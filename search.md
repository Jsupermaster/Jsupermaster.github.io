---
layout: default
nav: search
title: Search
description: Search posts and pages.
permalink: /search/index.html
---
<section class="archive-header">
  <p class="section-kicker">SEARCH</p>
  <h1>Search</h1>
  <p class="archive-copy">Search across posts and key pages in the blog.</p>
</section>

<section class="panel-block search-panel">
  <div class="panel-heading">
    <h3>Query</h3>
    <span>Posts and Pages</span>
  </div>
  <input id="search-input" class="search-input" type="search" placeholder="Search titles, tags, categories..." autocomplete="off">
  <div id="search-results" class="search-results"></div>
</section>

<script>
  (() => {
    const input = document.getElementById('search-input');
    const results = document.getElementById('search-results');
    if (!input || !results) return;
    let items = [];
    const render = (matches, query) => {
      if (!query) {
        results.innerHTML = '<p class="search-empty">Type something to start searching.</p>';
        return;
      }
      if (!matches.length) {
        results.innerHTML = '<p class="search-empty">No matching content found.</p>';
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
    fetch('{{ "/search.json" | relative_url }}')
      .then(res => res.json())
      .then(data => {
        items = data;
        render([], '');
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
