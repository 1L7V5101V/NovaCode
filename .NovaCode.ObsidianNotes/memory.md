pico 的记忆分两层：**working memory（工作记忆）** 和 **durable memory（持久记忆）**。

---

### 工作记忆（`session["memory"]` 内存中）

每次工具执行后自动更新：

```python
# default_memory_state() 结构
{
  "working": {
    "task_summary": "找出测试失败的根因",   # 当前任务
    "recent_files": ["src/main.py", ...]   # 最近8个文件
  },
  "episodic_notes": [             # 最多12条, 带标签
    {"text": "patch_file partial_success on src/main.py",
     "tags": ["process", "partial_success", "src/main.py"],
     "source": "patch_file", "kind": "process"}
  ],
  "file_summaries": {             # 带freshness哈希
    "src/main.py": {
      "summary": "def parse(): ... | def run(): ...",
      "freshness": "sha256..."
    }
  },
  "durable_topics": ["project-conventions", "key-decisions"]
}
```

**写入时机**（`runtime.py:717-757` `update_memory_after_tool()`）：

| 工具 | 行为 |
|---|---|
| `read_file` | 存文件短摘要到 `file_summaries`，路径记录到 `recent_files` |
| `write_file` / `patch_file` | 记录 `self_authored_freshness`，失效旧 `file_summary` |
| 工具失败/部分成功 | 追加 `kind="process"` 的笔记，带原因标签 |

---

### 持久记忆（`.pico/memory/` 磁盘上）

目录结构：

```
.pico/memory/
├── MEMORY.md                # 索引（<=200行，<=25KB）
├── logs/
│   └── 2026/06/2026-06-10.md  # 每日追加的 intake stream
└── topics/
    ├── project-conventions.md
    ├── key-decisions.md
    └── user-preferences.md
```

**写入方式有 3 种**：

1. **模型 final 中的 `<memory>` 标签** — turn 结束后自动提取，追加到 daily log
2. **模型直接写 topic 文件** — [[dream]] 阶段用 write_file 写带 frontmatter 的 `.md` 文件
3. **用户 `/remember <text>`** — 追加一条 daily log

---

### 跨层流动：两阶段固化

```
每轮 turn 结束时:

  extract_durable_promotions()                  # 从 final_answer 中匹配:
    用户含 "记住/remember/capture" 意图          #   "Project convention: ..."
    + 匹配 DURABLE_MEMORY_LINE_PATTERNS          #   "Decision: ..."
    → promote_durable()                          #   → 写入 topics/*.md + 更新 MEMORY.md

  maintain_memory_after_turn()                   # 后台:
    ├─ 提取 <memory> 标签 → 追加 daily log
    └─ 判断 auto-dream 条件:
         ├─ 距上次 consolidation > 24h
         ├─ 新增 session >= 5
         └─ 有跨进程锁 (lock 文件)
         → 启动独立 Pico agent:
             读取 daily logs → 写入/更新 topic 文件 → 更新 MEMORY.md
```

---

### 检索方式

```python
# retrieval_candidates(query, limit=3) — 无 embedding
ranking = (
    exact_tag_match,   # 标签精确命中 (最高权重)
    keyword_overlap,   # 关键词 token 重叠数
    recency            # 时间戳倒序
)
```

同时搜索 `episodic_notes`（session 内存）和 `DurableMemoryStore`（磁盘 topic 文件），统一排序取 top 3 放入 prompt 的 `relevant_memory` section。

---

**总结**：工作记忆是工具执行时自动沉淀的轻量上下文（文件路径、短摘要、过程笔记），持久记忆是跨 session 的结构化知识（带 frontmatter 的 topic 文件 + 索引），dream 后台进程负责把 daily log 消化为 topic 文件。