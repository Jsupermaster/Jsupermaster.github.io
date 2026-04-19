---
layout: default
nav: categories
title: Categories
description: Explore the main topic areas of the blog.
permalink: /categories/index.html
topics:
  - title: Latest Work
    description: My latest work, including papers and open-source projects.
    href: /blog/Latest_Work/index.html
  - title: Computer Architecture
    description: Architecture fundamentals, understanding, and tooling.
    href: /blog/Computer_Architecture/index.html
  - title: PIM/PNM
    description: Near-memory computing papers, toolchains, and architecture design.
    href: /blog/PIM/index.html
  - title: AI Accelerator
    description: Latest progress and open-source work on AI accelerators.
    href: /blog/AI_Accelerator/index.html
---
<section class="archive-header">
  <p class="section-kicker">CATEGORIES</p>
  <h1>Categories</h1>
  <p class="archive-copy">Browse the main topic areas covered in this blog.</p>
</section>

<section class="panel-block">
  <div class="panel-heading">
    <h3>Current Focus</h3>
    <span>Main Topic Areas</span>
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
