# 附录：命令、工具、配置与源码索引

## A.1 package scripts

| 命令 | 实际行为 |
| --- | --- |
| `pnpm start` | `tsx src/index.ts` |
| `pnpm run init` | `tsx src/index.ts init` |
| `pnpm run continue` | `tsx src/index.ts --continue`，当前等同普通启动 |
| `pnpm run trace:inspect <file>` | 打印 Trace 摘要 |

## A.2 交互命令

| 命令 | 作用 |
| --- | --- |
| `/context` 或 `context` | 显示上下文矩阵 |
| `/usage` 或 `usage` | 显示 Token、Cache 和费用 |
| `/memory` | 列出记忆 |
| `/memory search <query>` | BM25 搜索记忆 |
| `/lint` | 检查记忆健康度 |
| `/dream` | 让模型整理记忆 |
| `/rag` | 显示 RAG 片段数量 |
| `ingest <path>` | 手动导入 Markdown |
| `/skill list` | 列出 Skill |
| `/skill load <name>` | 激活 Skill |
| `/skill unload <name>` | 停用 Skill |
| `/<skill-name>` | Skill 命令快捷提示/处理 |
| `/plugin` | 列出已加载插件 |
| `/plugin load <name>` | 加载可用插件 |
| `/plugin unload <name>` | 卸载插件 |
| `/channel` | 列出 Channel |
| `/role` | 查看当前角色 |
| `/role owner|collaborator|guest` | 切换角色 |
| `/hooks` | 列出 Pre/Post Hook |
| `/cron` | 列出定时任务 |
| `/cron logs` | 查看近期执行记录 |
| `/agents` | 查看子 Agent 记录 |
| `/cache on`、`/cache off` | 切换 Mock Cache 模拟 |
| `sim` 或 `模拟长对话` | 生成调试用长对话 |
| `defend` 或 `执行防线` | 手动应用 Context Defense |
| `status` 或 `查状态` | 查看调试状态 |
| `exit` | 停止服务并退出 |

Cron 的更多子命令以 `src/commands/cron.ts` 为准；模型也可以通过 `cron_manage` 工具管理任务。

## A.3 内置工具

| 工具 | 类型 | 说明 |
| --- | --- | --- |
| `get_weather` | 只读/并发 | 本地模拟天气 |
| `calculator` | 只读/并发 | 表达式计算，教学实现使用动态函数 |
| `read_file` | 只读/并发 | 分段读取文件 |
| `write_file` | 写/独占 | 覆盖写文件 |
| `edit_file` | 写/独占 | 精确文本替换 |
| `list_directory` | 只读/并发 | 列目录 |
| `glob` | 只读/并发 | 简化 Glob 搜索 |
| `grep` | 只读/并发 | 正则内容搜索 |
| `bash` | 写/独占 | 执行 Shell 命令 |
| `web_search` | 只读/并发 | Tavily 或 Serper 搜索 |
| `web_fetch` | 只读/并发 | 网页转 Markdown |
| `tool_search` | 只读/并发 | 发现延迟工具 |
| `memory` | 状态工具 | 记忆增删读搜和检查 |
| `rag_ingest` | 状态工具 | 文档切片并加入内存索引 |
| `rag_search` | 只读 | 混合检索 |
| `cron_manage` | 状态工具 | 管理定时任务 |
| `spawn_agent` | 调度工具 | 派发子 Agent |

默认还会注册 Mock MCP GitHub 工具：

- `mcp__github__list_issues`
- `mcp__github__search_repositories`
- `mcp__github__get_file_contents`

Supabase 插件工具名称以 `supabase__` 开头，当前行为以 Mock/占位为主。

## A.4 环境变量

| 变量 | 用途 | 缺失行为 |
| --- | --- | --- |
| `DASHSCOPE_API_KEY` | 对话模型和 Embedding | 配置替换为空时使用 Mock |
| `FEISHU_APP_ID` | 飞书应用 ID | 与 Secret 任一缺失则不启动 Bot 长连接 |
| `FEISHU_APP_SECRET` | 飞书应用密钥 | 同上 |
| `FEISHU_PORT` | Dashboard 端口 | 默认 3000 |
| `TAVILY_API_KEY` | Tavily 搜索 | 优先于 Serper |
| `SERPER_API_KEY` | Serper 搜索 | Tavily 缺失时使用 |
| `SUPABASE_URL` | 插件配置 | 当前仍主要 Mock |
| `SUPABASE_KEY` | 插件配置 | 当前仍主要 Mock |

## A.5 配置结构

```text
version
model
  provider / name / baseURL / apiKey
plugins[]
  name / enabled / config
channels.feishu
  enabled / appId / appSecret / port
agents
  maxSpawnDepth / maxConcurrent / defaultTimeout
security
  defaultRole / auditLog / bashTimestamp
memory
  dataDir
rag
  enabled / docsDir
cron
  enabled / dataDir
session
  id
usage
  trackingFile
```

当前只有部分字段被 `main.ts` 消费，详见第 07、09 章。

## A.6 运行时数据

| 路径 | 内容 | 是否持久 | 是否可能敏感 |
| --- | --- | --- | --- |
| `.sessions/*.jsonl` | CLI 对话消息 | 是 | 是 |
| `.traces/*.jsonl` | System、消息、输出、Usage | 是 | 高 |
| `.usage/*.jsonl` | Token 和费用 | 是 | 中 |
| `.cron/jobs.json` | 任务定义 | 是 | 是 |
| `.cron/logs.jsonl` | 任务输出 | 是 | 是 |
| `.memory/*.md` | 长期记忆 | 是 | 高 |
| VectorStore | RAG 向量和片段 | 否，内存 | 是 |
| Channel sessions | 外部对话历史 | 否，内存 | 是 |
| SubAgentRegistry | 子任务记录 | 否，内存 | 是 |

## A.7 全源码索引

### 入口与模型

| 文件 | 职责 | 对应章节 |
| --- | --- | --- |
| `src/index.ts` | init/正常启动分支 | 01、07 |
| `src/main.ts` | 依赖组装、生命周期、CLI | 00、07 |
| `src/mock-model.ts` | 模拟模型、工具事件、Usage 和 Cache | 01 |

### Agent

| 文件 | 职责 | 对应章节 |
| --- | --- | --- |
| `src/agent/loop.ts` | 主 Agent 多步循环 | 01 |
| `src/agent/retry.ts` | 可重试判断和指数退避 | 01 |
| `src/agent/loop-detection.ts` | 重复、乒乓和全局熔断 | 01、09 |

### Sub-Agent

| 文件 | 职责 | 对应章节 |
| --- | --- | --- |
| `src/agents/types.ts` | 配置、请求和运行记录类型 | 06 |
| `src/agents/registry.ts` | 子 Agent 状态与限制 | 06 |
| `src/agents/spawn.ts` | 单个/并行子 Agent 执行 | 06 |

### Tool System

| 文件 | 职责 | 对应章节 |
| --- | --- | --- |
| `src/tools/registry.ts` | 注册、过滤、锁、Hook、执行适配 | 02、06 |
| `src/tools/index.ts` | 内置工具汇总 | 02 |
| `src/tools/file-tools.ts` | 文件读写编辑与列目录 | 02 |
| `src/tools/search-tools.ts` | Glob 和 Grep | 02 |
| `src/tools/shell-tools.ts` | Shell 执行 | 02、06 |
| `src/tools/web-search.ts` | 搜索和网页抓取 | 02 |
| `src/tools/utility-tools.ts` | 天气和计算器示例 | 02 |
| `src/tools/tool-search.ts` | 延迟工具发现 | 02 |
| `src/tools/mcp-client.ts` | 真实/Mock MCP Client | 02、05 |
| `src/tools/memory-tools.ts` | Memory 工具适配 | 04 |
| `src/tools/rag-tools.ts` | RAG ingest/search 工具 | 04 |
| `src/tools/cron-tools.ts` | Cron 管理工具 | 06 |
| `src/tools/spawn-tools.ts` | 子 Agent 工具 | 06 |

### Context、Session、Usage

| 文件 | 职责 | 对应章节 |
| --- | --- | --- |
| `src/context/prompt-builder.ts` | Prompt 管道和内置段落 | 03 |
| `src/context/prompt-pipes.ts` | Memory/RAG Prompt Pipe | 03、04 |
| `src/context/compressor.ts` | Microcompact 和 LLM 摘要 | 03、09 |
| `src/context/defense.ts` | TokenTracker、截断、TTL | 03 |
| `src/context/defense.spec.ts` | 当前已有测试 | 03、09 |
| `src/context/tool-result-output.ts` | 结构化工具结果文本转换 | 03 |
| `src/context/view.ts` | Context/Usage 终端视图 | 03 |
| `src/session/store.ts` | Session JSONL | 03、07 |
| `src/usage/tracker.ts` | Usage 规范化、费用和 Cache | 01、03、07 |

### RAG

| 文件 | 职责 | 对应章节 |
| --- | --- | --- |
| `src/rag/chunker.ts` | Markdown 切片 | 04 |
| `src/rag/embedder.ts` | Mock/DashScope Embedding 和缓存 | 04 |
| `src/rag/store.ts` | 内存向量存储 | 04、09 |
| `src/rag/search.ts` | 向量 + BM25-like + MMR | 04 |

### Memory

| 文件 | 职责 | 对应章节 |
| --- | --- | --- |
| `src/memory/store.ts` | Markdown 条目、索引和生命周期 | 04 |
| `src/memory/search.ts` | 中英文 BM25 | 04 |
| `src/memory/validator.ts` | 条目和路径健康检查 | 04 |

### Skills、Plugins、Channels

| 文件 | 职责 | 对应章节 |
| --- | --- | --- |
| `src/skills/loader.ts` | Skill 扫描、解析和 Prompt | 05 |
| `src/plugins/types.ts` | PluginDefinition 与 API | 05 |
| `src/plugins/manager.ts` | 激活、配置、命名空间、卸载 | 05 |
| `src/plugins/supabase-plugin.ts` | Supabase Mock 示例 | 05、09 |
| `src/channels/types.ts` | 收发消息与 Channel 接口 | 05 |
| `src/channels/gateway.ts` | Channel 会话和 Agent 转发 | 05 |
| `src/channels/feishu.ts` | 飞书长连接和 Dashboard | 05 |

### Security

| 文件 | 职责 | 对应章节 |
| --- | --- | --- |
| `src/security/roles.ts` | 角色工具权限 | 06 |
| `src/security/hooks.ts` | Pre/Post Hook 管线 | 06 |
| `src/security/bash-classifier.ts` | Shell 风险正则 | 06、09 |

### Cron

| 文件 | 职责 | 对应章节 |
| --- | --- | --- |
| `src/cron/types.ts` | 任务、Payload、日志类型 | 06 |
| `src/cron/parser.ts` | Cron/Interval/Once 解析 | 06 |
| `src/cron/store.ts` | Jobs 与 Logs 文件存储 | 06 |
| `src/cron/service.ts` | 调度、执行、失败禁用 | 06 |

### Config

| 文件 | 职责 | 对应章节 |
| --- | --- | --- |
| `src/config/schema.ts` | Zod Schema 和配置类型 | 07、09 |
| `src/config/loader.ts` | JSON 加载和环境替换 | 07 |
| `src/config/init.ts` | 交互式初始化向导 | 07 |

### Commands

| 文件 | 职责 | 对应章节 |
| --- | --- | --- |
| `src/commands/index.ts` | Context、Handler 和 Dispatcher | 07 |
| `src/commands/debug.ts` | Cache/Defense/状态调试 | 03、07 |
| `src/commands/context.ts` | Context 和 Usage | 03、07 |
| `src/commands/memory.ts` | Memory 列表、搜索和检查 | 04 |
| `src/commands/rag.ts` | RAG 状态和 ingest | 04 |
| `src/commands/dream.ts` | 记忆整理 | 04 |
| `src/commands/skill.ts` | Skill 管理 | 05 |
| `src/commands/plugin.ts` | Plugin 管理 | 05 |
| `src/commands/channel.ts` | Channel 列表 | 05 |
| `src/commands/security.ts` | Role 和 Hook | 06 |
| `src/commands/cron.ts` | Cron 管理与日志 | 06 |
| `src/commands/agent.ts` | 子 Agent 记录 | 06 |

### Trace

| 文件 | 职责 | 对应章节 |
| --- | --- | --- |
| `src/trace/recorder.ts` | JSONL Trace 与摘要读取 | 07 |
| `src/trace/inspect.ts` | Trace CLI 入口 | 07 |

## A.8 仓库其他文件

| 文件 | 说明 |
| --- | --- |
| `README.md` | 项目快速上手、能力和注意事项 |
| `package.json` | 依赖、脚本、Node 运行方式 |
| `tsconfig.json` | ES2022、Bundler resolution、strict |
| `example.env` | 环境变量模板 |
| `.npmrc` | npm registry |
| `.gitignore` | 依赖、配置和运行时数据忽略规则 |
| `.skills/code-review/SKILL.md` | 内置 Code Review Skill 示例 |
| `docs/api-design.md` | RAG 示例知识，不是本项目 API |
| `docs/deployment-guide.md` | RAG 示例知识，不是本项目部署事实 |

## A.9 分享前检查清单

- 删除或遮盖 `.env` 和配置中的真实密钥；
- 不提交 `super-agent.config.json`；
- 检查 Session、Trace、Memory、Cron Logs 是否含敏感信息；
- 明确 Mock 与真实能力；
- 不把 RAG 示例文档描述为项目实现；
- 标出当前代码版本或提交；
- 引用原课程和仓库；
- 对价格、模型名称和外部 API 行为注明时间敏感性。
