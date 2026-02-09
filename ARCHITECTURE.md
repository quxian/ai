# AI SDK 架构分析文档

> 本文档对 Vercel AI SDK 进行深度架构分析，涵盖核心模块的详细说明和整体设计思想。

## 目录

- [1. 项目概览](#1-项目概览)
- [2. 整体架构图](#2-整体架构图)
- [3. 核心依赖关系](#3-核心依赖关系)
- [4. 核心模块详细分析](#4-核心模块详细分析)
  - [4.1 packages/ai — 主 SDK 包](#41-packagesai--主-sdk-包)
  - [4.2 packages/provider — 提供者接口规范](#42-packagesprovider--提供者接口规范)
  - [4.3 packages/provider-utils — 提供者工具库](#43-packagesprovider-utils--提供者工具库)
  - [4.4 packages/gateway — 统一网关层](#44-packagesgateway--统一网关层)
  - [4.5 packages/mcp — Model Context Protocol](#45-packagesmcp--model-context-protocol)
- [5. AI 提供者模块分析](#5-ai-提供者模块分析)
  - [5.1 提供者架构模式](#51-提供者架构模式)
  - [5.2 主要提供者](#52-主要提供者)
- [6. 前端框架集成模块](#6-前端框架集成模块)
- [7. 次要模块概览](#7-次要模块概览)
- [8. 关键设计模式](#8-关键设计模式)
- [9. 数据流与处理管线](#9-数据流与处理管线)

---

## 1. 项目概览

Vercel AI SDK 是一个面向 TypeScript/JavaScript 的 AI 应用开发工具包，为大语言模型 (LLM) 及多模态 AI 服务提供统一的编程接口。项目采用 **pnpm monorepo** 架构，由 **Turborepo** 管理构建流程。

| 维度 | 说明 |
|------|------|
| **语言** | TypeScript 5.8+ |
| **运行时** | Node.js 18+ / Edge Runtime |
| **包管理** | pnpm 10+ workspaces |
| **构建工具** | tsup / esbuild + Turborepo |
| **测试框架** | Vitest（Node + Edge 双环境） |
| **支持的 AI 提供者** | 40+ 家（OpenAI、Anthropic、Google 等） |
| **前端框架集成** | React、Vue、Svelte、Angular |

---

## 2. 整体架构图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           应用层 (Application Layer)                        │
│                                                                             │
│   ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌────────────┐  │
│   │  React   │  │   Vue    │  │  Svelte  │  │ Angular  │  │    RSC     │  │
│   │  Hooks   │  │Composable│  │  Stores  │  │ Services │  │ (Server    │  │
│   │          │  │          │  │          │  │          │  │ Components)│  │
│   │ useChat  │  │ useChat  │  │  chat    │  │ inject   │  │            │  │
│   │ useObject│  │ useObject│  │ object   │  │          │  │            │  │
│   └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘  └─────┬──────┘  │
│        │             │             │             │              │           │
└────────┼─────────────┼─────────────┼─────────────┼──────────────┼───────────┘
         │             │             │             │              │
         └─────────────┴──────┬──────┴─────────────┴──────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         核心 SDK 层 (packages/ai)                           │
│                                                                             │
│  ┌─────────────┐ ┌─────────────┐ ┌──────────────┐ ┌───────────────────┐   │
│  │ generateText│ │generateObjec│ │generateImage │ │ generateSpeech/   │   │
│  │ streamText  │ │ streamObject│ │              │ │ generateVideo/    │   │
│  │             │ │             │ │              │ │ transcribe        │   │
│  └──────┬──────┘ └──────┬──────┘ └──────┬───────┘ └────────┬──────────┘   │
│         │               │               │                  │              │
│  ┌──────┴───────────────┴───────────────┴──────────────────┴──────────┐   │
│  │                        共享基础设施                                  │   │
│  │  ┌────────┐ ┌──────────┐ ┌──────┐ ┌─────────┐ ┌─────────────────┐ │   │
│  │  │ Agent  │ │Middleware│ │Prompt│ │Registry │ │   Telemetry     │ │   │
│  │  │ System │ │  Layer   │ │Engine│ │         │ │  (OpenTelemetry)│ │   │
│  │  └────────┘ └──────────┘ └──────┘ └─────────┘ └─────────────────┘ │   │
│  │  ┌──────────────┐ ┌──────────────────┐ ┌───────────────┐          │   │
│  │  │  UI/Chat     │ │  embed / rerank  │ │    Logger     │          │   │
│  │  │  Transport   │ │                  │ │               │          │   │
│  │  └──────────────┘ └──────────────────┘ └───────────────┘          │   │
│  └────────────────────────────────────────────────────────────────────┘   │
└────────────────────────────────────┬────────────────────────────────────────┘
                                     │
                    ┌────────────────┼────────────────┐
                    │                │                │
                    ▼                ▼                ▼
┌─────────────────────┐ ┌────────────────┐ ┌──────────────────┐
│  packages/gateway   │ │  packages/mcp  │ │  packages/       │
│  统一网关抽象层      │ │  MCP 协议客户端 │ │  provider-utils  │
│                     │ │                │ │  提供者实现工具    │
│ • 多模型统一接口    │ │ • 工具调用协议  │ │                  │
│ • Vercel 集成       │ │ • 多传输层支持  │ │ • API 调用封装    │
│ • 错误标准化        │ │ • OAuth2 认证   │ │ • Schema 验证     │
└─────────┬───────────┘ └────────┬───────┘ │ • 流式处理工具    │
          │                      │         └────────┬─────────┘
          │                      │                  │
          └──────────────────────┼──────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                    提供者接口层 (packages/provider)                          │
│                                                                             │
│   ┌──────────────┐ ┌──────────────┐ ┌───────────────┐ ┌────────────────┐   │
│   │LanguageModel │ │EmbeddingModel│ │  ImageModel   │ │  SpeechModel   │   │
│   │     V3       │ │     V2       │ │      V2       │ │      V1        │   │
│   └──────────────┘ └──────────────┘ └───────────────┘ └────────────────┘   │
│   ┌──────────────┐ ┌──────────────┐ ┌───────────────┐                      │
│   │  VideoModel  │ │Transcription │ │  Reranking    │                      │
│   │     V1       │ │   Model V1   │ │   Model V1   │                      │
│   └──────────────┘ └──────────────┘ └───────────────┘                      │
└────────────────────────────────────┬────────────────────────────────────────┘
                                     │
                                     ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                  AI 提供者实现层 (Provider Implementations)                  │
│                                                                             │
│  ┌─────────┐ ┌───────────┐ ┌────────┐ ┌───────┐ ┌─────────┐ ┌──────────┐ │
│  │ OpenAI  │ │ Anthropic │ │ Google │ │ Azure │ │ Amazon  │ │ Mistral  │ │
│  │         │ │           │ │        │ │       │ │ Bedrock │ │          │ │
│  └─────────┘ └───────────┘ └────────┘ └───────┘ └─────────┘ └──────────┘ │
│  ┌─────────┐ ┌───────────┐ ┌────────┐ ┌───────┐ ┌─────────┐ ┌──────────┐ │
│  │  Groq   │ │ DeepSeek  │ │Cohere  │ │  XAI  │ │Perplexty│ │ 30+ 更多 │ │
│  │         │ │           │ │        │ │       │ │         │ │ 提供者    │ │
│  └─────────┘ └───────────┘ └────────┘ └───────┘ └─────────┘ └──────────┘ │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. 核心依赖关系

```
                        ┌──────────────────────┐
                        │     packages/ai      │  ← 主入口，所有功能汇集于此
                        │   (npm: ai v6.x)     │
                        └──────────┬───────────┘
                                   │
               ┌───────────────────┼───────────────────┐
               │                   │                   │
               ▼                   ▼                   ▼
┌──────────────────────┐ ┌─────────────────┐ ┌──────────────────┐
│  packages/gateway    │ │ @ai-sdk/provider│ │@ai-sdk/provider  │
│  (@ai-sdk/gateway)   │ │  -utils         │ │                  │
│                      │ │                 │ │  接口定义层       │
│  统一网关            │ │  实现工具层     │ │                  │
└──────────┬───────────┘ └────────┬────────┘ └──────────────────┘
           │                      │                   ▲
           │                      │                   │
           └──────────────────────┴───────────────────┘
                                  │
                                  ▼
                    ┌──────────────────────────┐
                    │   各 AI 提供者实现        │
                    │   @ai-sdk/openai         │
                    │   @ai-sdk/anthropic      │
                    │   @ai-sdk/google         │
                    │   @ai-sdk/azure  ...     │
                    └──────────────────────────┘

依赖方向（从上到下）:
  ai ──▶ @ai-sdk/gateway
  ai ──▶ @ai-sdk/provider-utils ──▶ @ai-sdk/provider
  @ai-sdk/<provider> ──▶ @ai-sdk/provider-utils ──▶ @ai-sdk/provider
```

---

## 4. 核心模块详细分析

### 4.1 packages/ai — 主 SDK 包

> npm 包名: `ai`，版本 v6.x，是整个 SDK 的主入口。

#### 模块总览

```
packages/ai/src/
├── agent/              ← Agent 系统（工具循环、UI 流）
├── generate-text/      ← 文本生成（含流式）
├── generate-object/    ← 结构化输出生成（含流式）
├── generate-image/     ← 图片生成
├── generate-speech/    ← 语音合成
├── generate-video/     ← 视频生成
├── embed/              ← 向量嵌入
├── rerank/             ← 重排序
├── transcribe/         ← 语音转文字
├── prompt/             ← 提示词标准化引擎
├── middleware/          ← 中间件（模型包装器）
├── model/              ← V3 模型适配器
├── ui/                 ← Chat UI 工具与传输层
├── registry/           ← 模型注册中心
├── telemetry/          ← OpenTelemetry 遥测
├── logger/             ← 日志系统
├── types/              ← 全局类型定义
├── util/               ← 工具函数
└── index.ts            ← 公开 API 导出
```

#### 4.1.1 generate-text — 文本生成模块

**核心函数**: `generateText()` / `streamText()`

这是 SDK 中最重要的模块之一，负责与大语言模型交互生成文本。

```
generateText({
  model: openai('gpt-4o'),
  prompt: 'Hello!',
  tools: { ... },          // 可选工具调用
  maxSteps: 5,             // 多步推理
})
```

**关键能力**:
- 同步生成与流式生成两种模式
- 支持多步推理（multi-step reasoning）
- 内置工具调用（tool calling）支持
- Token 使用量追踪
- 丰富的元数据（响应时间、模型信息等）

#### 4.1.2 generate-object — 结构化输出模块

**核心函数**: `generateObject()` / `streamObject()`

将 LLM 输出约束为符合 JSON Schema / Zod Schema 的结构化数据。

```
generateObject({
  model: openai('gpt-4o'),
  schema: z.object({ name: z.string(), age: z.number() }),
  prompt: 'Generate a person',
})
```

**关键能力**:
- 基于 JSON Schema 或 Zod Schema 约束输出
- 支持 `json` / `tool` / `grammar` 三种模式
- 流式结构化输出（边生成边解析）
- 自动重试与校验

#### 4.1.3 agent — Agent 系统

**核心组件**: `agent()` / `toolLoopAgent()` / `createAgentUIStream()`

Agent 系统实现了具有自主决策能力的 AI 代理：

```
agent({
  model: openai('gpt-4o'),
  tools: { search, calculate },
  maxSteps: 10,
})
```

**关键能力**:
- **工具循环 (Tool Loop)**: Agent 可自主决定何时调用工具、处理结果、继续推理
- **多步执行**: 支持配置最大步数的迭代推理
- **UI 流集成**: `createAgentUIStream()` 将 Agent 执行过程实时流式传输到前端

#### 4.1.4 prompt — 提示词标准化引擎

负责将各种输入格式（字符串、消息数组、多模态内容）转换为各提供者理解的标准化提示词格式。

```
packages/ai/src/prompt/
├── convert-to-language-model-prompt.ts   ← 主转换逻辑
├── standardize-prompt.ts                 ← 输入标准化
├── prepare-tools-and-tool-choice.ts      ← 工具定义转换
├── data-content.ts                       ← 多模态内容处理
├── message-conversion-error.ts           ← 错误处理
└── ...
```

**关键能力**:
- 统一的消息格式转换（system/user/assistant/tool 角色）
- 多模态内容支持（文本、图片、音频、文件）
- 工具定义的标准化封装

#### 4.1.5 middleware — 中间件层

中间件层允许在不修改核心逻辑的情况下对模型行为进行包装和增强。

```
packages/ai/src/middleware/
├── wrap-language-model.ts            ← 语言模型包装器
├── wrap-provider.ts                  ← 提供者包装器
├── extract-json-middleware.ts        ← JSON 提取中间件
├── extract-reasoning-middleware.ts   ← 推理提取中间件
├── default-setting-middleware.ts     ← 默认设置中间件
├── simulate-streaming-middleware.ts  ← 模拟流式中间件
└── tool-example-middleware.ts        ← 工具示例中间件
```

**设计理念**: 采用装饰器模式，对模型的输入/输出进行透明拦截和处理，类似 Express/Koa 中间件。

#### 4.1.6 ui — Chat UI 与传输层

为前端 Chat 界面提供数据传输和消息管理能力。

```
packages/ai/src/ui/
├── chat-transport.ts                  ← 传输层接口
├── default-chat-transport.ts          ← 默认 HTTP 传输
├── direct-chat-transport.ts           ← 直接调用传输
├── text-stream-chat-transport.ts      ← 文本流传输
├── convert-to-ui-messages.ts          ← 消息格式转换
├── process-ui-messages.ts             ← 消息处理管线
└── ...
```

**关键能力**:
- 可插拔的传输层（HTTP/直接调用/文本流/SSE）
- UI 消息与内部消息格式互相转换
- 文件附件处理
- 消息验证与状态管理

#### 4.1.7 其他重要子模块

| 子模块 | 功能说明 |
|--------|---------|
| **embed/** | 调用嵌入模型生成向量表示，支持单个和批量嵌入 |
| **rerank/** | 对搜索结果进行重排序，支持 RAG 场景 |
| **transcribe/** | 语音转文字功能 |
| **model/** | V3 模型适配器，维护向后兼容性 |
| **registry/** | 模型注册中心，通过字符串 ID 查找和实例化模型 |
| **telemetry/** | OpenTelemetry 集成，用于分布式追踪和性能监控 |
| **logger/** | 统一的日志基础设施 |

---

### 4.2 packages/provider — 提供者接口规范

> npm 包名: `@ai-sdk/provider`

这是整个 SDK 的**类型基石**，定义了所有 AI 模型必须遵循的接口契约。

```
packages/provider/src/
├── language-model/
│   └── language-model-v3.ts        ← LanguageModelV3 接口（核心）
├── embedding-model/
│   └── embedding-model-v2.ts       ← EmbeddingModelV2
├── image-model/
│   └── image-model-v2.ts           ← ImageModelV2
├── speech-model/
│   └── speech-model-v1.ts          ← SpeechModelV1
├── video-model/
│   └── video-model-v1.ts           ← VideoModelV1
├── transcription-model/
│   └── transcription-model-v1.ts   ← TranscriptionModelV1
├── reranking-model/
│   └── reranking-model-v1.ts       ← RerankingModelV1
├── language-model-middleware.ts     ← 中间件接口
├── provider.ts                     ← Provider 基础接口
└── shared/                         ← 共享类型（JSON Schema、错误等）
```

#### 核心接口说明

**LanguageModelV3** — 最重要的接口：

```typescript
interface LanguageModelV3 {
  // 模型标识
  readonly specificationVersion: 'v3';
  readonly provider: string;
  readonly modelId: string;

  // 核心方法
  doGenerate(options: LanguageModelV3CallOptions): Promise<LanguageModelV3GenerateResult>;
  doStream(options: LanguageModelV3CallOptions): Promise<LanguageModelV3StreamResult>;
}
```

**设计原则**:
- 接口版本化（V1/V2/V3），支持平滑升级
- 最小化接口，只定义"做什么"，不限定"怎么做"
- 所有模型类型遵循相同的 `doGenerate` / `doStream` 双方法模式

---

### 4.3 packages/provider-utils — 提供者工具库

> npm 包名: `@ai-sdk/provider-utils`

为所有提供者实现提供开箱即用的工具函数，避免重复造轮子。

```
packages/provider-utils/src/
├── post-to-api.ts                   ← HTTP POST 请求封装
├── get-from-api.ts                  ← HTTP GET 请求封装
├── handle-fetch-error.ts            ← 请求错误统一处理
├── parse-json-event-stream.ts       ← JSON 事件流解析（SSE）
├── schema.ts                        ← JSON Schema 工具
├── validate-types.ts                ← 运行时类型验证
├── create-tool-name-mapping.ts      ← 工具名称映射
├── provider-tool-factory.ts         ← 工具工厂模式
├── data-uri-to-uint8array.ts        ← 数据 URI 转换
├── download-blob.ts                 ← Blob 下载
├── async-iterator-to-stream.ts      ← 异步迭代器 → 流
├── detect-media-type.ts             ← 媒体类型检测
├── generate-id.ts                   ← ID 生成
└── ...（40+ 工具文件）
```

#### 核心工具分类

| 分类 | 主要工具 | 用途 |
|------|---------|------|
| **网络请求** | `postToApi`, `getFromApi`, `handleFetchError` | 统一的 API 调用与错误处理 |
| **流式处理** | `parseJsonEventStream`, `asyncIteratorToStream` | SSE 事件流解析与转换 |
| **Schema 验证** | `schema`, `validateTypes` | JSON Schema 定义与运行时校验 |
| **工具系统** | `createToolNameMapping`, `providerToolFactory` | 跨提供者工具名称映射 |
| **数据转换** | `dataUriToUint8Array`, `downloadBlob`, `detectMediaType` | 多模态数据处理 |

---

### 4.4 packages/gateway — 统一网关层

> npm 包名: `@ai-sdk/gateway`

Gateway 是一个关键的抽象层，提供跨 40+ 提供者的统一访问接口。

```
packages/gateway/src/
├── gateway-provider.ts              ← 网关提供者工厂
├── gateway-language-model.ts        ← 统一语言模型接口
├── gateway-embedding-model.ts       ← 统一嵌入模型接口
├── gateway-image-model.ts           ← 统一图片模型接口
├── gateway-video-model.ts           ← 统一视频模型接口
├── gateway-*-settings.ts            ← 各模型类型的配置
├── gateway-error.ts                 ← 标准化错误处理
└── ...
```

**核心价值**:
- 通过 `gateway('openai/gpt-4o')` 形式的字符串 ID 选择模型
- 自动路由到正确的提供者实现
- 统一的认证和错误处理
- 与 Vercel 平台深度集成

```typescript
import { gateway } from 'ai';

// 一行代码切换不同提供者的模型
const result = await generateText({
  model: gateway('openai/gpt-4o'),       // OpenAI
  // model: gateway('anthropic/claude-3'), // 或 Anthropic
  prompt: 'Hello!',
});
```

---

### 4.5 packages/mcp — Model Context Protocol

> npm 包名: `@ai-sdk/mcp`

实现了 Model Context Protocol (MCP) 客户端，使 AI SDK 能够与 MCP 服务器交互获取工具。

```
packages/mcp/src/
├── mcp-client.ts                    ← MCP 客户端主逻辑
├── mcp-http-transport.ts            ← HTTP 传输层
├── mcp-sse-transport.ts             ← SSE 传输层
├── mcp-stdio/                       ← 标准输入输出传输层
├── oauth.ts                         ← OAuth2 认证
└── json-rpc-message.ts              ← JSON-RPC 协议消息
```

**核心能力**:
- 连接 MCP 服务器获取可用工具列表
- 支持 HTTP、SSE、Stdio 三种传输协议
- 内置 OAuth2 认证流程
- 将 MCP 工具自动转换为 AI SDK 工具格式

---

## 5. AI 提供者模块分析

### 5.1 提供者架构模式

每个提供者都遵循统一的实现模式：

```
packages/<provider>/src/
├── <provider>-provider.ts           ← 提供者工厂（入口）
├── <provider>-language-model.ts     ← 语言模型实现
├── <provider>-embedding-model.ts    ← 嵌入模型实现（可选）
├── <provider>-image-model.ts        ← 图片模型实现（可选）
├── <provider>-error.ts              ← 错误定义
├── <provider>-prepare-tools.ts      ← 工具转换
├── internal/                        ← 内部工具
└── index.ts                         ← 公开导出
```

**工厂模式示例 (OpenAI)**:

```typescript
import { createOpenAI } from '@ai-sdk/openai';

// 创建提供者实例
const openai = createOpenAI({ apiKey: '...' });

// 获取具体模型
const model = openai('gpt-4o');          // 语言模型
const embedder = openai.embedding('text-embedding-3-small');  // 嵌入模型
const imageGen = openai.image('dall-e-3'); // 图片模型
```

### 5.2 主要提供者

#### OpenAI (`@ai-sdk/openai`)

功能最完整的提供者实现，支持所有模型类型：

| 模型类型 | 实现 | 示例模型 |
|---------|------|---------|
| 语言模型 | Chat + Completion | gpt-4o, gpt-4o-mini, o1 |
| 嵌入模型 | ✅ | text-embedding-3-small/large |
| 图片模型 | ✅ | dall-e-3 |
| 语音合成 | ✅ | tts-1, tts-1-hd |
| 语音识别 | ✅ | whisper-1 |

**特殊能力**: 工具调用、JSON 模式、Vision、函数调用兼容层

#### Anthropic (`@ai-sdk/anthropic`)

以 Messages API 为基础的实现：

| 特性 | 说明 |
|------|------|
| 语言模型 | Claude 3.5 Sonnet, Claude 3 Opus/Haiku |
| 缓存控制 | 支持 Anthropic 独有的 Cache Control |
| 工具调用 | 完整的工具调用支持 |
| 长上下文 | 支持 200K token 窗口 |

#### Google (`@ai-sdk/google`)

多模态能力最丰富的实现：

| 模型类型 | 说明 |
|---------|------|
| 语言模型 | Gemini Pro, Gemini Ultra |
| 嵌入模型 | text-embedding-004 |
| 图片模型 | Imagen |
| 视频模型 | Veo |

**特殊能力**: 原生多模态输入、文件 URL 处理、Google Cloud 集成

#### 其他提供者速览

| 提供者 | 特点 |
|--------|------|
| **Azure** | 基于 OpenAI 兼容层，企业级部署支持 |
| **Amazon Bedrock** | AWS 生态集成，多模型选择 |
| **Mistral** | 开源模型，高性价比 |
| **Groq** | 超低延迟推理 |
| **DeepSeek** | 深度推理模型 |
| **Cohere** | 强力嵌入与 RAG 能力 |
| **XAI** | Grok 系列模型 |
| **Perplexity** | 搜索增强的语言模型 |

---

## 6. 前端框架集成模块

各框架集成遵循**适配器模式**，核心逻辑共享，UI 层适配各框架：

```
                    ┌────────────────────┐
                    │  packages/ai/ui/   │ ← 共享的 Chat/Transport 逻辑
                    └─────────┬──────────┘
                              │
          ┌───────────────────┼───────────────────┐
          │                   │                   │
          ▼                   ▼                   ▼
┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
│  packages/react │ │  packages/vue   │ │ packages/svelte │
│                 │ │                 │ │                 │
│ • useChat()     │ │ • useChat()     │ │ • chat store    │
│ • useCompletion │ │ • useCompletion │ │ • completion    │
│ • useObject()   │ │ • useObject()   │ │   store         │
│                 │ │                 │ │ • object store  │
│ React Hooks     │ │ Vue Composables │ │ Svelte Stores   │
└─────────────────┘ └─────────────────┘ └─────────────────┘
```

### React (`@ai-sdk/react`)

最成熟的框架集成，基于 React Hooks：

```typescript
import { useChat } from '@ai-sdk/react';

function ChatComponent() {
  const { messages, input, handleSubmit } = useChat({
    api: '/api/chat',
  });
  // ...
}
```

### Vue (`@ai-sdk/vue`)

基于 Vue 3 Composition API：

```typescript
import { useChat } from '@ai-sdk/vue';

const { messages, input, handleSubmit } = useChat({
  api: '/api/chat',
});
```

### Svelte (`@ai-sdk/svelte`)

基于 Svelte 响应式 Store：

```typescript
import { chat } from '@ai-sdk/svelte';

const chatStore = chat({ api: '/api/chat' });
```

---

## 7. 次要模块概览

| 包名 | 用途 | 说明 |
|------|------|------|
| `@ai-sdk/angular` | Angular 服务 | Angular 框架集成 |
| `@ai-sdk/rsc` | React Server Components | RSC 支持 |
| `@ai-sdk/langchain` | LangChain 适配器 | 与 LangChain 生态互通 |
| `@ai-sdk/llamaindex` | LlamaIndex 适配器 | 与 LlamaIndex 生态互通 |
| `@ai-sdk/valibot` | Valibot Schema | Zod 之外的 Schema 方案 |
| `@ai-sdk/openai-compatible` | OpenAI 兼容层 | 快速接入 OpenAI 兼容 API |
| `@ai-sdk/elevenlabs` | 语音合成 | ElevenLabs TTS |
| `@ai-sdk/deepgram` | 语音识别 | Deepgram STT |
| `@ai-sdk/assemblyai` | 语音识别 | AssemblyAI STT |
| `@ai-sdk/black-forest-labs` | 图片生成 | FLUX 模型 |
| `@ai-sdk/fal` | 图片/视频 | fal.ai 平台集成 |
| `@ai-sdk/luma` | 视频生成 | Luma AI 视频 |
| `@ai-sdk/hume` | 语音合成 | Hume AI 情感语音 |
| `@ai-sdk/devtools` | 开发者工具 | 调试与可视化 |
| `@ai-sdk/codemod` | 代码迁移 | 版本升级自动化 |
| `@ai-sdk/test-server` | 测试服务器 | 模拟 AI 服务 |

---

## 8. 关键设计模式

### 8.1 提供者工厂模式 (Provider Factory)

```
createOpenAI({ apiKey }) ──▶ openai('gpt-4o') ──▶ LanguageModelV3 实例
                           ──▶ openai.embedding('text-embedding-3-small')
                           ──▶ openai.image('dall-e-3')
```

每个提供者通过工厂函数创建，返回一个可调用对象：
- 直接调用返回语言模型
- 通过命名方法访问其他模型类型

### 8.2 中间件装饰器模式

```
原始模型 ──▶ [默认设置中间件] ──▶ [推理提取中间件] ──▶ [JSON 提取中间件] ──▶ 增强模型
```

通过 `wrapLanguageModel()` 实现透明的模型行为增强，不修改原始模型代码。

### 8.3 传输层抽象

```
                  ┌────────────────────────┐
                  │   ChatTransport 接口   │
                  └──────────┬─────────────┘
                             │
         ┌───────────────────┼───────────────────┐
         │                   │                   │
         ▼                   ▼                   ▼
┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
│  HTTP Transport │ │ Direct Transport│ │ TextStream      │
│  (fetch API)    │ │ (无网络，本地)  │ │  Transport      │
└─────────────────┘ └─────────────────┘ └─────────────────┘
```

Chat UI 通过可插拔传输层解耦前端展示与后端通信。

### 8.4 错误标记模式 (Error Marker Pattern)

```typescript
// 所有错误继承 AISDKError，使用 Symbol 标记实现跨包 instanceof 检查
class MyError extends AISDKError {
  private readonly [Symbol.for('vercel.ai.error.AI_MyError')] = true;

  static isInstance(error: unknown): error is MyError {
    return AISDKError.hasMarker(error, 'vercel.ai.error.AI_MyError');
  }
}
```

解决了 JavaScript 中跨包 `instanceof` 不可靠的问题。

### 8.5 版本化接口策略

```
LanguageModelV1 → LanguageModelV2 → LanguageModelV3（当前）
EmbeddingModelV1 → EmbeddingModelV2（当前）
ImageModelV1 → ImageModelV2（当前）
```

通过版本化接口实现平滑的 API 演进，旧版本通过适配器保持兼容。

---

## 9. 数据流与处理管线

### 9.1 generateText 完整数据流

```
用户调用 generateText()
        │
        ▼
┌─────────────────────────┐
│ 1. 标准化提示词          │  prompt/ 模块
│    string → Message[]   │
│    附加系统指令          │
└───────────┬─────────────┘
            │
            ▼
┌─────────────────────────┐
│ 2. 中间件处理            │  middleware/ 模块
│    应用默认设置          │
│    注入工具示例          │
└───────────┬─────────────┘
            │
            ▼
┌─────────────────────────┐
│ 3. 遥测记录              │  telemetry/ 模块
│    创建 OpenTelemetry    │
│    span                  │
└───────────┬─────────────┘
            │
            ▼
┌─────────────────────────┐
│ 4. 调用提供者            │  provider 实现
│    model.doGenerate()    │
│    HTTP 请求 → AI API    │
└───────────┬─────────────┘
            │
            ▼
┌─────────────────────────┐
│ 5. 处理工具调用          │  若模型要求调用工具
│    执行工具函数          │
│    结果加入上下文        │◄──── 循环直到无工具调用
│    回到步骤 4            │      或达到 maxSteps
└───────────┬─────────────┘
            │
            ▼
┌─────────────────────────┐
│ 6. 返回结果              │
│    text, toolResults,   │
│    usage, metadata      │
└─────────────────────────┘
```

### 9.2 Chat UI 数据流

```
                    前端 (React/Vue/Svelte)
                              │
                    useChat / chat store
                              │
                              ▼
                    ┌──────────────────┐
                    │  ChatTransport   │
                    │  (HTTP/SSE)      │
                    └────────┬─────────┘
                             │
                    POST /api/chat
                             │
                             ▼
                    ┌──────────────────┐
                    │  服务端 Handler  │
                    │  streamText()    │
                    └────────┬─────────┘
                             │
                    流式响应 (ReadableStream)
                             │
                             ▼
                    ┌──────────────────┐
                    │  前端消息处理    │
                    │  processUIMessages│
                    │  convertToUIMsg  │
                    └────────┬─────────┘
                             │
                             ▼
                    UI 更新 (响应式渲染)
```

---

## 附录：仓库文件结构概览

```
ai/
├── packages/
│   ├── ai/                    ★ 主 SDK 包 — 核心入口
│   ├── provider/              ★ 接口规范 — 类型基石
│   ├── provider-utils/        ★ 提供者工具 — 共享实现
│   ├── gateway/               ★ 统一网关 — 多提供者抽象
│   ├── mcp/                   ★ MCP 客户端 — 工具协议
│   ├── openai/                ● 主要提供者
│   ├── anthropic/             ● 主要提供者
│   ├── google/                ● 主要提供者
│   ├── azure/                 ● 主要提供者
│   ├── amazon-bedrock/        ● 主要提供者
│   ├── react/                 ● 前端框架集成
│   ├── vue/                   ● 前端框架集成
│   ├── svelte/                ● 前端框架集成
│   ├── angular/               ○ 次要模块
│   ├── rsc/                   ○ 次要模块
│   ├── devtools/              ○ 次要模块
│   ├── codemod/               ○ 次要模块
│   └── ...（30+ 其他提供者）  ○ 次要模块
├── examples/                  示例项目
├── content/                   文档源文件 (MDX)
├── contributing/              贡献者指南
├── tools/                     内部工具（eslint/tsconfig）
└── turbo.json                 Turborepo 构建配置
```

**图例**: ★ 核心模块 | ● 重要模块 | ○ 次要模块

---

*本文档基于 AI SDK v6.x 架构分析生成。*
