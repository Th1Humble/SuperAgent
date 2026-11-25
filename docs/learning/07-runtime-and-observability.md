# 07. 启动、配置、命令行与可观测性

前面把所有积木块都看完了，这一章回到入口，搞清楚三件很实际的事：**配置怎么读进来才不会踩坑**、**命令分发和启停顺序是什么**、**线上或实验跑歪了怎么靠 Trace 排查**。

## 7.1 入口与配置加载的微妙顺序

看 `src/index.ts`：总共只有 6 行，看命令行参数是不是 `init`，再决定动态 `import('./config/init.js')` 还是 `import('./main.ts')`。之所以用动态导入，是因为 `main.ts` 顶层有很多立即执行的初始化代码，跑配置向导时没必要把那些东西全拉起来。

进入 `main.ts` 后，前两行就决定了整个系统的运行模式：
1. `import 'dotenv/config'` 把 `.env` 读进 `process.env`；
2. `loadConfig()` 读取 `super-agent.config.json`（没有就用 Zod 默认配置），并把字符串里的 `${UPPER_CASE}` 替换成对应环境变量。

> **最容易踩坑的环境变量逻辑（`src/config/loader.ts`）**：
> - 如果你在配置里写了 `"apiKey": "${DASHSCOPE_API_KEY}"`，**而 `.env` 里根本没有这一行**（值为 `undefined`），`substituteEnvVars` 会打印一条警告并**保留原字符串 `"${DASHSCOPE_API_KEY}"`**。因为字符串非空，程序反而会误以为“有 Key”，去请求 DashScope 然后报认证错误；
> - 只有当 `.env` 里显式写了 `DASHSCOPE_API_KEY=`（值为空字符串 `""`）时，替换结果才是 `""`，程序才会安心进入 Mock 模式。

### Schema 里的字段 ≠ 运行时已经生效
`src/config/schema.ts` 为完整系统预留了 10 大块配置，但对照 `src/main.ts` 实际只读了这几项：
- `config.model.*`
- `config.memory.dataDir`
- `config.cron.dataDir`
- `config.agents.maxSpawnDepth` / `maxConcurrent`
- `config.session.id`（注意第 237 行创建 `SessionStore('default')` 时连这里都没传进去，只有 Trace 用了 `config.session.id`）

其余像 `plugins[].enabled`、`channels.feishu.*`（实际直接读 `process.env.FEISHU_*`）、`security.defaultRole`、`rag.docsDir`、`usage.trackingFile`、`agents.defaultTimeout` 目前都属于**“Schema 占了位、主程序还没接线”**的状态。看源码时想确认某个配置生效没有，直接全局搜 `config.<字段>` 最靠谱。

## 7.2 终端命令分发器：责任链模式

终端输入一行字后，怎么区分它是“跟模型聊的话”还是“控制系统的指令”？

`src/commands/index.ts` 用了一个极简的责任链：把 11 个命令模块的处理函数摊平进一个数组，按顺序挨个问“是不是归你管”：
- 返回 `false`：不归我管，继续问下一个；
- 返回 `true`：我同步处理完了，外面紧接着调 `ask()` 等下一行输入；
- 返回 `'async'`：我要干异步活（比如 `ingest`、`/dream`、`/<skill-name>`），等我活干完会自己调 `ctx.ask()`，外层先别急着再问。

如果全问完都返回 `false`，才把它当成普通对话交给 `agentLoop()`。

初学者一定要分清 **Command（终端命令）** 和 **Tool（工具）** 的区别：
- **Command** 是**你**手动敲给运行时看的（如 `/context`、`/role`），**不进模型上下文、不花 Token**；
- **Tool** 是写进说明书发给**模型**、由模型自己挑着调的（如 `read_file`、`rag_search`），模型调完之后才产生结果回到对话里。

## 7.3 启动顺序与关闭缺口

`startAgent()` 的实际启动顺序是（`src/main.ts`）：

```text
连接 Mock MCP → 加载 Plugins → 启动 Channels (Dashboard/飞书) → 加载并启动 Cron
  → 初始化 Session / Usage / readline → 打印欢迎提示 → 扫描 docs/*.md 构建 RAG → ask()
```

看这个顺序能发现两个工程上的小细节：
1. **RAG 构建放在了 Channel 和 Cron 启动之后**：理论上如果刚启动的一秒钟内飞书刚好进来一条消息，此时知识库还没导入完（生产项目通常会先把核心基础依赖准备好再开外部端口）；
2. **退出清理只在 `exit`/空输入分支里**：输入 `exit` 会依次停止 Cron、Channel 和 Plugin，但目前没有显式调用 `registry.closeAllMCP()`（接入真实子进程 MCP 后容易留僵尸进程），也没有监听 `SIGINT`（`Ctrl+C`）。所以建议养成用 `exit` 退出的习惯。

## 7.4 三份运行记录别弄混：Session、Trace、Usage

项目跑起来后，根目录会冒出 `.sessions/`、`.traces/`、`.usage/`、`.cron/`、`.memory/` 五个隐藏目录（都在 `.gitignore` 里）。前三个最容易弄混：

| 文件目录 | 它回答什么问题 | 什么时候写 |
| --- | --- | --- |
| `.sessions/default.jsonl` | **聊了什么？**（用户和助手的往来消息） | 每次终端发消息和一轮结束时 |
| `.traces/<id>.jsonl` | **模型每步到底看见了啥、怎么一步步决策的？** | 每一步模型调用前（`step_started`）和完成后（`step_completed`） |
| `.usage/today.jsonl` | **一共花了多少 Token、缓存省了多少钱？** | 每个 Step 结束时追加一行账单 |

### 为什么排查 Bug 主要看 Trace？
想象一个典型抓狂场景：“我明明记得提示词里写过某某规则，为什么模型还是调错了工具？”

光看 `.sessions/` 是看不出来的，因为 Session 里没有 System Prompt，也没有模型出错重试的过程。打开 `.traces/` 对应的 `.jsonl` 文件，你会清清楚楚看到：
- `step_started.context.system`：这一步发给模型的**逐字逐句完整 System Prompt**；
- `step_started.context.messages`：当时传进去的历史消息数组长什么样；
- `step_attempt_failed`：第几次尝试撞到了什么网络报错；
- `step_completed.output` & `usage`：模型吐出了什么原始结构化部件、耗时多少毫秒。

直接用项目自带的摘要命令行看一眼最方便：
```bash
pnpm run trace:inspect .traces/你的文件名.jsonl
```

当然，`src/trace/recorder.ts` 也有两个要注意的地方：
1. **脱敏只是按字段名正则替换**：`SECRET_KEY` 会把键名含 `token`、`password`、`apiKey` 的字段值替换成 `[REDACTED]`，但如果你用 `read_file` 把 `.env` 的文件内容读进字符串正文里了，它还是会原样躺进 Trace；
2. **只覆盖了普通终端主循环**：通过飞书/Dashboard（`ChannelGateway`）、Cron 定时任务和 Sub-Agent 内部跑的模型 Step，目前都没有传 `trace` 进去，出问题只能看终端控制台日志。

## 7.5 一份顺手的故障排查清单

- **现象：模型死活不调用某个工具**
  1. 输入 `/role` 看当前角色是不是误切成了 `guest` 或 `collaborator`；
  2. 看启动日志确认工具到底有没有注册进 `Registry`；
  3. 检查是不是延迟工具（`shouldDefer: true`），模型需要先调 `tool_search`；
  4. 如果是 Mock 模式，核对你的提问有没有命中 `src/mock-model.ts` 的关键词；
  5. 打开 `.traces/*.jsonl` 看 `step_started` 里消息是不是已经乱了。
- **现象：问项目文档问题，模型完全没查知识库**
  1. 输入 `/rag` 看片段数是不是 `0`，确认文档是不是放在 `docs/` 第一层（子目录不会自动导入）；
  2. 注意 Mock Embedder 不懂同义词语义，提问尽量带文档里的关键词（或接真实 Key）。
- **现象：改了 `super-agent.config.json` 但好像没生效**
  1. 看启动第一屏有没有打印 `✓ 已加载 super-agent.config.json`；
  2. 全局搜 `config.<你改的字段>`，确认它是不是属于上面提到的“Schema 预留但 `main.ts` 还没读取”的配置项。

## 7.6 动手试一试（对照三份持久化文件）

在终端跑一次需要调工具的对话（比如 `读取 package.json`），然后分别查看刚生成的 `.sessions/default.jsonl`、终端末尾打印的 `.traces/...jsonl` 以及 `.usage/today.jsonl`（也可以用 `pnpm run trace:inspect` 看摘要），用自己的话对比这三份文件各自记录了什么、没记录什么。

## 7.7 本章自检

1. 为什么在 `super-agent.config.json` 里写了 `"${DASHSCOPE_API_KEY}"`，但没建 `.env` 文件时程序会报错而不是走 Mock？
2. 终端输入 `/context` 和模型调用 `read_file`，两者最本质的区别是什么？
3. 如果 `LocalTraceRecorder` 写入磁盘时突然没权限报错了，会导致整个 Agent 对话崩溃吗？
4. 飞书发来的消息出错了，去 `.traces/` 目录下找得到它的轨迹吗？为什么？
5. 当前按 `exit` 退出和直接按 `Ctrl+C` 退出有什么区别？

<details>
<summary>参考思路（点击展开）</summary>

1. 当环境变量为 `undefined` 时，`substituteEnvVars()` 会保留原字面量 `"${DASHSCOPE_API_KEY}"`。因为是非空字符串，`createModel()` 认为已有 Key 从而连接真实接口，最终因 Key 无效报错。只有 `.env` 里定义了空值 `DASHSCOPE_API_KEY=`，替换后才是 `""` 从而进入 Mock。
2. `/context` 是用户敲给 CLI 分发器的本地控制命令，不走模型、不进 `messages`；`read_file` 是暴露给模型的工具，由模型在推理过程中自主决定调用并产生上下文消耗。
3. 不会。`recorder.ts` 的 `write()` 在首次写失败后会把 `this.writeFailed` 设为 `true` 并打印警告，后续不再写入，体现“可观测性失败不阻塞主业务”的设计原则。
4. 找不到。目前只在 `main.ts` 的普通终端主循环里创建并传入了 `LocalTraceRecorder`，`ChannelGateway`、`CronService` 和 `spawnAgent` 调用 `agentLoop` 时都没有传入 `trace` 实例。
5. 输入 `exit` 会主动调用 `cronService.stop()`、`gateway.stopAll()` 和 `pluginManager.unloadAll()` 做清理；直接按 `Ctrl+C` 会触发未处理的信号直接退出进程（另外即便走 `exit`，目前也还没调用 `registry.closeAllMCP()`）。

</details>
