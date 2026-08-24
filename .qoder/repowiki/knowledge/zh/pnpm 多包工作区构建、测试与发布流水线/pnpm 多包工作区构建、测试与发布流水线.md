---
kind: build_system
name: pnpm 多包工作区构建、测试与发布流水线
category: build_system
scope:
    - '**'
source_files:
    - package.json
    - pnpm-workspace.yaml
    - tsconfig.base.json
    - vitest.config.ts
    - .github/workflows/ci.yml
    - .github/workflows/release.yml
    - .github/workflows/python-release.yml
    - .github/workflows/landlock-run-release.yml
    - .github/workflows/build-exe-for-python-sdk.yml
    - scripts/run-gates.ts
    - apps/cli/package.json
    - apps/web/vite.config.ts
    - tsdown.config.ts
    - native/landlock-run/package.json
    - native/landlock-run/scripts/build.ts
    - native/landlock-run/scripts/pack-release.mjs
    - python/sdk/pyproject.toml
    - python/sdk-runtime/hatch_build.py
    - python/sdk-runtime/platforms.json
    - website/package.json
---

## 1. 使用的系统/工具

- **包管理器与工作区**: pnpm 11 (`packageManager` 字段锁定版本)，通过 `pnpm-workspace.yaml` 声明 `vendor/*`、`packages/*/*`、`native/landlock-run`、`apps/*`、`website`、`examples`、`python/sdk-runtime` 等成员，并使用 `linkWorkspacePackages: true` 将本地同名包解析到源码。
- **TypeScript 编译**: TypeScript 6（`tsconfig.base.json` 作为全局路径别名与严格选项基座），通过 `tsc -b` 的 Project References 增量编译；宿主端由 `tsconfig.host.json` 聚合，客户端由 `tsconfig.client.json` 聚合。产物由 `tsdown` 打包为单文件 CLI（`tsdown --env.DSH_BUILD_FACE host|client`）。
- **Web 前端构建**: `apps/web` 使用 Vite + `vite.config.ts`，通过根脚本 `pnpm --filter @deepseek-ai/dsh-web-frontend run build` 触发。
- **原生子项目**: `native/landlock-run` 独立维护 C/C++ 静态二进制与 npm 平台包，通过自身 `scripts/build.ts`、`scripts/pack-release.mjs` 等脚本完成 native 构建与发布。
- **Python SDK**: `python/sdk` 使用 Hatchling (`pyproject.toml`) 构建 wheel，依赖 `deepseek-harness-runtime-bin`（来自 `python/sdk-runtime`，通过 uv editable source 注入）。
- **测试框架**: Vitest 4（`vitest.config.ts` 及多个专用配置文件：`vitest.e2e.config.ts`、`vitest.snapshot.config.ts`、`vitest.web.config.ts`、`vitest.web-stress.config.ts`、`vitest.web.perf.config.ts`），覆盖率基于 v8 provider，强制每文件 100% 语句/分支/函数/行覆盖。
- **质量门禁编排器**: `scripts/run-gates.ts` 集中定义 `ci-primary`、`ci-static`、`ci-coverage`、`ci-snapshot`、`ci-consumers`、`ci-windows-blocking`、`ci-windows-complete` 等模式，以受控并发调度各 gate 命令。
- **CI**: GitHub Actions（`.github/workflows/ci.yml` 为主入口，另有 `release.yml`、`release-vendor.yml`、`python-release.yml`、`landlock-run-release.yml`、`build-exe-for-python-sdk.yml`、`e2e.yml`、`docs-pages.yml`、`sandbox.yml`、`pi-ai-provider-e2e.yml`、`expected-filenames.yml`、`issue-lifecycle.yml`、`issue-policy.yml`）。
- **Lint/检查**: oxlint（`.oxlintrc.json`）、knip（`knip.json`）、publint（`scripts/publint-all.ts`）、jscpd（`.jscpd.json`）。
- **预提交钩子**: lefthook（`lefthook.yml`，通过 `postinstall` 安装）。

## 2. 关键文件

- 顶层编排: `package.json`（所有脚本入口）、`pnpm-workspace.yaml`（工作区与依赖策略）、`tsconfig.base.json`（路径映射与严格选项）、`vitest.config.ts`（测试/覆盖率配置）
- CI: `.github/workflows/ci.yml`（主 CI）、`.github/workflows/release.yml`、`.github/workflows/python-release.yml`、`.github/workflows/landlock-run-release.yml`
- 网关编排: `scripts/run-gates.ts`（gate 模式与并发控制）
- 应用构建: `apps/cli/package.json`（`dsh` bin 指向 `lib/bin.js`）、`apps/web/vite.config.ts`、`tsdown.config.ts`
- 原生构建: `native/landlock-run/scripts/build.ts`、`native/landlock-run/scripts/pack-release.mjs`、`native/landlock-run/package.json`
- Python: `python/sdk/pyproject.toml`、`python/sdk-runtime/hatch_build.py`、`python/sdk-runtime/platforms.json`
- 文档站点: `website/package.json`（VitePress）

## 3. 架构与约定

### 构建流程
- `pnpm install --frozen-lockfile` 是全部 CI 与本地安装的统一入口，禁止锁外变更。
- 根 `build` 先执行 `build:lib`（`tsc -b tsconfig.host.json && tsdown --env.DSH_BUILD_FACE host`，再对 client 做同样操作），再执行 `build:web`（Vite 构建 `@deepseek-ai/dsh-web-frontend`）。
- 宿主/客户端分别通过独立的 `tsconfig.host.json` / `tsconfig.client.json` 聚合 `packages/*/*/src`，并通过 `tsconfig.base.json` 中的 `paths` 把 `@deepseek-ai/dsh-*` 包名直接映射到源码目录，使测试/开发时始终走源码而非已编译产物。
- 每个 package 的 `tsconfig.json` 仅声明自身源与 project references，不共享 include，保持编译边界清晰。

### 测试与覆盖率
- 默认 `test` 运行两个 Vitest project：`thread-safe`（fork pool，排除进程绑定测试）和 `process-bound`（单独 fork 隔离进程级状态）。
- Windows 上自动排除 bash/sandbox/hooks/terminal-bash 等 POSIX 专属套件；非 Windows 上排除 `packages/sandbox/sandbox-windows-acl` 等 Windows-only 源码。
- 覆盖率阈值：`perFile: true`，statements/branches/functions/lines 均为 100%，缺失位置由自定义 reporter 输出精确 `path:line:col`。
- 快照测试通过 `DSH_SNAPSHOT=record|refresh` 环境变量切换录制/回放模式。

### 质量门禁（gates）
- `check:all` 调用 `run-gates.ts check-all`，聚合 lint、typecheck、snapshot、artifact、consumer、coverage 等 gate。
- CI 拆分为三个企业级 job：`node-24`（静态分析）、`node-24-coverage`（覆盖率）、`node-24-consumers`（Linux 唯一构建+快照+artifacts），各自独立分配 runner。
- 通过 `DSH_GATE_CONCURRENCY`、`DSH_COVERAGE_MAX_WORKERS`、`DSH_SNAPSHOT_MAX_CONCURRENCY`、`DSH_OXLINT_THREADS`、`DSH_PUBLINT_CONCURRENCY` 等环境变量在 CI 中调节并发。
- 支持 failover：仓库变量 `DSH_CI_FAILOVER_LINUX`/`DSH_CI_FAILOVER_WINDOWS` 可将 Linux/Windows 作业重定向到自托管池（`vm-backup`、`dsh-win-ci`）。

### 多平台与原生
- Node 兼容性矩阵：CI 同时用 Node 22.19、PRIMARY_NODE_VERSION（24）、以及 Node 26 跑兼容 smoke。
- Windows 通过 Wine 在 Linux 上执行 `scripts/wine-windows-gates.sh` 进行阻塞性验证；另有一个真实 Windows runner 的 `windows-native` job 跑完整 native 套件。
- 原生沙箱 `native/landlock-run` 独立于主工作区，有自己的 `gha:matrix`、`release:*` 脚本，通过 `platforms.json` 描述目标平台。
- Python SDK 通过 uv 管理，wheel 依赖 `deepseek-harness-runtime-bin==0.0.0.dev0`，该二进制由 `build-exe-for-python-sdk.yml` 构建并注入。

### 发布流程
- 根脚本提供 `release:dsh`、`release:vendor`、`release:verify`、`release:pack`、`release:publish`，由 `scripts/release/bump.ts`、`scripts/release/publish.ts` 等驱动。
- Vendor 包通过 `rescope-vendor` 脚本将上游 semver 范围重写为 workspace link，确保本地构建使用源码。
- Python 发布由 `python-release.yml` 触发，先构建 exe 再打 wheel。

## 4. 约定与约束

- **依赖安装必须冻结**：所有 CI 步骤均使用 `pnpm install --frozen-lockfile`，禁止锁文件漂移。
- **Node 引擎要求**：根 `package.json` 声明 `engines.node: ^22.19.0 || >=24.0.0`，CI PRIMARY_NODE_VERSION 固定为 24。
- **pnpm strictDepBuilds 白名单**：`pnpm-workspace.yaml` 的 `allowBuilds` 显式允许 `esbuild`、`lefthook`、`node-pty`、`koffi`、`@deepseek-ai/dsh-subprocess-local` 等带生命周期脚本的包，其余一律拒绝（deny by default）。
- **覆盖率 100% 硬性门槛**：`vitest.config.ts` 中 per-file 100% 阈值意味着任何新增未覆盖代码都会导致合并失败；v8 ignore 注释必须附带原因说明。
- **工作区成员即构建目标**：`pnpm-workspace.yaml` 明确列出 `vendor/*`、`packages/*/*`、`apps/*`、`website`、`native/landlock-run`、`python/sdk-runtime` 为成员；`examples` 仅用于依赖解析（注释说明其不参与 tsdown 构建）。
- **路径映射契约**：`tsconfig.base.json` 的 `paths` 将 `@deepseek-ai/dsh-*` 通配映射到 `packages/*/src`，新增包无需修改此映射即可被解析——但需遵循命名约定。
- **CI 必需检查聚合**：`all-checks-passed` job 显式 `needs` 列表 `[node-24, node-24-coverage, node-24-consumers, node-compat, python-sdk, python-runtime, windows]`，新增 blocking job 必须加入此处。
- **Playwright 缓存键**：Chromium 与 pnpm store 均以 `pnpm-lock.yaml` 哈希作为 cache key，保证依赖/浏览器一致性。
- **bubblewrap 沙箱准备**：覆盖率与消费者 job 并行执行 `pnpm install` 与 `scripts/prepare-ci-bubblewrap.sh`，确保容器化环境可用。
- **Python SDK 可编辑安装**：`pyproject.toml` 中 `[tool.uv.sources]` 将 `deepseek-harness-runtime-bin` 指向 `../sdk-runtime` 的 editable 路径，使本地开发无需重新打包 wheel。
- **Lefthook 预提交**：通过 `postinstall` 自动安装 lefthook，本地提交前执行 staged 规则（`.oxlintrc.staged.json`）。
- **版本策略**：根与 `apps/cli` 使用 `-rc.x` 预发布版本号；`native/landlock-run` 独立维护 `0.1.1` 版本；Python SDK 使用 `0.0.0.dev0` 占位版本，由 release 流程实际填充。