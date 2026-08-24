---
kind: external_dependency
name: E2B 远程沙箱运行时（POC）
slug: e2b
category: external_dependency
category_hints:
    - client_constraint
scope:
    - '**'
---

实验性 provider-composition POC，将文件系统与进程执行世界放置在 E2B Linux 沙箱中。E2B 仅提供沙箱生命周期与两个基础 OS 适配器（fs、subprocess），上层能力通过 ctx.fs / ctx.subprocess 抽象消费，无需为 bash、terminal、lsp 等写 E2B 专用分支。当前标记为 POC，尚未作为稳定产品 API 发布。