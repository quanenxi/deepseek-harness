---
kind: logging_system
name: 基于 Cordis Context.logger 与进程级 stderr 的双层日志体系
category: logging_system
scope:
    - '**'
source_files:
    - packages/acp/acp/src/index.ts
    - packages/client/hmr/src/client/index.ts
    - packages/client/hmr/src/index.ts
    - packages/client/modules/src/index.ts
    - packages/client/ui-commands/src/client/service.ts
    - packages/compaction/compaction-basic/src/index.ts
    - packages/boot/app-boot/src/index.ts
    - packages/client/connection/src/client/connection.ts
    - packages/client/runtime/src/client/contract/store.ts
    - packages/api/gateway/src/client/index.ts
---

## 1. 使用的系统/方法

仓库没有引入独立的第三方日志库（如 pino、winston、bunyan、debug）。所有业务代码的日志输出统一通过 **Cordis 框架注入的 `Context.logger`**，其接口为 `{ warn(message), error(error), info(message) }`；此外，在应用启动阶段和测试中还会直接使用 Node 原生的 `console.log/info/warn/error` 以及 `process.stderr.write`。

- 插件/服务层：通过 `ctx.logger` / `logger`（从 `ctx` 解构）调用 `warn`、`error`、`info`。例如 `packages/acp/acp/src/index.ts` 中 `const logger = ctx.logger`，随后使用 `logger.warn(...)`；`packages/client/hmr/src/client/index.ts` 使用 `ctx.logger.warn`、`ctx.logger.error`；`packages/compaction/compaction-basic/src/index.ts` 使用 `ctx.logger.info`、`ctx.logger.warn`。
- 启动/引导层：`packages/boot/app-boot/src/index.ts` 中的 `installFailLoud` 直接写 `proc.stderr.write(...)` 输出致命错误，并 `proc.exit(1)`；`loadEnv`、`readEnvLayer`、`renderConfigDump` 等通过可注入的 `warn: (line: string) => void` 回调输出诊断行，默认 sink 是 `process.stderr.write`。
- 客户端运行时：`packages/client/connection`、`packages/client/runtime`、`packages/api/gateway/src/client` 等直接使用 `console.warn` / `console.error` 输出连接丢失、WebSocket frame 丢弃、store 持久化失败等运行时异常。

## 2. 关键文件与包

| 角色 | 关键路径 | 作用 |
|---|---|---|
| 插件日志入口 | `packages/acp/acp/src/index.ts` | 通过 `ctx.logger` 记录 ACP 会话更新失败、子 agent 清理失败、连接关闭等 |
| HMR 客户端日志 | `packages/client/hmr/src/client/index.ts`、`packages/client/hmr/src/index.ts` | 使用 `ctx.logger.warn/error` 报告热重载失败、不可解析事件帧 |
| 模块加载日志 | `packages/client/modules/src/index.ts` | 使用 `this.ctx.logger.error` 报告模块 flush 失败 |
| 压缩器日志 | `packages/compaction/compaction-basic/src/index.ts` | 使用 `ctx.logger.info` 记录步骤开始，`ctx.logger.warn` 记录压缩失败 |
| 命令 UI 日志 | `packages/client/ui-commands/src/client/service.ts` | 使用 `this.ctx.logger.warn` 记录命令监听器失败 |
| 启动致命错误 | `packages/boot/app-boot/src/index.ts` | `installFailLoud` 将未处理 rejection 写入 `stderr` 并退出；`assertEntriesActivated` 抛出带堆栈的诊断错误 |
| 配置 dump 警告 | `packages/boot/app-boot/src/index.ts` | `renderConfigDump` 把 Loader include 的 printf-style 警告 (`%C`) 转成普通字符串并通过 `warn` sink 输出 |
| 客户端控制台日志 | `packages/client/connection/src/client/*.ts`、`packages/client/runtime/src/client/**/*.ts`、`packages/api/gateway/src/client/index.ts` | 直接使用 `console.warn` / `console.error` 输出连接/存储/网关异常 |

## 3. 架构与约定

- **上下文注入式日志**：每个 Cordis 插件/Service 通过 `Context` 获取 `logger`，不直接依赖全局日志对象。这使日志目标由宿主（CLI/Web）装配，插件本身保持无副作用。
- **两级日志通道**：
  - 运行期插件日志走 `ctx.logger`，由 Cordis 宿主决定最终 sink（CLI 下通常落到 stdout/stderr 或终端输出）。
  - 启动期致命错误绕过插件树，直接写 `process.stderr` 并 `exit(1)`，确保即使插件初始化失败也能看到原因。
- **结构化字段**：当前实现以“前缀 + 消息”为主，没有统一的 JSON 结构化 schema。常见模式是在消息开头带上模块标识，如 `[web-runtime]`、`client-hmr:`、`acp:`、`[ui-commands]`、`[client-connection]` 等，便于 grep 定位来源。
- **日志级别**：仅观察到 `warn`、`error`、`info` 三种级别被使用；未见 `debug` 级别的常规使用（仅在 `code-runtime-worker-thread` 测试中 mock 了 `shim.debug`）。
- **测试断言日志**：测试通过覆盖 `harness.ctx.logger.warn` 或使用 `vi.spyOn(b.ctx.logger, 'warn')` 来捕获并验证警告输出，说明日志行为是可测试契约的一部分。

## 4. 约定与约束

- **插件必须通过 `ctx.logger` 输出**：业务代码（acp、hmr、modules、compaction、ui-commands）全部遵循这一约定，避免硬编码全局日志。
- **启动致命错误必须写 stderr 并 exit(1)**：`installFailLoud` 强制要求——任何未被捕获的插件初始化 rejection 都会先写一行 `fatal load failure` 到 stderr，再退出，保证 ACP 模式下 stdout 不被污染。
- **Loader include 的 printf-style 警告需适配**：`renderConfigDump` 显式注释说明 include 通过 Cordis 的 printf-style logger（`%C` 占位符表示 code），dump 场景没有 logger，因此用正则替换 `%C` 并把参数序列化进警告数组，再通过 `warn` sink 输出。这是对底层 Logger 接口的显式兼容。
- **客户端运行时允许 console.* 直出**：Web 客户端连接、store 持久化、UI 组件等直接使用 `console.warn/error`，因为它们在浏览器环境中本身就是标准输出通道。
- **日志内容包含来源前缀**：约定在消息开头标注模块名（如 `acp:`、`client-hmr:`、`[web-runtime]`、`[client-connection]`），用于快速区分不同子系统。
- **无集中 log-level 开关**：仓库中没有发现环境变量或配置文件控制日志级别的全局开关；级别选择由调用方自行决定（warn vs error vs info）。
- **无结构化 JSON 日志 schema**：未发现统一的日志字段定义（如 timestamp、level、traceId、sessionId 等），日志以纯文本拼接为主。