[[裁剪]]
[[prompt超预算]]

### 每轮 prompt 组装时（`context_manager.py`）

`ContextManager.build()` 把 6 个 section 拼成 prompt，每个有独立预算：

| Section                   | 默认预算  | 最低底线 | 裁剪优先级      |
| ------------------------- | ----- | ---- | ---------- |
| [[prefix]]（系统规则）          | 12000 | 4000 | 5（最后动）     |
| [[memory]]（工作记忆）          | 8000  | 1200 | 4          |
| [[skills]]                | 4000  | 600  | 2          |
| [[relevant_memory]]（相关笔记） | 6000  | 1000 | **1（最先动）** |
| [[history]]（对话历史）         | 30000 | 6000 | 3          |
| current_request（当前请求）     | —     | 不裁剪  | 永不裁剪       |

**判断逻辑**（第 146-171 行）：组装完成后检查 `len(prompt) > total_budget`（默认 60K chars），如果超了按 `reduction_order = ("relevant_memory", "skills", "history", "memory", "prefix")` 逐个降预算、重新渲染，直到不超或触及底线。



### 降预算

就是减小传给渲染函数的 `budget` 参数，让该 section 的渲染结果变短：

```python
# 第165-166行
budgets[section] = new_budget                                 # 例如: 6000 → 4000
rendered = self._render_sections(section_texts, budgets, ...) # 用新预算重新渲染
```

各 section 对"预算变小"的响应：

| Section | 预算减小时的行为 |
|---|---|
| `prefix` | `tail_clip(raw, 4000)` — 末尾更多内容被截断 |
| `relevant_memory` | `per_note_budget` 变小 → 每条笔记更短，或笔记条数更少 |
| `history` | `TurnHistoryBuilder.render_section(20000)` → 更多旧轮被折叠，每行 clip 更短 |

所以"降预算"本质上就是 **"告诉渲染函数：这次输出得更短，因为总空间不够了"**。

---

### floor

每个 section 允许降到的最低预算值，降到这个值后不能再降：

```python
# 默认计算: budget // 4
def _compute_section_floors(self):
    floors = {
        section: max(20, int(budget) // 4)   # 至少 20 chars
        for section, budget in self.section_budgets.items()
    }
```

| Section | 初始预算 | 默认 floor (budget//4) | 覆盖值 |
|---|---|---|---|
| `prefix` | 12000 | 3000 | **4000**（DEFAULT_SECTION_FLOORS 覆盖） |
| `memory` | 8000 | 2000 | **1200** |
| `skills` | 4000 | 1000 | **600** |
| `relevant_memory` | 6000 | 1500 | **1000** |
| `history` | 30000 | 7500 | **6000** |

在 reduction loop 中作为跳过条件：

```python
# 第150-153行
floor = int(self.section_floors.get(section, 0))
current_budget = int(budgets.get(section, 0))
if current_budget <= floor:
    continue           # 这个 section 不能再降了，看下一个
```

当所有 section 都触及 floor 但 prompt 仍然超预算时，循环直接 break，prompt 以超预算状态发出，不再继续[[裁剪]]（`context_manager.py:170-171`）。