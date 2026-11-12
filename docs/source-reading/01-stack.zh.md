# 01 · 项目的前端与后端

[阅读目录](README.zh.md) · [下一章](02-typescript.zh.md)

先以普通 Web 应用理解这个项目：浏览器负责界面，Node.js 进程负责会话、模型请求和工具执行。仓库把这两侧称为 Client 和 Host；Host 可以运行在你自己的电脑上，不一定是远程服务器。

| 部分 | 使用的技术 | 在本项目中的职责 | 源码位置 |
|---|---|---|---|
| 前端界面 | React + TypeScript | 用组件构建会话页面、输入栏和消息列表；组件常写在 `.tsx` 中 | [packages/client/ui-conversation/src/client/skeleton/InputBar.tsx](../../packages/client/ui-conversation/src/client/skeleton/InputBar.tsx) |
| 前端构建 | Vite | 将前端源码构建为浏览器加载的资源 | [apps/web/package.json](../../apps/web/package.json) |
| 输入编辑器 | Lexical | 管理输入框中的文本编辑行为 | [packages/client/ui-conversation/package.json](../../packages/client/ui-conversation/package.json) |
| 后端运行环境 | Node.js + TypeScript | 运行服务、调用模型、执行工具与管理会话 | [架构说明](../architecture.zh.md) |
| 应用组织框架 | Cordis | 用插件注册服务、事件和清理操作，并管理依赖与生命周期 | [Cordis 入门](../cordis-primer.zh.md) |
| Web 服务 | 项目的 WebServer 服务，底层使用 `node:http` / `node:https` | 提供 HTTP(S) 服务与路由注册 | [packages/host/webserver/src/index.ts](../../packages/host/webserver/src/index.ts) |
| 前后端业务调用 | 项目的 Typert Remote / Gateway 机制 | 将前端方法调用传给 Host 的业务服务，返回结果 | [packages/api/README.zh.md](../../packages/api/README.zh.md) |

## 前端：React 负责显示，业务对象负责提交

React 组件描述“页面显示什么、用户点击后做什么”。例如 `InputBar.tsx` 是完整的会话输入栏，不是通用按钮。它读取会话和输入状态，呈现编辑器与按钮，再把提交动作交给输入控制逻辑。Vite 是构建工具，不负责接收后端业务请求。

前端代码主要在 `packages/client/`，Web 应用入口与构建配置在 `apps/web/`。但目录名不能完全决定运行位置：例如 `packages/api/session-controller/src/client/sessions/session.ts` 位于 API 包里，却是前端会话对象，因为同一个功能包可以同时提供 Client 和 Host 实现。

## 后端：Cordis 组织服务，Controller 接收业务请求

后端应用的组织基础是 Cordis。插件把能力挂到共享上下文 `ctx` 上，例如通过 `ctx.agents` 访问 Agent 管理能力，通过 `ctx.sessions` 访问会话服务。模型适配、工具和执行循环也以插件方式组成应用。HTTP 服务底层直接使用 Node.js 的服务器 API；理解这里的业务代码，应优先理解 Cordis 的服务和依赖注入。

以发送消息为例，`SessionController` 继承 `TypertRemoteService`，用 `@Remote('prompt')` 声明远程方法。前端调用 `remote.session.prompt(...)` 后，服务端的 `SessionController.prompt()` 接收请求，再调用 `SessionCommandController.prompt()` 处理消息接纳，最后交给 Agent 执行。`Controller` 和 `Service` 是代码承担的职责，不代表每个类都单独启动一个服务器。

## 把技术栈连接到一次页面操作

```text
浏览器中的 React 会话页面
  → 用户输入文字，点击发送
  → 前端会话对象调用 remote.session.prompt(...)
  → Remote / Gateway 将调用送到 Host
Node.js 中的 Cordis 应用
  → SessionController.prompt() 接收
  → SessionCommandController.prompt() 处理
  → ReactLoopAgent.followup() 将消息入队并唤醒执行
  → 后续模型请求和工具执行
```

`ReactLoopAgent` 名字中的 `React` 不表示它是 React 界面组件；它位于 `packages/core/agent-loop/`，承担 Agent 执行逻辑。桌面版使用 Electron 承载应用，相关结构见 [桌面应用架构](../architecture.zh.md)。本笔记接下来只沿 Web 页面的发送流程阅读。

[阅读目录](README.zh.md) · [下一章](02-typescript.zh.md)
