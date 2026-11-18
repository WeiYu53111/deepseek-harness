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

Node 后端与浏览器前端分别有入口。执行 `bin.ts` 会启动后端应用；浏览器拿到页面资源后才执行 `main.ts`。组件渲染和页面按钮处理发生在后一个阶段。

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

## 8. 自测与已有问题索引

尝试解释：谁执行 `dev:web` 脚本？`--import` 在入口文件之前还是之后生效？`tsx/esm` 怎样对应到实际文件？`web` 为什么会出现在 `process.argv.slice(2)` 中？调整插件配置与调整 Vite 构建配置有什么区别？

此前涉及的其他源码阅读问题已有对应章节：

- [前后端使用什么技术](01-stack.zh.md)：React、Node.js、Cordis 和 Remote 的职责。
- [阅读源码需要哪些 TypeScript 知识](02-typescript.zh.md)：类型、回调、异步与模块语法。
- [点击发送后由哪个服务接收请求](03-send-message.zh.md)：前端请求到服务端 Controller 的路线。
- [InputBar 是否是通用按钮](04-composer.zh.md)：会话输入栏与按钮、状态机之间的关系。
- [流程有哪些参与模块及接口](06-modules.zh.md)：方法输入、输出和上下游关系。

[阅读目录](README.zh.md) · [上一章](06-modules.zh.md)
