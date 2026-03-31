# 第11章 可观察性内建：Progress、通知、Transcript 与性能指标

## 本章要回答的问题

1. Claude Code 的“可观察性”为什么不靠日志拼凑，而是内建到任务框架里？
2. Progress、通知、transcript 三条链路如何在不互相污染的情况下保持一致？
3. 性能指标如何以最小侵入方式贯穿 React 树并在进程退出时落盘？

## 关键源码入口

- [`src/components/App.tsx`](https://github.com/ohmuyi/claude-codes/blob/main/src/components/App.tsx)：根部挂载 `AppStateProvider`、`StatsProvider`、`FpsMetricsProvider`
- [`src/tasks/LocalAgentTask/LocalAgentTask.tsx`](https://github.com/ohmuyi/claude-codes/blob/main/src/tasks/LocalAgentTask/LocalAgentTask.tsx)：`createProgressTracker()`、`updateProgressFromMessage()`、`enqueueAgentNotification()`
- [`src/tasks/LocalMainSessionTask.ts`](https://github.com/ohmuyi/claude-codes/blob/main/src/tasks/LocalMainSessionTask.ts)：`startBackgroundSession()`、`completeMainSessionTask()`
- [`src/tasks/LocalShellTask/LocalShellTask.tsx`](https://github.com/ohmuyi/claude-codes/blob/main/src/tasks/LocalShellTask/LocalShellTask.tsx)：stall watchdog、`enqueueShellNotification()`

## 运行机制

Progress 先解决“我们能看见什么”。`LocalAgentTask.tsx` 的 `ProgressTracker` 把 token 与 `tool_use` 拆开统计：输入 token 是 API 累积值，用“最新值覆盖”；输出 token 是 per-turn，用“累计求和”。同时它从 `message.message.content` 中扫描 `tool_use`，并借助 `getToolSearchOrReadInfo()` 把活动分类为 read 或 search，最后保留一个固定窗口 `MAX_RECENT_ACTIVITIES`，让 UI 能以 O(1) 的成本展示“最近在干什么”。

通知再解决“我们要让谁看见”。`enqueueAgentNotification()`、`enqueueMainSessionNotification()`、`enqueueShellNotification()` 都把事件编码成带标签的 XML 文本，例如 `<task_notification>`、`<task_id>`、`<output_file>`、`<status>`、`<summary>`，并统一走 `enqueuePendingNotification()` 入队。这里有两个关键防抖：其一是 task 上的 `notified` 标志，通过 `updateTaskState()` 原子 check-and-set 防止重复通知；其二是通知入队会 `abortSpeculation(setAppState)`，避免 prompt suggestion 的预计算结果引用了过期的任务输出。

Transcript 解决“发生过什么”。主会话与子 agent 都把输出落到磁盘 JSONL，但主会话旁路更强调隔离：`startBackgroundSession()` 在 `for await (query())` 的每个 event 上调用 `recordSidechainTranscript([event], taskId, lastRecordedUuid)`，保证后台任务的 TaskOutput 与 transcript 同步推进。与此同时它在内存里用 `tokenCount/toolCount/recentActivities` 更新 `tasks[taskId].progress`，让 UI 既能看“文件里有什么”，也能看“这段时间做了多少事”。

性能指标解决“系统跑得怎么样”。`components/App.tsx` 把 `StatsProvider` 与 `FpsMetricsProvider` 固定挂在根上：前者用直方图 reservoir sampling 维护 p50、p95、p99，并在 `process.on('exit')` 时把结果写回项目配置；后者输出 `averageFps` 与 `low1PctFps`，用于捕捉低帧尾部表现。这样做的效果是，业务组件只需要“记录”，不需要“负责持久化”。

## 设计权衡

把通知编码成 XML 字符串的好处是跨边界简单：同一条消息既能被 TUI 展示，也能作为 queued command 进入模型上下文，且 `mode: 'task-notification'` 与队列优先级确保“用户输入不会被系统消息饿死”。代价是语义较隐式，必须靠稳定的 tag 约定与防重复机制来维持正确性。

Progress 与 transcript 分离是一次明确取舍。进度 tick 很高频，源码在 session storage 层明确规定 progress 不属于 transcript，因此 progress 必须以 UI 状态存在并随时可丢。收益是持久化文件不会被高频噪声撑爆，代价是“复盘时的细粒度进度”不一定能完全还原。

## 小结

可观察性在这里不是附属品，而是任务框架的默认产物：Progress 给出连续的“过程”，通知给出离散的“事件”，transcript 给出可回放的“事实”，Stats 与 FPS 给出系统层面的“健康度”。四者通过统一的 `AppState` 与队列系统协作，但彼此又刻意隔离，避免高频数据污染持久化与上下文。

## 导航

- [上一章：Agent 视图如何长出来，前后台切换、主会话旁路与面板投影](./10-Agent-视图-前后台切换与面板投影.md)
- [返回目录](../Claude-Code源码精讲.md)
- [下一章：交互体验的最后一公里，低打扰反馈、双击拖选与卡顿自诊断](./12-交互体验最后一公里-低打扰反馈与自诊断.md)
