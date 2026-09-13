# Akashic 纸张品牌系统

- 状态：implemented first slice
- 日期：2026-08-26
- 决策：[0043](../decisions/0043-paper-brand-tokens-replace-material-visual-semantics.md)
- 关联条款：WEBUI-001～WEBUI-007、MOB-001

## 1. 问题和用户意图



## 2. 品牌语法

```text
┌──────────────────────── 一张连续纸面 ────────────────────────┐
│ paper      纸张层级：canvas / editing / quiet / sheet / raised │
│ ink        阅读层级：primary / secondary / muted              │
│ rule       结构层级：subtle / default / strong / focus        │
│ typography 阅读与技术内容                                    │
│ status     success / warning / error / trace / info          │
└──────────────────────────────────────────────────────────────┘
```

品牌感觉由排版、留白、墨色关系和细规则线共同形成。不得用位图噪点、`feTurbulence`、随机纹理或持续动画冒充纸张；去掉装饰后，阅读层级和交互状态仍必须完整。

## 3. Token 合同

| 轴 | 公共前缀 | 负责 | 不负责 |
|---|---|---|---|
| Paper | `--ak-paper-*` | 页面和局部纸片的表面层级 | 文本、状态色 |
| Ink | `--ak-ink-*` | 正文、次要文字、弱化文字 | 背景和边框 |
| Rule | `--ak-rule-*` | 结构线、强边界、焦点 | elevation |
| Typography | `--ak-type-*` | 字体、字号、行高与辅助信息节奏 | 组件尺寸 |
| Status | `--ak-color-status-*` / `--ak-sys-color-*` | 成功、警告、错误、轨迹和信息 | 品牌强调 |

组件只能消费语义 token。若缺少角色，先证明至少两个产品位置应该一起变化，再补角色；不借用当前颜色恰好相同的 border 或 status token，也不为设想中的组件提前发布 token。

### 3.1 本次从成熟 WebUI 沉淀的角色

| 好的既有 CSS | Brand token | 共同消费者 |
|---|---|---|
| 安静侧栏底色 | `--ak-paper-quiet` | Desktop sidebar、配对界面、Dashboard rail |



```text
共享 WebUI
├─ ChatMessageView / message-view.css ── 用户气泡、回答、Markdown、工具过程
├─ theme.css / brand tokens ──────────── 纸张、墨色、字体与规则线
   ├─ 复用上面两层
   └─ 只拥有 viewport、触摸、抽屉、Bridge、草稿和离线状态
```

- 用户消息继续使用桌面 WebUI 已有的中性气泡，Akashic 回答直接显示正文，不添加角色标题。
- 原生能力在 Browser Lab 中只记录、实现或明确拒绝，不能用 mock success 隐藏边界。

## 5. 字体

- 阅读正文使用仓库随附的 `LXGW WenKai GB Screen` v1.522；浏览器运行时由四个 `unicode-range` WOFF2 分片组合为同一字体，正文不低于 16 px，三行以上正文行高不低于 1.4。
- 代码、时间、运行身份和短技术标签使用 `JetBrains Mono`。
- 最多同时出现阅读与技术两种字体；不能为单一组件增加第三种品牌字体。

## 6. 主题与兼容边界

Theme Runtime 仍从同一 Catalog 选择浅色和深色主题。`brand-tokens.css` 是新组件入口；Material 和旧 Akashic namespace 只提供迁移兼容。迁移期间的方向是：

```text
Theme Catalog → brand tokens → product components
             └→ legacy aliases → un-migrated consumers
```

新组件不得增加 `--md-sys-*` 直接依赖。迁移完成前不删除旧 namespace，避免破坏 Dashboard、插件和第三方公开控件。


## 7. 验收

2. conversation、stream、long、reconnecting 四个 fixture 均可操作。
3. 320 px 不发生横向溢出；200% 缩放仍可到达输入、发送和恢复动作。
4. light、dark、focus、selected、error、stopped 和 reduced-motion 均保持文字或图标信号。
5. WCAG 2 A/AA 自动检查通过；字体、色值和截图只在固定 Chromium 环境内比较。

## 8. 回滚
