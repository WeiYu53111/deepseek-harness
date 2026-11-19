# 02 · 阅读源码所需的 TypeScript

[阅读目录](README.zh.md) · [上一章](01-stack.zh.md) · [下一章](03-send-message.zh.md)

如果 TypeScript 还不熟，先按这一节的顺序学习。目标是看出每段代码的输入、输出和执行时机。以下短例子是语法示意，不是独立运行的程序。

## 1. 变量、对象和类型标注

`const task: string = 'hello'` 中，`task` 是变量名，`string` 是类型，`'hello'` 是实际值。类型通常在运行时被擦除，不会自动检查 JSON 等外部输入。`const` 不允许变量重新赋值，但对象内部字段仍可能修改；`let` 允许重新赋值。`let count = 0` 没写类型，编译器也能推断出 `number`。

`{ task }` 是 `{ task: task }` 的简写。`const { task } = config` 是对象解构，用于取出字段。`{ ...options, model: 'demo' }` 复制顶层字段，再设置 `model`；后面的同名字段覆盖前面的，嵌套对象仍可能共享引用。

## 2. 接口、类型别名和函数签名

```text
interface Options {
  task: string
  json?: boolean
}
function readTask(options: Options): string {
  return options.task
}
```

`interface` 描述对象需要提供的成员。`task` 必需，`json?` 可以省略，因此读取时可能得到 `undefined`。函数右括号后的 `: string` 是返回值类型。TypeScript 通常按对象成员判断兼容性，不要求对象由某个特定类创建。

`type Status = 'idle' | 'running'` 给类型起名，表示只能取这两个字符串之一。`readonly` 限制通过该类型进行赋值，`Readonly<T>` 把顶层属性标为只读；它们不会在运行时冻结对象，也不保证深层不可变。

在 [headless runner](../../packages/bundle/headless/src/index.ts) 中，`interface Config` 是编译期类型，同名的 `const Config = z.object(...)` 是运行时配置 schema。类型名与值名可以相同；声明接口本身不会校验输入。

## 3. 问号、默认值和比较

| 写法 | 含义 | 阅读时注意 |
|---|---|---|
| `task?: string` | 字段可以省略 | 属于类型声明 |
| `startup?.permissionMode` | 对象存在时读取字段 | 对象为 `null` 或 `undefined` 时得到 `undefined` |
| `a ?? b` | `a` 为 `null` 或 `undefined` 时取 `b` | 保留 `false`、`0`、空字符串 |
| `a || b` | `a` 为假值时取 `b` | `false`、`0`、空字符串也触发后备值 |
| `condition ? a : b` | 条件成立取 `a`，否则取 `b` | 整个表达式产生一个值 |
| `a === b` | 严格相等比较 | 对象比较的是身份，不是字段内容 |

例如 `activity !== this.activityDone` 比较两个 Promise 是否为同一个对象。不要把它理解为比较异步操作的结果。

## 4. 回调与闭包

`(value) => value.text` 是箭头函数，返回表达式结果；`(value) => { ... }` 使用函数体，需要显式 `return` 才返回结果。`cleanup` 是函数值，`cleanup()` 是立即调用，`() => cleanup()` 是创建一个稍后调用清理函数的函数。

把函数传给另一个函数，通常叫传入回调，传入时不代表已经执行。`ctx.on('agent/error', callback)` 注册事件监听器，事件发生时才调用回调；返回的函数用于取消监听。回调还能访问外层变量，这叫闭包。阅读时留意它访问的是执行时的变量，还是事先保存的副本。

## 5. Promise、async、await 与清理

`Promise<T>` 表示异步操作：成功时提供 `T`，失败时拒绝。`async` 函数返回 Promise。`await` 暂停当前异步函数等待结果，让其他工作有机会运行，不会阻塞整个进程。

```text
async function waitAndRead(): Promise<string> {
  await agent.whenIdle()
  return 'ready'
}
```

这段示意返回 `Promise<string>`，调用者用 `await waitAndRead()` 得到字符串。`Promise<void>` 表示异步完成后没有有用的返回值。连续两个 `await` 通常先等前一个，再执行后一个；`Promise.all([a(), b()])` 先调用两者，再等待全部成功，其中一个拒绝不会自动取消另一个。

`await` 遇到拒绝会抛错，由 `try/catch` 处理；`finally` 在正常返回或抛错时用于清理。runner 的 `void run(...).catch(...)` 启动操作并安排失败处理，然后丢弃返回值；`void` 本身既不等待，也不捕获错误。空闲不等于成功，还要检查结束原因。

## 6. 联合类型、缩窄与断言

下面用简化的结果类型演示分支判断：

```text
type Result =
  | { kind: 'completed'; text: string }
  | { kind: 'failed'; error: Error }

function describe(result: Result): string {
  switch (result.kind) {
    case 'completed': return result.text
    case 'failed': return result.error.message
  }
}
```

`A | B` 表示值可能属于两种类型之一。这里通过 `kind` 判断后，编译器知道对应字段存在，这叫类型缩窄。仓库的事件和结束原因经常采用这种写法。`never` 表示不应存在的值；`assertNever` 常用于检查封闭联合是否遗漏分支，可由插件扩展的联合则需要考虑未知分支。

`unknown` 表示尚不知道类型，检查之后才能使用，例如先用 `error instanceof Error` 判断再读取 `error.message`。`any` 放弃相关类型检查。`value as SomeType` 只是让编译器按指定类型理解，不会转换或验证运行时值。

### `invocation satisfies never` 是什么

源码位置：[apps/cli/src/bin.ts](../../apps/cli/src/bin.ts)，`runCli()` 的 `switch` 默认分支：

```text
default:
  invocation satisfies never
  throw new Error(`dsh: unhandled invocation mode ${JSON.stringify(invocation)}`)
```

`satisfies` 要求左边表达式的类型满足右边的类型。`never` 表示没有可能的值。这里要求编译器确认：前面的分支已经处理了 `invocation` 的全部可能类型，到了 `default` 不应再有剩余情况。这叫穷尽性检查。

例如教学用联合类型只有 `{ mode: 'web' }` 和 `{ mode: 'help' }` 两种情况，`switch` 分别处理它们后，默认分支里的变量可以缩窄为 `never`。如果后来增加 `{ mode: 'desktop' }` 却忘记添加对应分支，默认分支仍可能接收到 desktop 类型，`satisfies never` 就会产生类型错误，提醒开发者补齐处理。

| 代码 | 作用 |
|---|---|
| `invocation satisfies never` | 类型检查时确认没有遗漏情况；不会进行运行时验证或类型转换 |
| `throw new Error(...)` | 运行时如果真的进入该分支，抛错停止当前执行 |
| `invocation as never` | 强制告诉编译器按 `never` 理解，可能掩盖遗漏，不提供同样的检查 |

运行源码的 tsx 转换不等于完整类型检查，因此仍需依靠项目的类型检查流程发现这种遗漏。`satisfies` 的类型约束在运行时被移除，真正报告运行时异常的是后面的 `throw`。

## 7. 泛型和常用类型工具

泛型用类型参数保留输入与输出的关系。例如 `function first<T>(items: T[]): T | undefined` 表示：输入元素类型为 `T` 的数组，返回一个 `T` 或 `undefined`。传入字符串数组，返回类型就是 `string | undefined`。尖括号里的类型不是运行时参数。

| 写法 | 如何阅读 |
|---|---|
| `Promise<T>` | 异步成功结果是 `T` |
| `Record<K, V>` | 键类型为 `K`、值类型为 `V` 的对象类型 |
| `Partial<T>` | `T` 的顶层字段都可省略 |
| `Pick<T, 'model'>` | 选出 `T` 的 `model` 字段 |
| `keyof T` | `T` 的属性名类型 |
| `T[K]` | 类型 `T` 中键 `K` 对应的属性类型 |
| `K extends keyof T` | 限制类型参数 `K` 为 `T` 的属性名 |

runner 中的 `brandString<SessionId>()` 涉及品牌类型：在编译期区分不同用途的字符串 ID，减少混传。它不意味着已经验证会话存在。第一遍理解用途即可，高级条件类型和映射类型遇到后再深入。

## 8. 模块、类和声明合并

`import { value }` 引入运行时导出，`import type { Agent }` 只引入类型；`export` 让其他文件可以导入成员。`./types.ts` 是相对文件路径，`@deepseek-ai/dsh-agent` 是包名。跨包阅读时先“转到定义”，再找实现和调用方。

`class` 定义带状态和方法的对象，`new` 创建实例，`this` 指向当前调用所用的实例。`extends` 继承实现，`implements` 检查类是否满足接口，不会自动生成方法。`private` 限制 TypeScript 层面的访问，`#field` 是运行时私有字段。把普通实例方法单独传作回调时，留意 `this` 是否正确绑定。

[runtime-types.ts](../../packages/core/agent/src/runtime-types.ts) 中的 `declare module` 给其他模块的接口补充成员，称为模块扩展，使用了声明合并。它让编译器认识插件提供的字段，不会在运行时安装插件。runner 的 `import type {}` 可以引入这些扩展声明；实际服务是否存在仍取决于插件加载。相关机制见 [Cordis 入门](../cordis-primer.zh.md)。

## 9. 读懂一段真实调用

```text
agent.followup(createUserMessage({
  content: [{ type: 'text', text: task }],
  source: { kind: 'user' },
}))
```

从里往外读：创建对象，`content` 是文本块数组，`source` 是另一个对象；把它传给 `createUserMessage()`，再把返回值传给 `agent.followup()`。这里 `type` 和 `kind` 是属性名，后面的字符串是运行时值。这段没有等待任务完成，后面仍需等待空闲并检查结果。

阅读后文之前，尝试回答：`task?: string` 和 `task: string | undefined` 有何区别？前者允许省略字段，后者要求字段存在但允许值为 `undefined`。为什么 `declare module` 不能保证服务可用？因为类型扩展与运行时加载是两回事。为什么传入回调不等于执行回调？因为函数值可以保存到稍后调用。

[阅读目录](README.zh.md) · [上一章](01-stack.zh.md) · [下一章](03-send-message.zh.md)
