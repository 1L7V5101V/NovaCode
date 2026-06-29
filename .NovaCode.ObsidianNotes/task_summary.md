每次 `ask()` 开始时，`set_task_summary(user_message)` 把用户请求写进 `working.task_summary`，但只保留前 300 字符。作用是即使 history 被裁了，模型在 memory section 还能看到一个非常短的航向锚点，知道这一轮在干什么。

---

`task_summary` 就是每次 `ask(user_message)` 开始时，**把用户当前这一轮的请求压成一句短文本存进 memory**。代码在 `runtime.py:777`：

```python
self.memory.set_task_summary(user_message)
```

它最终出现在 memory section 的渲染文本里（`memory.py:565-568`）：

```
Memory:
- task: 给 main.py 加一个二分查找函数   ← 这就是 task_summary
- recent_files: src/main.py, tests/test_main.py
...
```

它的作用是：**即使 history 因为预算控制被大幅裁剪了，模型在 memory section 还能看到一个非常短的航向锚点，知道现在在干什么。** 不是全文历史，就是一句话的事。

不管 history 被压缩得多狠，不管中间跑了多少步工具调用、看了多少文件、出了多少错——**模型在每一轮的 prompt 开头都能看到这句话**，明确知道"我这一轮任务的起点是什么"。

不是全文历史，不是完整需求文档，就是一行字，让模型不会跑偏。