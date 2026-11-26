# SuperAgent

一个用 TypeScript 写的本地 AI Agent 教学项目。代码不到 7,000 行，把 **Agent Loop、工具治理、上下文工程、RAG、长期记忆、安全边界、定时调度与子 Agent** 拆成一眼能看懂的独立模块。

默认内置 Mock 模型和本地向量占位实现，**不需要申请 API Key，装完依赖就能跑通完整链路**。

> **配套教程**：[在线阅读源码学习手册](https://th1humble.github.io/SuperAgent/learning/) · [仓库 Markdown 版](docs/learning/README.md) · [18 组动手实验](docs/learning/08-hands-on-labs.md)

![SuperAgent 整体运行架构](docs/learning/assets/00-architecture.svg)

## 30 秒跑起来（免 API Key）

需要 **Node.js 22** 与 **pnpm 10.33.0**（版本不符可先 `corepack enable`）：

```bash
pnpm install
pnpm start
```

启动后自动进入 Mock 模式，同时在 [http://localhost:3000](http://localhost:3000) 起一个本地 Dashboard 测试页。看到 `You:` 后可以直接试：

- `读取 package.json` —— 触发 `read_file`，观察工具调用如何走两个 Step
- `测试并发` —— 并行触发多个只读工具，观察并发锁与流式输出
- `/context` —— 查看 16×16 上下文窗口占用矩阵
- `/rag` —— 查看启动时自动索引的演示文档（退出输入 `exit`）

> 首次安装慢可在 `~/.npmrc` 换镜像源；3000 端口被占可在 `.env` 设置 `FEISHU_PORT=3001`。完整命令与 Mock 触发词见[速查附录](docs/learning/appendix-reference.md)。

## 接上真实模型

Mock 模式靠关键词模拟工具调用，不支持 Cron 创建和 Sub-Agent 并行分治。想体验完整能力，接入 DashScope（通义千问）即可：

```bash
cp -n example.env .env
pnpm run init
```

1. 运行向导时 **API Key 留空直接回车**，生成带 `"${DASHSCOPE_API_KEY}"` 引用的 `super-agent.config.json`（避免密钥落进 JSON，也不会覆盖已有 `.env`）；
2. 在 `.env` 中填入 `DASHSCOPE_API_KEY`，再 `pnpm start`。

其余可选环境变量（飞书 Bot、Tavily/Serper 搜索等）直接看 [example.env](example.env)。

## 配套学习手册

仓库自带 10 章源码导读 + 18 组动手实验（位于 [docs/learning/](docs/learning/README.md)）：

- **核心链路（00–02 章）**：整体架构、`agentLoop` 闭环、重试退避、死循环检测、`ToolRegistry` 读写锁与延迟发现
- **状态与知识（03–04 章）**：`PromptBuilder` 管道、Prompt Cache、三层上下文防线、混合检索（向量 + BM25 + MMR）与 Markdown 长期记忆
- **扩展与治理（05–07 章）**：Skill / Plugin / MCP / 飞书通道、RBAC 与 Hook、Cron 调度、Sub-Agent、Trace 轨迹排查
- **实践与边界（08–09 章）**：18 个渐进实验，以及如实列出的 Mock 边界、未接线配置和源码已知缺陷

本地浏览器预览 Docsify 手册（在仓库根目录执行后访问 [http://localhost:8000/docs/learning/](http://localhost:8000/docs/learning/)）：

```bash
python3 -m http.server 8000
```

## 开发与注意

```bash
pnpm exec tsx --test src/context/defense.spec.ts   # 运行单元测试
pnpm exec tsc --noEmit                             # TypeScript 类型检查
pnpm run trace:inspect .traces/<轨迹文件>.jsonl     # 查看某轮执行轨迹摘要
```

- **仅作教学**：Dashboard 无鉴权、文件与 Shell 工具无容器沙箱，**不要把服务暴露到公网**，也不要把涉密文档导入真实 Embedding；
- `.env`、`super-agent.config.json` 以及 `.traces/` 等运行目录均已在 `.gitignore` 中忽略，分享日志前请先检查敏感内容。

## License

ISC
