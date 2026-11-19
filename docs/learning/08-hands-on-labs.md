# 08. 实践教程：把“看懂”变成“做过”

本章按风险和复杂度递增。实验默认使用 Mock 模式，不要求 API Key。涉及源码修改的练习建议先创建自己的 Git 分支；每次只改一个变量，观察现象后再还原或提交。

## 实验 0：建立基线

### 目标

确认项目、Dashboard、RAG 和基本命令能够启动。

### 操作

```bash
pnpm install
pnpm start
```

看到 `You:` 后输入：

```text
你好
/rag
/context
/role
exit
```

浏览器打开 `http://localhost:3000`，从 Dashboard 发送“你好”。

### 观察

- 没有 `super-agent.config.json` 时提示使用默认配置；
- 没有 API Key 时模型 ID 是 Mock；
- MCP 显示注册 Mock GitHub 工具；
- Supabase 显示 Mock 模式；
- `docs/` 第一层 Markdown 被导入；
- Dashboard 回复也会触发终端中的 Step 输出。

### 验收

你能说明为什么 Dashboard 可以使用和 CLI 相同的工具，却没有写入同一个 Session Trace。

## 实验 1：追踪一次不调用工具的请求

### 操作

输入：

```text
用一句话介绍你自己
```

记录：Step 数量、是否有 `[调用]`、最终 Trace 路径。然后运行：

```bash
pnpm run trace:inspect .traces/<trace-file>.jsonl
```

### 观察

无工具调用时，模型输出文本后 `hasToolCall=false`，Agent Loop 在第一个 Step 结束。

### 深入

直接打开 Trace JSONL，找到：

- `trace_started.model`
- `step_started.context.system`
- `step_completed.usage`
- `trace_finished.status`

## 实验 2：追踪一次工具调用

### 操作

```text
读取 package.json，告诉我 name、依赖数量和启动脚本
```

### 预期

通常至少两个 Step：第一步调用 `read_file`，第二步观察结果并回答。

### 思考

如果模型在工具调用前输出了一段文字，`fullText` 会收集它；但最终消息历史必须使用 `result.response.messages`，因为其中还含结构化工具部件。

## 实验 3：观察并行与独占

### 操作

```text
同时查询北京天气，并计算 (18 + 24) * 3
```

然后尝试一个包含读文件和写文件意图的任务，但不要让它修改重要文件，可以要求写入一个专用临时练习文件。

### 观察

- 天气和计算器标记为并发安全；
- 写文件是独占工具；
- Registry 负责锁，而不是 Agent Loop 自己判断。

### 延伸修改

暂时把一个只读工具的 `isConcurrencySafe` 改为 `false`，重复同样任务，比较执行顺序。理解后恢复。

## 实验 4：增加第一个工具

### 目标

增加 `project_summary`，返回文件数和顶层目录列表。

### 步骤

1. 新建 `src/tools/project-tools.ts`；
2. 按 `ToolDefinition` 定义 name、description、parameters 和 execute；
3. 标记只读和并发安全；
4. 在 `src/tools/index.ts` 导入并加入 `allTools`；
5. 启动后观察启动日志和 `/role` 的活动工具数量确认注册；
6. 用自然语言要求模型总结项目结构。

### 设计检查

- Schema 是否拒绝多余字段？
- 输出是否有数量上限？
- 是否跳过 `node_modules` 和 `.git`？
- 路径不存在时返回什么？
- 工具描述有没有说明使用场景？

## 实验 5：理解 Tool Search

### 操作

启动后查看：

```text
/context
/context
```

尝试让模型查询模拟 GitHub issue。观察它是否先调用 `tool_search` 获取 `mcp__github__...` 的完整定义。

### 修改实验

把一个本地只读工具暂时标记为 `shouldDefer: true`，设置 `searchHint`，比较 `/context` 中活动和延迟工具 Token。

### 结论

延迟发现优化的是 Schema 上下文，不是工具执行速度。

## 实验 6：验证角色过滤

### 操作

```text
/role owner
/role
/role guest
/role
读取 package.json
执行 pwd
```

### 观察

guest 仍可读文件，但 `bash` 不在可见集合。权限是在工具暴露阶段生效，而不只是 System Prompt 中的一句要求。

### 延伸

思考路径级权限应放在哪里：Role 规则、Pre Hook、文件工具内部，还是操作系统沙箱？答案通常是多层共同负责。

## 实验 7：观察 Hook

### 操作

```text
/hooks
用 bash 执行 pwd
```

观察 Bash 输出前的 ISO 时间戳。再让 Agent 向专用练习文件写一行，观察 audit log。

### 修改实验

增加一个 Pre Hook：当写入路径不在 `tmp/` 时 block。验证模型收到的是 Hook 拦截结果，而不是进程崩溃。

## 实验 8：研究上下文增长

### 操作

连续执行几次文件读取和追问，每次运行：

```text
/context
/usage
```

记录 System、Tools、Messages 的变化。

然后运行调试防线命令：

```text
defend
/context
```

### 观察

工具结果截断和 TTL 会改变 Messages；System 和 Tool Schema 通常保持不变。

### 延伸实现

在不改 Agent Loop 的前提下，写一个命令手动调用 `microcompact()` 并替换 `ctx.messages`。比较机械压缩前后。

## 实验 9：接通历史恢复

### 目标

理解“写了 Session”与“恢复 Session”之间还缺什么。

### 设计步骤

1. 在启动时识别 `--continue`；
2. 创建 `SessionStore` 后调用 `exists()` 和 `load()`；
3. 将返回值作为初始 `messages`；
4. 恢复或重建 timestamps；
5. 输出恢复消息数量；
6. 避免再次重复追加已加载消息。

### 验收场景

第一次启动告诉 Agent 一个临时名字，退出；用 continue 启动后询问名字。区分“从 Session 恢复”与“从 Memory 找回”。

## 实验 10：RAG 检索拆解

### 操作

准备一个小型 Markdown 文档，包含一个独特错误码和一段同义描述，用：

```text
ingest <文档路径>
```

分别用精确错误码和自然语言提问。

### 观察

- 精确错误码更依赖关键词路径；
- 同义问题更依赖向量路径；
- 最终结果中可同时看到 `vectorScore` 和 `keywordScore`；
- 相似段落过多时 MMR 会增加多样性。

### 参数实验

在 `rag/search.ts` 中分别尝试调整：

- `VECTOR_WEIGHT`
- `KEYWORD_WEIGHT`
- `CANDIDATE_MULTIPLIER`
- `MMR_LAMBDA`

每次只改一个值，用同一组查询比较排名。不要只凭一条查询下结论。

## 实验 11：Memory 生命周期

### 操作

要求 Agent 记住一个偏好，然后：

```text
/memory
/memory search <关键词>
/lint
```

打开 `.memory/MEMORY.md` 和具体条目，观察 Frontmatter 时间字段。

### 思考

- 读取为何也修改文件？因为更新 `lastReadAt`；
- 记忆为什么是线索？因为仓库和用户偏好会变化；
- 什么信息不应写入？密钥、可从代码推导的信息、临时输出。

## 实验 12：写一个 Skill

### 目标

创建 `.skills/architecture-review/SKILL.md`，要求模型按“入口、状态、依赖、失败、边界”五部分分析模块。

### 操作

启动后：

```text
/skill list
/skill load architecture-review
请分析 src/cron 模块
/skill unload architecture-review
```

比较激活前后的回答结构和 `/context` 中 System Prompt 占用。

## 实验 13：写一个 Plugin

### 目标

编写只读的项目元数据插件，注册 `<plugin>__package_info` 工具。

### 要点

- 使用 Plugin API，不直接访问全局 Registry；
- 在 activate 中注册工具；
- 配置支持 `${ENV_VAR}`；
- destroy 中演示资源清理日志；
- 通过 `/plugin` 查看工具前缀；
- unload 后确认工具消失。

## 实验 14：实现一个内存 Channel

### 目标

不接第三方平台，先实现 `ConsoleTestChannel`：它可以由测试代码触发 IncomingMessage，并收集 OutgoingMessage。

### 验收

- Gateway 注册并启动；
- 两个 senderId 有独立消息数组；
- 同一 senderId 能多轮追问；
- Channel 内没有复制模型和工具逻辑；
- stop 后释放所有资源。

## 实验 15：Cron 任务

### 操作

创建一个短间隔的随机名言任务，观察：

```text
/cron
/cron logs
```

检查 `.cron/jobs.json` 和 `.cron/logs.jsonl`。完成实验后用命令删除任务，避免持续触发。

### 故障实验

设计一个总是失败的测试 handler，观察连续失败次数达到阈值后是否自动禁用。不要用真实外部副作用作为失败实验。

## 实验 16：Sub-Agent 分治

### 操作

```text
请并行分析 src/rag、src/memory 和 src/context，各自总结职责和边界，最后给出对比
```

然后查看：

```text
/agents
```

### 观察

- 每个子 Agent 有独立消息；
- 最大并发限制如何裁剪任务；
- 主 Agent 如何拿到结果并综合；
- 子 Agent 工具集合为什么排除 `spawn_agent`。

## 实验 17：完整请求复盘

选择一个会触发 RAG、文件读取和最终总结的请求。建立表格：

| 时间/Step | 输入上下文 | 模型决定 | 工具 | 工具结果 | Token | 状态变化 |
| --- | --- | --- | --- | --- | --- | --- |

资料来源：终端、Trace、Session、Usage、RAG 源文档。最终用自己的话回答：

1. 模型做了哪些决定？
2. 运行时做了哪些决定？
3. 哪些结果被持久化？
4. 如果进程此刻崩溃，哪些状态会丢？

完成这个实验，才算真正把整个项目串起来。
