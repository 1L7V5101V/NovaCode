**WorkspaceContext** 是仓库的"第一印象"快照，包含：

|字段|来源|示例|
|---|---|---|
|`cwd`|启动目录|`/home/user/project`|
|`repo_root`|`git rev-parse --show-toplevel`|`/home/user/project`|
|`branch`|`git branch --show-current`|`main`|
|`default_branch`|`git symbolic-ref refs/remotes/origin/HEAD`|`main`|
|`status`|`git status --short`|`M src/foo.py`|
|`recent_commits`|`git log --oneline -5`|`[abc123 fix bug, ...]`|
|`project_docs`|白名单文档（README.md、pyproject.toml、AGENTS.md、package.json）每个截取 1200 字符|`{"README.md": "# pico\n..."}`|

子 agent 在 `build_child_runtime()` 里重新 `WorkspaceContext.build(parent.root, ...)` 扫了一遍仓库——这是有代价的，但保证了子 agent 看到的是当前最新状态，而不是父 agent 启动时的过时快照。`fingerprint()` 用 SHA256 对这些字段整体哈希，用来判断是否需要重建 prefix。