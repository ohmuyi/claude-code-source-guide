# 第14章 远程会话：把本地 CLI 变成可接管的运行中环境

## 本章要回答的问题

1. 远程会话要解决的根问题是什么，为什么不是简单的“把命令跑在服务器上”？
2. 本地进程如何被远端调度、续租、重连，并且在多会话下仍然可控？
3. “可接管”具体意味着什么，消息、权限与中断如何闭环？

## 关键源码入口

- [`src/bridge/bridgeMain.ts`](https://github.com/ohmuyi/claude-codes/blob/main/src/bridge/bridgeMain.ts)
- [`src/remote/RemoteSessionManager.ts`](https://github.com/ohmuyi/claude-codes/blob/main/src/remote/RemoteSessionManager.ts)
- `bridgeMain(...)`、`runBridgeLoop(...)`
- `SessionsWebSocket`：[`src/remote/SessionsWebSocket.ts`](https://github.com/ohmuyi/claude-codes/blob/main/src/remote/SessionsWebSocket.ts)

## 运行机制

概念上，远程会话把本地 CLI 变成“运行中环境的代理”：代码、凭据、文件系统与工具链都在本机，远端只负责会话编排与界面承载。这样既保留了本地开发环境的真实性，又让用户可以在网页或移动端接管正在运行的 session，做到“换设备不断线”。

在 `bridgeMain.ts` 中，这个代理以 `runBridgeLoop` 的形式长期运行。它先完成信任与鉴权前置检查，例如要求用户先用正常 `claude` 流程接受 workspace trust，然后进入 poll loop：通过 `api.pollForWork(...)` 拉取 work item，按类型处理 healthcheck 或 session。对 session work，它维护 `activeSessions`、`sessionWorkIds`、`sessionIngressTokens` 等映射，做到“同一个 session 重派发时更新 token，而不是重复 spawn”。当需要启动新会话时，通过 `SessionSpawner` 拉起子进程，并根据 `spawnMode` 决定是否为每个 session 创建隔离的 git worktree，完成后把 worktree 信息登记进 `sessionWorktrees`，在 `onSessionDone` 或 shutdown 阶段统一回收。

可接管的关键在“连接不断、租约不断”。源码把这件事拆成两条线：一条是 server 侧 work lease 的维持，`heartbeatActiveWorkItems` 用 `sessionIngressTokens` 对每个 active work 做 heartbeat，401 或 403 时用 `api.reconnectSession(...)` 触发重新派发，避免 ACK 过的 work 因 token 过期而永久卡死；另一条是子进程侧的连接续命，`createTokenRefreshScheduler` 会在 JWT 临近过期时刷新，必要时通过 reconnect 让 server 重新派发新 token。多会话时还引入 at-capacity 策略与 `capacityWake`，在容量满时进入“心跳但少 poll”的模式，有 session 结束立刻唤醒以抢新 work。

在客户端接入侧，`RemoteSessionManager` 组合了 `SessionsWebSocket` 与 HTTP 发送：WebSocket 负责订阅 session 消息流，HTTP 负责把用户输入送回去。控制面消息走 `control_request/control_response/control_cancel_request`：当远端请求 `can_use_tool` 权限时，manager 会把请求缓存到 `pendingPermissionRequests` 并回调 UI，用户决定后再发回 `allow` 或 `deny` 的响应；需要中断时调用 `cancelSession()` 发送 `interrupt`。`SessionsWebSocket` 自带有限重连、ping keepalive，以及对 4001、4003 等 close code 的差异化处理，使“短暂抖动可恢复，永久拒绝要尽快失败”。

## 设计权衡

第一组权衡是“强一致的远程控制”对“本地安全边界”的让步。桥接必须尊重 workspace trust、HTTPS baseUrl 限制、以及 permission mode 的策略校验；代价是启动路径更长、更像一个守护进程，而不是一次性命令。与此同时，多会话带来更高吞吐，但也必须引入 worktree 隔离、容量上限与更复杂的清理逻辑，否则并发会话会互相污染工作目录。

第二组权衡是“稳定性”对“响应性”的平衡。poll loop 的 backoff、sleep detection、心跳模式都在减少服务端压力与避免假死，但它也意味着某些状态变化不是即时可见，必须靠 `capacityWake`、token refresh 与重连事件把关键路径拉回实时。对用户而言，这是把最坏体验从“随机断线、任务丢失”换成“偶尔延迟、但可恢复”。

## 小结

`bridgeMain.ts` 把远程会话做成了一套完整生命周期系统：poll 获取工作、spawn 本地运行、心跳续租、token 刷新、失败退避、worktree 隔离与退出清理。`RemoteSessionManager` 与 `SessionsWebSocket` 则把“接管”落到消息与权限协议上，保证你接管的是一个正在运行、可中断、可授权的真实本地环境。

## 导航

- [上一章：MCP 客户端，把工具、命令与资源统一成协议入口](./13-MCP-客户端-协议入口.md)
- [返回目录](../Claude-Code源码精讲.md)
- [下一章：记忆系统，把当前对话沉淀为未来上下文](./15-记忆系统-沉淀未来上下文.md)
