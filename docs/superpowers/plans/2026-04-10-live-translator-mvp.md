# Live Translator MVP Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 构建一个可扩展的 Chrome MV3 扩展 MVP，完成“系统音频采集 -> Whisper 转写 -> OpenAI 翻译 -> Popup/Overlay 展示”的最小闭环。

**Architecture:** 采用纯扩展方案，运行时由 `Popup + Service Worker + Offscreen Document + Content Script` 组成。扩展性通过集中消息协议、会话状态机、Provider 注册表、分层存储封装和标准化结果结构来保证，避免后续新增 Provider 或增强功能时重写主链路。

**Tech Stack:** Chrome Extension MV3, Vanilla JavaScript ES Modules, Offscreen API, `getDisplayMedia`, MediaRecorder, Web Audio API, Fetch API, Vitest, jsdom

---

## File Structure

- `manifest.json`
  负责声明 MV3 入口、权限、`offscreen` 能力和模块化 background worker。

- `package.json`
  负责测试命令、开发依赖和后续一致的本地执行入口。

- `shared/constants.js`
  负责配置默认值、结果数量上限、日志上限、配置版本号等常量。

- `shared/messages.js`
  负责统一定义 Popup、Service Worker、Offscreen、Overlay 之间的消息协议。

- `shared/storage.js`
  负责按 `sync/session/local` 分层封装存储、配置归一化和配置迁移。

- `shared/logger.js`
  负责错误日志写入 `chrome.storage.local`，并裁剪最大日志条数。

- `shared/types.js`
  负责记录标准数据结构样例和字段说明，作为跨模块约定。

- `background/background.js`
  负责注册 `runtime.onMessage`、初始化路由器和会话管理器。

- `background/session-manager.js`
  负责唯一真实的会话状态源、结果列表截断、状态流转和订阅通知。

- `background/message-router.js`
  负责路由开始会话、停止会话、推送结果、推送错误、获取状态等消息。

- `background/providers/speech/base.js`
  负责定义语音识别 Provider 接口约定。

- `background/providers/speech/index.js`
  负责注册和选择具体语音识别 Provider。

- `background/providers/speech/whisper.js`
  负责 OpenAI Whisper 请求封装、错误分类和重试。

- `background/providers/translation/base.js`
  负责定义翻译 Provider 接口约定。

- `background/providers/translation/index.js`
  负责注册和选择具体翻译 Provider。

- `background/providers/translation/openai.js`
  负责 OpenAI 文本翻译请求封装。

- `background/speech-service.js`
  负责将音频分片送入语音识别 Provider，并输出统一文本结果。

- `background/translation-service.js`
  负责翻译请求、重复文本去重和统一翻译结果格式。

- `offscreen/offscreen.html`
  负责提供 Offscreen Document 载体。

- `offscreen/offscreen.js`
  负责初始化音频采集控制器和消息监听。

- `offscreen/audio-capture.js`
  负责 `getDisplayMedia()`、MediaStream 生命周期和 MediaRecorder 启停。

- `offscreen/audio-chunker.js`
  负责分片元数据、静音判断和向 background 发送标准分片对象。

- `popup/popup.html`
  负责 MVP 控制台界面结构。

- `popup/popup.css`
  负责 Popup 样式。

- `popup/popup.js`
  负责加载配置、开始/停止会话、订阅状态和渲染结果。

- `content/content.js`
  负责懒加载 Overlay、接收消息并控制渲染。

- `content/overlay.js`
  负责创建悬浮窗、渲染结果、处理拖动和关闭。

- `content/overlay.css`
  负责 Overlay 样式。

- `tests/shared/*.test.js`
  负责共享协议、配置归一化和日志工具的单元测试。

- `tests/background/*.test.js`
  负责会话状态、Provider 注册表、Provider 请求和翻译去重逻辑测试。

- `tests/offscreen/*.test.js`
  负责分片工具和静音判断等纯函数测试。

- `docs/manual-smoke-checklist.md`
  负责系统音频采集和扩展交互的手工验证步骤。

## Task 1: Shared Contracts and Tooling Baseline

**Files:**
- Create: `package.json`
- Create: `shared/constants.js`
- Create: `shared/messages.js`
- Create: `shared/storage.js`
- Create: `shared/logger.js`
- Create: `shared/types.js`
- Create: `tests/setup/chrome-stub.js`
- Create: `tests/shared/messages.test.js`
- Create: `tests/shared/storage.test.js`
- Test: `tests/shared/messages.test.js`
- Test: `tests/shared/storage.test.js`

- [ ] **Step 1: 写测试运行基线和首批失败测试**

```json
{
  "name": "live-translator",
  "private": true,
  "type": "module",
  "scripts": {
    "test": "vitest run"
  },
  "devDependencies": {
    "jsdom": "^26.0.0",
    "vitest": "^3.2.0"
  }
}
```

```js
// tests/setup/chrome-stub.js
globalThis.chrome = {
  storage: {
    sync: {
      async get() { return {}; },
      async set() {}
    },
    session: {
      async get() { return {}; },
      async set() {}
    },
    local: {
      async get() { return {}; },
      async set() {}
    }
  }
};
```

```js
// tests/shared/messages.test.js
import { describe, expect, it } from "vitest";
import { MESSAGE_TYPES } from "../../shared/messages.js";

describe("MESSAGE_TYPES", () => {
  it("暴露稳定的消息协议名", () => {
    expect(MESSAGE_TYPES.START_SESSION).toBe("session/start");
    expect(MESSAGE_TYPES.STOP_SESSION).toBe("session/stop");
    expect(MESSAGE_TYPES.AUDIO_CHUNK_READY).toBe("audio/chunk-ready");
    expect(MESSAGE_TYPES.RESULT_PUSHED).toBe("result/pushed");
  });
});
```

```js
// tests/shared/storage.test.js
import { describe, expect, it } from "vitest";
import { DEFAULT_CONFIG, normalizeConfig } from "../../shared/storage.js";

describe("DEFAULT_CONFIG", () => {
  it("使用可扩展的默认配置", () => {
    expect(DEFAULT_CONFIG).toMatchObject({
      schemaVersion: 1,
      speechProvider: "whisper",
      translationProvider: "openai",
      overlayEnabled: true,
      chunkDurationMs: 4000
    });
  });
});

describe("normalizeConfig", () => {
  it("补齐缺省字段并保留用户输入", () => {
    expect(normalizeConfig({ targetLanguage: "ja-JP" })).toMatchObject({
      schemaVersion: 1,
      speechProvider: "whisper",
      translationProvider: "openai",
      targetLanguage: "ja-JP",
      overlayEnabled: true
    });
  });
});
```

- [ ] **Step 2: 运行测试，确认当前失败**

Run: `npm install`

Run: `npm run test -- tests/shared/messages.test.js tests/shared/storage.test.js`

Expected: FAIL，错误包含 `Cannot find module '../../shared/messages.js'` 和 `Cannot find module '../../shared/storage.js'`

- [ ] **Step 3: 写共享协议和配置实现**

```js
// shared/constants.js
export const CONFIG_SCHEMA_VERSION = 1;
export const RESULT_HISTORY_LIMIT = 20;
export const ERROR_LOG_LIMIT = 100;
export const DEFAULT_CHUNK_DURATION_MS = 4000;
```

```js
// shared/messages.js
export const MESSAGE_TYPES = {
  START_SESSION: "session/start",
  STOP_SESSION: "session/stop",
  GET_SESSION_STATE: "session/get-state",
  SESSION_STATE_UPDATED: "session/state-updated",
  OFFSCREEN_READY: "offscreen/ready",
  START_CAPTURE: "audio/start-capture",
  STOP_CAPTURE: "audio/stop-capture",
  AUDIO_CHUNK_READY: "audio/chunk-ready",
  PUSH_ERROR: "error/push",
  RESULT_PUSHED: "result/pushed",
  OVERLAY_RENDER: "overlay/render"
};
```

```js
// shared/storage.js
import {
  CONFIG_SCHEMA_VERSION,
  DEFAULT_CHUNK_DURATION_MS
} from "./constants.js";

export const DEFAULT_CONFIG = {
  schemaVersion: CONFIG_SCHEMA_VERSION,
  speechProvider: "whisper",
  translationProvider: "openai",
  sourceLanguage: "auto",
  targetLanguage: "zh-CN",
  overlayEnabled: true,
  chunkDurationMs: DEFAULT_CHUNK_DURATION_MS,
  llmModel: "gpt-4o-mini",
  customPrompt: ""
};

export function normalizeConfig(input = {}) {
  return {
    ...DEFAULT_CONFIG,
    ...input,
    schemaVersion: CONFIG_SCHEMA_VERSION
  };
}

export async function getSyncConfig() {
  const stored = await chrome.storage.sync.get("config");
  return normalizeConfig(stored.config);
}

export async function setSyncConfig(config) {
  await chrome.storage.sync.set({ config: normalizeConfig(config) });
}
```

```js
// shared/logger.js
import { ERROR_LOG_LIMIT } from "./constants.js";

export async function appendErrorLog(entry) {
  const stored = await chrome.storage.local.get("errorLogs");
  const logs = Array.isArray(stored.errorLogs) ? stored.errorLogs : [];
  const nextLogs = [...logs, entry].slice(-ERROR_LOG_LIMIT);
  await chrome.storage.local.set({ errorLogs: nextLogs });
}
```

```js
// shared/types.js
export const SESSION_STATUS = ["idle", "starting", "capturing", "processing", "error"];
```

- [ ] **Step 4: 运行测试，确认共享契约通过**

Run: `npm run test -- tests/shared/messages.test.js tests/shared/storage.test.js`

Expected: PASS，2 个测试文件全部通过

- [ ] **Step 5: 提交**

```bash
git add package.json shared tests
git commit -m "chore: scaffold shared contracts and test tooling"
```

## Task 2: Extension Shell and Session State Core

**Files:**
- Modify: `manifest.json`
- Create: `background/background.js`
- Create: `background/session-manager.js`
- Create: `background/message-router.js`
- Create: `popup/popup.html`
- Create: `popup/popup.css`
- Create: `popup/popup.js`
- Create: `content/content.js`
- Create: `offscreen/offscreen.html`
- Create: `offscreen/offscreen.js`
- Create: `tests/background/session-manager.test.js`
- Test: `tests/background/session-manager.test.js`

- [ ] **Step 1: 先写会话状态失败测试**

```js
// tests/background/session-manager.test.js
import { describe, expect, it } from "vitest";
import { createSessionManager } from "../../background/session-manager.js";

describe("createSessionManager", () => {
  it("初始化为空闲状态", () => {
    const manager = createSessionManager();
    expect(manager.getState().status).toBe("idle");
    expect(manager.getState().results).toEqual([]);
  });

  it("只保留最近 20 条结果", () => {
    const manager = createSessionManager();
    for (let index = 0; index < 25; index += 1) {
      manager.pushResult({ id: String(index), translation: `line-${index}` });
    }
    expect(manager.getState().results).toHaveLength(20);
    expect(manager.getState().results[0].id).toBe("5");
  });
});
```

- [ ] **Step 2: 运行测试，确认状态管理器尚未实现**

Run: `npm run test -- tests/background/session-manager.test.js`

Expected: FAIL，错误包含 `Cannot find module '../../background/session-manager.js'`

- [ ] **Step 3: 写扩展骨架和状态核心**

```json
{
  "manifest_version": 3,
  "name": "Live Translator",
  "version": "1.0.0",
  "description": "Real-time system audio translator using screen capture",
  "permissions": ["activeTab", "tabs", "storage", "scripting", "offscreen"],
  "host_permissions": ["<all_urls>"],
  "background": {
    "service_worker": "background/background.js",
    "type": "module"
  },
  "action": {
    "default_popup": "popup/popup.html"
  },
  "content_scripts": [
    {
      "matches": ["<all_urls>"],
      "js": ["content/content.js"]
    }
  ]
}
```

```js
// background/session-manager.js
import { RESULT_HISTORY_LIMIT } from "../shared/constants.js";

function createInitialState() {
  return {
    status: "idle",
    startedAt: 0,
    lastError: null,
    lastTranscript: "",
    lastTranslation: "",
    results: []
  };
}

export function createSessionManager() {
  let state = createInitialState();

  return {
    getState() {
      return state;
    },
    setStatus(status) {
      state = { ...state, status };
    },
    start() {
      state = { ...createInitialState(), status: "starting", startedAt: Date.now() };
    },
    stop() {
      state = createInitialState();
    },
    setError(error) {
      state = { ...state, status: "error", lastError: error };
    },
    pushResult(result) {
      state = {
        ...state,
        results: [...state.results, result].slice(-RESULT_HISTORY_LIMIT),
        lastTranslation: result.translation ?? state.lastTranslation,
        lastTranscript: result.transcript ?? state.lastTranscript
      };
    }
  };
}
```

```js
// background/message-router.js
import { MESSAGE_TYPES } from "../shared/messages.js";

export function createMessageRouter({ sessionManager }) {
  return async function handleMessage(message) {
    switch (message?.type) {
      case MESSAGE_TYPES.GET_SESSION_STATE:
        return sessionManager.getState();
      case MESSAGE_TYPES.START_SESSION:
        sessionManager.start();
        return { ok: true };
      case MESSAGE_TYPES.STOP_SESSION:
        sessionManager.stop();
        return { ok: true };
      case MESSAGE_TYPES.RESULT_PUSHED:
        sessionManager.pushResult(message.payload);
        return { ok: true };
      case MESSAGE_TYPES.PUSH_ERROR:
        sessionManager.setError(message.payload);
        return { ok: true };
      default:
        return { ok: false, error: "unknown message" };
    }
  };
}
```

```js
// background/background.js
import { createSessionManager } from "./session-manager.js";
import { createMessageRouter } from "./message-router.js";

const sessionManager = createSessionManager();
const routeMessage = createMessageRouter({ sessionManager });

chrome.runtime.onMessage.addListener((message, _sender, sendResponse) => {
  routeMessage(message).then(sendResponse);
  return true;
});
```

```html
<!-- popup/popup.html -->
<!doctype html>
<html lang="zh-CN">
  <head>
    <meta charset="UTF-8" />
    <title>Live Translator</title>
    <link rel="stylesheet" href="./popup.css" />
  </head>
  <body>
    <main>
      <div id="status">空闲</div>
      <button id="startButton">开始翻译</button>
      <button id="stopButton">停止翻译</button>
      <section id="errorBox"></section>
      <section id="results"></section>
    </main>
    <script type="module" src="./popup.js"></script>
  </body>
</html>
```

```js
// popup/popup.js
import { MESSAGE_TYPES } from "../shared/messages.js";

async function refreshState() {
  const state = await chrome.runtime.sendMessage({ type: MESSAGE_TYPES.GET_SESSION_STATE });
  document.querySelector("#status").textContent = state.status;
}

document.querySelector("#startButton").addEventListener("click", async () => {
  await chrome.runtime.sendMessage({ type: MESSAGE_TYPES.START_SESSION });
  await refreshState();
});

document.querySelector("#stopButton").addEventListener("click", async () => {
  await chrome.runtime.sendMessage({ type: MESSAGE_TYPES.STOP_SESSION });
  await refreshState();
});

refreshState();
```

```js
// content/content.js
chrome.runtime.onMessage.addListener((message) => {
  if (message?.type === "overlay/render") {
    console.debug("overlay render placeholder", message.payload);
  }
});
```

```html
<!-- offscreen/offscreen.html -->
<!doctype html>
<html lang="zh-CN">
  <body>
    <script type="module" src="./offscreen.js"></script>
  </body>
</html>
```

```js
// offscreen/offscreen.js
chrome.runtime.sendMessage({ type: "offscreen/ready" });
```

- [ ] **Step 4: 运行测试，确认会话状态核心通过**

Run: `npm run test -- tests/background/session-manager.test.js`

Expected: PASS，`createSessionManager` 两个测试通过

- [ ] **Step 5: 手工加载扩展，确认骨架可打开**

Run: 在 Chrome 打开 `chrome://extensions`，开启开发者模式，加载 `D:\code\live-translator`

Expected: 扩展可成功加载，点击图标能打开 Popup，Popup 中能看到“空闲”“开始翻译”“停止翻译”

- [ ] **Step 6: 提交**

```bash
git add manifest.json background popup content offscreen tests
git commit -m "feat: add extension shell and session core"
```

## Task 3: Offscreen Audio Capture Pipeline

**Files:**
- Modify: `offscreen/offscreen.js`
- Create: `offscreen/audio-capture.js`
- Create: `offscreen/audio-chunker.js`
- Create: `tests/offscreen/audio-chunker.test.js`
- Test: `tests/offscreen/audio-chunker.test.js`

- [ ] **Step 1: 写分片与静音判断失败测试**

```js
// tests/offscreen/audio-chunker.test.js
import { describe, expect, it } from "vitest";
import { buildChunkEnvelope, isSilentLevel } from "../../offscreen/audio-chunker.js";

describe("isSilentLevel", () => {
  it("在低于阈值时标记为静音", () => {
    expect(isSilentLevel(-72, -60)).toBe(true);
    expect(isSilentLevel(-45, -60)).toBe(false);
  });
});

describe("buildChunkEnvelope", () => {
  it("包装统一的分片消息结构", () => {
    const blob = new Blob(["demo"], { type: "audio/webm" });
    const payload = buildChunkEnvelope({
      blob,
      sessionId: "session-1",
      chunkIndex: 3,
      averageDbfs: -18
    });
    expect(payload.sessionId).toBe("session-1");
    expect(payload.chunkIndex).toBe(3);
    expect(payload.mimeType).toBe("audio/webm");
  });
});
```

- [ ] **Step 2: 运行测试，确认音频工具尚未实现**

Run: `npm run test -- tests/offscreen/audio-chunker.test.js`

Expected: FAIL，错误包含 `Cannot find module '../../offscreen/audio-chunker.js'`

- [ ] **Step 3: 实现 Offscreen 捕获和分片工具**

```js
// offscreen/audio-chunker.js
export function isSilentLevel(averageDbfs, thresholdDbfs = -60) {
  return averageDbfs < thresholdDbfs;
}

export function buildChunkEnvelope({ blob, sessionId, chunkIndex, averageDbfs }) {
  return {
    sessionId,
    chunkIndex,
    averageDbfs,
    mimeType: blob.type,
    createdAt: Date.now(),
    blob
  };
}
```

```js
// offscreen/audio-capture.js
import { MESSAGE_TYPES } from "../shared/messages.js";
import { buildChunkEnvelope } from "./audio-chunker.js";

export function createAudioCaptureController({ chunkDurationMs = 4000 }) {
  let mediaStream = null;
  let recorder = null;
  let chunkIndex = 0;

  async function start(sessionId) {
    mediaStream = await navigator.mediaDevices.getDisplayMedia({
      video: true,
      audio: true
    });

    recorder = new MediaRecorder(mediaStream, { mimeType: "audio/webm" });
    recorder.addEventListener("dataavailable", async (event) => {
      if (!event.data || event.data.size === 0) return;
      const payload = buildChunkEnvelope({
        blob: event.data,
        sessionId,
        chunkIndex,
        averageDbfs: -20
      });
      chunkIndex += 1;
      await chrome.runtime.sendMessage({
        type: MESSAGE_TYPES.AUDIO_CHUNK_READY,
        payload
      });
    });

    recorder.start(chunkDurationMs);
  }

  function stop() {
    recorder?.stop();
    mediaStream?.getTracks().forEach((track) => track.stop());
    recorder = null;
    mediaStream = null;
    chunkIndex = 0;
  }

  return { start, stop };
}
```

```js
// offscreen/offscreen.js
import { MESSAGE_TYPES } from "../shared/messages.js";
import { createAudioCaptureController } from "./audio-capture.js";

const audioCapture = createAudioCaptureController();

chrome.runtime.onMessage.addListener((message, _sender, sendResponse) => {
  if (message?.type === MESSAGE_TYPES.START_CAPTURE) {
    audioCapture.start(message.payload.sessionId).then(() => sendResponse({ ok: true }));
    return true;
  }

  if (message?.type === MESSAGE_TYPES.STOP_CAPTURE) {
    audioCapture.stop();
    sendResponse({ ok: true });
    return true;
  }

  return undefined;
});

chrome.runtime.sendMessage({ type: MESSAGE_TYPES.OFFSCREEN_READY });
```

- [ ] **Step 4: 运行测试，确认分片工具通过**

Run: `npm run test -- tests/offscreen/audio-chunker.test.js`

Expected: PASS，静音判断和分片封装测试通过

- [ ] **Step 5: 手工验证采集链路**

Run:
1. 加载扩展
2. 点击“开始翻译”
3. 在系统弹窗中选择一个屏幕或标签页，并勾选分享系统音频

Expected:
1. 不出现 `TypeError`
2. 可以进入捕获状态
3. 停止后能正常释放轨道

- [ ] **Step 6: 提交**

```bash
git add offscreen tests/offscreen
git commit -m "feat: add offscreen audio capture pipeline"
```

## Task 4: Speech Provider Registry and Whisper Integration

**Files:**
- Create: `background/providers/speech/base.js`
- Create: `background/providers/speech/index.js`
- Create: `background/providers/speech/whisper.js`
- Create: `background/speech-service.js`
- Modify: `background/message-router.js`
- Create: `tests/background/whisper-provider.test.js`
- Test: `tests/background/whisper-provider.test.js`

- [ ] **Step 1: 先写 Whisper Provider 失败测试**

```js
// tests/background/whisper-provider.test.js
import { describe, expect, it, vi } from "vitest";
import { createWhisperProvider } from "../../background/providers/speech/whisper.js";

describe("createWhisperProvider", () => {
  it("向 OpenAI audio transcription 端点提交 form-data", async () => {
    const fetchImpl = vi.fn(async () => ({
      ok: true,
      json: async () => ({ text: "hello world", language: "en" })
    }));

    const provider = createWhisperProvider({ fetchImpl });

    const result = await provider.recognize({
      blob: new Blob(["demo"], { type: "audio/webm" }),
      apiKey: "test-key"
    });

    expect(fetchImpl).toHaveBeenCalledTimes(1);
    expect(result.text).toBe("hello world");
    expect(result.language).toBe("en");
  });
});
```

- [ ] **Step 2: 运行测试，确认 Provider 尚未实现**

Run: `npm run test -- tests/background/whisper-provider.test.js`

Expected: FAIL，错误包含 `Cannot find module '../../background/providers/speech/whisper.js'`

- [ ] **Step 3: 写语音识别 Provider 注册表和实现**

```js
// background/providers/speech/base.js
export function createSpeechProviderError(message, recoverable = false) {
  return { name: "SpeechProviderError", message, recoverable };
}
```

```js
// background/providers/speech/whisper.js
import { createSpeechProviderError } from "./base.js";

export function createWhisperProvider({ fetchImpl = fetch }) {
  return {
    async recognize({ blob, apiKey, model = "whisper-1" }) {
      const formData = new FormData();
      formData.append("model", model);
      formData.append("file", blob, "chunk.webm");

      const response = await fetchImpl("https://api.openai.com/v1/audio/transcriptions", {
        method: "POST",
        headers: {
          Authorization: `Bearer ${apiKey}`
        },
        body: formData
      });

      if (!response.ok) {
        throw createSpeechProviderError(`Whisper failed with ${response.status}`, response.status >= 500);
      }

      const payload = await response.json();
      return {
        text: payload.text ?? "",
        language: payload.language ?? "auto"
      };
    }
  };
}
```

```js
// background/providers/speech/index.js
import { createWhisperProvider } from "./whisper.js";

export function createSpeechRegistry(deps = {}) {
  return {
    whisper: createWhisperProvider(deps)
  };
}
```

```js
// background/speech-service.js
export function createSpeechService({ speechRegistry }) {
  return {
    async transcribeChunk({ config, blob, apiKey }) {
      const provider = speechRegistry[config.speechProvider];
      return provider.recognize({ blob, apiKey });
    }
  };
}
```

```js
// background/message-router.js
// 在已有 switch 中补充 AUDIO_CHUNK_READY 分支
case MESSAGE_TYPES.AUDIO_CHUNK_READY: {
  const transcript = await speechService.transcribeChunk({
    config: await getSyncConfig(),
    blob: message.payload.blob,
    apiKey: message.payload.apiKey
  });
  return { ok: true, transcript };
}
```

- [ ] **Step 4: 运行测试，确认 Whisper Provider 通过**

Run: `npm run test -- tests/background/whisper-provider.test.js`

Expected: PASS，请求构造和返回值解析测试通过

- [ ] **Step 5: 提交**

```bash
git add background tests/background
git commit -m "feat: add whisper speech provider"
```

## Task 5: Translation Registry and Deduplicated Translation Flow

**Files:**
- Create: `background/providers/translation/base.js`
- Create: `background/providers/translation/index.js`
- Create: `background/providers/translation/openai.js`
- Create: `background/translation-service.js`
- Modify: `background/message-router.js`
- Create: `tests/background/translation-service.test.js`
- Test: `tests/background/translation-service.test.js`

- [ ] **Step 1: 写翻译去重和 Provider 失败测试**

```js
// tests/background/translation-service.test.js
import { describe, expect, it, vi } from "vitest";
import { createTranslationService } from "../../background/translation-service.js";

describe("createTranslationService", () => {
  it("重复 transcript 时直接复用上一条翻译", async () => {
    const provider = { translate: vi.fn(async () => ({ translation: "你好" })) };
    const service = createTranslationService({
      translationRegistry: { openai: provider }
    });

    const first = await service.translate({
      config: { translationProvider: "openai", sourceLanguage: "en", targetLanguage: "zh-CN" },
      text: "hello"
    });
    const second = await service.translate({
      config: { translationProvider: "openai", sourceLanguage: "en", targetLanguage: "zh-CN" },
      text: "hello"
    });

    expect(first.translation).toBe("你好");
    expect(second.translation).toBe("你好");
    expect(provider.translate).toHaveBeenCalledTimes(1);
  });
});
```

- [ ] **Step 2: 运行测试，确认翻译服务尚未实现**

Run: `npm run test -- tests/background/translation-service.test.js`

Expected: FAIL，错误包含 `Cannot find module '../../background/translation-service.js'`

- [ ] **Step 3: 写翻译注册表和 OpenAI 实现**

```js
// background/providers/translation/base.js
export function buildTranslationPrompt({ text, sourceLanguage, targetLanguage, customPrompt }) {
  return [
    customPrompt || "You are a translation engine. Return translation only.",
    `Source language: ${sourceLanguage}`,
    `Target language: ${targetLanguage}`,
    `Text: ${text}`
  ].join("\n");
}
```

```js
// background/providers/translation/openai.js
import { buildTranslationPrompt } from "./base.js";

export function createOpenAITranslationProvider({ fetchImpl = fetch }) {
  return {
    async translate({ apiKey, text, sourceLanguage, targetLanguage, model, customPrompt }) {
      const response = await fetchImpl("https://api.openai.com/v1/chat/completions", {
        method: "POST",
        headers: {
          "Content-Type": "application/json",
          Authorization: `Bearer ${apiKey}`
        },
        body: JSON.stringify({
          model,
          messages: [
            {
              role: "system",
              content: buildTranslationPrompt({
                text,
                sourceLanguage,
                targetLanguage,
                customPrompt
              })
            }
          ]
        })
      });

      const payload = await response.json();
      return {
        translation: payload.choices?.[0]?.message?.content?.trim() ?? ""
      };
    }
  };
}
```

```js
// background/providers/translation/index.js
import { createOpenAITranslationProvider } from "./openai.js";

export function createTranslationRegistry(deps = {}) {
  return {
    openai: createOpenAITranslationProvider(deps)
  };
}
```

```js
// background/translation-service.js
export function createTranslationService({ translationRegistry }) {
  let lastInput = "";
  let lastOutput = "";

  return {
    async translate({ config, text, apiKey }) {
      if (text === lastInput) {
        return { translation: lastOutput };
      }

      const provider = translationRegistry[config.translationProvider];
      const result = await provider.translate({
        apiKey,
        text,
        sourceLanguage: config.sourceLanguage,
        targetLanguage: config.targetLanguage,
        model: config.llmModel,
        customPrompt: config.customPrompt
      });

      lastInput = text;
      lastOutput = result.translation;
      return result;
    }
  };
}
```

- [ ] **Step 4: 运行测试，确认翻译去重通过**

Run: `npm run test -- tests/background/translation-service.test.js`

Expected: PASS，重复文本只调用一次 Provider

- [ ] **Step 5: 提交**

```bash
git add background tests/background
git commit -m "feat: add extensible translation pipeline"
```

## Task 6: Popup and Overlay Result Rendering

**Files:**
- Modify: `popup/popup.html`
- Modify: `popup/popup.css`
- Modify: `popup/popup.js`
- Modify: `content/content.js`
- Create: `content/overlay.js`
- Create: `content/overlay.css`
- Create: `tests/shared/result-view-model.test.js`
- Create: `shared/result-view-model.js`
- Test: `tests/shared/result-view-model.test.js`

- [ ] **Step 1: 写结果视图模型失败测试**

```js
// tests/shared/result-view-model.test.js
import { describe, expect, it } from "vitest";
import { formatResultLine } from "../../shared/result-view-model.js";

describe("formatResultLine", () => {
  it("优先显示说话人标签，再显示翻译文本", () => {
    expect(formatResultLine({
      speakerLabel: "说话人 1",
      translation: "你好"
    })).toBe("[说话人 1] 你好");
  });
});
```

- [ ] **Step 2: 运行测试，确认视图模型未实现**

Run: `npm run test -- tests/shared/result-view-model.test.js`

Expected: FAIL，错误包含 `Cannot find module '../../shared/result-view-model.js'`

- [ ] **Step 3: 写 Popup/Overlay 统一结果渲染**

```js
// shared/result-view-model.js
export function formatResultLine(result) {
  const prefix = result.speakerLabel ? `[${result.speakerLabel}] ` : "";
  return `${prefix}${result.translation}`;
}
```

```html
<!-- popup/popup.html -->
<main class="popup">
  <header class="toolbar">
    <div id="status">空闲</div>
    <div class="actions">
      <button id="startButton">开始翻译</button>
      <button id="stopButton">停止翻译</button>
    </div>
  </header>
  <section class="settings">
    <label>
      Whisper Key
      <input id="whisperKey" type="password" />
    </label>
    <label>
      OpenAI Key
      <input id="openaiKey" type="password" />
    </label>
  </section>
  <section id="errorBox" class="error-box"></section>
  <section id="results" class="results"></section>
</main>
```

```css
/* popup/popup.css */
.popup {
  width: 360px;
  padding: 12px;
  font-family: "Segoe UI", sans-serif;
}

.results {
  max-height: 280px;
  overflow: auto;
}
```

```js
// popup/popup.js
import { MESSAGE_TYPES } from "../shared/messages.js";
import { formatResultLine } from "../shared/result-view-model.js";

function renderState(state) {
  document.querySelector("#status").textContent = state.status;
  document.querySelector("#results").innerHTML = state.results
    .map((item) => `<div class="result-line">${formatResultLine(item)}</div>`)
    .join("");
}

async function refreshState() {
  const state = await chrome.runtime.sendMessage({ type: MESSAGE_TYPES.GET_SESSION_STATE });
  renderState(state);
}

chrome.runtime.onMessage.addListener((message) => {
  if (message?.type === MESSAGE_TYPES.SESSION_STATE_UPDATED) {
    renderState(message.payload);
  }
});
```

```js
// content/overlay.js
import { formatResultLine } from "../shared/result-view-model.js";

export function createOverlay() {
  const root = document.createElement("div");
  root.id = "live-translator-overlay";
  root.innerHTML = `
    <div class="overlay-header">
      <span>Live Translator</span>
      <button data-close>×</button>
    </div>
    <div class="overlay-results"></div>
  `;
  document.body.appendChild(root);

  root.querySelector("[data-close]").addEventListener("click", () => {
    root.remove();
  });

  return {
    render(results) {
      root.querySelector(".overlay-results").innerHTML = results
        .map((item) => `<div class="overlay-line">${formatResultLine(item)}</div>`)
        .join("");
    }
  };
}
```

```css
/* content/overlay.css */
#live-translator-overlay {
  position: fixed;
  top: 24px;
  right: 24px;
  z-index: 2147483647;
  width: 320px;
  background: rgba(18, 18, 18, 0.92);
  color: #fff;
  border-radius: 12px;
  padding: 12px;
}
```

```js
// content/content.js
import { MESSAGE_TYPES } from "../shared/messages.js";
import { createOverlay } from "./overlay.js";

let overlay = null;

chrome.runtime.onMessage.addListener((message) => {
  if (message?.type === MESSAGE_TYPES.OVERLAY_RENDER) {
    overlay ??= createOverlay();
    overlay.render(message.payload.results);
  }
});
```

- [ ] **Step 4: 运行测试，确认结果格式化通过**

Run: `npm run test -- tests/shared/result-view-model.test.js`

Expected: PASS，结果展示文案格式测试通过

- [ ] **Step 5: 提交**

```bash
git add popup content shared tests/shared
git commit -m "feat: add popup and overlay rendering"
```

## Task 7: Config Persistence, Error Mapping, and Log Storage

**Files:**
- Modify: `shared/storage.js`
- Modify: `shared/logger.js`
- Create: `background/error-mapper.js`
- Modify: `popup/popup.js`
- Create: `tests/shared/logger.test.js`
- Create: `tests/background/error-mapper.test.js`
- Test: `tests/shared/logger.test.js`
- Test: `tests/background/error-mapper.test.js`

- [ ] **Step 1: 写错误映射和日志裁剪失败测试**

```js
// tests/background/error-mapper.test.js
import { describe, expect, it } from "vitest";
import { mapRuntimeError } from "../../background/error-mapper.js";

describe("mapRuntimeError", () => {
  it("把 provider 错误映射成人类可读消息", () => {
    expect(mapRuntimeError({ message: "Whisper failed with 401" })).toContain("语音识别");
  });
});
```

```js
// tests/shared/logger.test.js
import { describe, expect, it, vi } from "vitest";
import { appendErrorLog } from "../../shared/logger.js";

describe("appendErrorLog", () => {
  it("只保留最后 100 条错误", async () => {
    const set = vi.fn(async () => {});
    chrome.storage.local.get = vi.fn(async () => ({
      errorLogs: Array.from({ length: 100 }, (_, index) => ({ index }))
    }));
    chrome.storage.local.set = set;

    await appendErrorLog({ index: 100 });

    const payload = set.mock.calls[0][0];
    expect(payload.errorLogs).toHaveLength(100);
    expect(payload.errorLogs[0].index).toBe(1);
  });
});
```

- [ ] **Step 2: 运行测试，确认可靠性模块未实现完整**

Run: `npm run test -- tests/shared/logger.test.js tests/background/error-mapper.test.js`

Expected: FAIL，`mapRuntimeError` 缺失或行为不匹配

- [ ] **Step 3: 实现配置分层和错误展示**

```js
// shared/storage.js
export async function getSessionSecrets() {
  const stored = await chrome.storage.session.get("secrets");
  return stored.secrets ?? {};
}

export async function setSessionSecrets(secrets) {
  await chrome.storage.session.set({ secrets });
}
```

```js
// background/error-mapper.js
export function mapRuntimeError(error) {
  const message = error?.message ?? "";

  if (message.includes("Whisper")) {
    return "语音识别服务调用失败，请检查 API Key 或网络状态。";
  }

  if (message.includes("chat/completions")) {
    return "翻译服务调用失败，请检查模型配置或网络状态。";
  }

  return "翻译会话发生未知错误，请重试。";
}
```

```js
// popup/popup.js
function renderError(errorMessage) {
  document.querySelector("#errorBox").textContent = errorMessage || "";
}

chrome.runtime.onMessage.addListener((message) => {
  if (message?.type === MESSAGE_TYPES.SESSION_STATE_UPDATED) {
    renderState(message.payload);
    renderError(message.payload.lastError?.message ?? "");
  }
});
```

```js
// shared/logger.js
export async function appendErrorLog(entry) {
  const stored = await chrome.storage.local.get("errorLogs");
  const logs = Array.isArray(stored.errorLogs) ? stored.errorLogs : [];
  const nextLogs = [...logs, { ...entry, createdAt: Date.now() }].slice(-100);
  await chrome.storage.local.set({ errorLogs: nextLogs });
}
```

- [ ] **Step 4: 运行测试，确认可靠性基础通过**

Run: `npm run test -- tests/shared/logger.test.js tests/background/error-mapper.test.js`

Expected: PASS，日志裁剪和错误映射测试通过

- [ ] **Step 5: 提交**

```bash
git add background popup shared tests
git commit -m "feat: add config storage and error handling"
```

## Task 8: End-to-End Wiring and Manual Smoke Checklist

**Files:**
- Modify: `background/background.js`
- Modify: `background/message-router.js`
- Modify: `popup/popup.js`
- Modify: `content/content.js`
- Create: `docs/manual-smoke-checklist.md`
- Test: `docs/manual-smoke-checklist.md`

- [ ] **Step 1: 写 MVP 闭环所需的手工验证清单**

```md
# Manual Smoke Checklist

## 启动前准备

1. 在 Popup 中填入 Whisper 和 OpenAI API Key。
2. 打开 `chrome://extensions`，确认当前目录已重新加载。

## 基础流程

1. 点击“开始翻译”。
2. 在系统分享弹窗中选择带系统音频的屏幕或标签页。
3. 播放一段英文音频。
4. 确认 Popup 中出现转写后的中文结果。
5. 确认页面 Overlay 同步出现最新结果。
6. 点击“停止翻译”，确认状态恢复为空闲。

## 异常流程

1. 不填写 API Key 点击开始，确认出现提示。
2. 在分享弹窗中取消授权，确认出现错误提示。
3. 关闭 Overlay 后继续播放音频，确认 Popup 仍更新结果。
```

- [ ] **Step 2: 运行现有自动化测试，确认所有单测通过**

Run: `npm run test`

Expected: PASS，全部单元测试通过

- [ ] **Step 3: 完成主链路接线**

```js
// background/background.js
import { createSpeechRegistry } from "./providers/speech/index.js";
import { createTranslationRegistry } from "./providers/translation/index.js";
import { createSpeechService } from "./speech-service.js";
import { createTranslationService } from "./translation-service.js";

async function ensureOffscreenDocument() {
  const offscreenUrl = chrome.runtime.getURL("offscreen/offscreen.html");
  const contexts = await chrome.runtime.getContexts({
    contextTypes: ["OFFSCREEN_DOCUMENT"],
    documentUrls: [offscreenUrl]
  });

  if (contexts.length === 0) {
    await chrome.offscreen.createDocument({
      url: "offscreen/offscreen.html",
      reasons: ["USER_MEDIA"],
      justification: "Capture system audio for live translation"
    });
  }
}

const speechRegistry = createSpeechRegistry();
const translationRegistry = createTranslationRegistry();
const speechService = createSpeechService({ speechRegistry });
const translationService = createTranslationService({ translationRegistry });
const routeMessage = createMessageRouter({
  sessionManager,
  ensureOffscreenDocument,
  speechService,
  translationService
});
```

```js
// background/message-router.js
case MESSAGE_TYPES.START_SESSION: {
  const sessionId = `session-${Date.now()}`;
  await ensureOffscreenDocument();
  sessionManager.start();
  await chrome.runtime.sendMessage({
    type: MESSAGE_TYPES.START_CAPTURE,
    payload: { sessionId }
  });
  sessionManager.setStatus("capturing");
  return { ok: true, sessionId };
}

case MESSAGE_TYPES.STOP_SESSION: {
  await chrome.runtime.sendMessage({ type: MESSAGE_TYPES.STOP_CAPTURE });
  sessionManager.stop();
  return { ok: true };
}

case MESSAGE_TYPES.AUDIO_CHUNK_READY: {
  const config = await getSyncConfig();
  const secrets = await getSessionSecrets();
  const transcript = await speechService.transcribeChunk({
    config,
    blob: message.payload.blob,
    apiKey: secrets.whisper
  });

  if (!transcript.text) {
    return { ok: true, skipped: true };
  }

  const translation = await translationService.translate({
    config,
    text: transcript.text,
    apiKey: secrets.openai
  });

  const result = {
    id: `${message.payload.sessionId}:${message.payload.chunkIndex}`,
    createdAt: Date.now(),
    transcript: transcript.text,
    translation: translation.translation,
    sourceLanguage: transcript.language,
    targetLanguage: config.targetLanguage,
    speakerLabel: null
  };

  sessionManager.pushResult(result);

  await chrome.runtime.sendMessage({
    type: MESSAGE_TYPES.SESSION_STATE_UPDATED,
    payload: sessionManager.getState()
  });

  await chrome.tabs.query({ active: true, currentWindow: true }).then(([tab]) => {
    if (tab?.id) {
      return chrome.tabs.sendMessage(tab.id, {
        type: MESSAGE_TYPES.OVERLAY_RENDER,
        payload: { results: sessionManager.getState().results }
      });
    }
    return undefined;
  });

  return { ok: true };
}
```

```js
// popup/popup.js
import { setSessionSecrets } from "../shared/storage.js";

document.querySelector("#startButton").addEventListener("click", async () => {
  await setSessionSecrets({
    whisper: document.querySelector("#whisperKey")?.value ?? "",
    openai: document.querySelector("#openaiKey")?.value ?? ""
  });
  await chrome.runtime.sendMessage({ type: MESSAGE_TYPES.START_SESSION });
  await refreshState();
});
```

- [ ] **Step 4: 按清单做手工烟测**

Run: 按 `docs/manual-smoke-checklist.md` 从上到下执行

Expected:
1. 系统音频可以被捕获
2. Whisper 能产出文本
3. OpenAI 翻译能进入 Popup 和 Overlay
4. 取消授权、缺少 Key、关闭 Overlay 都有可理解反馈

- [ ] **Step 5: 提交**

```bash
git add background popup content docs/manual-smoke-checklist.md
git commit -m "chore: wire mvp flow and add smoke checklist"
```

## Self-Review Checklist

- 覆盖性：
  - 系统音频采集：Task 3、Task 8
  - 语音识别：Task 4
  - 文本翻译：Task 5
  - Popup/Overlay 展示：Task 2、Task 6、Task 8
  - 配置与错误处理：Task 1、Task 7、Task 8
  - 扩展性约束：Task 1、Task 4、Task 5

- 占位检查：
  - 计划中未使用 `TODO`、`TBD`、`later` 等占位词
  - 所有任务均给出明确文件路径、命令和最小代码示例

- 一致性检查：
  - 语音 Provider 键名统一为 `whisper`
  - 翻译 Provider 键名统一为 `openai`
  - 消息协议统一使用 `MESSAGE_TYPES`
  - 会话状态统一由 `session-manager` 维护

## Execution Handoff

Plan complete and saved to `docs/superpowers/plans/2026-04-10-live-translator-mvp.md`. Two execution options:

**1. Subagent-Driven (recommended)** - I dispatch a fresh subagent per task, review between tasks, fast iteration

**2. Inline Execution** - Execute tasks in this session using executing-plans, batch execution with checkpoints

Which approach?
