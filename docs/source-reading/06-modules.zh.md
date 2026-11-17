# 06 · 发送与执行流程中的模块和接口

[阅读目录](README.zh.md) · [上一章](05-practice.zh.md)

这里的“接口”指模块向调用者提供的方法、输入和输出，不只指 TypeScript 的 `interface` 关键字。本章是阅读流程时的对照表，不是完整 API 文档。先看谁调用谁，再打开链接检查类型声明。行号对应当前源码，代码更新后请按方法名定位。

## 1. 从页面提交到 Agent：谁接收谁的调用

| 参与模块 | 职责与调用关系 | 关键入口 | 输入与输出 | 源码位置 |
|---|---|---|---|---|
| 会话界面 `client/ui-conversation` | 准备文本和附件内容，调用前端会话对象 | `sendSession()` | 草稿、附件标识、提交模式等 → 异步提交结果 | [service.ts](../../packages/client/ui-conversation/src/client/service.ts)，第 234 行 |
| 前端会话对象 `api/session-controller` 的 Client 部分 | 将提交转为远程调用 | `prompt()` → `remote.session.prompt()` | 内容块、模式等 → `Promise<RemoteResult<{ accepted: true }>>` | [session.ts](../../packages/api/session-controller/src/client/sessions/session.ts)，第 270 行 |
| Remote / Gateway | 在 Client 与 Host 之间传递调用和结果 | `session/prompt` 逻辑端点 | 远程方法名与参数 → 业务返回值或错误 | [API 模块说明](../../packages/api/README.zh.md) |
| `SessionController` | 接收远程请求，检查内容和时区，构造消息来源 | `prompt(request, signal)` | `SessionPromptRequest`、取消信号 → `Promise<SessionPromptValue>` | [index.ts](../../packages/api/session-controller/src/index.ts)，第 468 行 |
| `SessionCommandController` | 查找或恢复 Agent、检查重复请求、接纳内容 | `prompt(request, source)` | 请求、`MessageSource` → `Promise<SessionPromptValue>` | [commands.ts](../../packages/api/session-controller/src/commands.ts)，第 321 行 |
| Agent 操作接口 | 接收下一轮输入并唤醒执行 | `followup(input: UserMessage): void` | 用户消息 → 无返回值；消息入队并触发执行 | [实现 agent.ts](../../packages/core/agent-loop/src/agent.ts)，第 180 行；[接口扩展](../../packages/core/agent/src/runtime-types.ts) |

`RemoteResult` 是前端收到的成功或失败包装。服务端成功值中的 `accepted: true` 只确认消息进入目标 Agent 的处理流程，不包含模型最终回答。`followup()` 返回 `void` 也不代表任务已经结束。

请求字段的声明位于 [packages/api/session-controller/src/types.ts](../../packages/api/session-controller/src/types.ts) 第 324 行。读这几个字段即可理解普通发送请求：

| 字段 | 类型 | 含义 |
|---|---|---|
| `requestId` | `SessionRequestId` | 区分每一次提交，用于重复请求识别 |
| `sessionId` | `SessionId` | 指定目标会话 |
| `mode` | `'queue'` 或 `'steer'` | 下一轮排队或运行中插话 |
| `content` | `readonly PromptContentPart[]` | 文本、附件等内容块 |
| `clientTimeZone` | 可选 `string` | 客户端时区 |

## 2. 进入执行循环后：哪些模块继续参与

下面从 `agent.followup()` 后继续。先掌握接口用途；模型提供商的 HTTP 实现和具体工具的文件操作属于下一层，不在本表展开。

| 参与模块 | 谁调用它、做什么 | 关键接口 | 返回什么、交给谁 | 源码位置 |
|---|---|---|---|---|
| `core/agent-loop` | 驱动器启动轮次，轮次执行步骤 | 内部 `turn()`、`step()` | 轮次是否继续、步骤结束原因；由循环自己判断 | [agent.ts](../../packages/core/agent-loop/src/agent.ts)，第 333、441 行 |
| `core/tools` | `preStep()` 获取本步骤可用工具 | `resolve(context: ToolActivationContext)` | `Promise<ResolvedToolSet>`，交给提示词组装和本步骤工具执行 | [index.ts](../../packages/core/tools/src/index.ts)，第 264 行 |
| `core/system-prompt` | `preStep()` 组装模型所需提示内容与工具声明 | `assemble(context: AssembleContext)` | `Promise<PromptAssembly>`，交给步骤准备与请求构建 | [index.ts](../../packages/core/system-prompt/src/index.ts)，第 621 行 |
| `llm/llm` | 循环准备模型调用配置与适配器调用入口 | `prepareCall(config, signal?, configure?)` | `Promise<PreparedLlmCall>`，包含已准备配置与流式调用入口 | [index.ts](../../packages/llm/llm/src/index.ts)，第 975 行 |
| `llm/llm` 与所选模型适配器 | `step()` 发起模型生成并消费返回流 | `stream(options: GenerateOptions)` | `AsyncIterable<StreamChunk>`，循环通过 `for await` 逐块读取 | [index.ts](../../packages/llm/llm/src/index.ts)，第 1185 行；[调用位置](../../packages/core/agent-loop/src/agent.ts)，第 496 行 |
| `core/tools` | `step()` 将模型返回的工具调用交给执行管线 | `execute(input: ToolBatchInput)` | `Promise<ToolBatchResult>`，循环读取附加上下文和结束标记 | [index.ts](../../packages/core/tools/src/index.ts)，第 375 行 |
| `core/session` | 循环及相关服务记录输入、回答、工具结果与执行阶段 | `session.append(...)` 等事件与请求记录接口 | 会话日志供后续模型请求构建和观察使用 | [Session 模块说明](../../packages/core/session/README.zh.md)；[循环记录位置](../../packages/core/agent-loop/src/agent.ts) |

`turn()`、`step()` 是 `ReactLoopAgent` 的私有实现方法，不是前端可调用的远程接口。`resolve()`、`assemble()`、`prepareCall()`、`execute()` 则是循环调用其他服务的入口。它们虽然都叫“方法”，但调用者和可见范围不同。

## 3. 把接口连接成一次工具调用

假设用户要求读取项目中的一个文件：页面提交后，Agent 先获取可用工具、组装提示内容并准备模型请求。模型返回的是“调用哪个工具、传什么参数”的结构化内容；实际读取文件由工具实现完成。

源码位置：[packages/core/agent-loop/src/agent.ts](../../packages/core/agent-loop/src/agent.ts)，第 590 行附近。`step()` 从助手消息中筛选 `tool-call` 内容块，第 596 行调用 `tools.execute()`，传入调用列表、Agent、取消信号、本步骤工具集合、日志位置和并行上限。

工具执行结束后，循环把 `result.additionalContexts` 加入下一步队列，并检查 `result.concluded`。未结束时 `step()` 返回 `null`，轮次循环可以进入下一步模型请求。工具结果通过会话记录参与后续请求；`additionalContexts` 是额外上下文，不应把它当作全部工具结果。

没有工具调用时，普通完成路径返回 `{ kind: 'completed' }`。此外还有输出长度上限、取消和失败等结束路径，因此不能只看是否收到文本来判断成功。

## 4. 阅读每个接口时记录什么

对照源码记录五件事：调用者、参数类型、返回类型、产生的记录或其他副作用、失败如何传回。遇到 `Promise<T>`，先问等待成功后拿到什么；遇到 `AsyncIterable<T>`，先问每次迭代得到什么；遇到 `void`，继续查执行结果在哪里观察。

用本章检查自己：`SessionPromptValue` 与模型回答是否相同？`turn()` 能否由网页直接远程调用？模型返回工具调用后，哪个模块负责真正执行？回答时同时指出源码文件和方法。

[阅读目录](README.zh.md) · [上一章](05-practice.zh.md)
