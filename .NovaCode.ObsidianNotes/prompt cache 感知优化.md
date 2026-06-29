这个说法需要纠正——**pico 其实有基本的 prompt cache 支持**。让我更正：

---

## pico 已有的 prompt cache

`runtime.py:544` 把 `prefix_state.hash` 作为 `prompt_cache_key`，`models.py:276-279` 在请求中发送 `prompt_cache_key` 和 `prompt_cache_retention` 给后端：

```python
# runtime.py:866-871
if getattr(self.model_client, "supports_prompt_cache", False):
    prompt_cache_key = prompt_metadata.get("prompt_cache_key")  # = prefix_state.hash
    prompt_cache_retention = "in_memory"
```

**策略**：cache key 只覆盖**稳定前缀**（system prompt + 工具定义 + workspace 信息），不包含历史和用户请求。前缀不变时，后端可以命中缓存。这是合理的"前缀缓存"策略。

---

## 缺少的是 Claude Code 的"缓存感知"优化

pico 有缓存，但**没有围绕缓存做任何特殊设计**。Claude Code 多了三个关键优化：

|                | pico                                      | Claude Code                                                                 |
| -------------- | ----------------------------------------- | --------------------------------------------------------------------------- |
| **缓存感知的压缩**    | 压缩后整个 prompt 重建 → cache key 内容变了 → 缓存失效   | **`cache_edits`** 手术式删除旧工具结果块，不动缓存前缀                                        |
| **压缩复用缓存 key** | 无。`context_manager.build()` 每次构建全新 prompt | 压缩摘要请求**复用主对话的 system prompt + tools + prefix**，追加为一条 user message，cache 命中 |
| **分层缓存策略**     | 无                                         | microcompact 先手术式清理（不伤缓存），cache_edits 不够才走 LLM 摘要（复用 cache key）             |

核心差距一句话：**pico 只是"发了 cache key"，但压缩/截断操作完全无视缓存状态，每次 prompt 变化都导致缓存重写。**