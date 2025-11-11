# 快速开始指南：时刻记录

**Feature**: 时刻记录（Moments Timeline）
**Date**: 2025-11-11
**Audience**: 博客作者（开发者）

---

## 概述

本指南将帮助你在 Hexo 博客中快速搭建"时刻记录"功能，从零开始到发布第一条时刻记录，整个过程约 30 分钟。

---

## 前提条件

在开始之前，请确保你已经：

- ✅ 拥有一个运行中的 Hexo 博客（版本 >= 6.0）
- ✅ 博客托管在 GitHub 仓库，并使用 GitHub Pages 部署
- ✅ 了解基本的 Git 操作和 Markdown 语法
- ✅ 有权限在仓库中创建 Issues 和编辑 GitHub Actions

---

## 第一步：创建 GitHub Actions 工作流

### 1.1 创建工作流文件

在仓库根目录创建文件：`.github/workflows/sync-moments.yml`

```yaml
name: Sync Moments from Issues

on:
  issues:
    types: [opened, edited, closed, labeled, unlabeled]

jobs:
  sync-moments:
    runs-on: ubuntu-latest
    # 只处理带 "moment" 标签的 Issue
    if: contains(github.event.issue.labels.*.name, 'moment')

    steps:
      - name: Checkout 代码
        uses: actions/checkout@v4
        with:
          token: ${{ secrets.GITHUB_TOKEN }}

      - name: 提取 Issues 数据
        id: fetch-issues
        run: |
          gh issue list \
            --label "moment" \
            --state open \
            --json number,body,createdAt,updatedAt \
            --limit 1000 > raw-moments.json
        env:
          GH_TOKEN: ${{ secrets.GITHUB_TOKEN }}

      - name: 转换为目标格式
        run: |
          jq '{
            moments: [.[] | {
              id: .number,
              content: .body,
              createdAt: .createdAt,
              updatedAt: .updatedAt
            }] | sort_by(.createdAt) | reverse,
            lastSync: (now | todate),
            totalCount: length
          }' raw-moments.json > source/_data/moments.json

      - name: 提交更新
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git add source/_data/moments.json
          if git diff --staged --quiet; then
            echo "无变更，跳过提交"
          else
            git commit -m "build: 同步时刻记录数据 [skip ci]"
            git push
          fi
```

### 1.2 验证工作流

提交并推送工作流文件：

```bash
git add .github/workflows/sync-moments.yml
git commit -m "feat: 添加时刻记录同步工作流"
git push
```

前往 GitHub 仓库的 **Actions** 标签页，确认工作流已成功创建。

---

## 第二步：创建时刻页面

### 2.1 创建页面目录和文件

在 `source/` 目录下创建：`source/moments/index.md`

```markdown
---
title: 时刻
date: 2025-11-11
layout: page
---

<div class="moments-timeline">
  <% if (site.data.moments && site.data.moments.moments.length > 0) { %>
    <% site.data.moments.moments.forEach(function(moment) { %>
      <article class="moment-item">
        <div class="moment-meta">
          <time datetime="<%= moment.createdAt %>">
            <%= new Date(moment.createdAt).toLocaleString('zh-CN') %>
          </time>
          <% if (moment.updatedAt !== moment.createdAt) { %>
            <span class="moment-edited">（已编辑）</span>
          <% } %>
        </div>
        <div class="moment-content">
          <%- markdown(moment.content) %>
        </div>
      </article>
    <% }) %>
  <% } else { %>
    <p class="empty-state">还没有记录任何时刻</p>
  <% } %>
</div>

<style>
.moments-timeline {
  max-width: 800px;
  margin: 0 auto;
}

.moment-item {
  margin-bottom: 2rem;
  padding: 1.5rem;
  border-left: 3px solid #3498db;
  background: #f8f9fa;
  border-radius: 4px;
}

.moment-meta {
  font-size: 0.9rem;
  color: #6c757d;
  margin-bottom: 0.75rem;
}

.moment-edited {
  margin-left: 0.5rem;
  font-style: italic;
}

.moment-content {
  line-height: 1.8;
}

.moment-content p:last-child {
  margin-bottom: 0;
}

.empty-state {
  text-align: center;
  color: #6c757d;
  padding: 2rem;
}
</style>
```

### 2.2 添加导航菜单

编辑 `_config.yml`（Hexo 配置文件），在 `menu` 部分添加：

```yaml
menu:
  首页: /
  归档: /archives
  分类: /categories
  标签: /tags
  关于: /about
  时刻: /moments  # 新增此行
```

如果使用 Fluid 主题，编辑 `_config.fluid.yml`，在 `navbar` 部分添加：

```yaml
navbar:
  menu:
    - { key: "home", link: "/", icon: "iconfont icon-home-fill" }
    - { key: "archive", link: "/archives/", icon: "iconfont icon-archive-fill" }
    - { key: "category", link: "/categories/", icon: "iconfont icon-category-fill" }
    - { key: "tag", link: "/tags/", icon: "iconfont icon-tags-fill" }
    - { key: "about", link: "/about/", icon: "iconfont icon-user-fill" }
    - { key: "moments", link: "/moments/", icon: "iconfont icon-time-fill" }  # 新增此行
```

并在语言文件中添加翻译（`themes/fluid/languages/zh-CN.yml`）：

```yaml
nav:
  moments: 时刻
```

---

## 第三步：创建第一个时刻记录

### 3.1 创建 moment 标签

1. 前往 GitHub 仓库的 **Issues** 标签页
2. 点击 **Labels** → **New label**
3. 创建标签：
   - **Name**: `moment`
   - **Description**: `时刻记录`
   - **Color**: 随意选择（推荐 `#3498db` 蓝色）

### 3.2 创建第一条时刻

1. 点击 **New issue**
2. 填写内容：
   - **Title**: 留空或填写简短标题（当前版本暂不使用）
   - **Body**: 写下你的第一条时刻（支持 Markdown）
     ```markdown
     今天开始使用时刻记录功能，记录灵感和随想！

     这是一个测试时刻，支持 **粗体**、*斜体* 和 [链接](https://example.com)。
     ```
3. 在右侧 **Labels** 中选择 `moment` 标签
4. 点击 **Submit new issue**

### 3.3 等待同步

1. 前往 **Actions** 标签页
2. 查看 "Sync Moments from Issues" 工作流是否正在运行
3. 等待工作流完成（通常 < 1 分钟）
4. 检查仓库中是否生成了 `source/_data/moments.json` 文件

### 3.4 本地预览

拉取最新代码并本地预览：

```bash
git pull
hexo clean
hexo server
```

访问 `http://localhost:4000/moments/` 查看你的第一条时刻记录！

---

## 第四步：发布到生产环境

### 4.1 推送到远程仓库

如果你在本地做了任何修改（如配置文件），提交并推送：

```bash
git add .
git commit -m "feat: 完成时刻记录功能配置"
git push
```

### 4.2 触发 GitHub Pages 部署

GitHub Pages 会自动检测到仓库更新并重新部署。等待几分钟后，访问你的博客：

```
https://<username>.github.io/moments/
```

---

## 日常使用

### 发布新时刻

1. 前往 GitHub 仓库 Issues 页面
2. 点击 **New issue**
3. 写下内容（支持 Markdown）
4. 添加 `moment` 标签
5. 提交 Issue
6. 等待 1-2 分钟，GitHub Actions 自动同步并部署

**提示**：你可以在手机浏览器或 GitHub App 中操作，随时随地发布时刻！

### 编辑时刻

1. 找到对应的 Issue
2. 点击 **Edit** 编辑内容
3. 保存后，GitHub Actions 自动同步更新

### 删除时刻

1. 找到对应的 Issue
2. 点击 **Close issue**
3. 该时刻将从页面上移除（但 Issue 本身仍保留在 GitHub）

---

## 故障排查

### 问题 1：Actions 工作流未触发

**检查**：
- Issue 是否添加了 `moment` 标签？
- 工作流文件路径是否正确（`.github/workflows/sync-moments.yml`）？

### 问题 2：数据文件未生成

**检查**：
- 前往 Actions 日志，查看 "提取 Issues 数据" 步骤是否成功
- 确认 `gh` CLI 命令是否正确执行
- 检查是否有开放状态的带 `moment` 标签的 Issue

### 问题 3：页面显示为空

**检查**：
- `source/_data/moments.json` 文件是否存在？
- 文件内容是否符合 JSON 格式？
- Hexo 是否成功读取数据文件（`hexo clean && hexo server` 重新生成）

### 问题 4：时间显示不正确

**原因**：时间戳为 UTC 时区，浏览器会自动转换为本地时区。

**解决**：在前端使用 JavaScript 格式化时间（参见第二步的示例代码）。

---

## 下一步

恭喜！你已经成功搭建了时刻记录功能。接下来可以：

1. **自定义样式**：修改 `source/moments/index.md` 中的 CSS
2. **优化时间显示**：使用 moment.js 或 day.js 库实现相对时间（"2小时前"）
3. **添加分页**：如果时刻记录超过 50 条，考虑实现分页功能
4. **SEO 优化**：为时刻页面添加 meta 描述和关键词

更多技术细节，请参阅：
- [data-model.md](./data-model.md) - 数据模型定义
- [research.md](./research.md) - 技术研究文档
- [contracts/moments-data.schema.json](./contracts/moments-data.schema.json) - JSON Schema

---

## 支持

如有问题，请在项目 Issues 中提问。
