"按当前请求召回 top 3" 意思是：用用户当前这次输入的消息作为 **query**，从 `episodic_notes` 中找出最相关的 3 条笔记，拼进 prompt。

代码实现链路：

1. **入口** — `context_manager.py:119-121`：把用户消息传给 memory 系统
```python
selected_notes = self.agent.memory.retrieval_candidates(
    user_message, limit=RELEVANT_MEMORY_LIMIT  # 3
)
```

2. **排序** — `memory.py:519-547` 的 `retrieval_candidates()`：遍历所有笔记，对每条计算一个四元组 `(exact_tag_match, keyword_overlap, recency, note_index)`，按元组降序排，取前 `limit` 条

3. **组装** — `context_manager.py:243-288`：把返回的笔记列表渲染成：
```
Relevant memory:
- <笔记1>
- <笔记2>
- <笔记3>
```
插入到 prompt 的第三段（prefix → memory → **relevant_memory** → history → current_request）。

简单说：拿用户当前的说话内容当搜索词，从历史笔记里用 tag/关键词/时间做排序，挑出得分最高的 3 条塞进上下文。