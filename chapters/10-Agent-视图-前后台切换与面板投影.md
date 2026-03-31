# 第10章 Agent 视图如何长出来：前后台切换、主会话旁路与面板投影

## 本章要回答的问题

1. 为什么“看见一个 Agent 的输出”不是跳转页面，而是从 `AppState` 里自然长出来的视图？
2. 前台执行与后台执行如何在同一套 `TaskState` 上切换，而不打断任务本身？
3. “主会话后台化”为什么要走旁路，既不污染主 transcript，又能随时拉回前台？

## 关键源码入口

- [`src/state/AppStateStore.ts`](https://github.com/ohmuyi/claude-codes/blob/main/src/state/AppStateStore.ts)：定义 `tasks`、`foregroundedTaskId`、`viewingAgentTaskId`、`viewSelectionMode`、`coordinatorTaskIndex`
- [`src/tasks/LocalAgentTask/LocalAgentTask.tsx`](https://github.com/ohmuyi/claude-codes/blob/main/src/tasks/LocalAgentTask/LocalAgentTask.tsx)：`LocalAgentTaskState` 的 `isBackgrounded`、`retain`、`diskLoaded`、`evictAfter`
- [`src/tasks/LocalMainSessionTask.ts`](https://github.com/ohmuyi/claude-codes/blob/main/src/tasks/LocalMainSessionTask.ts)：`registerMainSessionTask()`、`startBackgroundSession()`、`foregroundMainSessionTask()`

## 运行机制

概念上可以把“Agent 视图”理解为一次投影：同一份 `tasks` 数据，既能在主消息区渲染，也能在面板里渲染，差别只是哪些 task 被选中。`AppStateStore.ts` 把这些选择位做成全局状态，比如 `foregroundedTaskId` 决定主消息区是否显示某个后台任务的 `messages`，而 `viewingAgentTaskId` 和 `viewSelectionMode` 决定是否进入“看某个 agent transcript”的模式。

对普通子 Agent，前后台切换是直接翻 `isBackgrounded`。`LocalAgentTask.tsx` 里既支持“先前台跑一会再后台化”的 `registerAgentForeground()`，也提供 `backgroundAgentTask()`：它原子地把 `tasks[taskId].isBackgrounded` 置为 `true`，并通过 `backgroundSignalResolvers` 解开等待，让上层 agent loop 知道“该把 UI 让回给用户输入”。这意味着 UI 切换不等于停止任务，停止只发生在 `killAsyncAgent()` 里通过 `abortController.abort()`。

面板投影依赖两件事：生命周期与可回看性。`LocalAgentTaskState` 用 `evictAfter` 表达“面板上还能留多久”，用 `retain` 表达“UI 正在持有这个 task”。当用户进入 agent 视图时，状态机会把 `retain` 置为 `true` 并清掉 `evictAfter`，这样 task 不会被面板淘汰，同时允许把流式 `messages` 立刻 append 到 `tasks[taskId].messages`。而 `diskLoaded` 则标记“是否已把磁盘上的 JSONL transcript 与内存里的 live suffix 合并完成”，避免重复 bootstrap。

主会话的旁路更像是一次隔离实验。`LocalMainSessionTask.ts` 复用 `LocalAgentTaskState`，但把 `agentType` 固定为 `'main-session'`，并用独立 taskId 与独立 transcript 文件承载后台查询的输出：`registerMainSessionTask()` 明确禁止写回主会话 transcript，而是改为 `initTaskOutputAsSymlink(taskId, getAgentTranscriptPath(asAgentId(taskId)))`，保证用户 `/clear` 后主对话仍然干净，而后台任务仍能继续写自己的 sidechain。需要拉回前台时，`foregroundMainSessionTask()` 只改 `foregroundedTaskId` 和该 task 的 `isBackgrounded = false`，任务本身不重启，消息展示则通过主视图区读取 task 的 `messages` 实现。

## 设计权衡

把“视图开关”放进 `AppState` 的好处是可组合：面板选择、主区前台化、转看某个 agent 都可以叠加而不互相抢控制权，代价是状态空间变大，需要用清晰的谓词收口，例如 `isPanelAgentTask()` 统一规定哪些 task 应该出现在面板里，避免过滤逻辑分散漂移。

旁路 transcript 的代价是多一套写入与合并逻辑：既要保证“磁盘先写、再 yield”，又要处理“UI retain 后 live 先 append”的并发，最后靠 UUID 去重合并前缀与后缀。收益是确定性与安全性：后台主会话不会腐蚀主 transcript，也不会被 `/clear` 的 `sessionId` 变化打断。

## 小结

这一章的要点是把“Agent 视图”从 UI 术语降维为状态投影：`tasks` 是事实，`foregroundedTaskId/viewingAgentTaskId` 是观察角度，`retain/diskLoaded/evictAfter` 是面板与回看的一组最小机制。主会话后台化之所以走旁路，是为了把“继续跑”和“继续写”从主对话中隔离出去，最终让前后台切换成为一次纯状态变换，而不是一次重启。

## 导航

- [上一章：后台任务总线，Shell、Agent 与 Main Session 的三种后台化](./09-后台任务总线-Shell-Agent-Main-Session.md)
- [返回目录](../Claude-Code源码精讲.md)
- [下一章：可观察性内建，Progress、通知、Transcript 与性能指标](./11-可观察性内建-Progress-通知-Transcript-性能.md)
