# Claude-code-workflows
Claude code 工作流
# Claude Code 任务编排工作流对比文档
# 很多都是设想架构做没做我也不知道，很久没去维护了
> 路径：`C:\Users\30823\.claude\workflows\`
> 更新时间：2026-05-21

---

## 概览

两个工作流均为 **多 Agent 任务编排系统**，通过 `/orchestrate <任务描述>` 指令在 Claude Code 中触发，将复杂任务分解后由多个隔离 Agent 并行执行，最终汇总审查归档。

| 工作流 | Skill 名 | 定位 |
|--------|---------|------|
| `task-orchestration`（v1） | `orchestrate` | 基础版，轻量易用 |
| `task-orchestration-v2`（v2.1） | `workflowsv2.1` | 企业级，功能完备 |

---

## task-orchestration（v1）

### 用途

适用于**中等复杂度任务**，需要将一个功能模块拆分为多个子任务并行执行，但不需要前后端协同论证或跨窗口调度。

**典型场景：**
- 实现一个独立功能模块（如用户认证、文件上传）
- 重构某个服务或组件
- 快速验证多任务编排流程

### 核心流程（4个阶段）

```
Phase 1: Orchestrator  →  分解任务，生成 task-framework.md
Phase 2: Worker × N   →  隔离 worktree 中并行执行子任务，输出 task-report-N.md
Phase 3: Reviewer      →  审查所有 Worker 输出，验证一致性，输出 review-report.md
Phase 4: Archiver      →  交叉验证、压缩记忆、生成 project-spec.md
```

### 关键特性

- **Worker 隔离**：每个 Worker 运行于独立 worktree（独立文件系统 + git 状态 + 记忆上下文）
- **并行执行**：最多 4 个 Worker 并行
- **通用 Agent**：所有 Worker 使用 `general-purpose` 类型，无专业分工
- **简单配置**：质量阈值 7 分，安全阈值 8 分

### 输出文件

| 文件 | 说明 |
|------|------|
| `task-framework.md` | 任务分解框架 |
| `task-report-N.md` | 各 Worker 执行报告 |
| `review-report.md` | 审查报告 |
| `project-spec.md` | 最终归档规范 |

### 文件结构

```
task-orchestration/
├── skill.md          # Skill 执行入口
├── README.md
├── config.json
├── OVERVIEW.md
├── QUICKSTART.md
├── prompts/          # Orchestrator / Worker / Reviewer / Archiver 提示词
├── templates/        # 规范文档模板
└── examples/         # 使用示例
```

---

## task-orchestration-v2（v2.1）

### 用途

适用于**复杂全栈项目**或**多人协作级别**的开发任务，需要前后端协同论证、跨窗口并行、状态同步和专业 Agent 路由。

**典型场景：**
- 全栈功能开发（前端 + 后端 + 数据库）
- 涉及多个独立模块的大型需求
- 需要接口契约对齐的前后端协作
- 代码债务检测与历史因子复用

### 完整流程（8步）

```
PHASE 0: 前置检查
    复杂度评分 ≤ 5  →  直接执行，跳过编排
    复杂度评分 > 5  →  进入 Orchestrator

PHASE 1: Orchestrator
    复杂度评估 → 架构论证（新项目/新模块）→ 功能论证派发

PHASE 1.5: 功能论证（核心新增）
    frontend-dev  →  生成功能清单（.claude/memory/decisions/[功能名]-功能论证.md）
    backend-dev   →  生成 API 契约（.claude/memory/decisions/[功能名]-API契约.md）
    reviewer      →  产出 alignment.json（status = aligned 才允许开始实现）

PHASE 2: Worker Pool
    智能路由 → 并行执行子任务 → 跨窗口容量调度扩容

PHASE 3: Reviewer
    质量阈值检查 + 功能链完整性 + 前后端接口一致性 + specialized 输出契约

PHASE 4: Archiver
    归档到 .claude/rules/project-spec.md + 更新项目长期记忆
```

### 核心新增能力（v1 → v2.1）

#### 1. 功能论证阶段

每个前端任务自动检查以下要素，防止遗漏：

| 类别 | 必需功能 |
|------|----------|
| CRUD | 增 / 删 / 改 / 查 |
| 选择 | 单选 / 多选 |
| 交互 | 删除确认弹窗、撤销/重做 |
| 状态 | 骨架屏 / Loading / 空状态 / 错误提示+重试 |
| 体验 | 自动保存 / 快捷键 / 响应式适配 |

#### 2. 专业 Agent 智能路由

根据任务关键词自动派遣：

| 关键词 | 派遣 Agent | 职责 |
|--------|-----------|------|
| 前端 / UI / 组件 / react / vue | `frontend-dev` | 功能论证 + 前端实现 |
| 后端 / API / 接口 / node / python | `backend-dev` | API 契约 + 后端实现 |
| bug / 修复 / 错误 / 崩溃 | `bug-fixer` | 复现 → 定位 → 修复 → 验证 |
| 数据库 / SQL / 迁移 | `db-admin` | 迁移 / 回滚 / 验证 |
| 安全 / 漏洞 / 渗透 | `security-expert` | 风险分级 / 修复建议 |

#### 3. 跨窗口并行（容量模型）

```
窗口池配置：
  main      → 容量 4（orchestrator + worker）
  window-2  → 容量 2（worker-host）
  window-3  → 容量 2（worker-host）
  window-4  → 容量 2（worker-host）

调度逻辑：
  if 可用槽位 < 待调度任务数 AND 当前窗口数 < maxWindows:
      powershell -File scripts/spawn-window.ps1
  else:
      复用已有窗口
```

#### 4. 状态三文件协议

| 文件 | 职责 |
|------|------|
| `framework.json` | 静态任务框架（只写一次） |
| `assignments.json` | 调度账本（认领 / lease / 窗口分配） |
| `state.json` | 活体运行状态（心跳 / 进度 / 窗口在线） |

#### 5. 向量记忆引擎（ChromaDB）

解决"Agent 只会写新代码不会删旧代码"的问题：

```bash
# 检索历史相似任务
/factor-search <关键词>

# 扫描"被替代但未清理"的代码债务
/debt-check

# 标记旧模块为待清理
/deprecate <旧模块路径>
```

### 关键脚本（PS1）

| 脚本 | 用途 |
|------|------|
| `init-workspace-state.ps1` | 初始化工作区（framework / assignments / state） |
| `spawn-window.ps1` | 容量判断 + 启动新窗口 |
| `register-window.ps1` | 新窗口注册到窗口池 |
| `release-window.ps1` | 释放窗口槽位 |
| `window-health-check.ps1` | 扫描并清理 stale / offline 窗口 |
| `heartbeat.ps1` | Worker 心跳刷新 |
| `claim-task.ps1` | Worker 认领任务 |
| `release-task.ps1` | Worker 释放任务（完成/失败） |
| `reconcile-state.ps1` | 汇总 assignments / reports / windows 状态 |
| `debt-check.ps1` | 代码债务扫描 |
| `search-similar.ps1` | 向量语义检索 |

### 输出文件

| 文件路径 | 说明 |
|----------|------|
| `.claude/workspace/framework.json` | 静态任务结构 |
| `.claude/workspace/assignments.json` | 调度账本 |
| `.claude/workspace/state.json` | 活体运行状态 |
| `.claude/workspace/reports/*.json` | Worker 子任务报告 |
| `.claude/workspace/contracts/*.json` | 前后端对齐产物 |
| `.claude/workspace/review.json` | 审查报告 |
| `.claude/memory/decisions/*.md` | 功能论证 + API 契约文档 |
| `.claude/rules/project-spec.md` | 最终归档规范 |

### 文件结构

```
task-orchestration-v2/
├── skill.md                          # Skill 执行入口（workflowsv2.1）
├── README.md
├── config.json
├── agents.manifest.json              # Agent 统一清单
├── architecture-v2.1.svg             # 完整架构图
├── architecture-v2.1-simple.svg      # 简化架构图
├── docs/
│   ├── state-sync.md                 # 状态同步协议
│   ├── cross-window-execution.md     # 跨窗口执行规范
│   ├── agent-catalog.md              # Agent 角色目录
│   ├── specialized-agent-coordination.md
│   ├── factor-memory-system.md       # 因子化记忆设计
│   └── vector-memory-engine.md       # 向量记忆引擎
├── prompts/                          # 各角色提示词
├── schemas/                          # JSON Schema 定义（7个）
├── scripts/                          # PS1 脚本（13个）
├── tools/
│   └── vector-memory.py              # ChromaDB 向量引擎核心
├── templates/                        # 任务/报告/规范模板
└── memory/
    └── architecture/                 # 前端/后端/数据库架构规范
```

---

## 横向对比

| 维度 | v1 | v2.1 |
|------|-----|------|
| **触发方式** | `/orchestrate` 手动触发 | `/orchestrate` + 自动复杂度评估（评分 >5 才编排）|
| **适用规模** | 中等复杂度，单领域任务 | 复杂全栈，多模块协作任务 |
| **功能论证** | ❌ | ✅ 前后端功能清单 + API 契约 + alignment.json |
| **Agent 类型** | 通用 Worker | 5 种 specialized agent + 智能路由 |
| **并行调度** | 简单 worktree 隔离 | 容量模型 + 窗口池 + PS1 脚本管理 |
| **跨窗口执行** | ❌ | ✅ 最多 4 窗口 × 各 2 槽位 |
| **状态同步** | ❌ | ✅ framework / assignments / state 三文件协议 |
| **向量记忆** | ❌ | ✅ ChromaDB 因子检索 + 代码债务检测 |
| **额外命令** | 无 | `/debt-check` `/factor-search` `/deprecate` |
| **文件体量** | 13 个文件 | 53 个文件 |
| **上手难度** | 低 | 中高（需理解状态协议和容量模型）|

---

## 选择建议

```
任务预估步骤 ≤ 3 步，单文件修改，简单查询
    → 不使用编排，直接执行

任务预估步骤 4~8 步，单领域（纯前端或纯后端）
    → task-orchestration（v1）
    → 命令：/orchestrate <任务描述>（Skill: orchestrate）

任务涉及前后端协作，或多模块并行，或需代码债务管理
    → task-orchestration-v2（v2.1）
    → 命令：/orchestrate <任务描述>（Skill: workflowsv2.1）
```

---

## 快速启动命令

```bash
# v1 基础编排
/orchestrate 实现用户认证模块，包含注册、登录、密码重置功能

# v2.1 完整编排
/orchestrate 实现工作流编辑器，支持节点拖拽、连线、保存和导出

# v2.1 附加命令
/debt-check                          # 扫描代码债务
/factor-search auth module           # 检索相似历史因子
/deprecate src/old-auth/index.ts     # 标记旧模块待清理
```
