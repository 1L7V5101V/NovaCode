### 为什么promptptefix里面要包括一个hash

> 因为 hash 出现在一堆依赖它的地方，而`build_prefix()` 是唯一知道 "prefix 到底是什么" 的地方，所以它一次性把 hash 算好带上。
> 
> 你看 hash 被用到的地方：
> 
> 1. **作为 prompt cache key 发给模型 API**（`runtime.py:543, 870`）——模型后端需要知道"这是哪份 prefix 的标识"，不是"这份 prefix 的文本是什么"。
> 
> 2. **`refresh_prefix()` 判断 prefix 这次到底变没变**（`runtime.py:397-398`）——它比较的是新旧两次 `build_prefix()` 返回的 hash。如果不存 hash，它就要把新旧两份 prefix 全文拿来逐字符对比，或者把旧 prefix 全文存着，都更麻烦。
> 
> 3. **写进 trace 和 report 做审计标记**（`runtime.py:543-544`）——运行后复盘时，看一眼 `prefix_hash` 就能知道"这轮用的 prefix 跟上一轮是同一份还是重建的"，不需要拉出整段 prefix 文本去做比较。
> 
> 所以 hash 不是 "text 的一个属性"，它是 **prefix 的身份标识**。`PromptPrefix` 作为一个打包好的 "系统 prompt 单元"，它需要同时提供**内容**（text）和**身份**（hash），因为不同消费者要的是不同的东西——模型后端要身份来命中缓存，prompt 元数据要身份来做审计追踪，`refresh_prefix()` 要身份来做变更检测。让 `build_prefix()` 在构造时一次性算出所有东西，比让每个调用方自己再算一遍 hash 更干净。