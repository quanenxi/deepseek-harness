---
kind: error_handling
name: 结构化错误体系：HarnessError/LlmError 与 code 路由
category: error_handling
scope:
    - '**'
source_files:
    - packages/llm/llm/src/error.ts
    - packages/core/agent-loop/src/agent.ts
    - packages/core/agent-loop/src/index.ts
    - packages/sandbox/sandbox/src/index.ts
    - packages/core/tools/src/index.ts
    - packages/core/tools/src/json-schema.ts
    - packages/core/tools/src/schema.ts
    - packages/core/tools/src/code-mode.ts
    - packages/fs/fs/src/types.ts
    - packages/goal/goal/src/runtime.ts
    - packages/attachment/attachment/src/error.ts
    - packages/client/runtime/src/client/sessions/service.ts
    - packages/client/runtime/src/client/workspaces/service.ts
    - packages/client/ui-slots/src/renderer.ts
    - packages/api/gateway/src/index.ts
    - packages/sandbox/sandbox-windows-acl/src/errors.ts
    - packages/subprocess/subprocess-local/src/index.ts
---

## 1. 使用的系统/方法

仓库采用**基于 `@deepseek-ai/dsh-llm` 的 `HarnessError` 基类的结构化错误模型**，所有跨包可被上层（agent-loop、session、RPC 网关等）统一消费的错误都携带一个稳定的机器可读 `code` 字段。LLM 调用链路额外使用同包的 `LlmError`（继承自 `HarnessError`），将 provider 返回的失败原样包装为 `failure: { message, code }` 结构，供重试策略和会话事件消费。

非 LLM 领域（sandbox、tools、fs、goal、client runtime 等）各自定义业务错误类，多数直接 `extends HarnessError`；少数因依赖循环而“重新实现”相同形状（如 `AttachmentError`）。普通 `Error` 仍用于纯内部校验失败（参数非法、阶段不一致等），不会被序列化到 session 或 RPC 边界。

## 2. 关键文件与包

- **错误基类与工具**：`packages/llm/llm/src/error.ts` — 定义 `HarnessError`、`errorChain`、`isHarnessError`、`isContextWindowExceededError`、`isQuotaExceededError` 以及常量 `CONTEXT_WINDOW_EXCEEDED_CODE`、`QUOTA_EXCEEDED_CODE`、`EMPTY_RESPONSE_CODE`、`INVALID_CREDENTIAL_CODE`。
- **Agent Loop 错误归一化**：`packages/core/agent-loop/src/agent.ts` — 在 turn/end 处把任意 `unknown` 异常转换为 `{ message, code }`：`LlmError` 保留其 `failure`，否则用 `errorChain(error)` + `UNKNOWN` 代码。
- **Sandbox 错误**：`packages/sandbox/sandbox/src/index.ts` — 导出 `SANDBOX_UNAVAILABLE = 'SANDBOX_UNAVAILABLE'` 及 `SandboxUnavailableError extends HarnessError`，并注释要求 provider “fail closed”。
- **Tools 错误**：`packages/core/tools/src/index.ts`（`ToolNotFoundError`、`ToolOutputError`）、`packages/core/tools/src/json-schema.ts`（`JsonSchemaError`）、`packages/core/tools/src/schema.ts`（`ToolArgsError`）、`packages/core/tools/src/code-mode.ts`（`CodeRunFailedError`）均 `extends HarnessError`。
- **FS 错误**：`packages/fs/fs/src/types.ts` 中的 `FsError extends HarnessError`。
- **Goal 错误**：`packages/goal/goal/src/runtime.ts` 中的 `GoalError extends HarnessError`。
- **Client/Runtime 错误**：`packages/client/runtime/src/client/sessions/service.ts`（`SessionCreateError`、`SessionForkError`）、`workspaces/service.ts`（`WorkspaceCreateError`、`DirectoryBrowseError`）、`ui-slots/renderer.ts`（`StaleAuthorizationError`、`SlotOwnershipError`）等，均为 `extends Error` 的业务错误。
- **API Gateway**：`packages/api/gateway/src/index.ts` 中 `TypertGatewayError extends Error` 作为 RPC 层错误。
- **Windows ACL 错误**：`packages/sandbox/sandbox-windows-acl/src/errors.ts` 中 `Win32Error extends Error`。
- **子进程清理聚合**：`packages/subprocess/subprocess-local/src/index.ts` 在 teardown 失败时抛出 `AggregateError`，把多个清理失败合并。

## 3. 架构与约定

### 3.1 分层错误类型
- **基础设施层**：`HarnessError`（`@deepseek-ai/dsh-llm`）是所有“可被外部消费”的错误基类，强制带 `code` 与可选 `cause` 链。
- **LLM 层**：`LlmError` 继承 `HarnessError`，并通过 `failure` 字段透传 provider 原始失败（message + code），agent-loop 在 request-error waterfall 中据此决定 retry / abort。
- **领域层**：每个能力域（sandbox、tools、fs、goal、client runtime 等）定义自己的业务错误类，优先 `extends HarnessError`；若存在循环依赖则“重新实现相同 shape”（见 `AttachmentError` 注释）。
- **内部校验层**：直接使用 `new Error(...)` / `new TypeError(...)` 表达参数非法、状态机越界等编程期错误，不期望被捕获后序列化。

### 3.2 错误传播路径
- **Agent loop 是核心汇聚点**：`agent.ts` 的 catch 块把任何异常归一化为 `{ kind: 'error', error: { message, code } }`，写入 `turn/end` 事件；`LlmError` 保持结构化，其他错误降级为 `UNKNOWN` 代码并用 `errorChain` 渲染完整 cause 链。
- **Provider 错误向上包装**：当 LLM stream finish 为 `error` 且 action 不是 `retry` 时，agent-loop 会 `throw new LlmError(finish.failure.message, finish.failure.code, finish.failure)`，使下游能 `instanceof LlmError` 并读取 `.code`。
- **Sandbox 必须 fail-closed**：`SandboxProvider.confine` 无法执行请求模式时必须抛错（`SandboxUnavailableError`），禁止静默回退到无限制模式。
- **RPC/客户端边界**：client runtime 与 API gateway 各自定义 `extends Error` 的错误类，通过 RPC 映射到 host 侧对应错误。

### 3.3 错误诊断与展示
- `errorChain(value)` 递归渲染 `Error.cause` 链与 `AggregateError.errors`，跳过重复消息，对不可渲染值兜底为 `<unrenderable value>`，专供日志/notice 等只读场景。
- `isContextWindowExceededError` / `isQuotaExceededError` 用正则识别 provider 文本，把不同厂商措辞归一为 `CONTEXT_WINDOW_EXCEEDED` / `QUOTA` 两类稳定码。

## 4. 约定与约束

- **路由依据 `code` 而非 `name`/`message`**：`HarnessError` 文档明确 `code` 是“stable, programmatic”，消费方应 `instanceof` 或检查 `code` 分支处理，不应解析 `message`。
- **LLM 失败必须结构化**：agent-loop 注释要求“Every failure is structured: an `LlmError` keeps its facts, anything else flattens to `errorChain` text under the `UNKNOWN` code.”——即非 LLM 异常也要带上 `UNKNOWN` 码进入会话。
- **上下文溢出/配额耗尽使用常量码**：`CONTEXT_WINDOW_EXCEEDED_CODE`、`QUOTA_EXCEEDED_CODE`、`EMPTY_RESPONSE_CODE`、`INVALID_CREDENTIAL_CODE` 是跨 provider 的统一码，避免硬编码字符串散落各处。
- **Sandbox 不可用时抛 `SANDBOX_UNAVAILABLE`**：`SandboxUnavailableError` 的构造器固定使用该 code，禁止静默降级。
- **工具错误需携带 `code`**：`ToolNotFoundError`、`ToolOutputError`、`JsonSchemaError`、`ToolArgsError`、`CodeRunFailedError`、`FsError`、`GoalError` 等均继承 `HarnessError`，确保 tool/result 事件能携带结构化失败。
- **子进程清理失败聚合**：teardown 阶段多个清理操作失败时使用 `AggregateError` 汇总，保证全部清理问题可见。
- **AbortSignal 原因透传**：agent-loop 多处将 `signal.reason instanceof Error ? signal.reason : new Error(String(signal.reason))` 作为拒绝原因，保证取消语义可被捕获方识别。
- **跨包共享错误时的依赖规避**：`AttachmentError` 刻意不继承 `HarnessError`，而是“重新实现相同 shape”，以避免 `@deepseek-ai/dsh-llm` 反向依赖 attachment 包造成的循环依赖；注释说明“consumers route on `code`, never on the prototype chain”。

## 5. 未覆盖但观察到的模式

- 大量内部参数校验仍使用裸 `throw new Error(...)` / `throw new TypeError(...)`（如 agent-loop index.ts 中对 `maxParallelToolCalls`、`maxTokens` 的检查），这些错误不会出现在 session 事件中，属于开发期防御。
- Windows ACL 子模块使用自定义 `Win32Error extends Error` 封装 FFI 调用失败，体现平台特定错误的本地化封装。
- client 层错误（`SessionCreateError`、`WorkspaceCreateError` 等）仅 `extends Error`，因为它们在 client 运行时内消费，不需要跨进程结构化传输。
