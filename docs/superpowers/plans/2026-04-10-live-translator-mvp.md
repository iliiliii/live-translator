# Live Translator WXT + React Migration Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 将现有 `Whisper + OpenAI` MVP 原型迁移到最新 `WXT + React` 工程结构，并产出可同时面向 `Chrome` 与 `Edge` 的 `MV3` 扩展版本。

**Architecture:** 采用 `WXT` 作为扩展工程底座，使用 `React` 重写 `Popup` 与 `Overlay`，同时保留 `Background / Offscreen / Provider / Shared` 的既有职责边界。迁移优先保证主链路不回退，再逐步完成类型化、构建验证和打包发布准备。

**Tech Stack:** `WXT 0.20.25`、`React`、`TypeScript`、`Vitest`、`jsdom`、`Chrome MV3`、`Offscreen API`、`MediaRecorder`、`Fetch API`

---

## File Structure

- `package.json`
  负责 `WXT`、`React`、测试、构建与打包脚本。

- `tsconfig.json`
  负责 `TypeScript` 编译与路径基线。

- `wxt.config.ts`
  负责 `manifest` 生成、权限、宿主权限、图标、浏览器目标和模块注册。

- `entrypoints/background/index.ts`
  负责声明 `Background Service Worker` 入口并挂载消息路由。

- `entrypoints/popup/index.html`
  负责提供 `Popup` 页面载体。

- `entrypoints/popup/main.tsx`
  负责挂载 `React Popup` 根组件。

- `entrypoints/content/index.ts`
  负责注入并挂载 `React Overlay`。

- `entrypoints/offscreen/index.html`
  负责提供 `Offscreen` 页面载体。

- `entrypoints/offscreen/main.ts`
  负责初始化音频采集控制器和消息监听。

- `src/shared/*.ts`
  负责消息协议、配置结构、存储封装、日志与类型约定。

- `src/background/*.ts`
  负责会话状态、消息路由、Provider 适配与翻译链路。

- `src/offscreen/*.ts`
  负责系统音频采集、分片与轻量预处理。

- `src/ui/popup/*.tsx`
  负责 `Popup` 的 React 组件与状态渲染。

- `src/ui/overlay/*.tsx`
  负责 `Overlay` 的 React 组件与结果渲染。

- `tests/shared/*.test.ts`
  负责共享契约与存储工具测试。

- `tests/background/*.test.ts`
  负责会话状态、Provider 调度与翻译去重测试。

- `tests/offscreen/*.test.ts`
  负责音频分片工具等纯函数测试。

- `tests/ui/*.test.ts`
  负责 `Popup` 与 `Overlay` 的最小渲染和视图模型测试。

- `docs/manual-smoke-checklist.md`
  负责迁移后 `Chrome / Edge` 端到端手工验收。

## Task 1: 建立 WXT + React + TypeScript 工程底座

**Files:**
- Create: `package.json`
- Create: `tsconfig.json`
- Create: `wxt.config.ts`
- Create: `entrypoints/background/index.ts`
- Create: `entrypoints/popup/index.html`
- Create: `entrypoints/popup/main.tsx`
- Create: `entrypoints/content/index.ts`
- Create: `entrypoints/offscreen/index.html`
- Create: `entrypoints/offscreen/main.ts`
- Create: `public/icon-16.png`
- Create: `public/icon-32.png`
- Create: `public/icon-48.png`
- Create: `public/icon-128.png`

- [ ] **Step 1: 初始化依赖与脚本**

要求：

- 添加 `wxt`、`react`、`react-dom`、`typescript`；
- 保留 `vitest` 与 `jsdom`；
- 添加 `dev`、`build:chrome`、`build:edge`、`zip:chrome`、`zip:edge`、`test`、`typecheck` 脚本。

- [ ] **Step 2: 写 `WXT` 最小配置**

要求：

- 在 `wxt.config.ts` 中声明 `MV3`；
- 配置 `permissions`：`activeTab`、`tabs`、`storage`、`scripting`、`offscreen`；
- 配置 `host_permissions`：`<all_urls>`；
- 接入 `React` 模块；
- 配置图标与扩展基础名称。

- [ ] **Step 3: 建立最小入口壳**

要求：

- `background` 能成功加载；
- `popup` 能渲染一个最小的 React 根节点；
- `content` 能注入一个最小挂载点；
- `offscreen` 页面能被构建产出。

- [ ] **Step 4: 运行构建验证**

Run:

```bash
npm install
npm run build:chrome
npm run build:edge
```

Expected:

- 两个构建命令都成功；
- 生成 `Chrome` 与 `Edge` 产物；
- 生成后的 `manifest` 权限与设计文档一致。

- [ ] **Step 5: 提交**

```bash
git add package.json tsconfig.json wxt.config.ts entrypoints public
git commit -m "chore: bootstrap wxt react extension shell"
```

## Task 2: 迁移共享契约、存储与日志工具

**Files:**
- Create: `src/shared/constants.ts`
- Create: `src/shared/messages.ts`
- Create: `src/shared/storage.ts`
- Create: `src/shared/logger.ts`
- Create: `src/shared/types.ts`
- Create: `tests/setup/chrome-stub.ts`
- Create: `tests/shared/messages.test.ts`
- Create: `tests/shared/storage.test.ts`
- Create: `tests/shared/logger.test.ts`

- [ ] **Step 1: 先迁移共享层失败测试**

要求：

- 将现有 `messages`、`storage`、`logger` 测试迁移到 `TypeScript`；
- 先让测试因模块不存在而失败。

- [ ] **Step 2: 运行失败测试**

Run:

```bash
npm run test -- tests/shared/messages.test.ts tests/shared/storage.test.ts tests/shared/logger.test.ts
```

Expected:

- 失败原因指向缺失的 `src/shared/*` 模块，而不是测试环境错误。

- [ ] **Step 3: 实现共享层**

要求：

- 常量、消息协议、配置默认值、会话密钥、错误日志结构全部迁入 `src/shared/`；
- 继续使用 `storage.sync / session / local` 分层封装；
- 保留 `schemaVersion`、`overlayEnabled`、`chunkDurationMs` 等字段；
- 字段命名保持与现有 MVP 一致。

- [ ] **Step 4: 运行共享层测试**

Run:

```bash
npm run test -- tests/shared/messages.test.ts tests/shared/storage.test.ts tests/shared/logger.test.ts
```

Expected:

- 共享层测试全部通过。

- [ ] **Step 5: 提交**

```bash
git add src/shared tests/setup tests/shared
git commit -m "feat: migrate shared contracts to typescript"
```

## Task 3: 迁移 Background 会话与 Provider 链路

**Files:**
- Create: `src/background/session-manager.ts`
- Create: `src/background/message-router.ts`
- Create: `src/background/speech-service.ts`
- Create: `src/background/translation-service.ts`
- Create: `src/background/error-mapper.ts`
- Create: `src/background/providers/speech/base.ts`
- Create: `src/background/providers/speech/index.ts`
- Create: `src/background/providers/speech/whisper.ts`
- Create: `src/background/providers/translation/base.ts`
- Create: `src/background/providers/translation/index.ts`
- Create: `src/background/providers/translation/openai.ts`
- Modify: `entrypoints/background/index.ts`
- Create: `tests/background/session-manager.test.ts`
- Create: `tests/background/translation-service.test.ts`
- Create: `tests/background/whisper-provider.test.ts`
- Create: `tests/background/error-mapper.test.ts`

- [ ] **Step 1: 迁移会话与 Provider 测试**

要求：

- 先迁入 `session-manager`、`translation-service`、`whisper-provider`、`error-mapper` 测试；
- 保持测试覆盖当前 MVP 的行为，不扩容功能。

- [ ] **Step 2: 运行失败测试**

Run:

```bash
npm run test -- tests/background/session-manager.test.ts tests/background/translation-service.test.ts tests/background/whisper-provider.test.ts tests/background/error-mapper.test.ts
```

Expected:

- 失败原因是背景模块尚未实现。

- [ ] **Step 3: 实现 Background 运行时**

要求：

- 迁移现有会话状态机、消息路由和 Provider 注册表；
- 背景入口只做装配，不堆业务细节；
- 继续只支持 `whisper` 与 `openai` 两个 Provider；
- 结果结构保持 `transcript / translation / sourceLanguage / targetLanguage / speakerLabel`。

- [ ] **Step 4: 运行背景测试**

Run:

```bash
npm run test -- tests/background/session-manager.test.ts tests/background/translation-service.test.ts tests/background/whisper-provider.test.ts tests/background/error-mapper.test.ts
```

Expected:

- 背景模块测试全部通过。

- [ ] **Step 5: 提交**

```bash
git add src/background entrypoints/background tests/background
git commit -m "feat: migrate background runtime to wxt"
```

## Task 4: 迁移 Offscreen 音频采集链路

**Files:**
- Create: `src/offscreen/audio-capture.ts`
- Create: `src/offscreen/audio-chunker.ts`
- Modify: `entrypoints/offscreen/main.ts`
- Create: `tests/offscreen/audio-chunker.test.ts`

- [ ] **Step 1: 迁移分片测试**

要求：

- 先迁入 `audio-chunker` 测试；
- 保留现有静音判断与分片元数据行为。

- [ ] **Step 2: 运行失败测试**

Run:

```bash
npm run test -- tests/offscreen/audio-chunker.test.ts
```

Expected:

- 因 `src/offscreen/audio-chunker.ts` 不存在或未实现而失败。

- [ ] **Step 3: 实现 Offscreen 运行时**

要求：

- 保留 `getDisplayMedia()`、`MediaRecorder`、停止采集与错误上报逻辑；
- `Background` 通过 `offscreen` 页面路径创建文档；
- `Offscreen` 只处理采集与分片，不承担业务渲染。

- [ ] **Step 4: 运行 Offscreen 测试**

Run:

```bash
npm run test -- tests/offscreen/audio-chunker.test.ts
```

Expected:

- 分片测试通过；
- 本地构建后 `offscreen` 页面可在产物中找到。

- [ ] **Step 5: 提交**

```bash
git add src/offscreen entrypoints/offscreen tests/offscreen
git commit -m "feat: migrate offscreen audio capture pipeline"
```

## Task 5: 用 React 重建 Popup

**Files:**
- Create: `src/ui/popup/App.tsx`
- Create: `src/ui/popup/components/ConfigForm.tsx`
- Create: `src/ui/popup/components/StatusPanel.tsx`
- Create: `src/ui/popup/components/ResultsList.tsx`
- Create: `src/ui/hooks/use-session-state.ts`
- Modify: `entrypoints/popup/main.tsx`
- Create: `tests/ui/popup.test.ts`

- [ ] **Step 1: 写 Popup 组件测试或最小渲染测试**

要求：

- 至少验证状态展示、错误展示和结果列表渲染；
- 不要求在本轮引入复杂状态管理库。

- [ ] **Step 2: 运行失败测试**

Run:

```bash
npm run test -- tests/ui/popup.test.ts
```

Expected:

- 测试因 `Popup` 组件尚未实现而失败。

- [ ] **Step 3: 实现 React Popup**

要求：

- 保留开始 / 停止按钮；
- 保留 `Whisper Key` 与 `OpenAI Key` 输入；
- 保留错误信息和最近结果展示；
- 所有运行时动作仍通过消息发送给 `Background`。

- [ ] **Step 4: 运行 Popup 测试与类型检查**

Run:

```bash
npm run test -- tests/ui/popup.test.ts
npm run typecheck
```

Expected:

- 组件测试通过；
- `TypeScript` 类型检查通过。

- [ ] **Step 5: 提交**

```bash
git add src/ui/popup src/ui/hooks entrypoints/popup tests/ui/popup.test.ts
git commit -m "feat: rebuild popup with react"
```

## Task 6: 用 React 重建 Overlay

**Files:**
- Create: `src/ui/overlay/OverlayApp.tsx`
- Create: `src/ui/overlay/components/OverlayResults.tsx`
- Create: `src/ui/overlay/components/OverlayFrame.tsx`
- Modify: `entrypoints/content/index.ts`
- Create: `tests/ui/overlay.test.ts`

- [ ] **Step 1: 先补 Overlay 视图模型测试**

要求：

- 至少验证结果列表格式化与空状态表现；
- 保持和 `Popup` 相同的结果结构。

- [ ] **Step 2: 运行失败测试**

Run:

```bash
npm run test -- tests/ui/overlay.test.ts
```

Expected:

- 测试因 `Overlay` 组件尚未实现而失败。

- [ ] **Step 3: 实现 React Overlay**

要求：

- 使用 `WXT` 内容脚本 UI 能力挂载；
- 使用 `ShadowRoot` 做样式隔离；
- 保留显示、隐藏和结果更新能力；
- 首轮不强求拖拽，优先保住展示与同步链路。

- [ ] **Step 4: 运行 Overlay 测试与构建**

Run:

```bash
npm run test -- tests/ui/overlay.test.ts
npm run build:chrome
```

Expected:

- `Overlay` 视图测试通过；
- `Chrome` 构建通过，内容脚本入口正常生成。

- [ ] **Step 5: 提交**

```bash
git add src/ui/overlay entrypoints/content tests/ui/overlay.test.ts
git commit -m "feat: rebuild overlay with react"
```

## Task 7: 集成联调、浏览器构建与文档收尾

**Files:**
- Create: `docs/manual-smoke-checklist.md`
- Modify: `design.md`
- Modify: `docs/superpowers/plans/2026-04-10-live-translator-mvp.md`

- [ ] **Step 1: 运行完整测试与类型检查**

Run:

```bash
npm run test
npm run typecheck
```

Expected:

- 单元测试全部通过；
- 类型检查无错误。

- [ ] **Step 2: 运行双浏览器构建与打包**

Run:

```bash
npm run build:chrome
npm run build:edge
npm run zip:chrome
npm run zip:edge
```

Expected:

- 两个浏览器的构建与打包都成功；
- 生成可供商店上传的压缩包。

- [ ] **Step 3: 执行手工烟测**

按 `docs/manual-smoke-checklist.md` 验证以下场景：

1. 输入 `Whisper` 与 `OpenAI` 密钥后可正常启动；
2. 授权系统音频后可看到翻译结果；
3. `Popup` 与 `Overlay` 同步更新；
4. 缺少密钥、取消授权、关闭 `Overlay` 时提示清晰；
5. 在 `Chrome` 与 `Edge` 中都能完成最小闭环。

- [ ] **Step 4: 提交**

```bash
git add docs package.json tsconfig.json wxt.config.ts entrypoints src tests
git commit -m "chore: finish wxt react migration for chrome and edge"
```

## Self-Review Checklist

- 覆盖性：
  - `WXT` 工程底座：Task 1
  - 共享契约与存储：Task 2
  - `Background` 主链路：Task 3
  - `Offscreen` 采集：Task 4
  - `Popup React`：Task 5
  - `Overlay React`：Task 6
  - `Chrome / Edge` 构建与验收：Task 7

- 占位检查：
  - 计划中未保留未决占位词；
  - 每个任务都给出明确文件范围和验证命令。

- 一致性检查：
  - 语音 Provider 统一为 `whisper`；
  - 翻译 Provider 统一为 `openai`；
  - 消息协议统一由 `src/shared/messages.ts` 提供；
  - 会话状态统一由 `src/background/session-manager.ts` 管理；
  - UI 统一由 `React` 承担，`Background` 与 `Offscreen` 不直接处理 DOM。

## Execution Handoff

Plan complete and saved to `docs/superpowers/plans/2026-04-10-live-translator-mvp.md`. Two execution options:

**1. Subagent-Driven (recommended)** - I dispatch a fresh subagent per task, review between tasks, fast iteration

**2. Inline Execution** - Execute tasks in this session using executing-plans, batch execution with checkpoints

Which approach?
