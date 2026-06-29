`record()` 覆盖了每次 turn 中发生的 **所有关键事件**，不只是工具调用。生产中一共 **9 处**调用，按角色分三类：

**`role: "user"`** — 2 处
```python
# engine.py:90   — 每次用户输入新消息，每轮 turn 第一条记录
agent.record({"role": "user", "content": user_message})

# engine.py:42   — 子代理完成后，通知以用户消息形式灌入
agent.record({"role": "user", "content": notification})
```

**`role: "tool"`** — 1 处
```python
# engine_helpers.py:36-44  — 每个工具执行完毕后，记录工具名+参数+结果
agent.record({"role": "tool", "name": name, "args": args, "content": result})
```

**`role: "assistant"`** — 6 处

| 位置 | 触发时机 |
|------|---------|
| `engine.py:374` | 模型正常返回 `<final>` |
| `engine.py:336` | 模型输出格式错误，要求重试 |
| `engine.py:355` | plan mode 下模型试图 final 但 plan 文件未就绪 |
| `engine_helpers.py:87` | 用户主动 abort |
| `engine_helpers.py:134` | 达到 `max_steps` 上限被切断 |
| `model_errors.py:29` | 模型调用异常（超时、429 等） |

所以一条完整 turn 的 history 记录序列是：

```
[user]    "重构 auth 模块"          ← 用户输入
[tool]    read_file main.go        ← 每个工具执行一次
[tool]    write_file auth.go
[tool]    run_shell go test ./...
[assistant] "改好了，加了 JWT 校验"  ← 模型最终回答
```

`record()` 做的事情不止是 append——每次调用 `turn_history.enrich(item)` 会附加 `turn_id`、`run_id`、`event_id`，然后调用 `session_store.save()` 落盘。所以 history 中的每一条记录有完整的溯源信息。