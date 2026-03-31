# 《Claude Code源码精讲》

这个文件是目录版，适合按章节跳读。

如果你想快速看全书结构，请读 [SUMMARY](./SUMMARY.md)。

## 目录

### 开始阅读

0. [前言](./chapters/00-前言.md)

### 第一篇 启动与执行世界观

1. [启动：Claude Code 如何从 main.tsx 走到可交互会话](./chapters/01-启动-从-main-tsx-到可交互会话.md)
2. [回合：一次用户输入如何变成一个可递归的 turn](./chapters/02-回合-一次用户输入如何变成可递归的-turn.md)
3. [工具：Tool 接口为何是整个系统的协议中心](./chapters/03-工具-Tool-接口为何是协议中心.md)

### 第二篇 任务、权限与执行内核

4. [任务：为什么 Claude Code 还需要 Task 这一层](./chapters/04-任务-为什么还需要-Task-层.md)
5. [权限：权限不是一个开关，而是一条决策链](./chapters/05-权限-权限不是开关而是决策链.md)
6. [执行引擎：tool_use 如何真正落地为执行、并发与回写](./chapters/06-执行引擎-tool_use-如何落地为执行与回写.md)

### 第三篇 TUI、状态与后台任务

7. [终端不是字符串输出：Claude Code 的 TUI 运行时](./chapters/07-终端不是字符串输出-TUI-运行时.md)
8. [统一交互状态机：从 createStore 到 AppStateStore](./chapters/08-统一交互状态机-从-createStore-到-AppStateStore.md)
9. [后台任务总线：Shell、Agent 与 Main Session 的三种后台化](./chapters/09-后台任务总线-Shell-Agent-Main-Session.md)
10. [Agent 视图如何长出来：前后台切换、主会话旁路与面板投影](./chapters/10-Agent-视图-前后台切换与面板投影.md)
11. [可观察性内建：Progress、通知、Transcript 与性能指标](./chapters/11-可观察性内建-Progress-通知-Transcript-性能.md)
12. [交互体验的最后一公里：低打扰反馈、双击拖选与卡顿自诊断](./chapters/12-交互体验最后一公里-低打扰反馈与自诊断.md)

### 第四篇 协议接入、生态与长期运行

13. [MCP 客户端：把工具、命令与资源统一成协议入口](./chapters/13-MCP-客户端-协议入口.md)
14. [远程会话：把本地 CLI 变成可接管的运行中环境](./chapters/14-远程会话-本地-CLI-到可接管环境.md)
15. [记忆系统：把当前对话沉淀为未来上下文](./chapters/15-记忆系统-沉淀未来上下文.md)
16. [技能系统：把 prompt 约定升级为可加载能力](./chapters/16-技能系统-prompt-到可加载能力.md)
17. [插件系统：从本地能力到可分发生态](./chapters/17-插件系统-本地能力到可分发生态.md)
18. [压缩与演进：在有限上下文里维持长期运行](./chapters/18-压缩与演进-有限上下文里长期运行.md)

## 说明

本书按 team agent 方式完成：先用多个 agent 并行 brainstorm 大纲，再按章节组并行起草正文，最后统一收口术语、结构与阅读顺序。正文采用“问题、源码入口、运行机制、设计权衡、小结”的固定结构，便于把源码细节还原成稳定的系统心智模型。
