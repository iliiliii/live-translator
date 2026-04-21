# Live Translator 技术设计文档

## 1. 文档目标

本文档基于 [`requirements.md`](D:/code/live-translator/requirements.md#L1) 和现有 MVP 原型的验证结果，重新定义项目的工程底座与实现路线。本文档重点回答以下问题：

- 现阶段应采用什么技术栈继续开发；
- 如何把现有原型迁移到最新的 `WXT + React` 架构；
- 如何同时满足 `Chrome` 与 `Edge` 的上架要求；
- 哪些能力属于本轮迁移范围，哪些能力明确延后。

当前任务不再是从零定义一个纯手写 `Chrome MV3` 扩展，而是把已经验证过主链路的原型迁移到更适合持续开发和发布的工程结构中。

## 2. 项目现状

截至 `2026-04-20`，项目存在两个并行状态：

- 仓库根目录仍以需求文档和旧版实现计划为主，技术路线基于手写 `manifest.json + HTML/CSS/JS`。
- 本地工作树 `D:\code\live-translator\.worktrees\live-translator-mvp` 已经存在一版可运行的 MVP 原型，实现了以下最小闭环：
  `系统音频采集 -> 分片 -> Whisper 转写 -> OpenAI 翻译 -> Popup / Overlay 展示`。

已验证的信息如下：

- 原型为纯 `Chrome MV3` 实现；
- 原型当前只支持 `Whisper + OpenAI`；
- 原型已有单元测试，并在本次检查中通过；
- 原型的 `Popup` 与 `Overlay` 仍是原生 `HTML + CSS + JavaScript`；
- 现有设计文档和实施计划已经落后于目标技术路线。

因此，项目当前阶段应定义为：

**“基于已验证原型，迁移到最新 `WXT + React` 工程底座，并为 `Chrome / Edge` 发布做准备。”**

## 3. 设计结论

本轮设计结论如下：

1. 使用最新 `WXT` 作为扩展工程底座。
   截至本文档更新时间，已核对到的 WXT 官方版本为 `0.20.25`。
2. 使用 `React` 重写 `Popup` 与 `Overlay` 两类界面入口。
3. 使用 `TypeScript` 承载新的入口层、UI 层和共享模块。
4. 保留现有 MVP 原型已经验证过的运行时职责边界：
   `Background / Offscreen / Content / Shared / Provider`。
5. 本轮迁移不做业务扩容，不新增 `DeepL`、说话人分离、服务端代理或高级音频预处理。
6. 交付目标不是“重做一个新产品”，而是“把现有可验证 MVP 迁移到可持续开发、可上架的工程结构”。

结论上，`WXT + React` 路线没有阻塞性问题，可以作为后续实现文档与实际开发的基线。

## 4. 范围定义

### 4.1 本轮包含

1. 用最新 `WXT` 重建扩展工程底座。
2. 使用 `React` 重写 `Popup`。
3. 使用 `React` 重写页面内 `Overlay`。
4. 把 `Background`、`Offscreen`、`Shared` 与 `Provider` 逻辑迁入新工程。
5. 保留现有 MVP 主链路：`Whisper + OpenAI`。
6. 提供同时面向 `Chrome` 与 `Edge` 的构建、打包与手工验收流程。
7. 保留并迁移现有单元测试基础。

### 4.2 本轮不包含

1. `DeepL`、`Google Translate`、`Gemini`、`Claude` 等新增 Provider。
2. 说话人分离（Speaker Diarization）。
3. 高级降噪、音频增强、浏览器侧复杂预处理。
4. 后端代理、密钥托管、统一监控平台。
5. 多会话并发与跨设备同步敏感密钥。
6. 独立的设置页（Options Page）和复杂账户体系。

## 5. 技术方案比较

### 方案 A：`WXT + React + TypeScript`，保留现有业务逻辑边界

特点：

- 用 `WXT` 接管工程、构建与产物生成；
- `Popup` 与 `Overlay` 改为 `React`；
- `Background`、`Offscreen`、`Provider` 逻辑按现有边界迁移；
- 新入口与共享模块采用 `TypeScript`；
- 已验证的音频链路与消息链路尽量不在迁移首轮做重写式改造。

优点：

- 风险最低；
- 能同时满足“换框架”和“保住主链路”；
- 最适合当前阶段。

缺点：

- 迁移期间会存在“新 UI + 旧业务逻辑逐步转移”的过渡态；
- 部分模块会经历先迁入、后整理的两阶段演进。

### 方案 B：`WXT + React + TypeScript` 全量重写

特点：

- 所有模块一次性全面迁移到新的结构和类型系统中；
- 消息协议、Provider、状态管理全部同步重构。

优点：

- 技术面貌统一；
- 代码风格一致。

缺点：

- 迁移风险大；
- 容易把“工程升级”演变为“业务重做”；
- 对当前 MVP 验证节奏不利。

### 方案 C：`WXT + React` + 业务重构 + 能力扩容

特点：

- 在迁移阶段同步增加 Provider、配置页、音频增强等功能。

优点：

- 看起来“一次做完”。

缺点：

- 超出当前项目控制范围；
- 回归成本和排查成本最高；
- 不适合作为首轮迁移方案。

### 结论

采用 **方案 A**。

用户已经明确要求使用最新 `WXT` 和 `React`，因此本轮唯一合理的工程策略是：

**“在 `WXT + React` 下重建入口层与 UI 层，同时尽量保留现有 MVP 原型已经验证过的运行时链路。”**

## 6. 总体架构

### 6.1 架构组件

迁移后的系统由以下部分组成：

- `WXT Config`
- `Background Service Worker`
- `Offscreen Page`
- `Content Script + React Overlay`
- `React Popup`
- `Speech Provider Adapter`
- `Translation Provider Adapter`
- `Shared Contracts / Storage / Logger`

### 6.2 总体数据流

整体数据流保持不变：

`Popup -> Background -> Offscreen -> Speech Provider -> Translation Provider -> Background -> Popup / Overlay`

区别在于：

- `Popup` 将由原生页面改为 `React`；
- `Overlay` 将由原生 DOM 脚本改为 `React` 挂载；
- `manifest.json` 不再作为源码维护，而由 `WXT` 在构建时生成；
- 构建与打包由 `WXT` 管理，而不是手工维护扩展目录。

## 7. WXT 架构决策

### 7.1 配置来源

迁移后不再把根目录 `manifest.json` 作为运行时真实来源。新的事实来源如下：

1. `wxt.config.ts`
2. `entrypoints/`
3. `public/`

这意味着权限、宿主权限、Action、图标、浏览器目标等配置都应进入 `WXT` 配置，而不是继续手写根目录 `manifest.json`。

### 7.2 入口组织

建议使用以下入口：

- `entrypoints/background/index.ts`
- `entrypoints/popup/index.html`
- `entrypoints/popup/main.tsx`
- `entrypoints/content/index.ts`
- `entrypoints/offscreen/index.html`
- `entrypoints/offscreen/main.ts`

其中：

- `background` 负责会话调度和消息路由；
- `popup` 挂载 `React` 应用；
- `content` 负责注入并挂载页面内 `React Overlay`；
- `offscreen` 负责 `getDisplayMedia`、音频采集与分片。

### 7.3 Overlay 挂载方式

`Overlay` 必须采用带隔离边界的挂载方式。

设计要求：

- 使用 `WXT` 的内容脚本 UI 能力；
- 优先使用 `ShadowRoot` 隔离样式；
- 页面样式不得污染扩展的 `Overlay`；
- `Overlay` 只消费标准结果数据，不直接依赖 Provider 响应格式。

### 7.4 Offscreen 页面

`Offscreen` 保持为独立页面，而不是塞进 `Popup` 或 `Content Script` 中。

原因如下：

- `getDisplayMedia()` 与 `MediaRecorder` 更适合在独立上下文中维护生命周期；
- 音频采集与 UI 解耦后，故障边界更清晰；
- 与现有 MVP 原型的职责边界一致，迁移风险更低。

## 8. 目录规划

建议的新目录结构如下：

```text
live-translator/
  docs/
    superpowers/
      plans/
        2026-04-10-live-translator-mvp.md
  entrypoints/
    background/
      index.ts
    content/
      index.ts
    offscreen/
      index.html
      main.ts
    popup/
      index.html
      main.tsx
  public/
    icon-16.png
    icon-32.png
    icon-48.png
    icon-128.png
  src/
    background/
      error-mapper.ts
      message-router.ts
      session-manager.ts
      speech-service.ts
      translation-service.ts
      providers/
        speech/
          base.ts
          index.ts
          whisper.ts
        translation/
          base.ts
          index.ts
          openai.ts
    offscreen/
      audio-capture.ts
      audio-chunker.ts
    shared/
      constants.ts
      logger.ts
      messages.ts
      storage.ts
      types.ts
    ui/
      popup/
        App.tsx
        components/
      overlay/
        OverlayApp.tsx
        components/
      hooks/
        use-session-state.ts
  tests/
    background/
    offscreen/
    shared/
    setup/
  package.json
  tsconfig.json
  wxt.config.ts
```

目录原则如下：

- `entrypoints/` 只负责运行时入口与挂载；
- `src/background/` 只负责业务调度与 Provider 调用；
- `src/offscreen/` 只负责采集与分片；
- `src/shared/` 只负责跨上下文共享的契约、存储与日志；
- `src/ui/` 只负责 React 界面；
- 测试目录按职责拆分，不混入运行时代码目录。

## 9. 组件职责

### 9.1 Background Service Worker

职责：

- 管理翻译会话状态；
- 协调 `Popup`、`Content`、`Offscreen`；
- 调用语音识别与翻译 Provider；
- 推送标准化结果；
- 记录错误与日志。

边界：

- 不直接处理 DOM；
- 不直接承担音频采集实现；
- 不把 UI 状态与业务状态混在一起。

### 9.2 Offscreen Page

职责：

- 调用 `getDisplayMedia()`；
- 管理 `MediaStream` 生命周期；
- 负责音频分片与轻量预处理；
- 向 `Background` 推送标准音频块。

边界：

- 不负责结果渲染；
- 不负责配置表单与展示；
- 不直接请求翻译 Provider。

### 9.3 React Popup

职责：

- 展示 API Key 与基础配置表单；
- 发起开始 / 停止翻译动作；
- 订阅会话状态；
- 展示最近结果与错误信息。

边界：

- 不直接请求外部 AI Provider；
- 不直接持有音频流；
- 所有运行时动作通过消息与存储接口完成。

### 9.4 React Overlay

职责：

- 在页面内展示最近翻译结果；
- 提供最小关闭、拖拽与状态展示能力；
- 与页面样式隔离。

边界：

- 不直接请求 Provider；
- 不维护完整会话状态；
- 不与页面业务 DOM 深度耦合。

### 9.5 Shared Layer

职责：

- 定义消息协议；
- 定义结果结构与配置结构；
- 封装 `storage.sync / session / local`；
- 统一错误日志写入。

边界：

- 不承担 UI 逻辑；
- 不承担浏览器入口生命周期。

## 10. 关键技术判断

### 10.1 `WXT + React` 路线可行

该路线没有发现阻塞性问题，原因如下：

- `WXT` 原生支持 `MV3`、多入口、浏览器目标切换与打包；
- `React` 适合 `Popup` 与 `Overlay` 的状态渲染；
- `Chrome` 与 `Edge` 都属于 Chromium 扩展生态，当前 API 路线兼容性足够高；
- 现有 MVP 原型已经验证了核心业务链路，本轮只是在工程层和 UI 层升级。

### 10.2 系统音频采集仍是第一风险点

无论是否迁移到 `WXT`，系统音频采集仍然是首要风险。

需要继续遵守以下结论：

- 必须优先保证 `Offscreen + getDisplayMedia()` 链路稳定；
- 请求参数必须满足浏览器的实际能力约束；
- 必须把拒绝授权、取消分享、无音频轨等异常视为一等场景。

### 10.3 Overlay 必须做样式隔离

原生内容脚本直插 DOM 时，页面样式与扩展样式容易互相污染。迁移到 `React Overlay` 后，必须优先做样式隔离，否则页面复杂样式会导致悬浮层不可控。

### 10.4 `Chrome / Edge` 发布应共用一套 Chromium 构建

本项目不应拆成两套代码库。正确做法是：

- 共用一套 `WXT` 源码；
- 通过浏览器目标生成不同产物；
- 在打包前核对生成后的权限与图标配置是否符合两个商店要求。

### 10.5 渐进迁移优于一次性重写

当前最有价值的资产不是旧的 UI，而是已经验证过的业务链路。因此：

- 首先迁移工程底座和 UI；
- 然后迁移共享契约与会话逻辑；
- 最后再做结构性整理。

这比“一次性全量重写”更符合当前项目阶段。

## 11. 验证策略

迁移完成后，至少要完成以下验证：

1. 单元测试通过：
   `shared / background / offscreen` 的核心纯逻辑测试保持可运行。
2. 类型检查通过：
   新增的 `TypeScript` 入口、UI 与共享模块无类型错误。
3. 浏览器构建通过：
   能分别生成 `Chrome` 与 `Edge` 产物。
4. 手工烟测通过：
   - 可启动会话；
   - 可授予系统音频权限；
   - 可看到转写与翻译结果；
   - `Popup` 与 `Overlay` 同步更新；
   - 缺少密钥、取消授权等异常路径提示清晰。

## 12. 风险与缓解

### 风险 1：WXT 入口映射错误

表现：

- `Background`、`Content`、`Offscreen` 实际没有被正确打包或注入。

缓解：

- 迁移首轮先只建立最小入口；
- 每完成一个入口就执行构建验证；
- 对照生成后的 `manifest` 检查权限与入口是否正确。

### 风险 2：React Overlay 生命周期不稳定

表现：

- 页面切换后悬浮层重复挂载、丢失或样式错乱。

缓解：

- 内容脚本只负责单一挂载点；
- 使用标准结果流驱动渲染；
- 明确挂载、卸载与隐藏逻辑。

### 风险 3：Offscreen 页面路径或权限错误

表现：

- `chrome.offscreen.createDocument()` 调用失败；
- 运行时找不到 `offscreen` 页面。

缓解：

- 把 `offscreen` 入口作为单独页面维护；
- 在构建后核对实际产物路径；
- 用手工烟测覆盖授权链路。

### 风险 4：迁移时顺手改太多

表现：

- 业务回归难以定位；
- 文档与实现多次偏离。

缓解：

- 本轮只迁移工程底座与 UI；
- Provider 能力与高级功能延后；
- 严格按实施计划分步推进。

## 13. 分阶段实施建议

### 阶段 1：工程底座迁移

- 建立 `WXT + React + TypeScript` 工程；
- 配置 `Chrome / Edge` 构建目标；
- 建立入口与图标、构建脚本、测试脚本。

### 阶段 2：运行时迁移

- 迁移 `shared`、`background`、`offscreen`；
- 保住 `Whisper + OpenAI` 主链路；
- 补齐最小类型与测试。

### 阶段 3：UI 迁移

- 用 `React` 重建 `Popup`；
- 用 `React` 重建 `Overlay`；
- 把状态展示与错误提示迁入新 UI。

### 阶段 4：发布准备

- 输出 `Chrome` 与 `Edge` 构建产物；
- 执行手工烟测；
- 整理上架所需图标、描述与打包步骤。

## 14. 最终结论

原有实现文档中“纯手写 `Chrome MV3` 原生扩展”这一路线，已经不再是当前项目的正确目标状态。

更新后的正确目标状态是：

**以最新 `WXT` 作为工程底座，以 `React` 作为 `Popup / Overlay` 的 UI 层，以现有 MVP 原型验证过的运行时链路作为迁移基础，交付同时面向 `Chrome` 与 `Edge` 的可持续开发版本。**
