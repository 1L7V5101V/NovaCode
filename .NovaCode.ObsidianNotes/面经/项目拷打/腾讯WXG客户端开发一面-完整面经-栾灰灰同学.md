一、自我介绍和校园情况
二、项目:主要问agent，因为组里在做agent

# 1. 开发最基础的agentloop用了多久，做了哪些准备工作
> [!NOTE]
> 基础的 agent loop（就是现在 `engine.py:run_turn()` 里那个 `build prompt → call model → parse output → execute tool → repeat` 的主循环）从开始写到能跑通一个完整的 `tool → tool → final` 链路，大概用了 **3–4 天**。
> 
> 准备工作主要做了几块：
> 
> 1. **定协议边界** — 决定了用 `<tool>...</tool>` / `<final>...</final>` 文本协议而不是 provider 原生 tool-use。这个选择直接影响后面所有代码怎么写。花了一天调研和实验，确认了 Anthropic / OpenAI / DeepSeek 三个协议都能用同一套 parser 覆盖。
> 
> 2. **拆 runtime 和 engine** — 写循环之前先确定了 `Pico`（状态持有）和 `Engine`（控制流推进）的边界。如果一开始就把 session、memory、tools 全塞进 while 循环里，后面每个功能都会变成耦合。这个拆分大概花了大半天画图、验证。
> 
> 3. **搭 provider 抽象层** — 循环的核心是 `complete_model()` 调用，需要一个统一的 `model_client.complete(prompt, max_tokens)` 接口，不管底层是 REST 还是 SDK。写 base client 和两个 provider adapter 用了差不多一天。
> 
> 4. **写最小的 parser 和工具框架** — 模型吐出来的文本要能变成可执行动作，同时工具执行结果要能喂回下一轮 prompt。parser + 一个 `read_file` 工具的 demo 链路又花了一天。
> 
> 所以整体从零到第一个端到端可用大概是 **一周**：头两天做协议和架构准备，中间两天搭核心循环骨架，后面几天补了工具执行、session 持久化和错误处理。如果只是最小可用的 demo 循环（不考虑 checkpoint、memory、sandbox、worker），**3–4 天** 就能跑通。

# 2. 上下文配置为什么是12组

# 3. 上下文怎么分层

# 4. 如何保证压缩之后不丢失关键信息

# 5. [[token是什么，为什么不好的设计会浪费token]]

# 6. 两类子agent，explore和worker的边界如何划分的

> [!NOTE]
> 边界划分在 `worker_runtime.py:build_child_runtime()` 和 `tool_profiles.py`，核心是三层：
> 
> **1. 读写能力**
> - **Explore**：`read_only=True`，tool profile 用 `"readonly"`，只能调文件读取/搜索这类只读工具
> - **Worker**：`read_only=False`（除非没给 `write_scope`），tool profile 用 `"worker"`，可以写文件，但不能调 `run_shell`、子 agent 和交互类工具
> 
> **2. 审批策略**
> - **Explore**：`approval_policy = "never"`，完全静默，适合 coordinator 派出去做快速调研
> - **Worker**：`approval_policy = "auto"`，写操作自动通过，不需要用户确认
> 
> **3. 行为约束**
> - 两者都不能再 spawn 子 agent（coordinator 工具被排除）
> - Worker 有 `write_scope` 边界，写文件被锚定在限定目录内
> - Plan mode 下只允许派 Explore，不允许 Worker
> - Worker 可以走后台线程（如果有 `model_client_factory`），Explore 目前是同步执行
> 
> 本质上 Explore 是"只读侦察兵"，Worker 是"有写范围的执行者"。这个划分让 coordinator 可以安全地派 Explore 出去做代码调研而不担心副作用，派 Worker 去做有边界的改代码任务。

# 7. 为什么要这么划分

# 8. 是靠提示词约束还是规则
[[主流codding agent的子agent边界划分]]
> [!NOTE]
> **规则为主，提示词为辅**，双层防御。
> 
> 提示词层面：`build_prefix()` 只把 `available_tools()` 过滤后的工具列表写进 system prompt——Explore 看不到写工具，Worker 看不到 `run_shell` 和子 agent 工具。这是"软约束"，让模型按直觉选择合法工具。
> 
> 执行层面有四道硬规则：
> 
> 1. **Tool profile** — `tool_executor.py:13` 查注册表，`permissions.py:33` 用 `profile.allows()` 二次校验，即使模型输出了不允许的工具名也会在执行时拒绝
> 2. **read_only 锁** — Explore 子 agent 的 `read_only=True`，`permissions.py:45-46` 直接拒绝所有有副作用的工具，跟 profile 无关
> 3. **approval_policy** — Explore = `never` 全拒绝，Worker = `auto` 自动放行，这是写死的构造参数
> 4. **write_scope** — Worker 的文件写操作还要过 `_check_write_scope()`，路径不在 scope 内直接拒绝
> 
> 所以模型即使通过 prompt injection 或者格式错误试图调用一个不在列表里的工具，执行层的 PermissionChecker 和 tool registry 会挡下来。提示词只是引导正常路径，安全边界在执行层。

# 9. 如何触发subagent的调用

> [!NOTE]
> 完整的触发链路是 **模型输出 → parser 识别 → tool 执行管道 → worker_manager.spawn**：
> 
> **1. 模型输出** — 模型在当前回合输出 `<tool>{"name":"agent","args":{"subagent_type":"Explore","prompt":"...","description":"..."}}</tool>`
> 
> **2. engine 解析派发** — `engine.py:312-333` 收到 `kind="tool"`，进入 `execute_tool_payload()`
> 
> **3. 标准 tool 执行管道** — `tool_executor.py:80` 调用 `tool.execute(args)`，命中注册表里 `"agent": tool_agent`
> 
> **4. tool_agent 委派** — `agents.py:45-53` 一行代码：`agent.worker_manager.spawn(description, prompt, subagent_type, write_scope)`
> 
> **5. worker_manager.spawn** — `worker_manager.py:38-48`：
>    - 用 `build_child_runtime()` [[构造子 `Pico` runtime]]（区分 Explore/Worker 的 read_only、approval_policy、tool_profile 全在这里定）
>    - 如果父 agent 有 `model_client_factory`，走后台线程异步执行；否则[[同步阻塞]]
>    - `run_worker()` 调用 `task.runtime.ask(prompt)` — 子 agent 自己跑一整套 `run_turn()` 循环
>    - 完成后把结果塞进 notification queue，父 agent 下一轮 [[drain]] 时拿到结果
> 
> 所以本质上 **subagent 的触发和任何其他工具（read_file、run_shell）走的是完全相同的路径** — 唯一的区别是 `agent` 工具的执行函数不是读文件或跑命令，而是造出一个子 Pico 实例跑它自己的 turn loop。

# 10. 安全边界如何实现的

# 11. 项目开源了吗

# 12. 你觉得你的项目和成熟产品比，有哪些亮眼的地方

> [!NOTE]
> **Checkpoint 的 stale 检测** 不是无脑存 checkpoint。续接时比较 `runtime_identity`（model、approval_policy、tool_signature、feature_flags 等）和工作区指纹，能精确告诉你哪些文件 stale 了、哪些 runtime 参数变了。这比"resume 后重新跑一遍"或者"resume 后直接信任旧 checkpoint"都更精确。
> 平心而论：功能层面没有哪个是其他产品做不了的。真正亮眼的是 **"在小体积和少依赖的约束下，把架构边界画清楚了"**——文本协议统一 provider、三层证据解耦、控制面分离、checkpoint stale 检测有 structured 的 runtime_identity 比较。这些设计选择让整个代码库 ~1350 行就能解释清楚一个 agent 从请求到证据的全过程，而不是藏在一万个文件和 SDK 抽象层后面。

# 13. 你自己写的这个codingagent平时会使用吗

> [!NOTE]
> 会用，但不是所有场景都用。
> 
> 日常大部分工作还是用 Claude Code 和 Codex——它们模型好、迭代快、生态完善，没必要和自己较劲。
> 
> 但这个项目在我的工作流里有几个不可替代的位置：
> 
> **实验室保密项目。** 接的是内部课题，代码和数据都不能出实验室。Claude Code 和 Codex 再强，数据等到别人服务器上就是红线。这时候 pico 接本地部署的模型（比如 DeepSeek 或者 ollama 跑的模型），所有请求都在内网，没有数据泄露问题。
> 
> **还有就是我需要"能理解的 agent"。** 用商业产品的时候，碰到诡异的 behavior——比如模型反复调用同一个工具、context 莫名其妙超了——你只能猜它内部怎么处理的。pico 是 ~1350 行代码，我能直接读 engine.run_turn() 看每一轮发生了什么，能在 tool_executor 里加一行 print 看 permission checker 怎么拦的。这对 debug 和理解 agent 行为边界很有用，尤其当你需要跟导师或者合作者解释"这个 agent 为什么这么做"的时候。
> 
> 所以不是替代关系——商业产品做主力，pico 做保密场景和可解释性场景。

# 14. 你说你的这个agent快，具体算过指标吗，可以作为后续的迭代方向

# 15. 日常token的使用量是多少

# 16. 这里你做了测试，是去跑了测试集吗

# 17. 我很好奇你在写完loop之后，增加的每一个模块的完整设计，比如上下文，是按照刚刚说的写代码、跑测试吗?

> [!NOTE]
> 不是严格按照"先写测试再写代码"走的，但每个模块的路径类似：**先画边界 → 写核心逻辑 → 用真实场景验证 → 补边界测试**。我举几个模块怎么做的：
> 
> **Context 和 Compact**：我先去看 **Claude Code** 怎么处理 transcript——它的 `query.ts` 里有 auto-compact 和 microcompact 策略。我没抄它的实现（它基于 message SDK，我是文本协议），但抄了它的思路：保留最近 N 轮完整对话，把更早的对话总结成一段结构化摘要塞进 history 开头。CompactManager 的 `_group()` 按 turn_id 分组、`_summary_text()` 提取读过的文件和关键决策，这些设计直接来自看了 Claude Code transcript 管理后的启发。
> 
> **Memory 系统**：参考了 **MemGPT/Letta** 的分层记忆架构。它们把记忆分成 working memory（当前 session 的工作集）、episodic memory（具体事件的记录）、durable memory（跨 session 的长期知识）。pico 的实现简化了这套分层——`LayeredMemory` 管 working memory 和 notes，`daily_logs` 和 `dream` 做 durable memory 的持久化和整理，没有 MemGPT 那么复杂的检索系统，但分层思路是一样的。
> 
> **[[Evidence]]/[[Tracing]]**：trace 的 span_id 和 parent_span_id 字段来自 **OpenTelemetry** 的 tracing 模型。我只需要一个很轻的版本——每个 `emit_trace()` 生成 `span_{seq}` 作为 span_id，记录上一轮的 span_id 作为 parent_span_id，这样 trace 里就能还原出"哪次 prompt 导致了哪次 tool execution，又导致了哪个 checkpoint"。没有用 OTLP exporter 或 collector，只是 JSONL 落盘，但 tracing 的嵌套结构来自 OTel。
> 
> **Tool profiles / PermissionChecker**：这是自己画的安全边界。灵感来自 **Android 权限模型**——不是"这个工具有风险所以弹框"，而是"当前处于什么 mode，允许哪些工具集"。plan mode 下只暴露 readonly + 写 plan artifact，worker profile 去掉 agent 工具防止递归 spawn，readonly profile 去掉所有 coordinator 和 mode 工具。每个 profile 是显式声明的工具集合，不是在 prompt 里"请尽量只用读工具"。
> 
> 每个模块写完核心逻辑之后，我做的不是写单元测试——而是先跑一个真实场景（比如"帮我重构这个模块"），看 prompt 里出现了什么、工具调用次数、有没有不必要的 retry。问题暴露了再写对应的 test case 固化行为。所以测试是后补的，不是前驱的，但每个模块最终都有 tests/ 里的验收用例覆盖。

# 18. 你的项目跟成熟产品比没有做到的地方有哪些，为什么没有实现

# 19. 项目中你后续最想迭代的方向是什么
**1. 统一的 tool result budget** — 现在只有 `run_shell` 长输出会落 artifact，`read_file` 读大文件、`search` 扫全仓库的结果都直接塞进 history。应该做一个统一的策略：短结果进 prompt，长结果落 `artifacts/` 目录，prompt 里只放摘要和路径引用。这样 history 长度更可控，compact 的压力也小很多。

# 三、 八股

# 1. hashmap

# 2. java的垃圾回收、python的垃圾回收

# 3. 数据库索引的底层数据结构

# 4. 索引失效的场景

# 5. tqp和udp的区别

# 6. 如何保证可靠

# 7. 快重传如何实现

# 四、 手撕

# 1. 最长递增子序列

# 2. 编辑距离