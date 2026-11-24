# 07 · Web 启动、配置文件与命令行基础

[阅读目录](README.zh.md) · [上一章](06-modules.zh.md)

本章沿启动命令解释开发工具、入口文件、插件配置和浏览器入口。先分清每个工具负责什么，再看一条命令如何启动 Web。命令在仓库根目录执行；以下是源码阅读说明，不表示已经完成本机启动验证。

## 目录

- [1. Node.js、pnpm 和 Corepack 各是什么](#1-nodejspnpm-和-corepack-各是什么)
- [2. Web 如何启动](#2-web-如何启动)
- [3. 拆解 Node 启动命令](#3-拆解-node-启动命令)
- [4. tsx/esm 具体是什么](#4-tsxesm-具体是什么)
- [5. web 参数怎样进入程序](#5-web-参数怎样进入程序)
- [6. 启动过程中经过哪些源码](#6-启动过程中经过哪些源码)
- [7. 有哪些配置文件](#7-有哪些配置文件)
- [8. 自测与已有问题索引](#8-自测与已有问题索引)

## 1. Node.js、pnpm 和 Corepack 各是什么

| 名称 | 职责 | 在启动命令中的作用 |
|---|---|---|
| Node.js | JavaScript 运行环境 | 执行 CLI 和后端代码 |
| pnpm | 包管理器 | 安装依赖、运行 `package.json` 中的脚本 |
| Corepack | 包管理器的管理工具 | 选择并启动项目使用的 pnpm 等包管理器 |
| tsx | TypeScript 执行工具 | 为源码启动提供 TypeScript 转换与模块加载支持 |
| Vite | 前端构建工具 | 将 Web 前端源码构建为浏览器资源 |

`corepack pnpm run dev:web` 的调用关系是 Corepack 启动 pnpm，pnpm 读取根目录 `package.json` 中的 `scripts.dev:web` 并执行。项目通常通过 `packageManager` 字段指定包管理器版本。Corepack 本身不负责启动 HTTP 服务，也不属于 React 或 Cordis 框架。

如果终端能直接找到项目要求的 pnpm，可以使用 `pnpm run dev:web`。如果找不到 pnpm 但存在 Corepack，则可以使用 `corepack pnpm run dev:web`。不能假定所有 Node.js 安装都附带 Corepack；这取决于安装环境。

## 2. Web 如何启动

在仓库根目录运行开发模式：

```powershell
corepack pnpm run dev:web
```

根目录 [package.json](../../package.json) 将它映射到 `tsx scripts/dev-web.ts --poll`。[scripts/dev-web.ts](../../scripts/dev-web.ts) 先构建，再启动前端相关构建监听和 `dsh web`。首次准备环境需要安装项目依赖，具体前置要求见 [开发指南](../development.zh.md)。

已经有构建产物时，可以直接启动：

```powershell
corepack pnpm run start:web
```

`start:web` 不负责补齐构建产物，它映射到下一节的 Node 命令。Web 默认监听 `127.0.0.1:3080`；默认值位于 [packages/bundle/web-app/cordis.patch.yml](../../packages/bundle/web-app/cordis.patch.yml) 的 `webserver` 条目。查看启动参数可使用 `corepack pnpm dsh web --help`，指定端口和禁止自动打开浏览器可使用 `corepack pnpm dsh web --port 3081 --no-open`。

### 按 F5 前为什么要构建

构建产物是构建工具从源码生成的文件。例如，Vite 会把页面源码构建成浏览器加载的 HTML、JavaScript 和 CSS，放在 `apps/web/dist/`。项目的 `build:artifacts` 还准备运行时依赖的工作区包产物。Web 启动后，后端从构建目录读取页面资源并发送给浏览器。

1. 缺少产物或需要更新产物时，在仓库根目录运行 `corepack pnpm run build:artifacts`，等命令成功结束。
2. 用 VS Code 打开仓库，在「运行和调试」中选择 **DSH Web：调试源码启动**。这个配置等同于运行 `start:web`，会从 `bin.ts` 启动 Node 调试会话。[launch.json](../../.vscode/launch.json)
3. 在 `runCli()` 打断点，按 **F5** 启动。停住后可用 **F11** 进入 `parseDshArgs()`，或在 `runProfile()` 设置断点并用 **F5** 继续到那里。页面交互则可在浏览器开发者工具中调试。[CLI 入口](../../apps/cli/src/bin.ts) · [Profile 启动](../../apps/cli/src/profile-boot.ts)

已有可用产物时，可以直接按 **F5**。调试命令中的 `tsx/esm` 会在 Node 加载 CLI 源码时处理 TypeScript 语法，因此无需先手动生成 `bin.js`。浏览器资源仍由构建脚本准备。

## 3. 拆解 Node 启动命令

```powershell
node --import tsx/esm apps/cli/src/bin.ts web
```

| 部分 | 含义 |
|---|---|
| `node` | 启动 Node.js |
| `--import tsx/esm` | 执行主程序前，先加载 `tsx/esm` 模块 |
| `apps/cli/src/bin.ts` | 相对于当前工作目录的程序入口路径 |
| `web` | 传给入口程序的业务参数 |

这里的 `--import` 是 Node.js 命令行选项：预先加载模块，并执行其初始化代码。它不会在入口文件里自动创建名为 `tsx` 的变量。

源码中的 `import` 则是模块语法，例如下面这句从另一个文件导入函数，供当前文件使用：

```text
import { parseDshArgs } from './args.ts'
```

两者都涉及加载模块，但命令行选项用于预先准备环境，源码中的导入声明用于表达文件之间的依赖。`import type` 又不同：它只引入 TypeScript 类型，不用于运行时加载对应值。

## 4. tsx/esm 具体是什么

`tsx/esm` 是 npm 包 `tsx` 公开的一个子入口。`tsx` 是包名，`esm` 是这个包的导出入口名；它不是网址，也不是仓库自己定义的服务。

查看安装后的 `node_modules/tsx/package.json`，可以找到以下导出映射。这里展示的是相关字段摘录：

```json
{
  "exports": {
    "./esm": "./dist/esm/index.mjs"
  }
}
```

因此 Node.js 解析 `tsx/esm` 时，会定位 `tsx` 包，再根据 `exports` 加载它的 `dist/esm/index.mjs`。`.mjs` 表示 JavaScript ESM 模块。ESM 使用 `import` 和 `export` 来组织模块。

这个入口注册模块加载钩子，让 Node.js 在后续寻找和加载 ESM 模块时调用对应处理逻辑。`resolve` 负责确定模块对应的文件，`load` 负责加载并在需要时转换源码。tsx 使用 esbuild 转换 TypeScript 等语法，例如将 `const message: string = 'hello'` 中的类型标注移除，再让 Node.js 执行。

这不需要用户先手动输出一个 `.js` 文件，但不等于完成 TypeScript 类型检查。能够运行与类型检查通过是两件事，项目仍有单独的类型检查命令。

```text
Node.js 加载 tsx/esm
  → 注册 ESM 加载钩子
  → 加载 apps/cli/src/bin.ts
  → 处理需要转换的 TypeScript 语法
  → 执行入口代码
  → 后续导入的模块也经过相应加载处理
```

`tsx` 工具包与 `.tsx` 文件扩展名不要混淆。前者是运行工具；后者表示可包含 JSX 的 TypeScript 文件，常用于 React 组件。这里使用 `tsx/esm` 与是否编写 React 页面没有直接关系。

### 没有 `tsx/esm` 时，Node 能运行 TypeScript 吗

在仓库根目录创建示例文件 `hello.ts`。这里的 `: string` 分别标注参数和返回值的类型。

```text
function greet(name: string): string {
  return `你好，${name}`
}

console.log(greet('小明'))
```

运行 `node --import tsx/esm hello.ts` 会打印 `你好，小明`。加载时，tsx 处理类型标注，执行效果相当于用 JavaScript 写 `function greet(name) { ... }`。

| 情况 | 结果 |
|---|---|
| 保留 `--import tsx/esm`，但 Node 找不到 `tsx` 包 | 预加载失败，报 `ERR_MODULE_NOT_FOUND`，入口文件还未执行 |
| 在 Node.js 24 里运行 `node hello.ts` | 这个例子也能运行，Node 自带的类型擦除会移除其中的标注 |
| 源码使用 `enum Mode { Web, Cli }`，并按 Node.js 24 默认方式直接运行 | `enum` 需要生成运行时对象，默认的类型擦除无法处理。tsx 可以转换它 |

因此，能用 `node hello.ts` 运行这个小例子，说明它用到的语法在 Node 的原生支持范围内。仓库的 `start:web` 和调试配置使用 `tsx/esm` 来加载项目源码。

## 5. web 参数怎样进入程序

入口 [apps/cli/src/bin.ts](../../apps/cli/src/bin.ts) 的 `runCli()` 使用 `process.argv.slice(2)` 获取业务参数。对于本章命令，`process.argv` 大致如下：

```text
[
  'Node.js 可执行文件的路径',
  'apps/cli/src/bin.ts 的绝对路径',
  'web'
]
```

`slice(2)` 跳过前两项，得到 `['web']`，再交给 [apps/cli/src/args.ts](../../apps/cli/src/args.ts) 的 `parseDshArgs()`。`web` 的含义由项目定义，不是 Node.js 的内置命令。`--import tsx/esm` 由 Node.js 自己处理，不混入这里的业务参数。

参数解析完成后，`runCli()` 进入 profile 分支并调用 `runProfile()`，启动名为 `web` 的插件组合。

### `import.meta.main` 如何判断入口

源码位置：[apps/cli/src/bin.ts](../../apps/cli/src/bin.ts)，文件末尾：

```text
if (import.meta.main) {
  await runCli()
}
```

`import.meta` 是当前 ESM 模块的元信息对象，不是导入名为 meta 的模块。运行环境提供的 `.main` 属性表示当前模块是否为程序入口。

用 `node --import tsx/esm apps/cli/src/bin.ts web` 直接启动时，`bin.ts` 是入口，条件为真，于是调用 `runCli()`。若其他文件通过 `import { runCli } from './bin.ts'` 导入它，`bin.ts` 的条件为假，不会在这个 `if` 中自动启动 CLI；调用方可以自己决定何时调用 `runCli()`。被导入时，其他顶层代码仍会执行，不是整个文件都被跳过。

这种写法让文件既能直接运行，也能作为模块供其他入口使用。用途类似 Python 的 `if __name__ == '__main__'`。`.main` 不是所有运行环境或旧版 Node.js 都具备的属性，应使用项目支持的运行环境。

### `web` 在哪里变成 `--profile web`

在 [apps/cli/src/args.ts](../../apps/cli/src/args.ts) 第 206 行，`parseDshArgs()` 检查第一个参数。只要它存在、不以 `-` 开头且不是 `plugin`，就会在参数数组前加上 `'--profile'`。因此 `['web']` 变成 `['--profile', 'web']`。`...argv` 在这里是数组展开，把原数组中的每个参数依次放进新数组。其他 profile 名称也走这段通用逻辑。

### `program.parse()` 的实现在哪里

`program` 是 `new Command()` 创建的 Commander 实例。项目在 `apps/cli/src/args.ts:209` 调用 `program.parse(expanded, { from: 'user' })`。安装依赖后，可以在 `apps/cli/node_modules/commander/lib/command.js` 搜索 `parse(argv, parseOptions)`，当前安装版本位于第 1081 行。

`parse()` 先调用 `_prepareUserArgs()`。`from: 'user'` 表示传入的数组已经只包含用户参数，因此无需再跳过 Node 和脚本路径。然后 `_parseCommand()` 按 `.option('--profile <name>', ...)` 解析参数并触发项目注册的 `.action()` 回调，回调收到 `options.profile === 'web'`。`parse()` 返回 `program` 自身，启动指令由回调赋给 `resolved`，再由 `parseDshArgs()` 返回。

### `.action()` 参数中的 `BootOptions & { profile?: string }`

`&` 表示交叉类型，要求 `options` 同时满足两边的类型。它保留 `BootOptions` 定义的 `patch`、`dumpConfig` 等字段，并允许包含可选的字符串字段 `profile`。省略 `profile` 时，读取 `options.profile` 得到 `undefined`。第 174 行的回调会检查这种情况，让单独的 `dsh -h` 显示启动器帮助，普通启动缺少 profile 时则报错。Commander 根据命令行设置字段，这段类型只供 TypeScript 检查。

### `resolveBoot()` 怎样决定是否启动

在 `apps/cli/src/args.ts:111`，`resolveBoot()` 接收已经解析好的 `profile`、`options` 和应用参数 `args`。`options.patch ?? []` 在没有补丁时得到空数组。三个配置预览选项组成数组后，`.filter(Boolean)` 去掉未启用的项，`dumps.length` 就是开启的预览选项数。

| 输入 | `resolveBoot()` 返回的关键字段 |
|---|---|
| `dsh web` | `mode: 'profile'`，随后启动 Web |
| `dsh web --dump-config` | `mode: 'dump-config'`，打印合成配置 |
| `dsh web --dump-default-config` | `mode: 'dump-config'` 和 `defaultOnly: true`，打印默认配置 |
| `dsh web --dump-config-schema` | `mode: 'dump-config-schema'`，打印配置的 JSON Schema |

普通 `dsh web` 的 `dumps` 是空数组，直接走 `dumps.length === 0` 的 `return`。返回对象里的 `profile,` 是 `profile: profile` 的简写，值仍然是 `'web'`。预览命令会继续检查参数组合，例如同时开启两个预览选项，或给预览命令附加应用参数，都会报错。

### Web 在 `runCli()` 中走哪个分支

源码位置：[apps/cli/src/args.ts](../../apps/cli/src/args.ts) 的 `resolveBoot()` 与 [apps/cli/src/bin.ts](../../apps/cli/src/bin.ts) 的 `runCli()`。普通 Web 启动参数解析后的关键字段相当于：

```text
{
  mode: 'profile',
  profile: 'web',
  patches: [],
  args: [],
}
```

`mode` 表示要执行哪类操作，`profile` 表示具体启动哪个配置组合。`switch (invocation.mode)` 因此走 `case 'profile'`，将 `invocation.profile` 的值 `'web'` 传给 `runProfile()`；不存在为了普通 Web 启动而进入 `case 'web'` 的步骤。此处示意省略了其他字段；配置预览等命令会进入对应的其他模式。

```text
命令行 web
  → 展开为 ['--profile', 'web']
  → Commander parse() 执行 .action() 回调
  → resolveBoot() 构造启动指令
  → mode 为 profile，profile 为 web
  → runCli() 的 case 'profile'
  → runProfile({ profile: 'web', ... })
```

### 为什么在分支中动态导入

源码位置：[apps/cli/src/bin.ts](../../apps/cli/src/bin.ts)，`case 'profile'` 内：

```text
const { runProfile } = await import('./profile-boot.ts')
```

`import(...)` 是动态导入表达式，返回 Promise；`await` 等待模块加载，`{ runProfile }` 从模块导出对象中取出函数。加载模块和调用函数是两步，后面的 `await runProfile(...)` 才真正调用它。

把这句放在分支里，意味着先解析命令，再按所选命令加载实现。相比在文件顶部静态导入，它可以避免其他分支因为这条依赖而加载 profile 启动模块及其依赖，也把该模块的顶层初始化推迟到需要时。帮助、版本等简单命令因此可以减少无关加载工作；具体能提速多少需要测量，不能仅从写法得出数据。

动态导入也有代价：执行到这一行时才等待加载，并可能在这里出现加载失败。这里的导入位于后续 `try` 之外，加载失败不会被下面专门处理 `runProfile()` 启动错误的 `catch` 捕获。正常情况下，相同模块会被缓存，不是每次动态导入都重新执行整个模块。

| 写法 | 含义 |
|---|---|
| `node --import tsx/esm ...` | Node 启动选项，主程序执行前预加载模块 |
| `import { runProfile } from './profile-boot.ts'` | 静态导入，声明当前模块的依赖 |
| `await import('./profile-boot.ts')` | 动态导入，执行到表达式时加载模块 |
| `import.meta.main` | 读取当前模块的入口身份，不是导入操作 |

## 6. 启动过程中经过哪些源码

| 顺序 | 源码文件 | 要找的方法或内容 | 作用 |
|---|---|---|---|
| 1 | [package.json](../../package.json) | `scripts.start:web`、`scripts.dev:web` | 定义启动命令 |
| 2 | [apps/cli/src/bin.ts](../../apps/cli/src/bin.ts) | `runCli()` | 解析命令并选择启动模式 |
| 3 | [apps/cli/src/profile-boot.ts](../../apps/cli/src/profile-boot.ts) | `runProfile()`、`composeProfile()` | 组合 profile 并调用启动库 |
| 4 | [packages/boot/app-boot/src/index.ts](../../packages/boot/app-boot/src/index.ts) | `boot()` | 启动 Cordis 插件树 |
| 5 | [packages/host/webserver/src/index.ts](../../packages/host/webserver/src/index.ts) | `WebServer` | 提供 HTTP(S) 监听与路由注册 |
| 6 | [packages/bundle/web-app/src/index.ts](../../packages/bundle/web-app/src/index.ts) | Web runtime 插件 | 连接构建后的前端资源与 Web 应用启动 |
| 7 | [apps/web/index.html](../../apps/web/index.html) | `#root`、模块脚本 | 浏览器加载页面入口 |
| 8 | [apps/web/src/main.ts](../../apps/web/src/main.ts) | `new AppWebEntry(el)`、`entry.run()` | 启动浏览器侧应用 |

### 后端怎样通过 Cordis 启动

这条启动链在 Node.js 中运行。Cordis 负责组合和加载插件，HTTP 监听由 WebServer 插件通过 Node.js 的 `node:http` 或 `node:https` 建立。这个项目的 Web 后端使用这套 Node.js 技术栈，Flask 则是 Python 的 Web 框架。

1. `runProfile()` 确定 profile 的根配置路径，调用 `readProfilePatches()` 取得补丁，然后把两者和准备回调交给 `boot()`。从 `boot(NAME, rootConfig, ...)` 定位调用。[profile-boot.ts](../../apps/cli/src/profile-boot.ts)
2. `boot()` 创建 `new Context()`，通过 `await ctx.plugin(Loader)` 安装加载器，再用 `await prepare?.(ctx)` 调用传入的准备回调。插件通过 Cordis 的 `ctx` 提供和获取服务。[app-boot/src/index.ts](../../packages/boot/app-boot/src/index.ts)
3. `mountRootInclude()` 创建一条 `cordis:include` 配置项，用 `config.path` 指定根配置文件，并附上补丁。它调用 `ctx.loader.create(rootInclude)`，让 Loader 展开配置并加载插件。随后 `boot()` 等待 Loader 完成并检查启动结果。[mountRootInclude 实现](../../packages/boot/app-boot/src/index.ts)
4. Web bundle 的 `id: webserver` 指定了提供监听服务的插件。该插件最终调用 `this.server.listen(this.config.port, this.config.host, ...)`，此时 Node.js 开始建立监听。[Web 插件配置](../../packages/bundle/web-app/cordis.patch.yml) · [监听实现](../../packages/host/webserver/src/index.ts)

调试时可以依次在 `ctx.plugin(Loader)`、`ctx.loader.create(rootInclude)` 和 `this.server.listen(...)` 打断点，分别观察加载器安装、根配置与补丁，以及最终监听的地址和端口。

### 从配置到插件，代码怎样分工

Web 和 headless 复用同一套启动逻辑。Profile 描述本次要运行的插件组合，Loader 把配置项变成运行中的插件，每个插件实现自身的功能。

| 代码 | 负责什么 |
|---|---|
| `composeProfile()` 和 `readProfilePatches()` | 找到 profile，确定插件包的解析方式，收集 bundle 配置和补丁 |
| `runProfile()` | 准备环境、命令行参数和退出处理，将本次启动所需的数据交给 `boot()` |
| `boot()` 和 Loader | 创建 Cordis 上下文，读取根配置并应用补丁，加载插件并检查启动结果 |
| WebServer 插件 | 接收地址和端口，调用 Node.js 的 `server.listen()` |

以 WebServer 为例，配置项的 `id: webserver` 标识这一项，补丁可按该 ID 修改它。条目中的 `name` 指定 Loader 要加载的插件包，`config` 提供插件运行所需的参数。端口从 `config.port` 传给插件，最后由插件通过 `this.config.port` 传给 `server.listen()`。同一个 WebServer 可以用不同的端口启动，通用的 `boot()` 只需处理统一的插件配置结构。[配置条目](../../packages/bundle/web-app/cordis.patch.yml) · [监听代码](../../packages/host/webserver/src/index.ts)

`boot()` 和 `runProfile()` 用 `prepare(ctx)` 回调交接启动数据。`boot()` 掌握调用时机，在创建上下文、安装 Loader 之后和加载配置树之前执行回调。`runProfile()` 掌握本次启动需要提供的数据，例如 profile 信息、环境变量快照和命令行参数。这样，配置树中的插件一开始就能从上下文获取这些数据。[runProfile 实现](../../apps/cli/src/profile-boot.ts) · [boot 实现](../../packages/boot/app-boot/src/index.ts)

`runProfile()` 也管理退出。准备回调会先把上下文保存到 `app.current`，这样 `boot()` 尚未返回时收到停止信号，也能清理已创建的上下文。`dispose()` 用 `??=` 保存第一次清理的 Promise，后续调用复用它。启动成功且应用仍处于活动状态时，`appReady.commit()` 通知等待启动的代码。

调试时可以先在 `boot(...)` 前查看 `options.profile`、`rootConfig` 和 `profileContext.startedBundles`，再在 `server.listen()` 查看最终端口。沿着端口的传递路径，就能看到配置、启动器和插件各自负责哪一步。

### 浏览器在哪里真正挂载 React

`runProfile()` 启动后端，后端为浏览器提供页面资源。浏览器打开页面后，才开始执行前端代码。React 的实际挂载点是 `mountApp()`。[React 渲染器](../../packages/client/ui-renderer/src/client/index.ts)

1. 浏览器拿到由 Web 构建产物提供的页面。源码模板中，`index.html` 提供 `<div id="root">`，并声明模块入口 `main.ts`。[index.html](../../apps/web/index.html)
2. `main.ts` 取得 `#root`，创建 `new AppWebEntry(el)`，再调用 `entry.run()`。这个方法先执行 `bootClient()`，然后调用 `mountClient(ctx, this.container)`。[main.ts](../../apps/web/src/main.ts) · [boot.ts](../../packages/client/web/src/boot.ts)
3. `mountClient()` 通过 Cordis 获取 `uiRenderer` 服务，在挂载效果中调用 `scope.uiRenderer.mount(container)`。[mount.ts](../../packages/client/web/src/mount.ts)
4. `uiRenderer.mount()` 调用 `mountApp(container, buildRenderApp({ ctx }))`。如果容器已有启动页 DOM，`hydrateRoot()` 先接管它，再切换到应用界面。否则走 `createRoot(container)` 和 `root.render(app())`。[ui-renderer/src/client/index.ts](../../packages/client/ui-renderer/src/client/index.ts)

在浏览器开发者工具的 Sources 中给 `mountApp()` 打断点，再刷新页面，可以观察 `container` 和 `boot`。后端启动的断点则放在 VS Code 的 Node 调试会话里。

## 7. 有哪些配置文件

| 文件或位置 | 配置内容 |
|---|---|
| 根目录 `package.json` | 项目脚本、依赖和包管理器声明 |
| `apps/web/package.json` | 前端构建脚本与依赖 |
| `apps/web/vite.config.ts` | 浏览器资源的构建配置 |
| `packages/bundle/base/cordis.patch.yml` | Agent、工具和会话等基础插件 |
| `packages/bundle/web-app/cordis.patch.yml` | Web 服务、传输层与浏览器插件 |
| `$DSH_HOME/profiles/web/package.json` | Web profile 的 bundle 选择与依赖 |
| `$DSH_HOME/profiles/web/cordis.patch.yml` | Web profile 自身的配置覆盖 |
| `$DSH_HOME/cordis.patch.yml` | Harness home 层面的配置覆盖 |
| `--patch` 指定的 YAML 文件 | 本次启动的额外覆盖 |
| 启动目录 `.env` 与 `$DSH_HOME/.env` | 允许从文件加载的环境变量 |

`$DSH_HOME` 表示 Harness 用户数据目录，不是仓库目录。默认 Web 组合先加载基础 bundle 和 Web bundle，再应用 profile、自身 home 层以及命令行 patch。后加载的 patch 可以覆盖前面的条目；替换一个条目的 `config` 时替换的是整个对象，需要保留的字段也要写上。更完整的规则见 [app-boot 说明](../../packages/boot/app-boot/README.zh.md)。

环境变量有单独的优先级：进程继承的环境高于启动目录 `.env`，后者高于 Harness home `.env`。并非所有变量都允许放入 `.env`；决定启动位置等行为的变量存在限制，应查阅 app-boot 的环境加载规则。不要把凭据写入提交到仓库的配置文件。

查看最终合成的插件配置：

```powershell
corepack pnpm dsh web --dump-config
```

这个命令用于理解最终插件树。Vite 配置负责前端构建，Cordis patch 负责运行时插件组合，两者处理的阶段不同。

### Profile 和 patch 分别记录什么

**Profile 是一套启动方案，patch 是对插件配置列表的一组修改。** 例如 `dsh web` 选中 `web` profile。这个 profile 目录下的 `package.json` 用 `dsh.profile.bundles` 记录按顺序选用的 bundle。同目录的 `cordis.patch.yml` 可以继续定制该方案。bundle 也会提供自己的 patch，所以一次启动可以应用多份 patch。[Profile 配置说明](../../packages/boot/app-boot/README.zh.md)

| 配置例子 | 内容 | 加载后的效果 |
|---|---|---|
| Web bundle 的 `insert` | 添加 `id: webserver`，`name` 指向 WebServer 插件包，`inject` 声明依赖 `webStartup`，`config` 传入地址和端口 | WebServer 用收到的配置建立监听，缺省端口为 `3080`。[Web bundle patch](../../packages/bundle/web-app/cordis.patch.yml) |
| Headless bundle 的 `insert` | 添加 `id: headless-runner`，从 `headlessStartup` 获取任务、会话 ID 和 JSON 输出选项 | 启动一次命令行任务。[Headless bundle patch](../../packages/bundle/headless/cordis.patch.yml) |
| Headless bundle 的 `id: hmr` 和 `disabled: true` | 按 ID 找到基础 bundle 已有的热重载条目，将它禁用 | Headless 不启用该热重载插件。[Headless bundle patch](../../packages/bundle/headless/cordis.patch.yml) |

`id` 标识配置条目，后续 patch 可以据此修改同一项。不同 profile 可以用同一套加载逻辑，分别从自己的 bundle 和覆盖文件得到最终的插件列表。

### Windows 在哪里找到 `profiles`

`$DSH_HOME/profiles` 中的 `$DSH_HOME` 代表 DSH 实际使用的用户目录。如果设置了 `DSH_HOME` 环境变量，CLI 使用该位置。否则默认使用 Node.js 识别的用户主目录下的 `.dsh`。因此环境变量为空时，仍可以有 `profiles` 目录。启动 `dsh web` 时，程序会按需初始化对应的 Web profile。[用户目录解析](../../packages/util/home-paths/src/index.ts) · [Profile 加载](../../packages/boot/app-boot/src/profile.ts)

在使用默认目录的 Windows 环境中，打开 PowerShell 查看 Web profile 的文件：

```powershell
Get-ChildItem -Force "$env:USERPROFILE\.dsh\profiles\web"
```

如果设置了 `DSH_HOME`，改用 `Get-ChildItem -Force (Join-Path $env:DSH_HOME 'profiles\web')` 查看。目录中的 `package.json` 是实际选择 bundle 的位置，前面链接的 bundle patch 则描述各自对插件条目的修改。

## 8. 自测与已有问题索引

尝试解释：谁执行 `dev:web` 脚本？`--import` 在入口文件之前还是之后生效？`tsx/esm` 怎样对应到实际文件？`web` 为什么会出现在 `process.argv.slice(2)` 中？调整插件配置与调整 Vite 构建配置有什么区别？直接启动与导入 `bin.ts` 时，`import.meta.main` 有何区别？为什么 `web` 对应的是 `profile` 分支？动态导入是否会自动调用导出的函数？

此前涉及的其他源码阅读问题已有对应章节：

- [前后端使用什么技术](01-stack.zh.md)：React、Node.js、Cordis 和 Remote 的职责。
- [阅读源码需要哪些 TypeScript 知识](02-typescript.zh.md)：类型、回调、异步、模块语法和 `satisfies never` 穷尽性检查。
- [点击发送后由哪个服务接收请求](03-send-message.zh.md)：前端请求到服务端 Controller 的路线。
- [InputBar 是否是通用按钮](04-composer.zh.md)：会话输入栏与按钮、状态机之间的关系。
- [流程有哪些参与模块及接口](06-modules.zh.md)：方法输入、输出和上下游关系。

[阅读目录](README.zh.md) · [上一章](06-modules.zh.md)
