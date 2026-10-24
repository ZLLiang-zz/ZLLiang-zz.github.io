# ZLLiang 的技术笔记

个人技术博客，基于 [Hugo](https://gohugo.io/) + [PaperMod](https://github.com/adityatelange/hugo-PaperMod)，通过 GitHub Actions 自动部署到 GitHub Pages。

线上地址：<https://zlliang-zz.github.io/>

## 内容方向

- **Java 运行时** —— 内存布局、类加载、GC、JIT
- **并发** —— JMM、锁升级、AQS、线程池、并发容器、虚拟线程
- **数据库** —— InnoDB 内核、索引与优化器、事务与锁
- **分布式** —— 注册中心、RPC、消息队列、缓存一致性

写作原则：讲机制而不是背结论，给出可执行的排查路径，不写没验证过的内容。

## 本地运行

需要 Hugo Extended ≥ 0.146。

```bash
hugo server -D
```

打开 <http://localhost:1313/>。改动会热重载。

构建静态文件：

```bash
hugo --gc --minify
```

产物在 `public/`。

## 写一篇新文章

```bash
hugo new content posts/my-new-post.md
```

或者在 `content/posts/` 下直接建文件，front matter 示例：

```yaml
---
title: "文章标题"
date: 2026-10-24T09:00:00+08:00
summary: "出现在列表页的一句话摘要"
categories: ["Java 并发"]
tags: ["AQS", "并发"]
ShowToc: true
draft: false
---
```

`draft: true` 的文章只在 `hugo server -D` 时可见，不会被部署。

## 部署

推送到 `main` 分支即触发 `.github/workflows/hugo.yaml`：

1. 安装指定版本 Hugo
2. `hugo --gc --minify` 构建
3. 上传产物并发布到 GitHub Pages

首次部署后需在仓库 **Settings → Pages → Build and deployment** 中把 Source 设为 **GitHub Actions**。

## 目录结构

```
.
├── assets/css/extended/    # 自定义样式（自动合并进主题 CSS）
├── content/
│   ├── posts/              # 文章
│   ├── about.md            # 关于页
│   ├── archives.md         # 归档页
│   └── search.md           # 搜索页
├── static/                 # 静态资源（favicon 等）
├── themes/PaperMod/        # 主题（随仓库提交，避免子模块）
├── hugo.toml               # 站点配置
└── .github/workflows/      # 自动部署
```

## License

文章内容版权归作者所有，转载请注明出处。主题与配置代码遵循 MIT。
