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

# 新建文章
hugo new content posts/my-new-post.md
```

CI 使用 Hugo 0.159.1 extended 版本（与本地开发环境保持一致），通过 `.github/workflows/hugo.yml` 在 push 到 `main` 时自动部署到 GitHub Pages。升级本地 Hugo 时记得同步修改 workflow 里的 `HUGO_VERSION`。

## 项目结构

- `config/_default/` — 所有站点配置，包括主配置 (`hugo.toml`)、语言 (`languages.zh-cn.toml`)、菜单 (`menus.zh-cn.toml`)、主题参数 (`params.toml`)、Markdown 渲染 (`markup.toml`)
- `content/` — 博客文章和页面内容（Markdown）
- `layouts/partials/` — 仅覆盖了主题的 favicon partial (`favicons.html`)
- `assets/img/` — 首页背景图和头像
- `themes/blowfish/` — Blowfish 主题 git submodule，不要直接修改主题源码

## 关键配置说明

- 主题通过 `.gitmodules` 以 git submodule 方式引入，不要用 Hugo Modules
- `public/` 目录不提交（已在 `.gitignore` 中忽略），部署由 CI 现场构建，本地 `public/` 只是临时产物
- Markdown 渲染启用了 unsafe HTML 和 LaTeX 数学公式支持
- 博客文章的 front matter 支持 tags、categories、series 等分类法
- 自定义布局只需在 `layouts/` 下放置同名文件即可覆盖主题默认模板
