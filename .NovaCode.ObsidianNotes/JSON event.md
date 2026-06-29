JSON event 是 `SessionEventBus.emit()` 往 `.events.jsonl` 里追加的一行 JSON。结构很简单（`session_events.py:20-28`）：

```json
{
  "event": "turn_started",
  "session_id": "20260609-120000-abc123",
  "created_at": "2026-06-09T12:00:00+00:00",
  "run_id": "run_...",
  "task_id": "task_..."
}
```

每个 event 都固定带 `event`、`session_id`、`created_at`，其余字段随事件类型变化。

全量事件类型覆盖整个 session 生命周期：

| 类别 | 事件 |
|------|------|
| **Session** | `session_started`, `session_cleared`, `model_changed` |
| **Turn** | `turn_started`, `user_message`, `model_requested`, `model_parsed`, `assistant_message`, `turn_finished` |
| **Tool** | `tool_started`, `tool_finished`, `permission_decision`, `tool_policy_decision` |
| **Worker** | `worker_started`, `worker_finished`, `worker_stop_requested`, `worker_notification_drained` |
| **Memory** | `memory_note_appended`, `auto_dream_started`, `memory_auto_dream_failed`, `memory_maintenance_failed` |
| **Plan** | `plan_mode_entered`, `plan_mode_exited` |
| **Skill** | `skill_invoked`, `skill_completed`, `skill_fork_completed` |
| **Model error** | `model_error`, `model_retry_scheduled` |
| **Context** | `context_usage_recorded` |
| **Compact** | `compaction_created` |
| **Todo** | `todo_changed` |

这个文件是 **session 级别的可持久化时间线**，和每个 turn 的 `trace.jsonl`（按 run 粒度、带 span_id 的详细诊断数据）用途不同——event bus 记录的是用户可见的粗粒度交互过程，trace 记录的是 engine 内部每一步的细粒度执行证据。