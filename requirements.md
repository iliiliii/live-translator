# 需求文档

## 简介

Live Translator 是一款 Chrome 扩展（Manifest V3），通过 `getDisplayMedia` API 捕获屏幕共享时的系统音频（包括非 Chrome 应用的声音），将音频流经过预处理（降噪、格式转换、音量归一化）后送入语音识别 API（Whisper 或 Google Speech-to-Text）进行语音转文字，支持说话人分离（Speaker Diarization）以区分不同说话人并在结果中标注说话人标签，再调用翻译引擎（DeepL、Google Translate，或大语言模型如 OpenAI GPT、Claude、Gemini 等）将识别结果翻译为目标语言，最终在扩展 Popup 或悬浮窗中实时展示翻译结果。

## 词汇表

- **Extension**：本 Chrome 扩展整体，包含 Popup、Service Worker、Content Script 等组成部分。
- **Popup**：点击扩展图标后弹出的控制面板，用于配置参数和展示翻译结果。
- **Overlay**：注入到当前页面的悬浮字幕窗口，用于在不打开 Popup 的情况下实时展示翻译结果。
- **Service_Worker**：扩展的后台服务工作线程（Manifest V3 background），负责协调音频捕获、API 调用和状态管理。
- **Audio_Capturer**：负责通过 `getDisplayMedia` 捕获系统音频的模块。
- **Audio_Preprocessor**：负责对原始音频流进行预处理（降噪、格式转换、音量归一化等）的模块，位于 Audio_Capturer 与 Speech_Recognizer 之间。
- **Speaker_Diarizer**：负责对音频流进行说话人分离（Speaker Diarization），识别并区分不同说话人的模块。
- **Speech_Recognizer**：负责将音频数据发送至语音识别 API 并返回文字结果的模块。
- **Translator**：负责将识别文字发送至翻译 API 并返回翻译结果的模块，支持 DeepL、Google Translate 及大语言模型引擎。
- **LLM_Translator**：Translator 的子模式，使用大语言模型（OpenAI GPT、Claude、Gemini 等）作为翻译引擎。
- **Result_Display**：负责在 Popup 或 Overlay 中渲染并更新翻译结果的模块。
- **Config_Store**：负责持久化存储用户配置（API 密钥、语言设置等）的模块，使用 `chrome.storage.sync`。
- **说话人标签**：用于标识某段语音归属于哪位说话人的标记，格式为"说话人 1"、"说话人 2"等。
- **源语言**：被识别音频的语言。
- **目标语言**：翻译结果所使用的语言。

## 需求

### 需求 1：系统音频捕获

**用户故事：** 作为用户，我希望扩展能捕获屏幕共享时的系统音频（包括非 Chrome 应用的声音），以便对任意应用的音频进行实时翻译。

#### 验收标准

1. WHEN 用户在 Popup 中点击"开始翻译"按钮，THE Audio_Capturer SHALL 调用 `getDisplayMedia({ audio: true, video: false })` 请求系统音频权限。
2. WHEN 用户在系统弹窗中授予屏幕共享权限并勾选"分享系统音频"，THE Audio_Capturer SHALL 获取包含系统音频轨道的 `MediaStream`。
3. IF 用户拒绝屏幕共享权限，THEN THE Audio_Capturer SHALL 停止捕获流程并向 Popup 返回权限拒绝错误。
4. IF `getDisplayMedia` 调用抛出异常，THEN THE Audio_Capturer SHALL 捕获该异常并向 Service_Worker 上报错误信息。
5. WHEN 用户点击"停止翻译"按钮，THE Audio_Capturer SHALL 停止所有音频轨道并释放 `MediaStream`。
6. WHILE 音频捕获处于活动状态，THE Audio_Capturer SHALL 持续将音频数据以不超过 5 秒的分片发送给 Speech_Recognizer。

---

### 需求 1.5：音频流预处理

**用户故事：** 作为用户，我希望捕获的音频在送入语音识别之前能经过预处理，以便提升识别准确率并减少噪音干扰。

#### 验收标准

1. WHEN Audio_Capturer 提供一个原始音频分片，THE Audio_Preprocessor SHALL 在将该分片传递给 Speech_Recognizer 之前完成所有预处理步骤。
2. THE Audio_Preprocessor SHALL 对音频分片应用降噪处理，将背景噪声功率降低至原始水平的 20% 以下。
3. THE Audio_Preprocessor SHALL 将音频分片转换为语音识别 API 所需的目标格式（采样率 16kHz、单声道、16-bit PCM 或 WebM Opus）。
4. THE Audio_Preprocessor SHALL 对音频分片进行音量归一化，将峰值音量调整至 -3 dBFS 至 -1 dBFS 范围内。
5. IF 音频分片的平均音量低于静音阈值（-60 dBFS），THEN THE Audio_Preprocessor SHALL 将该分片标记为静音并跳过后续处理，不向 Speech_Recognizer 传递。
6. IF 预处理过程中发生异常，THEN THE Audio_Preprocessor SHALL 记录错误信息并将原始未处理的音频分片直接传递给 Speech_Recognizer，以保证流程不中断。
7. WHILE 音频预处理处于活动状态，THE Audio_Preprocessor SHALL 在 200 毫秒内完成单个音频分片的全部预处理步骤，以保证实时性。

---

### 需求 2：语音识别

**用户故事：** 作为用户，我希望捕获的音频能被自动转换为文字，以便后续进行翻译处理。

#### 验收标准

1. WHEN Audio_Capturer 提供一个音频分片，THE Speech_Recognizer SHALL 将该分片编码为目标 API 所需格式（WAV 或 WebM）后发送至已配置的语音识别 API。
2. WHEN 语音识别 API 返回识别结果，THE Speech_Recognizer SHALL 将识别文本传递给 Translator。
3. IF 语音识别 API 返回 HTTP 4xx 错误，THEN THE Speech_Recognizer SHALL 停止当前会话并向 Service_Worker 上报包含错误码的错误信息。
4. IF 语音识别 API 返回 HTTP 5xx 错误或网络超时，THEN THE Speech_Recognizer SHALL 对该分片最多重试 3 次，每次重试间隔不少于 1 秒。
5. WHERE 用户配置了 Whisper API，THE Speech_Recognizer SHALL 使用 OpenAI Whisper API 端点进行识别。
6. WHERE 用户配置了 Google Speech-to-Text API，THE Speech_Recognizer SHALL 使用 Google Speech-to-Text v1 端点进行识别。
7. WHEN 音频分片中未检测到语音内容，THE Speech_Recognizer SHALL 丢弃该分片并不向 Translator 传递任何内容。

---

### 需求 2.5：说话人分离

**用户故事：** 作为用户，我希望翻译结果能标注是哪位说话人说的内容，以便在多人对话场景中区分不同发言者。

#### 验收标准

1. WHERE 用户在配置中启用了说话人分离功能，THE Speaker_Diarizer SHALL 对每个音频分片进行说话人识别，并为识别出的文本附加说话人标签。
2. WHEN Speaker_Diarizer 识别出音频分片中的说话人，THE Speaker_Diarizer SHALL 为该说话人分配一个在当前会话内唯一的说话人标签（如"说话人 1"、"说话人 2"）。
3. THE Speaker_Diarizer SHALL 在同一翻译会话内保持说话人标签的一致性，即同一说话人在整个会话中始终使用相同的标签。
4. WHEN Translator 提供带有说话人标签的翻译文本，THE Result_Display SHALL 在翻译结果前显示对应的说话人标签（如"[说话人 1] 翻译内容"）。
5. WHERE 用户启用了说话人分离功能，THE Result_Display SHALL 为不同说话人的翻译结果使用不同的视觉样式（如不同颜色或缩进）加以区分。
6. IF Speaker_Diarizer 无法确定音频分片的说话人归属，THEN THE Speaker_Diarizer SHALL 将该分片标记为"未知说话人"并继续处理，不中断翻译流程。
7. WHERE 用户未启用说话人分离功能，THE Speech_Recognizer SHALL 直接将识别文本传递给 Translator，不附加任何说话人标签。
8. THE Speaker_Diarizer SHALL 支持在单个音频分片内识别最多 10 位不同的说话人。

---

### 需求 3：文本翻译

**用户故事：** 作为用户，我希望识别出的文字能被自动翻译为我指定的目标语言，以便理解外语内容；同时希望能选择使用不同的翻译引擎，包括传统翻译 API 和大语言模型。

#### 验收标准

1. WHEN Speech_Recognizer 提供识别文本，THE Translator SHALL 将该文本连同源语言和目标语言参数发送至已配置的翻译引擎。
2. WHEN 翻译引擎返回翻译结果，THE Translator SHALL 将翻译文本传递给 Result_Display。
3. IF 翻译 API 返回 HTTP 4xx 错误，THEN THE Translator SHALL 停止当前翻译请求并向 Service_Worker 上报包含错误码的错误信息。
4. IF 翻译 API 返回 HTTP 5xx 错误或网络超时，THEN THE Translator SHALL 对该请求最多重试 3 次，每次重试间隔不少于 1 秒。
5. WHERE 用户配置了 DeepL API，THE Translator SHALL 使用 DeepL REST API 端点进行翻译。
6. WHERE 用户配置了 Google Translate API，THE Translator SHALL 使用 Google Cloud Translation v2 端点进行翻译。
7. WHEN 识别文本与上一条识别文本完全相同，THE Translator SHALL 跳过重复翻译并直接复用上一条翻译结果。
8. WHERE 用户配置了 OpenAI GPT 作为翻译引擎，THE LLM_Translator SHALL 通过 OpenAI Chat Completions API 发送翻译请求，并在系统提示中指定源语言、目标语言及翻译角色。
9. WHERE 用户配置了 Claude（Anthropic）作为翻译引擎，THE LLM_Translator SHALL 通过 Anthropic Messages API 发送翻译请求，并在系统提示中指定源语言、目标语言及翻译角色。
10. WHERE 用户配置了 Gemini（Google）作为翻译引擎，THE LLM_Translator SHALL 通过 Google Generative Language API 发送翻译请求，并在系统提示中指定源语言、目标语言及翻译角色。
11. WHEN 使用大语言模型作为翻译引擎，THE LLM_Translator SHALL 仅提取模型响应中的翻译文本内容，过滤掉模型输出的解释、注释或其他非翻译内容。
12. WHERE 用户配置了大语言模型翻译引擎，THE Config_Store SHALL 允许用户自定义翻译提示词（System Prompt），以便调整翻译风格或添加领域专业词汇指引。

---

### 需求 4：实时结果展示

**用户故事：** 作为用户，我希望在 Popup 或悬浮窗中实时看到翻译结果，以便在不中断当前工作的情况下理解音频内容。

#### 验收标准

1. WHEN Translator 提供翻译文本，THE Result_Display SHALL 在 500 毫秒内将翻译文本更新至 Popup 的结果区域。
2. WHEN Translator 提供翻译文本，THE Result_Display SHALL 在 500 毫秒内将翻译文本更新至页面中已激活的 Overlay。
3. THE Result_Display SHALL 在结果区域中保留最近 20 条翻译记录，超出部分自动滚动至最新条目。
4. WHERE 用户启用了 Overlay 模式，THE Result_Display SHALL 在当前活动标签页中注入并显示 Overlay 悬浮窗。
5. WHEN 用户拖动 Overlay 窗口，THE Result_Display SHALL 将 Overlay 移动至用户释放鼠标的位置，并在页面刷新前保持该位置。
6. WHEN 用户点击 Overlay 上的关闭按钮，THE Result_Display SHALL 隐藏 Overlay 并停止向其推送新结果。
7. WHILE 翻译会话处于活动状态，THE Result_Display SHALL 在 Popup 中显示"翻译中"状态指示器。

---

### 需求 5：API 配置管理

**用户故事：** 作为用户，我希望能够配置语音识别和翻译 API 的密钥及语言选项，以便使用我自己的 API 账户；同时希望能选择和配置不同的翻译引擎。

#### 验收标准

1. THE Config_Store SHALL 提供配置界面，允许用户输入语音识别 API 密钥（Whisper 或 Google Speech-to-Text）。
2. THE Config_Store SHALL 提供配置界面，允许用户选择翻译引擎类型，可选项包括：DeepL、Google Translate、OpenAI GPT、Claude（Anthropic）、Gemini（Google）。
3. THE Config_Store SHALL 根据用户选择的翻译引擎，动态显示对应的 API 密钥输入字段。
4. THE Config_Store SHALL 提供下拉菜单，允许用户选择源语言（包含"自动检测"选项）。
5. THE Config_Store SHALL 提供下拉菜单，允许用户选择目标语言。
6. WHERE 用户选择了大语言模型翻译引擎，THE Config_Store SHALL 提供文本输入区域，允许用户自定义翻译提示词（System Prompt），并提供默认提示词作为初始值。
7. WHERE 用户选择了大语言模型翻译引擎，THE Config_Store SHALL 提供下拉菜单，允许用户选择具体的模型版本（如 GPT-4o、claude-3-5-sonnet、gemini-1.5-pro 等）。
8. WHERE 用户启用了说话人分离功能，THE Config_Store SHALL 提供开关控件允许用户启用或禁用说话人分离，默认为禁用。
9. WHEN 用户保存配置，THE Config_Store SHALL 使用 `chrome.storage.sync` 将配置持久化存储，并在不同设备间同步。
10. IF 用户未配置任何 API 密钥即点击"开始翻译"，THEN THE Extension SHALL 阻止启动并在 Popup 中显示"请先完成 API 配置"提示。
11. THE Config_Store SHALL 以明文之外的方式存储 API 密钥，不得将密钥暴露在扩展的 DOM 或日志中。

---

### 需求 6：错误处理与用户反馈

**用户故事：** 作为用户，我希望在出现错误时能收到清晰的提示，以便了解问题原因并采取相应措施。

#### 验收标准

1. WHEN Service_Worker 收到来自任意模块的错误信息，THE Extension SHALL 在 Popup 的错误提示区域显示人类可读的错误描述，显示时长不少于 5 秒。
2. IF 网络连接中断，THEN THE Service_Worker SHALL 暂停 API 调用并在 Popup 中显示"网络连接已断开，等待重连"提示。
3. WHEN 网络连接恢复，THE Service_Worker SHALL 自动恢复 API 调用并清除网络断开提示。
4. IF 翻译会话因错误意外终止，THEN THE Extension SHALL 在 Popup 中显示"会话已中断"提示，并提供"重新开始"按钮。
5. THE Extension SHALL 将所有错误事件记录至 `chrome.storage.local` 的日志条目中，每条日志包含时间戳、错误类型和错误描述。
