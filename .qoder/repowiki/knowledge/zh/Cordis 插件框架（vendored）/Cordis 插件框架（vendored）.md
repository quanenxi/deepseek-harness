---
kind: external_dependency
name: Cordis 插件框架（vendored）
slug: cordis
category: external_dependency
category_hints:
    - framework_behavior
scope:
    - '**'
---

DeepSeek Harness 的核心插件运行时，以 vendored 形式拷贝到 `vendor/` 目录并由根 pnpm workspace 通过 `link:` 解析。包含 cordis core、plugin-loader、plugin-include、plugin-group、plugin-timer、plugin-hmr、plugin-logger-console，以及底层依赖 cosmokit 与 schemastery。插件通过实现 Service、使用 ctx.inject / ctx.effect / ctx.on / ctx.waterfall 声明依赖与生命周期；事件分为 emit、waterfall、parallel、serial 四种分发模式。更新需遵循 vendor/README.md 的同步流程：从上游 fork 拷贝源码、重放本地修改、更新版本与 commit hash，再执行 `pnpm install && pnpm run test && pnpm run build`。