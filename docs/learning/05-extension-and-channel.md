# 05. Skills、Plugins、MCP 与 Channel

## 5.1 四种扩展不要混在一起

| 机制 | 扩展对象 | 是否增加工具 | 是否改变输入来源 | 典型用途 |
| --- | --- | --- | --- | --- |
| Skill | System Prompt | 不一定 | 否 | 注入领域流程、规范和知识 |
| Plugin | 当前进程能力 | 是 | 否 | 注册一组工具和生命周期逻辑 |
| MCP | 外部工具服务 | 是 | 否 | 标准化连接第三方工具 |
| Channel | 消息入口与出口 | 否 | 是 | 飞书、Web、Slack 等接入 |

可以把它们记成：Skill 教它怎么做，Plugin 给它装能力，MCP 从外部借能力，Channel 让别人从新入口使用它。

## 5.2 Skill：Prompt 层扩展

项目 Skill 位于：

```text
.skills/<skill-name>/SKILL.md
```

文件可以带 Frontmatter：

```yaml
---
name: code-review
description: 以高级工程师视角审查项目代码
when_to_use: 当用户要求审查代码时
---
```

`SkillLoader`：

1. 扫描 `.skills/` 的直接子目录；
2. 读取 `SKILL.md`；
3. 解析简单 Frontmatter；
4. 保存 name、description、whenToUse、content、dirPath；
5. 根据 `activeSkills` 构造 Prompt 段落。

未激活的 Skill 只以简短列表出现在 Prompt；激活后完整正文进入 System Prompt。

命令：

```text
/skill list
/skill load code-review
/skill unload code-review
```

Skill 适合表达：步骤、检查清单、输出格式、领域约束。它本身不能突破工具权限，也不能直接执行代码。

当前 Frontmatter 解析器是简单的逐行 `key: value`，不支持完整 YAML 的数组、嵌套、多行值等能力。

## 5.3 Plugin：进程内扩展

PluginDefinition 包含：

- `name`、`version`、`description`；
- 默认 `config`；
- `activate(api)`；
- 可选 `destroy()`。

激活时 PluginManager 提供 API：

- `registerTools()`：注册工具；
- `getConfig()`：获取环境变量替换后的配置；
- `log()`：带插件前缀输出日志。

插件工具自动加 `<plugin>__` 前缀。卸载时调用 `destroy()` 并从 Registry 注销所有工具。

## 5.4 Supabase Plugin 示例

`src/plugins/supabase-plugin.ts` 展示插件如何：

- 声明环境变量配置；
- 注册查询和写入类工具；
- 在无配置时提供 Mock 数据；
- 在日志中说明当前模式。

但它目前不会真正连接 Supabase，即使配置 URL 和 Key，主要逻辑仍是占位/模拟。学习重点是 Plugin API 和生命周期，不是数据库功能。

## 5.5 当前插件配置的真实情况

`config/schema.ts` 定义了 `plugins[]` 和 `enabled`，初始化向导也会生成 Supabase disabled 配置。但 `main.ts` 当前不读取 `config.plugins` 决定是否加载，而是遍历 `availablePlugins` 全部尝试加载。

因此“配置中 enabled=false”并不等于当前主程序不会加载插件。这是配置定义与运行时接线尚未完全对齐的例子。

## 5.6 MCP：跨进程工具协议

MCPClient 负责连接一个实现 Model Context Protocol 的工具服务。真实客户端以子进程方式工作：

```text
SuperAgent
  ↕ JSON-RPC / stdio
MCP Server Process
  ↕
GitHub / Database / Other service
```

典型过程：

1. 启动 MCP Server 子进程；
2. 建立消息通信；
3. `listTools()` 获取能力；
4. Registry 把工具映射为 AI SDK Tool；
5. 调用时 `callTool()` 转发；
6. 关闭 Agent 时终止连接。

当前 `main.ts` 使用 `MockMCPClient`，返回模拟 GitHub 工具结果。真实 `MCPClient` 已有代码，但没有在默认启动路径实例化。

## 5.7 Plugin 与 MCP 如何选择

使用 Plugin，当：

- 能力和 Agent 在同一 Node.js 进程；
- 希望直接复用内部对象；
- 需要项目自定义生命周期；
- 不需要跨语言和标准协议。

使用 MCP，当：

- 工具由独立服务维护；
- 希望跨语言或跨进程；
- 同一工具服务要供多个 Agent 客户端使用；
- 希望通过标准协议发现 Schema 和调用。

两者最终都进入 ToolRegistry，所以 Agent Loop 不需要知道来源。

## 5.8 Channel 抽象

ChannelDefinition 负责：

- `start()`：启动连接或服务；
- `stop()`：释放资源；
- `send()`：发送回复；
- `onMessage()`：注册收到消息的回调。

IncomingMessage 统一不同平台的字段：Channel ID、发送者 ID、发送者名称、文本和可选元数据。

这使 ChannelGateway 可以不关心飞书 SDK 的具体事件结构。

## 5.9 ChannelGateway

Gateway 保存：

- 已注册 Channel；
- 每个发送者的内存消息历史；
- 模型、Registry 和 System Prompt 构造函数。

会话 key 是：

```text
<channelName>:<senderId>
```

收到消息后：

1. 获取或创建该发送者的 messages；
2. 追加 user message；
3. 构造 System Prompt；
4. 调用同一个 `agentLoop()`；
5. 从最后一条 assistant message 提取文本；
6. 调用 Channel 的 `send()`。

因此终端、飞书、Dashboard 虽然入口不同，核心推理和工具能力相同。

## 5.10 飞书 Channel 与 Dashboard

`FeishuChannel` 同时承担两种入口：

- 有 App ID/Secret 时建立飞书 Bot 长连接；
- 始终通过 Hono 启动本地 Dashboard HTTP 服务。

Dashboard 默认监听 `FEISHU_PORT`，默认 3000。它的测试入口会构造 IncomingMessage 交给 Gateway，回复在终端和页面间流转。

飞书长连接与 Dashboard 端口无关。缺少任一飞书凭据时，只运行 Dashboard。

安全注意：Dashboard 测试入口没有鉴权，而 Agent 能调用文件和 Shell 工具。不要把端口暴露到不可信网络。

## 5.11 Channel 的状态边界

Gateway 的会话只保存在内存 Map：

- 重启后丢失；
- 不写入 `SessionStore`；
- 不记录主终端那套 Trace 和 Usage；
- 同一发送者在不同 Channel 下是不同会话；
- 群聊是否应按群、线程或用户隔离，当前模型较简单。

生产化时需要明确 Session identity、并发消息排序、重复事件去重和持久化策略。

## 5.12 如何设计新的扩展

### 新 Skill

适合加入代码评审规范、写作风格或团队流程。先写清触发场景，正文只放模型确实需要的指令，避免把整本参考书塞进 Prompt。

### 新 Plugin

定义配置和一组有共同生命周期的工具，例如内部工单插件。工具统一前缀，卸载时释放数据库连接或订阅。

### 新 MCP

先保证 Server 可以独立列出和调用工具，再在 `connectMCP()` 中创建真实客户端。明确哪些工具只读、是否适合并发、是否应延迟发现。

### 新 Channel

实现 ChannelDefinition，把平台事件转换为 IncomingMessage，把 OutgoingMessage 转回平台 API。不要在 Channel 中复制 Agent Loop。

## 5.13 实践

```text
/skill list
/skill load code-review
/plugin
/channel
```

打开 `http://localhost:3000` 从 Dashboard 发送一条消息，比较它和终端消息：两者工具能力相同，但对话历史、Session、Trace 和 Usage 记录方式不同。

## 5.14 本章自检

1. Skill 为什么不能替代 Tool？
2. Plugin 和 MCP 最终在哪一层汇合？
3. Channel 中为什么不应该重新实现 Agent Loop？
4. 当前 Supabase 和 GitHub MCP 是真实接入吗？
5. 为什么 Dashboard 不能直接暴露到公网？
