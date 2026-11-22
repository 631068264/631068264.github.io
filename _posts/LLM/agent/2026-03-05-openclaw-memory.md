---
layout:     post
rewards: false
title:   openclaw
categories:
    - AI
tags:
   - 大模型



---

# OpenClaw 完整调用原理

```
用户发消息（WhatsApp/Telegram/Discord...）
           │
           ▼
┌─────────────────────────────────────────┐
│  Channel Adapter（渠道适配器）            │
│  src/telegram/ src/discord/ src/slack/  │
│  1. 鉴权（bot token / QR / OAuth）       │
│  2. 解析消息（文字/图片/语音/reaction）   │
│  3. 检查 allowlist / dmPolicy            │
└────────────────┬────────────────────────┘
                 │ 标准化消息对象
                 ▼
┌─────────────────────────────────────────┐
│  Auto-Reply 路由层                       │
│  src/auto-reply/reply.ts                │
│  1. resolveSessionKey() 确定会话 ID      │
│     主聊天 → "main"                      │
│     群组  → "group:channel:id"          │
│     DM    → "dm:channel:id"             │
│  2. 检查 queue mode（steer/followup）    │
│  3. 派发给 PiEmbeddedRunner              │
└────────────────┬────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────┐
│  Gateway WebSocket 控制面               │
│  src/gateway/server.ts  :18789          │
│  协议帧格式：                            │
│  req  {type:"req", id, method, params}  │
│  res  {type:"res", id, ok, payload}     │
│  event{type:"event", event, payload}    │
└────────────────┬────────────────────────┘
                 │
                 ▼
┌─────────────────────────────────────────┐
│  PiEmbeddedRunner（Agent 推理核心）      │
│  src/agents/piembeddedrunner.ts         │
│                                         │
│  ① 加载会话历史                          │
│     ~/.openclaw/agents/<id>/sessions/   │
│     <SessionId>.jsonl                   │
│                                         │
│  ② 构建 System Prompt                   │
│     src/agents/prompt-builder.ts        │
│     注入 AGENTS.md / SOUL.md /          │
│     TOOLS.md / USER.md 等文件            │
│     + Memory 搜索结果（SQLite）          │
│                                         │
│  ③ 调用模型 API（流式）                  │
│     provider/model → Claude / GPT 等    │
│                                         │
│  ④ Tool Calling 循环                    │
│     收到 tool_use block →               │
│     执行工具（可选 Docker 沙箱）→        │
│     注入 tool_result → 继续推理          │
│                                         │
│  ⑤ 持久化会话状态                        │
└────────────────┬────────────────────────┘
                 │ 流式文本/tool结果
                 ▼
┌─────────────────────────────────────────┐
│  Streaming + Chunking 层                │
│  src/concepts/streaming                 │
│  按段落/字符数切割，防消息过长            │
└────────────────┬────────────────────────┘
                 │
                 ▼
         回写到原渠道（reply）
```



### 关键源码文件对应关系

| 职责                 | 源码位置                                               |
| -------------------- | ------------------------------------------------------ |
| CLI 入口             | `openclaw.mjs` → `src/index.ts` → `src/cli/program.ts` |
| Gateway WS 服务      | `src/gateway/server.ts`                                |
| 渠道适配（Telegram） | `src/telegram/` (grammY)                               |
| 渠道适配（Discord）  | `src/discord/` (discord.js)                            |
| 统一路由分发         | `src/auto-reply/reply.ts`                              |
| 会话 Key 解析        | `src/config/sessions.ts` → `resolveSessionKey()`       |
| Agent 推理主循环     | `src/agents/piembeddedrunner.ts`                       |
| System Prompt 组装   | `src/agents/prompt-builder.ts`                         |
| 工具执行 + 沙箱      | `src/agents/sandbox.ts`                                |
| 记忆搜索             | `src/memory/`（SQLite 存储）                           |
| 插件加载             | `src/plugins/loader.ts`                                |
| Schema 验证          | `src/config/schema.ts`（TypeBox）                      |





## memory

OpenClaw Memory 

OpenClaw 的记忆系统分成**三层**，这是它比普通 RAG 方案设计更完整的地方：

```
┌───────────────────────────────────────────┐
│  Layer 1: 文件记忆（Markdown）              │
│  MEMORY.md / memory/*.md                  │
│  人类可读、AI 可直接写入的持久化笔记         │
├───────────────────────────────────────────┤
│  Layer 2: 向量+关键词混合索引（SQLite）     │
│  sqlite-vec (HNSW) + FTS5 (BM25)          │
│  对 Layer 1 文件 + 历史会话的双轨检索       │
├───────────────────────────────────────────┤
│  Layer 3: 会话历史（JSONL）                │
│  ~/.openclaw/agents/<id>/sessions/*.jsonl │
│  可选编入向量索引，供 memory_search 召回   │
└───────────────────────────────────────────┘
```

完整流程
```
用户发消息
    │
    ▼
【写入阶段：会话进行中】
    Agent 推理时，模型可主动调用：
    memory_search(query) → 语义检索
    memory_get(path)     → 精确读取某个 .md 文件(从n到m行， 减少context长度，类似过滤的方法)
    write(memory/xxx.md) → 把重要信息写进记忆文件

    │
    ▼
【危险边界：context 快满了】固定长度 （模型不经过训练调参，是不会主动处理的context的）
    pre-compaction memory flush 触发（静默）：
    系统发送一个隐藏 prompt 给模型：
    "Session nearing compaction. Store durable memories now."
    模型把值得保留的内容写入 memory/YYYY-MM-DD.md
    如果没什么要写的，回复 NO_REPLY（用户看不到）
    然后执行 Context Compaction（摘要压缩）

    │
    ▼
【索引阶段：异步后台】
    chokidar 监听 memory/ 目录变化（防抖 1500ms）
    触发 sync：把新的/改动的 .md 文件 chunk 化
    同时检查 session transcript 的 delta：
      - deltaBytes > 100KB，或
      - deltaMessages > 50 行
    → 增量索引会话记录到 SQLite

    │
    ▼
【下次对话开始：session warming】
    第一次 memory_search 触发 warmSession()
    后台补一次 index sync，保证索引是最新的

    │
    ▼
【检索阶段（核心）：Hybrid Search】
    见下节详细展开
```

- 什么是重要信息：取决于 `AGENTS.md` / `SOUL.md` 里用户写的指令
- 会话历史 JSONL，格式是标准 JSONL，每行一条消息记录，内容包含 role（user/assistant/tool_result 等）、content、timestamp 等字段。**全量保存**，不会自动删除历史，只有执行 `/reset` 或手动清理才会切换。
- Markdown 文件按段落/语义边界切分，每个 chunk 截断到 700 字符，Session JSONL 按消息粒度切分（每条消息或几条消息一组）

- 长短期记忆
  - 区分逻辑完全通过文件路径约定 + 模型行为引导实现。日常笔记（`memory/YYYY-MM-DD.md`）是一个日志，适合记录几天内的连续性上下文，包含当天发生了什么、做了哪些决定。`MEMORY.md` 更像一个画像文件，存放跨时间都应保持为真的内容，比如用户偏好、技术栈、项目结构，应该保持小而精
- 模型context window







memory_search

向量搜索和关键词搜索**并行执行**，各自取top24

```
query: "我上次提到的那个项目deadline是什么时候？"
          │
          ├─────────────────────────────────────┐
          ▼                                     ▼
  【Vector Search】                     【Keyword Search】
  src/memory/manager-search.ts          src/memory/manager-search.ts
  用 embedding 把 query 向量化           用 buildFtsQuery() 解析关键词
  chunks_vec 表 (sqlite-vec)             chunks_fts 表 (FTS5)
  vec_distance_cosine() 余弦相似度       BM25 排名打分
  取 top-24 候选                         取 top-24 候选
  输出：[chunkId, cosineSimilarity]      输出：[chunkId, bm25Rank]
          │                                     │
          └──────────────┬──────────────────────┘
                         ▼
              【Hybrid Merging】
              src/memory/hybrid.ts: mergeHybridResults()

              对每个 chunkId 计算最终分数：
              finalScore = vectorScore × 0.7 + textScore × 0.3

              某 chunk 只在一边出现 → 另一边得分记 0
              按 finalScore 降序排列
                         │
                         ▼
              【过滤 + 截断】
              minScore < 0.35 的丢弃
              保留 top-6 结果
              每条 snippet 截断到 700 字符
                         │
                         ▼
              返回给 Agent，附带文件路径+行号
```

example
```
向量结果：[chunk-A(0.92), chunk-B(0.85), chunk-C(0.71), chunk-D(0.68)]
关键词结果：[chunk-B(0.90), chunk-C(0.78), chunk-E(0.65)]

合并时，对所有出现的 chunk 计算 finalScore：
chunk-A: 向量=0.92, 关键词=0（没出现）  → 0.92×0.7 + 0×0.3 = 0.644
chunk-B: 向量=0.85, 关键词=0.90        → 0.85×0.7 + 0.90×0.3 = 0.865  ← 排第一
chunk-C: 向量=0.71, 关键词=0.78        → 0.71×0.7 + 0.78×0.3 = 0.731
chunk-D: 向量=0.68, 关键词=0（没出现）  → 0.68×0.7 + 0×0.3 = 0.476
chunk-E: 向量=0（没出现）, 关键词=0.65  → 0×0.7 + 0.65×0.3 = 0.195
```

两个可选后处理阶段

**MMR（Maximal Marginal Relevance）**：迭代选择能最大化 `λ × relevance - (1-λ) × max_similarity_to_selected` 的结果，防止返回一堆几乎相同内容的 chunks，默认关闭。**Temporal Decay**：对较老的 chunk 打指数衰减折扣，半衰期默认 30 天；但 `MEMORY.md` 和 `memory/` 下没有日期的文件是 evergreen，永远不衰减，默认关闭。 [DeepWiki](https://deepwiki.com/openclaw/openclaw/7.3-memory-search)

搜索结果还会按 `(path, startLine, endLine)` 元组去重，避免向量 + 关键词两边都命中同一个 chunk 后返回重复内容







```sql
-- 主 chunk 表
chunks (
  id           TEXT PRIMARY KEY,
  source       TEXT,        -- "memory" 或 "sessions"
  path         TEXT,        -- 文件路径（相对 workspace）
  startLine    INTEGER,
  endLine      INTEGER,
  text         TEXT,        -- chunk 原文
  updatedAt    INTEGER
)

-- 向量搜索表（sqlite-vec 扩展，HNSW 索引）
chunks_vec (
  chunk_id     TEXT,
  embedding    BLOB         -- float32 向量
)

-- 全文检索表（FTS5）
chunks_fts (
  chunk_id,
  text         -- 同 chunks.text，建了 BM25 倒排索引
)

-- embedding 缓存表
embedding_cache (
  cache_key    TEXT PRIMARY KEY,  -- SHA-256(provider + model + text)
  embedding    BLOB,
  createdAt    INTEGER
)

```





