# 飞书 + Hermes 项目管理完全指南

> **永久地址:** https://grootwu55-code.github.io/hermes-feishu-pm-guide/
> **版本:** v1.0 · 最后更新: 2026-07-29

---

## 概述

本指南教你如何通过飞书 + Hermes 实现 **AI 驱动的项目管理**：

- 🗂️ 每个飞书话题 = 一个项目，上下文独立隔离
- 🤖 Hermes 自动维护项目进度、需求、任务、Bug 用例库
- 🚀 基于项目资料驱动 Hermes 完成开发任务
- 👥 团队成员可共同参与（人工编辑 + 各自 Hermes Bot）

---

## 一、架构总览

```
飞书群聊 "产品团队"
│
├── 🧵 话题 "项目A：用户管理系统"
│   ├── Hermes 自动绑定 → ~/projects/user-mgmt/
│   │   ├── project.md          ← 项目元信息 + 当前状态
│   │   ├── requirements.md     ← 需求清单（字典化字段管理）
│   │   ├── tasks.md            ← 任务看板
│   │   ├── bugs.md             ← Bug 用例库（每修一个 Bug 登记一条）
│   │   ├── design/             ← 设计文档
│   │   └── notes/              ← 进度日志
│   └── Bitable 仪表盘（可选，只读同步）
│
├── 🧵 话题 "项目B：数据分析平台"
│   └── Hermes 绑定 → ~/projects/data-platform/
│
└── 🧵 话题 "项目C"
    └── Hermes 绑定 → ~/projects/project-c/
```

**设计原则（第一性原理 + 剃刀）：**

| 决策 | 理由 |
|---|---|
| Git + Markdown 作为 Source of Truth | 零新代码依赖，Hermes 现有工具全覆盖，天然版本控制 + 冲突解决 |
| Bitable 作为可选的只读仪表盘 | 满足"在飞书里看到表格"的需求，但不作为主数据源（避免双向同步冲突） |
| 每个项目一个话题 | 利用飞书话题的天然隔离，无需额外路由逻辑 |
| project.md 驱动一切 | 单文件绑定，Hermes 进入话题后加载 200 字摘要即可恢复全部上下文 |

---

## 二、准备工作

### 2.1 飞书 Bot 配置

> 详细飞书 Bot 配置请参考：[飞书接入 Hermes 完整配置指南](./docs/feishu-setup.md)

核心步骤：

1. 在 [飞书开放平台](https://open.feishu.cn/app) 创建企业自建应用
2. 开启机器人能力 + 开通消息权限（`im:message`, `im:chat`, `im:resource`）
3. 配置事件订阅（WebSocket 模式，无需公网 IP）
4. 获取 **App ID** 和 **App Secret**

### 2.2 Hermes 配置

在 `~/.hermes/.env` 中添加：

```bash
FEISHU_APP_ID=cli_xxxxxxxxxxxx
FEISHU_APP_SECRET=***
```

在 `~/.hermes/config.yaml` 中添加：

```yaml
platforms:
  feishu:
    enabled: true
    extra:
      app_id: ${FEISHU_APP_ID}
      app_secret: ${FEISHU_APP_SECRET}
      connection_mode: websocket
      require_mention: true
      # 话题内多会话：共用上下文（协作模式）
      group_sessions_per_user: false
      thread_sessions_per_user: false
```

启动网关：

```bash
hermes gateway install
hermes gateway start
```

### 2.3 安装自动化 Skill

Hermes 的项目管理自动化 Skill 位于 `~/.hermes/skills/devops/feishu-project-management/`，会在进入飞书话题时自动触发。

确认 Skill 已安装：

```bash
hermes skills list | grep feishu-project
```

### 2.4 创建项目模板

```bash
ls ~/projects/_template/
# project.md  requirements.md  tasks.md  bugs.md  design/  notes/
```

---

## 三、日常使用

### 3.1 创建新项目

在飞书中创建一个**群聊**，然后创建**话题**：

```
群聊 → 右上角 "···" → 开启「话题模式」 → 创建话题「项目A：用户管理系统」
```

在话题中 @Hermes：

```
@Hermes 这是新项目：用户管理系统。
需求是给公司内部做一个员工信息的增删改查系统，
技术栈用 Python FastAPI + Vue3。
帮我初始化项目。
```

Hermes 会自动：
1. 在 `~/projects/` 下创建 `user-mgmt/` 目录（从模板复制）
2. 填充 `project.md`（项目名、状态、描述、日期）
3. 填充 `requirements.md`（把你的需求结构化录入）
4. 初始化 Git 仓库
5. 保存话题→项目绑定到 memory

### 3.2 更新进度

在话题中自然对话即可，Hermes 会自动识别并更新：

| 你说的话 | Hermes 自动操作 |
|---|---|
| "完成了用户列表接口" | 在 `tasks.md` 将该任务标记为已完成，在 `notes/2026-07-29.md` 添加进度日志 |
| "新需求：要支持批量导入" | 在 `requirements.md` 追加 REQ-003，自动分配 ID |
| "登录接口有个 bug" | 在 `bugs.md` 添加 BUG-002，记录描述和发现日期 |
| "开始做前端页面" | 在 `tasks.md` 将该任务移至进行中，更新 `project.md` 状态 |

查看项目状态：

```
@Hermes 项目状态怎么样了？
```

Hermes 会读取 `project.md` + `tasks.md`，回复当前进度摘要。

### 3.3 驱动开发

当项目资料充分后，直接让 Hermes 开发：

```
@Hermes 按照 requirements.md 的 REQ-001 和 REQ-002，帮我实现用户管理 API。
设计文档在 design/architecture.md，记得写测试。
```

Hermes 会：
1. 读取 `requirements.md` + `design/architecture.md` 获取完整上下文
2. 调用 Claude Code 执行开发
3. 遵循你的编码规范（第一性原理 + 剃刀、对抗性测试、Bug 用例库）
4. 完成后运行测试验证
5. 更新 `tasks.md` 标记任务完成

### 3.4 查看项目文档

```
@Hermes 给我看看这个项目的完整需求
@Hermes 有哪些 Bug 还没修？
@Hermes 最近一周的进度日志
```

---

## 四、团队协作

> 📖 完整团队协作指南请阅读：[团队成员接入指南](./team-collaboration.md)

### 4.1 协作模型

```
           Git 仓库（Source of Truth）
                    ↕
    ┌───────────────┼───────────────┐
    ▼               ▼               ▼
  你              团队成员A       团队成员B
  (飞书话题A)    (飞书话题B)     (GitHub PR)
  Hermes 自动    人工编辑md     Her own Hermes
  写 project.md  写 requirements  也绑定同项目
```

**Git 是唯一的真相来源。** 所有修改无论来自 Hermes 还是人工，都通过 Git commit 记录，永远不会冲突丢数据。

### 4.2 团队成员如何参与

**方式 1：直接在 GitHub 编辑文件**（最简单）

团队成员不需要任何配置，直接访问项目仓库的 Markdown 文件：

- 编辑 `requirements.md` 添加需求
- 编辑 `tasks.md` 更新任务状态
- 编辑 `bugs.md` 报告 Bug

Hermes 下次进入话题时会自动 `git pull` 同步最新变更。

**方式 2：用自己的 Hermes Bot**（推荐）

每个团队成员如果也有 Hermes，可以绑定到同一个项目：

1. 把自己的 Hermes Bot 添加到飞书群
2. 创建自己的话题（如「项目A - Alice 的工作区」）
3. 在话题中 @自己的Bot，让它绑定到同一个 Git 仓库：

```
@Alice的Hermes 绑定到项目 ~/projects/user-mgmt/，这是我的工作区
```

4. Alice 的 Hermes 会 `git pull` 加载项目上下文，之后可独立工作

> ⚠️ 注意：两个 Hermes 不要同时编辑同一个文件。如果发生冲突，Git 会标记冲突行，人工解决即可。

**方式 3：发布到飞书群公告**

Hermes 可以定期推送项目摘要到飞书群主聊天区：

```
@Hermes 把项目进度摘要发到群公告
```

### 4.3 多人协作最佳实践

| 场景 | 推荐方式 |
|---|---|
| 产品经理提需求 | GitHub 编辑 `requirements.md` → Hermes 自动 pull |
| 开发人员认领任务 | 编辑 `tasks.md` 把自己加到"分配"列 |
| QA 报告 Bug | GitHub 编辑 `bugs.md` |
| 每日站会 | @Hermes 在话题中口述，Hermes 更新 tasks.md + notes |
| 代码审查 | GitHub PR（Claude Code 生成的代码自动创建 PR） |

---

## 五、Bitable 仪表盘（可选）

### 5.1 概述

如果你希望在飞书中看到一个可视化的任务看板/进度表，可以配置 Bitable 作为**只读仪表盘**。

```
Git 仓库（真相来源）
    │
    │ Hermes 单向同步
    ▼
Bitable 多维表格（只读仪表盘）
    │
    │ 团队成员在飞书中查看（不编辑）
    ▼
飞书群聊中的 Bitable 卡片
```

> ⚠️ **Bitable 是只读镜像。** 所有修改必须在 Git 中进行。这避免了双向同步的冲突。

### 5.2 配置 Bitable

1. 在飞书中创建一个多维表格
2. 创建以下表：

| 表名 | 对应文件 | 用途 |
|---|---|---|
| 任务看板 | `tasks.md` | 任务状态可视化 |
| 需求清单 | `requirements.md` | 需求总览 |
| Bug 追踪 | `bugs.md` | Bug 状态 |
| 进度日志 | `notes/` | 最近更新 |

3. 在 `project.md` 中添加 Bitable 的 app_token：

```markdown
## Bitable
- app_token: BQRMbcxxxxxx
- 用途: 只读仪表盘，由 Hermes 自动同步
```

4. 启用 Bitable 工具：

```bash
hermes tools enable feishu_doc  # Bitable 工具注册在此 toolset 下
```

5. Hermes 在每次更新 Git 后，会检查是否有 Bitable 配置，如有则运行单向同步。

### 5.3 Bitable 工具命令

在飞书话题中：

```
@Hermes 把 tasks.md 同步到 Bitable
@Hermes bitable 上显示一下所有未完成的 Bug
@Hermes 更新 Bitable 仪表盘
```

---

## 六、项目数据结构

### project.md（关键文件）

```markdown
# 项目名
## 当前状态
| **阶段** | 开发 / 测试 / 上线 |
| **进度** | 60% |
| **优先级** | P1 |

## 项目概述
一句话描述

## 关键链接
- 代码仓库: https://github.com/.../repo
- 飞书话题: oc_xxx / thread_yyy
- Bitable: BQRMbcxxxxxx
```

### requirements.md

```markdown
| ID | 需求 | 验收标准 | 状态 | 备注 |
|---|---|---|---|---|
| REQ-001 | 用户登录 | 能通过账号密码登录 | ✅ 已完成 | |
| REQ-002 | 用户列表 | 分页展示用户 | 🚧 开发中 | |
| REQ-003 | 批量导入 | 支持 CSV 导入 | 📋 待开始 | |
```

### tasks.md

```markdown
## 进行中
| T-003 | 实现批量导入接口 | REQ-003 | 4h | Alice |
## 已完成
| T-001 | 搭建项目框架 | REQ-001 | 2h | Alice |
| T-002 | 实现用户列表接口 | REQ-002 | 3h | Alice |
```

### bugs.md

```markdown
## 未修复
| BUG-002 | P2 | 登录后 token 未刷新 | 登录后等 30 分钟再请求 |
## 已修复
| BUG-001 | P1 | 用户列表返回 500 | 缺少分页参数默认值 | 7/28 | ✅ 回归通过 |
```

---

## 七、原理说明

### 为什么 Git + Markdown 而不是纯 Bitable？

> 从第一性原理出发：

1. **Hermes 的核心能力是读写文件**（`read_file`, `write_file`, `patch`），不是调用 Web API。用文件系统是最短的路径。

2. **Git 天然解决了所有协作问题**：版本历史、冲突检测、回滚、分支隔离——这些都是项目管理刚需。如果用 Bitable 做唯一数据源，你需要自己实现这些。

3. **Bitable 的实时协同编辑模型与 AI 异步会话模型冲突。** Hermes 的一次对话可能持续数分钟，期间用户手动编辑 Bitable 会造成版本不一致。Git 的 commit-based 模型完美解决这个问题——Hermes 编辑时 lock 的是本地文件，最终通过 commit 合并。

4. **剃刀原理**：Markdown 文件可以被任何人、任何工具读写。Bitable 需要飞书 API、lark_oapi SDK、自定义工具。复杂度高 10 倍，但功能没有本质提升。

### Bitable 的正确角色

Bitable 是 **展示层**，不是 **数据层**。它让不看 Git 的团队成员有一个直观的仪表盘，但所有修改必须经过 Git。这是一个单向同步管道：

```
[Git 仓库] --单向同步--> [Bitable 仪表盘]
     ↑                        ↓
  Hermes 编辑             团队成员查看
  成员编辑                  （不通过这里修改）
```

---

## 八、常见问题

**Q: 如果我没有飞书，能用这个方案吗？**

A: 核心机制（Git + Markdown + Hermes）与飞书无关。你在 Discord、Telegram、CLI 中都可以使用同样的项目模板和 Skill。飞书只是会话载体。

**Q: 一个项目可以有多个话题吗？**

A: 可以。创建话题「项目A-后端」和「项目A-前端」，都绑定到同一个 `~/projects/project-a/` 目录。它们共享 Git 仓库的数据，但各自有独立的会话上下文。

**Q: Bitable 同步频率是多少？**

A: 每次 Hermes 更新 Git 文件后自动触发。如果 Bitable API 调用失败，不影响 Git 数据（Git 永远是 source of truth），Hermes 会在下一次会话重试。

**Q: 团队成员的 Hermes 怎么知道绑定哪个项目？**

A: Memory。Hermes 会记住话题→项目的绑定。团队成员可以让自己的 Hermes "加入"已有项目。

---

## 九、相关链接

- 代码仓库: [grootwu55-code/hermes-feishu-pm-guide](https://github.com/grootwu55-code/hermes-feishu-pm-guide)
- 飞书 Bot 配置详情: [./docs/feishu-setup.md](./docs/feishu-setup.md)
- Hermes 官方文档: [hermes-agent.nousresearch.com](https://hermes-agent.nousresearch.com/docs/)
- 项目模板: `~/projects/_template/`
- 自动化 Skill: `~/.hermes/skills/devops/feishu-project-management/`
