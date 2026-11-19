# 03. 上下文工程：让有限窗口装下最有用的信息

## 3.1 上下文不只是聊天记录

一次模型请求的上下文至少包括：

```text
System Prompt
  ├─ 核心规则
  ├─ 工具使用指南
  ├─ 延迟工具摘要
  ├─ 长期记忆索引
  ├─ RAG 状态
  ├─ 激活的 Skill
  └─ Session 信息

Tool Schemas
Conversation Messages
  ├─ 用户与助手文本
  ├─ tool-call
  └─ tool-result
```

上下文工程的目标不是“塞得越多越好”，而是在有限窗口里保留最能帮助下一步决策的信息。

## 3.2 PromptBuilder：用管道组装 System Prompt

`PromptBuilder` 保存一组有顺序的 Pipe：

```ts
builder
  .pipe('coreRules', coreRules())
  .pipe('toolGuide', toolGuide())
  .pipe('deferredTools', deferredTools())
  .pipe('memoryContext', memoryContext(memoryStore))
  .pipe('ragContext', ragContext(vectorStore))
  .pipe('skillContext', ...)
  .pipe('sessionContext', sessionContext());
```

每个 Pipe 接收 `PromptContext`，返回字符串或 `null`。`build()` 按顺序执行并以空行连接。

这种设计比一个巨型模板更容易：

- 单独启停某一段；
- 保持顺序稳定；
- 注入依赖；
- 测试每段输出；
- 让 Plugin 或后续功能增加新段落。

`PromptContext` 当前包含工具数量、延迟工具摘要、会话消息数和 Session ID。值得注意的是，`makePromptCtx()` 目前把 `sessionMessageCount` 固定为 0，因此该字段没有反映真实对话长度。

## 3.3 稳定前缀与缓存

System Prompt 前部越稳定，越有机会命中模型供应商的 Prompt Cache。管道顺序应遵循：

```text
稳定规则 → 稳定工具说明 → 半动态知识 → 高频变化的会话信息
```

如果把时间戳、随机数或每轮变化的统计放在最前面，可能导致后续大段内容无法复用缓存。当前项目展示了 Cache usage 和费用，但没有为不同供应商实现显式 cache_control 策略。

## 3.4 Session 持久化

`SessionStore` 把每条消息写成一行 JSON：

```json
{"type":"message","timestamp":"...","message":{"role":"user","content":"你好"}}
```

JSONL 的优点：

- 追加写简单；
- 单行损坏时可以跳过；
- 大文件可以流式处理；
- 便于命令行检查。

当前路径固定为 `.sessions/<sessionId>.jsonl`。`load()` 已经实现并会忽略格式错误行，但主启动流程始终从 `messages = []` 开始，`pnpm run continue` 也没有读取历史。因此当前是“持久化已写、恢复未接通”。

## 3.5 为什么上下文会膨胀

常见增长来源：

- 用户和助手多轮文本；
- 读取大文件产生的 tool-result；
- 搜索和网页抓取结果；
- 相同系统信息重复出现；
- 多步 Agent Loop 产生大量结构化消息；
- 工具 Schema 数量增加。

如果不治理，会导致费用上升、延迟增加、超过窗口或模型注意力被噪声分散。

## 3.6 Microcompact

`microcompact()` 是不调用模型的机械压缩：

- 保留最近若干工具结果；
- 对较早且可清理的工具结果替换成简短占位；
- 返回清理后的消息和节省 Token 估计。

优点是快、可预测、无额外模型费用；缺点是不能理解语义。如果早期工具结果中包含后续仍需要的关键数据，机械清理可能丢失信息。

适合优先清理的通常是可重新获取的观察结果，而不是用户约束、最终决策或无法重现的外部信息。

## 3.7 LLM 摘要压缩

`summarize()` 在消息超过阈值时：

1. 保留最近 6 条消息；
2. 把更早消息序列化；
3. 用模型根据固定 Prompt 生成结构化摘要；
4. 用一条摘要消息替换旧历史；
5. 返回压缩前后 Token 估计。

摘要要求保留目标、已完成工作、关键决策、文件变更、错误、下一步和用户偏好。这比普通“总结一下”更适合 Agent 接续任务。

风险包括：

- 摘要模型可能遗漏细节；
- 旧 tool-call 与 tool-result 的结构关系会消失；
- 每次再次压缩摘要会产生信息衰减；
- 压缩本身也消耗 Token。

当前 `summarize()` 和 `microcompact()` 已实现，但主 Agent Loop 没有自动调用它们。它们属于课程能力和可复用模块，不是当前默认运行路径。

## 3.8 三层即时防线

`src/context/defense.ts` 提供无需模型的即时治理。

### 工具结果截断

`truncateToolResults()` 遍历结构化 tool-result，对超过上限的文本保留头尾。它与 Registry 执行后的 `truncateResult()` 处于不同层：

- Registry 截断：结果刚产生时；
- Context Defense 截断：历史消息已经存在时再次治理。

### TTL Prune

`ttlPrune()` 根据消息时间和角色清理过旧内容。它依赖 `timestamps: Map<number, number>`，索引必须与消息数组保持一致。

### TokenTracker

TokenTracker 可以接受 API 提供的精确基线，再对新增或被替换消息做字符估算增减，避免每次从零估计。

### Apply Defense

`applyDefense()` 组合截断与 TTL，并给出前后 Token 估算和删除数量。当前主要通过 `defend`（或“执行防线”）调试命令手动触发，没有在每次 Agent Loop 前自动执行。

## 3.9 上下文可视化

`src/context/view.ts` 将上下文拆成多个 Slice，例如 System、Tools、Memory、RAG、Messages 和可用空间，生成终端矩阵与图例。

`/context` 会：

1. 重新构造 System Prompt；
2. 统计 Memory、RAG、工具 Schema 和消息字符；
3. 使用字符/token 比估算；
4. 渲染各部分占比。

这是一种诊断估算，不等价于特定模型 tokenizer 的精确结果。中文、代码和结构化 JSON 的换算比例不同。

## 3.10 Usage 与费用

`UsageTracker` 保存每个 Step：

- 时间；
- 模型名；
- 四类 Token；
- 估算费用。

记录会追加到 `.usage/today.jsonl`。`totals()` 还能计算缓存命中率、没有缓存时的假想成本和节省额。

当前主程序创建 Tracker 时硬编码 `.usage/today.jsonl`，没有使用 `config.usage.trackingFile`。价格表也只是代码内静态值。学习时应把它看作成本观测模式，而不是财务账单。

## 3.11 Memory/RAG 如何进入 Prompt

`memoryContext(memoryStore)` 调用 `buildPromptSection()`，主要注入记忆索引和使用原则，不会把每条完整记忆全部放入 System Prompt。需要详情时模型再调用 `memory read/search`。

`ragContext(vectorStore)` 当前只注入知识库片段数量和使用提示，不会根据本轮用户问题自动检索并把结果放进 Prompt。检索由模型主动调用 `rag_search` 工具完成。

这是“Agentic RAG”：模型决定何时检索，而不是每个请求都强制召回。

## 3.12 上下文治理的推荐顺序

实践中可以按成本从低到高治理：

```text
1. 控制工具输出长度
2. 延迟加载不常用 Tool Schema
3. 删除可重取的旧工具结果
4. TTL 清理明显过期内容
5. 对早期对话做 LLM 摘要
6. 必要时开启新 Session，并把关键决策写入 Memory
```

## 3.13 实践

在终端进行几轮文件读取和问答，然后运行：

```text
/context
/usage
defend
/context
```

比较防御前后的 Messages 占用。随后查看：

```text
.sessions/default.jsonl
.usage/today.jsonl
```

注意不要把包含敏感上下文的文件直接分享出去。

## 3.14 本章自检

1. System Prompt、Tool Schema 和 Messages 为什么要分别统计？
2. Registry 截断与 Context Defense 截断处于什么不同阶段？
3. Microcompact 和 LLM 摘要的优缺点分别是什么？
4. 当前项目的压缩是否自动接入 Agent Loop？
5. Memory 索引为何适合常驻 Prompt，而完整记忆不适合？
