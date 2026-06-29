每个失败模式对应的防线：

---

### ① 模型不按协议输出

| 防线 | 位置 | 机制 |
|------|------|------|
| **parse() 三路判定** | `model_output.py:7-23` | 检测 `<tool>` / `<final>` / 都不匹配 → `("retry", notice)` |
| **空响应检测** | `model_output.py:21-22` | `raw.strip()` 为空 → retry |
| **JSON 校验 + XML fallback** | `model_output.py:34-122` | tool body 必须合法 JSON 或 XML，否则报错 |
| **retry 循环** | `engine.py:335-349` | parse 返回 retry → 记录 notice → continue，不计 `tool_steps` |
| **attempts 上限** | `engine.py:108,434-436` | `max_attempts = max_steps + 2`，超限 → `stop_retry_limit()` |
| **prefix 格式约束** | `runtime.py:467-473` | 预设 work 手册告诉模型只能返回 `<tool>` 或 `<final>` |

---

### ② 工具循环不收敛

| 防线 | 位置 | 机制 |
|------|------|------|
| **硬 step 上限** | `engine.py:110` | `tool_steps < max_steps`（默认 50） |
| **重复工具调用检测** | `tool_repetition.py:6-25` | 同一轮相同 `(name, args)` ≥ 2 次 → 拦截报错 |
| **step 上限摘要** | `engine_helpers.py:212-251` | 达上限后请模型写进展摘要，不直接断掉 |
| **abort 请求** | `engine.py:111,270,321` | 3 处检查 `agent.abort_requested`，用户可 `/stop` 中断 |

---

### ③ Provider 空响应

| 防线             | 位置                          | 机制                                                  |
| -------------- | --------------------------- | --------------------------------------------------- |
| **空响应重试（1 次）** | `engine_helpers.py:186-192` | 仅 `ProviderError(code="empty_response")` 可重试，最多 1 次 |
| **自动压缩**       | `runtime.py:605-611`        | prompt 超预算且 history > 4 → 自动 compact 后重试            |
| **用户错误消息**     | `model_errors.py:15-20`     | 中文提示："模型返回空响应。可能原因：max_new_tokens 太小…"              |

---

### ④ 工具输出过长

| 防线 | 位置 | 机制 |
|------|------|------|
| **clip() 通用截断** | `workspace.py:26-30` | 所有工具输出截断至 `MAX_TOOL_OUTPUT=4000` |
| **run_shell 溢出转文件** | `tool_executor.py:131-139` | 输出 > 1000 chars 且是 run_shell → 写入 artifact，inline 只留前 1000 + 链接 |
| **Context 分预算** | `context_manager.py:15-31` | 总 prompt 预算 60000 chars，按 section 分配（history 30000、memory 8000…），超额按优先级缩减 |

---

### ⑤ Final 之前 Plan 未落盘

| 防线 | 位置 | 机制 |
|------|------|------|
| **can_finish() 文件检查** | `plan_mode.py:62-66` | plan 文件必须在磁盘存在且非空 |
| **engine 拦截 final** | `engine.py:353-372` | mode=plan 且 `can_finish()=False` → 替换为 `final_notice()`，继续循环 |
| **prefix 指令** | `plan_mode.py:74-81` | 注入提示："只有在写完了 plan artifact 之后才能返回 final" |
| **tool profile 限制** | `plan_mode.py:34` | plan 模式下只能写 `.pico/plans/` 路径 |

---

### ⑥ Worker 结果晚到

| 防线 | 位置 | 机制 |
|------|------|------|
| **每轮循环开始 drain** | `engine.py:121` | 每次构建 prompt 前拉取 worker 通知 |
| **每次 tool 后 drain** | `engine_helpers.py:45` | 每个工具执行完也拉通知 |
| **final 前后 drain** | `engine.py:352,422` | 收尾阶段再拉两次，确保通知不漏 |
| **通知作为 user 消息注入** | `engine.py:42` | 拉到的通知 `record({role:"user"})` → 下一轮 prompt 自动可见 |
| **notification_drained 标志** | `worker_manager.py:128,164-167` | 每个通知只消费一次，worker 启动时重置 |