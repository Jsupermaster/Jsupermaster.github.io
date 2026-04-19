---
title: 博客模板结构说明
date: 2026-04-18 20:00:00 +0800
description: 说明当前博客模板的目录结构和维护方式。
categories:
  - Computer Architecture
tags:
  - Computer Architecture
  - Studing Note
  - Jekyll
article_kicker: TEMPLATE NOTE
---

现在这套博客模板已经从纯 HTML 结构切换到了 Jekyll。

当前最重要的目录有几个：

- `_layouts/`：放页面模板，比如首页、归档页、文章页。
- `_posts/`：放博客文章，文件名必须是 `年-月-日-标题.md`。
- `assets/`：放样式、图片和其他静态资源。
- `_config.yml`：放站点配置，比如标题、URL、Markdown 渲染方式。

## 以后发文章的推荐流程

1. 在 `_posts/` 里新建一个 `.md` 文件。
2. 写好文章标题、日期、分类和标签。
3. 推送到 GitHub。
4. 等 GitHub Pages 自动重新构建。

这样你以后写博客就不再需要手写整页 HTML 了。
