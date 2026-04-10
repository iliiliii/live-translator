# Live Translator 技术设计文档

## 1. 文档目标

本文档用于将 [`requirements.md`](D:/code/live-translator/requirements.md#L1) 中的产品需求收敛为可执行的技术方案，并给出：

- MVP 范围定义
- 系统架构设计
- 模块职责拆分
- 技术难点与风险判断
- 分阶段任务编排计划

当前仓库尚未进入实现阶段，本文档优先解决“先做什么、暂时不做什么、按什么顺序做”的问题。

## 2. 项目现状

当前仓库仅包含以下内容：

- `manifest.json`：Manifest V3 扩展声明
- `requirements.md`：完整需求规格
- `design.md`：设计文档

当前 `manifest.json` 中声明的以下入口文件均未创建：

- `background/background.js`
- `popup/popup.html`
- `content/content.js`
- `icons/icon128.png`

因此项目当前状态应定义为“规格阶段”，而不是“可运行原型”。

## 3. 需求拆解

原始需求可以拆成 6 个子系统：

1. 系统音频捕获
2. 音频预处理
3. 语音识别
4. 文本翻译
5. 结果展示
6. 配置与错误处理

从交付角度看，这 6 个子系统不是同等优先级。

### 3.1 核心链路

首版必须打通的最小闭环是：

`采集音频 -> 分片 -> 语音识别 -> 文本翻译 -> Popup/Overlay 展示`

只要这条链路能稳定跑通，项目就具备 MVP 价值。

### 3.2 高复杂度增强项

以下功能不适合作为首版必须项：

- 说话人分离
- 多识别引擎一次性全支持
- 多翻译引擎一次性全支持
- 浏览器侧强预处理能力
- 真正安全的长期 API Key 持久化

这些内容应放入二期，否则项目会在首轮实现阶段显著失控。

## 4. MVP 范围

### 4.1 MVP 包含

1. Chrome MV3 扩展基本骨架
2. Popup 启动/停止翻译
3. 通过屏幕共享获取系统音频
4. 音频按固定窗口分片
5. 接入 1 个语音识别引擎
6. 接入 1 个翻译引擎
7. Popup 中实时显示最近翻译结果
8. Overlay 悬浮层显示翻译结果
9. 基础配置保存
10. 基础错误提示和日志记录

### 4.2 MVP 不包含

1. 说话人分离
2. 复杂降噪与高级音频增强
3. 多模型智能路由
4. 真正加密级别的密钥长期持久化
5. 多会话并发
6. 跨设备同步敏感密钥

### 4.3 MVP 推荐首版 Provider

为了降低集成复杂度，首版建议只支持：

- 语音识别：`OpenAI Whisper`
- 翻译引擎：`OpenAI GPT` 或 `DeepL`

推荐优先级：

1. `Whisper + OpenAI GPT`
2. `Whisper + DeepL`

如果目标是尽快打通单一供应商链路，则优先 `Whisper + OpenAI GPT`。
如果目标是降低翻译提示词与响应解析复杂度，则优先 `Whisper + DeepL`。

## 5. 技术方案选择

### 方案 A：纯扩展 MVP 方案

特点：

- 所有逻辑放在浏览器扩展内
- 使用 `Popup + Service Worker + Offscreen Document + Content Script`
- API 直接由扩展调用第三方服务

优点：

- 启动快
- 依赖少
- 最适合做首版原型

缺点：

- 密钥安全边界弱
- 浏览器侧音频处理能力有限
- 可观测性与服务治理能力较弱

### 方案 B：纯扩展全量方案

特点：

- 一次性实现需求文档中的大部分能力

优点：

- 规格覆盖完整

缺点：

- 项目跨度过大
- 首版失败概率高
- 调试与稳定性成本过高

### 方案 C：扩展 + 服务端代理方案

特点：

- 扩展负责采集、展示、会话控制
- 服务端负责密钥管理、转写/翻译代理、日志与风控

优点：

- 安全边界更合理
- 后期可扩展性最好
- 方便做统一监控和重试策略

缺点：

- 初期开发成本更高
- 需要部署后端

### 结论

当前推荐路线：

1. 首版采用 `方案 A`
2. 二期演进到 `方案 C`

不建议采用 `方案 B`。

## 6. 架构设计

### 6.1 总体架构

MVP 采用以下组件：

- `Popup UI`
- `Service Worker`
- `Offscreen Document`
- `Content Script / Overlay`
- `Speech Provider Adapter`
- `Translation Provider Adapter`
- `Config Store`
- `Session State / Event Bus`

整体数据流：

`Popup -> Service Worker -> Offscreen Document -> Speech Provider -> Translation Provider -> Service Worker -> Popup / Overlay`

### 6.2 组件职责

#### Popup UI

职责：

- 展示配置界面
- 提供开始/停止按钮
- 展示当前状态
- 展示最近翻译结果
- 展示错误消息

边界：

- 不直接处理音频
- 不直接调用外部 API
- 所有运行态操作通过消息发送给 Service Worker

#### Service Worker

职责：

- 管理翻译会话状态
- 协调 Popup、Offscreen、Content Script 之间的消息
- 负责调用识别与翻译 Provider
- 记录错误与日志
- 管理运行期缓存

边界：

- 不直接访问 DOM
- 不承担音频采集与编码实现

#### Offscreen Document

职责：

- 调用 `getDisplayMedia`
- 接入音频流
- 进行分片
- 执行轻量预处理
- 将分片后的音频数据发送给 Service Worker

边界：

- 不负责 UI
- 不承担配置持久化

#### Content Script / Overlay

职责：

- 注入悬浮显示层
- 渲染结果列表
- 处理拖拽与关闭

边界：

- 不直接请求外部 API
- 不维护完整业务状态

#### Speech Provider Adapter

职责：

- 封装音频识别请求
- 屏蔽具体 Provider 差异
- 输出统一识别结果结构

#### Translation Provider Adapter

职责：

- 封装文本翻译请求
- 屏蔽具体 Provider 差异
- 输出统一翻译结果结构

#### Config Store

职责：

- 保存非敏感配置
- 读取当前运行配置
- 向 UI 提供配置表单默认值

### 6.3 可扩展性约束

首版虽然按 MVP 实现，但代码结构必须为二期能力预留扩展点。

约束如下：

1. Provider 必须通过统一接口接入，禁止在业务流程中直接写死第三方 API 调用细节。
2. 识别与翻译能力必须采用注册表或工厂模式装配，便于后续增加 `Google Speech-to-Text`、`DeepL`、`Claude`、`Gemini` 等实现。
3. Popup、Overlay、Service Worker、Offscreen 之间的消息协议必须集中定义，禁止在各处散落硬编码消息名。
4. 会话状态必须集中由 `session-manager` 管理，避免多个上下文各自维护状态副本。
5. 配置结构必须带版本意识，后续新增字段时应支持迁移或默认值补全。
6. 结果数据结构首版即预留 `speakerLabel` 等增强字段，但不在 MVP 中强依赖。
7. 存储能力必须按用途分层：`sync`、`session`、`local` 分开封装，避免未来更换安全策略时全局改动。
8. UI 展示与业务状态解耦，Popup 与 Overlay 只消费标准结果对象，不直接依赖 Provider 返回格式。

## 7. 目录规划

建议的首版目录结构如下：

```text
live-translator/
  manifest.json
  design.md
  requirements.md
  background/
    background.js
    session-manager.js
    message-router.js
    providers/
      speech/
        whisper.js
      translation/
        openai.js
        deepl.js
  popup/
    popup.html
    popup.css
    popup.js
  content/
    content.js
    overlay.js
    overlay.css
  offscreen/
    offscreen.html
    offscreen.js
    audio-capture.js
    audio-chunker.js
  shared/
    constants.js
    messages.js
    storage.js
    logger.js
    types.js
  icons/
    icon128.png
```

设计原则：

- `shared/` 只放跨端复用的常量、消息协议和工具模块
- `background/` 负责状态与服务协调
- `offscreen/` 负责媒体能力
- `popup/` 与 `content/` 只负责 UI 与交互

## 8. 关键数据设计

### 8.1 配置结构

建议配置结构：

```js
{
  speechProvider: "whisper",
  translationProvider: "openai",
  sourceLanguage: "auto",
  targetLanguage: "zh-CN",
  overlayEnabled: true,
  chunkDurationMs: 4000,
  llmModel: "gpt-4o-mini",
  customPrompt: "",
  apiKeys: {
    whisper: "",
    openai: "",
    deepl: ""
  }
}
```

其中：

- 非敏感配置可持久化到 `chrome.storage.sync`
- 敏感字段不应默认进入 `sync`

### 8.2 会话状态结构

```js
{
  status: "idle" | "starting" | "capturing" | "processing" | "error",
  startedAt: 0,
  lastError: null,
  lastTranscript: "",
  lastTranslation: "",
  results: []
}
```

### 8.3 结果结构

```js
{
  id: "uuid",
  createdAt: 0,
  transcript: "source text",
  translation: "translated text",
  sourceLanguage: "en",
  targetLanguage: "zh-CN",
  speakerLabel: null
}
```

首版保留 `speakerLabel` 字段，但默认不启用说话人分离逻辑，便于二期扩展。

## 9. 关键技术判断

### 9.1 系统音频采集

系统音频采集是项目第一风险点。

原因：

- 必须依赖浏览器和系统层的屏幕共享能力
- 用户必须主动授权
- 是否包含系统音频与平台有关
- 相关调用必须放在合适的扩展上下文中执行
- `getDisplayMedia()` 实际上要求提供视频轨，请求参数不能将 `video` 设为 `false`

设计结论：

- MVP 必须首先验证 `Offscreen Document + getDisplayMedia` 链路
- 实现时应使用 `getDisplayMedia({ video: true, audio: true })` 或等价约束，然后主动忽略视频轨，只消费音频轨
- 如果采集链路不稳定，后续所有模块都没有意义

### 9.2 音频预处理

原需求中的预处理目标过于理想化，首版建议降级为：

1. 固定时长分片
2. 基础静音检测
3. 基础格式适配

不建议首版承诺：

- 强降噪
- 明确 dBFS 目标
- 200ms 全流程硬实时指标

### 9.3 API Key 安全

原需求要求“非明文存储”，但在纯扩展模式下：

- 前端必须在某个时刻拿到明文才能发请求
- 持久化秘密本身就意味着存在被读取的风险

设计结论：

- MVP 中将敏感 Key 视为“用户本地配置”
- 默认仅在当前会话使用或放入受限存储
- 若要实现真正可接受的安全策略，应在二期接入服务端代理

### 9.4 说话人分离

说话人分离属于高复杂度增强特性，应拆出独立二期项目。

原因：

- 需要额外模型能力
- 会话级标签一致性复杂
- UI 需要针对多说话人渲染
- 测试成本高

## 10. 难易度评估

### 10.1 低难度

- Popup 基础界面
- 状态栏与错误提示
- Overlay 基础渲染
- 最近结果列表
- 配置表单基础交互

### 10.2 中难度

- 配置读写
- Service Worker 消息路由
- 翻译 Provider 接入
- 去重与结果缓存
- 错误日志记录

### 10.3 中高难度

- Offscreen 生命周期管理
- 录音开始/停止状态同步
- 音频分片与基础格式处理
- 重试机制与会话恢复

### 10.4 高难度

- 浏览器侧音频预处理
- 多 Provider 抽象一致性
- 稳定的系统音频捕获兼容性

### 10.5 很高难度

- 说话人分离
- 会话级说话人标签保持
- 真正安全的密钥长期持久化
- 大规模兼容性与稳定性治理

## 11. 任务编排计划

### 阶段 0：需求收敛与设计冻结

目标：

- 明确 MVP 与二期边界
- 冻结首版 Provider 选择
- 冻结目录结构和消息协议

输出：

- 更新后的 `design.md`
- MVP 范围清单

难度：

- 中

### 阶段 1：扩展骨架搭建

目标：

- 创建 MV3 扩展基础结构
- 补齐 manifest 声明文件
- 建立 Popup、Service Worker、Content Script、Offscreen 入口

输出：

- 可加载的扩展骨架

前置依赖：

- 阶段 0 完成

难度：

- 低

### 阶段 2：音频捕获链路验证

目标：

- 建立 Popup 启动会话能力
- 建立 Offscreen 文档
- 打通 `getDisplayMedia` 采集系统音频
- 建立基础音频分片

输出：

- 可获取音频分片的最小闭环

前置依赖：

- 阶段 1 完成

难度：

- 中高

### 阶段 3：语音识别接入

目标：

- 接入单一 Speech Provider
- 完成音频上传、响应解析、空结果过滤
- 加入 4xx/5xx 基础错误处理

输出：

- 可将音频转为文本

前置依赖：

- 阶段 2 完成

难度：

- 中

### 阶段 4：文本翻译接入

目标：

- 接入单一 Translation Provider
- 处理源语言/目标语言参数
- 处理重复文本跳过逻辑

输出：

- 可将文本实时翻译

前置依赖：

- 阶段 3 完成

难度：

- 中

### 阶段 5：结果展示完成

目标：

- Popup 实时结果渲染
- Overlay 注入和更新
- 保留最近 20 条记录
- 支持 Overlay 关闭和拖动

输出：

- 用户可见的 MVP 交互闭环

前置依赖：

- 阶段 4 完成

难度：

- 低到中

### 阶段 6：配置与可靠性增强

目标：

- 配置持久化
- 状态恢复
- 网络错误提示
- 会话中断恢复
- 错误日志写入

输出：

- 可持续测试和内部试用的 MVP

前置依赖：

- 阶段 5 完成

难度：

- 中

### 阶段 7：二期增强

目标：

- 说话人分离
- 多 Provider 并行支持
- 复杂预处理
- 服务端代理

输出：

- 增强版本架构

难度：

- 高到很高

## 12. 优先级排序

推荐开发优先级如下：

1. 先验证系统音频是否可采
2. 再打通识别
3. 再打通翻译
4. 再做可视化展示
5. 最后补配置与可靠性

如果第一步采集链路失败，项目需立刻调整方案，而不是继续实现后续模块。

## 13. 测试策略

首版测试重点不是“全自动化覆盖”，而是优先验证关键链路。

### 13.1 必测链路

1. 扩展可加载
2. Popup 可启动和停止会话
3. 可拉起系统分享窗口
4. 可采集音频并产生分片
5. 分片可调用识别 API
6. 识别结果可进入翻译 API
7. 翻译结果可同步到 Popup 和 Overlay

### 13.2 错误场景

1. 用户拒绝屏幕共享
2. 未配置 API Key
3. 网络中断
4. 第三方 API 4xx
5. 第三方 API 5xx
6. Overlay 已关闭但后台仍在运行

## 14. 结论

本项目适合按“先闭环、后增强”的方式推进。

明确结论如下：

1. 当前应按 MVP 路线推进
2. 说话人分离必须拆到二期
3. 真正安全的密钥长期持久化不应作为纯扩展 MVP 的强约束
4. 首轮实现的唯一核心目标，是验证系统音频采集到翻译展示的完整闭环

后续若继续推进，应基于本文档生成详细实施计划，并按阶段逐步落地。
