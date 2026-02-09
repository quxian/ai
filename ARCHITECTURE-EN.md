# AI SDK Architecture Analysis

> A deep-dive architecture analysis of the Vercel AI SDK, covering core module details and overall design philosophy.
>
> Last updated: 2026-02-09 | Version: v6.x

## Quick Summary

The Vercel AI SDK is a TypeScript/JavaScript toolkit for building AI-powered applications. It uses a **pnpm monorepo** managed by **Turborepo**.

### Four-Layer Architecture

```
┌─────────────────────────────────────────┐
│  Application Layer                      │
│  (React, Vue, Svelte, Angular, RSC)     │
├─────────────────────────────────────────┤
│  Core SDK Layer (packages/ai)           │
│  generateText, streamText, agent, etc.  │
├─────────────────────────────────────────┤
│  Provider Interface (packages/provider) │
│  LanguageModelV3, EmbeddingModelV2, ... │
├─────────────────────────────────────────┤
│  Provider Implementations (40+)         │
│  OpenAI, Anthropic, Google, Azure, ...  │
└─────────────────────────────────────────┘
```

For the full detailed analysis in Chinese, see [ARCHITECTURE.md](./ARCHITECTURE.md).
