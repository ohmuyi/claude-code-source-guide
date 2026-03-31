# 第8章 统一交互状态机：从 createStore 到 AppStateStore

## 本章要回答的问题

- 为什么这里选择 `createStore` 这种极小 Store，而不是 Redux 或复杂的 reducer 体系？
- `AppStateStore` 如何把“交互状态”变成可推导、可订阅的单一事实源？
- 副作用，例如持久化、外部同步、权限模式联动，应该挂在哪一层才不失控？

## 关键源码入口

- [`src/state/store.ts`](https://github.com/ohmuyi/claude-codes/blob/main/src/state/store.ts)
- [`src/state/AppStateStore.ts`](https://github.com/ohmuyi/claude-codes/blob/main/src/state/AppStateStore.ts)
- 关联封装与副作用：[`src/state/AppState.tsx`](https://github.com/ohmuyi/claude-codes/blob/main/src/state/AppState.tsx)、[`src/state/onChangeAppState.ts`](https://github.com/ohmuyi/claude-codes/blob/main/src/state/onChangeAppState.ts)

## 运行机制

`createStore<T>` 的设计几乎“刻意贫穷”：只提供 `getState`、`setState(updater)`、`subscribe`，内部用一个 `Set<Listener>` 广播变化，并用 `Object.is(next, prev)` 作为最小的变更判定。这种接口把“状态如何变化”的权力交回调用方：每次更新必须显式写成 `prev => next`，自然形成一条可审计的状态迁移链，而不是隐含在 action、middleware、selector 的多层间接里。

`AppStateStore.ts` 则把“交互状态机”的状态空间一次性摊开成 `AppState`：从 `toolPermissionContext`、`expandedView`、`footerSelection` 到 `tasks`、`plugins`、`mcp`、`promptSuggestion/speculation`。这里的关键不是字段多，而是边界清晰：UI 读 `AppState`，后台系统也读 `AppState`，任何影响交互的事实必须最终反映为某个字段的变更。`getDefaultAppState()` 把初始状态集中构造出来，尤其把 permission mode 的初始分支，例如 teammate 加 `plan_mode_required`，前置为单点决策，避免后续组件各自推断。

真正把它变成“统一状态机”的，是 `AppState.tsx` 的 Provider 与订阅策略。Provider 只创建一次 store，并把 `onChangeAppState` 作为 `createStore` 的 `onChange` 回调挂进去。读取侧使用 `useSyncExternalStore(store.subscribe, get, get)`，并要求 selector 返回稳定引用，配合 `Object.is` 达到“只在关心的值变化时重渲染”。写入侧则通过 `useSetAppState()` 暴露稳定的 `setState` 引用，让只写不读的组件不被状态变化牵连。

副作用集中在 `onChangeAppState.ts`。它不是监听“事件”，而是对 `oldState/newState` 做差分，并把外部世界同步，例如 permission mode 对 CCR 或 SDK 的元数据同步、配置持久化、settings 变化触发缓存清理与 env 重新应用，收敛成单个 choke point。这样，任何路径只要调用 `setAppState`，就天然获得一致的外部联动，不需要在每个命令、对话框、快捷键里复制粘贴同步逻辑。

## 设计权衡

极简 store 的代价是约束不来自框架，而来自团队纪律：没有 action 类型系统，就必须靠类型与命名约定维护可读性；没有 reducer 分层，就要避免在业务点散落“半更新”，否则会出现状态迁移难以追溯的问题。这里用 `onChangeAppState` 把副作用统一收口，本质上是用“集中式后处理”换“分布式显式调用”的复杂度。

把 `AppState` 做大，也是一种取舍。它让任何功能都能找到落点，但也会诱发“把临时 UI 状态塞进全局”的冲动，导致重渲染压力与心智负担增加。代码用 selector 加 `useSyncExternalStore` 抵消渲染成本，用 `DeepImmutable` 表达“默认不可变”的意图，但最终仍要依赖边界意识去筛选什么该进全局。

最后是可测试性与演进：`createStore` 的纯度很高，便于在 headless、SDK 或测试环境复用；同时它也不提供时间旅行、事务等高级能力。这里的策略是把“高级能力”外置成约定，例如用 updater 形成原子更新，用 `onChange` 做集中副作用，而不是把复杂度塞进 store 内核。

## 小结

`createStore` 提供最小可用的状态容器，`AppStateStore` 提供完整的状态空间定义，`AppState.tsx` 把它们绑定到 React，并用 `useSyncExternalStore` 实现精确订阅。配合 `onChangeAppState` 的差分副作用，这套结构把“交互”从零散事件提升为统一状态机：只要状态是单一事实源，机制就自然收敛，扩展也更少走形。

## 导航

- [上一章：终端不是字符串输出，Claude Code 的 TUI 运行时](./07-终端不是字符串输出-TUI-运行时.md)
- [返回目录](../Claude-Code源码精讲.md)
- [下一章：后台任务总线，Shell、Agent 与 Main Session 的三种后台化](./09-后台任务总线-Shell-Agent-Main-Session.md)
