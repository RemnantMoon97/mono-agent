
- 状态：confirmed design；实现已授权
- 日期：2026-08-26
- 关联条款：AKC-001～AKC-003、MOB-001～MOB-008、SES-001～SES-008、MIG-001～MIG-002、WSP-003

## 1. 目标与边界

三件事：

1. Core 只注册一个内建 `akashic` Channel。

```text
Web transport ───┐
                 ├─ AkashicChannel ── existing Session / Message / Turn
```

本规格不重做 Session、Message、Turn、附件、模型、流式协议、Turn 控制、通知、客户端本地
平台效果继续由各自现有实现拥有。

## 2. 为什么只新增一个组合 owner

`ChannelHost`，再分别发布 `CoreChannelDefinition`。Committed Channel catalog 禁止两个
Core definition 使用同名 Channel，所以不能只把两者的 `name` 都改成 `akashic`。

目标是一个很薄的组合 owner：

```text
AkashicChannel
├─ WebAdapter      现有浏览器认证、HTTP/WebSocket、上传与呈现
```

`AkashicChannel` 只拥有一次 Channel 注册、共同的 `channel/chat_id` 路由和两个 adapter 的
启动/停止。它复用现有 `Channel`、`ChannelContext`、Core `ChannelAdapter`、SessionStore 和
EventBus，不增加 Port、客户端框架、共同 wire schema 或共同 reducer。

## 3. 身份与可观察行为

唯一身份是：

```text
channel     = "akashic"
chat_id     = "<bare canonical id>"
session_key = "akashic:<chat_id>"
```

- `channel` 与 `chat_id` 继续使用通用 Channel 合同；`chat_id` 不包含 `akashic:` 前缀。
  只增加这个既有操作。持久 Session 仍由首次消息提交路径创建，不增加空 Session 生命周期。
- 当前选择、草稿、已读位置和 UI 设置仍是各端本地事实，不同步导航。
- 历史分页、实时流、停止、并发、模型和附件继续服从现有 owner；本变更只让两个 adapter
  使用同一 `session_key`，不重新规定这些能力。
- endpoint/device identity 只留在各 adapter 的认证、去重和诊断内部，不进入 Session 身份。

同一设备可以进入多个 Session；旧 device→last-session identity rows 在迁移中退役。

## 4. 必须消除的一处旧耦合

输入误走成普通 Web 输入。

这些 channel-name 特判改为读取已有 handoff marker/owner；不增加新的耐久协议，也不让

## 5. Schedule、Proactive 与 Akasha

  改为 `{"channel": "akashic", "chat_id": "<new id>"}`，不增加 `target_session_id`。
- Wake 的投递目标与工作所属 Session 是两个独立字段：分别迁移 `channel/recipient` 与
  `session_id`。Content、Drift 和 durable delivery 中真实保存的 accepted/selected Session
  引用使用同一张映射表。
- Wake、Scheduler 与 `message_push` 面向 `akashic` 发送完整消息时，共享 dispatcher 先向
  目标 Session 幂等追加一条 assistant Message。SessionDB 是唯一正文 owner，分配 canonical
  `message_id + seq`，并保存 `effects.post_commit = suppress`。
- 两个 adapter 都会被调用，但只负责更新提示。Web 收到带 `session_message_id` 的 final 后
  拉取缺尾。adapter 离线、拒绝或结果未知不回滚 Session Message。
  独立 history cursor。Realtime ACK cursor 只保留传输重放职责。
- Akasha 算法不变。因为 sidecar 保存 `session_key`，Session rekey 后调用现有
  ordinary Akasha 插件的显式 `/akasha_reindex confirm` 流程从 SessionDB 固定输入备份并重建。
- 退役 `memory2.db` 归档不导入、不改写、不删除；0041 的 Turn effect 合同不在本规格重述。

## 6. 一次性 breaking 迁移

### 6.1 前置条件

迁移在维护窗口停止全部 workspace writer，取得独占锁并创建可验证完整备份。
receipt/import 必须先收束到迁移工具明确支持的状态；不能证明安全时 fail-loud，什么都不改。

cursor、WebUI cache 和非 Session 设置；Room 删除旧 Session 图、outbox、附件传输和 pending
通知/stop。Core 给每个有效设备追加 `sync.reset_required`，客户端从该准确 event sequence

### 6.2 映射合同

每个完整旧 key 生成一个不同的新 bare ID：

```text
web:<old id>    ──▶ akashic:<new id A>
```

映射必须确定、一对一且可在迁移 plan 中审阅。新 bare ID 由固定 namespace 对完整旧
Session key 做 UUIDv5 后取 32 位小写十六进制；不得按尾部 ID、正文或相似历史合并。
迁移工具是唯一 mapping owner，各服务端持久 owner 使用同一 plan，不形成长期 registry。

历史 Message 身份也随 Session 迁移：每条旧 Message 使用迁移后的 Session key 与原 `seq`
重新生成 `akashic:<new id>:<seq>`。Message 正文、role、seq、时间与 Turn 身份保持不变。

### 6.3 已证明需要处理的引用

| owner | 迁移动作 |
|---|---|
| `sessions.db` | rekey `sessions.key`、`messages.session_key/id`、`turns.session_key`、`message_attachments.message_id`、`message_embeddings.message_id` |
| Session 历史 JSON | rekey compaction 的 Session/Message/source ref；prepare 必须为空；旧 checkpoint 逻辑失效并把 cursor 归零，正文历史不减少；rekey delete/source mutation audit 引用 |
| 活动 fence | `session_admissions`、`inbound_handoffs`、`session_compaction_prepares` 在 preflight 收束，不猜测改写活动 owner |
| Scheduler/Wake/delivery | 分别 rekey target/recipient、Wake context、accepted/projection Session 与 `projection_message_id` |
| Content/Drift | rekey `selected_session_id` 与 `accepted_session_id`，保留 selection/turn/settlement 身份 |
| 配置 | 删除 `[channels.chat].channel_name`；Akashic Channel 名称不再可配置 |
| Akasha sidecar | 不逐行改写，调用现有固定输入 rebuild |

Content 与 Drift 的真实插件 SQLite 已由 schema 证明并纳入迁移；不存在另一套按名称猜测的
`proactive.db` 或 `wake_proactive.db` 迁移。

### 6.4 受保护事实

迁移改变路由用的 Session key，不改变消息正文、role、seq、Turn/Interaction、附件 artifact、
模型选择或任务内容。

当前 `messages.id` 由创建时的 `session_key + seq` 组成，因此它属于需要统一的历史身份。
迁移必须在同一 plan 中级联更新 message embedding、附件绑定、reply、compaction provenance
和可继续查询的审计引用。旧 compaction digest 属于旧 identity plan，因此 checkpoint 保留但
明确失效，Session cursor 归零并在以后按新身份自然重建；消息正文不减少。FTS 只索引正文，
不因 identity rekey 重建，只做完整性检查。

迁移备份、完整性 manifest 与 old→new plan 保留旧身份作为恢复证据；它们不是运行时可读的
第二套身份。除这些迁移证据外，已知持久 owner 不得留下可路由、可查询或可回复的旧身份。

## 7. 验收

2. 两端创建的 Session 都使用 `akashic:<id>`，并能被另一端列出、打开和继续；本地导航互不
   强制跳转。
4. Schedule/Wake/Proactive 的真实目标引用迁移后仍指向同一会话；每次逻辑投递在通知前只
   追加一条 suppress assistant Message，并推进目标 Session `seq`。
5. 迁移保持 old→new 一对一、Session/Message 与全部已知引用无旧身份、无悬空引用；正文、
   seq 与 Turn 身份不变，Akasha 固定输入 rebuild 通过。失败不留下半新半旧 workspace。

若实现需要新增 Port、共同客户端协议、Session 生命周期、通用 delivery 语义或未列出的
持久 owner，必须停止并重新批准设计。
