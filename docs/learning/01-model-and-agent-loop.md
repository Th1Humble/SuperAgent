# 01. 从模型调用到 Agent Loop

## 1.1 聊天程序为什么不是 Agent

一个最小聊天程序只需要：读取输入、调用模型、打印输出。它是单向的：

```text
user prompt → model → text
```

Agent 多了行动和观察：

```text
user task
  ↓
model thinks and requests a tool
  ↓
runtime executes the tool
  ↓
tool result returns to model
  ↓
model decides to call another tool or answer
```

关键变化不是“用了更聪明的模型”，而是程序允许模型在一个受控循环中选择行动。

## 1.2 模型抽象

`main.ts` 的 `createModel()` 根据配置决定模型实现：

```ts
if (!cfg.apiKey) return createMockModel();
const provider = createOpenAI({ baseURL: cfg.baseURL, apiKey: cfg.apiKey });
return provider.chat(cfg.name);
```

这里体现了 Provider 抽象：Agent Loop 不需要知道背后是 Mock 还是 Qwen，只依赖 AI SDK 统一的模型接口。

DashScope 提供 OpenAI 兼容协议，所以可以用 `createOpenAI` 并替换 `baseURL`。这降低了业务层对厂商 SDK 的耦合，但“协议兼容”不代表所有模型的工具能力、缓存字段和错误格式完全相同，接入新模型仍需验证。

## 1.3 Mock Model 的教学价值

`src/mock-model.ts` 是仓库最大的单文件。它不仅返回固定文本，还模拟了：

- 普通文本响应；
- 流式 `text-delta`；
- 工具调用意图识别；
- 并行工具调用；
- 工具结果后的继续回答；
- Token Usage；
- Cache read/write 行为。

它的作用不是模拟真实智能，而是让整个运行时在没有 API Key 时仍能经过真实的 AI SDK 调用路径。这样学习工具协议、上下文和 Trace 时，不会被账号、网络或费用阻塞。

需要注意：Mock 通过关键词和规则识别工具意图，不能用于评价真实模型的规划能力。

## 1.4 消息为什么是数组

模型并不是自动记得上一轮。多轮对话依赖程序把历史重新发送：

```ts
const messages: ModelMessage[] = [
  { role: 'user', content: '我叫小明' },
  { role: 'assistant', content: '你好，小明' },
  { role: 'user', content: '我叫什么？' },
];
```

工具调用后还会出现结构化消息部件，例如 tool-call 和 tool-result。不能假设每条 `content` 都是字符串，这也是代码中经常判断 `typeof content === 'string'` 或 `Array.isArray(content)` 的原因。

常见角色包括：

- `system`：行为规则和动态上下文，在本项目中作为 `streamText` 的独立 `system` 参数；
- `user`：用户请求，以及循环检测产生的系统提醒；
- `assistant`：模型的文本或工具调用；
- `tool`：工具执行结果。

## 1.5 为什么使用流式输出

同步生成需要等待完整回答结束。流式生成会不断产生事件，用户能更早看到文本，运行时也能观察工具调用：

```ts
const result = streamText({ model, system, tools, messages });

for await (const part of result.fullStream) {
  // text-delta / tool-call / tool-result / ...
}
```

本项目主要处理三类事件：

| 事件 | 含义 | 当前行为 |
| --- | --- | --- |
| `text-delta` | 模型新生成的一段文本 | 终端立即打印并累积到 `fullText` |
| `tool-call` | 模型请求某个工具 | 打印调用、进行循环检测并记录 |
| `tool-result` | 工具执行完成 | 打印预览并记录结果哈希 |

真正要追加回历史的结构化消息来自 `await result.response`，用 `messages.push(...stepResponse.messages)` 保存。`fullText` 主要用于终端展示和 Trace。

## 1.6 Agent Loop 的骨架

`src/agent/loop.ts` 的逻辑可以简化为：

```ts
while (step < stepLimit) {
  const result = streamText({ model, tools, messages });
  consume(result.fullStream);
  messages.push(...(await result.response).messages);

  if (tokenBudgetExhausted) break;
  if (loopDetectorSaysStop) break;
  if (!hasToolCall) break;
}
```

结束条件不是只有一个：

1. 模型没有再调用工具，说明已给最终回答；
2. 达到最大步数，默认 30；
3. Token 累计超过 200,000；
4. 循环检测触发严重级别；
5. `AbortSignal` 被取消；
6. 出现不可重试错误并向外抛出。

这是 Agent 工程的重要原则：任何自主循环都必须有资源上限和外部中止机制。

## 1.7 一次工具调用为何需要两次模型调用

假设用户说“读取 `package.json` 并告诉我项目名”：

### Step 1

输入包括用户请求和工具定义。模型返回 `read_file({ path: "package.json" })`。运行时执行工具，把文件内容作为 tool-result 写回消息。

### Step 2

模型再次收到完整消息历史。这次能看到工具结果，于是生成“项目名是 SuperAgent”，不再请求工具，循环结束。

模型并不会直接执行本地函数。AI SDK 的工具适配层把“模型请求”与“JavaScript 执行器”连接起来。

## 1.8 重试策略

`src/agent/retry.ts` 将网络错误、超时、限流和部分服务端错误判断为可重试错误。`calculateDelay()` 使用指数退避并加入随机抖动，避免大量请求同时重试。

Agent Loop 把 AI SDK 自带重试设为 0，由项目自己控制：

```ts
maxRetries: 0
```

外层最多执行 3 次重试。失败时 Trace 会记录 `step_attempt_failed`。每次重试前会重置本次尝试收集的 `hasToolCall`、`fullText` 和最后调用信息，防止失败尝试污染控制判断。

注意边界：工具本身已经产生外部副作用而后续网络失败时，简单重试可能重复副作用。当前项目没有为所有写操作实现幂等键，生产系统需要额外设计。

## 1.9 循环检测

只设最大步数还不够，因为模型可能在限制内大量重复无效工作。`src/agent/loop-detection.ts` 维护最近 30 次调用记录，并对工具名、参数和结果做稳定哈希。

它检测三种情况：

- `generic_repeat`：同样的工具和参数不断重复；
- `ping_pong`：两个调用交替出现；
- `global_circuit_breaker`：整体调用次数过多。

阈值分警告、严重和熔断。非严重时，系统向 `messages` 追加一条提醒，让模型换思路；严重时停止本轮。

局限也很明确：记录历史是模块级数组，`agentLoop()` 开始时会 `resetHistory()`。多个并行 Agent 共享同一模块时可能互相影响，生产实现应把检测器实例化到每次运行。

## 1.10 Token 预算与 Usage

每个 Step 从 AI SDK 得到 usage，经过 `normalizeUsage()` 统一成：

- `inputTokens`
- `outputTokens`
- `cacheReadTokens`
- `cacheWriteTokens`

Agent Loop 累计四类 Token。超过 90% 时提示，超过预算时停止。这里的预算是一次 `agentLoop()` 内所有 Step 的累计用量，不是模型上下文窗口的精确剩余容量。

`UsageTracker` 还根据模型价格表估算费用和缓存节省额。价格是代码中的静态教学数据，实际结算必须以供应商当前账单为准。

## 1.11 终端交互循环

`main.ts` 中 `ask()` 通过 `readline.question()` 接收输入：

1. 空输入或 `exit`：停止 Cron、Channel 和 Plugin，再关闭 readline；
2. 斜杠命令：交给 Command Dispatcher；
3. 普通消息：写 Session、建 Trace、运行 Agent Loop；
4. 完成后持久化新增响应，再次调用 `ask()`。

这里用递归回调继续提问，但每次下一轮都在异步回调结束后注册，并非同步递归堆栈不断增长。

## 1.12 关键源码阅读顺序

1. `src/index.ts`：只有 init 和正常启动两个分支；
2. `src/main.ts` 中的 `createModel()` 和 `ask()`；
3. `src/agent/loop.ts`：完整执行循环；
4. `src/agent/retry.ts`：失败分类和退避；
5. `src/agent/loop-detection.ts`：调用与结果历史；
6. `src/mock-model.ts`：从 `createMockModel()` 逆向阅读事件生成。

## 1.13 实践

启动 Mock 模式后依次尝试：

```text
你好
北京天气怎么样
读取 package.json，告诉我项目名
先列出 src 下的目录，再读取 src/index.ts
```

观察每个请求出现了几个 `Step`，哪些请求触发 `[调用: ...]`，工具结果后为什么还会显示“继续下一步”。然后查看终端给出的 `.traces/*.jsonl` 路径：

```bash
pnpm run trace:inspect .traces/你的轨迹文件.jsonl
```

## 1.14 本章自检

1. 为什么 `fullText` 不能代替 `result.response.messages`？
2. 工具调用为何至少需要“模型决定调用”和“模型观察结果”两个阶段？
3. 最大步数、循环检测和 Token 预算分别防什么？
4. 为什么自动重试对有副作用的工具存在风险？
5. Mock Model 能验证运行时协议，但不能验证什么？
