从用户输入 `/resume latest` 到模型输出第一个 token，内部经历了 **6 个阶段**：

---

### 阶段 1：解析 session ID（`cli.py:563-575`）

```python
def _resolve_session_id(agent, target):
    if target == "latest":
        return agent.session_store.latest()   # 找 .pico/sessions/ 下最新 .json
    if target.isdigit():
        return list_sessions()[int(target)]["id"]
    return target    # 直接是 session id
```

---

### 阶段 2：清场（`session_lifecycle.py:14-15`）

```python
def resume_runtime_session(runtime, session_id):
    _shutdown_workers(runtime)    # 关闭所有子 agent 进程
```

---

### 阶段 3：从磁盘加载完整 session（`session_store.py:35-37`）

```python
runtime.session = json.loads(
    Path(".pico/sessions/<id>.json").read_text(encoding="utf-8")
)
```

加载的内容：

```
session = {
  "id": "20260610-143022-a1b2c3",
  "history": [user_msg, assistant, tool_call, tool_result, ...],  # 全量对话记录
  "memory": { working, episodic_notes, file_summaries, ... },     # 工作记忆快照
  "checkpoints": {                                                 # checkpoint 链
    "current_id": "ckpt_a1b2",
    "items": {
      "ckpt_a1b2": { ... }  # 包含 key_files, runtime_identity
    }
  },
  "runtime_identity": { ... }  # 上次运行时的环境指纹
}
```

---

### 阶段 4：重建运行时（`session_lifecycle.py:34-66` `_rebind()`）

```python
# 重新绑定事件总线（写入旧的 events.jsonl）
runtime.session_event_bus = SessionEventBus(...)

# 从 session["memory"] 恢复 LayeredMemory
runtime.memory = LayeredMemory(session["memory"], workspace_root=root)

# 重建运行时组件
runtime.plan_mode = PlanModeController(runtime)
runtime.todo_ledger = TodoLedger(runtime)
runtime.worker_manager = WorkerManager(runtime)

# 校验 checkpoint
runtime.resume_state = runtime.evaluate_resume_state()
# 状态可能是: full-valid / partial-stale / workspace-mismatch / schema-mismatch

# 重建 system prompt prefix（fresh）
runtime.refresh_prefix(force=True)

# 清空运行态
runtime.current_turn_id = ""
runtime.current_task_state = None
```

---

### 阶段 5：用户发消息（如"继续"）

此时在 REPL 中：

```
pico> 继续
```

调用 `agent.ask("继续")` → `engine.run_turn("继续")`

---

### 阶段 6：prompt 组装 + 模型续接

`ContextManager.build()` 把 6 个 section 拼成 prompt。其中 history section 包含了**上一轮完整的对话记录**（模型输出、工具结果等），memory section 尾部附带 checkpoint 信息：

```text
[... prefix, memory, skills, relevant_memory ...]

Transcript:
Turn task_001:
[user] 找出测试失败的根因
[tool:read_file] {"path": "tests/test_main.py"}
    1: def test_foo():...
[tool:run_shell] {"command": "pytest"}
    exit_code: 1
    stdout: FAILED tests/test_main.py::test_foo
[assistant] ...
[final] ...
Turn task_002:                              ← 上一轮到 step_limit 截断的部分
[user] 继续上次的任务
[tool:read_file] {"path": "src/main.py"}
    ...

Task checkpoint:
- Resume status: full-valid
- Current goal: 找出测试失败的根因
- Current blocker: step_limit_reached
- Next step: Resume from the latest checkpoint and continue the task.
- Key files: src/main.py, tests/test_main.py
- Completed: 读取了测试和主逻辑

Current user request:
继续
```

模型看到完整的 previous history + checkpoint 摘要 + "继续"，就从上次 tool_steps 停下的地方接着工作。

---

### 关键区别：`--resume` vs `/resume` 在同一个 session 内

```
CLI --resume latest:  新进程 → 磁盘加载 session → history完整保留 → 用户发消息
REPL /resume <id>:    同一进程 → 磁盘加载另一个session → 替换当前 session → 用户发消息
```




两条入口，最终汇聚到同一个内部流程：

---

### 入口一：CLI `--resume latest`

```python
# cli.py:204-213
session_id = args.resume                # "--resume latest"
if session_id == "latest":
    session_id = store.latest()         # 找 .pico/sessions/ 下最新 .json 文件
if session_id:
    return Pico.from_session(
        model_client, workspace, store, session_id, **kwargs
    )
# 没有 --resume → 走 Pico() 构造函数，新 session
```

`from_session()` 只是 `Pico.__init__` 的包装，传入从磁盘加载的 session 数据代替默认空 session：

```python
# runtime.py:220-228
@classmethod
def from_session(cls, ..., session_id, **kwargs):
    return cls(
        ..., 
        session=session_store.load(session_id),   # 从磁盘加载完整 session JSON
        **kwargs,
    )
```

---

### 入口二：REPL `/resume latest`

```python
# cli.py:440-446
if user_input.startswith("/resume "):
    session_id = _resolve_session_id(agent, target.strip())
    agent.resume_session(session_id)
```

`resume_session()` 委托给 `session_lifecycle.py`：

```python
# runtime.py:811-812
def resume_session(self, session_id):
    return resume_runtime_session(self, session_id)
```

---

### 核心流程：`resume_runtime_session()` + `_rebind()`

```python
# session_lifecycle.py:14-18
def resume_runtime_session(runtime, session_id):
    _shutdown_workers(runtime)                # 1. 关闭旧 worker
    runtime.session = runtime.session_store.load(session_id)  # 2. 从磁盘加载
    _rebind(runtime, emit_started=False)      # 3. 重建运行时
    return runtime.session["id"]
```

`_rebind()` 是完整的"运行时重建"过程：

```python
# session_lifecycle.py:34-66
def _rebind(runtime, emit_started):
    runtime._ensure_session_shape()           # 填充 session 默认字段

    # 重建事件总线（绑定到新 session 的 events.jsonl）
    runtime.session_event_bus = SessionEventBus(
        runtime.session["id"],
        runtime.session_store.event_path(runtime.session["id"]),
        ...
    )

    # 重建记忆系统（从 session["memory"] 恢复）
    runtime.memory = LayeredMemory(
        runtime.session.setdefault("memory", default_memory_state()),
        workspace_root=runtime.root,
    )

    # 重建运行时组件
    runtime.plan_mode = PlanModeController(runtime)
    runtime.todo_ledger = TodoLedger(runtime)
    runtime.worker_manager = WorkerManager(runtime)

    # 校验 checkpoint + 刷新 prefix
    runtime.resume_state = runtime.evaluate_resume_state()   # ← 关键
    runtime.refresh_prefix(force=True)

    # 清空运行态
    runtime.current_turn_id = ""
    runtime.current_run_id = ""
    runtime.current_task_state = None
    runtime.current_run_dir = None
```

---

### 恢复后模型怎么知道"我在续接"

两个点把 checkpoint 信息注入下一轮 prompt：

**1. `evaluate_resume_state()` 在 prompt metadata 中标记状态**（`runtime.py:633-642`）

```python
prompt_metadata = {
    "resume_status": "partial-stale",           # 续接状态
    "stale_paths": ["src/main.py"],             # 哪些文件已过期
    "runtime_identity_mismatch_fields": ["model"],  # 哪些环境变了
}
```

**2. `render_checkpoint_text()` 渲染到 memory section**（`runtime.py:362-396`）

```
Task checkpoint:
- Resume status: partial-stale
- Current goal: 找出测试失败的根因
- Current blocker: -
- Next step: Resume from the latest checkpoint and continue the task.
- Key files: src/main.py, tests/test_main.py
- Completed: 读取了测试文件和主逻辑
- Stale paths: src/main.py
```

---

### 两条入口的区别

|           | CLI `--resume` | REPL `/resume`                              |
| --------- | -------------- | ------------------------------------------- |
| 时机        | 进程启动时          | 运行中切换 session                               |
| Worker 处理 | 无旧 worker      | `_shutdown_workers()` 关闭子 agent             |
| 事件总线      | 新开事件流          | `emit_started=False`，不重复发 `session_started` |