few-shot 上下文在 agent 系统里的意思是：**模型看到的历史记录会自动充当它的示例。**

不是你刻意给它的 few-shot demonstration，而是 session history 里之前几轮的工具调用格式，模型在生成下一轮调用时**自动跟着学**。

代码里体现得很直接。`ContextManager` 组装 prompt 时，history section 的格式是（`context_manager.py:437-442`）：

```python
def _render_history_item(self, item, line_limit):
    if item["role"] == "tool":
        prefix = f"[tool:{item['name']}] {json.dumps(item['args'], sort_keys=True)}"
        content = _tail_clip(item["content"], max(20, line_limit))
        return [prefix, content]
```

模型看到的实际文本是：

```
[tool:read_file] {"path": "sample.txt", "start": 1, "end": 4}
# sample.txt
   1: alpha
   2: beta
[tool:patch_file] {"path": "sample.txt", "old_text": "beta", "new_text": "beta-locked"}
patched sample.txt
```

这就是隐式的 few-shot prompt。模型在生成下一个 `<tool>...</tool>` 时，会参照历史里的格式——因为历史里是 `{"path": ..., "args": ...}` 这种 JSON 风格，它就更倾向于继续输出 JSON 风格，而不是突然换成 `<tool name="..." path="...">` 的 XML 风格。

切模型的问题就在这里：**旧模型的 few-shot 例子还在历史里，但新模型可能训练数据里见到的格式偏好不同。** 新模型看到 JSON 格式的历史，可能跟着输出 JSON，但如果它本身在 XML 上更稳定，就会产生"格式混搭"的乱象。即使 parser 两种都支持，混搭格式也可能导致 parse 失败或参数提取错误。

所以"few-shot 上下文受影响"，不是历史内容错了，而是**历史中隐式夹带的"模型行为风格"不适用于新模型了**，但模型自己不知道这件事。