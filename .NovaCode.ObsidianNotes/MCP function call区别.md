pico **没有使用 MCP**。它使用自建的 **纯文本 XML 协议**。以下是两者在三个核心维度上的对比：

---

### 1. Tool 定义方式

| | MCP | pico |
|---|---|---|
| 定义位置 | 独立的 MCP server，通过 JSON-RPC 注册 schema | system prompt prefix 中的纯文本列表（`runtime.py:440-447`） |
| 格式 | JSON Schema | 手写字段描述字符串 |
| 动态性 | 可热加载，server 运行时增减工具 | 启动时固定，重建 prefix 才变 |

pico 的 tool 定义在 prompt 里长这样：

```
- read_file(path: str, start: int=1, end: int=200) [safe] Read a UTF-8 file.
- run_shell(command: str, timeout: int=20) [approval required] Run a shell cmd.
```

---

### 2. Tool 调用方式

| | MCP | pico |
|---|---|---|
| 请求格式 | API 原生 `tool_use` / `function_call`（结构化 JSON 块） | 模型输出 `<tool>...</tool>` XML 标签 |
| 解析方式 | SDK 层自动解析（`stop_reason: tool_use`） | 正则 `re.findall(r"<tool\b...>(.*?)</tool>", raw)` |
| 多工具 | 一次响应可含多个 `tool_use` block | 支持 `<tool>...</tool><tool>...</tool>` 或 JSON list |
| 最终回复 | 模型输出纯文本，工具结果回传 | 专用 `<final>...</final>` 标签区分 |

pico 的两种调用格式：

```xml
<!-- JSON 格式 -->
<tool>{"name":"read_file","args":{"path":"src/main.py","start":1,"end":80}}</tool>

<!-- XML 属性格式 -->
<tool name="write_file" path="foo.py"><content>print("hello")</content></tool>
```

---

### 3. 协议层 vs 业务层

| | MCP | pico |
|---|---|---|
| 定位 | 通用协议层，连接 LLM 与外部工具服务器 | 自包含的文本协议，只服务于自己的 runtime |
| 校验 | MCP server 自己做参数校验 | `Pydantic` 模型校验 + `validate_tool()` 工作区感知规则 |
| 权限 | MCP 不涉及，由宿主自己控制 | `PermissionChecker` + `ToolPolicyChecker` 两层门禁 |
| 重试机制 | 无标准，宿主自实现 | `model_output.parse()` 返回 `"retry"` → 模型重试 |

---

### 总结

```
MCP:        LLM → [MCP JSON-RPC] → Tool Server (外部, 通用)
pico:       LLM → [<tool> XML 标签] → tool_executor.run_tool() (内存内, 自建)
                                          ↓
                                    validate_tool() → Pydantic
                                    permission_check
                                    tool_policy_check
                                    execute
```

pico 不依赖 MCP 的原因：它是一个**单体本地 agent**，所有工具都是内存内的 Python 函数（文件读写、shell、搜索），不需要外部 server 进程来回通信。自建 XML 协议的代价只是一次正则解析，省去了 MCP 的 JSON-RPC 序列化/反序列化和进程间通信开销。