# 00. 阅读指南与整体架构

## 0.1 项目是什么

SuperAgent 是一个以 TypeScript 编写的本地 AI Agent 教学项目。它不是一个网页聊天壳，而是一个小型 Agent Runtime：模型只是其中一个部件，系统还负责工具执行、上下文组装、知识检索、长期记忆、安全控制、任务调度、消息接入和运行记录。

它的核心价值不在某个模型有多强，而在于展示了一个 Agent 系统的基本骨架：

```text
输入 → 组装上下文 → 调用模型 → 解析事件 → 执行工具
 ↑                                            ↓
 └──────────── 把结果写回消息，再次调用模型 ────┘
```

当模型不再请求工具并给出最终文本时，本轮才结束。

## 0.2 技术栈

| 技术 | 在项目中的作用 |
| --- | --- |
| TypeScript + ESM | 类型约束与模块组织 |
| Node.js 22 | 文件、进程、网络、终端和定时器运行环境 |
| `tsx` | 直接运行 TypeScript，不要求预先构建 |
| Vercel AI SDK | 统一模型调用、流式事件和工具协议 |
| `@ai-sdk/openai` | 连接 OpenAI 兼容协议；本项目用于 DashScope |
| Zod | 配置结构校验和默认值填充 |
| Hono | Dashboard HTTP 服务 |
| 飞书 Node SDK | Bot 长连接与消息收发 |
| Croner | Cron 表达式解析和下次执行时间计算 |
| Turndown | 把网页 HTML 转成 Markdown |

项目没有正式 build 脚本，`pnpm start` 通过 `tsx src/index.ts` 直接执行源码。

## 0.3 顶层目录地图

```text
SuperAgent/
├── src/
│   ├── agent/       主 Agent Loop、重试、循环检测
│   ├── agents/      子 Agent 的记录、限制和执行
│   ├── channels/    消息通道抽象、飞书和 Dashboard
│   ├── commands/    终端斜杠命令
│   ├── config/      Schema、配置加载、初始化向导
│   ├── context/     Prompt、上下文压缩、防御和可视化
│   ├── cron/        调度、解析、存储和执行日志
│   ├── memory/      Markdown 长期记忆与 BM25 搜索
│   ├── plugins/     插件生命周期与 Supabase 示例
│   ├── rag/         切片、Embedding、向量库与混合检索
│   ├── security/    角色权限、Hook、Bash 风险识别
│   ├── session/     对话 JSONL 持久化
│   ├── skills/      项目 Skill 加载器
│   ├── tools/       工具定义、注册表、MCP 和内置工具
│   ├── trace/       每轮执行的结构化轨迹
│   ├── usage/       Token 与费用统计
│   ├── index.ts     进程入口
│   ├── main.ts      系统组装入口
│   └── mock-model.ts 无密钥时的模型模拟器
├── .skills/         项目内 Skill
├── docs/            RAG 演示文档与本学习手册
├── example.env      环境变量模板
└── package.json     依赖和运行脚本
```

## 0.4 静态架构

```text
                           ┌───────────────────┐
 CLI readline ────────────▶│                   │
 飞书 / Dashboard ────────▶│   输入适配层       │
 Cron ────────────────────▶│                   │
                           └─────────┬─────────┘
                                     │ user message
                                     ▼
┌──────────────┐            ┌───────────────────┐
│ Memory Index │───────────▶│   PromptBuilder   │◀──────── Skill
│ RAG Summary  │            └─────────┬─────────┘
└──────────────┘                      │ system prompt
                                     ▼
                           ┌───────────────────┐
                           │    Agent Loop     │◀──── Model
                           └─────────┬─────────┘
                                     │ tool call
                                     ▼
                           ┌───────────────────┐
                           │   Tool Registry   │
                           │ role/hook/lock/   │
                           │ truncate/defer    │
                           └─────────┬─────────┘
                                     │
          ┌───────────┬──────────────┼────────────┬────────────┐
          ▼           ▼              ▼            ▼            ▼
       File/Shell   RAG/Memory     MCP/Plugin    Cron       Sub-Agent

旁路记录：Session / Trace / Usage / Cron Logs
```

## 0.5 `main.ts` 为什么重要

`main.ts` 是 Composition Root，也就是“负责组装，而不是负责具体业务”的位置。它按以下顺序建立系统：

1. 加载 `.env` 和 `super-agent.config.json`；
2. 根据 `model.apiKey` 创建 Mock 或真实模型；
3. 创建 `ToolRegistry`，注册内置工具和 `tool_search`；
4. 初始化 Memory、RAG 和对应工具；
5. 准备 MCP、Skill、Plugin、Hook；
6. 创建 Cron 和 Sub-Agent；
7. 建立 Prompt 管道；
8. 建立 Channel Gateway 并注册飞书通道；
9. 建立命令分发器；
10. `startAgent()` 中连接 MCP、加载插件、启动 Channel 和 Cron；
11. 自动导入 `docs/` 第一层 Markdown；
12. 启动终端 `ask()` 循环。

阅读其他模块时，如果不确定它从哪里被创建、谁把依赖传给它，就回到 `main.ts`。

## 0.6 一条终端消息的完整生命周期

用户在 `You:` 后输入普通文本时：

```text
readline 收到文本
  ↓
Command Dispatcher 尝试识别斜杠命令
  ↓ 不是命令
把 user message 写入内存 messages 和 .sessions/default.jsonl
  ↓
PromptBuilder 重新构造 system prompt
  ↓
LocalTraceRecorder 创建本轮 trace
  ↓
agentLoop 调用 streamText
  ↓
模型返回文本、tool-call、tool-result 等流事件
  ↓
AI SDK 通过 ToolRegistry 暴露的 execute 执行工具
  ↓
模型响应消息追加回 messages
  ↓ 若调用过工具
继续下一 Step，让模型观察结果并决定下一步
  ↓ 若未调用工具
结束；记录 Usage、Trace 和新增 Session 消息
```

这里有两个容易混淆的循环：

- `ask()` 是人与程序的多轮终端交互循环；
- `agentLoop()` 是一次用户请求内部的多步自主执行循环。

## 0.7 三类状态

理解 Agent 项目时，先区分状态的生命周期：

| 状态 | 生命周期 | 当前存放位置 |
| --- | --- | --- |
| 单次 Step | 一次模型调用 | 局部变量、Trace |
| 当前对话 | 当前进程中的多轮消息 | `messages` 数组、`.sessions/` |
| 长期知识 | 跨会话保留 | `.memory/`、项目文档 |
| 检索索引 | 当前进程 | 内存 `VectorStore` |
| 定时任务 | 跨重启配置 + 当前进程定时器 | `.cron/` + `setTimeout` |
| 子 Agent 记录 | 当前进程 | `SubAgentRegistry` 内存 Map |

## 0.8 课程路径与最终源码

课程采用“先最小实现，再逐层演化”的路线；最终仓库已经把多数能力拆成模块。两种视角都需要：

| 课程阶段 | 最终源码 |
| --- | --- |
| 模型聊天和流式输出 | `mock-model.ts`、`agent/loop.ts` |
| 工具调用 | `tools/registry.ts` 和具体工具文件 |
| Context Engineering | `context/`、`session/`、`usage/` |
| Memory + RAG | `memory/`、`rag/` |
| Skills + Plugins + Channel | `skills/`、`plugins/`、`channels/` |
| 权限 + Cron + Multi-Agent | `security/`、`cron/`、`agents/` |
| 配置、CLI、上线 | `config/`、`commands/`、`main.ts` |
| Trace 加餐 | `trace/` |

## 0.9 本章自检

尝试不看上文回答：

1. `index.ts` 和 `main.ts` 的职责有什么不同？
2. 为什么 `main.ts` 适合作为 Composition Root？
3. `ask()` 和 `agentLoop()` 分别循环什么？
4. 为什么 Session、Memory 和 VectorStore 不能视为同一种状态？
5. 一个飞书消息和一个终端消息最终复用了哪一段核心逻辑？

下一章从最小模型调用开始，进入整个系统最核心的 Agent Loop。
