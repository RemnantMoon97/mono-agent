# 0043 · 纸张品牌 Token 取代 Material 视觉语义

- 状态：accepted
- 日期：2026-08-26
- supersedes：[0023](0023-akashic-tokens-own-material-3-semantics.md)
- 关联条款：WEBUI-001～WEBUI-007

## 背景


用户确认后续不再以 Material Design 作为视觉目标。新的纸张品牌必须继续保留主题唯一 owner、状态色语义和平台能力边界。

## 决定

1. `frontend/theme/src/brand-tokens.css` 是组件使用的品牌 token 入口，公开四条正交轴：`paper`、`ink`、`rule`、`typography`；success、warning、error、trace 和 info 继续使用独立状态角色。
2. token 以角色而不是组件命名。允许 `paper-canvas`、`paper-editing`、`ink-secondary`，不新增 `card-background`、`chip-radius` 或 `button-blue`。公共 token 必须已经有产品消费者；只有设想、没有像素归属的角色留在设计讨论中，不提前进入 API。
3. 页面默认是连续纸面。留白、字级、缩进和细规则线建立层级；只有输入、附件、错误、选择和需要隔离的工具详情形成局部纸片。卡片、气泡、胶囊、阴影和纹理不得成为默认容器。
5. `--md-sys-*`、`--ak-color-*` 和 `@material/web` 只作为迁移兼容接口，不能继续拥有品牌含义。尚未迁移的页面可以通过兼容别名取值，新实现只消费 `--ak-paper-*`、`--ak-ink-*`、`--ak-rule-*` 和 `--ak-type-*`。
6. 本决定只改变 WebUI 展示和 token API，不取得 SessionDB、Room、outbox、Bridge、配对、附件传输、原生生命周期或 WebUI generation 的所有权。

## 目标结构

```text
┌──────────────── Akashic Theme Catalog ────────────────┐
│ 主题色值 + status roles                               │
└────────────────────────┬──────────────────────────────┘
                         ▼
┌──────────────── Paper brand contract ────────────────┐
│ paper │ ink │ rule │ typography │ status │
└───────────────┬───────────────────────┬──────────────┘
                ▼                       ▼
                │                       │
                └──────────┬────────────┘
                           ▼
           legacy Material aliases during migration
```

## 理由


## 影响与回滚

- 0023 变为 superseded；Material namespace 在消费者迁移完成前保留，不再接受新的直接依赖。

## 验收

- 关闭阴影和背景装饰后，角色、状态和交互边界仍可辨认。
- Browser Lab 覆盖 snapshot、stream、terminal、send 和原生能力拒绝；自动可访问性检查无 A/AA 违规。
