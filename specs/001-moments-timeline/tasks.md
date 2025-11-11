---

description: "时刻记录功能的任务列表"
---

# Tasks: 时刻记录（Moments Timeline）

**Input**: 设计文档来自 `/specs/001-moments-timeline/`
**Prerequisites**: plan.md, spec.md, data-model.md, contracts/moments-data.schema.json

**注意**: 本功能使用手动验证方式，不包含自动化测试任务。

**组织方式**: 任务按用户故事分组，每个故事可独立实现和测试。

## Format: `[ID] [P?] [Story] Description`

- **[P]**: 可并行运行（不同文件，无依赖）
- **[Story]**: 属于哪个用户故事（US1, US2, US3）
- 包含准确的文件路径

## Path Conventions

- **Hexo 博客项目**: 仓库根目录结构
- **工作流**: `.github/workflows/`
- **数据文件**: `source/_data/`
- **页面**: `source/moments/`
- **配置**: `_config.yml` 和 `_config.fluid.yml`

---

## Phase 1: Setup (共享基础设施)

**Purpose**: 项目初始化和基本结构准备

- [x] T001 创建 `.github/workflows/` 目录（如不存在）
- [x] T002 [P] 创建 `source/_data/` 目录（如不存在）
- [x] T003 [P] 创建 `source/moments/` 目录
- [ ] T004 [P] 在 GitHub 仓库创建 `moment` 标签（Label: moment, Color: #3498db, Description: 时刻记录）<!-- 需手动完成 -->

---

## Phase 2: Foundational (阻塞性前置条件)

**Purpose**: 核心基础设施，必须在所有用户故事之前完成

**⚠️ CRITICAL**: 此阶段必须完成，所有用户故事才能开始

- [x] T005 创建 GitHub Actions 工作流文件 `.github/workflows/sync-moments.yml`（监听 Issues 事件，过滤 moment 标签，使用 gh CLI 提取数据，jq 转换 JSON，Git 提交）
- [x] T006 验证工作流语法正确性（提交后检查 Actions 页面，确认工作流已注册）<!-- 将在提交后由 GitHub 验证 -->
- [x] T007 创建空的 `source/_data/moments.json` 文件（初始内容：`{"moments":[],"lastSync":"","totalCount":0}`）

**Checkpoint**: 基础设施就绪 - 用户故事实现现在可以开始

---

## Phase 3: User Story 1 - 浏览时刻记录 (Priority: P1) 🎯 MVP

**Goal**: 读者可以在博客上浏览已发布的时刻记录，按时间倒序排列

**Independent Test**:
1. 手动创建一个带 `moment` 标签的 GitHub Issue
2. 等待 GitHub Actions 执行完成（检查 Actions 页面）
3. 验证 `source/_data/moments.json` 文件已更新
4. 本地运行 `hexo clean && hexo server`
5. 访问 `/moments/` 页面，验证时刻记录显示正常

### Implementation for User Story 1

- [x] T008 [US1] 创建时刻页面文件 `source/moments/index.md`（包含 Front Matter: title, date, layout; 包含 EJS 模板代码遍历 site.data.moments.moments; 处理空状态显示；渲染 Markdown 内容；显示时间戳）
- [x] T009 [US1] 在 `_config.yml` 添加导航菜单项（menu 部分添加 `时刻: /moments`）<!-- Fluid 主题使用 _config.fluid.yml -->
- [x] T010 [US1] 在 `_config.fluid.yml` 添加导航菜单项（navbar.menu 部分添加 moments 条目，icon: iconfont icon-time-fill）
- [x] T011 [US1] 在 Fluid 主题语言文件 `themes/fluid/languages/zh-CN.yml` 添加翻译（nav.moments: 时刻）<!-- 使用 name 属性直接指定 -->
- [x] T012 [US1] 在时刻页面添加基础 CSS 样式（.moments-timeline, .moment-item, .moment-meta, .moment-content, .empty-state 等类）<!-- 已包含在 index.md 中 -->
- [x] T013 [US1] 实现时间格式化显示（在 index.md 中使用 JavaScript Date 对象格式化 ISO 8601 时间戳为 `zh-CN` 本地格式）<!-- 已包含在 index.md 中 -->
- [x] T014 [US1] 添加空状态处理（当 moments 数组为空时显示"还没有记录任何时刻"提示）<!-- 已包含在 index.md 中 -->

**Checkpoint**: 用户故事 1 应该完全可用且可独立测试

---

## Phase 4: User Story 2 - 快速发布新时刻 (Priority: P2)

**Goal**: 作者可以通过 GitHub Issues 快速发布新的时刻记录，自动同步到博客

**Independent Test**:
1. 在 GitHub 创建新 Issue，内容使用 Markdown 格式
2. 添加 `moment` 标签
3. 提交 Issue
4. 等待 2 分钟，检查 Actions 工作流是否成功执行
5. 验证 `source/_data/moments.json` 已更新
6. 等待 GitHub Pages 部署完成
7. 访问博客 `/moments/` 页面，确认新时刻出现在列表顶部

### Implementation for User Story 2

- [ ] T015 [US2] 验证 GitHub Actions 工作流触发机制（测试创建带 moment 标签的 Issue 是否触发工作流）
- [ ] T016 [US2] 验证 gh CLI 数据提取（检查 Actions 日志，确认成功获取 Issues 数据）
- [ ] T017 [US2] 验证 jq 数据转换（检查生成的 JSON 文件格式是否符合 moments-data.schema.json）
- [ ] T018 [US2] 验证时间戳自动获取（确认 createdAt 和 updatedAt 字段正确来自 GitHub API）
- [ ] T019 [US2] 验证 Markdown 内容渲染（测试粗体、斜体、链接等格式是否正确显示）
- [ ] T020 [US2] 测试完整发布流程（从创建 Issue 到页面显示，全流程计时验证是否在 5 分钟内完成）

**Checkpoint**: 用户故事 1 和 2 现在都应该独立工作

---

## Phase 5: User Story 3 - 编辑和删除已发布时刻 (Priority: P3)

**Goal**: 作者可以编辑或删除（关闭）已发布的时刻记录

**Independent Test**:
1. 选择一个已有的带 `moment` 标签的 Issue
2. 编辑 Issue 内容，保存
3. 等待 Actions 执行，验证 `source/_data/moments.json` 中对应记录的 `updatedAt` 已更新
4. 访问博客验证内容已更新，`createdAt` 保持不变
5. 关闭该 Issue
6. 等待 Actions 执行，验证该记录从 `moments.json` 中移除
7. 访问博客验证该时刻不再显示

### Implementation for User Story 3

- [ ] T021 [US3] 验证编辑功能（编辑 Issue 后检查 Actions 是否重新同步数据）
- [ ] T022 [US3] 验证时间戳更新逻辑（确认编辑后 updatedAt 更新，createdAt 不变）
- [ ] T023 [US3] 可选：在页面显示"已编辑"标记（当 updatedAt !== createdAt 时显示）
- [ ] T024 [US3] 验证删除（关闭）功能（关闭 Issue 后检查 moments.json 中该记录是否移除）
- [ ] T025 [US3] 验证关闭的 Issue 不显示在页面（确认工作流只提取 state:open 的 Issues）

**Checkpoint**: 所有用户故事现在都应该独立可用

---

## Phase 6: Polish & Cross-Cutting Concerns

**Purpose**: 跨用户故事的改进和优化

- [ ] T026 [P] 优化时间显示格式（实现相对时间："2小时前"、"昨天"、"2天前"等，超过7天显示绝对时间）
- [ ] T027 [P] 优化页面样式（调整字体、间距、颜色，确保与博客整体风格一致）
- [ ] T028 [P] 添加响应式设计（确保时刻页面在移动设备上正常显示）
- [ ] T029 [P] 处理边界情况：超长内容（添加 CSS `word-break: break-word` 防止内容溢出）
- [ ] T030 [P] 处理边界情况：特殊字符（验证 Markdown 渲染器正确转义 HTML 标签）
- [ ] T031 验证 quickstart.md 中的步骤（按照快速开始指南完整走一遍流程，确保文档准确）
- [ ] T032 编写简体中文提交信息（所有 Git 提交使用 `feat/fix/docs` 前缀 + 简体中文描述）

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: 无依赖 - 可立即开始
- **Foundational (Phase 2)**: 依赖 Setup 完成 - **阻塞所有用户故事**
- **User Stories (Phase 3-5)**: 所有依赖 Foundational 完成
  - 用户故事之间**相互独立**，可并行实现（如果有多人）
  - 或按优先级顺序实现（P1 → P2 → P3）
- **Polish (Phase 6)**: 依赖所有需要的用户故事完成

### User Story Dependencies

- **User Story 1 (P1)**: Foundational 完成后即可开始 - 无其他故事依赖
- **User Story 2 (P2)**: Foundational 完成后即可开始 - 依赖 US1 的页面用于验证显示
- **User Story 3 (P3)**: Foundational 完成后即可开始 - 依赖 US1 和 US2 有数据可编辑/删除

### Within Each User Story

- User Story 1: 页面文件 → 配置菜单 → 样式 → 时间格式化 → 空状态
- User Story 2: 全部是验证任务，可并行执行
- User Story 3: 全部是验证任务，可并行执行

### Parallel Opportunities

- Phase 1: 所有任务标记 [P] 可并行
- Phase 2: T005-T007 必须顺序执行（工作流 → 验证 → 数据文件）
- User Story 1: T009-T011（配置文件修改）可并行，其他需顺序
- User Story 2: T015-T020 全部可并行（都是验证任务）
- User Story 3: T021-T025 全部可并行（都是验证任务）
- Phase 6: 除 T031 外，所有任务可并行

---

## Parallel Example: User Story 2

```bash
# 并行启动多个验证任务：
Task: "验证 GitHub Actions 工作流触发机制"
Task: "验证 gh CLI 数据提取"
Task: "验证 jq 数据转换"
Task: "验证时间戳自动获取"
Task: "验证 Markdown 内容渲染"

# 最后串行执行完整测试：
Task: "测试完整发布流程"
```

---

## Implementation Strategy

### MVP First (仅 User Story 1)

1. 完成 Phase 1: Setup
2. 完成 Phase 2: Foundational（CRITICAL - 阻塞所有故事）
3. 完成 Phase 3: User Story 1
4. **STOP and VALIDATE**: 手动创建测试 Issue，验证 US1 独立工作
5. 可选：立即部署演示

### Incremental Delivery

1. 完成 Setup + Foundational → 基础就绪
2. 添加 User Story 1 → 独立测试 → 部署/演示（**MVP!**）
3. 添加 User Story 2 → 独立测试 → 部署/演示
4. 添加 User Story 3 → 独立测试 → 部署/演示
5. 每个故事增加价值，不破坏之前的故事

### Parallel Team Strategy

如果有多个开发者：

1. 团队一起完成 Setup + Foundational
2. Foundational 完成后：
   - 开发者 A: User Story 1
   - 开发者 B: User Story 2（需等待 A 完成 US1 的页面用于验证）
   - 开发者 C: User Story 3（需等待 A 和 B 有数据可测试）
3. 故事独立完成并集成

---

## Notes

- [P] 任务 = 不同文件，无依赖
- [Story] 标签将任务映射到特定用户故事，便于追踪
- 每个用户故事应该可独立完成和测试
- 本功能使用手动验证，无自动化测试任务
- 在任何检查点停下来独立验证故事
- 避免：模糊任务、相同文件冲突、破坏独立性的跨故事依赖
- 提交格式：`feat: 添加时刻页面` 或 `fix: 修复时间格式显示`
