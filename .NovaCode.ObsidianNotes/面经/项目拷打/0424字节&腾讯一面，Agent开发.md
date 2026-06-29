
| **面试问题**        | **表面上问什么**     | **实际考什么**                                              |
| --------------- | -------------- | ------------------------------------------------------ |
| 上下文如何压缩？        | prompt 怎么变短    | 你是否把上下文当成状态管理问题，而不是简单 summary                          |
| 恢复机制怎么做？        | 中断后怎么继续        | 你是否能防止 agent 盲信旧状态，是否有 freshness / runtime identity 校验 |
| 工具如何设计？         | 有哪些工具          | 你是否知道工具调用必须 fail-closed，模型不能裸奔执行外部动作                   |
| 压缩率怎么测试？        | 简历上16.19% 怎么来的 | 你是否做了可复现 benchmark，而不是凭感觉说优化了                          |
| 隐私问题怎么处理？       | 会不会泄露数据        | 你是否区分本地持久化、模型请求、环境变量、运行工件、长期记忆这几类风险                    |
| 5000 个工具如何精准调用？ | prompt 塞不下怎么办  | 你是否理解大规模工具调用要做工具路由、候选集检索、schema-aware rerank 和执行期校验    |

# 1.上下文是如何压缩的？（字节一面）

**我理解的上下文膨胀，本质上不是一个单纯的 token 问题，而是一个状态管理问题。**

**（先讲清楚需要压缩什么）**在多轮代码任务里，模型上下文里会混进很多不同寿命的信息，比如工具协议、仓库结构、当前任务目标、最近工具输出、读过的文件、失败过的尝试、临时结论、用户偏好等等。如果这些内容都按聊天记录不断往后追加，最后一定会出现三个问题：**成本越来越高，模型注意力被稀释，甚至旧信息会压过新事实**。

所以 Pico 在这块的设计不是简单做 history summary，也不是粗暴截断上下文，而是每一轮都重新构建一个分层、带预算、可恢复的 prompt。

**第一步是分层。Pico 会把上下文拆成几类不同寿命的信息。**

最稳定的是 prefix，比如 agent 身份、工具协议、工具清单、工具调用格式和仓库基础信息。这部分相当于模型的运行手册，告诉模型它现在处在什么环境里、能用什么工具、有哪些边界。

然后是 working memory，也就是当前任务的工作记忆。它不是完整聊天记录，而是一个紧凑的任务仪表盘，记录当前目标、任务进展、最近接触过的文件、文件摘要、关键决策和待办事项。它解决的是“模型下一轮需要知道任务做到哪里了”，而不是让模型回放所有历史。

再往下是 relevant memory。它会根据当前用户请求，从历史 notes 或 durable memory 里召回少量相关信息。这里我更强调可解释性，没有完全黑盒地把一大堆 embedding 结果塞回来，通过结合 tag、关键词重叠、时间新旧程度等因素排序，只取少量候选，避免“召回记忆”本身变成新的噪声源。

接着是 history。历史不会完全丢掉，但也不会原样堆进去。Pico 会优先保留最近几轮，因为下一步决策最依赖刚刚发生的工具结果；更早的历史会被折叠或摘要。比如重复读取同一个文件时，旧的 read_file 结果可以折叠；如果某个文件已经有可复用摘要，就用摘要替代全文；旧的 shell 输出也只保留命令和关键结果。

最后是 current request，也就是当前用户请求。这个一定放在最后，而且尽量不裁剪。因为无论前面有多少上下文，这一轮推理最终都必须对齐当前用户真正要解决的问题。很多 agent 跑偏，就是因为当前请求被埋在大量历史信息后面。

**第二步是压缩**。Pico 的压缩不是简单从尾部截断，而是按信息价值来收缩。大致原则是：当前请求和工具协议最不能丢；任务状态、关键约束、重要结论要尽量保留；可替代的历史细节、重复工具输出、长日志可以折叠或外置。

所以它会给不同 section 分预算，比如 prefix、memory、relevant memory、history 各有自己的预算。拼完之后如果仍然超预算，不是直接把后面的内容砍掉，而是按固定优先级收缩：先压缩相关记忆和历史，再压缩工作记忆，最后才考虑稳定前缀。current request 通常不动。

**第三步是恢复。**Pico 并不是把被压缩掉的信息放到本地状态和运行痕迹里。比如会话级状态会落在 `.pico/sessions/`，每次运行的 trace、task_state、report 会保存在 `.pico/runs/<run_id>/`。这样 prompt 里只保留当前推理真正需要的内容；如果后面需要细节，可以重新读文件、重跑工具，或者从 trace 里恢复。

最后还有一点我觉得很重要，就是可观测性。每一轮 prompt 构建时，Pico 会记录每个 section 原始有多大、实际渲染了多少、是否触发预算收缩、折叠了哪些重复内容、复用了哪些摘要。这使得上下文压缩不是一个黑盒行为，后面可以解释“为什么这一轮 prompt 是这样组成的”。

**所以总结一下，Pico 解决上下文膨胀的思路可以概括成三句话：第一，分层存放，不同寿命的信息不要混在一起；第二，按价值压缩，保留目标、约束、结论和锚点，丢掉噪音和冗余过程；第三，外置恢复，完整历史不一直占用上下文窗口，但需要时可以从文件系统和工具链里取回来。**

![](https://cdn.nlark.com/yuque/0/2026/png/65082237/1777261206953-d85b5320-c561-4007-9a89-b021df66cd44.png)

# 2.Pico怎么做恢复机制？（腾讯一面）

Pico 的恢复机制主要解决两个关键的问题：

第一，任务应该从哪里接着做？

第二，之前保存下来的状态现在还可信吗？

因为代码任务跑多轮以后，会产生很多中间信息，比如当前目标、读过哪些文件、执行过哪些工具、哪些尝试失败了、当前结论是什么、下一步准备做什么。如果这些东西全靠模型上下文一直记住，最后上下文会越来越大，而且更危险的是，代码文件可能已经变了，但模型还在相信旧摘要、旧结论。

所以 Pico 的思路是：**上下文可以压缩，但关键状态必须持久化；恢复时不回放完整历史，而是先恢复任务骨架，再按需补细节。**

这里可以拆成三个部分来看。

**第一部分是保存**。每一轮运行过程中，Pico 会把重要信息落到本地，而不是只放在模型上下文里。比如会话状态放在 `.pico/sessions/`，用来记录这个会话之前做到哪里；每次运行还会留下任务状态、运行轨迹和结果报告，也就是 `task_state`、`trace`、`report`。这些东西相当于任务的进度单和操作日志。

**第二部分是检查**。恢复时，Pico 不会直接相信旧状态，而是先判断它还能不能用。它主要会检查三类东西。

第一类是结构有没有变。比如旧检查点的结构版本和当前程序版本是否兼容。如果不兼容，就不能正常恢复，因为旧状态可能已经解释不对了。

第二类是文件有没有变。检查点里会记录当时哪些文件是关键文件，以及这些文件当时的状态。如果现在文件内容变了，那旧的文件摘要和旧结论就不能直接相信。这种情况不是完全不能恢复，而是只能部分恢复：任务目标、下一步这些骨架还能参考，但涉及变更文件的判断必须重新读文件确认。

第三类是运行环境有没有变。比如工作目录、模型配置、审批策略、只读模式、工作区指纹、工具签名这些是否一致。如果环境变了，Pico 就不能把它当成普通的继续执行，因为同一个任务在不同工作区或不同工具条件下，风险是不一样的。

**第三部分是装配**。校验完成后，Pico 给模型看的不是完整历史，而是一段结构化的恢复说明。这个恢复说明会告诉模型：当前恢复状态是什么，当前目标是什么，阻塞点是什么，下一步是什么，关键文件有哪些，哪些文件已经过期需要重新确认。

所以模型看到的不是一大段混乱历史，而是一张恢复后的任务仪表盘。它知道自己现在应该继续做什么，也知道哪些旧信息不能盲信。

举个例子，如果恢复状态显示某个关键文件已经变了，那模型下一步就应该先重新读取这个文件，而不是继续基于旧摘要做判断。这样可以避免 agent 带着过期状态继续执行。

最后，Pico 会把恢复后的任务状态、关键记忆、当前用户请求、必要的最近对话、少量文件片段和工具结果，重新组装成下一轮上下文。完整历史仍然保存在本地，可以用于复盘和追溯，但不会全部进入提示词。

所以我总结一下Pico 的恢复机制：

**它不是恢复聊天记录，而是恢复一个经过校验的任务状态。**

再展开就是三步：

**先落盘，保存任务状态和运行轨迹；再校验，判断旧状态是否可信；最后重装配，只把当前推理需要的高价值信息放回上下文。**

这样既能控制上下文体积，又能避免模型误信过期信息。

![](https://cdn.nlark.com/yuque/0/2026/png/65082237/1777261862382-368cd7c7-6f12-46bc-af9d-6392e5a8c1d8.png)

# 3.工具怎么设计（腾讯一面）

参考**0417字节一面**，类似的问题：

当前基础工具主要有 6 个：list_files、read_file、search 负责看目录、读文件、搜字符串；run_shell、write_file、patch_file 负责跑命令、写文件、精确改文件。除此之外，还有一个受限的 delegate，可以把一个小调查任务交给只读的子 agent 去做。

Pico 现在大概是 6 个基础工具，加 1 个受限委派工具。

工具调用这条链路，我做的是模型提申请，runtime 做安检，然后才执行。

模型在 prompt 里先看到工具清单、参数格式和调用示例，返回的是结构化的工具调用。

runtime 拿到模型输出后，会先解析成调工具还是直接结束，如果是调工具，不会直接跳到底层函数，而是统一先走 run_tool()。这里会按顺序做几层检查：先看工具是不是存在，再做参数校验，再拦连续重复调用，高风险工具再走审批，最后才真正执行。执行完以后，结果会写回 history、trace、report；如果涉及读写文件，还会顺手更新 memory，所以后面的上下文、恢复、审计都能接上。

这里在设计上我没有那么看重工具数量，更多考虑能不能有合适的边界。比如所有文件类工具都会先做路径解析，确保路径不能逃出当前工作区；patch_file 不是模糊修改，而是要求 old_text 精确命中且只能出现一次，这样修改行为才是确定的；run_shell 也不是完全放开，它有超时限制，工作目录固定在仓库根目录，环境变量也走 allowlist，不会把父进程环境整包带进去。还有一层很实用的保护，就是如果模型连续两次发起完全一样的工具调用，runtime 会直接拦掉，避免它在没有新信息的情况下空转。

如果要加一个新工具，可以顺着目前的链路加：

第一步，在 pico/tools.py 里把这个工具加进工具定义表，写清楚参数格式、风险等级和说明；

第二步，补一个调用示例，让模型知道该怎么正确发起调用；

第三步，在参数校验逻辑里把这类工具的输入边界写死；

第四步，写真正的执行函数；第五步，把它挂到 runner 映射里，让工具注册表自动把定义和执行绑起来。

这样接完以后，现有的审批、审计、重复调用拦截、路径边界这些护栏会自动接上。如果这个工具会影响工作区或者会产生对后续推理有价值的结果，我再补 memory 更新和测试。

总结：

Pico 的工具层本质上是显式工具表 + 统一执行闸口 + 风险护栏。设计的重点在它每次调工具时，harness这套有没有先把边界守住。

# 4.压缩率怎么测试？（腾讯一面）

我当时没有简单看某一轮 prompt 变短了多少，而是把上下文治理单独拆成了一组 benchmark 来测。

具体做法是固定任务、固定会话历史、固定工具预算和模型配置，然后在同一组任务上跑两版：一版不开 context reduction、memory、relevant memory，作为 baseline；另一版打开这些机制，作为 reduced 版本。最后比较两版最终组装出来、真正发给模型的 prompt 长度。

压缩率的计算比较直接：

compression_rate = (baseline_prompt_len - reduced_prompt_len) / baseline_prompt_len

在 Pico 这个阶段，我用的是字符数作为近似指标，而不是直接用 token。原因是不同模型后端的 tokenizer 不一样，同一段中文、代码、日志，在 GPT、Claude、Qwen 里切出来的 token 数可能不同。如果一开始就绑定某个 tokenizer，指标会偏向某个模型后端。Pico 当时更想验证的是上下文治理机制本身有没有效果，所以先用字符数作为跨模型的统一 proxy。

当然，我不会只记录一个总长度。Pico 的 prompt 是分 section 组装的，比如 prefix、memory、relevant memory、history、current request。每个 section 都会记录 raw chars、budget chars、rendered chars。这样我能看到压缩具体发生在哪里：是旧 history 被折叠了，还是重复工具输出被摘要替代了，还是相关记忆被裁剪了。

但这里最重要的是，我不会只看 prompt 变短。因为 prompt 变短不一定代表系统变好，最极端的情况是把历史全裁掉，prompt 肯定很短，但模型可能忘记用户约束或者任务状态。所以我会同时看正确性指标：任务有没有在预算内完成，verifier 有没有通过，stop reason 是否是正常 final answer，current request 有没有被完整保留，关键记忆和 checkpoint 状态有没有丢。

所以我理解的压缩率不是一个孤立指标，而是要和任务正确性一起看。Pico 里最终报的平均压缩率 16.19%，是在固定 benchmark、固定配置，并且 verifier 通过、当前请求不被裁坏的前提下统计出来的。这样这个数字才有意义，否则单纯说 prompt 少了多少，很容易变成一个“好看的数字”，但不能证明系统真的变好了。

# 5.隐私问题怎么处理？（腾讯一面）

这个问题不要简单答“我们是本地项目，所以隐私没问题”。

本地运行只解决了一部分问题。真正需要分清楚几类风险：

- 本地会话和运行工件会不会泄露；
- prompt 发给远程模型时会不会带敏感代码或 secret；
- shell 命令会不会继承敏感环境变量；
- trace/report 里会不会保存 API key、token、password；
- durable memory 会不会把临时状态或敏感内容长期保存；
- 工具路径会不会越过工作区访问其他文件。

所以回答时要诚实：**Pico 是 local-first，不是天然零泄露。它通过本地持久化、secret redaction、环境变量 allowlist、路径边界、审批和 durable memory 过滤来降低风险；如果接远程模型，仍然要考虑模型请求的数据边界。**

Pico 的隐私设计主要分几层。

第一层是 **local-first 持久化**。session 保存在本地 `.pico/sessions/`，运行工件保存在本地 `.pico/runs/<run_id>/`，里面包括 task_state、trace、report。这些默认只在本地，不需要跟仓库一起提交。

第二层是 **secret redaction**。Pico 会识别环境变量名里带 `API_KEY`、`TOKEN`、`SECRET`、`PASSWORD` 这类 marker 的变量，也支持用户显式配置 secret_env_names。写 trace、report、artifact 时，会对这些值做 redaction。也就是说，运行工件里尽量只留下 secret 名称和数量，不留下真实值。

第三层是 **shell 环境隔离**。run_shell 不是直接继承整个父 shell 环境，而是只保留 allowlist 里的环境变量，比如 HOME、PATH、LANG、TMPDIR、USER 等。这样即使模型让系统跑命令，也不会默认把一堆 API key、token 带进子进程。

第四层是 **路径边界**。所有文件工具都会通过 workspace root 解析路径，并检查 resolved path 是否仍在 repo root 下面。这样可以防止 `../` 逃逸，也可以防止符号链接解析后跳出仓库。

第五层是 **高风险审批**。run_shell、write_file、patch_file 是 risky tools，会受 approval_policy 控制。ask 模式需要人工批准，never 模式直接拒绝。delegate 子 agent 是 read-only，不会拿到写权限。

第六层是 **durable memory 过滤**。长期记忆不是模型想写什么就写什么。Pico 会过滤 secret-shaped text，比如 API key、token、sk- 形态文本；也会过滤 traceback、stdout/stderr、exit_code 这类 noisy output，以及 current goal、next step、key files 这类 transient task state。这样可以避免把短期过程、错误日志或敏感信息沉淀成长期记忆。

## 追问：如果接远程模型，代码内容不是还是发出去了？

是的，这个要诚实承认。

Pico 的 local-first 主要是指运行状态和工件默认本地保存，不代表 prompt 不会发给模型后端。如果用户配置的是远程 OpenAI/Anthropic 兼容接口，那么当前 prompt 里包含的仓库片段、工具结果和当前请求会发送给模型服务端。

所以更严谨的说法是：Pico 当前做的是 **最小化、分层和脱敏**，不是完全离线保密。如果企业级上线，我会再补几类能力：

第一，支持本地模型或企业私有模型网关，让敏感仓库不出内网。

第二，prompt 出口前加敏感信息扫描，比如 secret scanner、PII detector、路径 denylist。

第三，对 `.pico/` 本地工件做加密或至少提供自动清理策略。

第四，给项目加 `.picoignore`，明确哪些路径不能进入 prompt，比如 `.env`、证书、密钥目录、客户数据样本。

第五，把审计日志和模型请求日志分开，方便企业按合规要求做 retention。

# 6.如果有 5000 个工具，如何让模型精准调用？（腾讯一面）

**5000 个工具时，肯定不能让模型记住 5000 个工具，上下文直接撑爆，更合适的做法是让 harness 把 5000 个工具变成当前任务下的几十个候选动作。我的设计会是 ToolRegistry 离线管理工具卡片，ToolRetriever 高召回找候选，ToolRouter 做 schema-aware rerank，prompt 只注入 active toolset，runtime 只允许调用 active set，并继续做 schema、权限、审批和审计。**

如果有 5000 个工具，我不会把它们直接塞进 prompt。这个问题本质上不是 prompt 长度问题，而是工具路由问题。直接塞 5000 个工具，不仅 prompt 会爆，模型也会在大量相似工具之间混淆。

我会把工具调用拆成三层。

第一层是 ToolRegistry，每个工具都有 ToolCard，包含 namespace、description、schema、risk、side effect、权限、tags、examples 和 negative examples。

第二层是 ToolRetriever/ToolRouter，根据用户请求、当前上下文、权限和风险策略做混合检索和 rerank。候选生成要高召回，可以用关键词、namespace、embedding、schema 匹配；rerank 要看 schema 是否能满足、当前权限是否允许、风险是否符合用户意图。

第三层才是模型调用：prompt 里只注入 top-k active tools，比如 10 到 30 个，让模型在这个小集合里选择。

执行时 runtime 还要兜底。模型只能调用 active set 里的工具，不能凭空写一个工具名；参数必须通过 schema validation；高风险工具必须审批；如果没找到合适工具，可以调用 search_tools 或 describe_tool 继续检索，也可以 no_tool 追问用户。

（**如果你了解一点后训练，也可以补充一下）**后训练可以优化的是模型使用工具路由协议的能力，比如什么时候搜索工具、怎么读 ToolCard、怎么区分相似工具、缺参数时怎么追问、工具失败后怎么修正。但我不会让模型靠参数记住 5000 个工具，因为工具集会变，schema 会变，权限也会变。系统侧的候选集管理和安全边界仍然必须存在。

## 追问：ToolCard 应该怎么设计？

一个比较完整的 ToolCard 可以包括：

```
id: github.issue.create
namespace: github.issue
name: create_issue
description: Create a GitHub issue in a repository.
when_to_use: 用户明确要求创建 issue，且 repo、title、body 已知
when_not_to_use: 只是搜索 issue、评论 issue、关闭 issue 时不要用
input_schema: {repo: string, title: string, body: string, labels?: string[]}
output_schema: {issue_url: string, issue_number: number}
risk_level: write_external_system
side_effects: creates an external issue
required_auth: github_token
permissions: repo:write
tags: [github, issue, create, ticket]
examples: ...
negative_examples: ...
version: v2
owner: devtools-platform
```

比如 “把这个 bug 记录一下” 可能对应 Jira、GitHub Issue、飞书文档、Slack 消息、内部缺陷系统。只靠描述相似度，很容易选错。必须结合当前 workspace、用户权限、组织默认流程、请求里的实体和风险策略去判断。

## 追问：运行时路由怎么做？

我会把路由拆成四步。

**第一步，粗粒度意图分类。**

先让系统判断当前任务属于哪个大类，比如代码仓库读写、CI/CD、GitHub、Jira、Slack、数据库、云资源、文档、监控告警等。这里不需要模型看到所有工具，只需要看到几十个 namespace 或 capability category。

**第二步，候选生成。**

在被选中的 namespace 里做候选工具检索。候选生成要高召回，宁愿多召回一些相近工具，也不要一开始把正确工具漏掉。可以混合使用：

- 工具名、namespace、tag 的精确匹配；
- BM25 / keyword search；
- embedding semantic search；
- schema 字段匹配；
- 当前上下文约束，比如当前仓库、当前用户权限、当前平台；
- 历史使用统计，比如这个项目常用哪些工具。

**第三步，schema-aware rerank。**

候选生成后，再做排序。这里不能只看语义相似，还要看工具 schema 是否能被当前上下文满足。比如某个工具需要 `ticket_id`，但当前上下文只有 repo 和 commit hash，那它就不应该排在前面。某个工具会产生外部写入副作用，如果用户只是问“查一下”，也要降权。

一个简单的打分可以是：

```
score = lexical_match
      + semantic_similarity
      + schema_fit
      + context_fit
      + permission_fit
      + historical_success
      - risk_penalty
```

这里权重可以按业务调，比如代码工具里精确符号/路径命中权重更高，办公工具里语义和权限权重更高。

**第四步，active toolset 注入。**

最后只把 top-k 工具注入 prompt，比如 10 到 30 个。每个工具卡片也不要太长，只保留 name、description、schema、risk、关键示例。完整说明可以通过 `describe_tool` 再查。

这时模型看到的是一个小而相关的工具集合，而不是 5000 个全量工具。

## 追问：为什么只做工具 RAG 还不够？

工具 RAG 只解决找候选的一部分问题，但不解决三个关键问题。

第一，它不一定理解 **schema 是否可满足**。一个工具语义上很像，但当前缺关键参数，模型可能硬填一个幻觉参数。

第二，它不一定理解 **side effect 和权限**。比如发送消息、删资源、改数据库，这些工具就算语义匹配，也不能随便调用。

第三，它不一定理解 **相似工具的边界**。比如 `search_issue`、`create_issue`、`comment_issue`、`close_issue` 都和 issue 相关，但动作完全不同。RAG 召回以后还需要 schema-aware rerank 和 runtime validation。

## 追问：如果相似工具很多怎么办？

相似工具多的时候，不能只靠工具描述。要引入 hard negative 和反例。

比如：

- `github.issue.create`：创建 issue；
- `github.issue.comment`：评论 issue；
- `github.issue.search`：搜索 issue；
- `jira.ticket.create`：创建 Jira ticket；
- `slack.message.send`：发 Slack 消息。

这些工具都可能和“记录一下这个 bug”相关。ToolCard 里要明确 when_to_use 和 when_not_to_use。rerank 时还要看当前上下文有没有 repo、issue_number、channel、jira_project_key 这些必要参数。没有必要参数的工具要降权，写外部系统的工具要加风险惩罚。

如果用户意图不清楚，系统应该让模型追问，或者先调用 read-only 查询工具，而不是直接调用写工具。