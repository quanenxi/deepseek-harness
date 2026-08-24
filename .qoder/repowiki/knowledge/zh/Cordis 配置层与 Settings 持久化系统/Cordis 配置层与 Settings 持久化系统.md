---
kind: configuration_system
name: Cordis 配置层与 Settings 持久化系统
category: configuration_system
scope:
    - '**'
source_files:
    - packages/settings/settings/src/index.ts
    - packages/settings/settings-file/src/index.ts
    - apps/cli/src/profile-boot.ts
    - apps/cli/src/args.ts
    - docs/cordis-tutorial/05-config.md
    - docs/config-catalog.md
    - apps/cli/config/agent-presets/standard/preset.yml
    - examples/headless-agent/cordis.yml
    - examples/acp-agent/cordis.yml
---

## 1. 整体方案

DeepSeek Harness 采用 **两层配置** 的架构：

- **部署/组合配置（Composition Config）**：通过 Cordis 插件框架，以 `cordis.yml` 描述插件树、每个 entry 的 `config` 块以及 patch 叠加层。加载时由 schemastery schema 校验，失败即中止启动。
- **运行时用户设置（User Settings）**：通过 `@deepseek-ai/dsh-settings` 抽象 + `@deepseek-ai/dsh-settings-file` 文件实现，将 per-namespace 的用户覆盖值持久化到 `$DSH_HOME/settings.yaml`（或 `.json`），支持热重载、跨进程写锁、注释保留的 YAML diff 写入。

两套系统在 CLI 启动流程中交汇：`apps/cli/src/profile-boot.ts` 先组装 Cordis profile 的 patch 栈，再 boot 出包含 settings provider 的完整 fiber 树。

## 2. 关键文件与包

| 路径 | 职责 |
|---|---|
| `packages/settings/settings/src/index.ts` | `SettingsProvider` 抽象基类：命名空间注册、三层合并（schema defaults → base → user）、变更监听、revision 冲突检测、`settings/updated` 事件 |
| `packages/settings/settings-file/src/index.ts` | `FileSettingsProvider`：YAML/JSON 文档存储、chokidar 热重载、原子写+文件锁、comment-preserving 增量渲染 |
| `apps/cli/src/profile-boot.ts` | 组装 profile 的 patch 栈（bundle patches → profile.patch.yml → $DSH_HOME/cordis.patch.yml → --patch overlays → telemetry switch），并挂载 HMR watcher |
| `apps/cli/src/args.ts` | CLI 参数解析（`--profile`、`--patch`、`--dump-config`、`--dump-default-config`） |
| `docs/cordis-tutorial/05-config.md` | 官方教程，说明 `config:` 块、schemastery schema、`!!js` 标签等约定 |
| `docs/config-catalog.md` | 自动生成所有包的 `Config` 类型目录（由 `scripts/gen-config-catalog.ts` 生成） |
| `apps/cli/config/agent-presets/*/preset.yml` | 内置 agent preset 的默认配置根 |
| `examples/*/*.cordis.yml` | 示例工程的 Cordis 组合配置文件 |

## 3. 架构与约定

### 3.1 Cordis 组合配置（`cordis.yml`）

- 每个 entry 可带 `config` 块，其 schema 必须为同名的 `Schema<T>` 导出（Schemastery 或 Standard Schema 兼容），加载时校验，错误直接让 fiber 进入 FAILED。
- 支持 `!!js` tag 在 `config` 和 `disabled` 字段中进行运行时计算（如读取环境变量）。
- 组合顺序严格分层：`dsh.profile.bundles` 中的 bundle patches → profile 自身 `cordis.patch.yml` → `$DSH_HOME/cordis.patch.yml`（机器级覆盖，优先级更高）→ `--patch` 命令行 overlay → 遥测开关补丁。
- 运行时对 `cordis.patch.yml` 和 home patch 启用 HMR 热重载（通过 `watchUserPatches` + cordis-plugin-hmr）。

### 3.2 Settings 命名空间系统

- 每个能力域通过 `ctx.settings.register(ns, schema, { base, validate })` 声明一个命名空间；`ns` 必须符合 kebab-case 正则 `/^[a-z][a-z0-9-]*$/`。
- 最终值 = `mergeLayers(schema.defaults, base, user_section)`，三者不可变冻结后下发。
- 提供三种写入 API：`update(patch)` 合并、`replace(section)` 全量替换、`mutate(ops)` 路径级 set/unset（专为持有 redacted 视图的 UI 设计）。
- 每次 raw section 变化递增 monotonic `revision`，调用方通过 `expectedRevision` 实现乐观并发控制，冲突抛出 `SettingsConflictError`。
- 通过 `installSettingsSection(ctx, ns, schema, entry, hooks)` 将消费方与 settings provider 解耦：provider 存在时走 settings scope，不存在时回退到 composition entry。
- secrets 通过 schema 的 `role('secret')` 标记，经 `redactSecrets` 脱敏后暴露给 wire/UI。

### 3.3 文件持久化（`FileSettingsProvider`）

- 文档位置：默认 `<harness home>/settings.yaml`，也支持 `.yml` / `.json`；harness home 来自 `resolveDshHome()`（优先 `$DSH_HOME`，否则 `~/.dsh`）。
- 写路径：`withFileLock` 独占锁 → reconcileFromDisk 读回磁盘最新内容 → 对 YAML 使用 `yaml.Document` 做 leaf-level diff 保留注释 → `writeFileAtomic` 原子写，权限 `0600`，目录 `0700`。
- 读路径：启动时 `load()` 解析；chokidar 监听文件变化，debounce 后 queueRefresh，解析失败仅 warn 并保留 last good document。
- 所有 namespace 共享同一份文档，不同 namespace 的写队列通过单例 `operations` Promise 链串行化，避免并发写互相覆盖。

### 3.4 启动装配

`runProfile` 在 `boot()` 之前注入 `LaunchEnvironmentSnapshot`（冻结的环境快照）和 `cmdline`（命令参数 + `exit` 钩子），随后任何插件可通过 `ctx.inject(['settings'], ...)` 获取 settings provider 并使用。

## 4. 约定与约束

- **Schema 驱动**：所有 `cordis.yml` 中的 `config` 必须匹配同名导出的 Schemastery schema；`gen-config-catalog.ts` 会交叉校验 runtime schema 与声明类型，CI 中通过 `verify-config-catalog` 强制。
- **命名空间格式**：`settingsNamespace(value)` 强制 kebab-case，不合法直接抛 TypeError。
- **JSON-only 写入**：`cloneJsonShaped` 拒绝非 JSON 值（Date、Map、BigInt、循环引用、undefined 数组项等），确保 YAML/JSON 持久化可逆。
- **安全权限**：settings 文档以 `0600` 写入，目录以 `0700` 创建；secrets 字段通过 `redactSecrets` 从 wire 表面剥离。
- **热重载安全**：reload 失败仅 warn 并保留 last good document；dispose 时等待所有 write chain 和 watcher 回调 settle。
- **Telemetry 开关**：`DSH_TELEMETRY_DISABLED` 非空即禁用遥测行，通过 patch 注入而非硬编码，且对无 telemetry row 的自定义 profile 透明跳过。
- **Home patch 优先级**：`$DSH_HOME/cordis.patch.yml` 始终高于 profile 自身 layer，用于机器级偏好覆盖。
- **配置 dump**：CLI 提供 `--dump-config` 与 `--dump-default-config` 输出当前/默认组合结果，便于调试。

## 5. 相关脚本与验证

- `scripts/gen-config-catalog.ts`：扫描各包导出 `Config` 类型生成 `docs/config-catalog.md`。
- `scripts/verify-config-source-ownership.ts`：校验配置源归属。
- `scripts/verify-cordis-config.ts`：校验 Cordis 配置文件合法性。
- `scripts/cordis-config-files.spec.ts`：断言示例工程均含 `cordis.yml`。
- `scripts/cordis-walk.ts`：遍历 Cordis 配置树的工具。

这套体系把“部署期组合”（Cordis profile + patch 栈）与“运行期用户设置”（settings namespaces + file provider）清晰分离，前者决定“装载哪些插件”，后者决定“这些插件如何被用户覆盖”。