# 03 · 点击发送后，请求由谁处理

[阅读目录](README.zh.md) · [上一章](02-typescript.zh.md) · [下一章](04-composer.zh.md)

本轮从页面操作追踪到 Agent 执行循环。假设你已经进入一个会话，Agent 当前空闲，在输入框输入“解释这个项目”，然后点击发送按钮。只跟踪普通文本，不包含附件、斜杠命令、子 Agent 或运行中插话。前端组件使用 `.tsx`，其余逻辑主要使用 `.ts`；`.tsx` 可以包含描述界面的 JSX。

行号对应当前工作区版本，代码更新后可能移动；定位时同时搜索下面列出的方法名。路径均相对于仓库根目录 `deepseek-harness/`，可在 VS Code 中用 `Ctrl+P` 粘贴 `路径:行号` 打开。

点击发送后，首先关注业务请求：前端调用 `remote.session.prompt(...)`，逻辑端点为 `session/prompt`；服务端由 `SessionController.prompt()` 接收，再交给 `SessionCommandController.prompt()` 处理，最后调用 `agent.followup()` 启动任务。下面先沿这条业务路线阅读，再补充按钮如何连接到请求。

| 操作与职责 | 源码位置 | 方法 |
|---|---|---|
| 前端准备文本消息 | `packages/client/ui-conversation/src/client/service.ts:234` | `sendSession()` |
| 前端发出远程请求 | `packages/api/session-controller/src/client/sessions/session.ts:270` | `prompt()` 调用 `remote.session.prompt(...)` |
| 服务端接收 `session/prompt` | `packages/api/session-controller/src/index.ts:468` | `SessionController.prompt()` |
| 服务端处理消息接纳 | `packages/api/session-controller/src/commands.ts:321` | `SessionCommandController.prompt()` |
| Agent 接收任务 | `packages/core/agent-loop/src/agent.ts:180` | `ReactLoopAgent.followup()` |

`InputBar` 是会话输入栏组件，包含编辑器、附件交互和发送/停止等控件，并不是通用按钮组件。它负责页面交互；理解发送消息的业务流程，可以先从下面的远程请求读起。

## 1. 前端会话对象 → session.ts 的 prompt() → session/prompt 请求

源码位置：`packages/api/session-controller/src/client/sessions/session.ts:270` — 前端会话对象的 `prompt()`。

打开 [packages/api/session-controller/src/client/sessions/session.ts](../../packages/api/session-controller/src/client/sessions/session.ts)，搜索 `async prompt`。普通会话会调用：

```text
this.remote.session.prompt({
  requestId,
  sessionId,
  mode,
  content,
  clientTimeZone,
}, signal)
```

这段是字段关系示意；源码中还包含请求 ID 默认值等表达式。`sessionId` 指定把消息交给哪个会话，`requestId` 标识这一次提交，`content` 是前端构造的内容块，例如 `[{ type: 'text', text: '解释这个项目' }]`，`mode` 是排队或插话方式。

到这里才进入远程调用。`remote.session.prompt` 对应逻辑端点 `session/prompt`，不是一个名为 `prompt.ts` 的文件，也不能直接当作 HTTP URL。这一轮先跨过通用传输与分派实现，直接追踪它对应的业务接收方法。

## 2. 服务端接收 → index.ts 的 SessionController.prompt()

源码位置：`packages/api/session-controller/src/index.ts:468` — 服务端 `SessionController.prompt()`。

打开 [packages/api/session-controller/src/index.ts](../../packages/api/session-controller/src/index.ts)，搜索 `@Remote('prompt')`。它下面的 `SessionController.prompt(request, signal)` 就是这个远程调用的业务入口。

它检查取消信号、消息内容和时区，并构造消息来源 `source`，然后调用：

```text
return this.commands.prompt(request, source)
```

这里 `@Remote('prompt')` 是装饰器，用于声明远程方法；`request` 是前端传来的参数对象。外层远程访问控制也参与调用，不能把“页面有发送按钮”当作服务端已经授权。

## 3. 服务端处理消息 → commands.ts 的 prompt() → agent.followup()

源码位置：`packages/api/session-controller/src/commands.ts:321` — `SessionCommandController.prompt()`。

打开 [packages/api/session-controller/src/commands.ts](../../packages/api/session-controller/src/commands.ts)，搜索 `async prompt`。`SessionCommandController.prompt()` 首先通过 `resolveAgent(request.sessionId)` 找到或恢复对应 Agent，再检查请求是否已经处理，完成内容接纳等检查。

普通消息经过 `createUserMessage({ content, source })` 构造后，进入以下分支：

```text
if (request.mode === 'steer') agent.steer(message)
else agent.followup(message)
```

本轮普通排队场景走 `agent.followup(message)`。方法成功时返回 `{ accepted: true }`，含义是消息已被接纳，不是模型已经回答完毕。若提交被拒绝，前端会显示相应错误并保留草稿供重试。

## 4. Agent 接收 → agent.ts 的 followup() → send() → wakeDriver()

源码位置：`packages/core/agent-loop/src/agent.ts:180` — `followup()`；同文件第 171 行 — `send()`；第 235 行 — `wakeDriver()`。

打开 [packages/core/agent-loop/src/agent.ts](../../packages/core/agent-loop/src/agent.ts)，搜索 `followup(input`。`ReactLoopAgent.followup()` 很短：

```text
followup(input: UserMessage): void {
  this.send(input, 'next-turn', true)
}
```

`input` 是上一层构造的用户消息，`'next-turn'` 表示进入下一轮消息队列，`true` 表示需要唤醒执行。继续看同文件的 `send()`：它将消息放进 inbox，再调用 `wakeDriver()`。

在当前空闲场景中，`wakeDriver()` 准备活动状态和完成通知，并启动 `kick()`。所以“用户点击发送”最终转换成了“Agent 队列里多出一条用户消息，并开始执行”。

## 5. 开始执行 → kick() → turn()

源码位置：`packages/core/agent-loop/src/agent.ts:274` — `kick()`；同文件第 333 行 — `turn()`。

仍在 [packages/core/agent-loop/src/agent.ts](../../packages/core/agent-loop/src/agent.ts)，搜索 `private async kick`，可以看到核心循环：

```text
while (await this.turn()) {}
```

`kick()` 调用 `turn()` 执行一轮，根据返回值决定是否继续。到这里，我们已经从页面按钮追踪到了 Agent 执行入口。模型请求与工具调用位于后续执行流程中，下一轮从 `turn()` 往下读。

页面不会通过等待 `session/prompt` 返回来取得完整回答。提交确认和会话后续更新是两件事；消息被接纳之后，页面还需要通过会话观察机制获得运行状态与输出。本轮先停在执行入口，不把提交成功当作任务完成。

## 本章自测

点击发送是否立即调用模型？`accepted: true` 是否表示任务完成？请沿本章路径找到对应方法，并分别解释请求接纳和任务执行的区别。


[阅读目录](README.zh.md) · [上一章](02-typescript.zh.md) · [下一章](04-composer.zh.md)
