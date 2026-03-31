# 第9章 后台任务总线：Shell、Agent 与 Main Session 的三种后台化

## 本章要回答的问题

- “后台化”在这里具体指什么？是线程、进程，还是 UI 的一种投影方式？
- Shell、Agent、Main Session 三类任务为何需要不同的后台化机制？
- 如何在后台继续产生可追踪输出，同时在前台保持交互流畅与一致通知？

## 关键源码入口

- [`src/tasks/LocalShellTask/LocalShellTask.tsx`](https://github.com/ohmuyi/claude-codes/blob/main/src/tasks/LocalShellTask/LocalShellTask.tsx)
- [`src/tasks/LocalAgentTask/LocalAgentTask.tsx`](https://github.com/ohmuyi/claude-codes/blob/main/src/tasks/LocalAgentTask/LocalAgentTask.tsx)
- [`src/tasks/LocalMainSessionTask.ts`](https://github.com/ohmuyi/claude-codes/blob/main/src/tasks/LocalMainSessionTask.ts)

## 运行机制

这套“后台任务总线”的核心，不是把执行搬到别的线程，而是把任务统一注册到 `AppState.tasks`，并把输出与通知统一编码成可消费的通道。对 UI 来说，后台任务就是一段可被面板展示、可被 foreground 或 background 切换的 `TaskState`；对模型与外部消费者来说，后台任务通过 `<task_notification>` 这样的结构化消息形成闭环。

`LocalShellTask` 的后台化依赖 OS 进程与磁盘输出。`spawnShellTask` 直接用 `shellCommand.taskOutput.taskId` 作为任务 ID，注册后调用 `shellCommand.background(taskId)`，让数据流自动落到 `getTaskOutputPath(taskId)` 对应文件，不需要再挂 stdout 流监听。它额外启动 `startStallWatchdog`：定期 `stat` 输出文件大小，若 45 秒无增长则 `tailFile` 取尾部并用 `looksLikePrompt` 判断是否卡在交互式提示，再 enqueue 一次性通知，指导“kill 并用非交互参数重跑”。任务结束时 `flushAndCleanup`，再用 XML 标签打包 summary、status、output file path，最后 `evictTaskOutput` 做落盘清理。

`LocalAgentTask` 的后台化更像“异步迭代器的控制权切换”。`registerAsyncAgent` 默认 `isBackgrounded: true`，并通过 `initTaskOutputAsSymlink(agentId, getAgentTranscriptPath(...))` 把输出绑定到独立 transcript。对于前台运行后再后台化的场景，`registerAgentForeground` 会创建一个 `backgroundSignal` Promise，并把 resolver 放进 `backgroundSignalResolvers`。当用户触发 `backgroundAgentTask` 或自动超时背景化时，它只做两件事：把 `isBackgrounded` 置真，并 resolve 该 promise，让上层 agent 执行循环可以在 `Promise.race(nextMessage, backgroundSignal)` 中及时切出前台渲染路径，避免前台继续占用 UI 帧预算。Agent 的进度则用 `updateProgressFromMessage` 扫描 assistant message 中的 `tool_use` 与 token usage，存入 `task.progress`，以便后台面板显示“最近做了什么”。

`LocalMainSessionTask` 处理的是“把主对话本身后台化”。它复用 `LocalAgentTaskState`，但用 `agentType: 'main-session'` 标识语义，并生成以 `s` 开头的任务 ID。关键点是隔离 transcript：`registerMainSessionTask` 明确不要写回主会话文件，而是把任务输出 symlink 到独立路径，保证 `/clear` 后不会污染新的会话上下文。`startBackgroundSession` 通过 `query()` 的 `for await` 拉取事件流，并在每条消息到达时 `recordSidechainTranscript([event], taskId, lastRecordedUuid)` 增量落盘，同时用 `roughTokenCountEstimation` 与 `tool_use` 计数更新 `task.progress`。完成时 `completeMainSessionTask` 只在任务仍后台时发送通知，若已被 foreground，则改走 SDK bookend 关闭，避免“用户正在看却又弹一条完成通知”。

## 设计权衡

三类后台化的差异，本质上是数据源不同：Shell 是外部进程流，Agent 是内部推理循环，Main Session 是主事件流的复制分支。统一之处是都以 `AppState.tasks` 为总线、以磁盘输出为稳定载体、以结构化 notification 为对外接口。代价是需要处理大量竞态：后台化与完成可能交错，因此要有 `notified` 原子检查；前台与后台的切换要避免重复注册与重复 SDK 事件，因此分别提供“就地翻转”的辅助函数。

磁盘作为输出媒介也带来取舍。它让 UI 重启、`/clear`、面板回放变得可靠，但引入了 I/O 延迟与一致性问题，所以你会看到对 `flush()`、symlink 重连、以及“输出停滞但仍在等待输入”的启发式判断。另一端，后台任务状态变更会触发 `abortSpeculation(setAppState)`，这是把“提示建议的推测结果”视为可失效缓存的明确选择，用保守策略换取正确性。

最后是用户体验层面的取舍：后台化要让前台尽快恢复提示符，但又不能丢失上下文与进度。Main Session 通过 sidechain transcript 与 agent context 隔离做到“可继续跑且可回看”，Agent 通过 `backgroundSignal` 做到“可中断前台渲染而不中断执行”，Shell 通过 watchdog 做到“模型能被提醒去处理交互式卡点”。它们共同组成了一套可操作的后台总线，而不是单纯的并发执行。

## 小结

这里的“后台化”是把执行与展示解耦：执行继续发生，展示从主视图移到任务面板，并用统一的任务状态与输出路径维持可追踪性。`LocalShellTask` 依赖 TaskOutput 文件流，`LocalAgentTask` 依赖 `backgroundSignal` 切换控制权，`LocalMainSessionTask` 依赖 sidechain transcript 复制主事件流。理解这三条路径，就能看清 Claude Code 如何在单个终端里同时容纳交互、渲染与并发任务。

## 导航

- [上一章：统一交互状态机，从 createStore 到 AppStateStore](./08-统一交互状态机-从-createStore-到-AppStateStore.md)
- [返回目录](../Claude-Code源码精讲.md)
- [下一章：Agent 视图如何长出来，前后台切换、主会话旁路与面板投影](./10-Agent-视图-前后台切换与面板投影.md)
