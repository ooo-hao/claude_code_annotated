# Claude Code 源码阅读 Checklist

> 仓库：`/Users/chenxianhao/Desktop/project/claude_code_annotated`
> 目标：按“先主干、后配套、再专题”的顺序，把 Claude Code 的核心设计吃透，而不是在 50 万行代码里迷路。
>
> 使用方式：按顺序勾选；每个阶段都先读“必读”，有余力再读“延伸”；每阶段结束后写一段自己的总结。

---

# 0. 阅读原则

- [ ] **不要从 `src/` 随机开啃**，优先按本清单顺序读
- [ ] **先看入口和主循环**，再看工具、任务、权限、记忆
- [ ] **遇到 `feature('XXX')` 不要误判为完整功能存在**，很多只是 feature gate 痕迹
- [ ] **读每个模块前先问自己**：它解决什么问题？输入/输出是什么？状态放哪？
- [ ] **每个阶段至少写 3~10 句笔记**，不然很容易看完就忘

---

# 1. 第一阶段：建立全局地图（先知道自己在看什么）

## 1.1 仓库说明与边界

### 必读
- [ ] `README_CN.md`
- [ ] `QUICKSTART.md`
- [ ] `package.json`

### 阅读目标
- [ ] 明确这份源码的来源：来自 `@anthropic-ai/claude-code` 公开包拆解
- [ ] 明确源码并不完整，很多模块在构建时被裁掉
- [ ] 明确构建体系依赖 Bun 的编译期特性，而不是普通 Node 项目
- [ ] 明确项目主战场在 `src/`

### 本阶段产出
- [ ] 写下：这份仓库**是什么**
- [ ] 写下：这份仓库**不是什么**
- [ ] 写下：你最想追的 2~3 条主线（例如 agent loop / tools / permissions）

---

# 2. 第二阶段：入口与命令分流（搞清程序怎么启动）

## 2.1 CLI 入口

### 必读
- [ ] `src/entrypoints/cli.tsx`

### 重点关注
- [ ] `--version` 等 fast path
- [ ] dynamic import 的使用方式
- [ ] `feature('XXX')` 如何参与入口裁剪
- [ ] daemon / bridge / bg session / mcp 等模式怎么在入口处分流

### 你应该回答的问题
- [ ] Claude Code 是单入口还是多入口？
- [ ] 它为什么大量使用 dynamic import？
- [ ] 入口层为什么需要把不同运行模式尽早切开？

## 2.2 命令注册总表

### 必读
- [ ] `src/commands.ts`

### 延伸
- [ ] `src/commands/help/index.js`
- [ ] `src/commands/status/index.js`
- [ ] `src/commands/session/index.js`
- [ ] `src/commands/review.ts`
- [ ] `src/commands/mcp/index.js`
- [ ] `src/commands/skills/index.js`

### 重点关注
- [ ] 命令是如何 import / 注册 / 汇总的
- [ ] 哪些命令是内建的，哪些是 feature-gated 的
- [ ] skills / plugins / dynamic skills 怎么接进命令体系
- [ ] `/xxx` 命令系统和主 agent loop 的边界在哪

### 本阶段产出
- [ ] 把命令按类别手工分组：会话 / 开发 / 环境 / 扩展 / 诊断
- [ ] 用 3~5 句话总结：命令层到底是“外壳”还是“核心”

---

# 3. 第三阶段：主循环（整份源码最重要的一层）

> 这是最高优先级阶段。时间有限的话，至少把这部分真正看懂。

## 3.1 会话级控制器

### 必读
- [ ] `src/QueryEngine.ts`

### 重点关注
- [ ] `QueryEngineConfig`
- [ ] `submitMessage()`
- [ ] `mutableMessages`
- [ ] `totalUsage`
- [ ] `canUseTool` 的包装方式
- [ ] `thinkingConfig`
- [ ] session persistence / replay / compact / SDK status

### 你应该回答的问题
- [ ] QueryEngine 管的是“单次请求”还是“整个会话”？
- [ ] 它持有哪些跨轮次状态？
- [ ] 它和 `query.ts` 的职责边界是什么？

## 3.2 真正的 agent loop

### 必读
- [ ] `src/query.ts`

### 配套跳读（边看边定位）
- [ ] `src/query/config.js`
- [ ] `src/query/deps.js`
- [ ] `src/query/tokenBudget.ts`
- [ ] `src/query/stopHooks.js`
- [ ] `src/query/transitions.js`

### 重点关注
- [ ] messages 是怎么进入 query 的
- [ ] system prompt / user context / system context 怎么拼装
- [ ] 调模型后如何判断是否进入 `tool_use`
- [ ] tool 执行后如何回写 `tool_result`
- [ ] query 循环如何继续下一轮
- [ ] compact / retry / fallback / summary 插在什么位置

### 建议阅读顺序
- [ ] 先找 `export async function* query(...)`
- [ ] 看 state 是怎么定义的
- [ ] 看一次完整循环从“准备请求”到“输出文本/工具结果”的路径
- [ ] 第二遍再看异常恢复、预算控制、compact 等支线

### 本阶段产出
- [ ] 画出 Claude Code 最小代理循环图
- [ ] 用自己的话解释：为什么 `tool_use` 让它从“聊天”变成“agent”
- [ ] 列出 query 里你认为最关键的 5 个辅助模块

---

# 4. 第四阶段：工具抽象层（Claude Code 的能力接口）

## 4.1 工具统一抽象

### 必读
- [ ] `src/Tool.ts`

### 重点关注
- [ ] `ToolUseContext`
- [ ] permission context
- [ ] app state 访问方式
- [ ] notification / UI hooks
- [ ] message / file cache / abort / compact / attribution 等上下文能力

### 你应该回答的问题
- [ ] Claude Code 的 tool 和普通 function calling 最大区别是什么？
- [ ] 一个 tool 要运行起来，为什么需要那么多上下文？
- [ ] 权限、UI、状态、通知这些东西为什么都在 ToolUseContext 里？

### 本阶段产出
- [ ] 用你自己的话定义：什么叫“富上下文工具运行时”

---

# 5. 第五阶段：核心工具实现（先看最值钱的 20%）

> 不要平均用力。先把 coding agent 的地基工具看懂。

## 5.1 文件读取工具

### 必读
- [ ] `src/tools/FileReadTool/FileReadTool.ts`
- [ ] `src/tools/FileReadTool/prompt.ts`
- [ ] `src/tools/FileReadTool/limits.ts`

### 延伸
- [ ] `src/tools/FileReadTool/UI.tsx`
- [ ] `src/tools/FileReadTool/imageProcessor.ts`

### 重点关注
- [ ] 读取能力怎么定义
- [ ] 大文件/图片处理边界
- [ ] 输出截断或限制怎么做

## 5.2 文件编辑工具

### 必读
- [ ] `src/tools/FileEditTool/FileEditTool.ts`
- [ ] `src/tools/FileEditTool/prompt.ts`
- [ ] `src/tools/FileEditTool/types.ts`

### 延伸
- [ ] `src/tools/FileEditTool/utils.ts`
- [ ] `src/tools/FileEditTool/constants.ts`
- [ ] `src/tools/FileEditTool/UI.tsx`

### 重点关注
- [ ] 编辑操作的输入结构
- [ ] patch/替换策略怎么设计
- [ ] diff/UI 是怎么配合的

## 5.3 Bash 工具

### 必读
- [ ] `src/tools/BashTool/BashTool.tsx`
- [ ] `src/tools/BashTool/prompt.ts`
- [ ] `src/tools/BashTool/bashPermissions.ts`
- [ ] `src/tools/BashTool/bashSecurity.ts`

### 第二层必读
- [ ] `src/tools/BashTool/commandSemantics.ts`
- [ ] `src/tools/BashTool/pathValidation.ts`
- [ ] `src/tools/BashTool/readOnlyValidation.ts`
- [ ] `src/tools/BashTool/shouldUseSandbox.ts`

### 延伸
- [ ] `src/tools/BashTool/destructiveCommandWarning.ts`
- [ ] `src/tools/BashTool/sedEditParser.ts`
- [ ] `src/tools/BashTool/sedValidation.ts`
- [ ] `src/tools/BashTool/modeValidation.ts`
- [ ] `src/tools/BashTool/UI.tsx`
- [ ] `src/tools/BashTool/BashToolResultMessage.tsx`

### 重点关注
- [ ] shell 命令执行的主路径
- [ ] 权限校验和危险命令识别
- [ ] sandbox 决策逻辑
- [ ] 输出结果怎么返回给上层 agent loop

## 5.4 搜索类工具

### 必读
- [ ] `src/tools/GlobTool/`
- [ ] `src/tools/GrepTool/`

### 重点关注
- [ ] 为什么 coding agent 必须有 glob/grep
- [ ] 它们和 read/edit/bash 的协作顺序

### 本阶段产出
- [ ] 总结 Claude Code 最小 coding toolbelt 包含哪些工具
- [ ] 画出一条“读代码 -> 搜索 -> 修改 -> 执行验证”的典型链路

---

# 6. 第六阶段：任务系统与子代理（从聊天走向工作流）

## 6.1 任务模型

### 必读
- [ ] `src/Task.ts`

### 重点关注
- [ ] `TaskType`
- [ ] `TaskStatus`
- [ ] task id 生成方式
- [ ] task output 如何落盘
- [ ] terminal state 如何判断

## 6.2 Task 工具族

### 必读
- [ ] `src/tools/TaskCreateTool/TaskCreateTool.ts`
- [ ] `src/tools/TaskGetTool/TaskGetTool.ts`
- [ ] `src/tools/TaskListTool/TaskListTool.ts`
- [ ] `src/tools/TaskOutputTool/TaskOutputTool.tsx`
- [ ] `src/tools/TaskStopTool/TaskStopTool.ts`
- [ ] `src/tools/TaskUpdateTool/TaskUpdateTool.ts`

### 配套 prompt / constants
- [ ] `src/tools/TaskCreateTool/prompt.ts`
- [ ] `src/tools/TaskGetTool/prompt.ts`
- [ ] `src/tools/TaskListTool/prompt.ts`
- [ ] `src/tools/TaskStopTool/prompt.ts`
- [ ] `src/tools/TaskUpdateTool/prompt.ts`
- [ ] `src/tools/TaskCreateTool/constants.ts`
- [ ] `src/tools/TaskGetTool/constants.ts`
- [ ] `src/tools/TaskListTool/constants.ts`
- [ ] `src/tools/TaskOutputTool/constants.ts`
- [ ] `src/tools/TaskUpdateTool/constants.ts`

## 6.3 AgentTool

### 必读
- [ ] `src/tools/AgentTool/AgentTool.tsx`
- [ ] `src/tools/AgentTool/runAgent.ts`
- [ ] `src/tools/AgentTool/loadAgentsDir.ts`
- [ ] `src/tools/AgentTool/prompt.ts`

### 延伸
- [ ] `src/tools/AgentTool/forkSubagent.ts`
- [ ] `src/tools/AgentTool/resumeAgent.ts`
- [ ] `src/tools/AgentTool/agentToolUtils.ts`
- [ ] `src/tools/AgentTool/agentMemory.ts`
- [ ] `src/tools/AgentTool/agentMemorySnapshot.ts`
- [ ] `src/tools/AgentTool/builtInAgents.ts`
- [ ] `src/tools/AgentTool/built-in/generalPurposeAgent.ts`
- [ ] `src/tools/AgentTool/built-in/planAgent.ts`
- [ ] `src/tools/AgentTool/built-in/verificationAgent.ts`

### 重点关注
- [ ] 子代理是如何被定义和调度的
- [ ] task 和 agent 是什么关系
- [ ] 内建 agent 与用户定义 agent 的边界

### 本阶段产出
- [ ] 用一句话定义：Claude Code 的任务系统解决了什么问题
- [ ] 用一句话定义：AgentTool 为什么是“高级能力”而不是普通工具

---

# 7. 第七阶段：上下文压缩与预算控制（生产可运行性的核心）

## 7.1 Compact 主模块

### 必读
- [ ] `src/services/compact/autoCompact.ts`
- [ ] `src/services/compact/compact.ts`
- [ ] `src/services/compact/prompt.ts`

### 第二层必读
- [ ] `src/services/compact/microCompact.ts`
- [ ] `src/services/compact/apiMicrocompact.ts`
- [ ] `src/services/compact/sessionMemoryCompact.ts`

### 延伸
- [ ] `src/services/compact/grouping.ts`
- [ ] `src/services/compact/compactWarningHook.ts`
- [ ] `src/services/compact/compactWarningState.ts`
- [ ] `src/services/compact/postCompactCleanup.ts`
- [ ] `src/services/compact/timeBasedMCConfig.ts`

## 7.2 预算控制与工具输出收敛

### 必读
- [ ] `src/query/tokenBudget.ts`
- [ ] `src/services/toolUseSummary/toolUseSummaryGenerator.js`
- [ ] `src/services/tools/StreamingToolExecutor.js`
- [ ] `src/services/tools/toolOrchestration.js`
- [ ] `src/utils/toolResultStorage.js`

### 重点关注
- [ ] 自动 compact 如何触发
- [ ] compact 前后 message 如何重建
- [ ] 为什么 tool output 不能原样无限保留
- [ ] token budget 如何影响 agent 的行为

### 本阶段产出
- [ ] 总结：Claude Code 如何防止长会话把自己撑爆
- [ ] 列出 3 个你认为最值得借鉴的生产策略

---

# 8. 第八阶段：权限系统与安全边界（真正商用必须看）

## 8.1 权限核心类型与规则

### 必读
- [ ] `src/utils/permissions/PermissionMode.ts`
- [ ] `src/utils/permissions/PermissionResult.ts`
- [ ] `src/utils/permissions/PermissionRule.ts`
- [ ] `src/utils/permissions/permissions.ts`
- [ ] `src/utils/permissions/permissionSetup.ts`
- [ ] `src/utils/permissions/permissionsLoader.ts`

## 8.2 风险分类与 shell 规则

### 必读
- [ ] `src/utils/permissions/bashClassifier.ts`
- [ ] `src/utils/permissions/dangerousPatterns.ts`
- [ ] `src/utils/permissions/shellRuleMatching.ts`
- [ ] `src/utils/permissions/pathValidation.ts`
- [ ] `src/utils/permissions/filesystem.ts`

## 8.3 权限状态迁移与解释

### 延伸
- [ ] `src/utils/permissions/getNextPermissionMode.ts`
- [ ] `src/utils/permissions/permissionExplainer.ts`
- [ ] `src/utils/permissions/denialTracking.ts`
- [ ] `src/utils/permissions/autoModeState.ts`
- [ ] `src/utils/permissions/classifierDecision.ts`
- [ ] `src/utils/permissions/classifierShared.ts`
- [ ] `src/utils/permissions/bypassPermissionsKillswitch.ts`
- [ ] `src/utils/permissions/shadowedRuleDetection.ts`
- [ ] `src/utils/permissions/permissionRuleParser.ts`
- [ ] `src/utils/permissions/yoloClassifier.ts`

### 结合工具回看
- [ ] 回看 `src/tools/BashTool/bashPermissions.ts`
- [ ] 回看 `src/tools/BashTool/bashSecurity.ts`
- [ ] 回看 `src/tools/FileEditTool/FileEditTool.ts`
- [ ] 回看 `src/tools/FileWriteTool/`

### 本阶段产出
- [ ] 总结 Claude Code 的 permission model
- [ ] 画出“工具请求 -> 风险判断 -> 用户确认/自动决策 -> 执行”的路径
- [ ] 判断它更偏“强约束”还是“高自由度+补救”

---

# 9. 第九阶段：记忆、持久化与会话连续性

## 9.1 memdir

### 必读
- [ ] `src/memdir/memdir.ts`
- [ ] `src/memdir/findRelevantMemories.ts`
- [ ] `src/memdir/memoryScan.ts`
- [ ] `src/memdir/memoryTypes.ts`
- [ ] `src/memdir/paths.ts`

### 延伸
- [ ] `src/memdir/memoryAge.ts`
- [ ] `src/memdir/teamMemPaths.ts`
- [ ] `src/memdir/teamMemPrompts.ts`

## 9.2 SessionMemory

### 必读
- [ ] `src/services/SessionMemory/sessionMemory.ts`
- [ ] `src/services/SessionMemory/sessionMemoryUtils.ts`
- [ ] `src/services/SessionMemory/prompts.ts`

## 9.3 状态与会话落盘

### 必读
- [ ] `src/bootstrap/state.ts`
- [ ] `src/utils/sessionStorage.js`
- [ ] `src/assistant/sessionHistory.ts`

### 重点关注
- [ ] memory prompt 怎么注入
- [ ] session id 如何关联历史
- [ ] transcript / replay / compact boundary 怎么衔接
- [ ] 它如何处理“不是一次性聊天”的场景

### 本阶段产出
- [ ] 总结 Claude Code 对“长期上下文”的思路
- [ ] 写出它和普通聊天机器人最大的结构差异

---

# 10. 第十阶段：MCP、扩展、插件生态

## 10.1 MCP 工具层

### 必读
- [ ] `src/tools/MCPTool/`
- [ ] `src/tools/ListMcpResourcesTool/`
- [ ] `src/tools/ReadMcpResourceTool/`
- [ ] `src/tools/McpAuthTool/`

## 10.2 MCP 服务层

### 必读
- [ ] `src/services/mcp/`

## 10.3 技能与插件

### 必读
- [ ] `src/tools/SkillTool/`
- [ ] `src/commands/skills/`
- [ ] `src/utils/plugins/`
- [ ] `src/services/plugins/`

### 本阶段产出
- [ ] 总结 Claude Code 的扩展边界在哪里
- [ ] 判断 MCP / skills / plugins 三者分别扮演什么角色

---

# 11. 第十一阶段：UI 与交互层（最后看，别先陷进去）

## 推荐只抓主干

### 必读
- [ ] `src/components/App.tsx`
- [ ] `src/components/Messages.tsx`
- [ ] `src/components/Message.tsx`
- [ ] `src/components/MessageRow.tsx`
- [ ] `src/components/MessageResponse.tsx`
- [ ] `src/cli/structuredIO.ts`
- [ ] `src/cli/remoteIO.ts`

### 延伸
- [ ] `src/components/permissions/`
- [ ] `src/components/tasks/`
- [ ] `src/components/mcp/`
- [ ] `src/components/skills/`

### 重点关注
- [ ] TUI/REPL 如何展示消息与工具进度
- [ ] 权限弹框怎么呈现
- [ ] structured IO 与 remote IO 的差异

### 本阶段产出
- [ ] 用一句话总结：UI 层是“视图”还是“逻辑中心”

---

# 12. 第十二阶段：专题分析与隐藏能力（放到最后）

## 文档专题

### 必读
- [ ] `docs/zh/01-遥测与隐私分析.md`
- [ ] `docs/zh/02-隐藏功能与模型代号.md`
- [ ] `docs/zh/03-卧底模式分析.md`
- [ ] `docs/zh/04-远程控制与紧急开关.md`
- [ ] `docs/zh/05-未来路线图.md`

## 对照代码阅读

### 推荐回看
- [ ] `src/services/analytics/`
- [ ] `src/services/remoteManagedSettings/`
- [ ] `src/bridge/`
- [ ] `src/buddy/`

### 阅读目标
- [ ] 结合代码去看专题，而不是只吃瓜
- [ ] 区分“已落地的代码证据”和“由代码推断出的产品方向”
- [ ] 区分“外部可见能力”和“内部 feature gate 痕迹”

---

# 13. 如果时间很少：最小必读清单

> 只读这些，也能抓住 Claude Code 的主骨架。

### 核心 10 文件 / 目录
- [ ] `README_CN.md`
- [ ] `src/entrypoints/cli.tsx`
- [ ] `src/commands.ts`
- [ ] `src/QueryEngine.ts`
- [ ] `src/query.ts`
- [ ] `src/Tool.ts`
- [ ] `src/tools/FileReadTool/FileReadTool.ts`
- [ ] `src/tools/FileEditTool/FileEditTool.ts`
- [ ] `src/tools/BashTool/BashTool.tsx`
- [ ] `src/utils/permissions/permissions.ts`

### 如果还能多看一层
- [ ] `src/Task.ts`
- [ ] `src/services/compact/autoCompact.ts`
- [ ] `src/services/compact/compact.ts`
- [ ] `src/memdir/memdir.ts`
- [ ] `docs/zh/04-远程控制与紧急开关.md`

---

# 14. 读完后建议整理的 3 份个人笔记

## 14.1 架构总图
- [ ] 入口层
- [ ] 命令层
- [ ] QueryEngine
- [ ] query loop
- [ ] Tool runtime
- [ ] task system
- [ ] permissions
- [ ] compact
- [ ] memory

## 14.2 工具运行时设计笔记
- [ ] tool schema
- [ ] tool context
- [ ] permission gate
- [ ] tool orchestration
- [ ] output budget
- [ ] UI / progress

## 14.3 你自己的 AI CLI 设计稿
- [ ] command router
- [ ] agent loop
- [ ] tool registry
- [ ] task manager
- [ ] permission system
- [ ] memory injection
- [ ] context compaction
- [ ] MCP/plugin layer

---

# 15. 一句话版结论

- **第一优先级**：`cli.tsx -> commands.ts -> QueryEngine.ts -> query.ts -> Tool.ts`
- **第二优先级**：`FileReadTool / FileEditTool / BashTool / permissions`
- **第三优先级**：`Task / AgentTool / compact / memdir`
- **最后再看**：`UI / docs 专题 / hidden features`

如果你是以“做自己的 AI coding assistant”为目标阅读，这个顺序基本不会错。
