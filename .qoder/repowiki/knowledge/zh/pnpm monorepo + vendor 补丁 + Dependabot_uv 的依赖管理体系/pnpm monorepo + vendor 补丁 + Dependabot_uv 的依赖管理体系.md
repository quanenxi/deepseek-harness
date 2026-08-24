---
kind: dependency_management
name: pnpm monorepo + vendor 补丁 + Dependabot/uv 的依赖管理体系
category: dependency_management
scope:
    - '**'
source_files:
    - package.json
    - pnpm-workspace.yaml
    - pnpm-lock.yaml
    - .github/dependabot.yml
    - patches/node-pty@1.1.0.patch
    - python/sdk/pyproject.toml
    - python/sdk/uv.lock
    - scripts/rescope-vendor.ts
    - scripts/check-vendor-manifest.sh
    - scripts/verify-vendored-links.ts
    - scripts/verify-dsh-package-licenses.ts
    - apps/cli/package.json
---

## 1. 使用的系统与方法

仓库采用 **pnpm workspace**（pnpm@11.7.0，通过根 `package.json` 的 `packageManager` 字段锁定）作为多包工作区与依赖解析核心。Node.js 工程分布在 `apps/*`、`packages/*/*`、`native/landlock-run`、`website`、`examples`、`python/sdk-runtime` 等目录；Python SDK 使用 **uv**（`pyproject.toml` + `uv.lock`）管理。

- 工作区成员由根 `pnpm-workspace.yaml` 显式声明，并通过 `linkWorkspacePackages: true` 强制所有 workspace 包以符号链接方式解析。
- 通过 `overrides` 将 `@deepseek-ai/cosmokit`、`@deepseek-ai/schemastery` 重定向到本地 `vendor/` 下的源码副本，实现“框架级”依赖的内嵌化。
- 通过 `allowBuilds` 白名单机制（pnpm 10+ 默认严格模式）禁止第三方包在 install 时执行未审查的构建脚本，仅允许 `esbuild`、`lefthook`、`node-pty`、`koffi` 及一个本地 `file:` 包。
- 通过 `patchedDependencies` 对 `node-pty@1.1.0` 应用 `patches/node-pty@1.1.0.patch`。
- Python 侧：`python/sdk/pyproject.toml` 声明 `deepseek-harness-runtime-bin==0.0.0.dev0` 为运行时依赖，并通过 `[tool.uv.sources]` 将其指向 `../sdk-runtime` 的 editable 安装。

## 2. 关键文件

| 文件 | 作用 |
|---|---|
| `package.json`（根） | 声明 pnpm 版本、workspace 成员、顶层脚本（`hygiene`、`release:*`、`verify-*`）、`devDependencies` |
| `pnpm-workspace.yaml` | 定义 workspace 成员、`overrides`、`peerDependencyRules`、`allowBuilds`、`minimumReleaseAgeExclude`、`patchedDependencies` |
| `.github/dependabot.yml` | 配置 npm、uv、GitHub Actions 三类生态的自动更新 PR（cron 每日凌晨，30 天冷却期） |
| `patches/node-pty@1.1.0.patch` | 针对 node-pty 的官方补丁 |
| `python/sdk/pyproject.toml` | Python SDK 依赖声明与 uv editable 源映射 |
| `python/sdk/uv.lock` | Python 依赖锁定文件 |
| `pnpm-lock.yaml` | Node 依赖锁定文件 |
| `scripts/rescope-vendor.ts` / `scripts/check-vendor-manifest.sh` | 校验 vendor 包重定向与清单一致性 |
| `scripts/verify-vendored-links.ts` | 校验 vendor 包的 link 关系 |
| `scripts/verify-dsh-package-licenses.ts` | 校验各 dsh 包的许可证合规性 |

## 3. 架构与约定

### 工作区分层
- **产品装配层**：`apps/cli`、`apps/web`、`website` 消费 `packages/*/*` 中的能力包。
- **能力包层**：`packages/*/*` 下每个子目录是一个独立发布的 npm 包（如 `@deepseek-ai/dsh-core`），彼此之间通过 `workspace:^` 引用，保证发布时版本号解耦但开发期零拷贝。
- **原生层**：`native/landlock-run` 是独立的 pnpm workspace，包含 C 静态二进制与平台打包脚本，被上层通过 npm 包形式消费。
- **示例层**：`examples` 整体作为一个 workspace 成员，其 `package.json` 将所有叶子插件声明为 `workspace:*`，以便在纯 Node 环境下直接通过真实 package exports 启动。
- **Python 层**：`python/sdk` 与 `python/sdk-runtime` 通过 uv 的 editable source 关联，SDK 依赖固定版本的 runtime bin。

### Vendor 策略
- 通过 `pnpm-workspace.yaml` 的 `overrides` 把 `@deepseek-ai/cosmokit`、`@deepseek-ai/schemastery` 指向 `link:vendor/<pkg>`，使这些框架级依赖脱离 npm registry 的版本漂移。
- `pnpm-workspace.yaml` 注释说明 vendored framework packages 保留上游 semver range，但本地构建必须解析到 workspace pinned 源。
- 配套脚本 `rescope-vendor`、`check-vendor-manifest`、`verify-vendored-links` 用于校验 vendor 包的重定向与链接正确性。

### 构建脚本安全
- `allowBuilds` 采取“默认拒绝”策略，任何第三方包若带有 install/build 生命周期脚本必须在列表中显式声明；`@google/genai`、`protobufjs`、`node-addon-require-builtin` 等被明确标记为 `false`（pnpm 仍会安装但不执行脚本）。
- `minimumReleaseAgeExclude` 对特定新发布包豁免冷却期，避免阻塞必要的模型目录更新。

### 版本管理与更新
- 根 `package.json` 通过 `engines.node` 锁定 Node 版本范围（`^22.19.0 || >=24.0.0`）。
- 根 `package.json` 的 `version` 与 `apps/cli/package.json` 的 `version` 同步为 `0.1.0-rc.5`，配合 `scripts/release/bump.ts`、`publish-npm-baseline.ts`、`release:pack`、`release:publish` 等脚本统一发布流程。
- GitHub Dependabot 每天 UTC+8 凌晨扫描 npm、uv、GitHub Actions，生成带 `kind/dependency`、`area/infra` 标签的 PR，并设置 30 天冷却期。

## 4. 约定与约束

- **内部包引用一律使用 `workspace:^`**：所有 `packages/*/*` 之间的依赖都以 workspace 协议声明，确保开发期符号链接、发布期按语义化版本解析。
- **vendor 包不通过 registry 升级**：`vendor/**` 路径被 Dependabot 排除，其版本维护遵循 vendor README 而非自动化 PR。
- **禁止未审查的构建脚本**：pnpm 10+ 的 `allowBuilds` 白名单是硬性门禁，新增依赖若带 build/install 脚本必须在此登记。
- **TypeScript 版本收敛**：`peerDependencyRules.allowedVersions.typescript` 限制为 `>=5 <7`，防止工作区内出现多版本 TS 冲突。
- **Python 依赖锁定**：`python/sdk` 使用 uv 的 `uv.lock` 锁定依赖，并通过 `[tool.uv.sources]` 将 `deepseek-harness-runtime-bin` 指向本地 `sdk-runtime` 的 editable 安装。
- **补丁集中管理**：所有 npm 补丁放在 `patches/` 目录，通过 `pnpm-workspace.yaml` 的 `patchedDependencies` 声明生效。
- **许可证合规检查**：根脚本 `verify-dsh-package-licenses` 在 CI/hygiene 中运行，确保各 dsh 包的许可证符合项目要求。
- **Hygiene 门禁**：根 `package.json` 的 `hygiene` 脚本串联了 `rescope-vendor:check`、`knip`、`publint`、`constraints`、`verify-dsh-package-licenses`、`verify-package-invariants`、`verify-built-package-invariants`、`verify-cordis-config`、`verify-node-next-types`、`verify-runtime-closure`、`verify-vendored-links` 等依赖相关校验，作为提交前/CI 的统一入口。

## 5. 适用性判断

本仓库存在完整的依赖管理系统：pnpm workspace + lockfile + vendor 内嵌 + patches + Dependabot + uv + 大量 verify 脚本，属于高度成熟的 monorepo 依赖治理方案，因此本类别完全适用。