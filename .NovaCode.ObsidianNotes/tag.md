不是大模型打的，是**代码逻辑硬编码**的。看 `runtime.py` 里的两个调用点：

**1. 读文件后（第 672 行）：**
```python
self.memory.append_note(summary, tags=(canonical_path,), source=canonical_path)
```
tag 就是被读文件的**路径**。比如读了 `config.txt`，tag 就是 `("config.txt",)`。

**2. 工具执行异常后（第 691 行）：**
```python
tags = ["process", status, *affected_paths]
```
比如 `patch_file` 执行出现 `partial_success`，tag 就是 `("process", "partial_success", "config.txt")`。

所以 tag 的来源很确定：

| 场景 | tag 内容 | 谁决定的 |
|------|---------|---------|
| 读文件 | 文件名路径 | 代码 |
| 部分成功/错误/被拒 | `"process"` + 状态 + 文件名 | 代码 |

这些 tag 的用途也不是给 LLM 看的，而是给后续的 `retrieval_candidates()` 做 **tag 精确匹配**用的——query 的词如果正好命中某个 tag，这条笔记排序优先级更高。

---
**所有笔记类型都有 `tags` 属性**，且都经过 `append_note()` 进入 `episodic_notes` 列表或持久化笔记系统：

| 来源        | 代码位置                 | tag 例子                                         | kind            |
| --------- | -------------------- | ---------------------------------------------- | --------------- |
| 读文件后记摘要   | `runtime.py:672`     | `("config.txt",)`                              | 默认 `"episodic"` |
| 工具异常/部分成功 | `runtime.py:691-692` | `("process", "partial_success", "config.txt")` | `"process"`     |
| stress 实验 | `metrics.py:204`     | `("recall",)`                                  | 默认 `"episodic"` |
| 上下文实验     | `metrics.py:462`     | (自定义)                                          | 默认              |
| 实时大模型实验   | `metrics.py:938`     | `("token",)`                                   | 默认              |
| 测试代码      | `test_memory.py:23`  | `("recall",)`                                  | 默认              |
| 持久化笔记     | `memory.py:92-94`    | 从 markdown 解析，如 `"convention"`                 | `"durable"`     |

**tag 种类汇总：**

| 分类 | 实际 tag 值 |
|------|------------|
| 文件路径 | 读过的文件名，如 `config.txt`, `facts.txt` |
| 流程状态 | `process`, `partial_success`, `error`, `rejected` |
| 实验标识 | `recall`, `token`, `unrelated` |
| 持久化笔记 | 自定义标签，如 `convention` |

注意 `kind="process"` 的笔记也进了 `episodic_notes` 列表，只是带了一个 `kind` 属性区分身份——它们是同一条存储管道。