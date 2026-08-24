---
kind: frontend_style
name: 基于 CSS 变量与 CSS Modules 的 DSW 设计系统（ui-theme + ui-layout）
category: frontend_style
scope:
    - '**'
source_files:
    - docs/web-styling.md
    - packages/client/ui-theme/src/styles/design-platform.css
    - packages/client/ui-theme/src/styles/base.css
    - packages/client/ui-theme/src/styles/scrollbar.css
    - packages/client/ui-theme/src/styles/gradient-shadow-text.css
    - packages/client/ui-theme/src/styles/shiki.css
    - packages/client/ui-layout/tests/theme-presenter.client.spec.ts
    - packages/client/tsdown.client.ts
    - apps/web/vite.config.ts
    - apps/web/package.json
---

## 1. 采用的体系/工具
- 样式方案：原生 CSS + CSS Modules，显式禁止 Tailwind 与外部组件库。
- 主题系统：`packages/client/ui-theme` 集中维护 `--dsw-*` 静态色板与 `--dsw-alias-*` 语义别名；`packages/client/ui-layout` 负责把主题快照应用到文档根节点（设置 `color-scheme`、`data-ds-dark-theme` 属性、注入 `theme-color` meta 及内联 CSS 变量），并暴露 `ThemePresenter.apply()` / `dispose()` 契约。
- 构建管线：`apps/web/vite.config.ts` 通过 `@vitejs/plugin-react` 编译 React；各 `ui-*` 包由 `packages/client/tsdown.client.ts` 中的 `dsh-css-modules-inline` 插件将 `.module.css` 内联为 JS 对象，从而在浏览器端以 `clsx` 组合类名。
- 字体/动效基线：`ui-theme/src/styles/base.css` 提供 `--dsw-font-family`、`--ds-ease-in-out`、`--ds-transition-duration*` 等基础变量，统一跨平台字体栈（含 CJK 回退）与过渡时长。
- 暗色模式：通过 `body[data-ds-dark-theme]` 覆盖 `--dsw-static-*` 与 `--dsw-alias-*` 两套变量实现 light/dark 切换。

## 2. 关键文件与包
- 设计令牌与全局样式：`packages/client/ui-theme/src/styles/design-platform.css`（静态色板 + 语义别名）、`base.css`（字体/动效基线）、`scrollbar.css`、`gradient-shadow-text.css`、`shiki.css`。
- 主题应用层：`packages/client/ui-layout/src/client/theme-presenter.ts`（测试位于 `tests/theme-presenter.client.spec.ts`），负责写入 `documentElement.style.colorScheme`、`body` 上的 `data-ds-dark-theme`、`meta[name="theme-color"]` 以及 body 内联 `--dsw-alias-*` 变量。
- Web 入口与打包：`apps/web/vite.config.ts`（Vite + React 插件、手动 vendor chunk 拆分、CSS 走 Vite 管线）、`apps/web/package.json`（依赖 `react`、`react-dom` 与 `@deepseek-ai/dsh-client-web` shell）。
- CSS Modules 类型声明：每个 `ui-*` 包根目录的 `css-modules.d.ts` 声明 `declare module '*.module.css'`，配合 `tsdown.client.ts` 的内联插件使用。
- 风格规范文档：`docs/web-styling.md` 明确所有权、组件规则与变更流程。

## 3. 架构与约定
- 所有权分层
  - `ui-theme` 拥有所有 `--dsw-*` 静态刻度、语义别名、排版、动效、渐变、阴影、滚动条样式与明暗偏好。
  - `ui-layout` 负责把“已解析的主题快照”渲染到文档上；功能组件只消费语义别名，不定义新的全局主题。
  - 全局样式放在 `ui-theme/src/styles/`；组件样式以 CSS Modules 形式与组件同目录放置。
- 组件样式约定
  - 使用 CSS Modules + `clsx` 拼接类名；禁止引入 Tailwind 或第三方 UI 组件库。
  - 组件中仅允许使用 `--dsw-alias-*` 语义 token，不得复制静态色值或写死颜色。
  - 组件 CSS 中不得出现主题选择器（如 `[data-ds-dark-theme]`），明暗覆盖由主题层负责。
  - 字体大小需搭配行高，优先复用主题排版变量；源码/终端输出/差异行保持不换行以保留列对齐；滚动条使用共享样式而非组件自定义。
  - 表现逻辑放入 CSS；React inline style 只能传递组件级 custom property，不得编码主题分支。
  - 必须保留键盘焦点可见性与 `prefers-reduced-motion` 行为。
- 运行时主题切换
  - `ThemePresenter.apply(snapshot)` 会替换上一次 `apply` 写入的所有 `--dsw-alias-*` 变量（不是合并），并同步 `color-scheme`、`data-ds-dark-theme` 与 `theme-color` meta；`dispose()` 撤销所有写入但保留其他内联样式。

## 4. 约束与强制规则
- 禁止 Tailwind：`docs/web-styling.md` 明文规定 “do not add a component library or Tailwind”。
- 禁止在组件 CSS 中写主题选择器：规则要求 “Keep theme selectors out of feature component CSS; light/dark overrides belong to the theme owner”，由 `ui-theme` 通过 `body[data-ds-dark-theme]` 覆盖变量实现。
- 禁止在组件中硬编码颜色：规则要求 “Use `--dsw-alias-*` semantic tokens in feature components. Do not copy static palette values or write literal colors there.”
- 主题变更路径：新增/修改共享 token 必须在 `ui-theme` 中完成，再由功能包消费语义别名；公共样式契约变更需更新对应包的引用。
- 视觉回归遵循仓库 testing policy，并通过 Playwright 快照（`apps/web/tests/snapshots/*`）保障样式稳定性。
- 构建侧约束：`apps/web/vite.config.ts` 通过自定义 plugin 拒绝裸 `vite serve`（`rejectStandaloneServe`），确保 Web 壳只能通过 `pnpm dsh web` 启动，避免无 `window.__DSH_BOOT__` 的无效预览。