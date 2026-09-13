# 0018 · 对话 WebUI 使用一个源码真源和两个平台适配器

- 状态：accepted
- 日期：2026-08-01
- 关联条款：WEBUI-001～WEBUI-003、MOB-001、GOV-001～GOV-005、TST-007～TST-008

## 背景


## 决定


## 理由

源码集中可以消除视觉与消息组件漂移；双入口保留平台能力差异，不需要把原生状态搬进浏览器运行时。固定产物让移动仓库在没有父仓库 checkout 时仍可离线构建，并使发布证据绑定不可变组合。

## 影响

- WebUI 改动只在 `akasic-agent` 编写和测试；baseline 变化更新移动仓库的固定 ZIP 与摘要，兼容的运行时 UI 变化也可按 0022 发布服务端 generation。

## 验收

- 主仓库能分别完成桌面与移动 WebUI 构建，移动状态测试通过。
- Web 与移动产物使用同一主题 token 和同一消息呈现组件。
- 篡改 ZIP 或摘要时构建 fail-loud。
- 两仓库报告包含各自 commit/tree、WebUI source commit/tree 和 ZIP SHA-256。
