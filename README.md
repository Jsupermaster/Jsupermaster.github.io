# GitHub Pages 个人主页和博客

这个目录已经是一个可以直接上传到 GitHub Pages 的静态站点。

## 站点内容

- `index.html`：个人主页
- `blog/index.html`：博客列表页
- `posts/getting-started.html`：第一篇示例文章
- `assets/styles.css`：站点样式

## 你的仓库应该怎么命名

GitHub Pages 个人主页仓库名必须和你的 **GitHub 用户名** 完全一致：

- 如果你的 GitHub 用户名是 `Jiayt2004`，仓库名就要建成 `Jiayt2004.github.io`
- 如果你的 GitHub 用户名是 `Jsupermaster`，仓库名就要建成 `Jsupermaster.github.io`

注意：两个用户名里只能有一个是真正的 GitHub 账号用户名，最终以你登录 GitHub 后页面右上角显示的账号为准。

## 上传步骤

1. 登录 GitHub。
2. 新建一个公开仓库，仓库名使用 `你的用户名.github.io`。
3. 把当前文件夹里的所有文件上传到仓库根目录。
4. 打开仓库的 `Settings`。
5. 进入 `Pages`。
6. 在 `Build and deployment` 中选择从分支部署。
7. 分支选 `main`，目录选 `/ (root)`。
8. 保存后等待几分钟。

完成后访问：

- `https://Jiayt2004.github.io`
- 或 `https://Jsupermaster.github.io`

实际访问哪个地址，取决于你真正使用的是哪个 GitHub 用户名。

## 后续怎么写博客

最简单的方法是继续复制 `posts/getting-started.html`，改成新的文件名，例如：

- `posts/my-second-post.html`
- `posts/learning-notes.html`

然后把新文章链接补到 `blog/index.html` 里。

## 你现在可以优先改的内容

1. 把主页里的自我介绍改成你自己的信息。
2. 把项目卡片改成你的真实项目。
3. 把示例文章改成你的第一篇正式博客。
4. 上传到 GitHub Pages 发布。
