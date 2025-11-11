# 技术研究文档：时刻记录功能

**Feature**: 时刻记录（Moments Timeline）
**Date**: 2025-11-11
**Purpose**: Phase 0 技术研究，解决实现中的关键技术决策

---

## 研究概述

本文档记录了实现"时刻记录"功能过程中的技术研究成果，包括 GitHub Actions 工作流设计、GitHub Issues API 使用、Hexo 数据文件处理、以及时间格式化等关键技术点。

---

## 1. GitHub Actions 工作流设计

### 决策：工作流触发方式

**选择**：使用 `issues` 事件触发器，监听 `opened`、`edited`、`closed` 和 `labeled` 事件

**理由**：
- `issues` 事件是 GitHub Actions 的标准事件，稳定可靠
- 可以精确控制在哪些操作时触发工作流
- 支持通过 `types` 过滤具体的事件类型

**替代方案考虑**：
- ❌ **定时任务（cron）**：延迟太高，用户体验差（Issue 创建后可能需要等待数分钟）
- ❌ **手动触发（workflow_dispatch）**：不符合"自动化"需求，增加操作步骤

**实现示例**：
```yaml
on:
  issues:
    types: [opened, edited, closed, labeled, unlabeled]
```

### 决策：Issue 过滤策略

**选择**：在工作流中通过条件判断过滤标签

**理由**：
- GitHub Actions 不支持在触发器层面过滤标签
- 需要在 job 层面使用 `if` 条件检查 `moment` 标签
- 避免所有 Issue 都触发工作流浪费资源

**实现示例**：
```yaml
jobs:
  sync-moments:
    runs-on: ubuntu-latest
    if: contains(github.event.issue.labels.*.name, 'moment')
    steps:
      # ...
```

### 决策：数据提取方法

**选择**：使用 GitHub CLI (`gh`) 命令行工具

**理由**：
- GitHub CLI 已预装在 Actions runners 中，无需额外安装
- 命令简洁，输出JSON格式易于处理
- 自动处理认证（使用 `GITHUB_TOKEN`）

**替代方案考虑**：
- ❌ **Octokit/rest (Node.js)**：需要编写额外的 JavaScript 脚本，增加复杂度
- ❌ **curl + GitHub API**：需要手动处理分页、认证等，代码冗长

**实现示例**：
```bash
gh issue list \
  --label "moment" \
  --state open \
  --json number,title,body,createdAt,updatedAt \
  --limit 100
```

---

## 2. GitHub Issues API 使用

### 决策：API 版本和端点

**选择**：使用 GitHub REST API v3 + GitHub CLI

**理由**：
- REST API v3 是稳定且广泛使用的版本
- GitHub CLI 封装了 API 调用的复杂性
- 支持所有需要的字段（number, title, body, timestamps）

**关键字段映射**：
```json
{
  "number": 42,           // Issue 编号（唯一标识）
  "title": "时刻标题",    // 可选使用
  "body": "内容...",      // Markdown 格式的时刻内容
  "createdAt": "2025-11-11T14:30:00Z",  // ISO 8601 时间戳
  "updatedAt": "2025-11-11T15:00:00Z"   // 编辑时间
}
```

### 决策：分页处理

**选择**：使用 `--limit` 参数限制返回数量

**理由**：
- 根据规格说明，预计长期 < 500 条记录
- GitHub CLI 默认返回 30 条，可以设置 `--limit 1000`
- 对于个人博客，一次性获取所有记录性能可接受

**未来优化**：
- 如果记录数超过 1000 条，考虑使用 `--paginate` 参数
- 或实现增量同步（仅同步最近更新的 Issues）

### 决策：时间戳处理

**选择**：保留 ISO 8601 格式存储，在前端渲染时转换

**理由**：
- ISO 8601 是标准格式，跨平台兼容性好
- 避免在 Actions 脚本中进行时区转换（复杂且容易出错）
- 前端 JavaScript 可以轻松解析并转换为本地时间

**示例转换**（前端 JS）：
```javascript
const createdAt = new Date('2025-11-11T14:30:00Z');
const relativeTime = getRelativeTime(createdAt); // "2小时前"
```

---

## 3. Hexo 数据文件和自定义页面

### 决策：数据文件位置和格式

**选择**：存储为 `source/_data/moments.json`

**理由**：
- Hexo 约定：`source/_data/` 目录下的文件自动加载为全局数据
- JSON 格式易于 GitHub Actions 生成和 Hexo 读取
- 文件名 `moments.json` 清晰表达用途

**数据结构**：
```json
{
  "moments": [
    {
      "id": 42,
      "content": "今天的灵感...",
      "createdAt": "2025-11-11T14:30:00Z",
      "updatedAt": "2025-11-11T14:30:00Z"
    }
  ],
  "lastSync": "2025-11-11T15:00:00Z"
}
```

### 决策：自定义页面实现方式

**选择**：使用 Hexo 的 Page 功能创建 `source/moments/index.md`

**理由**：
- Hexo Page 是创建独立页面的标准方式
- 支持 Front Matter 自定义布局
- 可以在页面中通过 `site.data.moments` 访问数据文件

**替代方案考虑**：
- ❌ **Hexo Generator 插件**：过度设计，增加复杂度
- ❌ **直接修改主题模板**：耦合度高，主题更新时容易冲突

**实现示例**（`source/moments/index.md`）：
```markdown
---
title: 时刻
layout: page
---

<div class="moments-timeline">
  <% site.data.moments.moments.forEach(function(moment) { %>
    <div class="moment-item">
      <div class="moment-time"><%= moment.createdAt %></div>
      <div class="moment-content"><%- markdown(moment.content) %></div>
    </div>
  <% }) %>
</div>
```

### 决策：Markdown 渲染

**选择**：使用 Hexo 内置的 `markdown()` 辅助函数

**理由**：
- Hexo 已经包含 Markdown 渲染器（hexo-renderer-marked）
- 与博客文章使用相同的渲染引擎，样式一致
- 无需额外依赖

**安全性考虑**：
- GitHub Issues 的 Markdown 内容已经过 GitHub 的安全过滤
- Hexo 的 `<%- %>` 输出原始 HTML（已渲染的 Markdown）
- 无需额外的 XSS 防护

---

## 4. 时间格式化最佳实践

### 决策：时间显示策略

**选择**：使用相对时间（"2小时前"）+ 绝对时间（"2025-11-11 14:30"）的组合

**理由**：
- 相对时间更人性化，适合近期的时刻记录
- 绝对时间提供精确信息，适合历史记录
- 符合用户习惯（类似 Twitter、微博）

**实现策略**：
```javascript
function formatMomentTime(isoString) {
  const date = new Date(isoString);
  const now = new Date();
  const diff = now - date;

  // 1 小时内：相对时间
  if (diff < 3600000) {
    const minutes = Math.floor(diff / 60000);
    return `${minutes} 分钟前`;
  }

  // 24 小时内：相对时间
  if (diff < 86400000) {
    const hours = Math.floor(diff / 3600000);
    return `${hours} 小时前`;
  }

  // 7 天内：相对时间
  if (diff < 604800000) {
    const days = Math.floor(diff / 86400000);
    return `${days} 天前`;
  }

  // 超过 7 天：绝对时间
  return date.toLocaleDateString('zh-CN') + ' ' + date.toLocaleTimeString('zh-CN', {hour: '2-digit', minute: '2-digit'});
}
```

### 决策：时区处理

**选择**：在前端使用浏览器本地时区

**理由**：
- 读者可能来自不同时区
- JavaScript `Date` 对象自动处理时区转换
- ISO 8601 格式（UTC）便于跨时区传输

**注意事项**：
- 服务器端（GitHub Actions）统一使用 UTC
- 前端渲染时转换为本地时区
- 避免在数据文件中存储时区相关的字符串

---

## 5. GitHub Actions 工作流完整设计

### 决策：工作流步骤

**选择**：5 步工作流

1. **Checkout 代码**：获取仓库最新代码
2. **提取 Issues 数据**：使用 `gh` CLI 获取所有带 `moment` 标签的开放 Issues
3. **转换为 JSON**：使用 `jq` 工具格式化数据
4. **提交更新**：使用 `git` 提交 `moments.json` 文件
5. **触发部署**：推送到 main 分支，触发 GitHub Pages 部署

**理由**：
- 步骤清晰，易于调试
- 每步职责单一，符合"拒绝过度设计"原则
- 利用现有工具（gh, jq, git），无需自定义脚本

### 决策：错误处理策略

**选择**：使用 `continue-on-error: false`（默认行为）

**理由**：
- 任何步骤失败都应该停止工作流
- GitHub Actions 自动发送失败通知邮件
- 作者可以在 Actions 日志中查看详细错误信息

**边界情况处理**：
- **无 Issues**：`gh issue list` 返回空数组，生成空的 `moments.json`
- **API 限流**：GitHub CLI 自动处理重试，超时后失败并通知
- **Git 冲突**：使用 `git pull --rebase` 确保与远程同步

---

## 6. 性能优化策略

### 决策：缓存策略

**选择**：MVP 阶段不实现缓存，直接每次全量同步

**理由**：
- 预计 Issue 数量 < 100，API 调用和数据处理耗时 < 10 秒
- 过早优化违反 MVP 原则
- GitHub Actions 免费额度对公开仓库无限制

**未来优化方向**（当 Issues > 500 时）：
- 增量同步：仅同步最近更新的 Issues
- 使用 GitHub Actions cache 缓存历史数据
- 实现条件触发：仅当 `moment` 标签的 Issue 变更时触发

### 决策：前端渲染性能

**选择**：服务端渲染（Hexo 构建时生成 HTML）

**理由**：
- 静态 HTML 加载速度快
- 无需前端 JavaScript 动态加载数据
- SEO 友好

**预计性能**：
- 100 条记录：页面大小 < 100KB（压缩后），加载时间 < 1 秒
- 500 条记录：考虑分页或无限滚动

---

## 总结

### 关键技术决策

| 决策点 | 选择 | 理由 |
|--------|------|------|
| 工作流触发 | `issues` 事件 | 实时性好，标准可靠 |
| API 调用方式 | GitHub CLI (`gh`) | 简洁，预装，易用 |
| 数据存储格式 | JSON 文件 | Hexo 原生支持，易于处理 |
| 页面实现方式 | Hexo Page | 标准方式，无过度设计 |
| 时间显示 | 相对时间 + 绝对时间 | 用户体验好，信息完整 |
| 错误处理 | 失败即停止 + 邮件通知 | 简单可靠 |

### 未解决问题

无。所有技术细节已明确。

### 下一步

进入 Phase 1，生成：
1. **data-model.md**：定义时刻记录的数据模型和 JSON Schema
2. **contracts/moments-data.schema.json**：JSON Schema 文件
3. **quickstart.md**：快速开始指南（如何创建第一个时刻）
