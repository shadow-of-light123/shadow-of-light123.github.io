# Jason Chen 的个人博客

这是我的个人博客源码仓库，基于 [Hugo](https://gohugo.io/) 静态站点框架和 [Blowfish](https://blowfish.page/) 主题构建。

- **线上地址**：<https://www.chenjingyi.top/>
- **博主**：我自己，熟悉Vue + React前端开发，目前正向全栈(Java)方向学习

## 技术栈

| 项目     | 说明                                         |
| -------- | -------------------------------------------- |
| 站点框架 | Hugo（extended 版本，CI 中使用 0.159.1）     |
| 主题     | Blowfish（通过 git submodule 引入）          |
| 语言     | 中文（zh-cn）                                |
| 部署     | GitHub Actions 自动构建并发布到 GitHub Pages |

## 项目结构

```text
.
├── config/_default/      # 站点配置（主配置、语言、菜单、主题参数等）
├── content/
│   ├── bagu/             # 八股：技术知识点整理（Markdown）
│   ├── essays/           # 随笔：想法与记录（Markdown）
│   └── about/            # 关于页面
├── layouts/partials/     # 覆盖主题的自定义模板（如 favicon）
├── assets/img/           # 背景图、头像等静态资源
├── themes/blowfish/      # Blowfish 主题（git submodule，勿直接修改）
└── .github/workflows/    # CI 自动部署配置
```

## 本地开发

确保已安装 [Hugo extended](https://gohugo.io/installation/) 版本：

```bash
# 拉取主题 submodule（首次克隆后执行）
git submodule update --init --recursive

# 启动本地开发服务器（含草稿、未来日期文章）
hugo server -E -F --minify --buildDrafts

# 构建生产版本
hugo --minify
```

## 写新文章

```bash
# 八股（技术知识点）
hugo new content bagu/my-new-post.md

# 随笔（想法与记录）
hugo new content essays/my-new-post.md
```

文章分别存放在 `content/bagu/`（八股）和 `content/essays/`（随笔）目录下，front matter 支持 `tags`、`categories`、`series` 等分类法。将 `draft` 设为 `false` 后，文章才会在正式构建中发布。带图片的文章建议用页面束（page bundle）形式：新建 `content/essays/文章名/index.md`，图片放同目录下的 `images/` 里，正文用相对路径 `images/xxx.jpg` 引用。

## 部署

推送（push）到 `main` 分支后，GitHub Actions 会自动完成以下操作：

1. 拉取代码和主题 submodule
2. 使用 Hugo 构建静态站点
3. 发布到 GitHub Pages

无需手动构建或提交 `public/` 目录（该目录已被 `.gitignore` 忽略）。

## 联系方式

- 邮箱：<cjy18297922182@163.com>
- GitHub：<https://github.com/shadow-of-light123>
