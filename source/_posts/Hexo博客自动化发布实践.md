---
title: Hexo博客自动化发布实践
date: 2026-06-17 22:35:00
tags:
- Hexo
- GitHub
- GitHub Actions
categories:
- Hexo
---

> 记录一次把 Hexo 博客从本地手动发布，改造成 GitHub Actions 自动发布的完整过程。

<!--more-->

## 背景

这个博客原本采用的是 Hexo 比较经典的发布方式：

- `hexo` 分支保存博客源码
- `master` 分支保存生成后的静态文件
- 本地执行 `hexo generate`
- 本地执行 `hexo deploy`

这套方式本身没有问题，但长期维护下来会有几个明显痛点：

- 发布依赖本机环境
- 需要本地 SSH、git deploy、`.deploy_git` 都正常
- 换一台电脑或者换一个维护者，接手成本比较高
- 一旦发布失败，排查信息主要留在本地

所以这次的目标很明确：保留当前仓库结构不变，把发布动作迁移到 GitHub Actions。

## 改造目标

最终希望达到下面这个效果：

1. 日常只需要维护 `hexo` 分支的源码
2. push 到 `hexo` 分支后自动触发构建
3. 自动把 `public` 目录内容同步到 `master`
4. GitHub Pages 继续从 `master` 分支发布站点
5. 本地 `hexo deploy` 继续保留，作为备用方案

## 改造前的清理

在上自动化之前，我先做了一轮低风险整理，避免把历史包袱直接带进工作流：

- 补充了 `package.json` 里的常用脚本
- 统一了搜索配置，去掉重复的索引生成方式
- 去掉未生效的 `sitemap` / `baidusitemap` 配置
- 关闭了默认开启但站点并不使用的 `links` 页面
- 修正了一篇文章里触发构建 warning 的裸链接
- 删除了仓库中未使用的 `themes/next` 和 `themes/landscape`
- 补充了根目录 `README.md`

这一步的好处是：后面如果自动化失败，我们更容易判断到底是工作流问题，还是历史配置问题。

## 工作流设计

这次采用的是“最少改动现有习惯”的方案：

- `hexo` 仍然作为源码分支
- `master` 仍然作为发布分支
- GitHub Actions 从 `hexo` 分支构建
- 构建成功后，把生成结果推送到 `master`

对应的工作流文件放在：

```text
.github/workflows/deploy.yml
```

核心步骤如下：

1. checkout `hexo` 分支源码
2. 安装 Node.js 和依赖
3. 执行 `npm run clean`
4. 执行 `npm run build`
5. 使用 `git worktree` 检出 `master`
6. 将 `public` 同步到 `master`
7. 自动提交并推送

## GitHub 仓库设置检查

为了让自动发布真正跑通，还需要确认仓库设置没有拦截：

### 1. Actions 权限

仓库 `Settings -> Actions -> General` 中，需要满足：

- 允许 GitHub Actions 运行
- `Workflow permissions` 为 `Read and write permissions`

因为工作流需要把构建产物推回 `master` 分支。

### 2. Branch 规则

如果 `master` 分支开启了保护规则，就要允许 Actions 推送，否则 workflow 会构建成功但 push 失败。

这次检查下来，仓库当前并没有开启 `master` 分支保护，所以不会拦截自动部署。

### 3. Pages 发布来源

仓库 `Settings -> Pages` 中，继续保持：

- Source: `Deploy from a branch`
- Branch: `master`
- Folder: `/ (root)`

这样 GitHub Actions 只需要负责更新 `master`，Pages 会继续按原来的方式发布。

## 第一次运行踩到的坑

第一版工作流推上去之后，第一次运行失败了。

失败现象是：

```text
fatal: not a git repository (or any of the parent directories): .git
```

问题出在同步静态文件这一步。

最初我是这样写的：

```bash
rsync -av --delete public/ ../deploy-branch/
```

这里用了 `--delete`，本意是把 `master` 分支里的旧文件清掉，确保生成结果干净。但 `deploy-branch` 是通过 `git worktree` 检出的工作树，它的根目录里也有 `.git` 元数据入口。结果 `rsync --delete` 把这部分一起误删了，后面的 `git add` / `git commit` 自然就报了 “not a git repository”。

修复方法很简单：

```bash
rsync -av --delete --exclude '.git' public/ ../deploy-branch/
```

加上排除规则后，第二次运行就顺利通过了。

## 最终结果

修复后的发布链路已经验证通过：

- push 到 `hexo`
- `Deploy Hexo Site` 自动执行
- 自动生成 `public`
- 自动提交到 `master`
- `pages-build-deployment` 接力发布
- 站点更新完成

同时，仍然保留了本地手动发布能力：

- 推荐方式：push `hexo` 分支，走 GitHub Actions
- 备用方式：本地执行 `npm run deploy`

## 当前推荐用法

日常发布：

```bash
git add .
git commit -m "更新博客内容"
git push origin hexo
```

备用兜底：

```bash
npm run deploy
```

如果只是写文章、改页面、调配置，优先走自动化路径就够了。

## 总结

这次改造最大的价值，不是“少敲几个命令”，而是把发布这件事从“依赖某台电脑的本地操作”，变成了“仓库内可追踪、可复现、可交接的自动流程”。

对于个人博客这种长期维护项目，这类改造往往比继续堆主题效果或页面样式更值得优先做，因为它直接影响未来每一次发布的稳定性和接手成本。
