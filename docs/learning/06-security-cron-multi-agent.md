# 06. 安全、Cron 与 Multi-Agent

## 6.1 为什么越自主越需要边界

聊天程序只生成文字，风险主要是回答错误；Agent 可以写文件、执行命令、访问网络、定时运行和派发子任务，风险变成真实副作用。

本项目从四个层次约束执行：

```text
模型可见性：Role 过滤工具
执行前：Bash 分类 + Pre Hook
执行中：并发锁、超时、步数和 Token 预算
执行后：Post Hook、Trace、日志
```

这些是互补机制。任何单层都不能独立提供完整安全保证。

## 6.2 角色权限

定义三种角色：

| 角色 | 权限 |
| --- | --- |
| `owner` | 所有工具 |
| `collaborator` | 除 `bash` 外的所有工具 |
| `guest` | 天气、计算、只读文件/目录/搜索和 RAG |

`canUseTool()` 根据工具名判断，`getActiveTools()` 在把 Schema 发给模型之前过滤。这比让模型“记住不要调用”更可靠，因为无权工具根本不在本轮工具集合中。

局限：权限按名称白名单/黑名单，不理解参数范围。例如 guest 能读取文件时，没有路径级沙箱；插件新增工具也可能因宽松规则自动暴露给 collaborator。

命令：

```text
/role
/role owner
/role collaborator
/role guest
```

## 6.3 配置与实际默认角色

Schema 的 `security.defaultRole` 默认值是字符串 `developer`，但真正的 Role 类型只允许 owner/collaborator/guest；`main.ts` 也没有把该配置设置到 Registry。Registry 当前默认 `owner`。

因此配置文件中的 `defaultRole` 目前不会控制运行时角色。这是需要在生产化前修正的接线缺口。

## 6.4 Bash 风险分类

`classifyBashCommand()` 用正则分三级：

- `safe`：未命中规则；
- `moderate`：删除、移动、改权限、kill、git push 等，打印警告但继续；
- `dangerous`：强制递归删除、sudo、格式化磁盘、远程脚本管道执行等，直接拒绝。

分类发生在工具执行器之前，即使模型已经生成调用，危险命令也不会进入 `execSync()`。

正则分类只能作为纵深防御，不能视为 Shell 沙箱。命令可以通过别名、编码、脚本文件、解释器或未覆盖语法绕过。真正安全的执行环境还需要：

- 文件系统/容器隔离；
- 工作目录和可执行程序白名单；
- 网络限制；
- 系统权限降级；
- 人工审批；
- 超时、CPU、内存、输出限制。

## 6.5 Hook Pipeline

Pre Hook 可以：

- `allow`：继续；
- `block`：返回拦截结果；
- `modify`：替换输入。

Post Hook 可以修改输出。多个 Hook 按注册顺序执行；前一个修改的值会传给后一个。Hook 自身异常会记录但不会中断主工具调用，这属于 fail-open 行为。

适合的用途：

- 审计日志；
- 敏感路径拦截；
- 参数规范化；
- 输出脱敏；
- 组织策略检查。

对关键安全策略，生产系统通常需要明确选择 fail-open 还是 fail-closed，不能统一吞掉 Hook 异常。

## 6.6 Cron 的任务模型

`CronJobConfig` 包括：

- ID 和名称；
- schedule 表达式和 scheduleType；
- enabled；
- payload；
- timeout、maxRetries；
- source：配置或运行时创建。

Payload 有两类：

```text
agent   → 到点后用 prompt 启动一次 Agent Loop
handler → 执行预先注册的固定处理器
```

当前内置 handler 只有 `random-quote`，其他 handler 会返回占位说明。

## 6.7 三种调度表达式

`parseSchedule()` 支持：

- Cron 表达式，如 `0 9 * * *`；
- Interval，如 `every 10m`、`every 2 hours`；
- Once，使用可被 `Date` 解析的时间。

Croner 用于计算 Cron 下次触发时间。最终都转换为一次 `setTimeout()`，任务完成后再安排下一次，而不是用永久 `setInterval()`。

这样可避免任务执行时间较长时固定间隔不断叠加，但 Interval 的实际周期会包含执行耗时。

## 6.8 Cron 生命周期

启动时：

1. `CronStore.init()` 创建数据位置；
2. `load()` 读取启用任务；
3. `setExecutor()` 注入 Agent 执行和通知函数；
4. `start()` 为每个任务安排 Timer。

执行时：

1. 防止同一任务重入；
2. 记录 startedAt；
3. 根据 Payload 运行；
4. 成功清零连续失败；
5. 失败累加，达到 `maxRetries` 后禁用；
6. 写 `.cron/logs.jsonl`；
7. 发送本地通知文本；
8. 重复任务重新调度，一次任务删除。

## 6.9 Cron 持久化边界

`.cron/jobs.json` 保存任务配置，`.cron/logs.jsonl` 保存执行记录。运行时创建的任务会和配置来源任务合并保存。

调度依赖当前 Node.js 进程持续运行：进程退出后没有后台守护者执行任务。重启后可以重新加载任务，但错过的触发不会自动补跑。

当前 `timeout` 会传给 `runAgentPrompt`，但 `main.ts` 注入的实现没有把它转换成 AbortSignal 传给 Agent Loop，因此 Cron Agent 任务的超时约束并未完整生效。

## 6.10 Sub-Agent 要解决什么

主 Agent 可以把独立任务交给子 Agent：

- 隔离上下文，减少主消息污染；
- 并行处理互不依赖的任务；
- 让主 Agent 聚合多个结果；
- 对每个子任务设置独立步数和超时。

它不是“换一个模型人格”，而是用相同模型、工具和系统规则开启一组独立消息。

## 6.11 SubAgentRegistry

Registry 保存每次运行：

- ID；
- task；
- depth；
- status；
- startedAt/finishedAt；
- result 或 error。

还维护：

- `maxSpawnDepth`，默认 1；
- `maxConcurrent`，默认 3；
- 活跃任务数量。

记录只在内存中，重启会丢失。`/agents` 用它展示当前进程历史。

## 6.12 子 Agent 执行流程

`spawnAgent()`：

1. 检查深度和并发限制；
2. 在 Registry 建立 running 记录；
3. 创建独立 `messages = [{ role: 'user', content: task }]`；
4. 创建 AbortController 和超时 Timer；
5. 构建工具集合，排除 `spawn_agent` 防止无界递归；
6. 最多执行 10 个 Step；
7. 最后一步强制 `toolChoice: 'none'`，要求生成文本结果；
8. 从最后一条 assistant 提取文本；
9. 标记完成、失败或超时。

子 Agent 使用 `toAISDKFormatUnlocked()`，绕开主 Registry 的执行锁，目的是允许多个子 Agent 真正并行。代价是它们如果同时调用写工具，可能发生竞争。

## 6.13 并行派发

`spawnParallel()` 先计算：

```text
available = maxConcurrent - activeCount
```

只执行可用数量，超出的任务返回拒绝结果；执行部分使用 `Promise.all()`。主 Agent 得到的是每个任务和结果组成的数组，再负责综合。

适合并行：

- 比较多个互不依赖的框架；
- 分别分析不同目录；
- 独立收集多个资料源。

不适合并行：

- 多个任务修改同一文件；
- 后一个任务依赖前一个输出；
- 共享不可并发的外部资源；
- 需要统一事务的写操作。

## 6.14 当前嵌套深度的真实行为

`SpawnContext` 有 `currentDepth`，并且类型/Registry 支持最大深度。但是子 Agent 的工具集合明确排除了 `spawn_agent`，因此当前子 Agent 不能继续派孙 Agent。深度配置更多是架构预留，而非默认实现中的多层树。

## 6.15 安全组合示例

当 guest 请求“用 bash 删除文件”时：

1. Role 过滤已使 `bash` 不可见，模型通常无法生成合法调用；
2. 即使通过其他路径构造调用，Bash 分类仍会检查危险模式；
3. Pre Hook 还可以按路径阻止；
4. Trace/审计可记录尝试。

这就是纵深防御：多层分别防错误、绕过和事后追溯。

## 6.16 实践

```text
/role guest
读取 package.json
执行 pwd
/role owner
/hooks
/cron
帮我创建一个每 10 分钟输出随机名言的任务
/cron logs
并行比较 Hono、Fastify 和 Express
/agents
```

不要为了测试危险分类而输入真实破坏性命令；阅读 `bash-classifier.ts` 的单元规则即可。

## 6.17 本章自检

1. Role 过滤和 Bash 分类分别发生在哪一层？
2. Hook 异常当前是 fail-open 还是 fail-closed？
3. Cron 为什么要求进程持续运行？
4. 当前 Cron timeout 是否完整中止 Agent？
5. 子 Agent 为何要有独立消息？
6. 为什么子 Agent 并行写同一文件仍然危险？
