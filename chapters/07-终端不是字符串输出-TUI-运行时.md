# 第7章 终端不是字符串输出：Claude Code 的 TUI 运行时

## 本章要回答的问题

- 为什么在 TUI 里，“打印一段文本”不等于“把字符串写到 stdout”？
- 输入事件如何从 `stdin` 的字节流，变成可组合的 key、mouse、focus 事件？
- 渲染如何做到“像 GUI 一样重绘”，同时又尽量避免整屏闪烁？

## 关键源码入口

- [`src/ink/components/App.tsx`](https://github.com/ohmuyi/claude-codes/blob/main/src/ink/components/App.tsx)
- [`src/ink/renderer.ts`](https://github.com/ohmuyi/claude-codes/blob/main/src/ink/renderer.ts)
- 关联调用点：[`src/ink/ink.tsx`](https://github.com/ohmuyi/claude-codes/blob/main/src/ink/ink.tsx)

## 运行机制

TUI 的“运行时”由两条主线组成：输入侧把终端协议解析成结构化事件，输出侧把 React 树计算成屏幕网格的差量更新。`App.tsx` 不是一个普通的根组件，它更像终端设备驱动的适配层，负责 raw mode 引用计数、终端模式开关、事件分发与异常兜底。

在 `handleSetRawMode(true)` 中，Ink 不只开启 `stdin.setRawMode(true)`，还会同步写入一组终端模式：括号粘贴 `EBP/DBP`、焦点报告 `EFE/DFE`、扩展按键 `ENABLE_KITTY_KEYBOARD` 与 `ENABLE_MODIFY_OTHER_KEYS`。这些不是“优化”，而是为了让上层交互具备稳定语义，例如区分 `ctrl+shift+<letter>`，以及在失焦时可靠收尾选择拖拽。输入读取采用 `readable` 加 `read()` 的主动抽取模式，并在读取间隔超过 `STDIN_RESUME_GAP_MS` 时触发 `onStdinResume`，用来修复 tmux 重连、睡眠唤醒导致的 DEC 私有模式被重置的问题。

解析路径集中在 `processInput`：`parseMultipleKeypresses` 维护 `keyParseState`，把字节流拆成 `ParsedInput[]`。它把“批量输入”作为一等公民，借助 `reconciler.discreteUpdates(processKeysInBatch, ...)` 将一批 key 的副作用压进一次高优先级更新，避免粘贴或长按导致的更新深度爆炸。对不完整 escape 序列，`flushIncomplete` 采用计时器延迟冲刷，并在 `stdin.readableLength > 0` 时重新挂起，避免渲染阻塞导致的“先触发 timer、后到达剩余字节”而误判成单独的 Escape。

输出侧由 `createRenderer` 构建每帧 `Frame`。它复用 `Output`，使字符切分与 grapheme clustering 的缓存跨帧存活，避免稳定画面也反复分词。渲染核心是 `renderNodeToOutput`，但真正决定性能与正确性的是“能不能 blit”：当 `absoluteRemoved` 或 `prevFrameContaminated` 为真时，`prevScreen` 置空，强制走全量绘制，避免叠加层移除或选择反色覆盖造成的脏像素回拷。alt-screen 模式下，渲染器还会把 `height` 强制钳到 `terminalRows`，并把 `viewport.height` 伪装成 `terminalRows + 1`，配合 `cursor.y` 的钳制，绕开 log-update 对“刚好填满屏幕”触发清屏或换行的路径，从而减少 flicker 与光标模型漂移。

## 设计权衡

这个 TUI 运行时最核心的取舍是：把“终端协议细节”下沉到 `App.tsx` 与 renderer，换取上层组件可以像写 GUI 一样组合。代价是入口复杂度上升，任何一个终端模式开关都要考虑 tmux、xterm.js、Bun 流实现差异，以及 suspend 或 resume 后的自愈。

第二个取舍是性能与一致性。允许使用 `prevScreen` 做 blit 能让稳定帧几乎是 O(变化量)，但一旦存在选择反色、绝对定位覆盖等跨子树绘制，就必须牺牲增量路径，宁可重绘也不回滚错误像素。对用户而言，这相当于用“偶发更贵的一帧”换“持续正确的屏幕”。

最后是输入事件的语义边界：`processKeysInBatch` 明确把 terminal response 排除在“用户交互时间”之外，把 hover 的无按键 motion 也排除在外。这让自动化回包、被动鼠标漂移不会压制 idle 逻辑，但也要求所有“算作交互”的路径必须经过同一个集中分发点，否则就会出现时钟不同步的幽灵问题。

## 小结

Claude Code 的 TUI 不是“写字符串”，而是“维护一个可恢复、可增量的屏幕模型”。`App.tsx` 把终端当作事件源与模式机来管理，`renderer.ts` 把 React 树投影成网格并在 alt-screen 下维持严格不变量。理解这两处，就能把后续的选择、超链接、后台任务面板都看成同一套运行时约束下的自然延伸。

## 导航

- [上一章：执行引擎，tool_use 如何真正落地为执行、并发与回写](./06-执行引擎-tool_use-如何落地为执行与回写.md)
- [返回目录](../Claude-Code源码精讲.md)
- [下一章：统一交互状态机，从 createStore 到 AppStateStore](./08-统一交互状态机-从-createStore-到-AppStateStore.md)
