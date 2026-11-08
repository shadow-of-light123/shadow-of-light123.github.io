# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

这是一个基于 Hugo 的个人博客，使用 Blowfish 主题（通过 git submodule 引入），语言为中文（zh-cn），部署在 GitHub Pages 上，地址为 `https://www.chenjingyi.top/`。

## 常用命令

```bash
# 本地开发服务器（含草稿）
hugo server -E -F --minify --buildDrafts

# 构建生产版本
hugo --minify

# 新建文章（八股 / 随笔）
hugo new content bagu/my-new-post.md
hugo new content mianjing/my-new-post.md
hugo new content essays/my-new-post.md
```

CI 使用 Hugo 0.165.0 extended 版本（与本地开发环境保持一致），通过 `.github/workflows/hugo.yml` 在 push 到 `main` 时自动部署到 GitHub Pages。升级本地 Hugo 时记得同步修改 workflow 里的 `HUGO_VERSION`。

## 项目结构

- `config/_default/` — 所有站点配置，包括主配置 (`hugo.toml`)、语言 (`languages.zh-cn.toml`)、菜单 (`menus.zh-cn.toml`)、主题参数 (`params.toml`)、Markdown 渲染 (`markup.toml`)
- `content/` — 博客文章和页面内容（Markdown），分为 `bagu/`（八股，技术知识点）、`mianjing/`（面经）和 `essays/`（随笔）三个 section，外加 `about/` 页面
- `layouts/partials/` — 站级覆盖：评论 (`comments.html`，Waline)、favicon (`favicons.html`)、首页背景 (`home/background.html`)、最近文章 (`recent-articles/main.html`)、文章 meta 行 (`article-meta/basic.html`，含浏览量/点赞挂件入口) 与点赞/浏览量 partial (`meta/waline-views.html`、`meta/waline-likes.html`)
- `assets/img/` — 首页背景图和头像
- `themes/blowfish/` — Blowfish 主题 git submodule，不要直接修改主题源码

## 关键配置说明

- 主题通过 `.gitmodules` 以 git submodule 方式引入，不要用 Hugo Modules
- `public/` 目录不提交（已在 `.gitignore` 中忽略），部署由 CI 现场构建，本地 `public/` 只是临时产物
- Markdown 渲染启用了 unsafe HTML 和 LaTeX 数学公式支持
- 博客文章的 front matter 支持 tags、categories、series 等分类法
- `params.toml` 里的 `mainSections = ["bagu", "mianjing", "essays"]` 决定首页「最近文章」和搜索索引收录哪些 section，新增 section 时要同步更新
- 带图片的文章用页面束（page bundle）：`content/<section>/<文章名>/index.md`，图片放同目录 `images/` 下，正文用 `images/xxx.jpg` 相对路径引用
- 自定义布局只需在 `layouts/` 下放置同名文件即可覆盖主题默认模板
- 文章浏览量/点赞：`params.toml` 里的 `article.showViews` / `article.showLikes` 保持 `false`（不使用主题的 Firebase 方案）；站级 `layouts/partials/article-meta/basic.html` 在单篇文章页注入 👁 浏览量与 ❤ 点赞，计数复用 Waline 的 `/api/article` 接口（浏览量 `type=time`，点赞 `type=reaction0`）；点赞脚本在 `layouts/partials/comments.html`，样式在 `assets/css/custom.css` 末尾。升级 Blowfish 主题后需对比主题原 `article-meta/basic.html` 同步覆盖文件。
