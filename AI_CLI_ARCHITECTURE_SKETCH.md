# AI CLI 架构草图（可复用版）

> 项目：`/Users/chenxianhao/Desktop/project/claude_code_annotated`
> 目的：把 Claude Code 暴露出的核心设计抽象成一套**可复用的 AI CLI 架构蓝图**，用于学习、复盘、或设计你自己的 AI coding assistant / agent CLI。
>
> 这份文档的重点不是复刻 Claude Code，而是提炼出一套可以迁移到你自己项目里的通用骨架。

---

# 1. 一句话定义

一个成熟的 AI CLI，本质上是：

> **以会话为中心、以 agent loop 为执行核心、以工具运行时为能力总线、以权限和上下文管理为护栏的命令行工作系统。**

它不是一个“会聊天的命令行外壳”，而是一个能在终端里持续执行任务、调用工具、管理状态、控制风险的工作系统。

---

# 2. 总体分层图

```text
┌─────────────────────────────────────┐
│ 1. Entrypoint / CLI Shell           │
├─────────────────────────────────────┤
│ 2. Command Router                   │
├─────────────────────────────────────┤
│ 3. Session / Conversation Manager   │
├─────────────────────────────────────┤
│ 4. Agent Loop Engine                │
├─────────────────────────────────────┤
│ 5. Tool Runtime                     │
├─────────────────────────────────────┤
│ 6. Permission & Safety Gate         │
├─────────────────────────────────────┤
│ 7. Task / Background Job System     │
├─────────────────────────────────────┤
│ 8. Context / Memory / Compaction    │
├─────────────────────────────────────┤
│ 9. Extension Layer (MCP/Plugins)    │
├─────────────────────────────────────┤
│10. UI / Transport / Observability   │
└─────────────────────────────────────┘
```

---

# 3. 分层设计

## 3.1 Entrypoint / CLI Shell

### 作用
负责程序启动、参数解析、运行模式分流。

### 典型职责
- 解析 `argv`
- 判断运行模式：
  - 交互模式
  - 单次问答模式
  - 后台任务模式
  - daemon / server 模式
  - debug / export / diagnose 模式
- 尽量延迟加载重模块
- 初始化最基本的运行环境

### 推荐接口

```ts
interface CliEntrypoint {
  run(argv: string[]): Promise<void>
}
```

### 设计原则
- 入口文件只负责分流，不要承载业务细节
- 对 `--version` / `--help` 之类的轻路径做快速返回
- 大模块用 dynamic import，避免“什么都不干也全量启动”

### 常见错误
- 把 agent loop 直接写进入口文件
- 一启动就加载全部命令、工具、UI、插件
- 入口层与具体业务逻辑耦合太深

---

## 3.2 Command Router

### 作用
负责把输入分发给：
- 显式命令
- agent 自然语言执行路径

### 输入形态
1. 显式命令
   - `app status`
   - `/review`
   - `/config`
2. 自然语言任务
   - “帮我看看这个项目结构”
   - “修复这个构建错误”

### 推荐抽象

```ts
type RouteResult =
  | { kind: 'command'; command: Command }
  | { kind: 'agent'; prompt: string }

interface CommandRouter {
  route(input: UserInput): Promise<RouteResult>
}
```

```ts
interface Command {
  name: string
  description: string
  run(ctx: CommandContext): Promise<CommandResult>
}
```

### 核心原则
- 命令系统是**控制面**
- agent loop 是**执行面**
- 二者必须解耦

### 设计建议
- 命令层优先解决：状态查询、配置管理、环境诊断、流程控制
- 自然语言路径优先解决：探索、修改、执行、总结、协作

---

## 3.3 Session / Conversation Manager

### 作用
负责维护一个“持续可恢复的工作会话”，而不是只处理一次输入。

### 它应该管理的状态
- session id
- cwd / workspace
- message history
- 当前模型配置
- 当前权限模式
- 当前上下文缓存
- usage 状态
- 已加载 memory / attachments
- resume / replay 所需信息

### 推荐接口

```ts
interface SessionState {
  sessionId: string
  cwd: string
  messages: Message[]
  config: SessionConfig
  usage: UsageState
}

interface SessionManager {
  create(): Promise<SessionState>
  load(sessionId: string): Promise<SessionState>
  save(state: SessionState): Promise<void>
}
```

### 设计原则
- 会话状态不能只活在 UI 里
- 必须考虑持久化，不然做不了 resume / task / 子代理
- 当前输入只是会话的一轮，不是全部

### 价值
这层决定了你的 AI CLI 是“聊天脚本”，还是“可持续工作系统”。

---

## 3.4 Agent Loop Engine

### 作用
负责执行 AI CLI 的核心循环：

```text
用户输入
  -> 组装上下文
  -> 调模型
  -> 如果返回 tool call:
       执行工具
       写回 tool result
       再次调模型
  -> 如果返回 final text:
       输出结果
```

### 推荐接口

```ts
interface AgentLoop {
  runTurn(input: TurnInput): AsyncGenerator<AgentEvent, TurnResult>
}
```

### TurnInput 示例

```ts
interface TurnInput {
  session: SessionState
  prompt: string
  tools: ToolRegistry
  permissions: PermissionService
  memory: MemoryService
}
```

### AgentEvent 示例

```ts
type AgentEvent =
  | { type: 'token'; text: string }
  | { type: 'tool_start'; toolName: string; input: unknown }
  | { type: 'tool_end'; toolName: string; output: unknown }
  | { type: 'status'; message: string }
  | { type: 'error'; error: string }
```

### 这层只该做什么
- 请求编排
- 循环推进
- 停止条件控制
- 工具调用衔接
- 错误恢复
- 预算/compact/summary 的接入点调度

### 不该做什么
- 不直接实现具体工具逻辑
- 不直接依赖 UI 组件
- 不把权限细节散落到循环各处

### 关键判断
agent loop 是核心引擎，但它不应该变成“大泥球”。

---

## 3.5 Tool Runtime

### 作用
统一承接所有工具的：
- schema
- 执行
- 权限接入
- 进度通知
- 结果回收

它是 AI CLI 的能力总线。

### 推荐抽象

```ts
interface Tool<I = any, O = any> {
  name: string
  description: string
  inputSchema: JSONSchema
  execute(input: I, ctx: ToolContext): Promise<O>
}
```

```ts
interface ToolContext {
  session: SessionState
  abortSignal: AbortSignal
  fs: FileService
  shell: ShellService
  logger: Logger
  notify(event: ToolProgressEvent): void
}
```

```ts
interface ToolRegistry {
  list(): Tool[]
  get(name: string): Tool | undefined
}
```

### 为什么 ToolContext 很关键
因为工具不是一个孤立函数。它通常依赖：
- 当前会话
- 当前工作目录
- 中断控制
- UI 进度上报
- 文件读写服务
- shell 执行服务
- 权限系统
- 结果缓存

### 推荐的工具分层

#### 基础工具
- read file
- write / edit file
- glob
- grep
- exec shell

#### 结构化工具
- fetch url
- search web
- read pdf
- parse diff
- git status / log

#### 协作工具
- ask user
- create task
- read task output
- send notification

#### 扩展工具
- mcp tools
- plugin tools
- team/org private tools

### 设计原则
- Tool Runtime 是统一协议，不是工具集合大杂烩
- 工具的执行、权限、结果、状态回收最好走统一通路

---

## 3.6 Permission & Safety Gate

### 作用
在所有高风险操作之前做统一拦截和判定。

### 必须纳管的操作
- shell 执行
- 文件写入/覆盖/删除
- git commit / push
- 网络写操作
- 外部 API 写操作
- 任务创建 / 后台运行

### 推荐接口

```ts
type PermissionDecision = 'allow' | 'deny' | 'ask'

interface PermissionService {
  check(action: PermissionRequest): Promise<PermissionDecision>
}
```

```ts
interface PermissionRequest {
  kind: 'shell' | 'file_write' | 'network_write' | 'task_spawn'
  summary: string
  raw?: string
  cwd?: string
  riskLevel?: 'low' | 'medium' | 'high'
}
```

### 最好具备的能力
- allow once
- allow always
- deny rule
- 危险命令识别
- destructive pattern 检测
- sandbox mode
- 自动分类器 / policy limits

### 设计原则
- 权限系统必须是统一层，不要每个工具各玩各的
- 风险判定要尽可能结构化，别只靠 prompt 硬判断
- 用户审批信息要保留原始命令或变更意图，减少误批风险

### 一句话总结
权限系统决定你的 AI CLI 是“好玩”，还是“敢长期开着用”。

---

## 3.7 Task / Background Job System

### 作用
把系统从“单轮聊天”升级成“可持续工作”。

### 典型任务类型
- 前台 shell 任务
- 后台 shell 任务
- 子代理任务
- 工作流任务
- 定时任务
- 监控/轮询任务

### 推荐数据模型

```ts
interface TaskRecord {
  id: string
  type: 'shell' | 'agent' | 'workflow'
  status: 'pending' | 'running' | 'done' | 'failed' | 'killed'
  createdAt: number
  updatedAt: number
  summary: string
  outputPath?: string
}
```

### 推荐接口

```ts
interface TaskService {
  create(input: TaskInput): Promise<TaskRecord>
  get(id: string): Promise<TaskRecord | null>
  list(): Promise<TaskRecord[]>
  stop(id: string): Promise<void>
  appendOutput(id: string, chunk: string): Promise<void>
}
```

### 价值
如果没有任务系统，很多真实开发场景都很难做好：
- 跑长时间测试
- 扫大仓库
- 修复一串错误
- 后台做 review
- 多步工作流推进

### 设计建议
- 任务状态必须可查询、可中断、可持久化
- 长输出要落盘，不要全塞消息上下文
- 任务是 agent 的执行载体之一，但不等于 agent 本身

---

## 3.8 Context / Memory / Compaction

### 作用
控制上下文成本，维持长期可用性。

这是 AI CLI 能否真正商用的关键层之一。

---

### 3.8.1 Context Builder

负责每轮请求时决定“带什么上下文给模型”。

```ts
interface ContextBuilder {
  build(input: {
    session: SessionState
    prompt: string
    memory: MemorySnippet[]
  }): Promise<ModelContext>
}
```

### 上下文的典型组成
- system prompt
- 对话历史
- 当前任务上下文
- memory snippets
- 关键工具结果摘要
- 当前附件/引用资料

### 核心原则
- 每轮都带全历史是最蠢也最贵的方案
- 上下文应该是“经过选择和压缩后的工作视图”

---

### 3.8.2 Memory Service

负责长期信息召回。

```ts
interface MemoryService {
  search(query: string): Promise<MemorySnippet[]>
  write(entry: MemoryEntry): Promise<void>
}
```

### 推荐分层
- 短期记忆：当前会话消息
- 中期记忆：最近任务摘要 / 阶段总结
- 长期记忆：偏好、约定、项目背景、持续上下文

### 价值
这层决定系统是否能：
- 跨轮记住项目背景
- 跨会话延续协作风格
- 不靠“带全历史”硬撑上下文

---

### 3.8.3 Compaction / Summarization

负责在上下文过长时压缩历史。

```ts
interface CompactionService {
  shouldCompact(session: SessionState): boolean
  compact(messages: Message[]): Promise<CompactResult>
}
```

### 应优先保留
- 用户目标
- 已完成动作
- 当前未完成事项
- 关键工具结果摘要
- 环境/权限/限制条件

### 应优先裁掉
- 大段工具原始输出
- 重复中间状态
- 低价值日志
- 已无后续依赖的旧细节

### 设计建议
- compact 不只是“摘要消息”，而是“重建工作上下文”
- 工具结果摘要最好与 compact 结合，而不是分别乱做

---

## 3.9 Extension Layer（MCP / Plugins / Skills）

### 作用
让系统具备扩展能力，而不是把所有业务能力写死在主程序里。

### 主要扩展方向
- MCP：连接外部工具、资源、服务
- Plugins：加载本地/组织级能力包
- Skills / Prompt Modules：承载任务模板、工作流模板、领域规则

### 推荐接口

```ts
interface ExtensionManager {
  loadPlugins(): Promise<void>
  loadMcpServers(): Promise<void>
  listTools(): Promise<Tool[]>
  listSkills(): Promise<Skill[]>
}
```

### 设计原则
- 核心层定义协议，扩展层提供能力
- 不让具体业务逻辑反向污染 agent loop
- skill 更像“工作方法模板”，plugin / MCP 更像“能力扩展”

### 适用场景
- 给企业内网系统接私有工具
- 给 IDE / 设计系统 / CI / 监控系统接特殊能力
- 给不同项目加载不同工作流模板

---

## 3.10 UI / Transport / Observability

### 作用
让系统可见、可控、可调试、可接入不同终端表面。

---

### 3.10.1 UI / Transport

### 可能形态
- TUI / REPL
- JSON / structured output
- WebSocket / remote bridge
- IDE 插件通道
- 机器人/聊天表面输出

### 推荐接口

```ts
interface OutputRenderer {
  render(event: AgentEvent): void
}
```

### 设计建议
- 输出层吃 `AgentEvent`，不要直接耦合底层实现
- 同一套核心逻辑应能支持多个 transport
- tool progress / status / final result 最好统一事件流

---

### 3.10.2 Observability

### 至少要有的内容
- usage 统计
- tool 调用日志
- task 状态
- error tracing
- session transcript
- 性能指标（可选）

### 推荐接口

```ts
interface TelemetryService {
  track(event: TelemetryEvent): void
}
```

### 注意事项
- 观测层要可裁剪、可配置、可分级
- 别默认全量采集所有敏感数据
- 对外部产品场景，隐私与最小采集是硬要求

---

# 4. 调用链草图

```text
User Input
  -> CLI Entrypoint
  -> Command Router
  -> Session Manager
  -> Context Builder
  -> Agent Loop
      -> Model Request
      -> Tool Call?
          -> Permission Gate
          -> Tool Runtime
          -> Task Service / FS / Shell / Web
          -> Tool Result
      -> Final Response
  -> Output Renderer
  -> Session Persist
  -> Telemetry / Logs
```

这条链路的意义是：
- 输入先进入控制层
- 执行由 agent loop 驱动
- 能力通过 tool runtime 提供
- 风险通过 permission gate 控制
- 长期性通过 session/memory/task 维持

---

# 5. 推荐的项目目录草图

如果你自己从零设计一个 AI CLI，目录结构建议像这样：

```text
src/
  entrypoints/
    cli.ts
    daemon.ts
    server.ts

  core/
    agentLoop.ts
    commandRouter.ts
    sessionManager.ts
    contextBuilder.ts

  tools/
    Tool.ts
    ToolRegistry.ts
    file/
    shell/
    web/
    task/

  permissions/
    permissionService.ts
    riskClassifier.ts
    sandbox.ts

  tasks/
    taskService.ts
    taskStore.ts
    backgroundRunner.ts

  memory/
    memoryService.ts
    memoryStore.ts
    compactionService.ts

  extensions/
    mcp/
    plugins/
    skills/

  ui/
    repl/
    structured/
    renderer/

  infra/
    logger.ts
    config.ts
    telemetry.ts
    storage.ts
```

### 这个结构的好处
- 控制面和执行面天然分离
- 工具、权限、任务、记忆各有边界
- 后续接 MCP / 插件 / IDE / Web 都不至于重构骨架

---

# 6. MVP（最小可用版本）建议

如果要自己做一个第一版 AI CLI，不建议一开始就追求全功能。

## 第一版建议只做 7 个模块
- CLI entrypoint
- Command router
- Session manager
- Agent loop
- Tool registry
- Permission service
- 核心工具：read / edit / grep / glob / shell

## 第二版再补
- task system
- compaction
- memory
- plugin / MCP
- structured output / IDE bridge

## 为什么这样拆
因为真正最容易把项目做死的，不是“少功能”，而是“第一版就试图做成 Claude Code 全家桶”。

---

# 7. 推荐的数据模型草图

## 7.1 会话模型

```ts
interface SessionState {
  sessionId: string
  cwd: string
  messages: Message[]
  config: SessionConfig
  usage: UsageState
  permissionMode: PermissionMode
}
```

## 7.2 工具模型

```ts
interface Tool<I = any, O = any> {
  name: string
  description: string
  inputSchema: JSONSchema
  execute(input: I, ctx: ToolContext): Promise<O>
}
```

## 7.3 权限请求模型

```ts
interface PermissionRequest {
  kind: 'shell' | 'file_write' | 'network_write' | 'task_spawn'
  summary: string
  raw?: string
  cwd?: string
  riskLevel?: 'low' | 'medium' | 'high'
}
```

## 7.4 任务模型

```ts
interface TaskRecord {
  id: string
  type: 'shell' | 'agent' | 'workflow'
  status: 'pending' | 'running' | 'done' | 'failed' | 'killed'
  createdAt: number
  updatedAt: number
  summary: string
  outputPath?: string
}
```

## 7.5 事件流模型

```ts
type AgentEvent =
  | { type: 'token'; text: string }
  | { type: 'tool_start'; toolName: string; input: unknown }
  | { type: 'tool_end'; toolName: string; output: unknown }
  | { type: 'status'; message: string }
  | { type: 'error'; error: string }
```

---

# 8. 三个最关键的设计原则

## 原则一：控制面与执行面分离
- Command Router / Config / Session control 是控制面
- Agent Loop / Tool Runtime / Task System 是执行面

如果这两层不分，项目很快会变成不可维护的一坨。

## 原则二：工具不是函数，而是运行时能力
一个真正可用的 tool，至少要挂上：
- 会话
- 权限
- 中断
- 状态通知
- 文件/命令服务
- 结果回收

所以不要把 tool 理解成 `fn(input) => output` 这么简单。

## 原则三：长期可用性比单次聪明更重要
真正难的不是“调一次模型”，而是：
- 长会话不爆
- 危险操作可控
- 任务可追踪
- 历史能恢复
- 工具输出不会污染上下文

这也是为什么 session / permission / compaction / task 这些层，比 prompt 小技巧更重要。

---

# 9. 你可以怎么用这份草图

## 用法一：作为阅读 Claude Code 的“抽象地图”
当你回头看 `QueryEngine.ts`、`query.ts`、`Tool.ts`、`Task.ts` 时，可以不断问自己：
- 这段代码属于哪一层？
- 它解决的是哪个架构问题？
- 它有没有越层？

## 用法二：作为你自己做 AI CLI 的设计蓝图
你可以直接按本文档拆模块、建目录、定接口，然后逐步落地 MVP。

## 用法三：作为评审别的 AI Agent 项目的分析模板
拿任意一个 AI coding agent 项目，对照这 10 层去看，很容易快速判断：
- 它是不是只有 demo 级能力
- 它缺的是工具、权限、任务，还是长期上下文
- 它是结构完整，还是只是 prompt 驱动拼出来的

---

# 10. 最后的判断

如果把这整套东西压缩成一句话：

> **成熟 AI CLI 的核心竞争力，不在“能不能调用工具”，而在“能不能把会话、工具、权限、任务、上下文和扩展能力组织成一个可持续运行的系统”。**

这也是为什么真正值得学的，不是某一段 prompt，而是这整套骨架设计。
