# 飞书（Feishu）接入 Hermes 完整配置指南

> 适用版本：Hermes Agent (latest)
> 适配器源码：`gateway/platforms/feishu.py`（~5120 LOC，最成熟的国内平台适配器）
> 最后更新：2026-07-17

## 目录

1. [飞书平台能力概览](#1-飞书平台能力概览)
2. [第一步：创建飞书应用](#2-第一步创建飞书应用)
3. [第二步：配置 Hermes](#3-第二步配置-hermes)
4. [第三步：启动网关](#4-第三步启动网关)
5. [第四步：多会话设置（话题模式）](#5-第四步多会话设置话题模式)
6. [第五步：测试验证](#6-第五步测试验证)
7. [进阶配置](#7-进阶配置)
8. [常见问题](#8-常见问题)

---

## 1. 飞书平台能力概览

飞书是 Hermes 在国内最成熟的平台适配器，具备以下能力：

| 能力 | 支持情况 | 说明 |
|---|---|---|
| **多会话** | ✅ | 群聊 + 话题（Topic）实现 Discord 式的多对话隔离 |
| **交互式卡片** | ✅ | Button / Select / DatePicker / Overflow，支持审批流 |
| **文档工具** | ✅ | 可读取飞书文档内容、管理评论区 |
| **WebSocket** | ✅ | 默认连接模式（推荐），低延迟实时推送 |
| **Webhook** | ⚠️ | 备选模式，需要公网可达的 URL |
| **双向文件** | ✅ | 收发图片、视频、文档 |
| **富文本** | ✅ | Feishu Markdown (`lark_md`) + 交互式卡片 |

**对比 Discord：** 飞书群聊话题 ≈ Discord Thread，群聊 ≈ Discord Channel。飞书独有的优势是文档集成和更丰富的交互式卡片。

---

## 2. 第一步：创建飞书应用

### 2.1 进入开发者后台

访问 [飞书开放平台](https://open.feishu.cn/app)，点击「创建企业自建应用」。

### 2.2 基本信息

- **应用名称：** 随便起（如 "Hermes"）
- **应用图标：** 可选
- **描述：** 可选

### 2.3 添加能力

在「添加应用能力」页面，开启：
- ✅ **机器人**（核心，收发消息）
- ✅ **网页应用**（可选，如果要用 Webhook 模式）

### 2.4 权限配置（重要！）

进入「权限管理」，搜索并开通以下权限：

| 权限 | 用途 | 必须？ |
|---|---|---|
| `im:message` | 收发消息 | ✅ 是 |
| `im:message.p2p_msg` | 私聊消息 | ✅ 是 |
| `im:message.group_msg` | 群聊消息 | ✅ 是 |
| `im:message.group_at_msg` | 群聊@消息 | ✅ 是 |
| `im:message:readonly` | 读取消息内容 | ✅ 是 |
| `im:resource` | 上传/下载文件 | ✅ 是 |
| `im:chat` | 获取群聊信息 | ✅ 是 |
| `im:chat:readonly` | 读取群聊列表 | ✅ 是 |
| `contact:user.employee_id:readonly` | 获取用户 ID（uniq_id 稳定绑定） | ⭐ 推荐 |

**开通后需要发布应用**，点击「创建版本」→ 填写版本号（如 1.0.0）→「发布」。

### 2.5 获取凭证

进入「凭证与基础信息」页面，记录以下信息：

| 凭证 | 位置 | 用途 |
|---|---|---|
| **App ID** | `app_id` | 应用唯一标识 |
| **App Secret** | `app_secret` | API 调用密钥 |
| **Verification Token** | 事件订阅页 | Webhook 模式需要 |

### 2.6 配置事件订阅（WebSocket 模式）

> **推荐使用 WebSocket 模式**，无需公网 IP/域名，本地即可运行。

在「事件订阅」页面：

1. 将请求地址留空（WebSocket 模式不需要 URL）
2. 点击「添加事件」→ 勾选以下事件：

| 事件 | 用途 |
|---|---|
| `im.message.receive_v1` | 接收所有消息 |
| `im.message.reaction.created_v1` | 消息表情回应 |
| `im.chat.disbanded_v1` | 群聊解散通知 |

3. 保存后点击「发布版本」使配置生效。

### 2.7 开启机器人

进入「机器人」页面，确保：
- 机器人模式：**开启**
- 消息类型：✅ 文本消息 ✅ Markdown 消息 ✅ 交互式卡片

---

## 3. 第二步：配置 Hermes

### 3.1 添加环境变量

编辑 `~/.hermes/.env`，添加：

```bash
# ===== 飞书核心凭证（必填）=====
FEISHU_APP_ID=cli_xxxxxxxxxxxx          # 从飞书后台获取
FEISHU_APP_SECRET=xxxxxxxxxxxxxxxx      # 从飞书后台获取

# ===== 飞书域名（一般不用改）=====
FEISHU_DOMAIN=feishu                    # feishu（飞书）/ lark（Lark国际版）

# ===== 连接模式 =====
FEISHU_CONNECTION_MODE=websocket        # websocket（推荐）/ webhook

# ===== Webhook 模式专用（如果用 WebSocket 可跳过）=====
FEISHU_VERIFICATION_TOKEN=xxxxxxxx     # 从飞书后台「事件订阅」获取
FEISHU_ENCRYPT_KEY=xxxxxxxx            # 从飞书后台「事件订阅」获取
FEISHU_WEBHOOK_HOST=0.0.0.0            # Webhook 监听地址
FEISHU_WEBHOOK_PORT=8080               # Webhook 监听端口

# ===== 群聊策略 =====
FEISHU_GROUP_POLICY=allowlist           # open(所有人) / allowlist(白名单) / blacklist(黑名单)
FEISHU_ALLOWED_USERS=ou_xxx            # 白名单用户 open_id，逗号分隔
FEISHU_REQUIRE_MENTION=true             # True = 群内需要 @bot 才能触发（推荐）
FEISHU_ALLOW_BOTS=none                  # none(禁止bot消息) / mentions(允许@) / all(全部)

# ===== Bot 身份（可选，用于调试）=====
FEISHU_BOT_OPEN_ID=ou_xxxx             # Bot 的 open_id（可在飞书后台「应用信息」查到）
FEISHU_BOT_NAME=Hermes                   # Bot 名称
```

### 3.2 配置 config.yaml

编辑 `~/.hermes/config.yaml`，添加 `platforms.feishu` 段：

```yaml
platforms:
  feishu:
    enabled: true
    extra:
      # ---------- 核心凭证（如果不设 env var，可直接写在这里）----------
      app_id: ${FEISHU_APP_ID}
      app_secret: ${FEISHU_APP_SECRET}
      domain: feishu                          # 或 lark（国际版）
      connection_mode: websocket              # websocket 或 webhook

      # ---------- 会话隔离 ----------
      group_sessions_per_user: true           # true = 群内不同用户各自独立会话
      thread_sessions_per_user: false         # false = 话题内所有用户共享会话（协作模式）

      # ---------- 访问控制 ----------
      require_mention: true                   # 群内需 @bot 才触发（推荐）

      # ---------- 可选：按群配置（覆盖全局）----------
      group_rules:
        oc_你的群聊chat_id_1:                 # 此群不需要 @bot
          policy: allowlist
          allowlist:
            - ou_用户open_id_1
          require_mention: false
        oc_你的群聊chat_id_2:                 # 只允许特定用户
          policy: allowlist
          allowlist:
            - ou_用户open_id_1
            - ou_用户open_id_2
```

> **注意：** Hermes 优先从 env var 读取凭证（`extra.get("app_id") or os.getenv("FEISHU_APP_ID")`）。推荐 env var 方式（安全、不与配置一起备份）。

---

## 4. 第三步：启动网关

```bash
# 方式 1：前台运行（调试用，能看到实时日志）
hermes gateway run

# 方式 2：安装为后台 systemd 服务（推荐）
hermes gateway install
hermes gateway start

# 查看状态
hermes gateway status

# 查看平台连接状态（含飞书）
hermes gateway status
# 或在 Hermes 对话中用 /platforms 命令
```

### 4.1 验证飞书连接

启动后，检查日志确认飞书适配器已加载：

```bash
grep -i feishu ~/.hermes/logs/gateway.log | tail -20
```

成功的标志应该类似：
```
[Feishu] WebSocket connected, event types: [...]
[Feishu] Adapter initialized: app_id=cli_xxx
```

### 4.2 添加 Bot 到群聊

1. 在飞书中创建一个群聊
2. 在群设置中 →「群机器人」→「添加机器人」
3. 搜索你创建的应用名，添加

> **如果搜不到：** 去飞书后台「应用发布」→ 确认应用已发布。如果刚发布，等 2-5 分钟生效。

---

## 5. 第四步：多会话设置（话题模式）

飞书的**话题**是 Discord 式多会话的关键。

### 5.1 开启话题

1. 进入飞书群聊 → 点击右上角「···」→ 群设置
2. 开启 **「话题模式」**

### 5.2 用法

开启后：

- **主聊天区** 发消息 → 一个共享会话（所有人可见，任何人 @bot 都在同一个对话中）
- **话题** 发消息 @bot → 每个话题一个独立会话

**示例：**

```
群聊 → 创建话题「bug 排查」
     → 创建话题「周报 draft」
     → 创建话题「部署 plan」

每个话题中 @hermes 提问 → 各自独立上下文，互不干扰
```

### 5.3 会话隔离策略

| 配置 | 值 | 效果 |
|---|---|---|
| `group_sessions_per_user` | `true`（默认） | 群内不同用户各自独立上下文 |
| `thread_sessions_per_user` | `false`（默认） | 话题内多人协作共享上下文 |

**推荐配置（等同于 Discord 话题行为）：**
```yaml
extra:
  group_sessions_per_user: false    # 群主聊区共享（像 Discord 文字频道）
  thread_sessions_per_user: false   # 话题内共享（像 Discord Thread）
```

---

## 6. 第五步：测试验证

### 6.1 私聊测试

在飞书中直接搜索你的应用名，进入与 Bot 的私聊：
- 发送 `你好` → Bot 应回复
- 发送 `/help` → 查看可用命令

### 6.2 群聊测试

在已添加 Bot 的群聊中：
- 发送 `@你的Bot名 你好` → Bot 应回复（需要 @ 触发）
- 发送 `/status` → 查看会话状态

### 6.3 话题测试

1. 在群聊创建话题
2. 在话题中 @bot 发送一个问题
3. 创建另一个话题，发送完全不同的问题
4. 确认两个话题的上下文不会互相污染

### 6.4 交互式卡片测试

在对话中执行需要审批的操作（如特定 shell 命令），飞书端应弹出交互式卡片：

```
┌─────────────────────────────────┐
│ ⚠️ Command Approval Required    │
│                                 │
│   rm -rf /tmp/cache/*           │
│   Reason: 清理临时文件          │
│                                 │
│  [✅ Allow Once] [✅ Session]   │
│  [✅ Always]     [❌ Deny]      │
└─────────────────────────────────┘
```

---

## 7. 进阶配置

### 7.1 群聊无需 @bot（Listen 模式）

特定群不要求 @bot，所有消息都会触发 Hermes：

```yaml
extra:
  group_rules:
    oc_xxxx:
      policy: allowlist
      allowlist:
        - ou_xxx          # 白名单用户
      require_mention: false   # ← 关键：不需要 @
```

### 7.2 文档工具

飞书支持 Hermes 读取飞书文档内容，甚至互操作文档评论区：

开启飞书文档工具：

```bash
hermes tools enable feishu_doc
hermes tools enable feishu_drive
```

然后在 `/reset` 后，Hermes 就能：
- `feishu_doc_read` — 读取指定飞书文档内容
- `feishu_drive_list_comments` — 列出文档评论
- `feishu_drive_add_comment` — 添加文档评论
- `feishu_drive_reply_comment` — 回复文档评论

### 7.3 Webhook 模式（需要公网）

如果必须用 Webhook（比如在内网环境），需要：

1. `FEISHU_CONNECTION_MODE=webhook`
2. 配置 `FEISHU_WEBHOOK_HOST` 和 `FEISHU_WEBHOOK_PORT`
3. 飞书后台「事件订阅」→ 填入 `https://你的域名/webhook/feishu`
4. 填写 `FEISHU_VERIFICATION_TOKEN` 和 `FEISHU_ENCRYPT_KEY`

> **注意：** Webhook 模式需要公网 IP 或域名 + HTTPS + 证书。个人用户推荐 WebSocket 模式。

### 7.4 按群设置不同行为

```yaml
extra:
  group_rules:
    # 技术群：所有消息都监听
    oc_tech_group:
      policy: allowlist
      require_mention: false
      allowlist: [ou_alice, ou_bob]
    # 水群：需要@（默认）
    oc_water_group:
      policy: open
      require_mention: true
    # 管理群：只有@消息，且只允许特定用户
    oc_admin_group:
      policy: allowlist
      allowlist: [ou_carol]
```

---

## 8. 常见问题

### Q: 飞书 Bot 不回复消息
1. 检查应用是否「已发布」
2. 群聊是否添加了机器人
3. 检查 `FEISHU_REQUIRE_MENTION=true` → 必须 @bot
4. 检查 `FEISHU_GROUP_POLICY=allowlist` → 用户是否在白名单

### Q: WebSocket 连接失败
```bash
# 查看日志
grep -i "feishu\|websocket" ~/.hermes/logs/gateway.log | tail -30
```
常见原因：
- App ID / Secret 错误
- 网络无法访问 `https://open.feishu.cn`
- 应用权限未开通

### Q: 会话会丢失上下文吗？
Hermes 默认 `session_reset.mode: none`，**不会自动重置**。所有对话持久化到 SQLite，即使飞书 bot 重启也不丢。只有在达到 token 限制时会自动压缩（保留前3条和后20条消息）。

### Q: 如何获取群聊的 chat_id？
在群聊中 @bot 发送消息后，查看日志：
```bash
grep "chat_id" ~/.hermes/logs/gateway.log | tail -5
```

### Q: Lark（国际版）和飞书的区别？
Lark 和飞书使用同一套 API，只需改 `domain`：
```yaml
domain: lark   # 替代 feishu
```

### Q: 速度怎么样？
WebSocket 模式在 4G/5G 网络下消息延迟 < 200ms，基本等同于飞书原生体验。LLM 推理时间取决于你配置的 provider。

---

## 快速核对清单

| 步骤 | 说明 | 状态 |
|---|---|---|
| 创建飞书应用 | 企业自建应用 | ☐ |
| 添加机器人能力 | 必须开启 | ☐ |
| 开通权限 | im:message 系列 + im:chat + im:resource | ☐ |
| 配置事件 | im.message.receive_v1 | ☐ |
| 发布应用 | 创建版本 → 发布 | ☐ |
| 获取凭证 | App ID + App Secret | ☐ |
| 添加 .env | FEISHU_APP_ID + FEISHU_APP_SECRET | ☐ |
| 配置 config.yaml | platforms.feishu 段 | ☐ |
| 启动网关 | `hermes gateway install && hermes gateway start` | ☐ |
| 群聊添加 Bot | 群设置 → 机器人 | ☐ |
| 开启话题模式 | 群设置 → 话题模式 | ☐ |
| 测试私聊 | @bot 你好 | ☐ |
| 测试群聊 | @bot 你好 | ☐ |
| 测试话题 | 创建话题 @bot | ☐ |
