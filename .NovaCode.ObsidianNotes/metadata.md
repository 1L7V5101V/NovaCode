在代码里，"metadata" 这个词有具体的技术含义，不是泛泛的"描述数据的数据"。它指这两样东西：

**第一**，`prompt_metadata`（`context_manager.py:456-508` 返回的 dict）——它是 `ContextManager.build()` 每次组装完 prompt 后一并返回的**统计报告**，包含：

```python
{
    "prompt_chars": 8234,           # 最终 prompt 长度
    "prompt_budget_chars": 12000,   # 预算上限
    "prompt_over_budget": False,    # 有没有超预算
    "sections": {                   # 每个 section 的原始/裁剪后长度
        "prefix": {"raw_chars": 3600, "rendered_chars": 3600},
        "history": {"raw_chars": 5210, "rendered_chars": 4800},
        ...
    },
    "budget_reductions": [...],     # 如果发生了裁剪，记录每个 section 减了多少
    "relevant_memory": {...},       # 召回了哪些笔记、长度
    "history": {...},               # 历史中有多少条被折叠、多少条用摘要复用
}
```

它不是 prompt 的一部分，是 prompt **生产过程**的记录。

**第二**，`completion_metadata`（`models.py:344-349`）——它是模型 API 返回的消耗统计：

```python
{
    "input_tokens": 8000,
    "output_tokens": 150,
    "total_tokens": 8150,
    "cached_tokens": 3600,    # 命中缓存了多少 token
    "cache_hit": True,
}
```

它从 `model_client.complete()` 的 HTTP 响应体里的 `usage` 字段提取而来。

这两份东西在 `Pico.ask()` 的每轮末尾被合并，写进 trace 和 report：

```python
# runtime.py:879-885
completion_metadata = dict(getattr(self.model_client, "last_completion_metadata", {}) or {})
if completion_metadata:
    prompt_metadata.update(completion_metadata)
```

**所以"metadata"在代码里的精确含义是：每一轮 prompt 组装过程和生产消耗的统计记录，用于 trace 和 report，用于回答"这轮 prompt 怎么来的、花了多少 token"。** 它不是给模型看的，是给人复盘用的。