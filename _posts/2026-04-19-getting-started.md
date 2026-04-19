---
title: 我的第一篇 GitHub Pages 博客
date: 2026-04-19 14:30:00 +0800
description: 记录博客从纯静态页面切换为 Jekyll Markdown 系统的第一篇文章。
categories:
  - Latest Work
tags:
  - Latest Work
  - Jekyll
  - Markdown
article_kicker: BLOG POST
---

这篇文章是整个站点上线后的第一条正式内容。和之前的纯静态页面相比，现在首页已经更接近一个完整的技术博客，而不是单纯的个人名片页。

我把结构拆成了三部分：首页负责展示最近内容和分类入口，归档页负责汇总文章，文章页则专注阅读体验。这样后续新增内容时，不需要推翻重做模板，只需要继续加文章即可。

参考站给我的启发主要有两点：

1. 博客首页要一眼看出“这是持续更新的内容站”。
2. 文章列表和右侧信息栏要足够清晰，让访问者快速了解方向、分类和标签。

## 接下来怎么继续更新

后面如果要发新文章，只需要继续在 `_posts/` 目录新建一个 Markdown 文件，例如：

```text
_posts/2026-04-20-my-next-post.md
```

然后写上 front matter 和正文即可：

```md
---
title: 我的下一篇文章
date: 2026-04-20 20:00:00 +0800
categories: [AI]
tags: [Notes, Demo]
---

这里是正文内容。
```

GitHub Pages 会在构建时自动把它渲染成博客文章。
