# Action Fusion 适配评估与实施计划

- 状态：**proposed；仅完成设计，尚未实现或启用**。
- 日期：2026-09-13。
- 核对基线：`ae775e67d5ef6548628d85a162ef20f50d034f61`，本地 `main`。
- 输入：用户提供的 SoL-Pi Action Fusion 设计说明；本文未把其中的源码行号当作已独立核验的证据。
- 本次交付：本实施文档与阅读索引。不修改生产代码、配置、数据库或现有长期需求。
- 关联合同：[FS-001、FS-002、SH-001～SH-003、CTX-001、SEC-001、SEC-010](../projectneed.md)、[工具 provider view](../decisions/0062-tools-flow-through-provider-views.md)、[执行失败与恢复](../decisions/0063-execution-failures-have-terminal-results.md)、[持久化状态地图](persistence-state-map.md)。

## 1. 结论与范围

**可以加入 mono-agent，建议保留“原文件工具追加可选 `then_run`”的交互方式，由 `standard_tools` 拥有实现。不能直接复制 Pi 的同名覆盖、异常拼接和三状态协议。**

收益来自提前声明一个已经确定的后续命令，减少一次模型周转。文件写入和进程执行仍是两个实际副作用，仍需分别授权、记录和恢复。对模型是一条融合调用，不等于内部只能有一份回执。

首期支持本地 `edit_file` / `write_file` 后运行一条非交互验证命令；复用 Shell 的短等待、硬超时和 `execution_id` 续接。默认关闭，由试点来源显式开启。没有 `then_run` 的请求直接进入原文件工具路径。

首期不覆盖 Host Bridge 文件操作、多文件事务、MCP 文件工具、交互式 PTY、命令数组、条件分支或自动重试。Bridge 环境收到融合参数时在编辑前明确拒绝；不把远端文件改完后再用本机哈希检查，也不静默忽略命令。

这不是“三个文件的小扩展”：文件包装本身较小，但当前持久执行、工具授权和 Shell cleanup 都必须接上。建议按第 10 节分阶段交付，完整上线以授权、恢复、清理和压缩验收全部通过为准。

## 2. 已确认的当前实现

以下事实来自本次读取的代码，符号比行号更适合作为后续定位入口。

| 领域 | 代码证据 | 对迁移的影响 |
|---|---|---|
| 工具注册 owner | [standard_tools/plugin.py](../../plugins/standard_tools/plugin.py) 的 `apply`、[files.py](../../plugins/standard_tools/files.py) 的 `register_file` | 文件、Shell 在同一普通插件内注册，优先在该 owner 增强 |
| 重名行为 | [tools/plugin.py](../../plugins/tools/plugin.py) 的 `ToolCatalog.register` | 重名会抛错，不支持 Pi 式同名覆盖；每个 Root 只能注册一次对应名称 |
| 当前模型 schema | [standard_tools/filesystem.py](../../plugins/standard_tools/filesystem.py) | 实际名称为 `edit_file`、`write_file`，不是 `edit`、`write` |
| 文件结果 | [standard_tools/files.py](../../plugins/standard_tools/files.py) 的 `FileTool.invoke` | 返回字符串是成功；`ToolResult.is_error` 表示失败。必须判断类型状态，不能依靠是否抛异常或文本措辞 |
| 文件底层 | [agent/tools/filesystem.py](../../agent/tools/filesystem.py) 的 `EditFileOperation`、`WriteFileOperation` | 已有 canonical path 锁、锁内重读、原子写入和 BOM/换行处理，应复用 |
| 路径语义 | 同文件的 `_resolve_path`、`_get_file_mutation_key` | 支持 `~`、相对路径和 `resolve`；相对路径以 `allowed_dir` 或进程 cwd 为基准，不具有 Pi 的 `@`、`file://` 引用语义 |
| 文件限制 | `standard_tools.apply`、`FileSettings` | 默认写工具的 `allowed_dir=None`；配置能指定限制。不能声称当前所有写操作都默认限制在 repository 内 |
| Shell | [standard_tools/shell.py](../../plugins/standard_tools/shell.py) 的 `ShellTool.prepare/invoke` | prepare 固定目录、shell、参数和 owner，并调用 `validate_command`；invoke 通过 `PROCESSES` 执行 |
| 长任务 | [unified_exec.py](../../agent/tools/unified_exec.py)、`ShellTool.invoke` | `Result.outcome=success` 可能只表示进程已接纳并返回句柄，不能据此判定测试通过 |
| 授权 | [tools/execution.py](../../plugins/tools/execution.py) 的 `_run`、`ToolCatalog.execution` | 先准备并保存最终参数，再走 provider 与来源授权，落盘 started 后执行；直接调用 `ShellTool.invoke` 会绕过这条授权链 |
| 来源范围 | [tools/menu.py](../../plugins/tools/menu.py) 的 `ToolMenu`、[subagent/plugin.py](../../plugins/subagent/plugin.py) 的 `program` | 来源取得具体 binding；子任务授权检查原 profile 的 binding 集合。持有文件工具不等于持有 Shell |
| 恢复 | `ToolExecution._run`、`FileTool.query`、`ShellTool.query` | 两类工具都非幂等，query 当前返回 None；started 后无结果不会盲目重发 |
| 进程清理 | `shell.py` 的 `shell_cleanup`、[conversation/program.py](../../plugins/conversation/program.py) | cleanup 从持久 ToolCall 找 binding，当前按 `shell/write_stdin/task_stop` 名称识别。融合后的文件 ToolCall 会被漏掉 |
| 上下文压缩 | [compaction/akashic.plugin.toml](../../plugins/compaction/akashic.plugin.toml)、[message_summary.py](../../plugins/compaction/message_summary.py) | 当前入口为 `message_plugin.py`，按完整消息组摘要；已有保留验证事实与 execution_id 的提示，没有 Action Fusion 标记解析器 |

当前调用关系：

```text
┌─────────────────────────────────────────────────────────┐
│ 来源程序 → ToolMenu（获授 ToolView / 固定 bindings）      │
└──────────────────────────┬──────────────────────────────┘
                           ▼
┌─────────────────────────────────────────────────────────┐
│ 持久 ToolCall → ToolExecution                            │
│ prepare → 最终参数落盘 → authorize → started → invoke    │
└──────────────┬─────────────────────────┬────────────────┘
               ▼                         ▼
┌─────────────────────────┐  ┌────────────────────────────┐
│ FileTool                │  │ ShellTool                  │
│ canonical 锁 / 原子写入 │  │ PROCESSES / execution_id   │
└──────────────┬──────────┘  └─────────────┬──────────────┘
               └────────────┬────────────┘
                            ▼
              ┌───────────────────────────┐
              │ ToolResult / 恢复回执     │
              │ Context 投影 / Compaction │
              └───────────────────────────┘
```

**已发现的文档差异**：SH-001 的清理描述仍提及 query 结束，当前 `shell_cleanup` 使用来源范围、abandon 分区和独立清理任务。本文以真实 owner/abandon 路径制定兼容验收，不另定义“融合结束即杀进程”。FS-001 的“失败保留旧文件”针对 mutation 失败；融合命令失败时 mutation 已提交，必须在实施决策中明确两者的区别。

**尚未验证的边界**：实际部署的插件配置与授权插件、不同模型的融合采用率、Bridge 的一致性与恢复能力。本次没有连接正式运行实例，也没有性能实测。本文所列新增接口、字段和文件均为建议，不是现有 API。

## 3. 原方案的保留与修正

| 原设计 | 适配决定 | 理由 |
|---|---|---|
| 可选 `then_run` | 保留 | 模型提前声明验证意图，不需要新增一个常驻复合工具名 |
| 包装而不重写 | 保留 | 复用文件操作和 Shell prepare/execute；不复制替换算法或自行 spawn |
| 同名覆盖注册 | 改为 owner 内单次注册 | ToolCatalog 拒绝重复名称，覆盖会破坏 provider 引用和归档身份 |
| 仅按 cwd 缓存工具 | 不引入全局缓存 | 工具还绑定 Root、generation、capture 配置、来源和执行域；现有 open/capture 管生命周期 |
| 命令失败抛拼接异常 | 改为预期失败返回 `Result(error, …)` | 当前 executor 对非预期异常保存通用失败并传播，原拼接内容不能保证进入模型观察 |
| 三个文本标记 | 扩展为版本化阶段结果 | Shell 有 running、取消和无法恢复的情况；文本标记不足以表达实际效果 |
| 双哈希等于防并发篡改 | 改为有限的干扰检测 | 不能消除 TOCTOU，也不能保证命令读取同一文件版本 |
| 原生 mutation 锁之外再排队 | 有条件保留短生命周期融合队列 | 不重复获取同一把非重入锁；不让文件锁跨越整个测试进程生命周期 |

“省一次模型调用”只对下一步命令已确定且原本单独推理的路径成立。多文件任务应先完成其他编辑，最后一次必要编辑才附带整体验证；把测试挂到每次编辑上可能增加成本。工具输出较长、命令运行很久或需要中途决策时，收益会下降。

## 4. 目标架构与权限合同

### 4.1 Owner 划分

| 合同项 | 建议 |
|---|---|
| `capability_owner` | `plugins/standard_tools` 拥有融合语义、文件包装和 Shell 适配 |
| `consumer_scope` | 仅当前获授文件与 Shell 能力、且显式开启的来源；主对话与子任务分别检查 |
| `authoritative_state_owner` | 文件操作 owner 管文件；Tools 管调用与阶段回执；PROCESSES 管活进程；MessageLog 管最终观察；Compaction 管派生摘要 |
| `runtime_patch` | 预计需要少量共享文件操作接口调整；不修改 ReAct 决策循环、模型 provider 或全局注册覆盖规则 |
| `runtime_patch_reason` | mutation 提供可信的写后指纹；通用 Tools 插件提供父调用限定的依赖准备/授权与阶段诊断，而不是旁路 Shell |
| `client_only_alternative` | Prompt 提醒或新复合工具名无法同时解决依赖授权、恢复与清理；仅 prompt 也不能保证省去中间推理 |

预计新增 `plugins/standard_tools/action_fusion.py`（编排）、`fusion_result.py`（阶段结果与渲染）、`fusion_queue.py`（队列）。仅在并发机制确有独立复杂度时拆出第三个文件，不以复刻原目录结构为目标。

### 4.2 依赖调用必须经过相同授权边界

首期最重要的接口工作是：让已授权的父工具调用准备并执行一个确定的 Shell 依赖，同时保留来源边界。当前 `BoundTool` 没有这个接口，不能把“内部复用 Shell”描述成已经安全可用。

建议在 `plugins/tools` 增加一个受限的依赖执行端口；具体符号在 P0 固定，必须满足以下合同：

1. 来源菜单在可信代码中把实际获授的文件、Shell 和续接 bindings 组成融合能力；模型不能传 `binding_id`、owner、generation 或 permission。
2. 依赖只能引用该来源原获授 view / fixed bindings 中的具体引用，不能查全局工具目录。来源仅获授文件工具时，手工构造 `then_run` 也必须在 mutation 前拒绝。
3. 从原 Shell binding 读取 capture 配置并执行原 prepare，不能重新实例化默认 `ShellSettings`。继承 `restricted_dir`、`allow_network`、working directory、owner 和 provider 授权 hooks。
4. 文件参数与命令最终参数在编辑前分别准备、冻结和授权。依赖预检必须没有文件、进程、网络副作用；准备失败或初次拒绝时整个融合调用不编辑。
5. 运行命令前再次检查尚可启动、来源未 abandon，以及必要的即时授权。排队期间发生撤权时保留已完成编辑并跳过命令。
6. 内部 Shell 阶段使用由父调用稳定 key 派生的独立 key；重放使用原 prepared 参数，不能第二次 prepare 后悄悄改变命令、cwd 或 owner。
7. 只支持单层、单个 Shell 后续阶段，拒绝循环依赖和递归融合。端口不发展成任意工作流引擎。

授权 UI 若存在，应展示文件变更与完整命令；现有授权 hook 若自动允许，也沿用原策略，不凭空新增人工审批。增强版工具的风险声明须覆盖 `external-side-effect`；试点开启导致风险元数据变化应在评审中明确，不能称整个 descriptor 严格不变。

如果这些边界无法通过现有 provider/view 结构表达，P0 必须先交付接口决策，不得降级为直接调用 `ShellTool.invoke` 或 `PROCESSES.exec_command` 后宣称授权兼容。

### 4.3 执行顺序

```text
┌─────────────────────────────────────────────┐
│ 未传 then_run → 原 FileTool 路径             │
└─────────────────────────────────────────────┘

┌─────────────────────────────────────────────┐
│ 已传 then_run：准备、冻结、两项授权、域检查 │
└──────────────────────┬──────────────────────┘
                       ▼
┌─────────────────────────────────────────────┐
│ 融合队列 → mutation（复用原文件短锁）       │
│ 保存编辑回执及预期内容 hash                │
└───────────┬──────────────────┬──────────────┘
            │失败              │成功
            ▼                  ▼
        命令 skipped     干扰检测 / 启动资格检查
                               │通过
                               ▼
                 Shell 原执行路径 / 阶段回执
                               ▼
┌─────────────────────────────────────────────┐
│ 一条模型观察：编辑结果 + 阶段状态 + 输出   │
│ 终态，或 running + execution_id            │
└─────────────────────────────────────────────┘
```

## 5. 工具参数与结果协议

### 5.1 模型可见参数

在原 schema 副本上追加可选 `then_run`，保留原必填字段、类型、描述和参数校验。解析后从传入 mutation 的参数中剥离 `then_run`，避免被 `**kwargs` 静默吞掉或透传至 Bridge。

| 字段 | 首期合同 |
|---|---|
| `command` | 必填、去空白后非空；由 Shell 原校验处理 |
| `timeout` | 可选，单位为秒；默认值与上下限引用 Shell 常量，表示进程硬超时，不是初次等待 |
| `cwd` | 可选；与原 Shell cwd 规则一致，不自动改成被编辑文件的父目录 |
| `description` | 可选；省略时由包装层生成“验证本次文件变更”，供原 Shell 必填描述使用 |

首期 `tty=false`，shell/login、初次等待和输出预算复用当前 Shell 默认值；不接受任意额外字段。完整 `then_run=null` 视为参数错误，不把它解释为省略。参数校验与 shell/路径/授权检查都在编辑前完成。

示例仅为拟议的工具请求，不是仓库改动：

```json
{
  "path": "/work/repo/src/client.py",
  "old_text": "timeout = 5",
  "new_text": "timeout = 30",
  "then_run": {
    "command": "python -m pytest tests/test_client.py -q",
    "cwd": "/work/repo",
    "timeout": 120
  }
}
```

建议 schema description 明说：“仅当下一条验证命令已确定时填写；编辑失败不执行命令；命令失败保留已提交编辑；返回 execution_id 表示仍在运行，需续接后确认验证结果。需要先观察编辑结果再决定下一步时省略。”

兼容边界分两层：开关关闭时保留原 schema 与行为；开关开启但未传 `then_run` 时保持原文件执行、错误和输出，不增加哈希、Shell 资源或融合标记。增强 schema 与风险声明本身是显式变化，不承诺字节级相同。

### 5.2 分阶段结果

沿用外层 `Result.outcome` 的 `success / denied / error / interrupted`，不新增 `unknown` 终态。阶段事实与外层终态分别表达。

建议结果载荷版本为 `action_fusion.v1`，至少记录：原调用引用、mutation 状态及目标路径、mutation 原输出引用、命令及 cwd、command 状态、skip/failure reason、exit code（已知时）、execution_id（已接纳时）、执行域与 owner 的内部引用。缺失证据不能填成成功；例如进程状态无法找回时外层为 error，载荷注明最后确认阶段。

| 场景 | 外层 outcome | 命令阶段 | 文件与观察 |
|---|---|---|---|
| 无效参数、unsupported backend | error | skipped | 未编辑，包含具体拒绝原因 |
| 编辑前授权拒绝 | denied | skipped | 未编辑，Shell 不启动 |
| 编辑失败 | error | skipped | 保留原编辑错误；按原 mutation 合同处理文件 |
| 编辑成功、hash 不同或无法读取 | error | skipped | 编辑已提交；说明命令未执行，不能写“回滚成功” |
| 编辑后撤权/abandon，尚未启动命令 | interrupted | skipped | 保留编辑，注明未启动及原因 |
| 命令退出 0 | success | succeeded | 合并编辑结果、退出码及有界输出 |
| 命令非零退出或硬超时 | error | failed | 编辑保留，保留真实退出/超时证据 |
| 初次等待结束但进程仍运行 | success | running | 返回 execution_id；此处 success 只表示调用接纳成功 |
| 等待取消，进程已注册 | interrupted | 按已知事实记录 running 或已知终态 | 不隐式杀进程；保留句柄/阶段回执供续接与 cleanup |
| 内部异常或重启后效果无法查回 | error / interrupted | 记录最后确认阶段和诊断 | 不推断效果为零，不自动重发 |

保留 `[then_run:succeeded]`、`[then_run:failed]`、`[then_run:skipped]`，增加 `[then_run:running]` 和 `[then_run:interrupted]` 作为可读摘要。标记不是授权凭据或权威状态；命令 stdout 也可能包含同名字符串。解析器只读取包装层结构，不扫描 stdout 决定状态。

当前 `plugins.tools.api.Result` 没有 metadata 字段，而 `ContentPart` 可携带版本化内容。P0 应选择并固定一种窄结构化 part，并为它注册内容校验、引用检查、模型渲染及摘要读取；单纯追加未注册 kind 不能视为链路完成。模型文本渲染优先保留编辑已提交、命令状态、退出码、句柄和失败摘要，Shell 长输出按现有预算处理，不把融合整体截断而丢掉前缀事实。

Shell 结果应在现有 `ExecutionResult` 仍为结构化对象时提取，通过共用执行 helper 提供给普通 Shell 与融合路径。不要从 `format_execution_result` 的展示文本反解析 exit code 或 execution_id。

## 6. 并发、路径与文件一致性

### 6.1 队列与现有锁

首期使用独立的融合队列，key 至少包含执行域与 canonical target。同一路径的融合调用从 mutation 开始串行到 Shell 首次返回；普通文件调用仍使用既有 mutation 短锁。融合队列不能每次 `open_tool` 都新建，否则不同请求之间不会互斥。

队列资源由本运行时的 `standard_tools` owner 管理，不按 cwd 建一个跨运行时全局缓存。Root/generation 切换时固定当前调用的资源；若新旧 generation 共存而未共享队列，明确只提供代内融合串行，由既有文件锁保证 mutation 串行、hash 检测额外干扰。不得声称跨进程或跨 generation 的整段互斥已经实现。

锁顺序固定为“融合队列 → mutation 原锁”，底层 mutation 结束即释放原锁；禁止在持有原锁时再调用会获取该锁的原 execute。Shell 返回 running 后释放融合队列，后续编辑可能影响运行中的测试。需要固定完整验证输入时，应使用隔离 worktree/快照验证，属于后续独立能力。

排队取消、执行异常和最后一个等待者退出都必须清理队列条目；不同文件不串行。多文件测试共享构建目录的竞争不由 per-file 队列解决。

### 6.2 指纹必须来自实际提交字节

直接在 mutate 返回后读取两次 hash 存在第一个读之前被外部覆盖的窗口，两次读到同一个外部版本也会放行。改进建议是由文件 mutation 在锁内、根据实际写入字节计算预期 hash，连同 canonical target 形成内部提交证据；不能在包装层重算编辑算法猜测写入结果。

对本地成功 mutation，以提交 hash 为基准，命令启动前再读实际内容核对。若保留“双读 + 让出事件循环”，两次实际读都与提交 hash 比较。`asyncio.sleep(0)` 只给其他协程调度机会，不等价于让所有外部进程完成写入，也不是正确性保证。

这些检查仍无法发现 A→B→A、第二次检查后的写入、命令运行期间的写入，或其他依赖文件的修改。SHA-256 检查回答的是“采样内容是否相同”，不是文件事务隔离或恶意写入防护。长时间间隔从来不是并发安全的天然保证，缩短间隔通常减少竞争暴露面，不需要为了“补偿缓冲”而刻意增加固定等待。

hash 失败、目标消失或解析发生变化时保留已提交编辑并跳过命令。hash 的计算与读取计入耗时和资源测量，大文件场景不得静默绕过守卫。

### 6.3 路径与执行域

路径解析复用原 `_resolve_path` 语义，不能只在融合队列里支持 `@` 或 `file://`。普通文件名中的 `@` 继续按现有路径规则解释；URL 支持若要新增，应作为单独文件工具需求。

覆盖测试包括相对路径、`~`、符号链接、缺失父目录、路径越界及 symlink 重新指向。canonical path 不能解决所有 hard-link 或外部重命名竞态，首期不作此承诺。进程 cwd、文件目标、hash 读取必须位于同一受支持执行域；不要把 Akashic 运行数据 workspace 当成代码 repository cwd。

## 7. 回执、取消、恢复与清理

### 7.1 持久阶段

融合保持 `idempotent=False`。父调用继续使用 `durable_call_key`；内部 mutation 与 Shell 阶段使用可确定派生、不会与其他调用冲突的 key，不使用每次重试随机生成的身份。一个模型 ToolCall 对应一个最终模型 ToolResult；内部阶段记录不伪造成模型发起的第二个 ToolCall。

建议阶段为：`prepared → mutation_started → mutation_committed → command_started → command_observed → done`。记录阶段只表示确认了哪些事实，不声称文件系统、SQLite 与进程启动可以原子提交。

| 恢复点 | 恢复行为 |
|---|---|
| 父调用尚未 started | 原 owner 依照当前授权首次执行 |
| mutation_started，但没有提交证据 | 文件可能已改变；返回 error 并要求检查现场，不重新编辑或执行命令 |
| 已保存 mutation_committed，命令尚未启动 | 首期保守终结：返回编辑已提交、命令 skipped；不在重启后延迟补跑 |
| command_started，无执行句柄/终态回执 | 查询同一阶段；查不回则返回 error，不重新 spawn |
| 已持久保存 command_observed | 返回原阶段结果；若曾为 running，仅在原执行域和进程 owner 可用时续接，不能重启后猜同一整数仍是原进程 |
| 父 ToolResult 已提交 | 读取原结果，不重跑文件、命令或模型步骤 |

当前非预期异常和取消可能由 `ToolExecution` 封口成通用失败。因此 P2 必须提供一个**只读阶段诊断出口**，由 executor 在失败封口时携带已提交编辑和已接纳进程事实，同时保留并传播原内部异常。不能用宽泛 `except Exception` 把数据库损坏、schema 错误或不变量违反伪装成可继续的测试失败。

stage store 写入失败时停止启动下一阶段；即使外部写入已经成功而回执落盘失败，也不能回滚用户文件或假设“未编辑”。该不确定窗口以 error + 最后确认事实结算，与 0063 一致。

### 7.2 进程生命周期

`shell_cleanup` 只看 Shell 工具名是本方案的确定缺口。必须让它从融合父 binding 的受信依赖描述发现原 Shell binding，并使用原 `SHELL_OWNERS`、capture state 和 abandon 分区完成清理，不能用文件工具的配置猜 Shell owner。

依赖清理关系要在启动进程前已持久可查；不能等 execution_id 返回后才建立，否则“spawn 成功、回执未保存”会漏清理。普通 Shell 与融合 Shell 共用实际 owner；cleanup 对同 owner 去重，并保留失败隔离与诊断行为。

取消等待不等于取消进程。排队前取消不产生副作用；mutation 后、spawn 前取消保留编辑并跳过命令；spawn 后取消按统一进程协议保留或清理真实执行。显式 abandon 的旧调用不得阻塞新来源分区，迟到结果仍按既有 abandon 合同保存与投影。

### 7.3 状态增改减合同

| 对象与 owner | 正常增加 / 更新 | 逻辑失效与物理删除 | 恢复证据 |
|---|---|---|---|
| 用户文件，文件 operation | 仅已授权 mutation 创建/原子替换目标；命令还可能产生其已授权副作用 | 无融合自动回滚或自动删除；后续修正需新的工具调用 | mutation 提交指纹、结果、现场文件；hash 不是完整文件备份 |
| MessageLog 的 ToolCall/ToolResult | 正常只追加，由 Tools/Message writer 提交唯一最终结果 | 不因融合失败、关闭开关或摘要而改写/删除旧消息 | 原 CallRef、binding、既有 SQLite 备份 |
| Tools 阶段记录，既有 OWNER_STATE | 为新父调用及阶段增加版本化记录；仅同 key 的阶段单调推进，终态不可反向更新 | done 是逻辑终结；当前不得自动减少，不设 TTL | 与 owner_records 同库的事务、父子 key 与结果引用 |
| 活进程，PROCESSES | 实际 spawn 后注册 owner/句柄，更新运行状态与增量读位置 | 只沿既有 stop、超时、清理和 shutdown 路径释放 | 原执行域、原 owner、执行句柄及清理回执；不承诺跨 runtime 续接 |
| 融合队列，standard_tools runtime | 有等待者时增加，进入/退出调整引用数 | 最后等待者退出可释放内存；无业务恢复事实 | 不持久化，重启从回执判断，不能从队列消失推断未执行 |
| 摘要，Compaction | 按既有协议追加派生摘要及原始消息引用 | 新摘要 supersede 旧投影；不删除源消息 | 既有摘要记录与原文读取接口 |

首选复用 owner_records，不新增数据库表或消息终态枚举。若实施发现必须修改存储 schema，应按项目 Yoyo 合同独立设计迁移，不能在插件 apply 中建表、扫描历史或补跑旧请求。

## 8. 上下文、展示与成本观测

首期要同时完成普通模型投影和 compaction 适配，不能只加入文本标记就认为 Evidence-Preserving Reducer 已存在。

摘要至少保留：修改目标、编辑是否提交、命令与 cwd、终态或 running、非零退出/超时的主要错误、可续接句柄及来源引用。对于未结束进程，摘要里的 running 仅代表该观察时刻的状态，后续必须查询。原 ToolResult 保持不可变。

建议结构化证据通过插件内容类型提供给摘要 owner，并用确定性校验防止“编辑保留但测试失败”被摘要成“修改失败”或“测试通过”。不能把 stdout 里伪造的标记当作受信证据。长输出的裁剪沿用 Shell 的 head/tail 与日志机制，结果头部的阶段事实须单独有界保留。

每次试验记录父调用 key、模型调用数、真实 mutation 次数、spawn 次数、LLM 实测时长/usage、排队与 hash 时长、命令时长、running/失败/跳过原因、最终任务是否通过。内部两个阶段不能只统计为“一次实际工具执行”。

收益估算为：省去的模型周转时间，减去新增准备、授权、hash 和回执开销。模型 token 成本还受 schema 增长、提示缓存和错误恢复影响；不能用“3 次变 2 次”推导普遍节省 33% 总成本。

## 9. 具体改动地图

所有路径为计划中的改动边界，本次没有修改这些实现文件。

| 位置 | 计划工作 | 必须复用 / 保护 |
|---|---|---|
| `plugins/standard_tools/plugin.py` | typed 开关，owner 内单次注册，运行时融合队列生命周期 | 原 `STANDARD_TOOLS` provider/view 与 Root 归档 |
| `plugins/standard_tools/files.py`、`filesystem.py` | schema 追加、参数拆分、普通路径与融合路径分派 | 原 prepare 校验、文件错误、read/list 行为 |
| 新增 `action_fusion.py`、`fusion_result.py`、可选 `fusion_queue.py` | 两阶段编排、状态载荷、渲染、队列 | 不实现新 shell，不复制 edit 算法 |
| `agent/tools/filesystem.py` | 窄内部 mutation 证据出口，暴露实际提交字节的 hash/目标 | 原公共结果、canonical 锁、原子写入、BOM/换行/mode |
| `plugins/tools/api.py`、`execution.py`、`plugin.py`、必要时 `menu.py` | 来源限定的依赖准备/授权端口、稳定阶段 key、失败诊断合并 | 原最终参数授权、非幂等恢复、唯一 ToolResult |
| `plugins/standard_tools/shell.py` | 共用结构化执行结果、受信依赖 cleanup 发现 | validate_command、原 owner、timeout、PROCESSES、续接工具 |
| `plugins/content/`、`plugins/context/`、`plugins/compaction/` 的实际扩展点 | 新内容类型校验/投影与证据保留 | 原文只追加，摘要只做派生投影 |
| `plugins/subagent/` 及来源菜单装配点 | 试点来源配置与组合能力校验 | research 不增权，scripting 原禁网/目录限制，旧 fixed bindings |
| `tests/`、`tests_scenarios/` 的相关合同 | 第 11 节的可观察回归与故障注入 | 既有受保护 oracle 不随实现降低预期 |

## 10. 分阶段实施计划

建议顺序执行。下列工作量为单人粗估，不是交付承诺；P0 完成后根据依赖端口与内容扩展的实际复杂度重新估算，整体约 7～12 个工程日。

| 阶段 | 工作与产物 | 退出条件 | 粗估 |
|---|---|---|---|
| P0：合同与接口定稿 | 将本 proposal 转为独立决策；固定依赖授权端口、结构化 part、cleanup 关系、Bridge fail-closed 和兼容策略；建立现有 baseline | 通过 file-only、受限 Shell、旧 binding 的接口走查；明确新字段均非模型可伪造的运行时身份 | 1～2 日 |
| P1：最小本地执行 | owner 单次增强注册、默认关闭；typed 参数；短命令成功/失败/skip；提交 hash 与融合队列；预检授权随功能一同接入 | 单文件路径端到端完成，未传 then_run 差分相同，授权失败零文件/进程副作用 | 2～3 日 |
| P2：执行生命周期 | 内部阶段回执、query、失败诊断；running 句柄、续接、abandon、cleanup、重启与归档恢复 | 第 11 节的取消/崩溃矩阵无盲重放和漏清理，命令失败保留编辑证据 | 2～3 日 |
| P3：投影与语义 Gate | 内容校验、模型渲染、compaction 保留、长输出；补充影响目录与契约场景 | 摘要与原文可核对，伪造标记无效，相关静态检查及 change-impact Gate 通过 | 1～2 日 |
| P4：隔离 A/B 与试点 | 固定模型与任务集，基线/候选独立 worktree；记录采用率、耗时、usage 与验证成功率 | 达到下述验收目标后才向明确选定来源开启；其他来源保持关闭 | 1～2 日 |

P1/P2 的中间提交可以用于隔离开发，不能因 happy path 已跑通就提前开启正式来源。开关、授权和生命周期缺口不作为“后续优化”。本次设计未成为已接受实施任务，因此不把这些阶段写入 NOW。

建议分成可独立评审的合同、工具执行、恢复清理、投影验收四组改动；性能报告与是否默认开启单独决定，不把失败试验的最佳结果拼成一份报告。

## 11. 验收矩阵与执行方法

| ID | 场景 | 独立观察标准 |
|---|---|---|
| AF-01 | 开关关闭；开启但未传 then_run | 原 edit/write 的成功、失败、diff 与文件完整字节一致；spawn/hash/阶段记录均无新增 |
| AF-02 | old_text 缺失、多匹配、目录目标、越界、写入失败 | 命令计数为零；原错误与旧文件保护不退化 |
| AF-03 | 修改成功 + exit 0 / 非零 / 硬超时 | 文件确实已改；独立 sentinel 确认恰好一次 spawn；返回真实状态与退出/超时证据 |
| AF-04 | 非法 then_run、空命令、未知字段、Bridge 请求 | mutation 计数为零；不能静默吞参数 |
| AF-05 | 只授文件、不授 Shell；restricted_dir/禁网；授权 hook 拒绝 | 未越过原权限，首次拒绝零副作用；排队后撤权跳过命令并如实保留编辑 |
| AF-06 | 同文件融合竞争、不同文件并行、普通 edit 与融合交错 | 同文件融合初次观察前不交错，不同文件可推进；无重复锁死锁；普通 mutation 遵守既有短锁 |
| AF-07 | 提交后插入同长度写入、伪造 mtime、目标消失、symlink 变化 | 以 barrier/事件精确注入，不依赖 sleep 运气；检测到差异则不启动命令 |
| AF-08 | BOM、CRLF、mode、缺失父目录、相对路径与 `~` | 最终完整字节/权限及目标与原工具一致；队列 key 与实际目标一致 |
| AF-09 | 长命令首读返回 running，再 write_stdin/task_stop | 句柄属于原 owner；新输出增量读取；首读不能写 succeeded；其他来源不能续接 |
| AF-10 | 队列等待、写入后、spawn 前、spawn 后、初次等待时取消 | 对应阶段的副作用计数正确；锁释放；进程不因等待读取消而丢 owner |
| AF-11 | 在第 7 节每个持久边界 kill/restart | 在第二个 runtime 上恢复；不重复 mutation/spawn；无证据返回 error，已确认编辑和句柄可追溯 |
| AF-12 | 回执事务失败、内部异常、cleanup 权限失败 | 不伪装成业务成功；保留原异常与阶段事实；cleanup 失败沿原 owner 隔离，不能推翻已提交回复 |
| AF-13 | abandon、插件换代、开关回退、旧 binding 重放 | 原归档可读，新权限不能重解释旧效果；融合进程被原 owner 发现并结算 |
| AF-14 | 输出超限、stdout 伪造标记、强制 compaction | 编辑已提交/验证失败/执行句柄不丢失；伪造文本不能改阶段；messages 无 UPDATE/DELETE |
| AF-15 | 固定验证命令的端到端任务 | 对照路径比融合路径恰好多一次模型周转；文件内容、命令及最终验证结果一致 |
| AF-16 | 多文件修改任务与需观察再决定的任务 | 模型能省略 then_run，最后一个编辑再验证；不增加无意义测试次数 |

现有回归入口优先使用 `tests/test_standard_tools.py`、`tests/test_tool_execution_receipts.py`、`tests/test_tool_abandon.py`、`tests/test_tool_views.py`、`tests/test_shell_tool.py`、`tests/test_unified_exec.py` 和上下文相关合同。新增 `tests/test_action_fusion.py` 与恢复/投影测试按实际规模拆分。

实施时的验证顺序：

1. 在一次性 repository、Akashic workspace、plugin home 和测试 HOME 中运行新增 targeted tests，以及上列直接受影响回归。
2. 按实际改动运行 Python 静态检查；若修改前端渲染再运行对应前端检查，不为纯后端改动笼统重建整个前端。
3. 用 `python docker/debug/gate.py plan --base <已核对基线>` 检查影响映射，再运行 `python docker/debug/gate.py run --base <已核对基线>`。未映射的可执行改动必须补齐目录，不能临时缩小场景。
4. 对关键 oracle 注入至少“编辑失败仍 spawn”“file-only 可执行 shell”“running 被当成功”“started 重放”的错误版本，确认测试能失败；不能以 import error 或零测试当作拦截成功。
5. 封存基线/候选 commit、环境、模型及 prompt 配置、原始测量。使用相同 synthetic/现实任务配对比较；昂贵完整 benchmark 另按项目授权流程执行。

建议的试点门槛：所有语义与安全回归通过；确定性样例稳定减少一次模型周转；在预先固定的短验证任务集（建议至少 20 个、每个重复 3 次）中任务结果无退化，端到端耗时中位数有明确改善且 p95 不出现无法解释的恶化。报告 schema 成本与采用率，样本不足不声称普遍收益。不以某个节省百分比强行打开功能。

## 12. 开关、迁移与回退

建议增加 `standard_tools` 插件配置 `action_fusion.enabled=false`，通过项目既有插件配置装配传入，不在 Core 顶层增加 Pi 风格 `config.actionFusion`。来源的融合能力还必须由自身 view/binding 授权导出；总开关不是来源授权的替代品。

旧 binding 没有融合 capture 字段时按关闭解释，保持原 schema/结果与来源配置。新 generation 冻结自身模式、Shell 依赖和执行域，不在调用中读取可变全局开关。关闭开关只影响新绑定，不撤销旧回执或销毁仍需清理的归档执行环境。

本设计预计不需要迁移历史消息，不回填 `[then_run:*]`，不重新解释旧工具错误，不扫描历史补跑验证。新增配置默认关闭；如最终实施引入必需的持久 schema 变化，另行提供 Yoyo 迁移与 SQLite 备份演练。

回退优先关闭新调用的融合入口，保留新结果解码器、归档 binding 与 cleanup 能力，排空在途操作。不能只回滚到不认识阶段载荷的旧版本，也不能用删回执、删消息、重置文件或杀掉所有用户进程来回退功能。

## 13. 实施前需要定稿的选择

这些选择不妨碍完成本次规划，但属于 P0 的评审产物，不是已经批准的产品事实：

1. 采用“原工具可选字段 + 来源限定依赖端口”的建议，确认允许的共享接口调整范围。
2. 接受首期只支持本地、非交互命令，并接受 running 仅保证启动顺序、不保证测试输入快照。
3. 固定结构化内容类型、失败诊断出口和 cleanup 依赖描述，确定普通路径兼容的差分范围。
4. 确认首次试点来源、任务集以及默认开关策略；Bridge 和多文件快照是否另立后续任务。

只有前述合同和验收完成后，才能把“可以适配”升级为“已安全接入”。本次已完成可行性审查和实施文档，业务实现与性能验证均未执行。
