# 团队成员接入指南

> **目标：** 让团队成员通过飞书 + Hermes（或 GitHub）参与到项目管理中
> **难度：** 从零配置到全员协作，30 分钟内完成

---

## 角色与接入方式

| 角色 | 推荐方式 | 需要什么 | 能做什么 |
|---|---|---|---|
| **你（项目 Owner）** | 飞书话题 + Hermes | 飞书 Bot 已配好 | 全部：创建项目、更新数据、驱动开发 |
| **技术成员** | 飞书话题 + 自己的 Hermes | 飞书 Bot + 本地 Hermes | 在自己的话题中独立工作，共享 Git 数据 |
| **非技术成员（PM/QA）** | GitHub 网页编辑 | GitHub 账号 | 编辑需求、报告 Bug、更新任务状态 |
| **只读成员** | 飞书 Bitable | 无 | 查看仪表盘，了解项目状态 |

---

## 方式 1：GitHub 网页编辑（无配置，即开即用）

**适合：** 产品经理、QA、不需要 Hermes 的团队成员

### 步骤

1. 打开项目仓库（如 `https://github.com/grootwu55-code/my-project`）
2. 点击要编辑的文件：
   - 提需求 → `requirements.md` → 点击 ✏️ 编辑
   - 报告 Bug → `bugs.md` → 点击 ✏️ 编辑
   - 更新任务 → `tasks.md` → 点击 ✏️ 编辑
3. 修改内容，填写 Commit message（如 "新增批量导入需求"）
4. 点击「Commit changes」→ 选择「Create a new branch」→「Propose changes」

### 效果

```
团队成员编辑 GitHub
    │
    │ git push
    ▼
Git 仓库更新
    │
    │ Hermes git pull
    ▼
你的飞书话题中，Hermes 自动加载最新数据
```

### 示例

> 产品经理小王在 GitHub 上编辑 `requirements.md`，添加了一行：
> `| REQ-005 | 数据导出 | 支持导出 Excel 和 CSV | 📋 待确认 | 小王 |`
>
> 你在飞书话题中 @Hermes：
> "有新需求了，看看 requirements.md"
>
> Hermes 会 `git pull` 并回复：
> "检测到新需求 REQ-005：数据导出（支持 Excel/CSV），来自小王。要现在评估排期吗？"

---

## 方式 2：自己的 Hermes Bot（完整能力）

**适合：** 开发人员，想用自己的 AI 助手工作

### 前提条件

团队成员需要：
- 本地或服务器上安装了 Hermes
- 配置了自己的飞书 Bot（App ID + App Secret）

### 配置步骤

#### Step 1: 创建自己的飞书 Bot

每个团队成员在 [飞书开放平台](https://open.feishu.cn/app) 创建自己的应用，流程与主 Bot 相同（详见 [飞书接入指南](./feishu-setup.md)）。

#### Step 2: 配置 Hermes

```bash
# 在成员自己的机器上
# ~/.hermes/.env
FEISHU_APP_ID=cli_成员的AppID
FEISHU_APP_SECRET=成员的secret
```

```yaml
# ~/.hermes/config.yaml
platforms:
  feishu:
    enabled: true
    extra:
      app_id: ${FEISHU_APP_ID}
      app_secret: ${FEISHU_APP_SECRET}
      connection_mode: websocket
```

#### Step 3: 加入项目

1. 把成员的 Bot 添加到飞书群聊
2. 成员在话题中 @自己的Bot：

```
@我的Hermes 绑定到项目 ~/projects/user-mgmt/，这是我的开发工作区
```

3. 成员的 Hermes 会自动 `git clone` 项目仓库，加载上下文

#### Step 4: 日常工作

成员在自己话题中工作，与你的主话题完全隔离，但共享同一个 Git 仓库：

```
# 成员的飞书话题 "项目A - Alice 的开发工作区"
@我的Hermes 我要认领 REQ-003 批量导入功能
@我的Hermes 批量导入接口写好了，帮我提交 PR
@我的Hermes 代码审查通过了吗？
```

### 冲突处理

如果两人同时编辑了同一个文件：

```bash
# Git 会自动标记冲突
<<<<<<< HEAD
| T-005 | 增加日志 | 2h | Alice |
=======
| T-005 | 增加监控 | 3h | Bob |
>>>>>>> feature/bob-update

# 人工选择保留哪个，或合并两个版本
```

Hermes 不会静默覆盖——Git 保证冲突是可见的。

---

## 方式 3：飞书群 + 公告（轻量协作）

**适合：** 只想看进度、不想编辑数据的成员

1. 你在飞书话题中 @Hermes：

```
@Hermes 把项目进度生成摘要，发到群公告
```

2. Hermes 会生成类似：

```
📊 **项目「用户管理系统」周报**
✅ 本周完成：REQ-001 登录, REQ-002 用户列表
🚧 进行中：REQ-003 批量导入 (Alice, 60%)
🐛 未修 Bug：2 个 (P1: 1, P2: 1)
📅 预计下周：完成批量导入，开始前端页面
```

3. Hermes 通过 `send_message` 发到飞书群主聊天区（需要配置 home channel）

---

## 权限控制

### Git 仓库权限

| 角色 | 权限 |
|---|---|
| 项目 Owner | Admin（仓库管理 + 分支保护） |
| 技术成员 | Write（可 push 到 feature 分支） |
| PM/QA | Write（可 push 到 feature 分支） |
| 只读成员 | Read（只能看） |

### 飞书群权限

在飞书群设置中配置：
- 哪些成员可以添加 Bot
- 哪些成员可以在话题发言
- 是否需要 @Bot 才触发（默认 `require_mention: true`）

---

## 快速参考

| 我想... | 怎么做 |
|---|---|
| 提新需求 | GitHub 编辑 `requirements.md` 或飞书话题中说"新需求: ..." |
| 认领任务 | GitHub 编辑 `tasks.md` 把自己加到"分配"列 |
| 报告 Bug | GitHub 编辑 `bugs.md` 或飞书话题中说"发现一个 Bug" |
| 查看所有人的进度 | GitHub 看 `tasks.md` 或 `notes/` |
| 让我的 Hermes 开发 | 在自己的话题中 @Bot 说"按照 requirements.md 开发 XXX" |
| 同步最新变更 | `git pull`（自动）或 @Hermes "拉取最新" |
