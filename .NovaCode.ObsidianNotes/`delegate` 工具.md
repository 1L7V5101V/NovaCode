`delegate` 是 pico 内置的一个工具（`tools.py:47-51`），允许主 agent **派一个只读的子 agent 去独立调查任务**，定义：

```python
DELEGATE_TOOL_SPEC = {
    "schema": {"task": "str", "max_steps": "int=3"},
    "risky": False,
    "description": "Ask a bounded read-only child agent to investigate.",
}
```

关键设计：
- **只读**（`read_only=True`）——子 agent 不能写文件，只能调查
- **步数限制**（默认 3 步）——用完自动停止
- **深度限制**——嵌套超过 `max_depth` 时 `delegate` 不再暴露给模型
- **结果返回**——子 agent 的结论以纯文本返回给父 agent，拼入父 history