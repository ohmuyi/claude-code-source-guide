# 第2章 回合：一次用户输入如何变成一个可递归的 turn

## 本章要回答的问题

- Claude Code 所谓的 turn 为什么是“可递归”的，而不是一次请求一次响应？
- `QueryEngine.submitMessage()` 和 `query()` 分别负责哪一段职责边界？
- `tool_use` 出现后，系统如何把 `tool_result` 组织成下一次模型调用的输入？

## 关键源码入口

- [`src/QueryEngine.ts`](https://github.com/ohmuyi/claude-codes/blob/main/src/QueryEngine.ts)
- [`src/query.ts`](https://github.com/ohmuyi/claude-codes/blob/main/src/query.ts)
- 关键连接点：`QueryEngine.submitMessage()` 内部 `for await (const message of query({...}))`

## 运行机制

概念上，一个用户输入触发的是“一次会话内的一个 turn”，但这个 turn 可能包含多轮模型调用。原因是模型可能在输出中产生 `tool_use`，而每次工具执行都会生成新的 `tool_result`，这些结果又必须回到模型上下文里继续推理，直到不再需要 follow-up。

`QueryEngine.submitMessage()` 先把“用户输入”变成“可查询的上下文”。它通过 `fetchSystemPromptParts()` 组装 `systemPrompt`，再创建 `ProcessUserInputContext`，把 `tools`、`commands`、`thinkingConfig`、`mcpClients`、以及 `readFileState` 等放入上下文。随后调用 `processUserInput()`，在真正发起模型请求前处理 `/slash` 命令、附件、允许工具规则 `allowedTools`、以及可能的模型切换 `modelFromUserInput`。

一个重要的机制细节是“先落盘再请求”。`submitMessage()` 在进入 `query()` 之前就对 `messagesFromUserInput` 做 `recordTranscript(messages)`，避免“用户发出消息但 API 尚未返回时进程被杀”导致会话不可恢复。此处还会把 `allowedTools` 写回 `toolPermissionContext.alwaysAllowRules.command`，让后续权限系统对本回合的选择可见。

进入 `query.ts` 后，`query()` 只是一个薄封装，核心在 `queryLoop()` 的 `while (true)`。它维护一个可变 `State`，其中 `messages` 与 `toolUseContext` 是每次递归的输入，`turnCount` 则刻画“在同一次用户输入驱动下已经进行了多少次 agentic 继续”。每次迭代都会初始化或递增 `queryTracking { chainId, depth }`，把这次递归嵌套关系写入上下文供分析与调试使用。

迭代的前半段负责“把上下文变成可提交的请求”。它会对 `messagesForQuery` 执行 microcompact、可能的 context collapse、以及 autocompact，然后流式调用模型，收集 `assistantMessages` 和 `toolUseBlocks`。只要出现 `tool_use`，`needsFollowUp` 就会被置为 true，这就是 turn 继续的开关。

后半段负责“执行工具并递归”。工具执行由 `StreamingToolExecutor` 或 `runTools()` 产生一个异步迭代器，持续 yield 工具过程中产生的消息。所有工具的 `tool_result` 会被归一化进 `toolResults`，并和新产生的 attachments，例如 memory、skill discovery、queued command，一起并入下一次递归的输入 `messages: [...messagesForQuery, ...assistantMessages, ...toolResults]`。最后在 `query_recursive_call` 处更新 `state` 并 `continue`，这就是“可递归的 turn”。

`maxTurns` 的限制也在这一刻生效：每次准备递归时计算 `nextTurnCount = turnCount + 1`，超过上限就 yield 一个 `attachment: max_turns_reached` 并返回。`QueryEngine` 接到该 attachment 后会立刻产出 `error_max_turns` 的 result，形成端到端的硬边界。

## 设计权衡

把“递归”实现为 `while (true) + State`，而不是递归调用函数栈，保证了长链路下的稳定性，也让 `transition`、`turnCount`、以及各类恢复策略，例如 `max_output_tokens` recovery、stop hooks、token budget continuation，能统一写成“更新 state 后 continue”。

把 `tool_result` 组织成 `UserMessage` 是一个强约束，但它换来了模型接口一致性。模型只需要遵循 `tool_use` 与 `tool_result` 的消息协议，系统内部就能把工具执行、附件注入、以及 stop hook 的产物都表达成“消息流”，从而用同一个管道驱动 UI、SDK、与 transcript。

生成器式的 `yield` 让调用方可以流式消费，但也要求系统在边界上非常谨慎，例如 `submitMessage()` 里对 assistant message 的 transcript 写入选择 fire-and-forget，避免阻塞导致 `message_delta` 无法及时更新 usage 和 `stop_reason`。

## 小结

一次用户输入并不等价于一次模型调用，它驱动的是一个可能多次“模型输出 `tool_use` -> 执行工具 -> 将 `tool_result` 回注 -> 再次调用模型”的递归 turn。`QueryEngine` 负责把输入与会话状态准备好并落盘，`query.ts` 负责在统一的消息协议下循环推进，直到不需要 follow-up 或触发硬退出条件。

## 导航

- [上一章：启动，从 main.tsx 到可交互会话](./01-启动-从-main-tsx-到可交互会话.md)
- [返回目录](../Claude-Code源码精讲.md)
- [下一章：工具，Tool 接口为何是整个系统的协议中心](./03-工具-Tool-接口为何是协议中心.md)
