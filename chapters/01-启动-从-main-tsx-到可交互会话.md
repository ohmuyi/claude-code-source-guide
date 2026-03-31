# 第1章 启动：Claude Code 如何从 main.tsx 走到可交互会话

## 本章要回答的问题

- 为什么启动入口不直接在 `main.tsx` 做所有事，而要先经过 `entrypoints/cli.tsx`？
- 交互式会话的“可交互”到底指什么，代码里以什么条件分流？
- 从解析参数到真正进入 REPL，关键的边界在哪里？

## 关键源码入口

- [`src/entrypoints/cli.tsx`](https://github.com/ohmuyi/claude-codes/blob/main/src/entrypoints/cli.tsx)
- [`src/main.tsx`](https://github.com/ohmuyi/claude-codes/blob/main/src/main.tsx)
- 关键调用点：`entrypoints/cli.tsx` 中的 `main()` 通过动态 `import('../main.js')` 执行 `cliMain()`

## 运行机制

启动首先被拆成两层：`entrypoints/cli.tsx` 负责“尽可能快地决定要不要继续加载”，`main.tsx` 负责“把一次 CLI 调用升级成一个会话”。前者的目标是把常见 fast-path，例如 `--version` 或少数内部子路径，用最少的 import 解决，避免昂贵的模块求值。

当确实需要完整 CLI 时，`cli.tsx` 会提前做几件只靠环境变量就能生效的事情，例如 `--bare` 立刻设置 `process.env.CLAUDE_CODE_SIMPLE = '1'`，保证后续模块在 import 时读到的是“简化模式”的世界。它还会 `startCapturingEarlyInput()`，把用户在加载期敲下的输入先缓存起来，让“可交互”不仅是渲染出来，更是不会丢键。

`main.tsx` 的第一层是启动期副作用并行化。它在文件开头就触发 `startMdmRawRead()`、`startKeychainPrefetch()`，并用 `profileCheckpoint()` 把重 import 前后的时间切开，等价于把“读配置、读钥匙串”这类 I/O 提前发射出去，和后续大量 import 并行跑。

真正的模式分流发生在 `main()` 早期。代码用 `-p/--print`、`--init-only`、`--sdk-url`、以及 `process.stdout.isTTY` 共同决定 `isNonInteractive`，再通过 `setIsInteractive(!isNonInteractive)` 把“这是会话 UI 还是一次性输出”写入全局状态。关键点在于这一步发生在 `init()` 之前，因为后续遥测、鉴权等逻辑会依赖这个模式开关避免弹 UI 或触发不安全的行为。

接下来进入 `run()`，`CommanderCommand` 只负责解析与分发，但它的 `preAction` hook 承担了“启动一次会话所需的前置条件”。这里先 `ensureMdmSettingsLoaded()` 和 `ensureKeychainPrefetchCompleted()`，再 `await init()`，再挂 logging sinks 与迁移逻辑。到这一层，“启动”已经从参数解析升级为一个具备配置、权限、插件与策略加载能力的运行时。

当走交互路径时，`main.tsx` 明确延后创建 Ink root：`createRoot()` 只在 `!isNonInteractiveSession` 下执行，因为 Ink 的 `patchConsole` 会吞掉 headless 的输出。随后 `showSetupScreens(root, ...)` 负责信任对话框、onboarding 等阻塞 UI，等这些门槛通过后，最终用 `launchRepl(root, {...}, sessionConfig, renderAndRun)` 把状态与配置交给 REPL，从而进入可交互会话。

## 设计权衡

这套结构用“入口分层 + 动态 import”换来了冷启动性能与 fast-path 的可维护性，但代价是启动链路被切得很细，定位问题时必须同时理解 `cli.tsx` 对 `process.argv` 的改写、`main.tsx` 的模式判定和 `preAction` 的初始化时序。

把大量副作用提前到模块顶层，例如 keychain 和 MDM 的并行预取，可以显著缩短首帧等待，但也提高了“导入即执行”的风险。所以代码用注释和 lint rule 明确这些副作用的必要性，并把更敏感的系统上下文预取放到“信任已建立”之后。

交互与非交互共用同一条 `main.tsx` 主干，减少了重复实现，但也迫使许多逻辑在运行时反复检查 `isNonInteractive`。它不是最优雅的分层，却是工程上让两条路径共享同一套配置、策略、工具池的直接方式。

## 小结

`entrypoints/cli.tsx` 负责快速分流与最小化加载，`main.tsx` 负责把一次 CLI 调用构造成“有状态的会话”。交互会话的关键落点是 `createRoot()` 与 `launchRepl()`，而它们能否执行取决于 `main()` 早期对 `isNonInteractive` 的判定与 `preAction` 中 `init()` 的完成。

## 导航

- [上一节：前言](./00-前言.md)
- [返回目录](../Claude-Code源码精讲.md)
- [下一章：回合，一次用户输入如何变成一个可递归的 turn](./02-回合-一次用户输入如何变成可递归的-turn.md)
