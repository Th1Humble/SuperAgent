# 02. 工具系统：让 Agent 能动手做事

模型本身只能“想”和“说”，不能读硬盘、也不能敲命令。工具系统的职责，就是把你愿意开放给它的本地函数包成标准说明书递给模型，并在模型提出请求时安全地替它跑起来。

## 2.1 一个工具长什么样

在 `src/tools/registry.ts` 里，每个工具都要按 `ToolDefinition` 来定义：

```ts
export interface ToolDefinition {
  name: string;
  description: string;
  parameters: Record<string, unknown>; // JSON Schema
  execute: (input: any) => Promise<unknown>;
  isConcurrencySafe?: boolean;
  isReadOnly?: boolean;
  maxResultChars?: number;
  profile?: string[];
  shouldDefer?: boolean;
  searchHint?: string;
}
```

前 4 个字段决定“**这工具干嘛用、收什么参数、怎么执行**”，后面 6 个字段全是给运行时做治理用的（并发锁、截断、裁剪、延迟加载）。

## 2.2 模型到底怎么“调”到本地函数的

很多新手会误以为模型直接获得了执行 JavaScript 的权限，其实不是：

```text
我们的 ToolDefinition
  ↓ ToolRegistry.toAISDKFormat() 转成 AI SDK 格式
连同系统提示词一起发给模型
  ↓
模型决定调用，吐出一段 JSON：{ toolName: "read_file", input: { path: "package.json" } }
  ↓
AI SDK 在本地 Node.js 进程里校验参数并执行 execute()
  ↓
拿到返回值，经截断和 Hook 处理后包装成 tool-result
  ↓
放进 messages，下一轮再发给模型看
```

换句话说，**模型只负责“填申请表”，真正把函数跑起来的是本地应用进程**。

## 2.3 `ToolRegistry` 不只是个 `Map`

把所有工具塞进一个对象里很容易，但真正工程化的难点在“管住它”。`src/tools/registry.ts` 把整个工具治理全收口在一个类里：

- 增删改查工具、按命名空间挂入 MCP 工具；
- 按当前 Profile（场景档位）、Role（用户角色）、是否已发现（`shouldDefer`）过滤“本轮模型能看到哪些工具”；
- 把定义转换成 AI SDK 需要的 `jsonSchema` + `execute` 包装；
- 执行前先跑 Bash 危险命令检查和 Pre Hook；
- 用并发/独占锁协调同一批并行工具；
- 执行完把超长结果截断，再过一遍 Post Hook。

## 2.4 注册、覆盖与命名空间

`registry.register(...tools)` 按工具 `name` 存进 `Map`，重名时**后注册的会覆盖先注册的**。

项目里网页搜索就用到了这个技巧：`src/tools/web-search.ts` 里的 `pickSearchTool()` 会检查环境变量，有 `TAVILY_API_KEY` 就返回 Tavily 版，否则看 `SERPER_API_KEY`，但对外暴露的工具名统一叫 `web_search`——上层逻辑完全不用关心换了哪家搜索。

而来自外部插件和 MCP 的工具为了防止重名冲突，会自动加上前缀：
- 插件工具：`<插件名>__<工具名>`，比如 `supabase__query`、`supabase__insert`；
- MCP 工具：`mcp__<服务名>__<工具名>`，比如 `mcp__github__list_issues`。

加前缀还有一个好处：看日志、做权限控制时一眼就能看出工具是谁提供的。

## 2.5 内置工具一览（顺便看清哪些只是教学简化版）

`src/tools/index.ts` 汇总了基础工具，其它领域工具则在 `src/main.ts` 里逐个注册进来：

| 分类 | 工具名 | 并发/读写属性 | 新手需要注意的局限 |
| --- | --- | --- | --- |
| 文件操作 | `read_file`、`list_directory` | 只读 / 可并发 | 没有做“工作区根目录沙箱”，理论上能读到项目外路径 |
| 文件写入 | `write_file`、`edit_file` | 写 / 独占 | `edit_file` 靠精确字符串替换，原文本出现多次或有空格差异会踩坑 |
| 项目搜索 | `glob`、`grep` | 只读 / 可并发 | 自己写的简版递归，分别上限 100 文件 / 50 处匹配，没处理二进制和复杂 `.gitignore` |
| Shell | `bash` | 写 / 独占 | 用同步 `execSync` 跑（超时 10 秒、缓冲 1MB），执行期间会卡住整个 Node 事件循环 |
| 网页 | `web_search`、`web_fetch` | 只读 / 可并发 | `web_fetch` 把抓回来的网页直接转 Markdown；注意网页里可能藏 Prompt 注入文本 |
| 示例工具 | `get_weather`、`calculator` | 只读 / 可并发 | 天气是写死城市数据；`calculator` 用 `new Function()` 算表达式，生产环境绝对不能这样用 |
| 领域能力 | `memory`、`rag_ingest`、`rag_search`、`cron_manage`、`spawn_agent` | 视读写而定 | 在 `main.ts` 分别注册（注意定时工具在源码里叫 `cron_manage`） |

## 2.6 并发安全与读写锁

在 `src/agent/loop.ts` 里开启了 `parallelToolCalls: true`。一次模型回复可能同时丢出来好几个工具请求，比如一边查天气一边读文件，或者一边写文件一边再读同一个文件。

怎么保证不会乱套？`ToolRegistry` 用两个状态做了一把简易读写锁：
- `concurrentCount`：当前正在并发跑的“安全工具”数量；
- `exclusiveLock`：是不是正在跑某个“独占工具”（如 `write_file`、`bash`）。

规则很简单：
- **并发安全工具**（`isConcurrencySafe: true`）：只要当前没有独占锁（`!exclusiveLock`），就能直接进，把 `concurrentCount + 1`；
- **独占工具**：必须等到 `exclusiveLock === false` 且 `concurrentCount === 0`（手头所有读操作都结束了），才能把 `exclusiveLock` 置为 `true` 单独跑。跑完调用 `drainQueue()` 唤醒排队的任务。

> **代码里的边界**：这是一把教学版“非公平锁”——只要读工具不断插队进来，排队的写工具可能会一直等（写饥饿），而且等待队列没有超时取消机制。真正高并发服务需要公平队列或按资源路径分段加锁。

## 2.7 结果截断：防止一个大文件把上下文撑爆

如果模型调 `read_file` 读了一个 5 万字的日志全塞回 `messages`，下一轮请求不仅巨贵，还可能直接超出模型窗口。

`truncateResult()`（`src/tools/registry.ts`）默认把工具输出限制在 **3000 字符**（单工具可通过 `maxResultChars` 改写）。而且它不是粗暴砍掉尾巴，而是**保留前 60% + 后 40%**，中间插一行 `... [省略 xxx 字符] ...`。

之所以“保头保尾”，是因为代码和日志最关键的信息往往就在开头（文件导入、报错入口）和末尾（堆栈底部的真正错误原因、最终汇总）。

## 2.8 为什么“已注册”不等于“模型能看见”

每一轮请求发给模型前，`getActiveTools()` 会连着过三道筛子（`src/tools/registry.ts`）：

1. **Profile 过滤**：如果工具声明了 `profile`，当前 `activeProfile` 必须在里面；
2. **延迟发现过滤**：如果工具设了 `shouldDefer: true`，只有被 `tool_search` 查过（进过 `discoveredTools` 集合）才会露面；
3. **角色权限过滤**：`canUseTool(currentRole, tool.name)` 必须放行。

所以以后调试发现“模型死活不用某个工具”，别光顾着改 Prompt，先检查它有没有被这三道筛子挡掉。

## 2.9 为什么需要 `tool_search`（延迟发现）

你可能会想：干嘛不把所有工具一股脑全塞给模型？

因为**每个工具的 JSON Schema 都要算进输入 Token**。如果接了十几个 MCP 服务、上百个工具，光工具说明书就能吃掉几万 Token，每轮对话都在白烧钱，而且几十个长得差不多的工具放一起，模型选错的概率会飙升。

项目演示了 Anthropic 常用的延迟加载思路：
1. MCP 工具在注册时默认加上 `shouldDefer: true` 和简短的 `searchHint`；
2. System Prompt 里只放一行简短清单（“以下工具可用，但需要先通过 `tool_search` 获取完整定义”）；
3. 当模型真的要查 GitHub Issue 时，先调 `tool_search({ query: "mcp__github__list_issues" })`；
4. `Registry` 把它加进 `discoveredTools`；从**下一个 Step** 开始，`toAISDKFormat()` 才会把完整参数 Schema 发给模型。

> **说清楚真实实现**：当前 `searchTools()`（`src/tools/registry.ts`）其实只支持**精确工具名**或逗号分隔的多个工具名，并没有做语义检索或关键词模糊匹配。`searchHint` 只是印在 System Prompt 里给模型看的提示语。

## 2.10 Hook 在执行链里的准确位置

当你调用 `toAISDKFormat()` 把工具包好，真正跑一次工具的顺序是这样的：

```text
1. 如果是 bash，先过 classifyBashCommand() 风险识别（危险直接拒）
2. 跑 Pre Hooks（可拦截 block，代码里也留了 modify 入口）
3. 抢并发锁 或 独占锁
4. 真正执行 tool.execute(input)
5. 转字符串并做 truncateResult 截断
6. 跑 Post Hooks（如给 bash 输出加上时间戳）
7. finally 里释放锁
```

注：子 Agent 内部调用的是 `toAISDKFormatUnlocked()`（`src/tools/registry.ts`），跳过了锁、Bash 风险识别和 Hook——这部分设计权衡在第 06 章细拆。

## 2.11 亲手加一个新工具要几步

假设想加一个只读统计目录文件数的工具 `count_files`：

1. 在 `src/tools/` 下新建文件（或加进现有工具文件），导出一个 `ToolDefinition`：
   - 写清 `description`（告诉模型**什么时候该用它**，不要只把名字翻译一遍）；
   - `parameters` 尽量加上 `required` 和 `additionalProperties: false`；
   - 只读且不改状态的，老实标上 `isReadOnly: true` 和 `isConcurrencySafe: true`。
2. 在 `src/tools/index.ts` 把它加进 `allTools` 数组。
3. 重启 `pnpm start`，看启动日志里的可用工具数量是不是多了一个。

## 2.12 动手试一试（Mock 模式专属口令）

启动 `pnpm start` 后，依次输入下面几条（**注意：`mock-model.ts` 对并行、搜索和 glob 只认固定触发词**）：

```text
读取 package.json
测试搜索
测试glob
测试并发
```

输入 `测试并发` 时仔细看终端日志：Mock 会一口气吐出两个 `get_weather` 和一个 `list_directory`，观察它们是如何触发并发安全路径的；再输入 `/role guest` 后重新输入 `测试bash`，看看工具不在可见集合时会发生什么（测完记得 `/role owner` 切回来）。

## 2.13 本章自检

1. 工具已经在 `allTools` 里注册了，为什么本轮模型还是看不见它？
2. `isReadOnly` 和 `isConcurrencySafe` 是一回事吗？为什么分开？
3. 为什么截断工具输出时要“保留前 60% 和后 40%”，而不是直接截前 3000 字？
4. 当前的 `tool_search` 支持拿自然语言模糊搜工具吗？
5. Hook Pipeline、权限控制和并发锁，为什么全部写在 `ToolRegistry` 里，而不是让每个工具自己写？

<details>
<summary>参考思路（点击展开）</summary>

1. 可能被 `getActiveTools()` 的任意一道过滤掉了：当前 `activeProfile` 不在工具的 `profile` 里；设置了 `shouldDefer: true` 且尚未被 `tool_search` 发现；或者当前 `Role`（如 `guest`）无权访问。
2. 不是一回事。大多数只读工具同时也是并发安全的，但有些只读操作可能依赖共享状态/限流器而不宜并发；分开标可以让安全治理和锁调度解耦。
3. 文件开头的导出/声明很重要，但错误堆栈、执行结论和汇总往往落在文件末尾，保头保尾比单纯截断开头更容易保留有效线索。
4. 不支持。`searchTools()` 源码里直接按 `this.tools.get(name)` 精确查表（支持逗号分隔多个名字）。
5. 把横切关注点收口在注册表统一包装，新工具作者只需要关心输入输出本身，不容易因为某个人忘写鉴权或忘加锁而在整个系统里撕开口子。

</details>
