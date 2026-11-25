# 05. 扩展机制与消息通道：Skill、Plugin、MCP 和 Channel

写完主循环和工具系统后，很快就会遇到下一个问题：别人想给这个 Agent 加新能力、新规范，或者不想在黑框终端里用，想接到飞书、网页上怎么办？

代码里一口气出现四种扩展方式：**Skill、Plugin、MCP、Channel**。初学者很容易把前三个搞混，其实看它们“改了系统的哪一层”，一下就分清楚了：

| 机制 | 改的是哪一层 | 会注册新工具吗 | 改变了消息入口吗 | 一句话概括 |
| --- | --- | --- | --- | --- |
| **Skill** | System Prompt（提示词层） | 不会 | 否 | 教模型“按什么套路、什么清单做事” |
| **Plugin** | 进程内工具与生命周期 | 会 | 否 | 给当前 Node 进程热插拔一组本地工具 |
| **MCP** | 跨进程标准工具协议 | 会 | 否 | 连外部独立进程借工具（如 GitHub、数据库） |
| **Channel** | 输入输出渠道 | 不会 | 是 | 把飞书、网页、Slack 的消息送进同一套 `agentLoop` |

## 5.1 Skill：只靠 Markdown 就能给模型“装技能”

看项目自带的 `.skills/code-review/SKILL.md` 和加载器 `src/skills/loader.ts`。

Skill 本质上就是一段约定格式的指令文档，放在 `.skills/<技能名>/SKILL.md`：

```yaml
---
name: code-review
description: 以高级工程师视角审查项目代码
when_to_use: 当用户要求审查代码时
---
（下面写具体审查步骤、检查清单、输出模板…）
```

`src/skills/loader.ts` 启动时会扫 `.skills/` 下一级目录：
- 没激活时，Prompt 里只放一行简短的目录提示，省 Token；
- 执行 `/skill load code-review` 激活后，整篇 `SKILL.md` 正文才会被 `skillContext` 管道塞进 System Prompt。

**Skill 的两条边界要记住**：
1. 它只是 Prompt 文本，**不会给模型新增任何代码能力，也突破不了角色权限**；
2. 仓库自带的 Frontmatter 解析器是手写的简单逐行 `key: value` 拆分，不支持完整 YAML 的多行字符串或数组嵌套。

> **看源码才发现的小细节**：除了 `/skill load <名>`，`src/commands/skill.ts` 还支持直接敲 `/<技能名>`（例如 `/code-review`）作为快捷触发。执行这个快捷命令时，代码一方面会把技能加进 `activeSkills`（进 System Prompt），另一方面又会把 `skill.content` 当成一条 `user` 消息直接发给模型——因此查看 Trace 会发现同一份 Skill 正文在这一轮里出现了两次。

## 5.2 Plugin：进程内热插拔一组工具

如果你想加的不是几句提示词，而是一套有初始化、有配置读取、有清理动作的工具（比如连接公司内部工单系统），就用 **Plugin**。

`src/plugins/manager.ts` 定义了标准的插件生命周期：
1. 插件导出一个 `PluginDefinition`（包含 `name`、`version`、`activate(api)`、可选的 `destroy()`）；
2. 加载时 `PluginManager` 传入受限的 `api`（提供 `registerTools()`、`getConfig()`、`log()`），不让插件乱改全局对象；
3. 注册进来的工具自动加 `<插件名>__` 前缀（如 `supabase__query`、`supabase__insert`）；
4. 卸载时调用 `destroy()` 释放连接，并把名下工具从 `ToolRegistry` 全部 `unregister()`。

### 看清楚示例插件与配置的“真实接线情况”
- `src/plugins/supabase-plugin.ts` 只是**教学占位示例**：即便你在 `.env` 里配了 `SUPABASE_URL` 和 `SUPABASE_KEY`，它也不会真连数据库，依然返回假表和模拟行。
- `config/schema.ts` 和向导生成的配置里有 `plugins: [{ name: 'supabase', enabled: false }]`，但 `src/main.ts` 当前**没有读取 `config.plugins` 的 `enabled` 开关**，而是直接把 `availablePlugins` 全加载了。这也是“配置文件有字段、主流程未完全接线”的一个典型例子。

## 5.3 MCP：为什么还要跨进程连外部工具

Plugin 要求工具必须用 TypeScript 写、且跑在同一个 Node.js 进程里。但如果某个工具服务别人已经用 Python/Go 写好了，或者你希望同一套 GitHub 工具同时给 Claude Desktop、Cursor 和 SuperAgent 用，就轮到 **MCP（Model Context Protocol）** 上场了。

```text
SuperAgent (Node 进程)  ◀── JSON-RPC / stdio ──▶  独立的 MCP Server 子进程 (GitHub / DB / …)
```

`src/tools/mcp-client.ts` 里其实写了**两个客户端**：
- 真实的 `MCPClient`（负责 `spawn` 启动子进程并通过标准输入输出发 JSON-RPC 请求）；
- 演示用的 `MockMCPClient`（返回模拟 GitHub issue、仓库搜索结果）。

`src/main.ts` 默认连的是 `MockMCPClient`（所以问 GitHub issue 不会真的发网络请求）。但从架构上看，不管 MCP Server 是真进程还是 Mock，一旦通过 `registry.registerMCPServer()` 挂进来，都会变成 `mcp__github__xxx` 工具（且默认标记 `shouldDefer: true` 走延迟发现），主循环 `agentLoop` 完全不需要改一行代码。

## 5.4 Channel 与 `ChannelGateway`：一套核心接多端入口

怎么把 CLI 里的 Agent 搬到飞书群或者网页上？千万别去 Channel 代码里把 `streamText()` 和 `while` 循环抄一遍。

项目的做法是抽了一层适配器 `src/channels/gateway.ts`：
- 每个通道只要实现 `ChannelDefinition`（`start`、`stop`、`send`、`onMessage`），把各平台五花八门的事件体统一转换成标准 `IncomingMessage`（`channelId`、`senderId`、`senderName`、`text`）；
- `ChannelGateway` 内部用 `<channelName>:<senderId>` 当 key 维护一张 `sessions` 内存表；
- 收到消息后，直接取该发送者的历史消息数组、拼一份最新 System Prompt，然后**原封不动调用同一个 `agentLoop()`**，最后把模型最后一条文本回复通过 `channel.send()` 发回去。

### 飞书与本地 Dashboard（`src/channels/feishu.ts`）
`src/channels/feishu.ts` 一边用 Hono 起了本地 `3000` 端口的 Dashboard 测试页，一边判断环境变量：只要 `FEISHU_APP_ID` 和 `FEISHU_APP_SECRET` 齐了才连飞书 WebSocket 长连接，缺一个就静默退化成纯 Dashboard（不会报错卡死）。

**实际开发时一定要清楚 Channel 的当前边界**：
1. **无鉴权风险**：Dashboard 网页接口没有任何登录校验，但背后连着能跑 `bash` 和写文件的 `agentLoop`，不要映射到公网；
2. **状态隔离与缺失**：`ChannelGateway` 的对话只放在内存 `Map` 里，**没有写进 `.sessions/`，也没有经过 `LocalTraceRecorder` 和 `UsageTracker`**；同一用户如果连着点两次发送，同一会话也没做并发排队锁。

## 5.5 动手试一试（Mock 模式即可）

启动 `pnpm start`，在终端依次运行：

```text
/skill list
/skill load code-review
/plugin
/channel
查一下有哪些表
```

然后打开浏览器访问 `http://localhost:3000`，在 Dashboard 页面发一句 `你好` 或 `查询 vercel/ai 的 issue`。

重点对比终端日志与结果：
1. 为什么在浏览器发消息，终端里也会打出一模一样的 `Step 1` 和 `[调用: tool_search(...)]`？
2. 发完消息去 `.sessions/default.jsonl` 和 `.traces/` 看看，网页里聊的那句有没有落盘？（答案：没有，这正是“统一了执行引擎、还没统一旁路记录”的直观体现。）

## 5.6 本章自检

1. 如果你只想让 Agent 严格按照团队的 SQL 规范写查询，该做成 Skill、Plugin 还是 MCP？
2. 什么时候选进程内 Plugin，什么时候选跨进程 MCP？两者最终在哪一层合流？
3. 为什么不应该在飞书 Channel 的回调里单独复制一份 `agentLoop`？
4. 如果我在 `super-agent.config.json` 里把 `supabase` 的 `enabled` 改成 `false`，启动时它还会加载吗？
5. 为什么当前不建议把本机的 3000 端口通过内网穿透暴露给外网体验？

<details>
<summary>参考思路（点击展开）</summary>

1. 做成 Skill（`.skills/<name>/SKILL.md`）最合适，因为规范和检查清单属于提示词知识，不需要执行外部代码。
2. 能力需要直接共享当前 Node 进程对象、或仅本项目自用时用 Plugin；能力已经用别的语言写好、或要跨客户端复用、需要进程隔离时用 MCP。两者最终都通过 `ToolRegistry` 转成标准工具定义喂给 `agentLoop`。
3. 否则工具注册、角色过滤、循环检测、Prompt 构造等所有升级都要改两份代码；应该把通道差异收口在 `IncomingMessage`/`OutgoingMessage` 格式转换上，核心逻辑全走 `ChannelGateway → agentLoop`。
4. 依然会加载。因为目前 `main.ts` 的 `startAgent()` 直接遍历 `availablePlugins` 全部加载，尚未读取 `config.plugins[].enabled`。
5. Dashboard HTTP 入口没有任何鉴权，且连接的 `ToolRegistry` 默认是 `owner` 角色，能读写宿主机文件并执行 `bash` 命令。

</details>
