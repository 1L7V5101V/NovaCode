从原始消息到最终 prompt 中的 history 文本，经过 **三条流水线**：记录 → 丰富 → 压缩渲染。

## 第一段：记录（谁往 history 里写）

每次 `agent.record()` 追加一条，共 5 种来源：

```
来源                    role        触发位置
────────────────────────────────────────────────
用户输入                user        engine.run_turn() 第 90 行
工具执行结果             tool        execute_tool_payload() 第 36 行
模型最终回答             assistant   engine.run_turn() 第 374 行
模型重试通知             assistant   engine.run_turn() 第 337 行
子代理通知(转交用户)     user        engine.drain_worker_notifications() 第 42 行
```

[[record()]] 内部：

```python
# runtime.py:593
def record(self, item):
    self.session["history"].append(
        self.turn_history.enrich(item)  # 加 turn_id / run_id / event_id
    )
    self.session_path = self.session_store.save(self.session)  # 立即落盘
```

每条原始消息被 enrich 成这样的结构：

```json
{
  "role": "tool",
  "name": "read_file",
  "args": {"path": "main.go", "start": 1, "end": 80},
  "content": "package main\n...",
  "created_at": "2026-06-08T10:00:00",
  "turn_id": "task_20260608-100000-abc123",
  "run_id": "run_20260608-100000-def456",
  "event_id": "event_000001",
  "source": "runtime"
}
```

## 第二段：组装（history 怎么进 prompt）

`context_manager.build()` 第 237 行调用 `_render_history_section(budget)`：

```python
# context_manager.py:298
def _render_history_section(self, budget):
    history = self.session["history"]  # ← 原始 list
    raw = self.history_builder.raw_text(history)
    rendered, details = self.history_builder.render_section(budget)
    return SectionRender(raw=raw, budget=budget, rendered=rendered, details=details)
```

`render_section()` 做三件事：

**① 按 turn_id 分组**（turn_history.py:58）
```
Turn task_001: [user] ... [tool:read_file] ... [assistant] ...
Turn task_002: [user] ... [tool:write_file] ... [assistant] ...
Turn compact_summary: 合并后的摘要
```

**② 区分最近 3 轮和旧轮**（turn_history.py:59-60）
```python
recent_window = 3
recent_turns = set(list(turns)[-recent_window:])
```

**③ 压缩旧轮**（_compressed_turn_entries, 第 87 行）：

| 规则 | 旧轮 | 新轮 |
|------|------|------|
| `compact_summary` 全文保留 | ✅ | ✅ |
| 重复 `read_file` | 合并为文件摘要 | 保留 |
| 工具调用行 | 单行缩写（80 chars） | 全文（900 chars） |
| 每行上限 | 80 chars | 900 chars |

```python
# 第 104-116 行的关键逻辑：
if not recent and item["role"] == "tool" and item["name"] == "read_file":
    # 旧轮重复读 → 用文件摘要替代
    summary = self._reusable_file_summary(path)
    lines.append(f"{path} -> {summary}")  # 几行搞定
elif not recent and item["role"] == "tool":
    # 旧轮工具调用 → 缩成一行
    lines.append(f"{command} -> 前3行输出")
```

## 第三段：预算裁剪（超了怎么办）

`render_section()` 从最新轮开始逆向装配（第 62-72 行）：

```python
for entry in reversed(entries):            # 从最新轮开始
    candidate = entry["lines"] + rendered_entries
    if len(joined) <= budget:              # 够放 → 继续加
        rendered_entries = candidate
        continue
    if entry["turn_id"] in recent_turns:   # 新轮超预算 → tail_clip
        clipped = [tail_clip(line, max(40, budget//len(lines))) for ...]
```

核心策略是：**从最新往最旧拼，直到预算用完**。

## 最终输出到 prompt

`_assemble_prompt()` 把 6 段拼在一起：

```
{prefix}\n\n{memory}\n\n{skills}\n\n{relevant_memory}\n\n{history}\n\n{current_request}
```

其中 history 段最终是一个纯文本：

```
Transcript:
Turn task_003:
[user] 帮我把 auth 重构一下
[tool:read_file] {"path":"src/auth.py","start":1,"end":200}
def login...
[tool:write_file] {"path":"src/auth.py"}
written 180 chars
[assistant] 改好了，加了 JWT 校验
```

所以回答你的问题：**history 来源于 `session["history"]` 数组，经过 turn_id 分组、旧轮压缩、按预算从新到旧逆向裁剪后，拼成一段纯文本注入 prompt**。每次 `record()` 都会即时落盘到 `.pico/sessions/<id>.json`，所以 `--resume` 时可以完整恢复。