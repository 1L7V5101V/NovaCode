代码层面就是 **new 了一个新的 `Pico(...)` 实例**（`worker_runtime.py:9`），通过构造函数参数传递关键差异：
[[session_store]]，[[workspace]]，
```Python
child = Pico(
    model_client=new_model_client(parent),   # 有 factory 就新建 client，否则共享父的
    workspace=WorkspaceContext.build(...),   # 重新扫描工作区
	    session_store=parent.session_store,      # 共享父 session 的存储
    run_store=parent.run_store,              # 共享 run 证据存储
    approval_policy="never"     # Explore
                   "auto"      # Worker
    read_only=True              # Explore
              False             # Worker（除非没给 write_scope）
    depth=parent.depth + 1,     # 深度 +1
    max_depth=parent.max_depth, # 继承父的阈值
    write_scope=...,            # Worker 传参，Explore 不传
    ...                         # 其余继承父：secret_env_names、feature_flags、sandbox_config 等
)
```

构造完之后还有 **两步收尾**（`worker_runtime.py:29-30`）：

```python
child.set_tool_profile("readonly" if subagent_type == "Explore" else "worker")
child.refresh_prefix(force=True)
```

`set_tool_profile` 把子 agent 能看到的工具集锁定到对应的 profile——Explore 看到的是从全工具集里减去 coordinator/mode/interactive 后的只读工具，Worker 看到的是全工具集减去 coordinator/mode/interactive/run_shell。`refresh_prefix` 则基于这个 profile 重新生成 system prompt，确保模型收到的工具列表和 profile 一致。

另外有个细节：`new_model_client()` 先尝试用父的 `model_client_factory()` 新建一个独立的 client，没有 factory 才共享父的 `model_client`。这决定了子 agent 能不能走后台线程——有独立 client 才能异步执行。