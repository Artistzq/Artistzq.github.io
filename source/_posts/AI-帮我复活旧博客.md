---
title: AI 帮我复活旧博客
date: 2026-03-28 17:06:09
tags:
  - Hexo
  - GitHub Pages
  - AI 工具
categories:
  - 日常
---

> **说明**：本篇文章由 AI 辅助撰写，包括博客的整个复活流程也是 AI 帮我完成的。

## 背景

我的博客托管在 GitHub Pages 上，之前一直是用 Hexo 搭建的。但好久没管它，最近想重新捡起来写点东西，结果发现完全不知道怎么搞了……

更糟糕的是，GitHub 仓库里只有编译后的 HTML 文件，源文件（Markdown 文章、配置文件什么的）全在本地，而我早就找不到原来的文件夹在哪了。

## 找回源文件

好在天无绝人之路，我在电脑里翻到了一个旧文件夹 `~/Projects/blogs`，里面居然完整地保存着 Hexo 项目的所有源文件：

- 6 篇之前的文章（都是关于 K8s 的学习笔记）
- Hexo 配置文件 `_config.yml`
- Fluid 主题配置 `_config.fluid.yml`
- 各种主题、模板之类的

看到 `source/_posts/` 目录里的 Markdown 文件时，真的松了一口气。

## 让 AI 帮忙

虽然找到了源文件，但我还是遇到了不少问题：

### 1. 本地环境没了

运行 `hexo server` 的时候发现命令不存在，Node.js 环境也不知道什么时候被我搞没了。

**解决方法**：用 nvm 重新安装 Node.js

```bash
brew install nvm
nvm install --lts
nvm use --lts
```

### 2. 源文件没备份到 GitHub

之前的我只会把编译后的 HTML 推送到 GitHub（`hexo deploy`），源文件从来没上传过。这意味着如果哪天电脑坏了，这些文章就全没了。

**解决方法**：

1. 在本地初始化 Git 仓库
2. 创建一个 `main` 分支专门放源码
3. 配置 `.gitignore` 排除 `node_modules/`、`themes/` 这些可以重新下载的文件
4. 推送到 GitHub

### 3. 分支管理混乱

一开始我没搞清楚 Hexo 的部署机制，以为 `main` 分支就是放源码的。后来才明白：

- GitHub Pages 会读取某个分支的内容来构建网站
- 如果把源码放在 `main` 分支，那部署的内容就没地方放了

**最终的分支结构**：

| 分支 | 内容 |
|------|------|
| `main` | Hexo 源文件（我日常开发的分支） |
| `gh-pages` | 编译后的静态 HTML（GitHub Pages 读取这个） |

### 4. 部署配置

修改 `_config.yml` 里的 deploy 配置：

```yaml
deploy:
  type: git
  repository: https://github.com/Artistzq/Artistzq.github.io.git
  branch: gh-pages
```

然后运行：

```bash
hexo clean && hexo generate && hexo deploy
```

就会自动把生成的静态文件推送到 `gh-pages` 分支，GitHub Pages 会自动部署。

## 最终成果

现在我的博客已经完全复活了：

- 源文件备份在 GitHub 的 `main` 分支，换电脑也不怕丢
- 运行 `hexo server` 可以本地预览
- 运行 `hexo deploy` 可以一键部署
- 写了一篇 README 记录整个项目结构

访问地址：https://artistzq.github.io

## 总结

这次复活博客的过程，让我意识到几个重要的点：

1. **源文件一定要备份**：最好是 Git 仓库，本地 + 云端双保险
2. **搞清楚部署机制**：Hexo 的 `deploy` 命令其实是把 `public/` 目录推送到指定的 Git 分支
3. **分支管理要清晰**：源码和部署文件分开，避免混乱

最后，感谢 AI 的帮忙，让我少走了很多弯路。接下来就可以安心写文章了！
