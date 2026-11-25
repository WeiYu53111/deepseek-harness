# Agent 源码阅读指南

这组教程面向 TypeScript 基础较少、希望看懂 Agent 项目源码的读者。先认识技术栈，再学习必要语法，然后以“页面点击发送 → 发出请求 → 服务端接收 → Agent 执行”为主线阅读。每个流程步骤给出仓库路径和方法名；行号是当前版本的定位提示，更新后以方法名为准。

## 阅读顺序

| 顺序 | 章节 | 读完后能回答 |
|---|---|---|
| 1 | [前后端技术栈](01-stack.zh.md) | React、Node.js、Cordis 和 Remote 各负责什么？ |
| 2 | [TypeScript 基础](02-typescript.zh.md) | 类型、回调、Promise、泛型和声明合并怎样读？ |
| 3 | [发送消息的业务流程](03-send-message.zh.md) | 点击发送后发出什么请求，由哪个 Controller 接收，如何进入 Agent？ |
| 4 | [前端交互细节](04-composer.zh.md) | 输入栏、状态机和会话对象如何连接到请求？ |
| 5 | [练习与后续方向](05-practice.zh.md) | 怎样验证自己读懂了，并继续探索模型调用与工具执行？ |
| 6 | [参与模块与接口](06-modules.zh.md) | 谁调用哪个模块，接口输入输出是什么，哪些方法只是内部实现？ |
| 7 | [Web 启动与命令行基础](07-web-startup.zh.md) | Corepack、Node、tsx/esm 是什么，Web 如何启动并加载配置？ |
| 8 | [第一个二次开发练习](08-first-extension.zh.md) | 怎样加载自己的 Cordis 插件，并在 Web 中验证模型调用工具？ |

## 如何配合编辑器阅读

打开仓库根目录，在 VS Code 中按 `Ctrl+P`，粘贴章节提供的 `路径:行号`。先找接收请求的方法，再沿它的调用往下读。每次只记录输入、调用的方法、返回值和失败条件，不需要一次展开所有辅助函数。

第三章优先说明业务请求和服务端处理，第四章才展开按钮回调。这里的主线是普通 Web 会话的非空文本提交；附件、运行中插话和子 Agent 需要在理解主线之后单独追踪。

## 后续学习地图

Agent 开发需要让模型在工具、状态和外部环境之间可靠地完成任务。按照下面的顺序学习，每一轮都应能解释一个具体的执行过程。

| 顺序 | 重点 | 需要回答的问题 | 阅读位置 |
|---|---|---|---|
| 1 | 执行循环 | 什么触发下一次请求？什么条件让执行停止？ | `packages/core/agent-loop` |
| 2 | 工具设计 | 参数如何校验？工具结果如何支持模型下一步决策？ | `packages/core/tools` |
| 3 | 上下文管理 | 每次请求包含什么？信息如何保留或压缩？ | `packages/core/session`、`packages/core/system-prompt`、`packages/compaction` |
| 4 | 状态与恢复 | 中断前执行了什么？恢复后如何避免重复副作用？ | `packages/core/session`、`packages/session` |
| 5 | 权限与隔离 | 哪些操作需要审批？工具实际能访问什么？ | `packages/guard`、`packages/sandbox`、`packages/credentials` |
| 6 | 评测与调试 | 用户目标是否完成？失败发生在哪里？ | `snapshots`、各包的 `tests`、`benchmarks` |

先理解单个 agent 的执行与工具调用，再学习恢复、上下文压缩和多 agent 协作。插件组合的整体说明见[架构文档](../architecture.zh.md)，Cordis 基础见[入门说明](../cordis-primer.zh.md)。

## 验证范围

这些笔记用于源码阅读，没有执行真实模型调用或完整页面操作。具体运行行为以当前源码及所属包文档为依据。
