`drain` 发生在 `engine.py:121`，在每轮 while 循环开头、build prompt 之前：

```
while tool_steps < agent.max_steps:
    if agent.abort_requested: ...
    yield from self._drain_worker_notification_events()  # ← 这里
    attempts += 1
    prompt, prompt_metadata = agent._build_prompt_and_metadata(...)
```

`drain_worker_notifications()` 做的事情是：从 `worker_manager._notifications` 这个 `queue.Queue` 里把所有待处理的通知一次性取出来，每条以 `{"role": "user", "content": notification}` 的形式 **写入 session history**。

这样后台 worker 完成的通知在下一轮 prompt 构建时就成了"用户消息"的一部分，模型会看到类似这样的内容：

```
<task-notification>
<task-id>agent_1</task-id>
<status>completed</status>
<summary>Agent 调研认证逻辑 completed</summary>
<result>找到了 auth.py 的入口...</result>
</task-notification>
```

所以"下一轮 drain"的意思是：**父 agent 在进入下一轮模型调用之前，先把所有已完成的后台 worker 结果从队列里取出来塞进对话历史**，让模型能在下一轮 context 里看到这些结果，然后决定下一步做什么。