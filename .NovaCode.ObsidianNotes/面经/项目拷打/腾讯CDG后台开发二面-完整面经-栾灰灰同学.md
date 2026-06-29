# 1. 自我介绍

# 2. 介绍agent项目

# 3. 项目里的长上下文是如何管理的?
> [!NOTE]
> pico 通过 **ContextManager + CompactManager + TurnHistoryBuilder** 三层机制控制上下文增长，不让它无限膨胀。
> 
> **第一层：ContextManager — 预算驱动裁剪**
> 
> 每一个 prompt 段都有独立字符预算：
> 
> ```
> prefix          12K chars   (系统规则 + 工具定义 + 工作区快照)
> memory           8K chars   (工作记忆 + todo + checkpoint)
> skills           4K chars   (可用技能描述)
> relevant_memory  6K chars   (关键词匹配到的最多 3 条笔记)
> history         30K chars   (对话历史转录)
> current_request  不裁剪      (用户本轮输入)
> 总计上限        60K chars
> ```
> 
> 超预算时按 relevant_memory → skills → [[history]] → memory → prefix顺序逐段削减到各自的地板值（默认预算的 25%），`current_request` 永不裁剪。
> 
> **第二层：CompactManager — 历史结构化压缩**
> 
> 当累积历史超出预算阈值时，`compact()` 按 turn_id 分组，只保留最近 N 轮（默认 2 轮）原文，把更早的全部压缩成一条 `compact_summary`：
> 
> ```
> Compacted session summary:
> - Goal: 用户的最新的请求
> - Files read: config.py, main.py
> - Files modified: src/handler.go
> - Key decisions: 选择了方案A而不是方案B
> - Current progress: 压缩了 23 条历史条目
> - Next step: 从最新轮次继续
> ```
> 
> 这条 summary 以 `system` 角色插入历史，替代被压缩的旧轮。模型看到的上下文不再是线性增长，而是 **常数级的有损摘要 + 最近 N 轮原文**。
> 
> **第三层：TurnHistoryBuilder — 按轮差异化渲染**
> 
> 历史转录渲染时区分最近轮和旧轮：
> 
> ```
> 最近 3 轮:   全文保留 (单行上限 900 chars，工具结果完整)
> 旧轮:        重复 read_file 合并，改为文件摘要引用
>              run_shell 缩为"cmd -> 前三行输出"
>              单行上限 80 chars
> ```
> 
> **第四层：自动触发**
> 
> `ContextUsageAnalyzer` 实时跟踪 token 使用量。每次构建 prompt 前检测到超预算且历史超过 4 条，自动触发 compact。后续如果再超，再次压缩旧轮——这是一个有损但收敛的压缩循环，不会无限递归。
> ### 2. Layered Memory（`features/memory.py`）— 分级记忆，避免上下文爆炸
> |层级|生命周期|用途|进入上下文方式|
> |---|---|---|---|
> |Working Memory|会话级|临时事实、当前任务状态|全量注入（预算内）|
> |Daily Logs|天级|会话摘要、关键决策|按天文件，按需检索|
> |Durable Topics|永久|4 类：project-conventions / key-decisions / dependency-facts / user-preferences|语义检索 → `relevant_memory`|
> |MEMORY.md|永久|索引文件，指向所有 topic|人类可读，agent 可更新|

# 4. 你提到压缩，是做的滑动窗口还是摘要?
> [!NOTE]
> **两者都不是，而是"结构化摘要 + 保留最近 N 轮原文"的混合策略。**
> 
> 看 `compact.py` 的核心逻辑：
> 
> ```python
> def compact(self, trigger="manual", keep_recent_turns=2):
>     # 按 turn_id 分组
>     groups = self._group(history)
>     # 保留最近 keep_recent_turns 组完整
>     compacted_turns = groups[:-keep_recent_turns]
>     kept_turns = groups[-keep_recent_turns:]
> ```
> 
> 具体做法：
> 1. **早期轮次（默认 2 轮前）** → 调用 `_summary_text()`，**非 LLM 摘要**，而是**规则抽取**：遍历 `user/tool/assistant` 消息，提取 `files_read`、`files_modified`、`user_requests[-1]`、`assistant_notes[-1]`，组装成固定模板：
>    ```
>    Goal: xxx
>    Files read: a.py, b.py
>    Files modified: c.py
>    Key decisions: xxx
>    Current progress: compacted N history items
>    Next step: continue from latest preserved turn
>    ```
> 
> 2. **最近 N 轮（默认 2 轮）** → **原文完整保留**，不压缩
> 
> 3. 合并为 `[summary_item, *kept_items]` 写回 `session["history"]`
> 
> **不是滑动窗口**（滑动窗口直接丢弃早期内容，这里保留了抽象摘要）；
> **也不是 LLM 摘要**（没有额外调用模型，而是规则提取关键元数据）。
> 
> 这种设计的权衡是：**压缩成本为 0（不消耗 token），但丢失了早期轮次中的细粒度上下文细节**。所以 `/compact` 后如果需要早期细节，依赖 memory 系统（durable topics）来保留关键决策。

# 5. 具体怎么摘要?


# 6. relevant memory是怎么从多轮笔记里召回的?
> [!NOTE]
> 关键在 `memory.py` 第 1086-1114 行的 `retrieval_candidates()`：
> 
> **召回策略**——三重打分，不使用 embedding：
> 
> ```python
> def retrieval_candidates(state, query, limit=3, workspace_root=None):
>     query_tokens = _tokenize(query)   # re.findall(r"[A-Za-z0-9_]+", ...)
>     for note in episodic_notes + durable_notes:
>         note_tokens = _tokenize(note["text"]) | _tokenize(note["source"]) | note_tags
>         exact_tag_match = int(bool(query_tokens & note_tags))
>         keyword_overlap = len(query_tokens & note_tokens)
>         if exact_tag_match == 0 and keyword_overlap == 0:
>             continue   # 无任何匹配直接跳过
>         recency = _parse_timestamp(note["created_at"])
>         ranked.append(((exact_tag_match, keyword_overlap, recency), note))
> 
>     ranked.sort(key=lambda x: x[0], reverse=True)
>     return [note for _, note in ranked[:limit]]
> ```
> 
> **排序依据**（优先级递减）：
> 1. **tag 精确命中**（最高优先）— 查询词是否直接命中 note 的 tags
> 2. **关键词重叠数** — `A-Za-z0-9_+` 的 token 交集计数
> 3. **时间新旧** — `created_at` 越新越靠前
> 
> **两个来源同时搜**：
> - `episodic_notes`（当前 session 的工作记忆）
> - `durable_store.retrieval_candidates()`（`.pico/memory/` 下的 topic 文件，额外引入 topic title 做 token）
> 
> **入口触发**：`ContextManager.build()` 第 120-121 行，以用户当前消息 `user_message` 作为 query 调用，默认取 `limit=3`。
> 
> ---
> 
> **设计的取舍**：故意没有用 embedding/向量检索。文档里写得很清楚（03-context-memory-compact.md 第 57 行）：
> 
> > 这个选择不花哨，但符合 Pico 当前定位。它保证行为容易调试，出错时能直接看出为什么某条 note 被选中。代价也明显：语义召回能力弱，跨语言和同义表达不稳。

# 7. 语义向量化是怎么做的?

# 8. 意思是你使用了rag的思想，实际应用中如果就是存在检索不到的情况怎么办?(不会rag，硬说的混合检索)

# 9. [[bm25底层思想]]了解过吗? (不会，项目里没用到rag，对rag只了解流程)

# 10. 工具调用这一块，你怎么保证lm一定能够正确调用工具的?你又是怎么定义正确调用的?

> [!NOTE]
> 分两个问题回答。
> 
> ---
> 
> ### "正确调用"的定义
> 
> | 层级                  | 校验点                                                                | 代码位置                                                                  | 错误后行为                     |
> | ------------------- | ------------------------------------------------------------------ | --------------------------------------------------------------------- | ------------------------- |
> | **1. 协议格式**         | 输出必须包含 `<tool>...</tool>` 或 `<final>...</final>`，且内部是合法 JSON 或 XML | `model_output.py:parse()`                                             | → `"retry"` + 错误文案回送模型    |
> | **2. 参数 Schema**    | 参数通过 Pydantic 模型校验（类型、范围、非空）                                       | `schemas.py` + `registry.py:validate_tool()` → `model_validate(args)` | → 拒绝执行 + 返回 `example` 给模型 |
> | **3. Workspace 语义** | 路径存在性、`old_text` 唯一匹配、路径不越界等                                       | `registry.py:validate_tool()` (161-193行)                              | → 同上层，返回具体错误              |
> | **4. 执行时合规**        | 权限(permission)、工具策略(policy)、重复调用检测                                 | `tool_executor.py:run_tool()` 第 48-76 行                               | → 拒绝执行，附带原因               |
> 
> 一个调用要**全部通过**才算"正确执行"。任何一个环节失败，结果都会以 error 形式返回给模型，模型可以修正后重试。
> 
> ---
> 
> ### 怎么保证模型正确调用
> 
> **不是"保证"，而是"引导 + 纠正"机制：**
> 
> **1. Prefix 内嵌完整规范**（`runtime.py:build_prefix()` 第 461-495 行）
> ```
> Rules:
> - Return exactly one <tool>...</tool> or one <final>...</final>.
> - Tool calls must look like: <tool>{"name":"tool_name","args":{...}}</tool>
> - For write_file/patch_file with multi-line text, prefer XML style: <tool name="write_file" path="..."><content>...</content></tool>
> 
> Tools:
> - read_file(path: str, start: int=1, end: int=200) [safe] Read a file...
> - write_file(path: str, content: str) [approval required] Write a text file.
> ...
> ```
> 把每个工具的 schema、风险等级、描述都列在 prompt 里。
> 
> **2. 双格式容错**（`model_output.py:53-75`）
> 同时接受 JSON 和 XML 两种格式：
> - `<tool>{"name":"read_file","args":{"path":"README.md"}}</tool>`
> - `<tool name="write_file" path="x.py"><content>...</content></tool>`
> 
> 降低格式门槛——模型不需要死记一种格式。
> 
> **3. 出错时回送具体原因 + 示例**
> [[tool调用出错后会发生什么]]
> ```python
> # tool_executor.py:29-32
> example = agent.tool_example(name)
> message = f"error: invalid arguments for {name}: {exc}"
> if example:
>     message += f"\nexample: {example}"
> ```
> 模型收到这个 error 后可以在下一轮修正。
> 
> **4. 写文件双格式支持特殊处理**
> ```python
> # model_output.py:120-122
> if name == "write_file" and "content" not in args and body.strip():
>     args["content"] = body
> ```
> 如果模型忘了加 `content` 属性，body 文本直接兜底为 content。
> 
> **5. 重复调用检测**（`tool_repetition.py`）
> 第 45-47 行，完全相同的 tool + args 连调会被拒绝：
> ```python
> if agent.repeated_tool_call(name, args):
>     return f"error: repeated identical tool call for {name}"
> ```
> 防止模型卡死。
> 
> ---
> 
> **总结**：Pico 的策略不是"保证模型第一次就正确"，而是 **低成本容错 + 快速失败 + 具体指引**。协议格式极简（只是 XML/JSON tag），校验链分层清晰，失败信息包含示例。这套设计的前提假设是：**当前的 LLM 对 text protocol 的可靠性远高于 function calling schema**（因为不需要对齐工具定义的 embedding 表示），代价是解析阶段多了一轮 regex 匹配。

# 11. 你提到边界控制，具体是什么边界，怎么实现的?

> [!NOTE]
> PICO 的边界控制是 **6 道独立防线**，每道拦截不同层面的越界：
> 
> ---
> 
> ### 1. 工作区路径边界（`runtime.py:911-919`）
> ```python
> def path(self, raw_path):
>     path = Path(raw_path)
>     path = path if path.is_absolute() else self.root / path
>     resolved = path.resolve()
>     if os.path.commonpath([str(self.root), str(resolved)]) != str(self.root):
>         raise ValueError(f"path escapes workspace: {raw_path}")
>     return resolved
> ```
> 所有文件类工具（read_file/write_file/patch_file）都必须经过 `path()` 锚定。`../` 逃逸、符号链接跳出、绝对路径指向外部——全部在此拦截。
> 
> ---
> 
> ### 2. 工具集边界（`tool_profiles.py`）
> | Profile | 可用工具 | 适用场景 |
> |---------|----------|----------|
> | `default` | 全部 15 个工具 | 正常模式 |
> | `readonly` | 仅 read_file / search / list_files | Explore agent |
> | `plan` | readonly + write_file/patch_file（仅限计划文件） | 计划阶段 |
> | `worker` | 除 agent/run_shell/send_message 外的全部 | 子 agent |
> | `dream` | readonly + write_file/patch_file | 后台 memory 整理 |
> 
> `PermissionChecker.check()` 第 33 行：**工具不在 profile 白名单内就直接 deny**。
> 
> ---
> 
> ### 3. 权限边界（`permissions.py`）
> 按 `approval_policy` 分 4 级：
> - `auto` — 自动允许所有 risky 工具
> - `ask` — 每次 risky 操作提示用户确认
> - `never` — 拒绝所有 risky 操作
> - `plan mode` — 只允许写到指定的计划文件（第 55-64 行）
> 
> ---
> 
> ### 4. 写范围边界（`permissions.py:66-75`）
> Worker subagent 创建时可指定 `write_scope`：
> ```python
> def _check_write_scope(self, tool, args):
>     requested = self.runtime.path(args.get("path", ""))
>     for scope in self.runtime.write_scope:
>         requested.relative_to(scope)  # 必须在 scope 目录下
>         return PermissionDecision.allow("write_scope")
>     return PermissionDecision.deny("write_scope_mismatch")
> ```
> 防止 worker 写到不该碰的目录。
> 
> ---
> 
> ### 5. 工具策略边界（`tool_policy.py`）
> 两条运行时规则：
> - **先读后写**（第 42-47 行）：修改文件前必须先 `read_file` 拿到最新内容，否则拒绝
> - **禁止 shell 搜索**（第 48-54 行）：`run_shell` 里出现 `grep/find/ls` 等命令会被拦截，强制使用 `search` 工具
> 
> ---
> 
> ### 6. 沙箱边界（Linux only，sandbox runner）
> Shell 命令可通过 bubblewrap（Linux 系统调用过滤）隔离，分为 off / best_effort / required 三级。
> 
> ---
> 
> **总结**：这些边界不是同一个地方检查的，而是 `tool_executor.py` 的执行链**逐级串起来**：
> 
> ```
> validate_tool()  →  permission_checker.check()  →  policy_checker.check()  →  execute()
>    ↑                    ↑                              ↑
>  路径锚定              profile + scope                先读后写 + shell 策略
> ```
> 
> 每一层只负责一个维度的边界，失败时都返回具体原因给模型，模型可以修正后重试。

# 12. 如果大模型调用工具出错，从何得知?举一个你遇到的具体的例子或者bug

> [!NOTE]
> ### 从何得知
> 
> 工具出错后，引擎通过 **4 条路径** 向模型和开发者反馈：
> 
> 1. **Tool result 直接返回**（实时）  
>    错误文案作为 tool result 注入下一轮 prompt，模型能直接看到并修正。
> 
> 2. **`last_tool_result_metadata`**（结构化）  
>    `tool_status`、`tool_error_code`、`affected_paths` 等字段写入内存，用于后续策略判断。
> 
> 3. **Event bus → `events.jsonl`**（审计）  
>    每条 permission_decision / tool_policy_decision / tool_executed 事件持久化，含 error_code。
> 
> 4. **Trace 和 Report**（复盘）  
>    `trace.jsonl` 记录每次 tool 调用的完整上下文，`report.json` 汇总最终状态。
> 
> ---
> 
> ### 具体 bug：S21 — 补读后重试被误判为重复调用
> **表现**：模型调用 `patch_file` → 被 `prior_read_required` 拒绝 → 模型补了 `read_file` → 用**完全相同参数**重试 `patch_file` → 被 `repeated_tool_call` 再次拒绝。
> 
> **根因**（`tool_repetition.py` 原实现，第 25 行）：
> ```python
> return len(matches) >= 2  # 只看参数是否完全相同
> ```
> 重复检测不区分"上一次是错误"还是"上一次成功了"，导致模型正确修正后仍然被拦。
> 
> **修复**（`_failed_file_write_retry_is_now_informed()`，第 41-57 行）：
> ```python
> def _failed_file_write_retry_is_now_informed(current_turn, last_index, last_match):
>     content = str(last_match.get("content", ""))
>     if not content.startswith("error:"):   # 上次成功了，不放过
>         return False
>     for item in current_turn[last_index + 1 :]:
>         if item.get("name") == "read_file" and item.get("args", {}).get("path") == path
>             and not str(item.get("content", "")).startswith("error:"):
>             return True  # 错误之后补读了同一路径，允许重试
>     return False
> ```
> 
> **实质**：对 `write_file` / `patch_file` 这两类工具引入了**"错误后补读放行"例外**。这里也说明了一个设计原则——**重复检测不是无脑拦截，而是理解错误上下文后的智能放行**。

# 13. 如果大模型重复调用工具，做无意义的重试，你遇到过吗，怎么解决?

# 14. 你提到动态调整上下文，具体怎么个动态调整法?

# 15. 重复读文件这里你提到比较语义相似度，具体怎么个比较法?

# 16. 那么多主流的codingagent，为什么cc是做的最好的?(没答上来，反向答了个cc不好的地方)

# 17. 你怎么定义agent loop的出口，也就是怎么算结束?

# 18. 成功返回你是怎么定义的?

# 19. 你这里采用的是react范式，结合项目讲解一下怎么设计的，为什么不是选择planand solve其他范式?

# 20. 我知道这些范式是共存不是互斥的关系，你实现planmodel其实还是建立在react范式上的，为什么会选择这个范式，好在哪里?

# 21. ai的coding能力很强，但是也会对code review带来灾难级的困难，比如一次性生成一万行代码?你会怎么处理与约束?

> [!NOTE]
> 这是一个非常击中当下痛点的深刻问题。AI 的高产确实容易带来“代码投毒”或“技术债爆炸”的灾难。作为求职者，我会从“规范约束”**、**“工具自动化防线”**和**“Code Review 机制改良”三个维度来系统化地处理和约束 AI 生成的代码。
> 
> ## 1. 源头治理：明确 AI 编码的“责任制”与规范
> 
> 首先，必须在团队内部确立一个底线原则：**AI 只是副驾驶（Copilot），人类工程师才是最终负责人（Owner）。**
> 
> - **建立 AI 代码准入规范：** 严禁未经理解的“盲目复制”。要求开发者对 AI 生成的每一行代码的运行逻辑、边界条件和潜在副作用了然于胸。
> - **显式标记（Optional 但推荐）：** 可以在 Commit Message 或 PR 描述中要求注明 _“部分代码由 AI 生成/辅助”_。这能提醒 Reviewer 提高警惕，重点关注逻辑漏洞和安全隐患。
> 
> ## 2. 工具防线：用自动化手段拦截“低级灾难”
> 
> 人类的精力是有限的，不能把所有的审查压力都堆给 Reviewer。必须在 Code Review 之前，用严密的 CI/CD 流程过滤掉低质量的 AI 代码。
> 
> - **静态代码分析（Linter & SAST）：** AI 经常会写出符合语法但违反团队最佳实践的代码。通过配置更严格的 Linter（如 ESLint、Ruff）和安全扫描工具（如 SonarQube），强制拦截代码风格不符、硬编码凭证、SQL 注入等低级错误。
> - **高标准单元测试卡点：** 要求AI伴随 PR 提交完备的测试用例，或者利用 AI 逆向生成边界测试，并设定硬性的**测试覆盖率（Coverage）卡点**。如果 AI 写的代码过不了自动化测试，就拒绝PR。
> 
> ## 3. 改良 Code Review 的模式
> 
> - **引入 AI 预审机制（AI-Powered PR Reviewer）：** 利用基于大模型构建的 **AI Agent（如结合 RAG 技术和团队内部架构文档、最佳实践库）** 进行第一轮 Code Review。AI 能够非常快地指出诸如“内存泄漏隐患”、“未处理的异常断点”或者“数组越界”等硬伤。
>     
>     > **举个例子：** AI 很容易在循环或者矩阵运算中犯下“索引越界”（Index out of bounds）的错误，或者混淆矩阵的维度。我们可以训练一个专门的 Lint Agent，在第一关就把这些算力浪费和逻辑硬伤揪出来。
>     
> - **重塑人类 Reviewer 的关注点：**
>     
>     当自动化和 AI 预审过滤了 80% 的低级问题后，人类工程师应**聚焦于更高维度的审查**：
>     - **架构合理性：** 这段代码是否符合整体的系统设计？
>     - **业务逻辑闭环：** AI 不懂复杂的业务上下游，它实现的功能是否真正满足了业务需求？是否破坏了业务领域的内聚性？
>     - **可扩展性与可维护性：** 这段代码在半年后、一年后还能不能看懂，会不会变成新的屎山？
>         
> - **控制 PR 的颗粒度：** 限制单个 PR 的代码修改量（例如不超过 200-300 行）
>     
> 
> ## 总结
> 对抗 AI 带来的代码通胀，核心在于“用自动化工具筑高堤坝，将人类精力留给高维思考”。通过这种方式，我们不仅能约束住 AI 带来的灾难，还能把 AI 真正转化为团队交付效率的放大器。

# 22. 为什么有了cc cursor，其他公司还要研发codex等等coding agent，为什么有openclaw之后，还有hermes、openhuman?
[[不同coding agent对比]]
> [!NOTE]
> **结论先行：Claude Code很强，但它不是终态，更不是唯一解。**
> **总结一句话送给面试官：**
> 
> > Claude Code是一个强大的选手，但coding agent的竞争是多维度的——模型能力只是其中一维，入口、生态、部署方式、用户习惯、企业合规，每一维都可以成为其他产品的立足点。有差异化的需求，就有差异化的产品空间。
> ---
> 
> ### 一、定位差异：谁来用，怎么用
> 
> Claude Code是terminal-native、深度集成Anthropic模型的coding agent，默认面向**开发者个人**使用。
> 
> 但不同产品在切入点上有本质差异：
> 
> - **Cursor / Windsurf**：IDE-native，用户不需要离开编辑器。对于习惯图形界面、不熟悉CLI的开发者，体验路径完全不同。
> - **GitHub Copilot / Codex**：深度嵌入GitHub生态，对企业级workflow（PR review、issue tracking、CI/CD）的集成是Claude Code目前不具备的。
> - **Devin / SWE-agent**：主打fully autonomous，强调"给一个issue，我给你一个PR"——这是更激进的agent范式，Claude Code更偏向human-in-the-loop。
> 
> 一句话：**入口不同、用户群不同、自动化程度定位不同。**
> 
> ---
> 
> ### 二、模型层与应用层可以解耦
> 
> Claude Code本质上是**Anthropic模型 + agent框架**的垂直整合产品。
> 
> 但应用层公司完全可以：
> 
> - 调用不同底层模型（GPT-4o、Gemini、自研）
> - 针对代码任务做专项fine-tune或RLHF
> - 在推理速度、成本、隐私合规上做差异化优化
> 
> 这就是为什么Cursor可以让用户**自选模型**——他们的护城河在UI/UX和上下文管理，而不是绑定某个模型。
> 
> ---
> 
> ### 三、企业客户有不可替代的特殊需求
> 
> 很多大厂不会直接用Claude Code，原因包括：
> 
> - **数据隐私**：代码不能出内网，必须私有化部署
> - **定制化集成**：对接内部代码库、monorepo、内部工具链
> - **合规审计**：需要操作留痕、权限控制
> 
> 这类需求催生了**专属enterprise coding agent**的市场，Claude Code的SaaS模式天然不覆盖。
> 
> ---
> 
> ### 四、竞争本身是健康的
> 
> 从宏观视角看，这其实是正常的**技术平台竞争**：
> 
> - 就像有了Chrome，Firefox、Safari依然存在
> - 就像有了AWS，Azure、GCP依然蓬勃
> 
> 每个玩家的存在都在推动整体能力边界前移。Claude Code的出现反而**验证了这个赛道的价值**，而不是终结了它。

[[三个 Agent Harness 框架对比]]：OpenClaw、Hermes Agent、OpenHuman 到底差在哪？

# 23. 你提到你深度使用hermes，讲讲如何辅助你日常开发或者帮助你解决一些日常任务的。

# 24. 自我进化机制的背后具体是怎么实现的，有了解过吗?(不知道，只是爱用)

# 25. 你觉得你的项目到目前为止，不管出于时间还是成本原因，最大的遗憾是什么地方?(我理解的是哪些模块可以迭代的更好)

# 26. 讲讲最近做research也好，看文章也好，学习一个新技术也好，让你感触最深的一个地方或者一个思想是什么?(想了半天硬说了一个)
> [!NOTE]
> "最近让我感触最深的，是读DeepSeek-R1的技术报告时的一个发现。
> 
> 他们在训练早期，没有用任何人工标注的思维链数据，只用了**纯粹的结果reward**——答案对就给正分，答案错就不给。然后模型**自己涌现出了**反思、验证、回溯这些推理行为。
> 
> 这让我很震动的点是——**我们一直以为'好的推理过程'需要人来示范，但其实只要reward设计得对，模型自己会摸索出来。**
> 
> 这让我重新思考一个问题：我们在设计agent系统时，花了大量精力去设计prompt里的推理格式、CoT模板。但DeepSeek-R1的结果在暗示，也许更根本的投入应该在**reward信号的质量**上，而不是推理格式的雕琢上。
> 
> 这个思想迁移到工程上，就是：与其告诉agent'怎么想'，不如定义清楚'什么叫做对'。"

# 27. 你平时在ai辅助下如何学习?

# 28. 你觉得ai agent会发展到全托管状态吗，以后会发展到代码纯ai写的时代吗?

> [!NOTE]
> 这题考察的是**技术判断力 + 独立观点 + 不随波逐流的思考深度**。面试官不想听"会的，AI很厉害"，也不想听"不会，人类不可替代"——他想看你怎么拆解这个问题。
> 
> ---
> 
> ## 结论先行：会，但不是"替代"，而是"重新分层"
> 
> ---
> 
> ## 第一层：全托管会到来，但边界由"可验证性"决定
> 
> **核心判断：** AI全托管的速度，和该领域的**reward是否可量化**成正比。
> 
> ```
> 可验证性高  →  全托管快
> ─────────────────────────────
> 数学证明   ✅ 答案对错明确
> 代码功能   ✅ 测试用例可跑
> UI设计     ⚠️ 审美难量化  
> 架构决策   ❌ 影响滞后数月
> 业务判断   ❌ 标准因人而异
> ```
> 
> > **所以不是"代码会不会全AI写"，而是"哪类代码会先被全托管"。** 增删改查、胶水代码、测试用例——已经快到了。 系统架构、技术选型、跨团队协作——还很远。
> 
> ---
> 
> ## 第二层：全托管 ≠ 无人监督，而是"监督界面上移"
> 
> 一个我自己的框架，叫**"控制权上移定律"**：
> 
> > 每一次自动化浪潮，都不是消灭人的参与，而是把人的介入点**往上推一层**。
> 
> 历史印证：
> 
> - 汇编 → 高级语言：程序员不消失，转而设计逻辑
> - IDE自动补全 → 程序员不消失，转而做架构
> - AI写代码 → 工程师转而做**需求定义、验收标准、系统边界**
> 
> 所以"代码纯AI写的时代"更准确的描述是：
> 
> > **人类从写代码，变成定义"什么叫做写对了"。**
> 
> ---
> 
> ## 第三层：全托管的真正瓶颈不是技术，是信任建立速度
> 
> 即使AI技术已经能全托管，现实部署还受三个非技术因素制约：
> 
> **1. 责任归属问题** AI写的代码出了生产事故，谁负责？在法律框架没跟上之前，人类签字仍是刚需。
> 
> **2. 组织惯性** 大公司的code review流程、安全审计、合规要求，不会因为AI能力提升而自动消失。
> 
> **3. 长尾风险的不对称性** 99%的任务AI全托管没问题，但那1%的边界case造成的损失可能是灾难性的——在高风险领域，人类在环是保险机制，不是效率瓶颈。
> 
> ---
> 
> ## 第四层：我的真实判断（给面试官一个明确立场）
> 
> > **5年内：** 70%的routine coding全托管，工程师角色向"AI系统的产品经理"转变
> > 
> > **10年内：** 端到端功能开发可全托管，但系统级决策仍需人类
> > 
> > **长期：** 全托管会成真，但"编程"这个职业不消失——它的定义会变，变成**对AI意图的精确表达**
> 
> ---
> 
> ## 一句话升华
> 
> > "全托管不是人类退场，而是人类的工作从**执行层**迁移到**意图层**。 未来最值钱的能力，不是写代码，而是**精确地知道自己要什么**。"
好，进入角色。



# 29. 你觉得什么场景是离不开人工介入的，什么场景又是可以ai潜代的?
> [!NOTE]
> 
> ---
> 
> ## 结论先行：边界不在"任务难度"，在"错误代价的不可逆程度"
> 
> ---
> 
> ## 第一层：我的核心判断框架
> 
> 我用两个维度来划分：
> 
> ```
>               错误可逆性
>                   ↑
>     可逆  |  AI全托管  |  AI辅助+人审  |
>           |____________|_______________|
>     不可逆 |  AI辅助   |   必须人工    |
>                   ←─────────────────→
>                低风险              高风险
>                       影响范围
> ```
> 
> > **可逆 + 低风险 = AI可替代** **不可逆 + 高风险 = 离不开人**
> 
> ---
> 
> ## 第二层：AI可替代的场景
> 
> **共同特征：有明确的验收标准，错误可被快速发现和回滚。**
> 
> **1. 代码生成与重构** 增删改查、单元测试、文档生成——测试用例就是天然的reward信号，AI写错了马上跑出来。
> 
> **2. 数据处理流水线** ETL、格式转换、报表生成——输入输出结构固定，验证逻辑明确。
> 
> **3. 客服一线响应** FAQ类问题、订单查询、标准流程引导——答案有限，错了用户会再问，成本低。
> 
> **4. 内容初稿生成** 营销文案、周报初稿、代码注释——人类做最终审核，AI做草稿，错了改掉就行。
> 
> **5. 监控告警分类** 日志异常分类、优先级排序——即使分错，下一轮还有人看，不是终态决策。
> 
> ---
> 
> ## 第三层：离不开人工介入的场景
> 
> **共同特征：错误后果滞后、不可逆、或涉及价值判断。**
> 
> **1. 高风险操作的最终授权** 生产环境部署、资金转账、删除核心数据——这类操作的特点是**单向门**，执行后无法回退。AI可以准备好一切，但最后那个"确认"必须人按。
> 
> **2. 跨组织的利益协调** 技术选型影响三个团队、架构调整涉及预算——这不是信息处理问题，是**政治和信任问题**，AI没有组织中的社会资本。
> 
> **3. 边界模糊的伦理判断** 内容审核里的灰色地带、用户隐私与功能体验的取舍——标准本身是动态的、文化相关的，AI无法持续校准"当下的社会共识"。
> 
> **4. 首次进入陌生领域** 当agent面对完全没有历史轨迹的新场景，它的置信度估计是不可靠的——人类需要在场做第一批标注和纠偏，建立初始reward信号。
> 
> **5. 出了事之后的责任承担** 这是非技术因素，但现实中最硬——出了生产故障、数据泄露，法律和组织需要一个**可问责的人**，AI不能签字，不能道歉，不能被开除。
> 
> ---
> 
> ## 第四层：一个容易被忽视的中间地带
> 
> > **不是"AI做"或"人做"，而是"AI做，人定义边界"**
> 
> 我在实际开发agent系统时，发现最难设计的不是能力，而是**escalation机制**——
> 
> 什么时候agent应该停下来说"我不确定，需要你来决定"？
> 
> 这个判断本身，目前还必须由人来设计和调优。AI的自知之明，是现阶段最大的工程挑战之一。
> 
> ---
> 
> ## 一句话总结
> 
> > AI替代的是**有标准答案的执行**，人工保留的是**标准本身的定义**和**不可逆后果的授权**。 我在做agent开发时，核心设计原则就是：**让AI跑得快，让人刹得住。**
> 