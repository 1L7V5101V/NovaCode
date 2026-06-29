这些数据来自 `pico/metrics.py` 里的 `run_large_scale_memory_experiment()`（大规模记忆实验），流程如下：

**步骤 1：定义 12 个任务**（`metrics.py:330-343`），分三类：
- `fact_lookup` — 比如"部署密钥是红色"这样的事实
- `edit_dependency` — 比如"第一行是被锁定的引言"
- `history_reference` — 比如"部署事实来自 facts.txt 文件"

**步骤 2：对每个任务跑三次变体**（`_run_memory_task_variant()`, `metrics.py:381-400`）：

| 变体 | 做法 |
|------|------|
| `memory_on` | 正常记忆系统，记下相关事实 |
| `memory_off` | 关闭 memory 和 relevant_memory 功能 |
| `memory_irrelevant` | 记忆里只塞一条无关信息（"团队吉祥物是蓝色"） |

**步骤 3：用假模型客户端**（`_MemoryExperimentModelClient`, `metrics.py:219-253`）模拟 agent 执行。它不调真实 LLM，而是检测 prompt 里是否已有答案——有就直接返回，没有就触发 `read_file`（计为一次"重复读取"）。

**步骤 4：算指标**，每个变体跑完所有任务后聚合：

| 指标 | 计算方式 | 对应结果 |
|------|---------|---------|
| `repeated_reads` | 所有任务中 `read_file` 触发次数之和 | `memory_off: 8次 → memory_on: 3次` |
| `avg_tool_steps` | 平均每次任务调了多少次工具 | `memory_off: 0.67 → memory_on: 0.25` |
| `correct_rate` | 答案正确的比例 | `memory_off: 66.7% → memory_on: 100%` |

**`memory_irrelevant` 没带来同样收益** → 证明效果来自**相关记忆内容本身**，而不是"prompt 里多塞点文字"这个副作用。