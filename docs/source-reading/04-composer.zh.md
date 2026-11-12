# 04 · 按钮怎样连接到发送请求

[阅读目录](README.zh.md) · [上一章](03-send-message.zh.md) · [下一章](05-practice.zh.md)

本章承接发送消息的业务流程，补充按钮与前端请求之间的连接。仍以已有会话、Agent 空闲、发送普通非空文本为例；先理解前一章的服务端处理，再阅读输入状态机。路径相对于仓库根目录，行号可能随源码更新移动，可同时搜索方法名。

## 1. 页面点击发送 → InputBar.tsx 的 onPrimary()

源码位置：`packages/client/ui-conversation/src/client/skeleton/InputBar.tsx:395` — `onPrimary()`。

打开 [packages/client/ui-conversation/src/client/skeleton/InputBar.tsx](../../packages/client/ui-conversation/src/client/skeleton/InputBar.tsx)，搜索 `onPrimary`。这个函数处理输入框主按钮的点击。当前场景下，按钮表示发送，输入非空且没有提交中的任务，于是执行：

```text
keyboard.submit(primarySubmitMode, 'click')
```

`primarySubmitMode` 决定消息进入哪个队列，`'click'` 标记操作来自鼠标点击。当前空闲场景使用普通排队提交。此时还没有向模型发请求：页面只是把提交意图交给输入控制器。Agent 运行时，同一个主按钮可能表示停止或另一种发送方式，因此阅读时必须先确定页面状态。

## 2. 按钮回调 → facade.ts 的 submit()

源码位置：`packages/client/ui-conversation/src/client/input/facade.ts:402` — `submit()`。

打开 [packages/client/ui-conversation/src/client/input/facade.ts](../../packages/client/ui-conversation/src/client/input/facade.ts)，搜索 `submit(mode`。`submit()` 收集当前草稿、引用、附件标识等信息，再调用 `dispatchRun()`，提交一个 `type: 'enter'` 的事件。

这里的 `enter` 是输入状态机的事件名，不代表用户一定按了 Enter 键；点击按钮也走这条路径。`draft` 保存这次提交的文本，`mode` 保存提交方式。

## 3. 输入事件 → machine.ts 的 onEnter() → send()

源码位置：`packages/client/ui-conversation/src/client/input/machine.ts:138` — `onEnter()`；同文件第 132 行 — `send()`。随后返回 `packages/client/ui-conversation/src/client/input/facade.ts:693` 处理 `default-sink`，第 727 行是 `sinkSerialized()`。

打开 [packages/client/ui-conversation/src/client/input/machine.ts](../../packages/client/ui-conversation/src/client/input/machine.ts)，搜索 `onEnter`。它先检查当前是否允许提交。对于不以 `/` 开头的普通文本，它调用 `beginAttempt()` 保存本次提交的信息，再调用 `send(attempt)`。

`send()` 把输入状态改为 `sending`，并返回一个 `type: 'default-sink'` 的效果描述。这里的“效果描述”只是一个对象，告诉外层接下来做什么；状态机本身还没有发网络请求。

返回 [packages/client/ui-conversation/src/client/input/facade.ts](../../packages/client/ui-conversation/src/client/input/facade.ts)，搜索 `case 'default-sink'`。外层收到这个描述后调用 `sinkSerialized()`，完成草稿序列化，再调用 `this.deps.defaultSink(...)`。

## 4. 默认提交处理 → hub.ts 的 sink() → service.ts 的 sendSession()

源码位置：`packages/client/ui-conversation/src/client/input/hub.ts:269` — `sink()`；接着进入 `packages/client/ui-conversation/src/client/service.ts:234` — `sendSession()`。

打开 [packages/client/ui-conversation/src/client/input/hub.ts](../../packages/client/ui-conversation/src/client/input/hub.ts)，搜索 `defaultSink:`，可以看到它被连接到 `this.sink(...)`。再搜索 `private sink`，普通非空文本会进入：

```text
this.conversation().sendSession(session, text, attachmentIds, mode, signal, accept)
```

随后打开 [packages/client/ui-conversation/src/client/service.ts](../../packages/client/ui-conversation/src/client/service.ts)，搜索 `async sendSession`。它准备本次消息提交记录，并把文本变成消息内容块。没有附件时，内容相当于：

```text
[{ type: 'text', text: '解释这个项目' }]
```

然后它调用 `session.prompt(content, mode, signal, submission.requestId)`。这里的 `session` 是前端会话对象；不要把它与服务端保存事件的 Session 或驱动模型的 Agent 混为一谈。

## 本轮调用路线与自测

按下面的顺序打开文件，每一步只回答“谁调用了谁、传了什么”：

```text
页面输入文字，点击发送
  → InputBar.tsx: onPrimary()
  → input/facade.ts: submit() → dispatchRun()
  → input/machine.ts: onEnter() → send()
  → input/facade.ts: default-sink → sinkSerialized()
  → input/hub.ts: sink()
  → client/service.ts: sendSession()
  → client/sessions/session.ts: prompt()
  → remote.session.prompt(...)：远程调用
  → session-controller/src/index.ts: SessionController.prompt()
  → session-controller/src/commands.ts: SessionCommandController.prompt()
  → agent-loop/src/agent.ts: followup() → send()
  → wakeDriver() → kick() → turn()
```

尝试回答三个问题：点击按钮时是否立即调用模型？没有，前面还要完成输入处理、远程调用和消息接纳。`accepted: true` 是否表示回答完成？不是，它只确认接纳。网页发送是否经过 headless runner 的 `run()`？不经过；网页和命令行入口不同，最终都可以驱动 Agent。

[阅读目录](README.zh.md) · [上一章](03-send-message.zh.md) · [下一章](05-practice.zh.md)
