# 第4章 任务：为什么 Claude Code 还需要 Task 这一层

## 本章要回答的问题

1. 已经有 query loop 和 `tool_use`，为什么还要再抽一层 Task？
2. Task 到底管理的是什么，和一次 `tool_use` 的生命周期有什么不同？
3. background、`/clear`、通知与输出落盘，为什么更适合放在 Task 而不是 Tool 里？

## 关键源码入口

- [`src/Task.ts`](https://github.com/ohmuyi/claude-codes/blob/main/src/Task.ts)：`TaskType`、`TaskStatus`、`TaskStateBase`、`generateTaskId()`、`createTaskStateBase()`
- [`src/tasks/LocalAgentTask/LocalAgentTask.tsx`](https://github.com/ohmuyi/claude-codes/blob/main/src/tasks/LocalAgentTask/LocalAgentTask.tsx)：`registerAsyncAgent()`、`registerAgentForeground()`、`backgroundAgentTask()`、`killAsyncAgent()`、`completeAgentTask()`
- [`src/tasks/LocalMainSessionTask.ts`](https://github.com/ohmuyi/claude-codes/blob/main/src/tasks/LocalMainSessionTask.ts)：`registerMainSessionTask()`、`startBackgroundSession()`、`completeMainSessionTask()`
- [`src/tasks/LocalShellTask/LocalShellTask.tsx`](https://github.com/ohmuyi/claude-codes/blob/main/src/tasks/LocalShellTask/LocalShellTask.tsx)：`spawnShellTask()`、`registerForeground()`、`backgroundAll()`、`startStallWatchdog()`

## 运行机制

从概念上看，`tool_use` 描述的是“模型这一回合想做什么”，它天然是回合内、请求内的。Task 解决的是另一类问题：工作会跨回合延续，甚至在 UI 清空或用户继续输入后还要继续跑，并且需要一个可观察、可回收、可通知的外壳。

`Task.ts` 先把这层外壳统一成 `TaskStateBase`：每个任务有 `id/type/status/description`，还有两组特别关键的字段。第一组是可追踪输出：`outputFile` 指向 `getTaskOutputPath(id)`，`outputOffset` 记录“读到哪里了”，这让“增量输出”成为通用能力，而不依赖某个 Tool 的实现细节。第二组是生命周期护栏：`notified` 防止重复通知，`isTerminalTaskStatus()` 明确哪些状态不可再转移，便于回收与防止向“死任务”注入消息。

具体到实现，`LocalShellTask` 把 shell 的数据流交给 `ShellCommand.taskOutput`，Task 只负责注册状态、切换前后台、结束时清理并发通知。它还加了一个很工程化的补丁：`startStallWatchdog()` 定期 `stat()` 输出文件并 `tailFile()`，只有尾部像交互式 prompt，例如 `(y/n)` 或 `Press Enter`，才发一条 `<task_notification>`，提醒“可能卡在交互输入”。这类“围绕执行的可观测性”很难优雅地塞进 BashTool 的同步调用里，却非常适合 Task 这种长生命周期容器。

`LocalAgentTask` 则把“子 agent 的对话与进度”也包装成 Task：`registerAsyncAgent()` 会 `initTaskOutputAsSymlink(agentId, getAgentTranscriptPath(...))`，让 output 指向独立 transcript；`backgroundAgentTask()` 只是翻转 `isBackgrounded` 并通过 `backgroundSignalResolvers` 触发中断点，真正的执行循环仍在别处。`LocalMainSessionTask` 更进一步，它把“主会话的一次 query”也当作 Task 背景化：为避免 `/clear` 后写回主 transcript 造成污染，它明确强调不要用主会话 transcript，而要写 sidechain transcript，并逐条 `recordSidechainTranscript()`，保证任务即使跨 `sessionId` 变化也能读到连续输出。

## 设计权衡

Task 的代价是引入第二套状态机和更多竞态面：同一任务既可能因用户操作被 kill，又可能因进程自然结束而完成。源码用几种策略降低复杂度。第一，状态更新统一走 `updateTaskState()`，并在“无变化”时返回同一引用避免无谓渲染。第二，通知去重靠 `notified` 的原子 check-and-set。第三，面向 UI 的“短暂保留”用 `evictAfter` 和 `PANEL_GRACE_MS`，既允许用户看完结果，又能 GC。

另一个权衡是“落盘与安全”。Task 输出必须落盘才能被异步观察，但落盘又会引入路径攻击面。`diskOutput.ts` 通过 `O_NOFOLLOW` 防 symlink 跟随，并把任务输出目录放在 project temp 下，同时把 `sessionId` 在首次调用时缓存，避免 `/clear` 让新旧路径分裂导致后台任务读不到输出。这些都说明：Task 层不是为了抽象而抽象，而是把“跨回合的工程问题”集中治理。

## 小结

`tool_use` 解决“这一回合做什么”，Task 解决“这件事做多久、怎么观察、怎么收尾”。把 background、输出增量、通知去重、`/clear` 存活、安全落盘都放进 Task，使 Tool 保持专注于“执行一次调用”，而系统获得可组合的异步基础设施。

## 导航

- [上一章：工具，Tool 接口为何是整个系统的协议中心](./03-工具-Tool-接口为何是协议中心.md)
- [返回目录](../Claude-Code源码精讲.md)
- [下一章：权限，权限不是一个开关，而是一条决策链](./05-权限-权限不是开关而是决策链.md)
