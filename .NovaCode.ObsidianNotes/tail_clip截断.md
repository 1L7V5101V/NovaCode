`_tail_clip` 截断：

从尾部截断，超限时末尾加 `...`：

```python
def _tail_clip(text, limit):
    if len(text) <= limit:
        return text
    if limit <= 3:
        return text[:limit]
    return text[:limit - 3] + "..."
```

例如 `_tail_clip("deploy key is red and timeout is 120s", limit=20)` → `"deploy key is red ..."`。

用在 `relevant_memory` 和 `history` 两个 section 的每条内容上，保证每段不超过各自的 per_note_budget（如 300 字符），从而把整个 section 塞进预算内。