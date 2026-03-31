# 第3章 工具：Tool 接口为何是整个系统的协议中心

## 本章要回答的问题

- `Tool` 为什么不只是一个 `call()`，而要同时携带 schema、权限、渲染等信息？
- `ToolUseContext` 在系统中扮演什么角色，为什么它比函数参数更像协议？
- 工具池如何统一内置工具与 MCP 工具，并把权限策略前置到“模型可见”层面？

## 关键源码入口

- [`src/Tool.ts`](https://github.com/ohmuyi/claude-codes/blob/main/src/Tool.ts)
- [`src/tools.ts`](https://github.com/ohmuyi/claude-codes/blob/main/src/tools.ts)
- 关键结构：`export type Tool`、`export type Tools`、`export type ToolUseContext`、`assembleToolPool()`、`getTools()`

## 运行机制

概念上，Tool 是 Claude Code 对“能力”的抽象边界，但它不是单纯的函数表，而是一份可被模型理解、可被权限系统约束、可被 UI 呈现的协议。也因此 `Tool.ts` 里 `Tool` 的核心不是实现细节，而是约定：输入如何描述、执行如何报告进度、输出如何回流到模型、以及在不同会话模式下如何展示。

在机制层面，`Tool` 至少包含三条通道。第一条是模型通道：`inputSchema` 或 `inputJSONSchema` 描述可调用形状，`description()` 生成提示文本，`mapToolResultToToolResultBlockParam()` 把执行结果映射为标准 `tool_result` block。第二条是执行通道：`call(input, context, canUseTool, parentMessage, onProgress)` 明确工具执行依赖的最小环境，并允许通过 `ToolResult.newMessages` 与 `contextModifier` 影响后续消息流与上下文。第三条是交互通道：大量 `renderToolUseMessage`、`renderToolResultMessage`、`renderToolUseProgressMessage` 等方法把同一个工具调用以 UI 组件的形式表达出来，让交互式 REPL 能以“工具块”而不是纯文本呈现过程。

`ToolUseContext` 是这套协议的中心载体。它把会话级状态与能力打包进一个对象：`options.tools`、`options.commands`、`options.mainLoopModel`、`thinkingConfig`、`mcpClients` 等描述“模型可用的世界”，`abortController` 与 `setInProgressToolUseIDs` 描述“执行期控制”，`getAppState/setAppState` 描述“状态读写”，`handleElicitation`、`requestPrompt` 描述“需要用户交互时如何回到 UI”。从外部看，工具只拿到一个 context，但从系统角度这是各模块共享的协议面，任何跨模块协作都通过它完成。

为了降低实现成本又不牺牲安全默认，`Tool.ts` 提供 `buildTool()`，把 `isConcurrencySafe`、`isReadOnly`、`isDestructive`、`checkPermissions` 等设置为偏保守的默认值。这样新工具不需要重复样板代码，同时避免因遗漏而默认并发安全或默认只读这类危险假设。

工具池的组装则在 `tools.ts` 完成。`getAllBaseTools()` 定义内置工具集合，并用 `feature()` 与 `process.env.USER_TYPE` 等条件编译或运行时 gating 控制可见性。`getTools(permissionContext)` 进一步根据 `CLAUDE_CODE_SIMPLE`、REPL mode 等模式过滤，并用 `filterToolsByDenyRules()` 把“被 blanket deny 的工具”在进入模型之前就剔除。最终 `assembleToolPool(permissionContext, mcpTools)` 把内置工具与 MCP 工具合并，按 name 排序并用 `uniqBy` 保证同名时内置优先，同时保持“内置工具是连续前缀”以稳定 prompt cache 的断点位置。

## 设计权衡

把 Tool 定义成“模型协议 + 执行协议 + UI 协议”的一体化接口，换来的是一致性：同一个工具既能被 `query.ts` 编排执行，又能被 REPL 渲染，还能被权限系统按同一套匹配器约束。代价是接口很大，工具作者需要理解的面更广，抽象上也更像框架而不是库。

`shouldDefer`、`alwaysLoad` 这类字段把“节省 tokens”前置到工具层，适配 `ToolSearch` 的延迟加载策略。它能显著降低默认 prompt 体积，但也引入一次“先搜索再调用”的间接性，对时延与实现复杂度都有影响。

`filterToolsByDenyRules()` 在“模型可见”层面就移除工具，提高安全与合规确定性，但会减少模型自适应解决问题的空间。代码选择了让 deny 规则与运行时权限检查使用同一 matcher，避免出现“模型以为能用但运行时全拒绝”的错配。

## 小结

在 Claude Code 里，Tool 不是普通的函数集合，而是一份贯穿模型、执行与交互的统一协议。`ToolUseContext` 让工具调用拥有稳定的环境语义，`tools.ts` 则让内置与 MCP 工具在同一套权限与缓存稳定性约束下合并成“模型可见的能力集合”，这也解释了为什么 Tool 接口会成为整个系统的协议中心。

## 导航

- [上一章：回合，一次用户输入如何变成一个可递归的 turn](./02-回合-一次用户输入如何变成可递归的-turn.md)
- [返回目录](../Claude-Code源码精讲.md)
- [下一章：任务，为什么 Claude Code 还需要 Task 这一层](./04-任务-为什么还需要-Task-层.md)
