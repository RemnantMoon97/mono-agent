# 0034 Turn 是逻辑工作单元

- 状态：accepted
- 日期：2026-08-13
- 决策者：维护者
- 关联：`CTX-003`、`SES-007`、`SES-008`、`MEM-011`、`OUT-001`、`OUT-004`、`SCH-003`

## 背景


## 决策

`Turn` 只表示用户可理解的逻辑工作单元，`Attempt` 表示 Turn 内部的一次执行。被动链路的 `U1 → interrupt → U2 → interrupt → U3 → A` 是一个 Turn、三个 Attempt；全部消息共享一个 `control_turn_id`，实时流的 `turn_id` 暂时继续标识可中断和可重放的 Attempt。

proactive、每次 `message_push`、每次 schedule fire 和每个 spawn completion assistant 分别创建独立 Turn。它们的 assistant 明确送达后立即关闭，不等待用户回复；随后用户回复创建新的被动 Turn，只能用显式引用关联来源。

```text
被动： U1 ── Attempt 1 ── interrupt ─┐
        U2 ── Attempt 2 ── interrupt ├── Turn L ── A final
        U3 ── Attempt 3 ─────────────┘

主动： source ── Turn P ── A delivered ── closed
```

## 结果

- 现有 `turns` 表仍保存 Attempt checkpoint，不迁移或删除既有记录。
- 主动来源在发送边界分配逻辑 Turn，不取得目标 session 的推理 lane。
- 旧客户端继续只读取 `turn_id`；新客户端使用 `control_turn_id` 完成 canonical 合并。
