整个流程精确如下，按代码执行顺序拆解：

---

### 步骤 1：按 turn_id 分组（第 44-53 行）

```
输入: session["history"] = [item1, item2, item3, ...]  # 扁平的 item 列表

_group() 遍历每个 item，按 turn_id 聚组。
turn_id 由 TurnHistoryBuilder.enrich() 在 record() 时自动赋予
（engine.py:89 → record() → enrich(): 若无 turn_id 则取 current_turn_id）。
```

示例：
```python
groups = [
    ("task_001", [user_msg, assistant_msg, tool_call_1, tool_call_2]),  # 第1轮
    ("task_002", [user_msg, assistant_msg, tool_call]),                  # 第2轮
    ("task_003", [user_msg, ...]),                                       # 第3轮(最近)
]
```

---

### 步骤 2：切割新旧轮次（第 19-22 行）

```python
compacted_turns = groups[:-2]      # 前 N-2 组 → 被压缩
kept_turns     = groups[-2:]       # 最后 2 组 → 完整保留

compacted_items = [item for _, items in compacted_turns for item in items]
kept_items      = [item for _, items in kept_turns      for item in items]
```

**判断标准**：`keep_recent_turns=2`，即最近 2 个完整的轮次（每个轮次包含 user→assistant→tool... 的完整周期）不被压缩。

---

### 步骤 3：提取摘要文本（第 68-97 行）

`_summary_text(compacted_items)` 遍历被压缩的全部 item，提取 4 类信息：

| 抽取对象              | 条件                                                      | 保留方式                                     |
| ----------------- | ------------------------------------------------------- | ---------------------------------------- |
| `user_requests`   | `role == "user"` 的 content                              | 只保留**最后一条**（`user_requests[-1]`→ `Goal`） |
| `assistant_notes` | `role == "assistant"` 的 content                         | 只保留**最后一条**（→ `Key decisions`）           |
| `files_read`      | `role == "tool" && name == "read_file"` 的 path          | 全部去重排序（→ `Files read`）                   |
| `files_modified`  | `role == "tool" && name` 为 write_file/patch_file 的 path | 全部去重排序（→ `Files modified`）               |

**丢弃的信息**：中间轮次里所有 `role == "tool"` 的完整输出内容（shell stdout/stderr、文件内容）、模型每步的推理文本、run_shell 命令、搜索工具的结果、中间的错误消息等全部丢失。

最终生成：
```
Compacted session summary:
- Goal: <最后一条用户消息>
- Constraints and preferences: -  (固定占位符)
- Files read: tests/test_a.py, src/main.py  (去重排序)
- Files modified: src/main.py                (去重排序)
- Key decisions: <最后一条 assistant 回复>
- Current progress: compacted 37 history items
- Open blockers: -                           (固定占位符)
- Next step: continue from the latest preserved turn
- Critical context: earlier turns were compacted; ...
```

---

### 步骤 4：替换 history（第 33 行）

```python
summary_item = {
    "role": "system",
    "kind": "compact_summary",
    "content": <上面生成的摘要文本>,
    "created_at": now(),
    "source": "compact",
    "turn_id": ..., "run_id": ..., "event_id": ...  # enrich() 补充
}

self.agent.session["history"] = [summary_item, *kept_items]
```

替换后 `session["history"]` 的结构：
```
[
    {"role": "system", "kind": "compact_summary", "content": "Compacted..."},  # 1条替代了 N-2 轮
    {"role": "user", "turn_id": "task_002", ...},     # 第2轮完整保留
    {"role": "assistant", "turn_id": "task_002", ...},
    {"role": "tool", "turn_id": "task_002", ...},
    {"role": "user", "turn_id": "task_003", ...},     # 最新轮完整保留
    ...
]
```

---

### 步骤 5：记录与持久化（第 34-40 行）

- 把 compaction 元数据（trigger、压缩前后的 token 数、items 数）追加到 `session["compactions"]` 列表
- 保存 session 到磁盘
- 发射事件和 trace

---

### 触发时机（`runtime.py:608-610`）

```python
# auto compaction: prompt 超预算且 history 超过 4 条
if metadata.get("prompt_over_budget") and len(self.session.get("history", [])) > 4:
    self.compact_history(trigger="auto_prompt_over_budget")
```

手动触发由 CLI REPL 或 TUI 中的 `/compact` 命令调用 `compact_history(trigger="manual")`。

---

**总结**：History Compaction 不是 clip/truncate，而是**以轮次为单位做结构化摘要替换**——丢弃旧轮次全量细节，保留一份轻量导航摘要 + 最近 2 轮完整上下文。