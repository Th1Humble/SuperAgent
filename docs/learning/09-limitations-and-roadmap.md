# 09. 当前实现、限制与演进路线

一份可信的学习文档不仅要讲“有什么”，还要明确“没有什么”。本章以当前源码为准，帮助读者避免把教学实现误当成生产承诺。

## 9.1 能力状态总表

| 能力 | 当前状态 | 说明 |
| --- | --- | --- |
| Mock/真实模型切换 | 已接入 | 由 `model.apiKey` 是否为空决定 |
| 流式 Agent Loop | 已接入 | 支持工具、多 Step、重试和预算 |
| 循环检测 | 已接入 | 模块级共享历史，并发隔离不足 |
| 文件/Shell/Web 工具 | 已接入 | 教学实现，安全与性能边界明显 |
| Tool Search | 已接入 | 精确工具名发现，不是语义检索 |
| MCP | 部分接入 | 默认使用 Mock GitHub Client |
| Session 写入 | 已接入 | 普通 CLI 消息写 JSONL |
| Session 恢复 | 未接通 | `load()` 已有，continue 未使用 |
| Prompt Pipe | 已接入 | 主 System Prompt 由管道构建 |
| Microcompact/LLM 摘要 | 模块已实现 | 未自动接入主 Agent Loop |
| Context Defense | 手动接入 | 主要由调试命令触发 |
| Usage | 部分接入 | CLI Agent 记录，路径配置未使用 |
| RAG | 已接入 | 向量库仅在内存，Mock/真实 Embedding |
| Memory | 已接入 | Markdown 文件持久化 |
| Dream | 已接入 | 自动整理有误删风险 |
| Skill | 已接入 | 简化 Frontmatter 解析 |
| Plugin | 部分接入 | Supabase 为 Mock，enabled 配置未消费 |
| Channel | 已接入 | 飞书 + Dashboard；会话仅内存 |
| RBAC | 已接入 | 工具名级过滤，配置默认角色未接线 |
| Bash 风险识别 | 已接入 | 正则规则，不是安全沙箱 |
| Hook | 已接入 | Hook 异常 fail-open |
| Cron | 已接入 | 依赖进程常驻，Agent timeout 未完整接线 |
| Sub-Agent | 已接入 | 单层派发，记录在内存 |
| Trace | 部分接入 | 只覆盖普通 CLI 主循环 |
| 正式部署 | 未提供 | 无 build、容器、进程守护和健康检查 |

## 9.2 课程与仓库差异

### RAG 存储

课程目录描述 `sqlite-vec + BM25`，当前仓库是内存 `VectorStore + BM25-like + MMR`，没有 SQLite 依赖。重启会重新索引。

### 对话恢复

课程包含持久化概念，代码也实现 `SessionStore.load()`，但入口没有根据 `--continue` 恢复。

### 上下文压缩

`compressor.ts` 已完整展示机械压缩和 LLM 摘要，但默认 Agent Loop 不调用。当前不会在上下文变长时自动压缩。

### MCP 和 Supabase

默认 GitHub MCP 与 Supabase 都是 Mock/占位行为，不能视为真实外部系统连接。

### 部署资料

`docs/deployment-guide.md` 是 RAG 示例，提到的 PostgreSQL、迁移、PM2 和 build 不属于当前项目实现。

## 9.3 配置接线缺口

Schema 和向导已经为完整系统预留很多字段，但主程序未全部消费：

- `plugins[].enabled/config`
- `channels.feishu.*`
- `security.defaultRole/auditLog/bashTimestamp`
- `rag.enabled/docsDir`
- `cron.enabled`
- `agents.defaultTimeout`
- `usage.trackingFile`

演进建议：把每个模块的创建放入工厂，统一从已验证 Config 取值；尽量不要同时维护 Config 和直接 `process.env` 两条配置路径。

## 9.4 安全缺口

- 文件工具没有工作区根目录沙箱；
- Dashboard 没有认证；
- Shell 使用宿主机权限执行；
- 正则分类可以绕过；
- `calculator` 使用 `new Function()`；
- Web 内容没有完整 Prompt Injection 隔离；
- Trace 只按字段名脱敏；
- 插件和 MCP 工具缺少细粒度信任等级；
- 写操作缺少确认、幂等和事务模型。

生产化优先级应是：隔离执行环境、认证入口、最小权限、人工批准高风险行为、完整审计。

## 9.5 并发与状态缺口

- Agent Loop 的循环检测历史是模块级共享状态；
- Channel 同一用户同时发送消息时没有串行队列；
- 子 Agent 绕开 Registry 锁，写工具可能竞争；
- Session 追加写没有多进程锁；
- Memory 索引和条目更新不是事务；
- Cron 只适合单进程，多个实例会重复调度；
- Plugin 工具变化时正在运行的请求缺少版本隔离。

演进时要先定义并发模型：单进程单用户、单进程多用户，还是多实例服务。不同模型需要完全不同的锁和存储方案。

## 9.6 数据与恢复缺口

- VectorStore 不持久化；
- Channel 会话不持久化；
- Sub-Agent 记录不持久化；
- Usage 启动后不读取历史聚合；
- Trace 没有索引和保留策略；
- Session load 不恢复 timestamps；
- Cron 不补跑离线期间错过的任务；
- JSONL 没有轮转、压缩和损坏恢复工具。

## 9.7 测试缺口

仓库目前主要只有 `context/defense.spec.ts`。高价值测试方向：

1. Agent Loop：无工具、单工具、多工具、重试、取消、预算；
2. Registry：Role/Profile/Deferred 组合，Hook 顺序，锁行为；
3. RAG：切片边界、归一化、混合排名、MMR；
4. Memory：并发写、索引上限、过期、路径校验；
5. Cron：解析、重入、失败禁用、重启恢复；
6. Channel：会话隔离、并发消息和发送失败；
7. 配置：环境替换、空值、未定义变量和字段接线。

## 9.8 推荐演进顺序

### 阶段一：让现有功能一致

- 接通所有配置字段；
- 修正默认角色类型；
- 实现 continue；
- 在退出时关闭 MCP；
- 给 SIGINT/SIGTERM 增加统一 shutdown；
- 使用配置中的 docsDir、trackingFile 和 timeout。

### 阶段二：补齐可观测和测试

- 给 Channel、Cron、Sub-Agent 增加 Trace；
- 为工具调用记录独立事件和耗时；
- 增加关键模块测试；
- 统一结构化日志和 request/run ID。

### 阶段三：加固安全和并发

- 工作区沙箱；
- 异步 Shell Runner；
- 高风险操作批准机制；
- 用户级执行队列；
- 公平锁和可取消等待；
- 外部输入隔离与内容安全策略。

### 阶段四：持久化和服务化

- 持久向量数据库；
- Session/Memory/Cron 的事务存储；
- 多实例调度协调；
- Dashboard 鉴权；
- 健康检查、容器和进程管理；
- 数据保留与删除机制。

## 9.9 不应过早做的事情

- 在没有评测集前微调 RAG 权重；
- 在没有状态模型前直接多实例部署；
- 仅靠 Prompt 声明安全；
- 把所有能力都做成工具并一次性暴露；
- 为追求“自主”取消步数、预算和审批；
- 在 Session 恢复没做好前堆叠更多长期状态。

## 9.10 如何继续学习

建议按以下问题驱动继续：

1. **正确性**：模型做错决定时，运行时如何检测和恢复？
2. **安全性**：工具调用的最小权限和批准边界在哪里？
3. **可观测性**：如何重放一次任务并解释每个决定？
4. **评测**：如何量化工具选择、RAG 召回和任务成功率？
5. **持久性**：哪些状态必须跨会话、跨进程、跨版本？
6. **并发性**：多个用户和多个 Agent 如何共享资源？
7. **经济性**：怎样在质量、延迟和 Token 成本间权衡？

项目的价值正是它足够小，可以清楚看到这些问题从哪里出现；又足够完整，可以在同一个代码库中尝试解决。
