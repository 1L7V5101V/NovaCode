就一行判断：

```python
# context_manager.py:146
while len(prompt) > self.total_budget:
```

---

### 怎么算 `prompt`

```python
# 第326-328行
def _assemble_prompt(self, rendered):
    return "\n\n".join(
        rendered[section].rendered for section in SECTION_ORDER
    ).strip()
```

把 6 个 section 渲染结果按顺序用 `\n\n` 拼接成一个字符串，然后取 `len()` 算字符数。

```
SECTION_ORDER = ("prefix", "memory", "skills", "relevant_memory", "history", "current_request")

最终 prompt 结构:
  <prefix渲染结果>
  （空行）
  <memory渲染结果>
  （空行）
  <skills渲染结果>
  （空行）
  <relevant_memory渲染结果>
  （空行）
  <history渲染结果>
  （空行）
  Current user request:
  <用户请求原文>
```

---

### 预算值

```python
# 第15行
DEFAULT_TOTAL_BUDGET = 60000        # 单位: chars
```

**为什么用 chars 而不是 tokens？** 因为 `len()` 是 O(1) 操作，估算 tokens 需要遍历或做除法。在每次 tool 执行后都要重建 prompt 的场景下，`len(prompt)` 的代价可以忽略。metadata 中的 token 估算是 final 才做，不影响裁剪路径。

在 `context_usage.py:8-9` 可以看到 token 估算只在最终上报时用：

```python
def estimate_tokens(chars):
    return max(0, (int(chars) + 3) // 4)    # chars ÷ 4 粗略估算
```

---

触发入口是每次模型调用前：

```python
# runtime.py:601-611
def _build_prompt_and_metadata(self, user_message):
    prompt, metadata = self.context_manager.build(user_message)
    # └─ 内部做了: render → 组装 → while len > 60000 循环降预算
    if metadata.get("prompt_over_budget") and len(self.session.get("history", [])) > 4:
        self.compact_history(trigger="auto_prompt_over_budget")
        prompt, metadata = self.context_manager.build(user_message)
```