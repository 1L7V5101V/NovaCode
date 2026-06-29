`test_memory.py` 覆盖 `LayeredMemory` 的四个核心功能，共 6 个测试：

| 测试                                                                   | 验证内容                                                                 |
| -------------------------------------------------------------------- | -------------------------------------------------------------------- |
| `test_working_memory_tracks_summary_and_recent_files`                | `working` 层的 task_summary 设置和 recent_files 去重+LRU 行为                 |
| `test_episodic_notes_append_and_retrieve_deterministically`          | 追加笔记后 `to_dict()` 返回完整列表，且 `retrieval_view()` 按 tag/关键词/时间排序返回 top N |
| `test_file_summaries_use_canonical_paths_and_freshness`              | 文件摘要的路径规范化、freshness 过期失效、手动失效清除                                     |
| `test_process_notes_keep_kind_and_latest_duplicate_wins`             | 同 text+同 kind 的笔记，保留最新一条（去重），并标记 `kind="process"`                    |
| `test_durable_memory_index_and_topic_notes_are_loaded_and_retrieved` | 从磁盘 `.pico/memory/` 加载持久化笔记，能在检索中混入结果                                |

本质上验证了三层记忆（working → episodic → durable）各自的基本操作以及 `retrieval_view()` 的跨层检索排序。

##### 1. `test_working_memory_tracks_summary_and_recent_files`

测试"短期记忆"：设置一个任务摘要（比如"调查不稳定的测试"），然后记住打开过的文件。多次打开同一个文件只算一次，且文件列表按最近使用排序，最新的在前面。

##### 2. `test_episodic_notes_append_and_retrieve_deterministically`

测试"事件笔记"的写入和查找：往记忆里塞几条笔记（有的带标签，有的不带，时间有先有后），然后验证：

- `to_dict()` 能原样吐出所有笔记
- 用关键词搜索时，结果按 **标签匹配 > 关键词重叠 > 时间新旧** 排序，只返回最相关的

##### 3. `test_file_summaries_use_canonical_paths_and_freshness`

测试"文件摘要"的自动过期：记下一段文件内容摘要后，如果文件被修改了，摘要就自动失效（不再显示）；也可以手动清除摘要。

##### 4. `test_process_notes_keep_kind_and_latest_duplicate_wins`

测试去重：如果两次追加完全一样的"过程笔记"（比如执行命令的记录），只保留最新的那一条，不重复堆积。

##### 5. `test_durable_memory_index_and_topic_notes_are_loaded_and_retrieved`

测试"长期记忆"的持久化：系统能在磁盘上保存笔记（`.pico/memory/` 目录下的 Markdown 文件），重启后还能加载回来，并且在搜索时一并参与排序召回。