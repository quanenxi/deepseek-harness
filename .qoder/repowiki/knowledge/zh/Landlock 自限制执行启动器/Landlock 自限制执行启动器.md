---
kind: external_dependency
name: Landlock 自限制执行启动器
slug: landlock-run
category: external_dependency
category_hints:
    - client_constraint
scope:
    - '**'
---

由 harness 仓库维护的原生二进制组件，提供 Landlock 自限制后 exec 的启动器，被 sandbox 能力在 Linux 上用于进程隔离。以三包 npm 家族（entry + linux-arm64 + linux-x64）发布，entry 包通过 optional dependencies 按平台安装对应原生包。开发时由根 workspace 直接 link，CI 中通过 GitHub Workflow 构建并测试各架构，Release 工作流负责打包与发布。