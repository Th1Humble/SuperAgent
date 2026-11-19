# 02. 工具系统：让 Agent 能做事

## 2.1 工具是什么

对模型来说，工具是一份可选择的能力说明；对运行时来说，工具是一个受约束的 JavaScript 执行器。两者通过结构化协议连接。

本项目中的 `ToolDefinition` 包含：

```ts
interface ToolDefinition {
  name: string;
  description: string;
  parameters: Record<string, unknown>;
  execute: (input: any) => Promise<unknown>;
  isConcurrencySafe?: boolean;
  isReadOnly?: boolean;
  maxResultChars?: number;
  profile?: string[];
  shouldDefer?: boolean;
  searchHint?: string;
}
```

最核心的是四项：名称、描述、参数 Schema、执行函数。其余字段负责运行时治理。

## 2.2 模型如何调用本地函数

完整链路如下：

```text
ToolDefinition
  ↓ ToolRegistry.toAISDKFormat()
AI SDK tool schema + execute
  ↓ 随模型请求发送
模型选择工具并生成 JSON 参数
  ↓ AI SDK 校验并调用 execute
本地函数执行
  ↓
结果转成 tool-result
  ↓
下一 Step 重新交给模型
```

模型只生成“我要调用什么和参数是什么”，没有直接获得 Node.js 执行权限。真正执行发生在应用进程中。

## 2.3 ToolRegistry 的职责

`src/tools/registry.ts` 是工具系统中心。它承担：

- 注册、覆盖和卸载工具；
- 连接 MCP Server 并把远程工具注册成本地定义；
- 根据 Profile 过滤；
- 根据角色过滤；
- 隐藏尚未发现的延迟工具；
- 将 JSON Schema 转成 AI SDK 格式；
- 在执行前做 Bash 风险识别和 Pre Hook；
- 管理并发/独占执行；
- 截断结果；
- 在执行后做 Post Hook。

这意味着 Registry 不只是一个 `Map`，而是工具执行的策略边界。

## 2.4 工具注册与覆盖

`register(...tools)` 以工具名作为 Map key。相同名称后注册会覆盖先注册的定义。网页搜索利用了这一点：`pickSearchTool()` 根据环境变量返回 Tavily 或 Serper 实现，但统一名称都是 `web_search`。

插件工具会增加命名空间，如：

```text
supabase__query
supabase__insert
```

MCP 工具则使用：

```text
mcp__github__list_issues
mcp__github__search_repositories
```

前缀避免不同来源的工具重名，也能让权限和审计更容易识别来源。

## 2.5 内置工具分类

### 文件工具

- `read_file`：按路径读取，支持 offset/limit；
- `write_file`：创建目录后写入完整内容；
- `edit_file`：用精确字符串替换局部内容；
- `list_directory`：列出目录和条目类型。

写工具被标记为非只读、非并发安全。`edit_file` 要求旧文本精确出现，找不到或出现多次时应谨慎处理，避免错误替换。

### 项目搜索

- `glob`：递归遍历后用简化通配符匹配，最多 100 个结果；
- `grep`：用正则逐文件搜索，最多 50 条匹配。

这两个是教学实现，不等价于成熟的 `fast-glob` 或 ripgrep：忽略规则、符号链接、二进制识别和大型仓库性能都较简单。

### Shell

`bash` 用 `execSync` 执行命令，超时 10 秒、缓冲区 1MB。执行前 Registry 会先经过 Bash 风险分类。

同步执行会阻塞 Node.js 事件循环，因此只适合教学和短命令。生产系统通常使用异步子进程、沙箱、工作目录约束和更严格的资源限制。

### Web

- `web_search`：按环境变量选择 Tavily 或 Serper；
- `web_fetch`：抓取 HTML，并用 Turndown 转 Markdown。

所有网页正文仍是不可信输入，不能把网页中的指令当作系统授权。当前代码尚未实现完整的 Prompt Injection 防护。

### 示例工具

- `get_weather`：本地固定数据；
- `calculator`：用动态函数执行表达式。

`calculator` 的实现便于演示，但 `new Function()` 不适合接收不可信表达式。生产实现应使用安全解析器和允许的运算符白名单。

### 领域工具

- `memory`：长期记忆的 list/read/write/search/delete/lint；
- `rag_ingest`、`rag_search`：导入和检索知识；
- `cron`：创建、列出、运行、启停、删除定时任务；
- `spawn_agent`：派发一个或多个子 Agent。

## 2.6 并发安全和读写锁

AI SDK 开启了 `parallelToolCalls`，一个模型响应可能同时请求多个工具。如果两个工具都读取独立数据，可以并发；如果涉及写入或不可重入资源，则需要串行。

Registry 使用两种状态：

- `concurrentCount`：当前并发安全工具数量；
- `exclusiveLock`：是否正在运行独占工具。

规则是：

```text
安全工具：只等待 exclusiveLock，然后 concurrentCount + 1
独占工具：等待 exclusiveLock=false 且 concurrentCount=0，再独占
```

结束时通过 `drainQueue()` 唤醒等待者重新检查条件。

这里不是完整公平读写锁：新来的读任务可能在写任务等待时继续进入，存在写饥饿可能。它足以解释并发分类，但生产实现需考虑公平性、超时和取消。

## 2.7 结果截断

工具可能返回超长文件或网页。全部塞回上下文会快速消耗 Token。`truncateResult()` 默认保留 3000 字符：

- 前 60%；
- 后 40%；
- 中间插入省略提示。

同时保留头尾比只保留开头更适合日志和代码：声明通常在前面，错误和总结可能在末尾。

工具可以通过 `maxResultChars` 自定义上限。注意这是按 JavaScript 字符数截断，不是精确 Token。

## 2.8 Profile、Role 与 Active Tools

工具是否发送给模型由 `getActiveTools()` 决定，依次检查：

1. 工具 Profile 是否包含当前 Profile；
2. 延迟工具是否已经发现；
3. 当前 Role 是否允许使用。

因此“已注册”不等于“本轮模型可见”。调试工具缺失时，应先检查活动 Profile、角色和延迟发现状态。

## 2.9 为什么需要 Tool Search

每个工具定义都占用输入上下文。工具少时可以全部发送，工具很多时会出现：

- Schema 消耗大量 Token；
- 相似描述让模型选不准；
- 每轮都为用不到的能力付费。

延迟工具设置：

```ts
shouldDefer: true,
searchHint: 'github issues repositories ...'
```

系统 Prompt 只列出简短提示。模型先调用 `tool_search({ query: "完整工具名" })`，Registry 把工具加入 `discoveredTools`。下一 Step 的 `toAISDKFormat()` 才会暴露完整定义。

当前 `searchTools()` 实际按精确名称或逗号分隔名称查找，并不是语义搜索。`searchHint` 只用于 Prompt 展示。文档和工具名应对此说清楚。

## 2.10 MCP 接入

`registerMCPServer()` 的过程：

1. `client.connect()`；
2. `client.listTools()` 获取远程工具；
3. 给名称加 `mcp__<server>__` 前缀；
4. 把 MCP inputSchema 原样作为参数 Schema；
5. `execute` 委托给 `client.callTool()`；
6. 默认标记为只读、并发安全、延迟工具。

仓库定义了真实进程型 `MCPClient`，但 `main.ts` 当前连接的是 `MockMCPClient`。所以内置 GitHub MCP 不会访问真实仓库。

## 2.11 Hook 在执行链中的位置

工具执行的顺序是：

```text
Bash 风险分类
  ↓
Pre Hooks：allow / block / modify input
  ↓
获取并发或独占锁
  ↓
execute
  ↓
结果转字符串并截断
  ↓
Post Hooks：modify output
  ↓
释放锁
```

当前主程序注册了：

- `audit-log`：文件写操作前打印路径，但不会阻止；
- `bash-timestamp`：给 Bash 输出加 ISO 时间戳。

## 2.12 如何增加一个工具

以只读的项目统计工具为例：

```ts
export const countFilesTool: ToolDefinition = {
  name: 'count_files',
  description: '统计指定目录中的文件数量',
  parameters: {
    type: 'object',
    properties: {
      path: { type: 'string', description: '目标目录' },
    },
    required: ['path'],
    additionalProperties: false,
  },
  isConcurrencySafe: true,
  isReadOnly: true,
  execute: async ({ path }) => {
    // 实现统计逻辑
    return { path, count: 0 };
  },
};
```

然后在 `src/tools/index.ts` 导入并加入 `allTools`。设计工具时注意：

- 名称稳定、短且语义明确；
- 描述说明何时使用，不只重复名称；
- 参数 Schema 尽量严格；
- 输出保持结构化和精简；
- 明确是否只读、是否并发安全；
- 写操作考虑幂等、权限和审计；
- 错误应作为可理解结果或明确异常，不要静默失败。

## 2.13 实践

在 Mock 模式输入：

```text
读取 package.json
搜索 src 目录下包含 agentLoop 的文件
列出所有 TypeScript 文件
同时查询北京天气和计算 12 * 8
```

观察只读工具是否可能并行，写工具为什么默认独占。再输入 `/role guest` 后重复尝试，观察活动工具变化。

## 2.14 本章自检

1. 工具已注册但模型看不见，可能有哪些原因？
2. `isReadOnly` 与 `isConcurrencySafe` 是否是同一个概念？
3. 为什么工具结果也需要截断？
4. Tool Search 当前是语义搜索吗？
5. MCP 工具和本地工具进入 Agent Loop 后有什么共同点？
