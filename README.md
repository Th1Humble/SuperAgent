# SuperAgent

SuperAgent 是一个用 TypeScript 编写的本地 AI Agent。你可以在终端和它对话，让它调用工具处理文件、检索项目文档，或把任务交给子 Agent。项目自带 Mock 模型和向量服务，不需要 API Key 就能体验基本流程。

## 快速开始

建议使用 Node.js 22 和项目指定的 pnpm 10.33.0。在项目根目录执行：

```bash
pnpm install --frozen-lockfile
pnpm start
```

首次启动且没有 `super-agent.config.json` 时，默认使用 Mock 模型。在 `You:` 提示符下，可以输入 `你好` 体验对话、输入 `/rag` 查看自动导入的文档，或输入 `exit` 退出。

启动后也可以打开 [Channel Dashboard](http://localhost:3000)，点击「发送」测试消息通道，回复会打印在终端。若端口已被占用，可在 `.env` 中修改 `FEISHU_PORT`。

> 安装疑难：当前 `pnpm-lock.yaml` 中部分包的下载地址指向 `registry.m.jd.com`。如果安装在下载阶段失败，请先确认当前网络能访问该地址。

## 配置真实模型和外部服务

```bash
cp -n example.env .env
pnpm run init
```

1. 在初始化向导中选择模型，将 API Key 留空，飞书选 `N`。这样生成的 `super-agent.config.json` 会引用 `${DASHSCOPE_API_KEY}`，不会把密钥直接写入 JSON。
2. 在 `.env` 中填写 `DASHSCOPE_API_KEY`，按需配置其余变量，然后运行 `pnpm start`。仅填写 `.env` 而不配置 `model.apiKey`，不会切换到真实模型。

所有变量及示例见 [example.env](example.env)。已有 `.env` 时，`cp -n` 不会覆盖它。

| 环境变量 | 用途 |
| --- | --- |
| `DASHSCOPE_API_KEY` | DashScope 对话模型和 RAG 向量化服务。使用上述配置且留空时走 Mock。 |
| `FEISHU_APP_ID`、`FEISHU_APP_SECRET` | 两者都填写才会尝试建立飞书 Bot 长连接。还需在飞书侧为应用开启机器人能力、订阅消息接收事件，并配置相应权限。 |
| `FEISHU_PORT` | Dashboard 的 HTTP 端口，默认 `3000`。不影响飞书长连接。 |
| `TAVILY_API_KEY`、`SERPER_API_KEY` | 网页搜索二选一；都填时优先 Tavily，都不填则网页搜索不可用。 |
| `SUPABASE_URL`、`SUPABASE_KEY` | Supabase 插件的预留配置。当前仅返回模拟数据或操作占位结果，不连接真实数据库。 |

> `.env` 不应提交到 Git。Dashboard 的测试消息入口没有鉴权，而 Agent 可以调用文件和 Shell 工具，请勿将其暴露给不可信网络。启用真实模型后，导入知识库的文档片段会发送到 DashScope 做向量化，请勿导入不应外传的内容。

## 主要功能

- **工具调用**：内置文件读写、文件和内容搜索、Shell、网页搜索等工具。工具注册表支持按角色过滤，并可通过 Hook 记录操作。
- **知识库和记忆**：启动时自动索引 `docs/` 下的一级 Markdown 文件，也可手动导入文档。向量索引保存在内存中，长期记忆默认存放在 `.memory/`。
- **子 Agent 与定时任务**：支持把任务交给子 Agent，并配置最大并发数和嵌套深度。定时任务及执行记录默认存放在 `.cron/`，调度依赖当前进程持续运行。
- **Skills、插件与消息通道**：从 `.skills/` 加载技能，支持运行时加载和卸载插件，并可通过飞书接收消息。内置 GitHub MCP 使用 Mock 客户端，不会访问真实仓库。
- **运行记录**：终端对话会追加到 `.sessions/default.jsonl`，普通对话轮次的执行轨迹写入 `.traces/`。分享日志前请检查其中是否包含敏感内容。

## 常用命令

以下命令在 `pnpm start` 后的 `You:` 提示符输入。

| 命令 | 用途 |
| --- | --- |
| `/rag`、`ingest docs/api-design.md` | 查看知识库，或手动导入文档。 |
| `/memory`、`/memory search <关键词>` | 查看和搜索长期记忆。 |
| `/context`、`/usage` | 查看上下文用量及 Token 使用统计。 |
| `/agents`、`/cron`、`/cron logs` | 查看子 Agent、定时任务及近期执行记录。 |
| `/skill list`、`/skill load <名称>` | 列出或激活技能。 |
| `/plugin`、`/channel` | 查看插件和已注册的消息通道。 |
| `/role [owner\|collaborator\|guest]`、`/hooks` | 查看或切换工具访问角色，以及查看已注册的 Hook。 |
| `exit` | 退出 Agent。 |

查看某次执行轨迹时，在 Shell 中运行 `pnpm run trace:inspect <轨迹文件路径>`，文件路径可从终端的 `[Trace]` 提示获取。当前版本会保存会话记录，但 `pnpm run continue` 尚未接入历史恢复逻辑。

## 项目结构

```text
src/
  index.ts             CLI 入口和初始化命令分发
  main.ts              组装模型、工具、通道和交互循环
  agent/               主 Agent 循环、重试与循环检测
  agents/              子 Agent 调度和执行记录
  tools/               内置工具和工具注册表
  context/             Prompt 构建、上下文压缩与统计
  memory/、rag/        长期记忆与文档检索
  channels/、cron/     消息通道与定时任务
  config/、commands/   配置加载与交互命令
  plugins/、skills/    插件与技能加载
  trace/               执行轨迹记录与查看
docs/                  用于检索演示的示例文档
.skills/               项目内置技能
example.env            环境变量模板
```
