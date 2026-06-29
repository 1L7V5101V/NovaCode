`relevant_memory` 不是独立存储层，而是**一个运行时构建的 prompt section**，内容来自对**工作记忆 + 持久记忆的实时检索**。

---

### 内容来源

`context_manager.py:120-121` 每轮调用 `retrieval_candidates()` 检索两个来源：

```python
selected_notes = self.agent.memory.retrieval_candidates(
    user_message, limit=RELEVANT_MEMORY_LIMIT  # = 3
)
```

**来源 1：`episodic_notes`**（工作记忆中的过程笔记，`memory.py:1090-1101`）

每条的典型结构：
```python
{
  "text": "patch_file partial_success on src/main.py; inspect diff before retry",
  "tags": ["process", "partial_success", "src/main.py"],
  "source": "patch_file", "kind": "episodic",
  "created_at": "2026-06-10T14:30:22+00:00",
  "note_index": 3
}
```

**来源 2：`DurableMemoryStore`**（持久记忆 topic 文件，`memory.py:1103-1111`）

从磁盘 `.pico/memory/topics/*.md` 中解析出的笔记：
```python
{
  "text": "这个项目用 DeepSeek 的 Anthropic-compatible endpoint",
  "tags": ["convention"], "source": "project-conventions",
  "kind": "durable", "created_at": "..."
}
```

---

### 检索方式

```python
# memory.py:1086-1114
def retrieval_candidates(state, query, limit=3):
    query_tokens = _tokenize(query)     # re.findall(r"[A-Za-z0-9_]+", text)
    ranked = []

    for note in episodic_notes + durable_notes:
        note_tags = {tag.lower() for tag in note["tags"]}
        note_tokens = _tokenize(note["text"]) | _tokenize(note["source"]) | note_tags
        exact_tag_match = int(bool(query_tokens & note_tags))   # 标签精确命中
        keyword_overlap = len(query_tokens & note_tokens)       # 关键词重叠数
        if exact_tag_match == 0 and keyword_overlap == 0:
            continue                                            # 完全不相关 → 跳过
        recency = _parse_timestamp(note["created_at"])
        ranked.append(((exact_tag_match, keyword_overlap, recency), note))

    ranked.sort(key=lambda item: item[0], reverse=True)  # 按(标签命中, 关键词重叠, 时间)降序
    return [note for _, note in ranked[:limit]]          # 取 top 3
```

**无 embedding，纯规则**：tag 精确命中 > 关键词重叠数 > 时间倒序。

---

### 渲染到 prompt

`_render_relevant_memory()`（`context_manager.py:244-289`）把 3 条笔记放入 `relevant_memory` section：

```
Relevant memory:
- 这个项目用 DeepSeek 的 Anthropic-compatible endpoint
- patch_file partial_success on src/main.py; inspect diff before retry
```

**预算控制**：每条笔记均分该 section 的 budget（默认 6000 chars），各自从末尾 `tail_clip`。如果某条笔记占不满自己的份额，多余空间不转移给其他笔记。

---

### 完整链路

```
用户请求 "找出测试失败的根因"
  │
  ▼
LayeredMemory.retrieval_candidates(query)
  ├─ 扫描 session["memory"]["episodic_notes"] (内存, ≤12条)
  ├─ 扫描 DurableMemoryStore.topics/*.md (磁盘)
  └─ token 重叠排序 → top 3
  │
  ▼
_render_relevant_memory(selected_notes, budget=6000)
  │ budget // 3 → 每条笔记最多 2000 chars → tail_clip
  ▼
prompt 中的 section:
  Relevant memory:
  - pytest failed with exit_code 1 in src/test_main.py
  - run_shell -> partial_success on tests/
  - project convention: 用 pytest 而非 unittest
```