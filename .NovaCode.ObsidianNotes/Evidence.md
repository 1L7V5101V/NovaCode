**Evidence** 是每个 run 产生的审计工件，存在 `.pico/runs/<run_id>/` 目录下，三个文件：

| 文件                | 格式    | 内容                                                                                                               | 用途                      |
| ----------------- | ----- | ---------------------------------------------------------------------------------------------------------------- | ----------------------- |
| `task_state.json` | JSON  | 状态机快照：status、stop_reason、tool_steps、attempts、final_answer                                                        | 运行结束后看"这次 run 停在哪儿了"    |
| `trace.jsonl`     | JSONL | 事件流（每步 append）：prompt_built / model_requested / model_parsed / tool_executed / checkpoint_created / run_finished | 调试：还原每一步发生了什么、花了多久      |
| `report.json`     | JSON  | 聚合摘要：run_id、final_answer、compactions、todo_changes、workers、artifact_graph                                         | 复盘：不看细节，知道"这次 run 做了什么" |