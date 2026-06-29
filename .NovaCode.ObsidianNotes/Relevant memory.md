`Relevant memory` 指的是：**从长期/跨轮记忆里，按当前用户请求临时召回出来的“相关笔记”**。

它不是全部 memory，也不是完整历史，而是 memory 里最可能对当前这一轮有帮助的少量内容。

在 Pico 里可以这样理解：

- `memory`：稳定的小仪表盘，比如当前任务摘要、最近读过的文件、文件摘要、笔记数量。
- `relevant_memory`：根据当前 `user_message`，从 `episodic_notes` 里挑出来的 top 3 条相关笔记。

比如之前 agent 读过一个文件，并记下：

> `config.py` 里模型默认 temperature 是 0.2。

后来用户问：

> “模型参数默认值在哪里改？”

这条笔记就可能被召回到 `relevant_memory` 里，因为它和当前问题相关。

所以这段话的意思是：

> 到 `ContextManager.build(user_message)` 组装 prompt 时，系统会根据当前用户输入，从已有的 episodic notes 里挑出最多 3 条最相关的记忆，放进 prompt 的 `relevant_memory` 区块。但不会把所有记忆都塞进去，而且这 3 条也会继续按预算压短。

面试里可以这样说：

> Relevant memory 是按当前请求动态召回的相关记忆。Pico 的 memory 里会有一些跨轮笔记，但每一轮不可能全部带进 prompt，所以在构造上下文时，会根据当前 user message 从 episodic notes 里选 top 3。它解决的是“哪些旧结论现在还值得带回来”的问题，而不是把所有历史都重新塞给模型。