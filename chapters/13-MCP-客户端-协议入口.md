# 第13章 MCP 客户端：把工具、命令与资源统一成协议入口

## 本章要回答的问题

1. MCP 在 Claude Code 里到底被当成什么，它解决的是“接入”还是“执行”？
2. 为什么同一套客户端要同时承载 tools、prompts 和 resources，它们如何映射到 CLI 的内部抽象？
3. 多服务器、多传输协议、鉴权与超时这些“现实因素”，在源码里是怎么被兜住的？

## 关键源码入口

- [`src/services/mcp/client.ts`](https://github.com/ohmuyi/claude-codes/blob/main/src/services/mcp/client.ts)
- `connectToServer(...)`、`ensureConnectedClient(...)`、`clearServerCache(...)`
- `fetchToolsForClient(...)`、`fetchCommandsForClient(...)`、`fetchResourcesForClient(...)`
- `getMcpToolsCommandsAndResources(...)`、`reconnectMcpServerImpl(...)`
- `callMCPToolWithUrlElicitationRetry(...)`、`processMCPResult(...)`

## 运行机制

概念上，`client.ts` 把 MCP 视为“统一协议入口”：外部世界不再以“某个插件的工具、某个脚本、某个资源库”的形态接入，而是被规约成 MCP 的三类能力。tools 是可调用的动作，prompts 被转换成可触发的 Command，resources 则是可浏览与可读取的对象集合。这样做的直接收益是，Claude Code 内部只需要围绕 `Tool` 与 `Command` 两个面向模型的接口组织权限、渲染、进度与错误，具体的连接细节被压到协议层。

机制上，连接由 `connectToServer` 承担并被缓存：它根据配置选择 `StdioClientTransport`、`SSEClientTransport`、`StreamableHTTPClientTransport`、IDE 或 WebSocket 相关 transport，完成握手、能力发现与 cleanup 注册，并把结果包装成 `MCPServerConnection`。工具与命令的“列举”分别走 `tools/list` 与 `prompts/list`，在 `fetchToolsForClient` 和 `fetchCommandsForClient` 中被转换为本地 `Tool` 和 `Command`，命名上用 `buildMcpToolName` 或 `mcp__{server}__{prompt}` 形成稳定前缀，避免与内置能力冲突，同时把 `mcpInfo` 挂到 Tool 上供权限与审计使用。resources 通过 `resources/list` 拉取，但为了让“资源”也能像工具一样被统一调用，客户端还会在合适时机注入 `ListMcpResourcesTool` 与 `ReadMcpResourceTool`，把“列目录、读内容”变成跨服务器一致的入口。

执行路径上，`Tool.call(...)` 最终落到 `callMCPToolWithUrlElicitationRetry` 与 `callMCPTool`：前者处理协议层的 URL elicitation，例如服务端返回 `ElicitRequestSchema`，在 REPL 或结构化 IO 模式下把“需要用户打开链接完成认证”变成可恢复的两段式流程；后者负责真正的 `client.callTool`，并用 `Promise.race` 叠加近似“无限”的 tool timeout，同时把进度事件转成 `MCPProgress` 回调。结果回到 `processMCPResult`，在输出过大时根据开关决定截断还是落盘，把“模型上下文限制”转译为“可追溯的文件指令”。

## 设计权衡

第一组权衡是“统一抽象”对“真实多样性”的压缩。MCP 服务器可能把 OpenAPI 文档塞进 description，源码用长度上限截断以保护 prompt 体积，但代价是丢掉长尾细节，只能依赖服务端更好的 annotations 与可发现性。类似地，统一命名和 Unicode 清洗提升了稳定性，却也让一些服务端自定义展示需要走 override 分支。

第二组权衡是“快速可用”对“鉴权正确性”的博弈。远端 server 的 401 会被记入本地缓存并设置 TTL，`getMcpToolsCommandsAndResources` 甚至会跳过近期失败或“发现但无 token”的服务器，以避免启动时被几十个无效探测拖慢；代价是状态可能短暂过期，需要用户显式 `/mcp` 或触发重连来刷新。对 `claudeai-proxy` 这类场景，客户端选择“401 时最多重试一次且必须 tokenChanged”，本质是在全量风暴和局部失败之间做了工程化折中。

## 小结

`client.ts` 的核心不是“又接了一个协议”，而是把外部能力压成统一的 `Tool`、`Command`、resource 入口，并围绕连接缓存、鉴权退避、执行重试、进度与大结果处理建立一条可控的数据通路。你在上层看到的是“多了很多工具”，在底层其实是“多了一个可治理的接入面”。

## 导航

- [上一章：交互体验的最后一公里，低打扰反馈、双击拖选与卡顿自诊断](./12-交互体验最后一公里-低打扰反馈与自诊断.md)
- [返回目录](../Claude-Code源码精讲.md)
- [下一章：远程会话，把本地 CLI 变成可接管的运行中环境](./14-远程会话-本地-CLI-到可接管环境.md)
