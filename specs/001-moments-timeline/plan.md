# Implementation Plan: 时刻记录（Moments Timeline）

**Branch**: `001-moments-timeline` | **Date**: 2025-11-11 | **Spec**: [spec.md](./spec.md)
**Input**: Feature specification from `/specs/001-moments-timeline/spec.md`

**Note**: This template is filled in by the `/speckit.plan` command. See `.specify/templates/commands/plan.md` for the execution workflow.

## Summary

为 Hexo 博客添加"时刻记录"功能，允许作者通过 GitHub Issues 快速记录灵感和随想，并在博客页面以时间线形式展示。核心价值：
1. **方便更新**：通过 GitHub Issues 在任何设备上发布，无需 Git 操作
2. **自动时间戳**：利用 GitHub API 的 `created_at` 字段，完全自动化
3. **简单展示**：在博客中以时间倒序展示所有时刻记录

技术方案：使用 GitHub Actions 监听 Issue 事件，自动提取带 `moment` 标签的 Issues 并转换为 JSON 数据文件，由 Hexo 在构建时渲染为静态页面。

## Technical Context

**Language/Version**: Node.js (Hexo 6.3.0 运行时) + JavaScript (ES6+)
**Primary Dependencies**:
- Hexo 6.3.0（静态站点生成器）
- Fluid 1.9.4（主题）
- GitHub Actions（CI/CD 自动化）
- GitHub REST API v3（获取 Issues 数据）
- Octokit/rest（可选，Node.js GitHub API 客户端）

**Storage**:
- JSON 文件（存储从 Issues 提取的时刻记录数据，位于 `source/_data/moments.json`）
- GitHub Issues（作为主数据源）

**Testing**:
- 手动验证（MVP 阶段）：验证 Actions 工作流执行、数据文件生成、页面渲染
- 后续可选：Hexo 的测试框架（如需单元测试）

**Target Platform**:
- 服务端：GitHub Actions（Ubuntu latest runner）
- 客户端：现代浏览器（Chrome/Firefox/Safari/Edge 最新两个版本）
- 部署：GitHub Pages（静态托管）

**Project Type**: 单一静态网站项目（Hexo 博客）

**Performance Goals**:
- GitHub Actions 工作流执行时间：< 2 分钟（从 Issue 创建到数据文件更新）
- 时刻页面加载时间：< 2 秒（标准网络条件）
- 支持至少 100 条时刻记录而不影响性能

**Constraints**:
- GitHub API 访问频率限制：每小时 5000 次（已认证），对于个人博客足够
- GitHub Actions 免费额度：公开仓库无限制
- 静态站点约束：无后端数据库，所有数据在构建时生成
- 与 GitHub Pages 兼容：纯静态 HTML/CSS/JS

**Scale/Scope**:
- 预计时刻记录数量：初期 < 50 条，长期 < 500 条
- 单条内容长度：通常 < 500 字，最多支持到 2000 字
- 并发用户：静态页面，理论上无限制（受 GitHub Pages 限制）

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

### I. MVP 优先 ✅

- [x] **功能精简到最小必要范围**：仅实现浏览、发布、编辑/删除三个核心用户故事
- [x] **独立交付和验证**：每个用户故事都可独立测试和演示
- [x] **拒绝一次性实现所有功能**：不包含评论、搜索、RSS、社交分享等非核心功能
- [x] **产生可演示的价值**：P1 用户故事（浏览时刻）即可作为 MVP 发布

**评估**：✅ 符合 MVP 原则。功能范围控制良好，优先级划分清晰。

### II. 可测试性强制要求 ✅

- [x] **明确的验收场景**：每个用户故事都包含 Given-When-Then 格式的验收场景
- [x] **测试方法明确**：手动验证路径清晰（创建 Issue → Actions 执行 → 页面渲染）
- [x] **需求可测试**：所有功能需求都可以通过具体操作验证

**评估**：✅ 符合可测试性要求。所有用户故事都有明确的验收标准。

### III. 拒绝过度设计 ✅

- [x] **当前不需要的抽象层一律不加**：直接使用 GitHub Actions + JSON 文件，无额外抽象
- [x] **不实现"可能有用"的功能**：明确排除了 8 项范围外功能
- [x] **最简单直接的实现方案**：利用现有 GitHub 基础设施，无需自建服务

**评估**：✅ 符合简洁原则。技术方案直接且实用，避免了不必要的复杂性。

### IV. 品质不可妥协 ✅

- [x] **清晰易读**：使用简体中文文档，命名规范（如 `moment` 标签）
- [x] **错误处理**：边界情况已识别（空状态、Actions 失败、API 限流等）
- [x] **适当的日志记录**：GitHub Actions 工作流自带执行日志

**评估**：✅ 符合品质要求。已考虑错误处理和边界情况。

### V. 简体中文规范 ✅

- [x] **文档使用简体中文**：所有规格说明和计划文档均为简体中文
- [x] **注释使用简体中文**：将遵循此规范
- [x] **Git 提交信息使用简体中文**：将遵循 `feat/fix/docs` 格式 + 简体中文描述

**评估**：✅ 符合语言规范要求。

### 整体评估

✅ **所有宪章检查通过**，无需额外证明复杂性。可以进入 Phase 0 研究阶段。

## Project Structure

### Documentation (this feature)

```text
specs/001-moments-timeline/
├── spec.md                  # 功能规格说明
├── plan.md                  # 本文件（实现计划）
├── research.md              # Phase 0 输出（技术研究）
├── data-model.md            # Phase 1 输出（数据模型）
├── quickstart.md            # Phase 1 输出（快速开始指南）
├── contracts/               # Phase 1 输出（API 合约/数据格式定义）
│   └── moments-data.schema.json  # JSON Schema 定义
└── checklists/              # 质量检查清单
    └── requirements.md      # 需求质量检查清单
```

### Source Code (repository root)

这是一个 Hexo 静态博客项目，添加时刻记录功能后的目录结构：

```text
.github/
└── workflows/
    └── sync-moments.yml      # GitHub Actions 工作流（监听 Issues 事件）

source/
├── _data/
│   └── moments.json          # 时刻记录数据文件（由 Actions 生成）
├── _posts/                   # 现有博客文章
├── about/                    # 关于页
│   └── index.md
└── moments/                  # 新增：时刻记录页面
    └── index.md              # 时刻页面主文件

themes/
└── fluid/                    # 现有主题
    └── layout/
        └── moments.ejs       # 可选：自定义时刻页面布局

scripts/
└── moments-helpers.js        # 可选：Hexo 辅助函数（时间格式化等）

_config.yml                   # Hexo 配置（添加导航菜单项）
_config.fluid.yml             # Fluid 主题配置
```

**Structure Decision**:

采用**单一静态网站项目**结构，因为：
1. Hexo 博客是典型的单一项目，无前后端分离
2. 所有功能通过 Hexo 构建时处理，生成静态 HTML
3. GitHub Actions 工作流作为辅助工具，不算独立项目
4. 符合现有 Hexo 项目的标准目录结构

## Complexity Tracking

> **Fill ONLY if Constitution Check has violations that must be justified**

无复杂性违规需要证明。所有宪章检查均已通过。
