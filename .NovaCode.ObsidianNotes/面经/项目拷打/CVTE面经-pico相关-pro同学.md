岗位：Agent开发 
# 1. 调用llm出现超时，报错情况如何处理 
> [!NOTE]
> - **第一层 — Provider 层（客户端自动重试）：** 捕获超时错误后默认最多重试 **2 次**，退避策略为 `0.5s → 1.0s → 1.5s`。若重试耗尽，抛出结构化的 `ProviderError`。
>     
> - **第二层 — Engine 层（全局兜底）：** 捕获上层抛出的错误后进入终端错误路径。系统不进行额外重试（遵循“失败必须被用户察觉”原则），转为优雅降级：记录错误历史并向用户展示提示 `"模型错误：timeout"`。

# 2. 项目整个运行流程 

> [!NOTE]
> 好的，我梳理一下 `pico` 项目的完整运行流程，从用户输入到最终输出：
> 
> ---
> 
> ### 1. 启动阶段
> 
> 用户执行 `pico <query>`：
> - `cli.py` 解析参数（provider、model、timeout 等）
> - `resolve_provider_config()` 按优先级解析配置：**CLI参数 > 环境变量 > .pico.toml > ~/.config/pico/config.toml > 内置默认值**
> - `_build_model_client()` 组装 `OpenAICompatibleModelClient` 或 `AnthropicCompatibleModelClient`
> - 构建 `Pico` Runtime（含 session store、memory、tool registry、event bus 等）
> - 进入 REPL（如 `--tui` 则启动 Textual TUI）
> 
> ---
> 
> ### 2. 一轮对话的执行（`Engine.run_turn()`）
> 
> 整个循环在一个 **turn loop** 中进行：
> 
> ```
> ① 构建 Prompt → ② 调用 LLM → ③ 解析响应 → ④ 执行工具 / 输出 → ⑤ 判断是否继续
> ```
> 
> **细致过程：**
> 
> | 步骤 | 位置 | 说明 |
> |------|------|------|
> | **① 构建 Prompt** | `runtime.py` | 将 system prompt + session history + tool definitions 组装成消息列表 |
> | **② 调用 LLM** | `providers/base.py` → `clients.py` | `complete_model()` → `client.complete()` → `_request_with_retries()` → `urllib.request.urlopen()` |
> | **③ 解析响应** | `providers/clients.py` | 解析 JSON，提取 content + tool_calls 等字段 |
> | **④ 处理输出** | `engine.py` | 根据模型回应类型走不同路径 |
> | **⑤ 循环或结束** | `engine.py` | 根据 turn limit 或 stop token 决定是否继续循环 |
> 
> ---
> 
> ### 3. 三种响应路径
> 
> LLM 返回的内容决定路径：
> 
> **A. 文本回复路径：**
> - 直接 yield 给用户/UI
> - 继续下一 turn 或结束
> 
> **B. 工具调用路径：**
> - `engine.py` 解析出 tool_calls
> - `tool_executor.py` 选取并执行对应 tool（读文件/写文件/Shell/ask user/todo 等）
> - 将执行结果（含 `tool_status`、`tool_error_code` 等元数据）追加回 history
> - loop 回到第①步，把工具结果喂回给 LLM 做进一步决策
> 
> **C. 错误/结束路径：**
> - 超时/空响应/认证错误 → `finish_model_error()` 记录失败并 yield `{"type": "stop"}`
> - abort 请求 → `finish_stopped_run()`
> - 达到 turn_limit → 正常结束
> 
> ---
> 
> ### 4. 后台维护
> 
> 在每个 turn 之间：
> - `maintain_memory_safely()` 维护记忆（daily log、durable topics、auto-dream 的触发条件检查）
> - 创建 checkpoint
> - 写入 trace/report 到本地磁盘
> 
> ---
> 
> ### 5. 完整流程图
> 
> ```
> 用户输入
>   │
>   ▼
> CLI 参数解析 & 配置加载
>   │
>   ▼
> 构建 ModelClient (OpenAI/Anthropic)
>   │
>   ▼
> 进入 Engine.run_turn() 循环
>   │
>   ├── build_prompt() ← system + history + tool_defs
>   │
>   ├── complete_model() ──→ _request_with_retries() ──→ LLM API
>   │        │                                              │
>   │        │ ← 异常？                                  ← JSON
>   │        ├── ProviderError.timeout → 已重试2次 → finish_model_error()
>   │        ├── ProviderError.rate_limited → 已重试 → finish_model_error()
>   │        └── ProviderError.empty_response → engine级再重试1次
>   │
>   ├── parse_response()
>   │     │
>   │     ├── 文本回复 → yield to user → 继续/结束
>   │     │
>   │     └── tool_calls → tool_executor.execute()
>   │             │
>   │             ├── 成功 → append 结果到 history → loop
>   │             └── 失败 → append 错误到 history → loop
>   │
>   └── maintain_memory_safely() (后台维护)
> ```
> 
> 关键设计原则：**不静默失败**，每一次超时/错误都有完整的 attempt count、retry count、sanitized URL 记录，让用户可见。

# 3. 感知什么信息，怎么拿到这些信息，后续如何做决策 
> [!NOTE]
> **感知信息 = 三类：**
> 1. **工具结果** — stdout/stderr/exit_code/`tool_status`
> 2. **模型错误** — `ProviderError(code, http_status, retryable, attempts, retry_count)`
> 3. **上下文** — 历史消息 + memory(daily_logs, durable_topics)
> 
> **获取方式：**
> - 工具结果：`ToolExecutor.execute()` 返回 `ToolResult` → append history
> - 模型错误：`_request_with_retries()` 异常捕获 → 映射为 `ProviderError` 结构化对象
> - 上下文：`PromptBuilder` 构建时从 `session_store` + `memory` 拉取
> 
> **决策逻辑（两层）：**
> 
> | 层 | 角色 | 输入 | 输出 |
> |----|------|------|------|
> | **LLM** | 任务推理 | 完整 prompt（含所有感知信息） | 文本回复 / tool_calls |
> | **Engine** | 元控制 | `ProviderError` + attempts/retry_count | 重试 / 终止 / 继续 |
> 
> 闭环：`感知 → LLM决策 → 执行工具 → 工具结果append回history → 下一轮LLM再感知 → 再决策`

# 4. 循环调用某个工具如何处理 

> [!NOTE]
> pico 通过 **三层面** 处理循环调用：
> 
> ### 1. 硬检测层：`tool_repetition.py`（`is_repeated_tool_call()`）
> 
> 在 `tool_executor.run_tool()` 执行前拦截（`tool_executor.py:45`）：
> 
> ```python
> if agent.repeated_tool_call(name, args):
>     return "error: repeated identical tool call for {name}; ..."
> ```
> 
> 逻辑分两路：
> 
> - **普通工具**（`read_file`, `run_shell` 等）：同一 turn 内，相同 `(name, args)` 出现 **≥2 次** → 拒绝
> - **写文件工具**（`write_file`, `patch_file`）：允许一次例外——如果上次写失败（结果以 `error:` 开头），且模型**在读了一次文件后重试**，则认为模型已经理解了问题，放行。这是 `_failed_file_write_retry_is_now_informed()` 的逻辑（第 41-57 行）
> 
> ### 2. 提示层：System prompt 引导
> 
> `runtime.py:480`，构建 prefix 时写了一条规则：
> 
> > "Do not repeat the same tool call with the same arguments if it did not help. Choose a different tool or return a final answer."
> 
> ### 3. 反馈层：拒绝结果回馈给模型
> 
> 当检测到重复时，返回的错误消息格式为：
> 
> ```
> error: repeated identical tool call for read_file; choose a different tool or return a final answer
> ```
> 
> 这条结果被 `agent.record()` 写入 history（`engine_helpers.py:36-44`），模型在下一轮看到这个错误后，应当主动换工具或返回 `<final>`。如果模型继续重复，会再次被拦截形成死循环保护——由 `max_steps` 和 `max_attempts` 兜底截断（`engine.py:110, 434-453`）。
> 
> ---
> 
> 总结链路：
> 
> ```
> 模型输出 <tool> → engine.run_turn() 解析 → execute_tool_payload() 
>   → agent.run_tool() → is_repeated_tool_call() 检查历史
>     ├─ 重复 → 返回 "error: repeated..." → 写入 history → 模型下一轮看见
>     └─ 不重复 → 正常执行 → 结果写入 history → 模型继续
> ```

# 5. 没有调对工具如何处理，给出常用的解决方案 

> [!NOTE]
> pico 对"调错工具"设计了**多层渐进式拦截**，每一层失败后都以结构化错误回写到 history，让模型在下一轮自动修正。完整链路如下：
> 
> ```
> 模型输出 <tool> → model_output.parse()
>   ├─ 格式错误(无标签) → ("retry", retry_notice)  → 模型重试
>   └─ 正确解析 → execute_tool_payload() → agent.run_tool()
> 
> agent.run_tool() 内逐层拦截:
>   Layer 1: 工具不存在           → "error: unknown tool '<name>'"
>   Layer 2: 参数校验(Pydantic)   → "error: invalid arguments: 字段类型/缺失/越界"
>   Layer 3: 重复调用检测         → "error: repeated identical tool call"
>   Layer 4: 权限(PermissionChecker)
>            ├─ ToolProfile 禁用  → "tool_not_allowed"
>            ├─ Plan 模式写保护   → "plan_mode_tool_not_allowed"
>            ├─ WriteScope 越界  → "write_scope_mismatch"
>            └─ 用户审批拒绝      → "approval_denied"
>   Layer 5: 策略(ToolPolicyChecker)
>            ├─ 写前未读文件      → "prior_read_required"
>            └─ shell 代替搜索    → "shell_search_should_use_tool"
>   Layer 6: 执行期异常           → "tool <name> failed: <异常消息>"
> ```
> 
> 每层失败结果被 `agent.record()` 写回 history（`engine_helpers.py:36-44`），Engine 控制循环回到 `while` 头部（`engine.py:110`），模型在下一轮 prompt 中看到错误，自主修正动作。
> 
> **除此之外的辅助手段**：
> 
> 1. **Prompt prefix 嵌入示例**（`runtime.py:448-458`）：每个工具的正确调用格式写在 system prompt 里，模型第一次犯错概率降低
> 2. **ToolProfile 主动缩窄选择面**：`plan` 模式只暴露读工具 + 写 plan artifact；`readonly` 模式彻底禁用写操作——让模型不可能选错
> 3. **重试兜底**：`max_attempts = max_steps + 2`（`engine.py:108`），如果模型连续输出格式错误、一直触发 `retry`，超限后强制截断：
>    ```
>    "Stopped after too many malformed model responses without a valid tool call or final answer."
>    ```

# 6. prompt压缩如何实现的，剪裁哪些内容，怎么判断的 

> [!NOTE]
> pico 的 prompt 压缩分 **3 个层次**，分别在构建 prompt 的**每个环节**和**跨轮阶段**执行：
> 
> ---
> 
> ### 层次 1：[[Section 预算裁剪]] — 每轮 prompt 组装时（`context_manager.py`）
> 
> `ContextManager.build()` 把 6 个 section 拼成 prompt，每个有独立预算：
> 
> | Section | 默认预算 | 最低底线 | 裁剪优先级 |
> |---|---|---|---|
> | `prefix`（系统规则） | 12000 | 4000 | 5（最后动） |
> | `memory`（工作记忆） | 8000 | 1200 | 4 |
> | `skills` | 4000 | 600 | 2 |
> | `relevant_memory`（相关笔记） | 6000 | 1000 | **1（最先动）** |
> | `history`（对话历史） | 30000 | 6000 | 3 |
> | `current_request`（当前请求） | — | 不裁剪 | 永不裁剪 |
> 
> **判断逻辑**（第 146-171 行）：组装完成后检查 `len(prompt) > total_budget`（默认 60K chars），如果超了按 `reduction_order = ("relevant_memory", "skills", "history", "memory", "prefix")` 逐个降预算、重新渲染，直到不超或触及底线。
> 
> ---
> 
> ### 层次 2：[[History Compaction — 跨轮压缩]]（`compact.py`）
> 
> 把完整旧轮次替换成一条 `compact_summary` 系统消息。
> 
> **触发时机**：
> - 手动：用户输入 `/compact`
> - 自动：`runtime.py:608` — prompt 超预算且 history > 4 条时触发
> 
> **怎么判断旧轮次**：`_group()` 按 `turn_id` 分组，保留最后 `keep_recent_turns=2` 组，前面的全部压缩。
> 
> **裁剪的内容**：从被压缩的旧轮次中提取结构化摘要：
> ```
> Compacted session summary:
> - Goal: <最后一条用户请求>
> - Files read: <所有 read_file 的 path>
> - Files modified: <所有 write_file/patch_file 的 path>
> - Key decisions: <最后一条 assistant 回复>
> - Current progress: compacted N items
> ```
> 
> ---
> 
> ### 层次 3：Turn-level 裁剪渲染 — 每轮渲染 history 时（`turn_history.py`）
> 
> `render_section()` 按"时间远近"做差异化处理：
> 
> | 条件 | 处理方式 |
> |---|---|
> | **最近 3 轮** | 完整保留，每行上限 900 chars |
> | **更早轮次中的 read_file** | 相同 path 只保留第一次，后续折叠；优先复用 memory 中的 file summary |
> | **更早轮次中的 tool** | 摘要化为 `command -> output(前3行)`，每行上限 80 chars |
> | **compact_summary 条目** | 原样展开，不作二次裁剪 |
> 
> **判断标准**是按 `budget` 从最新轮次往前倒推进渲染列表（第 63-76 行）：从最老到最新逐一加入，一旦加入后总长度超 budget，若该轮是 recent 则逐行 clip，否则整体丢弃。
> 
> ---
> 
> 总结：pico 的压缩策略是 **"先局部 clip → 跨轮 compaction → 逐轮汇总"** 的递进式设计，最新用户请求永不裁剪。

# 7. prompt被错误裁剪了怎么办 

> [!NOTE]
> pico 对"错误裁剪"有 **4 层防御 + 1 条逃逸路径**：
> 
> ---
> 
> ### 第 1 层：预防性兜底
> 
> | 机制 | 代码位置 | 说明 |
> |---|---|---|
> | **Section floors** | `context_manager.py:24-29` | 每个 section 有最低底线，prefix 最低 4000 chars，history 最低 6000，绝不允许裁到零 |
> | **Reduction order** | `context_manager.py:31` | `relevant_memory → skills → history → memory → prefix`，最不重要的先裁 |
> | **current_request 永不裁剪** | `context_manager.py:231-233` | 最新用户请求永远零预算，隔离开 |
> | **Auto compaction as backup** | `runtime.py:605-611` | 如果 section 裁剪后 prompt 仍然超预算且 history>4，触发一次 History Compaction 再重建 |
> 
> ---
> 
> ### 第 2 层：裁剪时打 checkpoint
> 
> ```python
> # engine.py:173-185
> if prompt_metadata.get("budget_reductions"):
>     checkpoint = agent.create_checkpoint(
>         task_state, user_message, trigger="context_reduction"
>     )
> ```
> 
> 每次本轮 prompt 被裁剪，**在发请求给模型前**先保存一个 checkpoint。这样即使模型因裁剪产生错误结论，用户可以用 `/resume` 从裁剪前的状态续接，不丢失进度。
> 
> ---
> 
> ### 第 3 层：全量可追溯
> 
> 所有裁剪决策被完整记录到 metadata → trace → report：
> 
> ```python
> # context_manager.py:343-353
> metadata = {
>     "prompt_over_budget": True,
>     "budget_reductions": [
>         {"section": "relevant_memory", "before_chars": 6000, "after_chars": 4000},
>         ...
>     ],
>     "sections": {  # 每个 section 的 raw/budget/rendered 对比
>         "history": {"raw_chars": 45000, "budget_chars": 30000, "rendered_chars": 30000},
>     }
> }
> ```
> 
> 同时 event stream 和 `.pico/runs/<run_id>/` 下保留完整 trace，人工排查时能看到每一轮"哪段被裁了多少"。
> 
> ---
> 
> ### 第 4 层：模型自身也能补
> 
> 如果裁剪导致模型缺少关键信息（比如文件内容被 clip），模型可以直接调用：
> 
> ```
> <tool>{"name": "read_file", "args": {"path": "src/main.py", "start": 1, "end": 200}}</tool>
> ```
> 
> 重新读取完整内容，不需要依赖被裁剪的 prompt 片段。
> 
> ---
> 
> ### 逃逸路径：provider 层拒绝
> 
> 如果最终 prompt 超出模型上下文窗口，provider API 返回 `prompt_too_long` / `context_length_exceeded`：
> 
> ```python
> # model_errors.py:21-24
> if code in {"prompt_too_long", "context_length_exceeded"}:
>     final = "Prompt 超出模型上下文窗口。建议：/compact 压缩历史、或 /clear 开新 session。"
> ```
> 
> 用户看到这条消息后手动执行 `/compact` 或 `/clear` 恢复。
> 
> ---
> 
> **总结**：pico 不保证裁剪永远正确，但保证"裁剪可感知、可恢复、可调试"——不会静默丢失重要上下文。

# 8. memory具体存什么内容，如何工作 
[[memory]]

# 9. check point具体怎么恢复的 

> [!NOTE]
> 三阶段：**写入 → 加载校验 → 注入 prompt**。
> 
> ---
> 
> ### 阶段一：写入 checkpoint
> 
> `create_checkpoint()`（`runtime_checkpoints.py:44-74`）在 5 个时机被调用：
> 
> | 触发时机 | 位置 | trigger |
> |---|---|---|
> | 每次工具执行后 | `engine_helpers.py:63` | `"tool_executed"` |
> | Section 预算裁剪后 | `engine.py:173-185` | `"context_reduction"` |
> | 文件摘要过期 | `engine.py:135-138` | `"freshness_mismatch"` |
> | 工作区变更 | `engine.py:161-163` | `"workspace_mismatch"` |
> | Turn 正常结束 | `engine.py:388-390` | `"run_finished"` |
> 
> 写入的内容：
> 
> ```python
> checkpoint = {
>     "checkpoint_id": "ckpt_a1b2c3d4",
>     "parent_checkpoint_id": "ckpt_e5f6g7h8",     # 链式链接
>     "schema_version": "phase1-v1",
>     "created_at": "...",
>     "current_goal": "找出测试失败的根因",           # 用户消息
>     "current_blocker": "",                         # 如果未完成则记录 stop_reason
>     "next_step": "Resume from the latest checkpoint...",
>     "key_files": [                                  # recent_files 的 freshness 哈希
>         {"path": "src/main.py", "freshness": "sha256:..."}
>     ],
>     "runtime_identity": {                           # 当前运行环境指纹
>         "cwd": "...", "model": "...", "approval_policy": "ask",
>         "workspace_fingerprint": "sha256:...",
>         "tool_signature": "sha256:..."
>     },
>     "summary": "context_reduction: 找出测试失败的根因",
> }
> ```
> 
> checkpoint 保存在 `session["checkpoints"]["items"][id]` 中，链式链接通过 `parent_checkpoint_id` 维护，每次写入后立即 persist 到磁盘 `.pico/sessions/<id>.json`。
> 
> ---
> 
> ### 阶段二：加载时校验（`evaluate_resume_state()`, `runtime.py:296-360`）
> [[resume 如何实现]]
> `/resume latest` 或 `--resume latest` 时触发：
> 
> ```
> session 加载 → _rebind() → evaluate_resume_state()
> ```
> 
> 逐项比较 checkpoint 和当前运行环境的差异：
> 
> ```python
> def evaluate_resume_state(self):
>     # 1. 模式版本
>     if checkpoint["schema_version"] != "phase1-v1":
>         status = "schema-mismatch"
> 
>     # 2. 文件新鲜度
>     for item in checkpoint["key_files"]:
>         current = file_freshness(item["path"])
>         if expected != current and path not in stale_paths:
>             stale_paths.append(path)
> 
>     # 3. 运行时身份
>     for key in ("cwd", "model", "approval_policy",
>                 "workspace_fingerprint", "tool_signature", ...):
>         if saved[key] != current[key]:
>             mismatch_fields.append(key)
> 
>     # 综合判断
>     if stale_paths:          status = "partial-stale"
>     elif mismatch_fields:    status = "workspace-mismatch"
>     else:                    status = "full-valid"
> ```
> 
> **5 种状态**：
> 
> | Status | 含义 |
> |---|---|
> | `no-checkpoint` | 没有 checkpoint（首次运行或刚刚 `/clear`） |
> | `full-valid` | 一切匹配，可以完整续接 |
> | `partial-stale` | key_files 有的已被修改，文件摘要已失效 |
> | `workspace-mismatch` | 模型/目录/工具集等运行环境变了 |
> | `schema-mismatch` | checkpoint 版本不兼容 |
> 
> ---
> 
> ### 阶段三：注入下一轮 prompt 中
> 
> `runtime.py:362-396` `render_checkpoint_text()` 将校验结果渲染为文本，挂载到 `memory` section 尾部（`context_manager.py:112-116`）：
> 
> ```
> Task checkpoint:
> - Resume status: partial-stale
> - Current goal: 找出测试失败的根因
> - Current blocker: -
> - Next step: Resume from the latest checkpoint and continue the task.
> - Key files: src/main.py, tests/test_main.py
> - Completed: 读取了测试文件和主逻辑
> - Stale paths: src/main.py
> - Summary: tool_executed: 找出测试失败的根因
> ```
> 
> 模型在提示中看到这段后，知道：
> 1. **之前的任务是什么**（current_goal）
> 2. **卡在哪里**（current_blocker）
> 3. **哪些文件需要重新读取**（Stale paths）
> 
> 同时 `evaluate_resume_state()` 还触发 engine 中的额外动作（`engine.py:135-172`）：
> 
> - **partial-stale** → 在发请求前先创建新 checkpoint，标记过期文件
> - **workspace-mismatch** → emit trace 记录变更字段，创建 checkpoint
> - **context_reduction** → 创建 checkpoint 记录裁剪前的状态

# 10. 恢复失败了，除了查历史记录外还有什么机制

> [!NOTE]
> 从代码追踪来看，当 checkpoint 恢复失败时，pico 的自动恢复机制很有限——设计上偏向**"快速失败 + 人工介入"**。具体有这几层：
> 
> ---
> 
> ### 当前实际上有的
> 
> **1. `list_sessions()` 的软跳过**（`session_store.py:53-56`）
> 
> ```python
> for path in ...:
>     try:
>         session = json.loads(path.read_text(...))
>     except (OSError, json.JSONDecodeError):
>         continue   # 坏文件直接跳过，不影响其他 session 列表
> ```
> 
> **2. `build_agent()` 的 ValueError 兜底**（`cli.py:609-613`）
> 
> ```python
> try:
>     agent = build_agent(args)
> except ValueError as exc:
>     print(str(exc), file=sys.stderr)    # 打印错误信息
>     return 2                             # 进程退出
> ```
> 
> 只捕获 `ValueError`（如路径逃逸），`FileNotFoundError`、`json.JSONDecodeError` 等会直接崩溃。
> 
> **3. 独立于 session 的持久记忆**（`.pico/memory/`）
> 
> 即使 session JSON 损坏、checkpoint 全部丢失，durable memory 的 topic 文件仍然在磁盘上。用户 `/clear` 开新 session 后，`relevant_memory` 检索依然能从 topic 文件中拉出历史知识。
> 
> ---
> 
> ### 没有实现的
> 
> | 机制 | 状态 | 说明 |
> |---|---|---|
> | **Event stream 重建 history** | 无 | `.events.jsonl` 存有完整事件流（`session_events.py:20-28`），但只在 `run_evidence.py` 中做离线分析，不用于恢复 session state |
> | **`STOP_REASON_RESUME_LOAD_ERROR`** | 死代码 | `task_state.py:23` 定义了常量，但没有任何代码引用过 |
> | **自动 fallback 到新 session** | 无 | `build_agent()` 中 if session_id → `Pico.from_session()`，没有 try/except → `Pico()` 的兜底路径 |
> | **Session 文件备份** | 无 | `session_store.save()` 是直接 `write_text` + `os.replace`，没有保留上一版本 |
> 
> ---
> 
> ### 实际恢复路径
> 
> ```
> session JSON 损坏 → build_agent() 抛出异常
>   └─ main() 未捕获特定错误 → 进程崩溃
>       └─ 用户再次运行 pico（不传 --resume）
>           └─ 创建新 session
>               └─ durable memory 仍在，模型可从 memory section 中找回历史上下文
> ```
> 
> 所以依赖的就是"启动新 session 不丢持久记忆"这个设计冗余。如果用户需要精确的上一轮 history，只能去 `.pico/sessions/<id>.events.jsonl` 中手动翻看。

# 11. [[MCP function call区别]] 

# 12. 项目中哪里用了function call 和mcp 

> [!NOTE]
> **pico 项目中没有使用 [[MCP]]，也没有使用 [[API 原生 function call]]。**
> 
> 从 provider 层的请求体可以清楚看到（`clients.py`）：
> 
> **OpenAI client** — 只发纯文本，无 `tools` 参数：
> ```python
> # clients.py:312-327
> payload = {
>     "model": self.model,
>     "input": [{
>         "role": "user",
>         "content": [{"type": "input_text", "text": prompt}]
>     }],
>     "max_output_tokens": max_new_tokens,
>     # 没有 "tools": [...]  ← 没有 function calling
>     # 没有 "tool_choice"   ← 没有 function calling
> }
> ```
> 
> **Anthropic client** — 同样只发纯文本：
> ```python
> # clients.py:462-476
> payload = {
>     "model": self.model,
>     "messages": [{
>         "role": "user",
>         "content": [{"type": "text", "text": prompt}]
>     }],
>     "max_tokens": max_new_tokens,
>     # 没有 "tools"         ← 没有 tool_use
> }
> ```
> 
> **处理响应的方式** — 提取纯文本，然后用正则解析：
> 
> ```python
> # providers/base.py:12-17
> def complete_model(model_client, prompt, max_new_tokens, **kwargs):
>     text = model_client.complete(prompt, max_new_tokens, **kwargs)  # 纯文本返回
>     return ModelResult(text=str(text), metadata=...)
> 
> # model_output.py:7-23
> def parse(raw):
>     if "<tool" in raw:
>         parsed = re.findall(r"<tool\b(.*?)>(.*?)</tool>", raw)  # 正则解析
>         ...
>     if "<final>" in raw:
>         return "final", extract(raw, "final")  # 专门标签
> ```
> 
> 所以结论：
> 
> | 技术 | pico 中的状态 |
> |---|---|
> | **MCP (Model Context Protocol)** | 未使用，代码中无任何引用 |
> | **API 原生 function call** (`tools` 参数) | 未使用，请求体中不传 |
> | **pico 自己的方案** | **纯文本 XML 协议**：模型读 system prompt 中的工具描述 → 输出 `<tool>...</tool>` 标签 → 正则解析 → 执行 → 结果写回 history |
> 
> 这样做的好处是 provider 无关——无论 OpenAI、Anthropic、DeepSeek，只要返回文本就能用，不需要 provider 各自实现 tool_use 协议。代价是模型需要从 prompt 中学习 XML 格式，偶尔会格式错误触发 `"retry"`。

# 13. 开发一个新工具，涉及到数据库账号，用mcp还是function call 那个合适

> [!NOTE]
> 这个问题需要从两个维度看：**pico 的现有架构** vs **通用最佳实践**。
> 
> ---
> 
> ### 在 pico 现有架构下
> 
> pico 既不支持 MCP 也不支持 API 原生 function call，新增工具的标准化路径是：
> 
> ```python
> # 1. 在 schemas.py 中定义 Pydantic 参数模型
> class QueryDBArgs(BaseModel):
>     sql: str
>     max_rows: int = 100
> 
> # 2. 在 registry.py 中注册 schema + specs + runner
> _TOOL_SCHEMAS["query_db"] = QueryDBArgs
> BASE_TOOL_SPECS["query_db"] = {
>     "schema": {"sql": "str", "max_rows": "int=100"},
>     "risky": True,      # ← 标记为高风险，触发审批
>     "description": "Execute SQL query on the project database.",
> }
> _TOOL_RUNNERS["query_db"] = tool_query_db
> 
> # 3. 数据库凭证通过 PICO_SECRET_ENV_NAMES 声明，自动从 trace/report 中脱敏
> ```
> 
> 然后通过现有的 `PermissionChecker` + `approval_policy` 控制访问：
> - `approval=ask`：每次执行问用户
> - `approval=auto`：自动放行（适合 CI）
> - `approval=never`：彻底禁止
> 
> ---
> 
> ### 如果从零设计，MCP vs Function Call
> | 维度       | MCP                                                      | API 原生 function call              |
> | -------- | -------------------------------------------------------- | --------------------------------- |
> | **凭证隔离** | 数据库账号配置在 MCP server 自己的环境里，agent 进程永远接触不到明文密码            | 凭证必须暴露给 agent runtime 或其所在进程      |
> | **进程隔离** | agent 崩溃不影响 DB 连接；MCP server 可独立重启、长连接池复用                | 工具在 agent 进程内运行，DB 连接随 agent 生命周期 |
> | **可复用性** | 同一个 MCP server 可供 Claude Desktop、Cursor、任意 MCP client 使用 | 绑定到特定 LLM provider 和特定 agent 框架   |
> | **审计边界** | DB 操作通过 JSON-RPC 进出，可在 server 侧做独立访问日志                   | 审计依赖 agent 自身的 event/trace 系统     |
> | **复杂度**  | 需要维护独立进程 + 通信协议                                          | 写一个 Python 函数即可                   |
> 
> **结论**：如果数据库账号是敏感的生产凭证，MCP 更合适——它提供了**进程级隔离 + 凭证零暴露**。如果只是开发/测试库且团队已用 pico，走 pico 现有工具注册流程即可，已有的 `PermissionChecker` + 环境变量脱敏机制足够覆盖安全需求。


