# 第16章 技能系统：把 prompt 约定升级为可加载能力

## 本章要回答的问题

技能系统要解决的核心不是“写一段更长的 prompt”，而是把一组可复用的行为约定变成可发现、可组合、可治理的能力单元。它如何把 `SKILL.md` 解析成可调用的 `Command`？在 managed、user、project 多来源并存时，如何决定加载顺序与去重？面对路径条件、动态发现、以及内联 shell 执行这样的“强能力”，系统如何在易用性与安全性之间划边界？

## 关键源码入口

- [`src/skills/loadSkillsDir.ts`](https://github.com/ohmuyi/claude-codes/blob/main/src/skills/loadSkillsDir.ts)：`getSkillDirCommands`、`loadSkillsFromSkillsDir`、`createSkillCommand`
- [`src/skills/loadSkillsDir.ts`](https://github.com/ohmuyi/claude-codes/blob/main/src/skills/loadSkillsDir.ts)：`parseSkillFrontmatterFields`、`estimateSkillFrontmatterTokens`
- [`src/skills/loadSkillsDir.ts`](https://github.com/ohmuyi/claude-codes/blob/main/src/skills/loadSkillsDir.ts)：`discoverSkillDirsForPaths`、`addSkillDirectories`、`activateConditionalSkillsForPaths`

## 运行机制

技能的“加载”被拆成两段：先注册元数据，再在调用时生成最终 prompt。`loadSkillsFromSkillsDir()` 只接受目录形态 `skill-name/SKILL.md`，读取内容后用 `parseFrontmatter()` 拆出 frontmatter 与正文，再由 `parseSkillFrontmatterFields()` 抽取统一字段，如 `allowed-tools`、`arguments`、`model`、`disable-model-invocation`、`hooks`、`context`、`agent`、`effort`、`shell` 等，最后交给 `createSkillCommand()` 构造 `Command`。

`createSkillCommand()` 把“可执行语义”集中在 `getPromptForCommand()`：先做参数替换 `substituteArguments()`，再注入 `${CLAUDE_SKILL_DIR}` 与 `${CLAUDE_SESSION_ID}`，最后在非 `loadedFrom === 'mcp'` 的情况下允许 `executeShellCommandsInPrompt()` 执行内联 shell。这里的安全边界很明确：MCP skills 被视为远端不可信内容，直接禁止正文中的 shell 执行，避免把“文本能力”意外升级成“代码执行”。

顶层入口 `getSkillDirCommands()` 负责多来源汇聚与治理。它并行加载 managed、user、project、`--add-dir`、legacy `commands`，随后用 `realpath()` 实现的 `getFileIdentity()` 按“文件真实身份”去重，解决符号链接和目录重叠导致的重复加载。路径条件也被显式建模：若 skill frontmatter 含 `paths`，先暂存到 `conditionalSkills`，只有在 `activateConditionalSkillsForPaths()` 里用 `ignore()` 匹配到当前操作文件后才转入 `dynamicSkills`，从而把“按需生效”变成一种稳定机制，而不是靠模型自觉。

动态发现用于降低启动成本并提升就近覆盖：`discoverSkillDirsForPaths()` 沿文件路径向上走到 `cwd`，寻找嵌套的 `.claude/skills`，并通过 `isPathGitignored()` 跳过例如 `node_modules` 这类 gitignored 目录，最后按“路径更深者优先”排序；`addSkillDirectories()` 再按这个优先级把更近的目录覆盖更远的目录。

## 设计权衡

技能目录坚持 `skill-name/SKILL.md` 的约束，牺牲了自由度，换来稳定的命名与根目录语义，也让 `${CLAUDE_SKILL_DIR}` 成为可依赖的约定。多来源并行加载提高了启动速度，但引入了“同一文件多处可见”的一致性问题，于是用 `realpath()` 去重而不是 inode，规避了虚拟或网络文件系统的不可靠性。路径条件与动态发现减少了模型上下文污染，但增加了状态机复杂度，因此需要显式信号去协调缓存失效。

最敏感的取舍在 shell：允许本地 skill 正文内联 shell 能显著增强表达力，但必须给远端来源更硬的限制，否则技能系统会变成一条隐蔽的执行通道。另一个现实取舍是 `--bare` 与 policy：`isBareMode()` 跳过自动发现，只加载显式 `--add-dir`，但 `skillsLocked` 仍然生效，表明“体验模式”不能成为策略绕过的后门。

## 小结

`loadSkillsDir.ts` 把技能定义为“可解析的 Markdown 加可控的运行时拼装”。前半段用 frontmatter 形成稳定接口，后半段在调用时注入目录、会话与参数，并以 `loadedFrom` 划出安全边界。通过去重、条件激活与动态发现，技能不再是散落的 prompt，而是可加载、可治理、可演进的能力集合。

## 导航

- [上一章：记忆系统，把当前对话沉淀为未来上下文](./15-记忆系统-沉淀未来上下文.md)
- [返回目录](../Claude-Code源码精讲.md)
- [下一章：插件系统，从本地能力到可分发生态](./17-插件系统-本地能力到可分发生态.md)
