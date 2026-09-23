# Phase 1 · D1-2 学习笔记
## OS 进程模型 ↔ TGT AgentLoop 映射

> 日期：2026-09-23 · 学习主题：Silberschatz《操作系统概念》第 3 章 进程
> 核心问题：TGT 里的 AgentLoop、Runtime 状态机、Worker、BullMQ 队列，到底分别对应 OS 里的什么？

---

## 一、OS 进程是什么（书本定义）

> 进程 = **程序的一次执行实例** + **资源的独立拥有者**（内存、文件描述符、寄存器）
>
> PCB (Process Control Block) = 内核里描述进程的那张表，包含：
> 进程状态（new/ready/running/waiting/terminated）、程序计数器、寄存器内容、内存管理信息、打开的文件、审计信息

### OS 进程生命周期（简化）

```
 new ──(加载到内存)──▶ ready ──(CPU 调度)──▶ running
                         ▲  │                    │
                         │  │(等待 IO)           │(IO 完成)
                         │  └──────▶ waiting ───┘
                         │
                         └────(被抢占或时间片用完)────┘
                                                    │
                                              (exit)┌─── terminated
                                                    └─── zombie（僵死）
```

### OS 进程 5 态

| 状态 | 含义 | 触发 |
|------|------|------|
| **New** | 进程刚创建，等待资源 | fork() / 系统请求 |
| **Ready** | 就绪，等 CPU 调度 | IO 完成、资源到位 |
| **Running** | 正在 CPU 上执行 | 调度器选中 |
| **Waiting / Blocked** | 等某个事件（磁盘、网络、锁）| IO 请求、锁竞争 |
| **Terminated** | 执行结束 | exit() |

---

## 二、TGT Runtime 12 态（对照）

TGT 已经有完整的 Runtime 状态机。我把它和 OS 进程 5 态对标一下：

### 完整状态图（来自 src/runtime/types.ts RUNTIME_STATES）

```
CREATED ──▶ QUEUED ──▶ STARTING ──▶ RUNNING ──▶ WAITING_TOOL ──▶ RUNNING ──▶ COMPLETING ──▶ COMPLETED
                                                                      ▲
                                                                      │
  (任意运行态) ──(PAUSE 命令)──▶ PAUSED ──(RESUME)──▶ RUNNING ───────┘
                                                                      │
  (任意运行态) ──(失败)──▶ FAILED ──(恢复)──▶ RECOVERING ──▶ RUNNING ─┘
                           └──(重试耗尽)──▶ FAILED_FINAL
                                                                      │
  (任意运行态) ──(STOP 命令)──▶ STOPPED ───────────────────────────────┘（三态终态）
```

### OS ↔ TGT 对照

| OS 状态 | TGT RuntimeStateName | 说明 |
|---------|----------------------|------|
| **New** | `CREATED` + `QUEUED` | `createRuntime()` 建行 = `fork()`；入 BullMQ 队列 = 等待父进程 fork 完分配资源 |
| **Ready** | `STARTING` | Worker 刚 pick up 任务，加载项目适配器、初始化 state machine — 相当于加载 ELF 到内存 + 映射地址空间 |
| **Running** | `RUNNING` + `COMPLETING` | AgentLoop 主循环跑 turn — CPU 真正在执行指令（turn 执行 LLM 一次调用 + 工具调用） |
| **Waiting / Blocked** | `WAITING_TOOL` | AgentLoop 发出工具调用后等结果 — 相当于 `read()` 系统调用等磁盘 / 网络 IO |
| — | `PAUSED` | OS 没有直接对应的，这是 TGT 的 **协作式暂停**，类似 `SIGSTOP`，比 OS 信号更安全（turn 边界才停，不会打断 LLM 半途中） |
| — | `RECOVERING` | OS 没有。TGT 特色：失败后自动尝试恢复（reinitialize / resume），OS 进程崩了就没了 |
| **Terminated** | `COMPLETED` / `FAILED_FINAL` / `STOPPED` | **TGT 有三个终态**！OS 只有一个 terminated，TGT 细分了：正常完成 / 不可恢复失败 / 人工停止 |

---

## 三、OS PCB ↔ TGT APCB（Agent Process Control Block）

### OS PCB 字段 ↔ TGT 现状

| OS PCB 字段 | 含义 | TGT 有吗？ | 在哪里？ | 差距 |
|------------|------|-----------|---------|------|
| **Process ID** | 唯一标识 | ✅ | `LoopSession.sessionId` + `Runtime.runtimeId` + `TgtTask.id` | 三套 ID，没统一到一个 APCB 主键 |
| **Process State** | 当前状态 | ✅ | `RuntimeStateName`（12 态） | 比 OS 5 态精细得多 |
| **Program Counter** | 下一条要执行的指令 | ⚠️ 退化 | AgentLoop `turn: number` | LLM 对话没有 PC 概念，turn 号最接近 |
| **CPU Registers** | 执行现场快照 | ⚠️ 退化 | AgentLoop `messages[]` + `currentTurn` | **LLM context 就是寄存器！** TGT 已经持久化了 messages，这是 Phase 4 的核心洞察 |
| **Memory Management** | 基址寄存器 + 限长寄存器 | ❌ 无 | — | 没有显式"内存"概念，context window 就是隐式内存 |
| **Open Files** | 文件描述符表 | ⚠️ 散 | `ToolContext` + `bindings` + 会话资源回收器 | 工具会话没有统一的 fd 概念 |
| **Scheduling Info** | 优先级、时间片 | ✅ | BullMQ `priority` + `AgentPriority` 枚举 | 完整 |
| **IO / Accounting** | 已用 CPU 时间、IO 计数 | ⚠️ 部分 | `tokenUsage` + `LLMCost` 表 + `AgentSchedulerMetrics` | 有 token 但没 wall-clock 秒数累计 |
| **审计** | 谁启动的、安全标签 | ✅ | `LoopSession.createdBy` + `projectId` 租户隔离 | 完整 |
| **Parent / Child** | 进程树 | ⚠️ 散 | `DelegationChainNode` + `SubAgentExecution` 父子关联 | 有，但没强制成树结构（可能有循环委派） |

### 结论

**TGT 已有 APCB 雏形，但散在 5 张表 + 3 个 service 里**。下一阶段（Phase 2）应该抽成单一的 `AgentPCB` 接口。

---

## 四、OS 进程创建 ↔ TGT 任务派发

### OS 路径

```
父进程 fork()
  ──▶ 内核分配新 PID + 空 PCB
  ──▶ 复制父进程地址空间（COW）
  ──▶ 设置 initial state = READY
  ──▶ 放入就绪队列
  ──▶ CPU 调度器选中 → 开始执行子进程
```

### TGT 路径（已实现）

```
API 层 POST /tasks 或 CeoAgent 派发子 Agent
  ──▶ RuntimeTransitionEngine.createRuntime()        ← 分配 runtimeId + 建 RuntimeState（= fork + 建行）
  ──▶ transition('QUEUE')                            ← 设置状态
  ──▶ AgentScheduler.queue.add('execute-agent', ...) ← BullMQ 入队（= 放入就绪队列）
  ──▶ Worker 进程 BullMQ worker.on('completed')      ← CPU 调度器选中
  ──▶ tgt-task.processor.ts: stateMachine.run()      ← 子进程开始执行
  ──▶ AgentLoop.run() 主循环                         ← 真正的"指令执行"
```

**一一对应，严丝合缝。** 关键节点：
- `RuntimeTransitionEngine` = **内核**（唯一权威，CAS + Receipt + Fact 同事务）
- BullMQ Queue = **就绪队列 / 等待队列**
- BullMQ Worker = **CPU**
- AgentLoop = **进程真正跑起来的那段代码**

---

## 五、OS 进程间通信 ↔ TGT B2B

OS 有 5 种传统 IPC：

| OS IPC | TGT 对应 | 代码位置 |
|--------|---------|---------|
| **Pipe**（匿名管道，父子进程） | 子 Agent 委派链 `delegationChain` | `AgentScheduler.DelegationChainNode` |
| **Named Pipe / Unix Domain Socket**（同机异进程） | Redis Pub/Sub 实时广播 | `src/lib/tgt/b2b/` |
| **Message Queue** | Redis Stream 持久化队列 | `src/lib/tgt/b2b/` transport 层 |
| **Shared Memory** | Agent 共享内存 / CronMemory / LoopSession.messages | `src/services/tgt-ceo/memory-store.ts` |
| **Semaphore / Mutex** | BullMQ 分布式锁 + `RuntimeState.version` CAS | `RuntimeStateView.version` + `findTransition` |

### TGT 比 OS 多出来的

OS IPC 是 **字节流 + 地址**；TGT B2B 是 **类型化消息 + 签名 + 幂等 + 共识**。具体：
- HMAC-SHA256 签名 + 消息序号防重放
- `deriveSourceEventId` 幂等去重（`reportedBy:trigger:correlationId`，**拒绝随机 UUID**）
- ConsensusManager 多 Agent 群聊共识

**→ 这是 AIOS 比传统 OS 强的地方：OS 只做传输层，AIOS 把语义层也做了。**

---

## 六、OS 中断 / 异常 ↔ TGT 信号模型

OS 信号：`SIGKILL / SIGSTOP / SIGCONT / SIGINT / SIGSEGV`

TGT 已经有类似语义：

| OS 信号 | TGT 对应 | 触发 | 代码 |
|---------|---------|------|------|
| SIGSTOP | PAUSE 命令 | operator 手动暂停 | `RuntimeTransitionEngine.transition('PAUSE')` + AgentLoop `WAITING_TOOL` 边界检查 → 协作式暂停 |
| SIGCONT | RESUME 命令 | operator 恢复 | AgentLoop 从 messages 快照恢复 |
| SIGKILL | STOP 命令 | operator 强停 | `transition('STOPPED')` — **OS SIGKILL 不可捕获，TGT STOP 是协作式的**（turn 边界才停，不会打断 LLM 半途中） |
| SIGSEGV（非法内存） | RuntimeFailure + RECOVERY_PROCESS_CRASHED | Worker 进程崩溃 | `INTERNAL_TRIGGERS` + `RuntimeFailureRecord` 表 |
| SIGALRM | wallClockTimeout | 超时熔断 | AgentLoop `Promise.race` + `AbortController` |

### 关键洞察：TGT 的 PAUSE/STOP 是协作式的，比 OS 信号更安全

```
OS:  内核可以在任意指令之间插入 SIGKILL — 用户进程完全不可控
TGT: PAUSE 只在 turn 边界（LLM 调用返回后 / 工具调用发出前）才生效
     — 避免 LLM 输出一半被截断、避免 DB 事务中间被打断
```

---

## 七、Phase 1 D1-2 结论

### TGT 已经实现了多少 OS 内核？

| OS 子系统 | 对应代码 | 完成度 |
|----------|---------|--------|
| **进程状态机** | RuntimeTransitionEngine（12 态 + CAS + Receipt + Fact 同事务） | **90%**，只差把 APCB 统一成单一接口 |
| **CPU 调度** | BullMQ + AgentPriority + PRIORITY_MAP | **60%**，缺 CFS 权重调度、缺 context switch 成本建模 |
| **IPC** | Redis Pub/Sub + Stream + HMAC + ConsensusManager | **85%**，比 OS 多了语义层和安全层 |
| **中断/信号** | PAUSE/RESUME/STOP 命令 + RuntimeFailure | **70%**，协作式暂停是优势，但还没显式 SIGKILL/SIGSTOP 抽象 |
| **内存管理** | AgentLoop.messages + MemoryStore + TgtMemory | **40%**，还没有 L1/L2/L3 分层、没有 pgvector、没有惰性降级 |
| **PCB** | 散在 LoopSession / RuntimeState / TgtTask / SubAgentExecution / MemoryStore | **30%**，需要 Phase 2 统一 |

### 下一步（Phase 1 D3-4）

读 Silberschatz 第 6 章 CPU 调度 → 对比 TGT BullMQ 现有调度 vs Linux CFS → 写 benchmark 方案。

---

## 八、关键代码索引（以后查用）

| 想找什么 | 去哪里 |
|---------|--------|
| Runtime 12 态定义 | `src/runtime/types.ts` RUNTIME_STATES |
| 状态迁移规则矩阵 | `src/runtime/transition-matrix.ts` |
| 唯一权威 transition | `src/runtime/transition-engine.ts` |
| AgentLoop 主循环 | `src/agent-loop/agent-loop.ts` |
| turn 边界协作式暂停 | `evaluateRuntimeBoundary()` + `handleRuntimeBoundary()` agent-loop.ts L117-149 |
| BullMQ queue 注册表 | `src/workers/index.ts` |
| 子 Agent 派发 | `src/services/tgt-ceo/agent-scheduler/index.ts` |
| Runtime 生命周期事实 | `RuntimeFact` 表 + `src/runtime/` |
| 禁止的伪状态 | `FORBIDDEN_PSEUDO_STATES` in types.ts |

---

## 九、错题本

| 坑 | 正确做法 |
|----|---------|
| OS 有 PAUSE/RESUME 状态 | 没有，OS 用 READY/RUNNING/WAITING + 信号实现暂停，状态机里没有 PAUSE 这个状态。TGT 把 PAUSED 做成一等状态，这是 AIOS 的合理扩展 |
| 进程树一定要是树 | OS 有 init 进程领养孤儿，但 TGT 委派链可能循环（A→B→A），需要循环检测。不能假设树结构 |
| RuntimeState 里有 "PAUSING" | 设计文档里 FORBIDDEN_PSEUDO_STATES 明确禁止，必须**瞬间迁移**（CREATED→RUNNING，不能过 PAUSING 中间态） |
| 把 RuntimeCommand 和 RuntimeState 混 | Command 是**意图**（PAUSE / RESUME / STOP），State 是**事实**（PAUSED / RUNNING / STOPPED）。引擎只接收 Command，迁移 State，再写 Fact。边界必须分清 |

【创建者：Trae】
