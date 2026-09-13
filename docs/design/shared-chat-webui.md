# 共享对话 WebUI 试点设计

- 状态：implemented pilot
- 日期：2026-08-01
- 决策：[0018](../decisions/0018-chat-webui-has-one-source-and-two-adapters.md)
- 关联条款：WEBUI-001～WEBUI-007、MOB-001、TST-007～TST-008
- 视觉系统：[0043](../decisions/0043-paper-brand-tokens-replace-material-visual-semantics.md)；[纸张品牌系统](akashic-paper-brand-system.md)

## 1. 用户意图


## 2. 当前事实与边界

- **F：** 两端都使用 React、Vite、`ChatMessageView` 和 `MessageResponse`。
- **U：** 本试点不定义 iOS 容器、远程动态下发 WebUI 或线上灰度更新协议。

## 3. 目标结构

```text
┌──────────────────────── akasic-agent ─────────────────────────┐
│ frontend/theme                                               │
│ ├─ theme-catalog.json       主题色值与领域状态目录             │
│ ├─ brand-tokens.css         paper / ink / rule / type          │
│ └─ material-tokens.css      迁移期兼容与既有适配器             │
│ frontend/chat                                                │
│ ├─ theme.css                共享 WebUI token 入口             │
│ ├─ message-view.tsx          共享消息、工具、流式正文          │
│ ├─ message-view.css          共享消息、工具与引用视觉          │
│ ├─ message-actions.tsx       共享引用、复制与引用预览          │
│ ├─ conversation-navigation.* 共享功能入口、会话与底部操作      │
│ ├─ main.tsx                  桌面适配器 + QR 配对能力          │
└───────────────┬──────────────────────────────┬─────────────────┘
                │ desktop Vite build           │ clean commit build
                ▼                              ▼
       ┌─────────────────┐          ┌──────────────────────────┐
       │ HTTP + WebSocket│          │ manifest + SHA-256       │
       └─────────────────┘          └────────────┬─────────────┘
                                                │ pinned consumer
                                                ▼
                                   ┌──────────────────────────┐
                                   │ Gradle verify + unzip    │
                                   │ WebViewAssetLoader       │
                                   └──────────────────────────┘
```

## 4. 能力矩阵

|---|---|---|---|
| 主题、消息、Markdown、工具轨迹 | 拥有 | 使用 | 使用 |
| 流式正文生长 | 单消息 rAF 发布器 | WebSocket delta 提交权威目标 | native patch 提交权威目标 |
| 会话侧栏、引用、复制 | 拥有 | 使用 | 使用 |
| 知识与插件入口 | 共享导航结构 | 跳转 Dashboard 公网端口 | 打开 Native bridge 页面 |
| 新聊天 | 共享导航结构 | Web session | Native bridge session |
| 扫码配对展示 | 复用视觉组件 | 生成 QR、确认设备 | 不挂载 |
| 相机扫码 | 无 | 无 | 原生 CameraX / ZXing |
| 设置、诊断、清理同步、重新扫码 | 无 | 不挂载 | Native bridge 拥有 |
| 离线队列、重试、阅读位置 | 只展示已验证状态 | 无 | Room 与 Native bridge 拥有 |

## 5. 性能合同

1. 历史消息保持 `content-visibility: auto`，streaming 行不启用该隔离，避免正在增长的消息高度估算错误。
2. Native patch 继续按 `requestAnimationFrame` 合并；React 消息行继续 memo，未变化历史行不重渲染。
5. `MessageResponse` 只在正文或 `isAnimating` 改变时更新。流式 Markdown 由 Markstream 的 append-tail parser 接管并复用稳定顶层节点；terminal 使用同一组件完成最终解析，不再维护第二套 block 冻结和未闭合修复补丁。
6. 代码块、数学公式与 Mermaid 在 streaming 阶段保持轻量源码节点，terminal 后交回 Markstream 内建 renderer 完成富化；历史行继续按 viewport 延后富化。
7. 桌面打开会话先按 `seq` 游标读取最新尾页；读取更早页时按稳定消息 identity 恢复阅读锚点。分页只读 SessionDB，不修改、压缩或删除权威消息。

### 5.1 已验证的更新放大故障


高频局部变化经过每个 adapter 后都必须保持局部：正文 delta 不得重新查询、物化、序列化或提交未变化历史；稳定消息保持对象身份；只有 terminal、history heal 或明确的会话切换可以用权威 snapshot 校准展示。性能验收除 TPS 与总耗时外，还要记录每个 delta 触发的查询行数、bridge 字节数、React 通知次数和长任务，避免把“结果一样”误判成“成本一样”。

### 5.2 观测与归因


```text
用户发送
  │
  ├─ send.received → send.ack → reply_sent
  │
  ├─ Akasha query → Provider raw first → Core first delta
  │                                      │
  │                                      ▼
  │                         durable queued → socket sent
  │                                      │
  │                                      ▼
  │                         Room → React commit → next frame
  │
  └─ Provider done → Akasha turn commit → runtime terminal
                                           │
                                           ▼
                              durable final → composer ready
```

定位规则：`send → provider.call.start` 属于准入、上下文与 Akasha 前置段；`provider.call.start → raw.first` 才是供应商首块段；`raw.first → next frame` 属于 Core、网络、Room 与 WebView 消费段。尾部同理拆成 Provider 完成、Akasha/AfterTurn、worker durable terminal 和客户端 composer 四段。

## 6. 产物、失败和回滚

### 桌面连接与异步任务

桌面 controller 使用 Effect v3 管理 WebSocket 重连、发送等待和状态轮询。选用稳定版，
先固定资源泄漏与轮询重叠的回归，再替换手写 timer 和取消记账；不增加独立连接管理类。
TypeScript 固定为 5.9.3，沿用现有 strict、ES2022 和 bundler 配置。

```text
┌─ React controller 生命周期 ─────────────────┐
│ 连接任务 → socket / 监听器 → 释放 → 重试等待 │
│ 轮询任务 → fetch 完成 → 等待 → 下一次 fetch │
└───────────────────┬────────────────────────┘
                    └─ 卸载：中断任务并释放资源
```

自动重试保留十二次上限、指数退避、三十秒上限和随机延迟；成功连接重置计数。
主动发送可以提前结束重试等待，但不重放已经发送的消息。发送等待的成功只表示
`WebSocket.send` 完成，不代表服务端确认；超时、关闭和取消仍以错误返回。
状态轮询在上次请求结束后等待 1.2 秒，卸载时中断 fetch，旧请求不能在卸载后发布状态。

`desktop-chat-lifecycle.test.mjs` 挂载真实 controller，使用受控 socket、HTTP 完成顺序和
时钟验证重连后卸载、串行轮询、重试上限及 React StrictMode。

### 构建与恢复

- 回滚主仓库到上一个 WebUI commit并重新打包；移动仓库恢复上一个 ZIP、摘要和 source lock。两边都不需要迁移数据库或 workspace。

## 7. 视觉语法


## 8. 试点验收

- 视觉：只在生产桌面 Chat 与移动 Web 构建中核对主题 token、布局、消息和工具轨迹；不得为验收复制第二套消息 DOM、静态数据或交互状态。
- 离线与降级场景：通过生产 Chat 的静态响应、fixture transport 或现有状态测试注入输入，继续使用正式 `ChatMessageView`；不维护平行方案页或独立消息实现。
- 移动仓库：ZIP 正向校验、篡改失败测试、Gradle debug build。
- 报告两仓库 commit/tree、ZIP digest；真机 WebView、内存、掉帧和冷启动单独列为未验证或设备证据。


## 历史附件、聊天投影与发送待办（2026-09-08，已确认）

```text
┌─ Core Message / Artifact ──────────────────────────────┐
│ 原始 Message + seq；不可变 artifact_id / metadata / bytes │
└────────────┬──────────────────────────────┬─────────────┘
             │ 完整同步                      │ Message 引用授权下载
             ▼                              ▼
│ 只读聊天投影；隐藏迁移诊断     │  │ Room / outbox / 文件缓存    │
│ 思考与工具记录留在原 Message  │  │ cacheId = H(server, artifact)│
└──────────────────────────────┘  └────────────────────────────┘
```

- `attachment.download` 请求携带 `message_id`、`artifact_id` 和 `offset`。Session reader 先确认该 Message 引用了文件，再由 Core ArtifactStore 核验和读取。回复保留完整附件 metadata；下载二进制头使用 `artifact_id`，上传头继续使用 Frame ID `attachment_id`。空文件允许零字节分片并按 SHA-256 验证。
- `history.provenance`、`history.record`、`history.turn_input` 不进入普通聊天；纯归档 Message 不占布局和可见未读数量。`history.transcript` 的已知旧格式按原组顺序展示思考、说明和工具记录，不生成新消息或执行状态。原始数据和同步进度不减少。旧阅读或导航锚若指向隐藏行，定位到后续首个可见行；末尾则定位前一可见行，不能直接跳到最新消息。
- 明确拒绝删除本地 outbox、保留失败正文并释放本地上传占用。结果未知保留原命令及其附件占用；核对复用原 ID。新一次发送创建新 Message，不迁移旧视觉身份。已落地 Input 或 ACK 都是接受证据，迟到错误不得将其降级。
- 文件缓存写入失败只结束该下载并消费对应回复。Room 持久化失败停止消费和 ACK，等待用户处理存储后重连；自动重连不作为本地数据修复。


## 9. 正文接替与历史恢复（2026-09-09）

### 9.1 已确认的正文展示

同一 source 的 `continue` 正文只是执行中的当前正文；新正文接替旧正文，`complete` 结束后只展示最终正文。思考与工具合为一条过程轨迹，仍沿原 `message_id + part_index` 查看，最后正文使用最终 Message 的复制、引用和时间。`pause`、`failure`、`abandon` 按 `through_seq` 隔开前后展示过程，停止前的过程保留在末条输出上；这不改变持久 Turn 的划分。其他 source 的输出不替换本来源正文。分页和实时追加使用同一展示规则，不改写任何 Message。

```text
Input → 等待 → 思考 / 工具 + 当前正文 → 最终正文
                  └─ 原过程引用仍保留 ──────┘
```


### 9.2 当前窗口与按需历史



```text
┌───────────────┐    ┌─────────────────────┐
│ 认证与会话目录 │───▶│ 当前会话尾页，head=H │
└───────────────┘    └──────────┬──────────┘
                              ▼
                   ┌──────────────────────┐
                   │ 从 H 订阅 + 回复状态 │──▶ 可以发送
                   └──────────────────────┘
┌───────────────┐    ┌─────────────────────┐
│ 上翻 / 引用跳转│───▶│ 旧页 / 目标所在窗口 │
└───────────────┘    └─────────────────────┘
```

#### 分页与展示合同

- `history.get(direction=backward)` 无 cursor 时读尾页；`before_seq` 排他地读更早消息；`around_id` 由服务端定位并返回以目标结尾的窗口，目标不存在返回 `message_not_found`。三者都复用 `MessageReader.read_tail`，不创建消息身份或改写日志。
- 页内 `(after_seq,next_after_seq]` 是完整收到的记录/下载清单范围；`through_seq` 是快照 head。向后页的 `next_after_seq=before_seq-1`；`next_before_seq` 是首条 seq，`has_more` 指更早记录。没有更早记录时下界为 -1。`request_id` 只关联当前请求，不成为历史进度。
- 初始页数 50 是调节值，不是聊天准入条件。单帧仍有 240 KiB 预算，尾页缩小时只舍去左侧整条记录，不能丢掉最新消息。超大可见正文沿既有整条 JSON 清单和 Range 传输，不截断内容。
- Room 19→20 只新增 `message_ranges(sessionId,afterSeq,throughSeq)` 与下载记录的表示标记；不删除旧消息、附件、草稿、outbox 或配对。范围和完整记录/清单在同一事务提交，之后才 ACK。重叠范围合并不丢覆盖；没有范围证据的旧缓存不推断为完整前缀。
- Native 拥有订阅、Room、下载与唯一连续显示窗口。上翻扩展左边界；引用跳转换成目标窗口；旧窗口中的新实时消息只进入缓存，回最新时重新取尾页。窗口替换推进 WebUI projection generation，旧代际事件不能混入新窗口。
- 初次 READY 等待当前尾页清单、当前订阅确认和已知回复状态，不等待全部旧正文、附件、其他会话或通知定位。outbox 仍持久化用户输入，在 READY 后由原 owner 发送。保存的阅读锚在近期窗口外时，单独定位该锚，不枚举它之前的全部记录。
- 缓存未命中不能判定通知过期；必须取得服务端明确不存在证据。通知仍在精确 `Output(finish=complete)` 的完整正文落地后发布。分页错误不把已就绪连接降为不可聊天。

#### 所有权与持久化

Core Message 日志和附件保持 append-only，只有既有 adapter 的读协议变化；不新增 SessionDB 状态或删除权限。Native 本地范围只增加或合并，明确清理投影时与对应缓存一起减少；事件 `reset_required` 只更新事件 cursor 并重读目录/当前尾页，不清空 Message。旧缓存重放时只允许去掉上述不可见归档展示值，并逐字段核对其余消息事实；权威归档仍在服务端。旧、新清单交错时保留当前下载 owner，不改写其已确认片段。


验收覆盖尾页帧预算、旧/新表示摘要、完整数据库未变、部分窗口与实时追加、Room 接收范围/ACK 原子性、旧引用定位、阅读锚、断线和未下载正文。性能分别记录首屏 bytes、接收行数和发送时刻，不能把本地输入接受当成服务器已发送。

参考：[Matrix limited timeline 与向前补页](https://spec.matrix.org/latest/client-server-api/#syncing)、[Stream 消息 ID 分页](https://getstream.io/chat/docs/javascript/channel-pagination/)；使用本项目已有 `message_id + seq`，不引入第三方 token 模型。

### 9.3 回复过程与调用统计

每条过程轨迹只在开头挂载一次 `turn.before_reasoning`，工具调用后的消息和后续草稿不重复挂载。实时回复从等待首段起就把插槽放在同一个思考面板内；思考到达后不移动插槽，避免 Akasha 卡片卸载重查。`model.selection` 是内部选择记录，不占正文布局，未知插件内容仍明确显示不可展示。



“加载更早的消息”位于已加载历史的最上端，占有独立行并随消息滚动，不悬浮遮挡正文。插件卡片查询从实际发出时开始计算 30 秒期限；本地排队不消耗传输期限，超时仍取消所属 UI owner 并释放容量。



### 9.4 召回卡片的页面缓存


UI owner 只拥有订阅。最后一个订阅卸载时，尚未发送的读取取消；已经发送的页面缓存读取由独立 wire owner 完成，最多等待现有 30 秒期限。catalog 切换取消所有在途读取，旧结果不能污染新版本。普通非缓存查询仍随原 UI owner 撤销。

renderer 可提供 `prefetch(context)`，只执行一次轻量读取，不挂载卡片内容；Host 仅保留近视口观察节点。已提交过程在邻近可视区域预取首个思考卡片；未展开的预取不轮询，主动展开可以提升尚在网页队列中的同一读取优先级。进行中的可见卡片仍按已有间隔刷新，失败保留已显示内容并提供局部重试。页面重载会丢弃内存结果，缓存不是新的召回权威状态。
