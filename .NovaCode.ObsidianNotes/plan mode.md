以下是 plan mode 各个核心模块的源码实现：

---

### 1. `pico/core/plan_mode.py` — 核心状态管理

```python
class PlanModeManager:
    def __init__(self, runtime):
        self.runtime = runtime

    @property
    def state(self):
        return self.runtime.session.setdefault("runtime_mode", {"mode": "default"})

    @property
    def mode(self):
        return str(self.state.get("mode", "default") or "default")

    @property
    def plan_path(self):
        return str(self.state.get("plan_path", "") or "")
```

**进入 plan mode**：切换 session state、切换 tool profile、刷新系统 prompt、发事件

```python
    def enter(self, topic, path=None):
        plan_path = _plan_path(topic, path)
        self.runtime.session["runtime_mode"] = {
            "mode": "plan",
            "topic": str(topic or ""),
            "plan_path": plan_path,
        }
        self.runtime.set_tool_profile("plan")
        self.runtime.session_path = self.runtime.session_store.save(self.runtime.session)
        self.runtime.refresh_prefix(force=True)
        self.runtime.session_event_bus.emit(
            "runtime_mode_changed",
            {"mode": "plan", "plan_path": plan_path, "topic": str(topic or "")},
        )
        return plan_path
```

**退出 plan mode**：恢复 state 为 `default`，切回默认 tool profile

```python
    def exit(self):
        previous = dict(self.state)
        self.runtime.session["runtime_mode"] = {"mode": "default"}
        self.runtime.set_tool_profile("default")
        self.runtime.session_path = self.runtime.session_store.save(self.runtime.session)
        self.runtime.refresh_prefix(force=True)
        self.runtime.session_event_bus.emit(
            "runtime_mode_changed",
            {"mode": "default", "previous_mode": previous.get("mode", "default"), ...},
        )
```

**final 门控**：检查 plan artifact 文件是否存在且非空

```python
    def can_finish(self):
        if self.mode != "plan":
            return True
        path = self.runtime.path(self.plan_path)
        return path.is_file() and bool(path.read_text(encoding="utf-8").strip())
```

**系统 prompt 文本**：拼接给 LLM 的 plan mode 指令

```python
    def prompt_text(self):
        if self.mode != "plan":
            return ""
        return (
            "Runtime mode: plan\n"
            f"- Active plan artifact: {self.plan_path}\n"
            "- You may inspect files, but writes must target only the active plan artifact.\n"
            "- You may launch Explore subagents, but not write-capable worker subagents.\n"
            "- Use todo tools to keep the task ledger current.\n"
            "- Return a final answer only after the active plan artifact has been written."
        )
```

**路径生成与安全检查**：slug 化 topic → `.pico/plans/<topic>-plan.md`，防止目录穿越

```python
_PLAN_DIR_MARKER = "/.pico/plans/"

def _plan_path(topic, path=None):
    if path:
        value = str(path).strip()
        if value.startswith("/") and _PLAN_DIR_MARKER in value:
            value = value[value.index(_PLAN_DIR_MARKER) + 1:]
        if value.startswith("./"):
            value = value[2:]
    else:
        value = f".pico/plans/{_slug(topic)}-plan.md"
    if (not value.startswith(".pico/plans/") or value.endswith("/") or ".." in value.split("/")):
        raise ValueError("plan path must stay under .pico/plans/")
    return value
```

---

### 2. `pico/core/tool_profiles.py` — 工具白名单

plan mode 的白名单 = 只读工具 + 白名单中的特定工具：

```python
plan_tools = read_only | frozenset({
    "write_file", "patch_file",
    "agent", "send_message", "task_stop",
    "ask_user", "exit_plan_mode",   # 注意：没有 enter_plan_mode，禁止重入
})
```

构建结果：`ToolSetProfile("plan", plan_tools & all_tools)`

---

### 3. `pico/core/permissions.py` — 写权限门控

```python
def _check_plan(self, tool, args):
    if tool.read_only:
        return PermissionDecision.allow("plan_read_only")
    if tool.name not in {"write_file", "patch_file"}:
        return PermissionDecision.deny("plan_mode_tool_not_allowed", "plan_mode_write_guard")
    requested = self.runtime.path(args.get("path", ""))
    active = self.runtime.path(self.runtime.plan_mode.plan_path)
    if Path(requested) != Path(active):
        return PermissionDecision.deny("plan_mode_path_mismatch", "plan_mode_write_guard")
    return PermissionDecision.allow("plan_artifact_write")
```

三层判断：只读工具放行 → 非 write/patch 拒绝 → 路径必须匹配当前 plan artifact。

---

### 4. `pico/core/engine.py` — final 阻塞（第 353–376 行）

```python
final = (payload or raw).strip()
yield from self._drain_worker_notification_events()
if agent.runtime_mode == "plan" and not agent.plan_mode.can_finish():
    notice = agent.plan_mode.final_notice()
    agent.record({"role": "assistant", "content": notice, ...})
    yield {"type": "runtime_notice", "run_id": task_state.run_id, "content": notice}
    continue  # 不退出，继续让 LLM 补充

agent.record({"role": "assistant", "content": final, ...})
if agent.runtime_mode == "plan":
    agent.exit_plan_mode()  # final 成功时自动退出
```

如果模型试图 final 但 plan 文件没写，引擎注入 `runtime_notice` 并让对话继续循环。

---

### 5. `pico/core/runtime.py` — 胶水层

```python
self.plan_mode = PlanModeController(self)          # 初始化 Manager
self._active_tool_profile_name = "plan" if self.runtime_mode == "plan" else ...

@property
def runtime_mode(self):
    return str(self.session.get("runtime_mode", {}).get("mode", "default") or "default")

def set_tool_profile(self, name):
    self._active_tool_profile_name = name

def enter_plan_mode(self, topic, path=None):
    return self.plan_mode.enter(topic, path=path)   # 委托给 Manager

def exit_plan_mode(self):
    return self.plan_mode.exit()
```

---

### 6. `pico/tools/plan.py` — Tool 定义

```python
PLAN_TOOL_SPECS = {
    "enter_plan_mode": {"schema": {"topic": "str", "path": "str?"}, "risky": False, ...},
    "exit_plan_mode":  {"schema": {}, "risky": False, ...},
}

def tool_enter_plan_mode(agent, args):
    path = agent.enter_plan_mode(args["topic"], path=args.get("path"))
    return f"mode: plan\nplan path: {path}"

def tool_exit_plan_mode(agent, args):
    agent.exit_plan_mode()
    return "mode: default"
```

---

### 整体数据流

```
/plan <topic> 或 LLM 调用 enter_plan_mode tool
  → PlanModeManager.enter()
    → 写 session["runtime_mode"] = {mode: "plan", topic, plan_path}
    → runtime.set_tool_profile("plan")       # 白名单切换
    → refresh_prefix(force=True)             # 注入 plan prompt 到系统前缀
    → emit("runtime_mode_changed")           # 通知事件总线

运行期间 permission 检查：
  PermissionChecker.check()
    → profile.allows(name)?                  # 工具在白名单中？
    → _check_plan()
      → 只读工具放行
      → write_file/patch_file 路径必须匹配 plan_path
      → 其他工具拒绝

LLM 试图 final：
  engine 检查 plan_mode.can_finish()
    → plan 文件存在且非空？ → 是：自动 exit_plan_mode，完成
                            → 否：注入 runtime_notice，继续循环
```

"LLM 试图 final" = LLM 输出了 `<final>...</final>` 标签，且该标签不在任何 `<tool>` 调用之后。

> [!NOTE]
> 核心在 `pico/core/model_output.py:7` 的 `parse()` 函数：
> 
> ```Python
> def parse(raw):
>     raw = str(raw)
>     # 1. 优先检测 <tool>，除非 <final> 出现在 <tool> 之前
>     if "<tool" in raw and (
>         "<final>" not in raw or raw.find("<tool") < raw.find("<final>")
>     ):
>         parsed = parse_tool_blocks(raw)
>         ...
>         if parsed:
>             return "tool", parsed   # ← 返回 kind="tool"
> 
>     # 2. 检测 <final> 标签
>     if "<final>" in raw:
>         return "final", extract(raw, "final")  # ← 返回 kind="final"
> 
>     # 3. 兜底：retry
>     return "retry", retry_notice("missing <tool> or <final> tag")
> ```
> 
> 检测逻辑：
> 
> 1. LLM 的输出是纯文本，引擎调用 `agent.parse(raw)` 得到 `(kind, payload)`
> 2. **优先级规则**：`<tool>` 优先于 `<final>`，除非 `<final>` 出现在 `<tool>` 之前
> 3. 如果没有任何标签，返回 `kind="retry"` 让 LLM 重试
> 
> 然后 engine 中根据 `kind` 分流（`engine.py:312-376`）：
> 
> ```Python
> if kind in {"tool", "tools"}:       # → 执行工具调用
>     ...
>     continue
> 
> if kind == "final":                  # → LLM 试图 final
>     if agent.runtime_mode == "plan" and not agent.plan_mode.can_finish():
>         # plan 文件没写，发 runtime_notice 继续循环
>         ...
>         continue
>     # 否则真正结束
>     agent.exit_plan_mode()           # 自动退出 plan mode
> ```