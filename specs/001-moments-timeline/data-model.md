# 数据模型：时刻记录

**Feature**: 时刻记录（Moments Timeline）
**Date**: 2025-11-11
**Purpose**: Phase 1 数据模型设计，定义核心实体和数据结构

---

## 概述

本文档定义了"时刻记录"功能的数据模型，包括核心实体、字段定义、验证规则和状态转换。

---

## 核心实体

### 1. Moment（时刻记录）

**描述**：表示某个时间点发布的灵感或随想，对应一个 GitHub Issue。

**字段定义**：

| 字段名 | 类型 | 必填 | 说明 | 示例 |
|--------|------|------|------|------|
| `id` | Integer | 是 | GitHub Issue 编号，唯一标识 | `42` |
| `content` | String | 是 | 时刻内容（Markdown 格式） | `"今天学到了一个新概念..."` |
| `createdAt` | ISO 8601 String | 是 | 发布时间戳（UTC） | `"2025-11-11T14:30:00Z"` |
| `updatedAt` | ISO 8601 String | 是 | 最后编辑时间戳（UTC） | `"2025-11-11T15:00:00Z"` |
| `title` | String | 否 | Issue 标题（可选，暂不使用） | `"关于 Vue 3 的思考"` |

**约束条件**：
- `id` 必须是正整数，全局唯一
- `content` 长度限制：最少 1 字符，最多 10000 字符（GitHub Issue body 限制）
- `createdAt` 和 `updatedAt` 必须是有效的 ISO 8601 格式
- `updatedAt` 必须 >= `createdAt`

**验证规则**：
```javascript
function validateMoment(moment) {
  if (!Number.isInteger(moment.id) || moment.id <= 0) {
    throw new Error('Invalid id: must be a positive integer');
  }
  if (typeof moment.content !== 'string' || moment.content.length === 0) {
    throw new Error('Invalid content: cannot be empty');
  }
  if (moment.content.length > 10000) {
    throw new Error('Invalid content: exceeds 10000 characters');
  }
  const created = new Date(moment.createdAt);
  const updated = new Date(moment.updatedAt);
  if (isNaN(created.getTime()) || isNaN(updated.getTime())) {
    throw new Error('Invalid timestamp format');
  }
  if (updated < created) {
    throw new Error('updatedAt cannot be earlier than createdAt');
  }
  return true;
}
```

**状态转换**：

Moment 实体本身无状态字段，但其生命周期与 GitHub Issue 状态关联：

```
[创建 Issue + moment 标签] → [开放状态]
                                   ↓
                            [编辑 Issue] → [开放状态]
                                   ↓
                            [关闭 Issue] → [隐藏状态]（不在页面显示）
```

**关系**：
- 一个 Moment 对应一个 GitHub Issue（1:1）
- Moments 之间无关联关系（独立存在）

---

### 2. MomentsCollection（时刻记录集合）

**描述**：包含所有时刻记录的集合，存储在 `source/_data/moments.json` 文件中。

**字段定义**：

| 字段名 | 类型 | 必填 | 说明 | 示例 |
|--------|------|------|------|------|
| `moments` | Array<Moment> | 是 | 时刻记录数组 | `[{...}, {...}]` |
| `lastSync` | ISO 8601 String | 是 | 最后同步时间戳（UTC） | `"2025-11-11T15:00:00Z"` |
| `totalCount` | Integer | 是 | 时刻记录总数 | `42` |

**约束条件**：
- `moments` 数组必须按 `createdAt` 降序排列（最新的在前）
- `totalCount` 必须等于 `moments.length`
- `lastSync` 表示最后一次成功同步的时间

**示例数据**：
```json
{
  "moments": [
    {
      "id": 45,
      "content": "刚刚读完《Clean Code》，感觉受益匪浅。",
      "createdAt": "2025-11-11T16:00:00Z",
      "updatedAt": "2025-11-11T16:00:00Z"
    },
    {
      "id": 42,
      "content": "今天学习了 GitHub Actions 的工作流设计，发现它真的很强大！",
      "createdAt": "2025-11-11T14:30:00Z",
      "updatedAt": "2025-11-11T15:00:00Z"
    }
  ],
  "lastSync": "2025-11-11T16:05:00Z",
  "totalCount": 2
}
```

---

## 数据流程

### 1. 数据写入流程（GitHub Actions）

```
1. GitHub Issue 事件触发
   ↓
2. gh CLI 获取所有 moment 标签的开放 Issues
   ↓
3. 转换为 Moment 实体数组
   ↓
4. 按 createdAt 降序排序
   ↓
5. 生成 MomentsCollection JSON
   ↓
6. 写入 source/_data/moments.json
   ↓
7. Git 提交并推送
   ↓
8. 触发 Hexo 重新构建
```

### 2. 数据读取流程（Hexo 构建）

```
1. Hexo 读取 source/_data/moments.json
   ↓
2. 数据加载到 site.data.moments
   ↓
3. moments/index.md 页面渲染
   ↓
4. 遍历 site.data.moments.moments 数组
   ↓
5. Markdown 渲染每个 moment.content
   ↓
6. 生成静态 HTML
```

---

## 数据存储

### 文件位置

```
source/_data/moments.json
```

### 文件权限

- 由 GitHub Actions 工作流自动生成和更新
- 不建议手动编辑（除非调试）
- Git 跟踪此文件，每次更新都会产生新的提交

### 备份策略

- 主数据源：GitHub Issues（永久存储）
- 次要存储：Git 历史（可恢复任意版本的 `moments.json`）
- 无需额外备份机制

---

## 扩展性考虑

### 未来可能添加的字段

如果需要扩展功能，可以考虑以下字段（当前 MVP 不包含）：

| 字段名 | 类型 | 说明 | 用途 |
|--------|------|------|------|
| `tags` | Array<String> | 标签列表 | 分类和筛选 |
| `mood` | String | 心情标签 | 情绪记录 |
| `location` | String | 地理位置 | 上下文信息 |
| `media` | Array<URL> | 媒体资源链接 | 图片、视频等 |

### 数据迁移

如果未来需要从 GitHub Issues 迁移到其他数据源：

1. 导出现有 Issues 为 JSON
2. 转换字段格式（保持相同的数据结构）
3. 修改 GitHub Actions 工作流以读取新数据源
4. 数据模型本身无需变更

---

## JSON Schema

完整的 JSON Schema 定义见 [contracts/moments-data.schema.json](./contracts/moments-data.schema.json)

---

## 总结

### 核心实体

1. **Moment**：单条时刻记录，对应一个 GitHub Issue
2. **MomentsCollection**：所有时刻记录的集合，存储为 JSON 文件

### 关键设计原则

- ✅ **简单直接**：最少必要字段，无过度设计
- ✅ **数据完整性**：所有必填字段都有验证规则
- ✅ **可追溯性**：时间戳记录创建和编辑历史
- ✅ **可扩展性**：预留扩展空间，但不实现未使用的字段

### 下一步

进入合约定义阶段，创建 JSON Schema 文件。
