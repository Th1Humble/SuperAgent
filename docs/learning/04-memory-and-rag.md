# 04. 记忆与 RAG：两种不同的“知道”

## 4.1 先区分问题

RAG 和 Memory 都会给 Agent 补充模型参数之外的信息，但来源和生命周期不同。

| 维度 | RAG | Memory |
| --- | --- | --- |
| 主要来源 | 文档、知识库、外部资料 | 对话中形成的偏好、反馈、决定 |
| 典型问题 | “项目部署规范是什么？” | “用户喜欢什么输出格式？” |
| 数据形态 | 文档切片 + 向量 | 带元数据的 Markdown 条目 |
| 检索 | 向量 + 关键词 + MMR | BM25 |
| 当前持久性 | 向量索引只在内存 | 文件持久化在 `.memory/` |
| 写入方式 | 启动自动导入或 `rag_ingest` | 模型调用 `memory write` 或人工文件 |

一句话概括：RAG 查“资料里写了什么”，Memory 记“我们之间形成了什么”。

## 4.2 RAG 完整流水线

```text
Markdown document
  ↓ chunkDocument
Chunks: source / text / index / token estimate
  ↓ embed
Vectors
  ↓ VectorStore.addBatch
内存索引

用户问题
  ↓ query embedding + tokenize
向量候选 + BM25 候选
  ↓ 分数归一化与加权
混合候选
  ↓ MMR
兼顾相关性和多样性的 Top K
```

## 4.3 文档切片

Embedding 模型和检索都不适合直接处理整本长文档。`chunkDocument()` 的目标大小约 256 Token，用每 Token 4 字符粗略估算。

切片策略会尝试沿 Markdown 结构拆分：标题、段落、文本长度。每个 `Chunk` 保存：

- `source`：来源路径；
- `text`：片段内容；
- `index`：在原文中的顺序；
- `tokenCount`：估算 Token。

切片需要平衡：

- 太大：召回内容噪声多，Embedding 表达不聚焦；
- 太小：上下文断裂，答案所需信息分散在多个片段；
- 没有重叠：边界处语义可能丢失；
- 重叠太多：索引冗余、结果重复。

当前实现是便于理解的轻量切分器，没有复杂的语义断句、重叠窗口和元数据继承。

## 4.4 Embedding

Embedding 把文本映射为数值向量，语义相似文本在向量空间中更接近。

项目提供两种实现：

### Mock Embedder

无 API Key 时使用固定维度的确定性本地向量。相同文本会得到相同结果，适合跑通流程，但不代表真实语义质量。

### DashScope Embedder

真实模式调用 DashScope Embedding API。批量文本会发送到外部服务，因此不要导入不能外传的文档。

`embed()` 还维护进程内缓存，按文本缓存向量，避免同一文本重复请求。缓存不会跨进程保存，也没有大小或淘汰策略。

相似度使用余弦相似度：

```text
cos(a,b) = a·b / (|a| |b|)
```

它比较方向而不是绝对长度，常用于文本向量。

## 4.5 VectorStore

`VectorStore` 是内存数组封装，支持：

- 增加单条/批量片段；
- 返回全部片段；
- 按向量相似度搜索；
- 清空；
- 查询数量。

课程目录提到 `sqlite-vec`，但当前仓库依赖和实现中没有 SQLite。进程退出后索引消失，重启时重新扫描 `docs/` 第一层 Markdown 并计算 Embedding。

这必须在分享时明确：当前展示的是 RAG 算法管线，不是生产级持久化向量数据库。

## 4.6 为什么不只用向量检索

向量擅长语义相似，但可能不擅长：

- 精确的函数名、错误码、版本号；
- 路径和缩写；
- 罕见专有名词；
- 必须字面匹配的短查询。

BM25 类关键词检索恰好在这些场景更可靠。项目的 `hybridSearch()` 同时走两条路径：

```text
Vector path: query → embedding → cosine score
Keyword path: query → tokenize → BM25-like score
```

先各取 `topK * 4` 候选，再合并。

## 4.7 分数为什么要归一化

余弦相似度和 BM25 分数不在同一尺度，不能直接 `0.7 * vector + 0.3 * keyword`。代码先归一化：向量分数使用 Min-Max，关键词分数使用 Sigmoid 风格处理，再按：

```text
hybrid = 0.7 × vector + 0.3 × keyword
```

权重不是通用真理，只是当前经验参数。不同语言、文档类型和查询分布都需要离线评测。

## 4.8 MMR：避免结果全在说同一件事

单纯取最高分，可能返回五个高度重复的片段。MMR（Maximal Marginal Relevance）在相关性和新信息之间平衡：

```text
MMR = λ × relevance - (1 - λ) × similarity_to_selected
```

当前 `λ = 0.7`，更偏重相关性，同时对已选结果的文本 Jaccard 相似度做惩罚。它使用词集合近似多样性，而不是再次计算向量相似度。

## 4.9 RAG 如何被 Agent 使用

`createRagTools()` 注册：

- `rag_ingest`：读取路径、切片、Embedding、加入 VectorStore；
- `rag_search`：对查询做混合检索，返回带来源和分数的片段。

启动时 `main.ts` 自动读取 `docs/` 目录**第一层**的 `.md` 文件。它不会递归导入 `docs/learning/`。

System Prompt 只告诉模型知识库有多少片段、可以用 `rag_search`，并不自动把用户问题拿去检索。这使模型能跳过无关检索，但也依赖模型正确判断何时使用工具。

## 4.10 Memory 的文件模型

每条记忆是 `.memory/` 中的 Markdown 文件，带 Frontmatter：

```markdown
---
name: typescript-preference
description: 用户偏好 TypeScript
type: user
lastWriteAt: 1730000000000
lastReadAt: 1730000000000
---

用户在示例和实现中优先使用 TypeScript。
```

允许的类型：

- `user`：用户偏好和稳定背景；
- `feedback`：用户纠正过的行为；
- `project`：无法仅从代码推导的项目决定；
- `reference`：外部资源或线索。

`MEMORY.md` 是精简索引，完整内容保存在各条目文件中。

## 4.11 MemoryStore 的约束

为避免记忆无限增长，代码设置：

- 索引最多 200 行；
- 单条读取最多 4000 字符；
- 30 天作为过期判断参考；
- 读写时更新 `lastReadAt` / `lastWriteAt`。

`buildPromptSection()` 只把索引和原则放入 Prompt，并提醒：记忆是线索，不是事实，使用前要用文件或搜索工具验证。

这是一个重要设计：长期记忆可能过期。Agent 不能因为“曾经记过”就跳过对当前仓库或外部状态的核实。

## 4.12 Memory 搜索

`bm25Search()` 对中英文做简单分词：

- 英文和数字按连续串；
- 中文按单字；
- 名称重复三次、描述重复两次，以提高元数据权重。

BM25 考虑：

- 词频饱和；
- 逆文档频率；
- 文档长度归一化。

中文按单字分词很粗糙，但短记忆通常仍可用。生产中文检索可考虑更好的分词、Embedding 或混合检索。

## 4.13 记忆校验与 Dream

`memory/validator.ts` 会检查：

- Frontmatter/字段健康度；
- 内容中提到的路径是否存在；
- 记忆是否可能陈旧或无效。

`/dream` 使用模型整理记忆：删除过期内容、合并重复条目、保持索引健康。Dream 体现“记忆需要维护”，但自动删除或合并长期信息存在误判风险，重要环境应先生成建议再由人确认。

## 4.14 什么应该记，什么不应该记

适合 Memory：

- “用户希望中文回答，代码注释用英文”；
- “团队决定在下一季度前继续支持旧 API”；
- “用户纠正：不要自动执行部署”；
- “外部设计稿链接和对应模块”。

不适合 Memory：

- `package.json` 已经写明的依赖；
- 可以从目录推导的技术栈；
- Git 能查到的修改历史；
- 临时日志和一次性错误；
- 密钥和敏感身份信息。

## 4.15 实践

### RAG

```text
/rag
部署时 CPU 告警阈值是多少？
API breaking change 有哪些规则？
```

观察模型是否调用 `rag_search`，结果来源是否指向演示文档。然后对照原文件判断召回是否准确。

### Memory

```text
请记住：我学习本项目时希望先讲原理，再看源码。
/memory
/memory search 学习
/memory lint
```

退出后检查 `.memory/`。重新启动时索引仍在，说明 Memory 持久化；RAG 则会重新构建。

## 4.16 本章自检

1. RAG 和 Memory 的信息来源分别是什么？
2. 为什么向量检索还需要关键词检索？
3. 为什么混合分数需要归一化？
4. MMR 解决的是召回率问题还是结果重复问题？
5. 当前 VectorStore 是否跨进程持久化？
6. 为什么记忆使用前仍要验证？
