# 08 · 二次开发：在 Web 中加载第一个工具

[阅读目录](README.zh.md) · [上一章](07-web-startup.zh.md)

做一个接收名字并返回问候语的 `greet` 工具。先确认 Cordis 加载了你的插件，再用 Web 会话观察模型调用工具。开始前需要能从当前仓库启动 Web，并在 Web 中选用一个支持工具调用的模型。[Web 启动与调试](07-web-startup.zh.md)

## 1. 创建插件和补丁

从仓库根目录新建 `scratch-plugin/src/my-plugin.ts`。先用官方教程中的最小插件，保留它的加载日志。接着创建 `scratch-plugin/cordis.yml`，按下文填写插件入口的实际绝对路径。Windows 路径可用正斜线。[第一个插件及完整代码](../user/develop/basic/index.zh.md)

```yaml
- insert:
    - id: scratch-hello
      name: 'D:/path/to/deepseek-harness/scratch-plugin/src/my-plugin.ts'
```

`id` 标识插件配置项，`name` 指向要加载的模块。这个补丁可以在启动时叠加到 Web profile 上。

## 2. 启动 Web，验证插件加载

在仓库根目录执行以下命令。如果当前已有 Web 实例占用同一端口，先停止该实例。

```powershell
corepack pnpm dsh web --patch ./scratch-plugin/cordis.yml
```

终端出现 `[hello-plugin] plugin loaded!`，说明 Cordis 已经调用了插件的 `apply(ctx)`。如果使用 VS Code 调试，将启动配置的 `args` 改为 `["web", "--patch", "./scratch-plugin/cordis.yml"]`，再在 `apply(ctx)` 的第一条可执行语句处打断点并按 F5。两种启动方式任选一种。

如果看不到日志，检查启动终端是否有加载错误，以及补丁中的绝对路径是否指向刚创建的文件。

## 3. 注册工具，从页面验证调用

将 `my-plugin.ts` 替换为官方教程里的 `greet` 实现，沿用同一份补丁并重启。实现通过 `inject = ['tools']` 等待工具服务，再在 `apply(ctx)` 中注册工具。工具的 `parameters` 声明输入，`execute(args)` 计算返回值，`output.render` 把返回值转换成模型可读的内容。[开发一个工具及完整代码](../user/develop/basic/tool.zh.md)

打开 Web 并发送 `Use the greet tool to greet Ada.`。在 VS Code 中给 `execute(args)` 打断点。模型发出工具调用时，应该能观察到 `args.name === 'Ada'`，工具结果为 `Hello, Ada!`。只看到模型回复问候语时，以工具调用记录或断点命中确认是否真的执行了工具。

## 4. 把练习换成自己的功能

可以先把问候逻辑换成一个只读业务查询，明确它接收什么参数、返回什么数据，以及查询失败时如何报告。需要让多个工具复用同一套业务逻辑时，再查阅 Cordis 的服务注册和依赖注入。[工具编写参考](../cookbook/adding-a-tool.zh.md) · [Cordis 服务教程](../cordis-tutorial/03-services.zh.md)
