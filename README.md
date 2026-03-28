# Yao's Blog

这是我的个人博客，使用 [Hexo](https://hexo.io/) + [Fluid 主题](https://hexo.fluid-dev.com/) 构建，托管在 [GitHub Pages](https://pages.github.com/)。

🌐 博客地址：https://artistzq.github.io

---

## 📌 分支说明

本仓库有两个分支：

| 分支 | 内容 | 用途 |
|------|------|------|
| **`main`** | Hexo 源文件（Markdown 文章、配置文件） | 日常写文章、编辑配置 |
| **`gh-pages`** | 编译后的静态 HTML | GitHub Pages 自动部署 |

### 工作流程

```
写文章 (main 分支)
    ↓
hexo generate
    ↓
生成静态文件
    ↓
hexo deploy
    ↓
推送到 gh-pages 分支 → GitHub Pages 自动部署
```

**日常开发请在 `main` 分支进行！**

---

## 📝 本地开发

### 环境要求

- [Node.js](https://nodejs.org/) (推荐 v18+)
- npm

### 安装依赖

```bash
npm install
```

### 启动本地服务器

```bash
npx hexo server
```

访问 http://localhost:4000

---

## ✍️ 写新文章

```bash
# 创建新文章
npx hexo new "文章标题"

# 文章位于 source/_posts/文章标题.md
# 编辑后保存即可
```

---

## 🚀 部署

```bash
# 清理 + 生成 + 部署
npx hexo clean && npx hexo generate && npx hexo deploy
```

部署会自动推送到 GitHub 的 `gh-pages` 分支。

---

## 📁 目录结构

```
.
├── source/              # 源文件
│   ├── _posts/          # 文章
│   ├── about/           # 页面
│   └── img/             # 图片
├── themes/              # 主题（由 npm 管理）
├── _config.yml          # 站点配置
├── _config.fluid.yml    # 主题配置
├── package.json         # 依赖配置
└── README.md            # 本文档
```

---

## 📄 License

MIT
