# 0016 · 渠道投递使用完整逻辑消息

- 状态：accepted
- 日期：2026-08-01
- 关联条款：OUT-001～OUT-003、RUN-001～RUN-003、MOB-001、MOB-005、SES-005～SES-006

## 背景


## 决定

1. Core 使用一个带类型附件的完整出站消息，并且每个 channel 只注册一个 delivery adapter。
2. adapter 返回结构化 receipt，至少区分 `success`、`partial` 和 `failed`；禁止解析人类可读字符串作为提交证据。
3. 被动回复和主动发送复用同一消息模型与 adapter。主动路径直接等待 adapter 的实际终态，不把 MessageBus 入队视作送达。
5. 只有完整成功才追加主动 SessionDB 消息并运行成功副作用。receipt 提供 canonical media，使历史保存已提交稳定副本而不是调用方的临时路径。

## 理由


## 影响

- channel 注册接口由多个 sender 收敛为一个 adapter。
- Telegram 等需要多个原生调用的渠道必须准确报告部分送达。
- 不新增附件自动清理、outbox、重试状态机或客户端迁移。

## 验收

- 一条含正文和附件的主动消息只调用一次 adapter。
- 任一必需部分失败时不追加主动历史，也不运行成功副作用。
- 历史页使用 receipt 返回的 canonical media。
