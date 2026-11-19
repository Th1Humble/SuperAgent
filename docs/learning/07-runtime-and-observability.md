# 07. 配置、命令、启动与可观测性

## 7.1 两个入口分支

`src/index.ts` 很短：

```text
argv[2] === "init" → 动态导入 config/init.ts
其他情况          → 动态导入 main.ts 并 startAgent()
```

动态导入很重要：运行初始化向导时不会提前执行 `main.ts` 顶层代码，也就不会加载配置、创建模型和启动服务。

`pnpm run continue` 传入的是 `--continue`，当前仍走普通启动分支，没有专门恢复逻辑。

## 7.2 配置来源与优先关系

运行时先通过 `dotenv/config` 加载 `.env`，再由 `loadConfig()` 读取 `super-agent.config.json`。

配置文件中的字符串可以写：

```json
{"apiKey":"${DASHSCOPE_API_KEY}"}
```

`substituteEnvVars()` 会递归替换对象和数组中的 `${UPPER_CASE_NAME}`。若变量未定义，会保留原字符串并警告。

注意：未定义时保留 `${NAME}` 是非空字符串。对于 `model.apiKey`，这可能让程序误以为存在 Key 并尝试真实模型。README 推荐创建 `.env` 且变量留空，使替换结果成为空字符串并进入 Mock。

## 7.3 Zod Schema

Schema 负责：

- 结构验证；
- 默认值；
- 数值范围；
- 推导 `SuperAgentConfig` 类型。

配置块包括 model、plugins、channels、agents、security、memory、rag、cron、session、usage。

安全原则：Schema 中“有字段”不代表运行时“已使用字段”。判断配置是否生效，要搜索 `config.<field>` 的读取位置。

当前 `main.ts` 实际读取的主要是：

- `config.model`
- `config.memory.dataDir`
- `config.cron.dataDir`
- `config.agents.maxSpawnDepth/maxConcurrent`
- `config.session.id`

插件 enabled、Channel 配置、RAG enabled/docsDir、安全配置、Usage 路径等没有完整接线。

## 7.4 初始化向导

`pnpm run init` 会询问：

- 模型档位；
- DashScope Key；
- 是否启用飞书及凭据；
- 子 Agent 最大并发数。

然后写入 `super-agent.config.json`；如果直接输入 Key 或飞书凭据，还会写 `.env`。

几个边界：

- 覆盖配置前会询问，但写 `.env` 时没有合并已有内容；
- 初始化生成 `security.defaultRole: developer`，与实际 Role 联合类型不一致；
- 飞书字段进入配置，但主程序实际读取 `process.env.FEISHU_*`；
- 推荐让 Key 留空并使用环境变量引用，避免凭据进入 JSON。

## 7.5 Command Dispatcher

命令系统使用责任链：

```ts
for (const handler of handlers) {
  const result = handler(cmd, ctx);
  if (result) return result;
}
return false;
```

返回值含义：

- `false`：不是我的命令，交给下一个；
- `true`：同步处理完成；
- `'async'`：命令自己负责异步流程和再次调用 `ask()`。

`CommandContext` 把 messages、timestamps、Registry、Builder、Tracker、Session、Model 等共享依赖集中传入。

命令和 Tool 的区别：

| Command | Tool |
| --- | --- |
| 用户在终端显式输入 | 模型自主选择 |
| 不进入模型消息 | 定义会暴露给模型 |
| 主要用于控制和诊断 | 主要用于完成任务 |
| 只服务当前 CLI | 可被 CLI/Channel/Cron Agent 复用 |

## 7.6 命令模块

`src/commands/` 按领域拆分：

- debug：模拟长对话、Cache、Context Defense 和状态查看；
- context：上下文矩阵和 Usage；
- memory、rag、dream；
- skill、plugin、channel；
- security：role、hooks；
- cron、agent。

带依赖的命令通过工厂函数创建，例如 `createCronCommands(cronService)`；纯命令导出数组，例如 `contextCommands`。

## 7.7 启动顺序

`main.ts` 的很多对象在模块加载时创建，`startAgent()` 负责有副作用的启动：

```text
连接 Mock MCP
  ↓
加载可用 Plugins
  ↓
启动所有 Channels
  ↓
加载、注入执行器并启动 Cron
  ↓
创建 Session / Usage / readline
  ↓
输出运行信息
  ↓
扫描 docs/*.md 构建 RAG
  ↓
ask()
```

目前 RAG 导入发生在 Channel 和 Cron 启动之后、终端 `ask()` 之前。理论上外部消息可能在知识库尚未构建完成时到达。

## 7.8 正常关闭

用户输入 `exit` 或空字符串时：

1. `cronService.stop()` 清除 Timer；
2. `gateway.stopAll()` 停止 Channel；
3. `pluginManager.unloadAll()` 调用插件清理并注销工具；
4. `rl.close()`。

当前没有显式调用 `registry.closeAllMCP()`，真实 MCP 接入后需要补充；也没有处理 SIGINT/SIGTERM 的统一优雅退出。

## 7.9 Session、Trace、Usage 的区别

| 记录 | 回答的问题 | 格式 |
| --- | --- | --- |
| Session | “对话里说了什么？” | `.sessions/<id>.jsonl` |
| Trace | “这一轮每一步发生了什么？” | `.traces/<traceId>.jsonl` |
| Usage | “用了多少 Token、估算多少钱？” | `.usage/today.jsonl` |

三者不能互换。Session 是面向对话恢复，Trace 是面向执行复盘，Usage 是面向成本统计。

## 7.10 Local Trace

每个普通终端请求创建一个 Trace。事件包括：

```text
trace_started
step_started
step_attempt_failed (可选，多次)
step_completed
...
trace_finished
```

`step_started` 记录当时完整 System 和 Messages；`step_completed` 记录输出消息、文本、耗时和 Usage。

`sanitize()` 会按 key 名匹配 api-key、token、secret、password、authorization 并替换为 `[REDACTED]`。这是基础防护，不保证正文中的秘密、非标准字段名或文件内容被清理。因此 Trace 在分享前仍要人工检查。

如果写文件失败，Recorder 会停止后续写入但不让 Agent 主流程失败，体现“观测失败不阻塞业务”的选择。

## 7.11 Trace Inspect

`pnpm run trace:inspect <file>` 会输出：

- Trace ID；
- 每步上下文消息数量；
- 耗时；
- Token；
- 工具名称；
- 最终状态。

它是摘要查看器，不会重放执行。真正复盘时可直接逐行阅读 JSONL。

## 7.12 可观测性缺口

当前 Trace 只包裹普通终端请求。以下路径没有同等记录：

- ChannelGateway 的 Agent Loop；
- Cron 的 Agent Loop；
- Sub-Agent 自己的模型 Step；
- 启动阶段 RAG 导入；
- Plugin/MCP 的连接耗时。

此外 Session 的时间戳和 `timestamps` Map 是两套来源，重启后 TTL 信息不会恢复。

## 7.13 运行时文件

```text
.sessions/  对话记录
.traces/    每轮执行轨迹
.usage/     Token 与费用
.cron/      定时任务和日志
.memory/    长期记忆
```

这些目录已在 `.gitignore` 中，但可能含用户输入、工具结果、路径和项目内容。Git 忽略不等于安全加密，应设置本地权限、清理策略和分享流程。

## 7.14 故障定位路径

### Agent 没使用某个工具

1. `/role` 看当前角色和活动工具数量，启动日志看注册情况；
2. 检查 Role；
3. 检查 Profile；
4. 看是否为 deferred，需要 tool_search；
5. 查看 Trace 中发给模型的工具调用。

### 回答没有使用知识库

1. `/rag` 看片段数量；
2. 确认文件在 `docs/` 第一层或已 ingest；
3. 直接测试 `rag_search`；
4. 检查 Mock Embedding 的质量边界；
5. 看模型是否决定调用工具。

### 配置看似不生效

1. 看启动日志是否加载配置；
2. 检查环境变量替换；
3. 搜索该字段是否在 `main.ts` 被消费；
4. 区分 config 字段和直接读取 `process.env`；
5. 检查是否被硬编码路径覆盖。

## 7.15 实践

执行一次需要两个工具的任务后：

```text
/context
/usage
/role
/hooks
```

然后同时对照：

```text
.sessions/default.jsonl
.traces/<本轮文件>.jsonl
.usage/today.jsonl
```

尝试写出同一轮请求在三个文件中分别能找到什么、找不到什么。

## 7.16 本章自检

1. 为什么 init 使用动态 import？
2. 配置 Schema 中存在的字段一定生效吗？
3. Command 与 Tool 的调用者分别是谁？
4. Session、Trace、Usage 各自解决什么问题？
5. 当前关闭流程还缺少哪些资源清理？
