# 附录：命令、工具、Mock 口令与源码速查

这一页当成工具书用：忘了某个命令怎么敲、Mock 模式要输什么词才能触发工具、或者想按文件名反查章节，直接来这里搜。

## A.1 `package.json` 脚本

| 命令 | 实际执行的命令 | 说明 |
| --- | --- | --- |
| `pnpm start` | `tsx src/index.ts` | 正常启动 Agent 和 Dashboard |
| `pnpm run init` | `tsx src/index.ts init` | 交互式生成 `super-agent.config.json`（和可选 `.env`） |
| `pnpm run continue` | `tsx src/index.ts --continue` | 当前逻辑与普通启动完全一样，尚未接入恢复历史 |
| `pnpm run trace:inspect <file>` | `tsx src/trace/inspect.ts <file>` | 打印指定 `.traces/*.jsonl` 的步数、耗时、Token 和工具摘要 |

## A.2 交互命令（在 `You:` 后输入）

| 命令 | 别名 / 子命令 | 作用 |
| --- | --- | --- |
| `/context` | `context` | 渲染 16×16 上下文窗口占用矩阵 |
| `/usage` | `usage` | 查看累计 Token、缓存命中率与估算费用 |
| `/memory` | `memory` | 列出所有记忆条目并标出有问题项 |
| `/memory search <词>` | `搜记忆 <词>` | 用 BM25 搜索记忆条目 |
| `/lint` | `lint` | 检查记忆库健康度（注意：**没有** `/memory lint`） |
| `/dream` | `dream` | 触发四阶段 Prompt 让模型自动整理/清理记忆 |
| `/rag` | `rag` | 查看知识库切片数和来源文件列表 |
| `ingest <路径>` | — | 手动把单个 Markdown 导入内存知识库（注意别在 `guest` 角色下跑） |
| `/skill list` | `/skill`、`skill list` | 列出 `.skills/` 下可用技能及激活状态 |
| `/skill load <名>` | — | 激活某个技能进 System Prompt |
| `/skill unload <名>` | — | 停用某个技能 |
| `/<skill-name> [指令]` | 例如 `/code-review` | 快捷方式：激活技能并把技能正文作为用户消息直接跑一轮 |
| `/plugin` | `/plugin list` | 查看已加载和可加载的插件 |
| `/plugin load <名>` | `/plugin unload <名>` | 运行时加载/卸载插件 |
| `/channel` | `/channel list` | 列出已注册的消息通道 |
| `/role` | `/role owner\|collaborator\|guest` | 查看或切换当前工具权限角色 |
| `/hooks` | — | 查看已注册的 Pre / Post Hook |
| `/cron` | `/cron list`、`/cron logs` | 列出定时任务或查看最近 10 条执行记录 |
| `/agents` | — | 查看当前进程的 Sub-Agent 执行记录 |
| `/cache on` | `/cache off`（可不带 `/`） | 开关 Mock 模型的 Prompt Cache 模拟 |
| `sim` | `模拟长对话` | 注入 20 条模拟历史消息用于测试上下文膨胀 |
| `defend` | `执行防线` | 手动执行一次三层即时上下文防御 |
| `status` | `查状态` | 打印消息数、Token 估算、记忆数、RAG 片段数 |
| `exit` | 空回车也会退出 | 停止 Cron、Channel 和 Plugin 后退出 |

## A.3 内置工具清单

| 工具名 | 读写/并发属性 | 默认可见 | 作用说明 |
| --- | --- | --- | --- |
| `get_weather` | 只读 / 并发 | 是 | 查固定城市模拟天气（北京/上海/深圳/广州/杭州/成都等） |
| `calculator` | 只读 / 并发 | 是 | 表达式计算（教学实现用 `new Function()`） |
| `read_file` | 只读 / 并发 | 是 | 分段读取本地文件（支持 offset / limit） |
| `write_file` | 写 / 独占 | 是 | 自动建目录并覆写文件 |
| `edit_file` | 写 / 独占 | 是 | 精确字符串替换局部内容 |
| `list_directory` | 只读 / 并发 | 是 | 列出目录条目和类型 |
| `glob` | 只读 / 并发 | 是 | 简化通配符查文件（最多 100 条） |
| `grep` | 只读 / 并发 | 是 | 正则搜文件内容（最多 50 条） |
| `bash` | 写 / 独占 | 是（`owner`） | 同步执行 Shell 命令（受 Bash 风险分级和 Hook 约束） |
| `web_search` | 只读 / 并发 | 是 | 按环境变量选用 Tavily 或 Serper 搜索 |
| `web_fetch` | 只读 / 并发 | 是 | 抓取网页 HTML 并转成 Markdown |
| `tool_search` | 只读 / 并发 | 是 | 按精确工具名激活延迟工具 |
| `memory` | 写 / 独占 | 是 | 长期记忆的 save / list / search / read / delete / lint |
| `rag_ingest` | 写 / 独占 | 是 | 读取文档、切块、向量化并加入 `VectorStore` |
| `rag_search` | 只读 / 并发 | 是 | 向量 + BM25 + MMR 混合检索 |
| `cron_manage` | 写 / 独占 | 是 | 让模型创建、删除、启停、查看定时任务 |
| `spawn_agent` | 调度 | 是 | 派发单个或并行的 Sub-Agent |
| `mcp__github__*` | 只读 / 并发 | **延迟** | 默认连 `MockMCPClient`：`list_issues`、`search_repositories`、`get_file_contents` |
| `supabase__*` | 视工具而定 | 是 | 示例插件工具：`list_tables`、`query`、`insert`（均返回模拟数据） |

## A.4 Mock 模式触发口令速查

在没配 `DASHSCOPE_API_KEY` 时，`src/mock-model.ts` 只认识下面这些固定关键词（不按顺序匹配到的话，会直接回复默认问候语）：

| 想测的场景 | 输入关键词（包含即可） | Mock 会触发什么 |
| --- | --- | --- |
| 并发工具调用 | `测试并发` 或 `test parallel` | 并行调用 2 次 `get_weather` + 1 次 `list_directory` |
| 死循环检测 | `测试死循环` | 反复调用 `get_weather(北京)` 直到触发熔断 |
| 重试机制 | `测试重试` 或 `test retry` | 前 2 次抛 429 限流错，第 3 次成功返回 |
| Dream 多步整理 | `dream`（或输入 `/dream` 命令） | 依次跑 `memory(list)` → `lint` → 删两条示例过期记忆 |
| 记住信息 | `记住 ...` 或 `remember ...` | 调用 `memory(save)` |
| 记忆体检 / 搜索 / 列表 | `lint 记忆` / `搜记忆 <词>` / `我的记忆` | 分别触发 `memory(lint)` / `search` / `list` |
| 延迟工具发现 | `issue` 或 `github` | 先调 `tool_search`，再调 `mcp__github__list_issues` |
| RAG 检索 | `部署` / `事故` / `回滚` / `监控` / `知识库` | 调用 `rag_search` |
| RAG 导入 | 同时含 `导入`/`ingest` 和 `文档`/`.md` | 调用 `rag_ingest` |
| Supabase 插件 | `有哪些表` / `查用户` / `查帖子` / `插入` / `数据库` | 调用对应 `supabase__*` 工具 |
| 内置工具测试 | `测试截断`、`测试编辑`、`测试搜索`、`测试glob`、`测试bash`、`测试危险`、`测试写文件` | 分别触发 `read_file`、`edit_file`、`grep`、`glob`、`bash`、`rm -rf` 拦截、`write_file` |
| 读特定文件 | `读取 package.json`（含 `读`/`read` + 带后缀文件名） | 调用 `read_file` |
| 查天气 / 算术 | `北京天气`（6 城市之一 + 天气词）/ `12 * 8` | `get_weather` / `calculator` |
| **不支持 Mock 的能力** | `cron_manage`、`spawn_agent` | **Mock 里没有对应规则，需接真实 API Key 测试** |

## A.5 环境变量与配置文件结构

| 环境变量 | 用途 | 缺失或留空会怎样 |
| --- | --- | --- |
| `DASHSCOPE_API_KEY` | 对话模型 + RAG Embedding | 仅当配置中引用它且 `.env` 里留空时进 Mock |
| `FEISHU_APP_ID` / `FEISHU_APP_SECRET` | 飞书 Bot 长连接 | 任一为空则跳过飞书，仅起 Dashboard |
| `FEISHU_PORT` | Dashboard HTTP 端口 | 默认 `3000` |
| `TAVILY_API_KEY` / `SERPER_API_KEY` | 网页搜索 | 优先 Tavily；都空则不可搜 |
| `SUPABASE_URL` / `SUPABASE_KEY` | 预留给插件示例 | 当前无论是否填写主要返回 Mock 数据 |

## A.6 全源码文件 → 章节索引

### 入口与模型
| 文件 | 职责 | 对应章节 |
| --- | --- | --- |
| `src/index.ts` | 区分 `init` 和正常启动 | 00、07 |
| `src/main.ts` | 总装配车间、生命周期、CLI 交互 | 00、07 |
| `src/mock-model.ts` | 无密钥模拟模型、流式和 Cache 模拟 | 01、附录 |

### Agent 核心与子 Agent
| 文件 | 职责 | 对应章节 |
| --- | --- | --- |
| `src/agent/loop.ts` | 主 Agent 多步循环 | 01 |
| `src/agent/retry.ts` | 错误分类与指数退避重试 | 01 |
| `src/agent/loop-detection.ts` | 重复/乒乓/全局熔断检测 | 01、09 |
| `src/agents/types.ts` | 子 Agent 请求与记录类型 | 06 |
| `src/agents/registry.ts` | 子 Agent 状态与并发/深度限制 | 06 |
| `src/agents/spawn.ts` | 单个及并行子 Agent 执行 | 06、09 |

### 工具系统
| 文件 | 职责 | 对应章节 |
| --- | --- | --- |
| `src/tools/registry.ts` | 工具注册、过滤、锁、Hook 包装、截断 | 02、06 |
| `src/tools/index.ts` | 基础内置工具清单 | 02 |
| `src/tools/file-tools.ts` | 文件读写、编辑与列目录 | 02 |
| `src/tools/search-tools.ts` | 简版 `glob` 和 `grep` | 02 |
| `src/tools/shell-tools.ts` | `bash` 同步执行 | 02、06 |
| `src/tools/web-search.ts` | Tavily/Serper 搜索和网页抓取 | 02 |
| `src/tools/utility-tools.ts` | 天气和计算器示例 | 02 |
| `src/tools/tool-search.ts` | 延迟工具发现工具 | 02 |
| `src/tools/mcp-client.ts` | 真实 stdio MCPClient 与 MockMCPClient | 02、05 |
| `src/tools/memory-tools.ts` | `memory` 工具包装 | 04 |
| `src/tools/rag-tools.ts` | `rag_ingest` / `rag_search` | 04 |
| `src/tools/cron-tools.ts` | `cron_manage` 工具 | 06 |
| `src/tools/spawn-tools.ts` | `spawn_agent` 工具 | 06 |

### 上下文、Session、Usage、Trace
| 文件 | 职责 | 对应章节 |
| --- | --- | --- |
| `src/context/prompt-builder.ts` | Prompt 管道和内置片段 | 03 |
| `src/context/prompt-pipes.ts` | Memory 和 RAG 上下文注入段 | 03、04 |
| `src/context/compressor.ts` | `microcompact` 与 LLM 摘要压缩 | 03、09 |
| `src/context/defense.ts` | TokenTracker、截断、TTL 防线 | 03 |
| `src/context/defense.spec.ts` | 单元测试 | 03、09 |
| `src/context/tool-result-output.ts` | 结构化工具结果文本提取 | 03 |
| `src/context/view.ts` | `/context` 与 `/usage` 终端渲染 | 03 |
| `src/session/store.ts` | Session JSONL 读写 | 03、07、08 |
| `src/usage/tracker.ts` | Usage 规范化、单价表和缓存统计 | 01、03、07 |
| `src/trace/recorder.ts` | JSONL 执行轨迹记录与摘要读取 | 07 |
| `src/trace/inspect.ts` | `trace:inspect` 命令行入口 | 07 |

### 记忆、RAG、扩展与治理
| 文件 | 职责 | 对应章节 |
| --- | --- | --- |
| `src/rag/chunker.ts` | Markdown 切片 | 04 |
| `src/rag/embedder.ts` | Mock/DashScope Embedding 与进程内缓存 | 04 |
| `src/rag/store.ts` | 内存向量存储 | 04、09 |
| `src/rag/search.ts` | 混合检索（向量 + BM25 + 归一化 + MMR） | 04 |
| `src/memory/store.ts` | `.memory/` 读写、索引与生命周期 | 04 |
| `src/memory/search.ts` | 中英文 BM25 搜索 | 04 |
| `src/memory/validator.ts` | 记忆字段和文件路径健康检查 | 04 |
| `src/skills/loader.ts` | `.skills/` 扫描、解析和 Prompt 构造 | 05 |
| `src/plugins/types.ts` | Plugin 接口定义 | 05 |
| `src/plugins/manager.ts` | 插件加载、命名空间隔离和卸载 | 05 |
| `src/plugins/supabase-plugin.ts` | Supabase Mock 示例插件 | 05、09 |
| `src/channels/types.ts` | 统一消息与通道接口 | 05 |
| `src/channels/gateway.ts` | 多通道会话路由与 `agentLoop` 转发 | 05 |
| `src/channels/feishu.ts` | 飞书长连接与 Hono Dashboard | 05 |
| `src/security/roles.ts` | 三档角色权限控制 | 06 |
| `src/security/hooks.ts` | Pre/Post Hook 管线 | 06、09 |
| `src/security/bash-classifier.ts` | Bash 危险正则分类 | 06、09 |
| `src/cron/types.ts` | 定时任务与日志类型 | 06 |
| `src/cron/parser.ts` | Cron/Interval/Once 时间解析 | 06 |
| `src/cron/store.ts` | `.cron/` 文件存储 | 06 |
| `src/cron/service.ts` | 定时器调度与连续失败熔断 | 06 |
| `src/config/schema.ts` | Zod Schema 与配置类型推导 | 07、09 |
| `src/config/loader.ts` | JSON 配置加载与 `${VAR}` 替换 | 07 |
| `src/config/init.ts` | 交互式初始化向导 | 07、09 |
| `src/commands/*.ts` | 责任链命令分发与 11 个领域命令模块 | 03~07 |
