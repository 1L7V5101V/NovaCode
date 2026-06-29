看代码里实际防污染的机制，比抽象解释更清楚：

### 1. 文件已修改，但记忆里的摘要还是旧的

```python
def file_freshness(raw_path, workspace_root=None):
    resolved = resolve_workspace_path(raw_path, workspace_root)
    return hashlib.sha256(resolved.read_bytes()).hexdigest()
```

存摘要时同时存了文件内容的 SHA256 哈希（`memory.py:479`）。渲染时发现哈希对不上 → 摘要自动失效，不再注入 prompt。这就是"外部修改导致记忆过期"的污染。

### 2. 笔记去重

```python
notes = [item for item in state["episodic_notes"] if item["text"] != note["text"]]
notes.append(note)
```

`append_note()` 里如果新笔记的 text 和已有某条完全一样，旧的那条被移除（`memory.py:465`）。避免同一个事实重复堆积。

### 3. 显式失效

```python
if name in {"read_file", "write_file", "patch_file"}:
    self.memory.remember_file(canonical_path)
if name == "read_file":
    ...
elif name in {"write_file", "patch_file"}:
    self.memory.invalidate_file_summary(canonical_path)  # 文件被改了 → 删摘要
```

`runtime.py:667-674`：写了文件就把对应的摘要删掉，下次需要时重新读、重新记。

### 4. `irrelevant` 对照实验存在的意义

```python
_set_irrelevant_memory(agent)  # 把记忆全换成无关节目的
```

实验结果 `memory_irrelevant` 表现和 `memory_off` 一样差，说明**塞了错的记忆比不塞更糟**——它占用预算、干扰 recall，但没贡献价值。

### 总结：为什么会被污染

| 污染来源 | 后果 | 防御手段 |
|---------|------|---------|
| 文件被外部修改 | 记忆的摘要和实际内容不一致 | freshness 哈希校验 |
| 重复追加同一事实 | 堆积多条，挤占预算 | 去重 |
| 修改了文件但未显式清除摘要 | agent 依据旧摘要做决策 | 写文件后自动 invalidate |
| 记下了无关信息 | 占用 relevant_memory 预算，干扰真正需要的 recall | `irrelevant` 实验已验证其危害 |

**"污染"本质就是记忆内容和执行环境的真实状态产生了偏差**——agent 以为某件事是真的，但它已经变了。

---

在 AI Coding Agent 的语境下，**Memory 污染（Memory Pollution/Contamination）** 是指 Agent 在长期对话或多步骤任务中，由于接收了过多**无关、矛盾、陈旧或低质量**的信息，导致其检索效率下降、逻辑混乱甚至产生幻觉的现象。
## 1. 污染的主要来源

Memory 污染通常由以下几种数据“杂质”引起：

### ❌ 错误/过时的尝试

在调试（Debug）过程中，Agent 可能会尝试多种方案。如果它将这些失败的尝试、错误的中间代码片段全部保存在“长期记忆”中，下一次生成代码时，它可能会混淆正确方案与错误方案。

### ❌ 冗余的上下文

Coding Agent 往往会通过检索（RAG）读取大量文档或库文件。如果检索算法精度不高，抓取了大量不相关的函数定义或过时的 API 文档，这些“噪声”会挤占模型有限的上下文窗口（Context Window），稀释关键指令。

### ❌ [[幻觉循环]]（Hallucination Loop）

如果 Agent 在某一步生成了错误的代码，而用户没有及时纠正，Agent 可能会在接下来的推理中基于这个“错误事实”继续推演。这种自我增强的错误信息会像毒素一样污染后续的所有决策。

### ❌ 格式/系统杂讯

大量的 Git Diff 记录、编译器报错堆栈、格式化的 JSON 响应等非逻辑文本，如果未经处理直接存入 Memory，会导致 Agent 在理解业务逻辑时被这些结构化噪声干扰。

## 2. 污染带来的后果

当 Memory 被污染后，Coding Agent 通常会出现以下表现：

- **指令飘移（Instruction Drift）：** 开始无视最初的架构设计要求，转而执行记忆中最近出现但错误的逻辑。
    
- **重复造轮子：** 忘记了之前已经定义过的函数，反复生成功能相同但名称不同的冗余代码。
    
- **性能衰减：** 随着对话轮次增加，生成速度变慢，且回复的相关性明显变差。
    
- **逻辑自相矛盾：** 前后代码块使用的变量命名规范或技术栈（如一会儿用原生 CSS，一会儿用 Tailwind）不统一。
    

## 3. 如何治理 Memory 污染？

为了保持 Agent 的“头脑清醒”，开发者通常采用以下技术手段：

### 🧼 记忆剪枝（Pruning & Summarization）

- **摘要化：** 定期将长对话压缩成简短的“事实摘要”，只保留核心结论。
    
- **遗忘机制：** 给信息设置权重或有效期，自动剔除那些被证明无效的调试代码。
    

### 🧼 结构化记忆（Structured Storage）

不直接存储原始聊天记录，而是将记忆分类存储。例如：

- **Long-term:** 存储项目全局架构、技术栈偏好。
    
- **Short-term:** 存储当前正在处理的 Bug 堆栈。
    

### 🧼 语义过滤（Semantic Filtering）

在 Agent 将信息写入数据库前，先通过一个小型模型进行“质量检测”，剔除无意义的报错信息或重复的确认语。

> **核心提示：**
> 
> 对于开发者来说，预防污染最简单的方法是**及时反馈**。当你发现 Agent 产生错误逻辑时，直接告诉它“这条方案是错的，请从记忆中删除”，或者在开启新功能模块时，重置/清理不相关的上下文。