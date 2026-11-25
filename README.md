# SuperAgent

一个用 TypeScript 写的本地 AI Agent 教学项目。你在终端里和它对话，它能读写文件、搜文档、执行 Shell 命令，也能定时跑任务或把活拆给子 Agent。

很多 Agent 教程一上来就卡在注册账号、充钱拿 API Key 上。这个仓库默认带了一套 Mock 模型和本地向量占位实现，装完依赖就能先把整条 Agent 运行链路跑通，后面想换真实模型再接也不迟。

如果你想顺着代码把 Agent 的核心机制系统学一遍，推荐配合手册一起看：
- 在线阅读：[完整源码学习手册](https://th1humble.github.io/SuperAgent/learning/)
- 仓库内直接看：[docs/learning/README.md](docs/learning/README.md)

## 先跑起来（不需要 API Key）

环境要求 **Node.js 22** 和 **pnpm 10.33.0**（`package.json` 里的 `packageManager` 字段锁定了版本；如果你本机 pnpm 版本对不上，可以先执行 `corepack enable`）。

```bash
pnpm install
pnpm start
```

> **装依赖太慢？** 项目里的 `.npmrc` 指向官方 `registry.npmjs.org`，且仓库没有提交锁文件，首次安装会由 pnpm 现场解析生成。如果直连官方源较慢，可以在你自己的 `~/.npmrc` 里换镜像源，不用改仓库配置。

首次启动没有 `super-agent.config.json`，会自动进入 Mock 模式。同时会在 [http://localhost:3000](http://localhost:3000) 起一个本地 Channel Dashboard 测试页（如果 3000 端口被占导致启动报错，在 `.env` 里改 `FEISHU_PORT=3001` 即可）。

看到 `You:` 提示符后，可以先试几条能直接触发内置能力的输入：

- `你好` —— 走一遍基础问答
- `读取 package.json` —— 触发 `read_file` 工具，观察模型如何经历两个 Step 才给出回答
- `测试并发` —— 一次并行调用多个只读工具，看工具锁和流式输出
- `/rag` —— 查看启动时自动导入了哪些演示文档
- `/context` —— 查看当前上下文窗口各部分的占用情况
- `exit` —— 停止定时器和通道并退出（建议用 `exit`，当前没有为 `Ctrl+C` 做单独的优雅清理）

打开 [http://localhost:3000](http://localhost:3000) 点「发送」，消息会经过同一个 `agentLoop` 并把执行过程打在终端里。

## 接上真实模型

想看真实大模型的规划和工具选择（比如体验 Cron 创建和 Sub-Agent 并行分治，这两块在 Mock 里没有写死触发词），再接 DashScope：

```bash
cp -n example.env .env
pnpm run init
```

1. 跑 `pnpm run init` 时，**DashScope API Key 那一步直接回车留空**，飞书按需选 `N`。这样生成的 `super-agent.config.json` 里会写入 `"${DASHSCOPE_API_KEY}"` 引用，密钥不会明文写进 JSON。
2. 打开 `.env`，把真实的 `DASHSCOPE_API_KEY=...` 填进去，再执行 `pnpm start`。

注意两点：
- 只填 `.env` 不跑 `init` 没用，程序看的是 `super-agent.config.json` 里的 `model.apiKey` 是否非空；没有那句 `"${DASHSCOPE_API_KEY}"` 引用，默认配置依然是空字符串，还是会走 Mock。
- 如果你在向导里直接填了 Key 或飞书凭据，向导会**整份重写** `.env`，把 `example.env` 里自带的其它环境变量和注释覆盖掉，所以还是建议向导里留空、手动改 `.env`。

| 环境变量 | 用途 |
| --- | --- |
| `DASHSCOPE_API_KEY` | DashScope 对话模型和 RAG Embedding。配置了引用但值留空时走 Mock。 |
| `FEISHU_APP_ID`、`FEISHU_APP_SECRET` | 两个都填才会建立飞书 Bot 长连接；缺一个就只起本地 Dashboard。飞书后台还需要开启机器人能力、订阅消息事件并配好权限。 |
| `FEISHU_PORT` | Dashboard 的 HTTP 端口，默认 `3000`，和飞书长连接无关。 |
| `TAVILY_API_KEY`、`SERPER_API_KEY` | 网页搜索，配一个就行；都填时优先用 Tavily，都不填则无法搜索。 |
| `SUPABASE_URL`、`SUPABASE_KEY` | 预留给 Supabase 插件示例。当前只返回模拟数据，不会连真实数据库。 |

### 安全和隐私提醒

- `.env` 和 `super-agent.config.json` 不要提交到 Git（`.gitignore` 已经忽略）。
- Dashboard 的测试接口没有做鉴权，而 Agent 具备文件和 Shell 工具权限，**千万不要把 3000 端口直接暴露到公网**。
- 接入真实模型后，RAG 导入的文档片段会发送到 DashScope 计算向量，涉密文档不要往里导。

## 目前实现了哪些核心能力

这个项目故意把“演示边界”留得很透明，方便你看清一个 Agent Runtime 的各个部件：

- **工具系统**：内置文件读写编辑、`glob`/`grep`、`bash`、网页搜索抓取，支持角色过滤（`owner`/`collaborator`/`guest`）、Pre/Post Hook、并发/独占锁以及 `tool_search` 延迟发现。内置的 GitHub MCP 是 Mock 客户端，不会连真实 GitHub。
- **上下文工程**：通过 `PromptBuilder` 管道拼接 System Prompt，提供 `/context` 占用矩阵和 `/usage` 费用估算；压缩模块已写好，但默认主循环还没有自动调用。
- **RAG 与长期记忆**：启动时自动把 `docs/` 第一层 `.md` 切片并做混合检索（向量 + BM25 + MMR），索引保存在内存里，重启重新建；长期记忆则以 Markdown 文件持久化在 `.memory/`，支持 BM25 搜索、健康检查（`/lint`）和整理（`/dream`）。
- **调度、子 Agent 与通道**：支持基于 `croner` 的定时任务（依赖当前进程常驻）、受并发/深度限制的 Sub-Agent 派发、`.skills/` 技能加载、进程内插件热插拔，以及飞书/Dashboard 通道。
- **执行记录**：终端对话会追加到 `.sessions/default.jsonl`，每轮执行轨迹写进 `.traces/`。注意当前**只写不读**，`pnpm run continue` 还没有接上历史恢复，重启后不会接着上次聊。

## 常用交互命令

启动后在 `You:` 后直接输入：

| 命令 | 作用 |
| --- | --- |
| `/rag`、`ingest docs/api-design.md` | 查看知识库片段数，或手动导入指定文档 |
| `/memory`、`/memory search <关键词>`、`/lint` | 列出记忆、BM25 搜记忆、检查记忆条目健康度 |
| `/dream` | 让模型结合 `lint` 报告整理、合并或删除过期记忆 |
| `/context`、`/usage` | 查看当前上下文占用矩阵、Token 与缓存命中统计 |
| `/agents`、`/cron`、`/cron logs` | 查看子 Agent 记录、定时任务和近期执行日志 |
| `/skill list`、`/skill load <名>`、`/skill unload <名>` | 管理 `.skills/` 下的技能；也可直接 `/<skill-name>` 触发 |
| `/plugin`、`/channel` | 查看已加载插件和已注册通道 |
| `/role [owner\|collaborator\|guest]`、`/hooks` | 查看或切换角色权限，查看注册的 Hook |
| `/cache on`、`/cache off` | 开关 Mock 模型的 Prompt Cache 模拟 |
| `sim`、`defend`、`status` | 调试命令：注入模拟长对话、手动执行上下文防线、看状态摘要 |
| `exit` | 停止服务并退出 |

想复盘某一轮到底给模型发了什么、调了什么工具，可以在另一个终端跑（轨迹文件路径会在每轮结束的 `[Trace]` 行打印出来）：

```bash
pnpm run trace:inspect .traces/<轨迹文件名>.jsonl
```

## 本地开发与预览文档

项目没有构建步骤，`pnpm start` 直接用 `tsx` 跑 TypeScript 源码：

```bash
pnpm exec tsx --test src/context/defense.spec.ts   # 跑现有的单元测试
pnpm exec tsc --noEmit                             # TypeScript 类型检查
```

如果在本地看 `docs/learning/` 的 Docsify 页面，直接双击 HTML 会被浏览器跨域拦截，需要在仓库根目录起一个静态服务：

```bash
python3 -m http.server 8000
```

然后浏览器访问 [http://localhost:8000/docs/learning/](http://localhost:8000/docs/learning/)。

## 目录结构

```text
src/
  index.ts             CLI 入口（处理 init 和正常启动）
  main.ts              组装模型、工具、记忆、RAG、通道和终端交互循环
  mock-model.ts        无 API Key 时的本地模拟模型（关键词触发工具和流式事件）
  agent/               主 Agent Loop、指数退避重试、死循环检测
  agents/              Sub-Agent 注册表与并行派发
  tools/               内置工具、ToolRegistry（权限/锁/Hook/截断）、MCP 客户端
  context/             Prompt 管道、上下文压缩、三层防线、/context 可视化
  session/             对话历史 JSONL 持久化
  usage/               Token 规范化、缓存统计与费用估算
  memory/              .memory/ Markdown 长期记忆、BM25 检索与健康检查
  rag/                 文档切片、Embedding、内存 VectorStore 与混合检索
  skills/              .skills/ 技能加载器
  plugins/             插件生命周期管理与 Supabase 示例插件
  channels/            消息通道抽象、Gateway、飞书与 Dashboard
  security/            RBAC 角色权限、Hook Pipeline、Bash 风险分级
  cron/                定时表达式解析、任务存储与调度器
  commands/            终端命令分发器和各模块斜杠命令
  config/              Zod 配置校验、环境变量替换、init 向导
  trace/               每轮执行轨迹录制（JSONL）和命令行查看器
docs/
  api-design.md        RAG 演示语料（假业务文档，非本项目真实接口）
  deployment-guide.md  RAG 演示语料（假部署文档，非本项目真实部署）
  learning/            从零理解 Agent 的完整源码学习手册
.skills/               自带技能示例（code-review）
example.env            环境变量模板
```

## License

ISC
