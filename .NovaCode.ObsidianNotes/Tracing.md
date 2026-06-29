**Tracing** 是 trace.jsonl 内部的事件关联方式。每个事件都带 `span_id` 和 `parent_span_id`（`runtime_events.py:34-37`）：

```
{"event": "prompt_built", "span_id": "span_000001", "parent_span_id": ""}
{"event": "model_requested", "span_id": "span_000002", "parent_span_id": "span_000001"}
{"event": "tool_executed", "span_id": "span_000003", "parent_span_id": "span_000002"}
```

这样 trace 文件里能还原出完整的调用链：哪次 prompt 导致了哪次 tool execution，又生成了哪个 checkpoint。这是借鉴了 **OpenTelemetry** 的 span 模型，但实现很轻——只是自增序列加一个 parent 引用，没有分布式追踪。

**和 SessionEventBus 的区别**：session event 是用户可见的粗粒度时间线（`turn_started`、`assistant_message`），一个 session 只有一个 `.events.jsonl`。trace 是 engine 内部的细粒度执行记录，每个 run 有自己的 `trace.jsonl`。session 可以包含几十个 run，每个 run 有自己的 trace。