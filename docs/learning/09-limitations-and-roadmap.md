# 09. 仓库现状、已知缺陷与演进路线

教学项目最忌讳“把 Demo 讲得像生产系统一样无懈可击”。这一章把整份代码里**真跑通了的、只写了一半的、以及已知的小 Bug**全部摊在桌面上，方便你以后基于它继续改造或对比真实生产框架。

## 9.1 全功能状态盘点（以当前源码为准）

| 模块/能力 | 真实状态 | 一句话实情 |
| --- | --- | --- |
| Mock / 真实模型切换 | 已接通 | 看配置里 `model.apiKey` 替换环境变量后是否非空 |
| 流式 `agentLoop` | 已接通 | 支持多步循环、重试退避、步数/Token 预算 |
| 死循环检测 | 已接通（有缺陷） | `history` 是模块级全局变量，多路并发会串台 |
| 内置文件/Shell/Web 工具 | 已接通（教学版） | 无路径沙箱；`bash` 用同步 `execSync`；`calculator` 用 `new Function()` |
| `tool_search` | 已接通（精确匹配） | 只按精确工具名查 `Map`，不是语义检索 |
| MCP 客户端 | 仅接了 Mock | 真实 `MCPClient` 类写了，但 `main.ts` 实例化的是 `MockMCPClient` |
| Session 持久化 | 只写不读 | `.sessions/*.jsonl` 会追加写入，但 `load()` 未接进 `--continue` |
| `PromptBuilder` 管道 | 已接通 | `makePromptCtx()` 里 `sessionMessageCount` 硬编码为 `0` |
| 上下文压缩 | 模块写好了，未接线 | `compressor.ts` 的 `microcompact` 和 `summarize` 没被主循环自动调用 |
| 上下文防线 | 手动触发 | 只有调试命令 `defend` 会调 `applyDefense()` |
| Usage 统计 | 部分接通 | 硬编码写 `.usage/today.jsonl`，未读 `config.usage.trackingFile` |
| RAG 混合检索 | 已接通（纯内存） | 向量 + BM25 + MMR 完整，但 `VectorStore` 没接数据库，重启重算 |
| Memory 长期记忆 | 已接通 | `.memory/` 持久化 + BM25 + `/lint` + `/dream` |
| Skill 技能系统 | 已接通 | 简化版逐行解析 Frontmatter；`/<skill>` 会同时写进 System 和 User |
| Plugin 插件系统 | 部分接通 | Supabase 是假数据示例；`main.ts` 没按 `config.plugins[].enabled` 过滤 |
| Channel 消息通道 | 已接通（仅内存） | 飞书 + Dashboard 都能进 `agentLoop`，但无鉴权、无持久化、无 Trace |
| 角色权限（RBAC） | 已接通（默认写死） | 三档角色生效；但 `config.security.defaultRole` 默认 `'developer'` 且未接线 |
| Bash 风险识别 | 已接通 | 几条正则分三级，防手滑不防故意绕过 |
| Hook Pipeline | 部分有 Bug | `block` 正常，异常 `fail-open`；`runPre()` 忘了把 `modifiedInput` 传出去 |
| Cron 定时调度 | 已接通（缺超时接线） | 进程内 `setTimeout` 调度；`runAgentPrompt` 没把 `timeout` 转成 `AbortSignal` |
| Sub-Agent 子任务 | 已接通 | 源码定义了 `SUB_AGENT_MAX_STEPS = 10` 但实际跑 `30` 步；绕开了锁和 Hook |
| Trace 追踪 | 仅覆盖 CLI 主循环 | Channel / Cron / Sub-Agent 都没接入 `LocalTraceRecorder` |
| 正式构建/部署 | 未提供 | 没有 `build` 脚本、Docker、健康检查或后台进程守护 |

## 9.2 源码里已知的小 Bug（按优先级值得顺手修掉）

1. **`HookPipeline.runPre()` 丢弃了修改后的参数**：`src/security/hooks.ts` 返回 `{ action: 'allow' }` 时没带上 `modifiedInput: currentInput`，导致 Pre Hook 改参数不生效；
2. **`Sub-Agent` 步数常量和实际不一致**：`src/agents/spawn.ts` 声明了 `SUB_AGENT_MAX_STEPS = 10` 却没用，下面写死了 `maxSteps = 30`；
3. **`guest` 角色执行 `ingest` 会崩溃**：`src/commands/rag.ts` 用 `!` 强行取 `rag_ingest` 工具，降权到 `guest` 后运行 `ingest` 会空指针报错；
4. **`pnpm run init` 会整份覆盖 `.env`**：`src/config/init.ts` 在用户直接输入 Key 时用 `writeFileSync` 重写 `.env`，会清掉 `example.env` 里原本的注释和其它变量；
5. **`/context` 统计展示小误差**：`src/commands/context.ts` 把 `skillsChars` 写死为 `0`，模型显示只判断 `process.env.DASHSCOPE_API_KEY`。

## 9.3 如果你想继续把这个项目改造成更完整的系统

建议按下面四步走，别一上来就乱加新概念：

### 第一步：先把现有“半截线”全部接通（最容易出成果）
- 修掉上面 5 个小 Bug；
- 接通 `pnpm run continue`（实验 9）；
- 在 `main.ts` 真正消费 `config.plugins[].enabled`、`config.rag.docsDir`、`config.usage.trackingFile`、`config.security.defaultRole`（把 Schema 默认值从 `'developer'` 改成 `'owner'`）；
- 在 `exit` 退出时补上 `await registry.closeAllMCP()`，并监听 `SIGINT` / `SIGTERM` 做统一清理；
- 给 Cron 的 `runAgentPrompt` 接上 `AbortController` 超时。

### 第二步：补测试与全链路可观测
- 除了现有的 `context/defense.spec.ts`，最值得补单测的是：`agentLoop`（重试、预算、取消）、`ToolRegistry`（权限 + 锁 + Hook）、`rag/search`（归一化与 MMR）、`config/loader`（变量替换）；
- 给 `ChannelGateway`、`CronService` 和 `spawnAgent` 也接入 `LocalTraceRecorder` 和 `UsageTracker`，不然外部通道出问题两眼一抹黑。

### 第三步：加固安全与并发隔离
- 把 `loop-detection.ts` 的全局历史改成每次 `agentLoop` 独立实例；
- 给 `ChannelGateway` 按 `sessionKey` 加串行队列，防止同一用户在网页上连点两次把消息数组跑乱；
- 给文件工具加上“工作区根目录校验”，禁止路径穿越读取 `~/.ssh` 等敏感文件；
- 把 `bash` 换成异步 `spawn` 子进程 + 可选沙箱，把 `calculator` 换成纯表达式解析器（去掉 `new Function()`）；
- 给 Dashboard HTTP 服务加上基本 Token 鉴权。

### 第四步：持久化与服务化
- 把 `VectorStore` 接到真正的持久化向量库（例如 SQLite + `sqlite-vec` 或 PostgreSQL `pgvector`）；
- 把 `Session`、`Memory`、`Cron` 从散落的文件升级为带锁或带事务的存储；
- 当上下文超过阈值时，自动在 `agentLoop` 前跑 `microcompact` / `summarize`。

## 9.4 学完这套代码后，可以带着哪些问题继续深挖

真正做 Agent 工程时，你会发现“调通模型”只占 10% 的工作量，剩下 90% 全在和下面这几个问题打交道：

1. **正确性**：模型参数填错、工具报异常、或者在多步任务里走偏了，运行时怎么帮它纠偏或优雅降级？
2. **安全边界**：最小权限怎么拆？哪些动作必须弹窗给人审批，绝对不能全自动跑？
3. **评测（Eval）**：改了切片大小、改了 RAG 权重或者换了模型后，怎么用一套固定测试集量化“到底变好了还是变差了”，而不是靠自己随便问两句拍脑袋？
4. **成本与延迟**：什么时候用贵的大模型做规划、什么时候用便宜快的小模型做摘要和路由？Prompt Cache 命中率怎么稳在 70% 以上？

这个仓库最大的好处，就是体量够小——上面这 4 个问题你都能在这个不到七千行的代码库里找到具体的落脚点，改完立马就能跑起来看到效果。
