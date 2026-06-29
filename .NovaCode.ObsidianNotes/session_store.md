**SessionStore** 是文件存储层，不持有数据。它管理两个文件：

- **`sessions/<id>.json`**  
	- 完整的 session 状态，包含：`id`、`created_at`、`workspace_root`、`history`（整轮对话）、`memory`、`checkpoints`、`runtime_mode`、`workers`、`compactions`、`resume_state` 等
- **`sessions/<id>.events.jsonl`**
	- 事件流，每行一个 [[JSON event]]（turn_started、model_requested、tool_finished 等）

子 agent 共享父的 `session_store`，但不会往父的 session 里写——子 agent 有自己的 `session["id"]`，每次 `session_store.save()` 写的是子 agent 自己的 `<session_id>.json`。所以每个子 agent 会生成独立的 session 文件和事件流。