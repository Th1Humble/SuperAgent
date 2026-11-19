# SuperAgent

一个用 TypeScript 写的本地 AI Agent。你在终端里跟它聊天，它可以读写文件、搜项目文档、跑 Shell 命令，也能把活儿分给子 Agent。

不想先去申请 API Key？没关系。项目自带 Mock 模型和 Mock 向量服务，装完就能跑一遍完整流程，之后再接真实模型也不迟。

## 先跑起来

需要 Node.js 22 和 pnpm 10.33.0（`package.json` 里已经锁定）。在项目根目录执行：

```bash
pnpm install
pnpm start
```

第一次启动时没有 `super-agent.config.json`，会直接用 Mock 模型。看到 `You:` 提示符后可以试试：

- 输入 `你好`，随便聊一句
- 输入 `/rag`，看看自动导入了哪些文档
- 输入 `exit`，退出

同时会在 [http://localhost:3000](http://localhost:3000) 起一个 Channel Dashboard。点页面上的「发送」可以测试消息通道，回复会打印在终端里。端口被占了就在 `.env` 里改 `FEISHU_PORT`。

> **装依赖很慢？** 项目的 `.npmrc` 把下载源指向了官方的 `registry.npmjs.org`。仓库里没有提交锁文件，第一次安装时 pnpm 会自己生成。网络访问官方源慢的话，可以在自己的 `~/.npmrc` 里换成镜像，不用改项目里的配置。

## 接上真实模型

```bash
cp -n example.env .env
pnpm run init
```

1. 在初始化向导里选模型，**API Key 留空**，飞书选 `N`。这样生成的 `super-agent.config.json` 里写的是 `${DASHSCOPE_API_KEY}` 这个引用，密钥不会明文落进 JSON。
2. 在 `.env` 里填好 `DASHSCOPE_API_KEY`，再 `pnpm start`。

注意：光填 `.env` 不够，必须有 `model.apiKey` 这一项（也就是上一步生成的引用），否则还是 Mock。`cp -n` 不会覆盖你已有的 `.env`。

| 环境变量 | 干什么用 |
| --- | --- |
| `DASHSCOPE_API_KEY` | DashScope 对话模型和 RAG 向量化。引用了但留空时走 Mock。 |
| `FEISHU_APP_ID`、`FEISHU_APP_SECRET` | 两个都填才会建立飞书 Bot 长连接，缺一个就只启动 Dashboard。飞书后台还要开机器人能力、订阅消息接收事件、配好权限。 |
| `FEISHU_PORT` | Dashboard 的 HTTP 端口，默认 `3000`，跟飞书长连接无关。 |
| `TAVILY_API_KEY`、`SERPER_API_KEY` | 网页搜索，二选一。都填时用 Tavily，都不填就不能搜。 |
| `SUPABASE_URL`、`SUPABASE_KEY` | 预留给 Supabase 插件。目前只返回模拟数据，不会连真实数据库。 |

### 几件要当心的事

- `.env` 别提交。如果你在向导里直接输入了 API Key，它会被写进 `super-agent.config.json`，那个文件同样不要提交（仓库的 `.gitignore` 已经忽略了）。
- Dashboard 的测试入口没有鉴权，而 Agent 能调文件和 Shell 工具。别把这个端口暴露给不可信的网络。
- 用真实模型时，导入知识库的文档片段会发给 DashScope 做向量化。不能外传的内容就别导入。

## 它能做什么

- **工具调用**：文件读写、文件和内容搜索、Shell、网页搜索。工具可以按角色过滤，也能挂 Hook 记录操作。
- **知识库和记忆**：启动时自动索引 `docs/` 下一级的 Markdown，也可以手动导入。向量索引只放在内存里，长期记忆存在 `.memory/`。
- **子 Agent 和定时任务**：可以把任务交给子 Agent，并发数和嵌套深度可配。定时任务和执行记录放在 `.cron/`，调度靠当前进程一直开着。
- **技能、插件和消息通道**：从 `.skills/` 加载技能，插件可以运行时加载和卸载，还能通过飞书收消息。内置的 GitHub MCP 是 Mock 客户端，不会碰真实仓库。
- **运行记录**：终端对话追加到 `.sessions/default.jsonl`，每轮对话的执行轨迹写进 `.traces/`。要把日志发给别人之前，先翻一遍有没有敏感内容。

## 常用命令

下面这些都在 `pnpm start` 之后的 `You:` 提示符里输入。

| 命令 | 作用 |
| --- | --- |
| `/rag`、`ingest docs/api-design.md` | 看知识库里有什么，或手动导入一个文档 |
| `/memory`、`/memory search <关键词>` | 查看、搜索长期记忆 |
| `/dream` | 让 Agent 整理一遍记忆库：删掉过期的，合并重复的 |
| `/context`、`/usage` | 看上下文占用和 Token 用量 |
| `/agents`、`/cron`、`/cron logs` | 看子 Agent、定时任务和最近的执行记录 |
| `/skill list`、`/skill load <名称>`、`/skill unload <名称>` | 列出、启用、停用技能 |
| `/plugin`、`/channel` | 看插件和已注册的消息通道 |
| `/role [owner\|collaborator\|guest]`、`/hooks` | 看或切换角色，看已注册的 Hook |
| `/cache on`、`/cache off` | 开关 Mock 模型的 cache 模拟 |
| `exit` | 退出 |

想看某次执行的轨迹，在另一个 Shell 里跑 `pnpm run trace:inspect <轨迹文件路径>`，路径在终端的 `[Trace]` 提示里。

会话记录现在只写不读：`pnpm run continue` 还没接上历史恢复，别指望它能接着上次聊。

## 开发相关

### 完整源码学习手册

如果你希望沿着课程的演进路线系统理解最终仓库，可以从 [SuperAgent 完整源码学习手册](docs/learning/README.md) 开始。手册覆盖 Agent Loop、工具系统、上下文工程、Memory/RAG、扩展机制、安全、Cron、Multi-Agent、可观测性，并提供逐步实践和全源码索引。

```bash
pnpm exec tsx --test src/context/defense.spec.ts   # 跑现有的测试
pnpm exec tsc --noEmit                             # 类型检查
```

项目没有 build 步骤，`pnpm start` 直接用 `tsx` 跑源码。

## 项目结构

```text
src/
  index.ts             CLI 入口，分发 init 和启动
  main.ts              组装模型、工具、通道和交互循环
  agent/               主 Agent 循环、重试、循环检测
  agents/              子 Agent 调度和执行记录
  tools/               内置工具和工具注册表
  context/             Prompt 构建、上下文压缩与统计
  memory/、rag/        长期记忆、文档检索
  channels/、cron/     消息通道（含 Dashboard）、定时任务
  config/、commands/   配置加载、交互命令
  plugins/、skills/    插件、技能加载
  security/            角色权限和 Hook
  trace/               执行轨迹记录与查看
docs/                  检索演示用的示例文档
.skills/               项目自带的技能
example.env            环境变量模板
```

## License

ISC。
