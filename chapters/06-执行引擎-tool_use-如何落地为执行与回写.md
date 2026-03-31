# 第6章 执行引擎：tool_use 如何真正落地为执行、并发与回写

## 本章要回答的问题

1. `tool_use` 从“消息块”到“真实执行”中间经历了哪些阶段？
2. 并发是怎么做的，为什么要先分批再执行？
3. tool 的结果、进度、`contextModifier`，如何回写到下一轮 query？

## 关键源码入口

- [`src/services/tools/toolOrchestration.ts`](https://github.com/ohmuyi/claude-codes/blob/main/src/services/tools/toolOrchestration.ts)：`runTools()`、`partitionToolCalls()`、`runToolsConcurrently()`、`runToolsSerially()`
- [`src/services/tools/toolExecution.ts`](https://github.com/ohmuyi/claude-codes/blob/main/src/services/tools/toolExecution.ts)：`runToolUse()`、`checkPermissionsAndCallTool()`
- [`src/query.ts`](https://github.com/ohmuyi/claude-codes/blob/main/src/query.ts)：query loop 里收集 `ToolUseBlock[]`，执行 `runTools(...)` 并把 `tool_result` 注入后续请求

## 运行机制

从概念上看，`tool_use` 是声明式的，它只说“要调用哪个 Tool、参数是什么、`tool_use_id` 是什么”。执行引擎的责任是把这段声明转成三件可验证的事实：能不能执行，例如 schema、权限、hooks；怎么执行，例如串行或并行；执行完怎么把结果写回消息流与上下文。

在 `query.ts` 中，query loop 在流式接收 assistant 消息时累积 `toolUseBlocks`，当进入工具执行阶段，选择 streaming executor 或直接走 `runTools(toolUseBlocks, assistantMessages, canUseTool, toolUseContext)`。`runTools()` 的第一步不是立刻并发，而是 `partitionToolCalls()`：它对每个 `tool_use` 先 `safeParse` 输入，再调用 `tool.isConcurrencySafe(parsedInput.data)`，把连续的并发安全调用合成一批，其余则单个成批。这样做的含义很务实：默认并发是不安全的，只有 Tool 明确声明“读多写少且无共享副作用”时才允许并行。

并发批次由 `runToolsConcurrently()` 执行，并通过 `CLAUDE_CODE_MAX_TOOL_USE_CONCURRENCY` 控制上限。这里有一个关键细节：并发批次里产生的 `contextModifier` 不会立刻生效，而是先按 `toolUseID` 收集，等整批结束再按原顺序应用到 `currentContext`，避免并发下的上下文写入竞态。串行批次则由 `runToolsSerially()` 一次一个执行，`contextModifier` 立即作用，语义更直观。

单个 `tool_use` 的执行落在 `runToolUse()`。它先通过 `findToolByName()` 找 Tool，必要时从 `getAllBaseTools()` 里按 alias 做兼容回退；找不到就构造一个 `is_error` 的 `tool_result`。找到了以后进入 `checkPermissionsAndCallTool()`：先 zod 校验输入，失败就返回 `InputValidationError` 并附带 deferred schema 的提示；然后跑 `runPreToolUseHooks()`，把 progress 与附件消息插入结果序列；接着通过 `resolveHookPermissionDecision(...)` 得到最终权限结论，deny 就立刻生成 `tool_result` 终止，allow 才调用 `tool.call(...)`。执行过程中通过 `Stream` 把 progress 和最终结果合并成一个异步序列，`query.ts` 在 `for await` 中逐条 yield 给上游 UI 或 SDK，同时把 `normalizeMessagesForAPI([update.message])` 的 user 或 `tool_result` 收集起来，为下一次模型调用准备输入。最后，`update.newContext` 会被带入下一轮迭代，形成真正的“回写”。

## 设计权衡

并发的收益是吞吐，风险是语义漂移。这里的核心取舍是“默认串行，显式声明才并发”，并且对 `contextModifier` 采取延迟应用策略，换取确定性。代价是实现更复杂，但边界很清晰：并发只发生在 `isConcurrencySafe` 批次内，且上下文修改被集中处理。

另一个权衡是“可观测性与稳定性”。为了让 hooks、权限、UI、遥测都能看到一致的输入，执行引擎支持 `backfillObservableInput` 在“可观察副本”上补字段，同时避免修改要回流 API 的原始输入，防止 prompt cache 与 transcript hash 漂移。你会看到很多看似啰嗦的保护逻辑，本质是在保证系统能流式展示、能恢复、能测试重放的前提下，把 `tool_use` 落地成可靠执行。

## 小结

执行引擎把 `tool_use` 拆成“分批并发策略”“单次调用的验证与权限”“进度与结果的流式产出”“上下文的确定性回写”四个层次。它让声明式的 `tool_use` 变成可运行、可并行、可审计、可继续下一轮的真实系统行为。

## 导航

- [上一章：权限，权限不是一个开关，而是一条决策链](./05-权限-权限不是开关而是决策链.md)
- [返回目录](../Claude-Code源码精讲.md)
- [下一章：终端不是字符串输出，Claude Code 的 TUI 运行时](./07-终端不是字符串输出-TUI-运行时.md)
