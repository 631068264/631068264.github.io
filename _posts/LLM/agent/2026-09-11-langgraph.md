---
layout:     post
rewards: false
title:   LangGraph and Deepagents
categories:
    - AI
tags:
   - 大模型


---



| 维度                   | LangGraph                                        | Deep Agents                                    |
| ---------------------- | ------------------------------------------------ | ---------------------------------------------- |
| 定位                   | **低层 Agent Orchestration Framework + Runtime** | **高层 Agent Harness**                         |
| 抽象层级               | 低                                               | 高                                             |
| 核心思想               | State + Node + Edge                              | Tool Loop + Middleware + Filesystem + Subagent |
| 控制流                 | **你显式定义**                                   | 大量默认行为已经封装                           |
| Deterministic Workflow | **非常强**                                       | 可以，但不是主要卖点                           |
| ReAct Loop             | 自己组合                                         | **已经给你搭好典型 agent loop**                |
| Planning               | 自己实现 Node/State                              | 有现成 planning/todo 能力                      |
| State                  | **一等公民**                                     | 底层使用 LangGraph State                       |
| Persistence            | **核心能力**                                     | 继承 LangGraph                                 |
| Checkpoint             | **核心能力**                                     | 继承 LangGraph                                 |
| Human-in-the-loop      | `interrupt()`                                    | 已封装进 Harness 能力                          |
| Long-term Memory       | Store                                            | 有现成 Memory 接入                             |
| Context Compact        | 自己设计                                         | **Auto summarization / filesystem offload**    |
| Filesystem             | 你自己提供 Tool                                  | **一等能力**                                   |
| Sandbox / Shell        | 你自己集成                                       | **已有 Backend / execute**                     |
| Subagent               | Subgraph / tool / graph                          | **内置 `task` delegation**                     |
| Context Isolation      | 自己控制 subgraph/state                          | **Subagent 主打能力**                          |
| Skills                 | 自己设计                                         | **原生支持**                                   |
| Middleware             | 可自己构建                                       | **大量内置 Middleware**                        |
| Permission             | 自己设计                                         | Filesystem permission 已有                     |
| Multi-agent            | Graph/Subgraph 自由组合                          | Supervisor + Subagent 更开箱                   |
| Time Travel            | ✅                                                | 继承                                           |
| Fault Recovery         | ✅                                                | 继承                                           |
| Streaming              | ✅                                                | 继承                                           |
| 学习成本               | 较高                                             | **低很多**                                     |
| 自由度                 | **最高**                                         | 中高                                           |
| 适合场景               | 精确业务 Workflow / State Machine                | Coding/Research/General-purpose Agent          |



- 有明确业务规则用 langGraph
- 其他可以用Deep Agents





# LangGraph 的核心原理

State Node Edge

- State

  表示整个 Agent 当前状态：Agent 当前世界状态的 Snapshot。

  ```
  class State(TypedDict):
      messages: list
      intent: str
      current_step: str
      search_results: list
      retries: int
      final_answer: str
  ```

  每一个 Node：

  ```
  读 State
  ↓
  执行逻辑
  ↓
  更新 State
  ```


- Node

  Node 就是普通函数

  ```
  def retrieve(state):
      results = search(state["query"])
  
      return {
          "search_results": results
      }
  ```

  Node 可以是：Python Code、LLM Call、Tool Call、Subagent、RAG、Human Approval

- Edge

  决定下一步去哪， 决定流程





**LLM 场景里的可持久化、有状态 Workflow / State Machine Runtime**。  而且最核心的工程能力是**Persistence+Checkpoint**，一旦某个分支失败了，可以从最近的checkpoint恢复，继续往下走。通过thread_id 保存

```
Agent
↓
准备 delete_database()
↓
interrupt
↓
保存 State
↓
等人审批

       ↓ Approve

恢复 State
↓
继续执行
```

LangGraph 的 `interrupt()` 会暂停 Graph，并通过 persistence 保存当前状态，之后可以 Resume。



Memory

```
Short-term Memory
→ Thread / Session

Long-term Memory
→ Cross-thread / Cross-session
```

Short-term 是 Graph State + Checkpointer。

Long-term 则是：Store，可以跨 thread 保存用户或应用数据



# Deep Agents 

agent harness + 封装好一堆agents 常用的东西

Tool Calling Loop、Filesystem、Subagents、Memory、Human Approval、Permissions、Planning

```
              Harness
┌───────────────────────────────┐
│ Context Manager               │
│                               │
│ LLM                           │
│ ↓                             │
│ Tool Call                     │
│ ↓                             │
│ Middleware / Hook             │
│ ↓                             │
│ Tool                          │
│ ↓                             │
│ Observation                   │
│                               │
│ State / Memory / Skill        │
│ Budget / Validator            │
└───────────────────────────────┘

```



- Filesystem

  长 Agent 最大问题之一：

  ```
  Tool Result
  Tool Result
  Tool Result
  网页
  代码
  文档
  ...
  ↓
  Context 爆炸
  ```

  Deep Agents 的思路不是：

  ```
  所有东西都塞 messages
  ```

  而是：

  ```
  Context Window
  ├── 当前必要信息
  │
  └── 大量中间结果
          ↓
     Filesystem
  ```

  比如：

  ```
  /research/vllm.md
  /research/sglang.md
  /results/benchmark.csv
  /draft/report.md
  ```

  Filesystem Backend 还可以换 

- Subagent

  ```
  # Deep Agents 给主 Agent：
  task(...)
  ```

  避免大量 Tool 输出污染主 Agent 上下文。

  LangGraph Subgraph 是低层组合方式，自由度高。Subagent是Agent 可直接调用的 task tool

- Context 管理

  ```
  自动处理
  
  Context 越来越大
  ↓
  Summarization
  +
  Filesystem Offload
  +
  Subagent Isolation
  ```

  三个配合起来

  ```
  Main Context
  │
  ├── 最近消息
  ├── 当前 TODO
  ├── 当前任务状态
  │
  ├── Summary
  │
  └── Subagent Final Result
  ```

- Middleware

  Deep Agents 目前大量功能实际上通过 Middleware 做,Deep Agents 默认架构特别依赖 Todo、Filesystem、Subagent 这些 Middleware

  所以架构上更像：

  ```
  User
   ↓
  Middleware
   ↓
  Model
   ↓
  Middleware
   ↓
  Tool
   ↓
  Middleware
   ↓
  Observation
  ```

  

  

