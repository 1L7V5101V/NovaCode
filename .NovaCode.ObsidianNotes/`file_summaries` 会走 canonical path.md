意思是对 `file_summaries` 的所有操作（存入、读取、失效），路径都会经过 `canonicalize_path()` 归一化后再作为 key。

看 `canonicalize_path()` 的逻辑（`memory.py:265-272`）：

```python
def canonicalize_path(raw_path, workspace_root=None):
    resolved = resolve_workspace_path(raw_path, workspace_root)
    ...
    root = Path(workspace_root).resolve()
    return resolved.relative_to(root).as_posix()
```

三步：
1. 把路径解析成绝对路径
2. 裁剪成相对于工作区根目录的相对路径
3. 反斜杠转正斜杠

结果：`".\\sample.txt"`、`"./sample.txt"`、`"SAMPLE.txt"`（如果 OS 大小写不敏感）、`"foo/../sample.txt"` 最终都会变成同一个 key `"sample.txt"`。这就避免了同一文件因写法不同而产生多条摘要记录。