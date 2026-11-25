# 08. 18 组实践实验：从“看懂”到“做过”

这一章把前面 7 章的内容拆成 18 个小实验，难度从“敲命令看日志”一路递进到“改源码接新能力”。

动手前先做两件小准备：
1. 建议先开一个自己的 Git 分支（比如 `git checkout -b learn/labs`），改乱了随时能 `git checkout -- .` 还原；
2. **分清哪些实验 Mock 模式就能跑、哪些必须接真实模型**：
   - **实验 0 ～ 实验 14**：基本都能在**无 API Key 的 Mock 模式**下跑通（文中标好了对应的 Mock 触发口令）；
   - **实验 15 ～ 实验 17**：涉及多步自主规划、`cron_manage` 和 `spawn_agent`，因为 `src/mock-model.ts` 没写这两类工具的触发规则，**建议接上真实 `DASHSCOPE_API_KEY` 再做**（或按实验里的提示用代码直接调对应模块验证）。

---

## 实验 0：建立基线（Mock 模式）

### 目标
确认终端、Dashboard 网页、RAG 自动导入和基础斜杠命令都能正常跑起来。

### 操作
```bash
pnpm install
pnpm start
```

正常启动后，终端大概率会打印出类似下面这段内容（如果报 3000 端口被占，先在 `.env` 配 `FEISHU_PORT=3001`）：

```text
  未找到 super-agent.config.json，使用默认配置
  运行 pnpm run init 生成配置文件

  已注册 3 个 Mock MCP 工具
  加载插件...
  [plugin:supabase] 未配置 SUPABASE_URL/SUPABASE_KEY，使用 Mock 数据模式
  ✓ supabase — 3 个工具
  启动 Channel...
  [feishu] 未配置 FEISHU_APP_ID / FEISHU_APP_SECRET，跳过飞书 Bot（仅启动 Dashboard）
  [feishu] Dashboard 已启动: http://localhost:3000
  [gateway] ✓ feishu 已启动
Super Agent v1.0 (type "exit" to quit)
...
  发现 2 个文档，自动导入知识库...
  知识库就绪，共 xx 个片段
```

看到 `You:` 后依次输入：

```text
你好
/rag
/context
/role
```

再去浏览器打开 `http://localhost:3000`，从页面里发一条 `你好`，最后回到终端敲 `exit` 退出。

### 验收标准
你能说清楚：为什么从 Dashboard 网页发消息，终端里照样会打印 `--- Step 1 ---`，但 `.sessions/default.jsonl` 和 `.traces/` 里却找不到网页那条对话？

---

## 实验 1：追踪一次不调工具的请求（Mock 模式）

### 操作
输入：

```text
你好
```

记下终端末尾打印的 `[Trace] .traces/default-xxx.jsonl` 路径，在另一个终端里跑：

```bash
pnpm run trace:inspect .traces/把这里换成真实文件名.jsonl
```

### 预期输出示例
`trace:inspect` 会打出一份摘要（格式来自 `src/trace/recorder.ts`）：

```text
Trace default-2026-...
  Step 1: context 1 messages
    completed in 612ms · 845 tokens
  Status: completed
```

### 再往下深挖一步
直接用编辑器打开那个 `.traces/*.jsonl` 文件，对照找这 4 个字段：
- `trace_started.model`（确认是 `mock-model` 还是真实模型）
- `step_started.context.system`（这里躺着拼完的完整 System Prompt）
- `step_completed.usage`（这一步的输入/输出/缓存 Token）
- `trace_finished.status`

---

## 实验 2：追踪一次工具调用（Mock 模式）

### 操作
输入：

```text
读取 package.json，告诉我项目名
```

### 预期现象
注意看终端分成两轮：
- `--- Step 1 ---`：打印 `[调用: read_file({"path":"package.json"})]` → `[结果: read_file] ...` → `→ 继续下一步...`；
- `--- Step 2 ---`：模型拿到文件内容后输出文本（Mock 模式下它会回 `工具返回了以下信息：...`），然后本轮结束。

### 思考检查
如果这一步里模型在调用工具前吐了一段文字，日志展示用的是 `fullText`，但 `agentLoop()` 塞回历史数组的为什么必须是 `(await result.response).messages`？

---

## 实验 3：观察并发安全工具与独占工具（Mock 模式）

### 操作
不要自己瞎编长句子，用 Mock 模型内置的并发触发词和写文件触发词：

```text
测试并发
测试写文件
```

（`测试写文件` 会在项目根目录生成一个 `test-output.txt`，实验做完记得删掉。）

### 观察重点
- 输入 `测试并发` 时，Mock 一次吐出 3 个调用（两次 `get_weather` + 一次 `list_directory`），这三者都标了 `isConcurrencySafe: true`，可以同时进 `acquireConcurrent()`；
- 输入 `测试写文件` 时，`write_file` 是独占工具，而且会顺便触发 Pre Hook `audit-log` 打印 `[audit] 文件写入操作`。

### 自己改一行代码验证
去 `src/tools/utility-tools.ts` 暂时把 `weatherTool` 的 `isConcurrencySafe` 改成 `false`，重启后再敲 `测试并发`，用 `console.log` 或时间观察它是不是变成串行排队了，理解后改回 `true`。

---

## 实验 4：亲手增加第一个工具

### 目标
写一个只读工具 `project_summary`，统计指定目录下的顶层子目录列表和文件数量。

### 起步代码骨架
新建 `src/tools/project-tools.ts`：

```ts
import fs from 'node:fs';
import path from 'node:path';
import type { ToolDefinition } from './registry.js';

export const projectSummaryTool: ToolDefinition = {
  name: 'project_summary',
  description: '统计指定目录下的文件总数与顶层子目录，用于快速了解项目结构。',
  parameters: {
    type: 'object',
    properties: {
      dir: { type: 'string', description: '目标目录，默认当前目录 "."' },
    },
    required: ['dir'],
    additionalProperties: false,
  },
  isReadOnly: true,
  isConcurrencySafe: true,
  execute: async ({ dir }: { dir: string }) => {
    if (!fs.existsSync(dir)) return `目录不存在: ${dir}`;
    const entries = fs.readdirSync(dir, { withFileTypes: true });
    const dirs = entries.filter(e => e.isDirectory() && e.name !== 'node_modules' && e.name !== '.git').map(e => e.name);
    const files = entries.filter(e => e.isFile()).map(e => e.name);
    return { dir, topLevelDirs: dirs, topLevelFileCount: files.length };
  },
};
```

然后打开 `src/tools/index.ts`，把 `projectSummaryTool` 导入并放进 `allTools` 数组。

### 验证方法（重要！）
1. 重启 `pnpm start`，输入 `/role`，确认可用工具数量比之前多 1 个；
2. **如果你用的是 Mock 模式**：因为 `mock-model.ts` 不认识新工具，可以在 `src/mock-model.ts` 的 `detectToolIntent()` 顶部加一条规则测试：
   ```ts
   if (text.includes('项目概览') && !hasToolResults(prompt)) {
     return { toolName: 'project_summary', args: { dir: '.' } };
   }
   ```
   然后输入 `项目概览`；如果你接了真实 API Key，直接用自然语言问 `帮我概览一下当前目录的项目结构` 即可。

---

## 实验 5：体验 `tool_search` 延迟发现（Mock 模式）

### 操作
启动后先看一眼初始上下文：

```text
/context
```

接着输入触发 GitHub MCP 延迟工具的口令：

```text
查询 vercel/ai 的 issue
```

跑完后再看一次上下文：

```text
/context
```

### 观察重点
1. Step 1 模型先调用 `tool_search({"query":"mcp__github__list_issues"})`；
2. Step 2 `mcp__github__list_issues` 已经被放进活动工具列表，模型顺利调用它拿到模拟 issue；
3. 对比前后两次 `/context` 中 `System tools` 占用的 Token 数量变化——记住：**延迟发现省的是 Schema Token，不是执行速度**。

---

## 实验 6：验证角色权限过滤（Mock 模式）

### 操作
```text
/role
/role guest
读取 package.json
测试bash
/role owner
```

### 观察重点
- 在 `guest` 下，`读取 package.json` 照常成功；
- 在 `guest` 下输入 `测试bash` 时：因为 `mock-model.ts` 不管当前角色硬吐了 `bash` 调用，而 `ToolRegistry` 里已经把 `bash` 过滤掉了，AI SDK 会报错提示工具不存在（被 `main.ts` 捕获打印 `[Agent] ...`）；如果换真实模型，模型压根看不到 `bash` Schema，会直接礼貌拒绝。

---

## 实验 7：观察 Hook 拦截与修改输出（Mock 模式）

### 操作
```text
/hooks
测试bash
测试写文件
```

观察两条内置 Hook 效果：
- `bash-timestamp`（Post Hook）：`测试bash` 的返回值最前面多了 `[2026-...]` ISO 时间戳；
- `audit-log`（Pre Hook）：`测试写文件` 执行前终端多了一行 `[audit] 文件写入操作: write_file → test-output.txt`。

### 进阶小改（顺便验证我们提到的源码细节）
去 `src/main.ts` 把 `audit-log` 改成：只要 `write_file` 的路径不是 `tmp/` 开头就返回 `{ action: 'block', reason: '只允许写入 tmp/ 目录' }`。重启后输入 `测试写文件`，看终端是不是返回了 `[Hook 拦截] 只允许写入 tmp/ 目录` 且没有崩溃。

---

## 实验 8：研究上下文膨胀与三层防线（Mock 模式）

### 操作
```text
/context
sim
/context
defend
/context
/usage
```

### 观察重点
- `sim` 会往内存里硬塞 20 条带大段代码的模拟历史消息，再看 `/context` 时 `Messages` 占比会明显涨起来；
- 跑完 `defend` 后，旧工具结果被截断或被 TTL 清理，`Messages` 占比立刻掉下来，而 `System prompt` 和 `System tools` 基本不变。

---

## 实验 9：把断掉的 `--continue` 历史恢复接通（动手改造）

### 目标
`src/session/store.ts` 已经写好了 `exists()` 和 `load()`，但 `src/main.ts` 启动时依然把 `messages` 初始化成 `[]`。这个实验把它接通。

### 改造要点
打开 `src/main.ts` 第 237 行附近，改成：

```ts
const store = new SessionStore(config.session.id);
const shouldContinue = process.argv.includes('--continue');
let messages: ModelMessage[] = [];
const timestamps = new Map<number, number>();

if (shouldContinue && store.exists()) {
  messages = store.load();
  const now = Date.now();
  messages.forEach((_, idx) => timestamps.set(idx, now));
  console.log(`  [Session] 已恢复 ${messages.length} 条历史消息`);
}
```

### 验收方法
启动 `pnpm start`，随便问一句 `读取 package.json`，然后 `exit` 退出；接着运行 `pnpm run continue`，输入 `/context` 或 `status`，确认消息数不是 `0`（注意：`mock-model.ts` 不会真正阅读早期聊天里的名字，要测跨重启记忆效果建议接真实模型，或者观察 `status` 里的消息计数和 `.sessions/default.jsonl` 不重复追加）。

---

## 实验 10：拆解 RAG 混合检索与权重（Mock / 真实模型对照）

### 操作
在项目临时放一个 `test-rag.md`，里面故意写一个怪异错误码（如 `ERR_XY99_TIMEOUT`）和一段意思相近但不用这个词的中文描述。然后在 `pnpm start` 里输入：

```text
ingest test-rag.md
查资料 ERR_XY99_TIMEOUT
```

### 看清楚 Mock 与真实模型差异
- 在 **Mock 模式**下，`Mock Embedder` 只是按文本字符做确定性哈希，不懂同义词语义；这时精确错误码主要靠 `keywordScore`（BM25）捞回来；
- 接了 **真实 DashScope API Key** 后，再问一句不包含错误码、只描述现象的问题，就能看到 `vectorScore` 怎么把同义切片拉上来；
- 打开 `src/rag/search.ts`，试着轮流修改 `VECTOR_WEIGHT`、`KEYWORD_WEIGHT`、`MMR_LAMBDA`，每次只改一个值对比排名和去重效果（做完记得删掉 `test-rag.md`）。

---

## 实验 11：体验 Memory 持久化与体检（Mock 模式）

### 操作
```text
记住：我写代码偏好使用 TypeScript 严格模式
/memory
/memory search 严格
/lint
```

### 观察重点
去 `.memory/` 目录打开刚刚生成的 `.md` 文件和 `MEMORY.md`，看里面的 Frontmatter 时间戳字段 `lastWriteAt` / `lastReadAt`。然后 `exit` 重启 `pnpm start`，再输入 `/memory`，验证它确实跨重启保留了下来。

---

## 实验 12：写一个自己的 Skill（Mock 模式）

### 操作
新建 `.skills/architecture-review/SKILL.md`：

```markdown
---
name: architecture-review
description: 按固定五步拆解代码模块
when_to_use: 当用户要求分析某个目录或模块架构时
---

分析模块时请严格按下面 5 个小节输出：
1. 入口在哪
2. 维护了哪些状态
3. 依赖了哪些外部模块
4. 失败了会怎样
5. 目前的边界和局限
```

启动 `pnpm start` 后输入：

```text
/skill list
/context
/skill load architecture-review
/context
/skill unload architecture-review
```

注意前面第 03 章提过的小细节：由于 `src/commands/context.ts` 硬编码了 `skillsChars: 0`，激活后多出来的 Token 会加在 `System prompt` 这一项里。如果接了真实模型，激活前后各问一次 `分析 src/cron 模块`，回答结构会明显不一样。

---

## 实验 13：写一个进程内 Plugin（动手改造）

### 目标
参照 `src/plugins/supabase-plugin.ts`，写一个读取 `package.json` 基本信息的 `pkgInfoPlugin`。

### 起步代码骨架
新建 `src/plugins/pkg-info-plugin.ts`：

```ts
import fs from 'node:fs';
import type { PluginDefinition } from './types.js';

export const pkgInfoPlugin: PluginDefinition = {
  name: 'pkginfo',
  version: '0.1.0',
  description: '读取当前项目 package.json 元信息',
  async activate(api) {
    api.registerTools({
      name: 'read_meta',
      description: '返回项目名称、版本和脚本列表',
      parameters: { type: 'object', properties: {}, additionalProperties: false },
      isReadOnly: true,
      isConcurrencySafe: true,
      execute: async () => {
        const pkg = JSON.parse(fs.readFileSync('package.json', 'utf-8'));
        return { name: pkg.name, version: pkg.version, scripts: Object.keys(pkg.scripts || {}) };
      },
    });
    api.log('pkginfo 插件已激活');
  },
  async destroy() {
    console.log('  [plugin:pkginfo] 已清理资源');
  },
};
```

在 `src/main.ts` 的 `availablePlugins` 里加上 `['pkginfo', pkgInfoPlugin]`。启动后输入 `/plugin` 会看到 `pkginfo__read_meta`，再试试 `/plugin unload pkginfo` 看工具是不是跟着注销了。

---

## 实验 14：写一个不依赖飞书的测试 Channel（动手改造）

### 目标
实现一个最小 `ChannelDefinition`，验证不用修改 `agentLoop`，就能多出一条消息通道。

### 起步代码骨架
实现 `src/channels/types.ts` 接口：

```ts
import type { ChannelDefinition, IncomingMessage, OutgoingMessage } from './types.js';

export class MemoryTestChannel implements ChannelDefinition {
  name = 'memtest';
  description = '本地内存测试通道';
  private handler?: (msg: IncomingMessage) => void;
  public outbox: OutgoingMessage[] = [];

  async start() {}
  async stop() { this.outbox = []; }
  onMessage(fn: (msg: IncomingMessage) => void) { this.handler = fn; }
  async send(msg: OutgoingMessage) { this.outbox.push(msg); }

  // 供测试代码主动注入一条消息
  simulateIncoming(senderId: string, text: string) {
    this.handler?.({ channelId: 'test-room', senderId, senderName: senderId, text });
  }
}
```

把它注册进 `gateway` 后，分别用 `userA` 和 `userB` 调用 `simulateIncoming`，检查 `gateway` 内部是不是为两个人隔开了历史消息，且没有复制任何推理代码。

---

## 实验 15：Cron 定时任务（建议接真实模型）

### 为什么需要真实模型
`src/mock-model.ts` 没有内置 `cron_manage` 的触发词。接好真实 `DASHSCOPE_API_KEY` 后启动，输入：

```text
帮我创建一个 id 为 quote-demo、每 20 秒跑一次 random-quote handler 的定时任务
```

等二十多秒看终端自动弹出的通知，再输入：

```text
/cron
/cron logs
```

并打开 `.cron/jobs.json` 和 `.cron/logs.jsonl` 看落盘结构。

**实验做完怎么删掉任务？**
注意 `src/commands/cron.ts` **没有** `/cron remove` 命令（它只有 `list` 和 `logs`），所以要直接对模型说：

```text
把 id 为 quote-demo 的定时任务删掉
```

（或者退出后直接把 `.cron/jobs.json` 清空。）

---

## 实验 16：Sub-Agent 并行分治（建议接真实模型）

### 为什么需要真实模型
同样因为 `src/mock-model.ts` 没写 `spawn_agent` 的触发词，请接真实 `DASHSCOPE_API_KEY` 后输入：

```text
请用子 Agent 并行分析 src/rag、src/memory 和 src/context 三个目录的职责和局限，最后给我一份对比总结
```

跑完后输入 `/agents`，对照终端彩色 `[Agent-1:xxx]` 日志观察：
1. 三个子 Agent 是不是同时启动、各有各的 Step 计数？
2. 如果你刻意派 4 个子任务，超过 `maxConcurrent: 3` 的第 4 个是怎么被拒绝的？
3. 子 Agent 为什么没法再往下派生孙子 Agent？

---

## 实验 17：一次复杂任务的端到端复盘（建议接真实模型）

挑一个既需要查知识库、又需要读代码、最后汇总回答的问题（例如：`根据部署文档里的监控告警要求，结合当前代码实现，说说还缺哪些能力？`）。

跑完后翻开终端日志、`.sessions/default.jsonl`、`.traces/<本轮>.jsonl` 和 `.usage/today.jsonl`，把每一步填进下面这张表里：

| Step | 发给模型的上下文概览 | 模型决定调用什么工具 | 工具返回摘要 | 本步 Token | 落盘了什么状态 |
| --- | --- | --- | --- | --- | --- |
| 1 | ... | `rag_search` | ... | ... | ... |
| 2 | ... | `read_file` | ... | ... | ... |
| 3 | ... | 无工具，最终回答 | — | ... | ... |

最后试着回答 4 个终极问题——能答上来，说明你已经真正吃透这套源码了：
1. 这一轮里，哪些决定是**模型**做的，哪些约束是**运行时代码**替你兜底的？
2. 哪些数据写进了硬盘，哪些还只留在内存里？
3. 如果进程在 Step 2 刚跑完工具那一秒突然断电，重启后会丢掉什么？
4. 如果今天要把这个项目给 100 个同事同时用，最先炸掉的三个模块是哪三个？
