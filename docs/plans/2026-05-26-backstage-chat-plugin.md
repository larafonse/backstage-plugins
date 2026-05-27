# Backstage Chat Plugin Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build an open-source Backstage plugin that adds an AI-powered floating chat widget, backed by Claude/OpenAI, that can query and execute MCP Actions.

**Architecture:** Two plugins in a standalone monorepo — `@larafonse/plugin-chat-backend` (Express router + LLM orchestration + `actionsServiceRef`) and `@larafonse/plugin-chat` (React floating widget with SSE streaming). The backend is a standalone Backstage plugin (`createBackendPlugin`) that injects `actionsServiceRef` from `@backstage/backend-plugin-api/alpha` to list and invoke MCP actions with the user's own credentials. The frontend is a `createFrontendPlugin` with a component extension that renders a floating chat bubble on every page.

**Tech Stack:** TypeScript, `@backstage/backend-plugin-api` (1.9.x), `@backstage/frontend-plugin-api`, `@anthropic-ai/sdk`, `openai`, React 18, Material UI 4, SSE streaming

---

## File Map

### Monorepo Root
- **Create:** `package.json` — Yarn workspaces root, scripts for build/test/lint
- **Create:** `tsconfig.json` — Extends `@backstage/cli/config/tsconfig.json`
- **Create:** `.gitignore` — Node, dist, coverage

### Backend Plugin (`plugins/chat-backend/`)
- **Create:** `package.json` — `@larafonse/plugin-chat-backend`, role: `backend-plugin`
- **Create:** `tsconfig.json` — Extends `@backstage/cli/config/tsconfig.json`
- **Create:** `src/index.ts` — Re-exports plugin default
- **Create:** `src/plugin.ts` — `createBackendPlugin` with deps, registers router
- **Create:** `src/router.ts` — Express router, `POST /v1/messages` endpoint with SSE streaming
- **Create:** `src/router.test.ts` — Router integration tests
- **Create:** `src/providers/types.ts` — `ChatProvider` interface, shared types (`ChatMessage`, `ChatEvent`, `ToolDefinition`)
- **Create:** `src/providers/anthropic.ts` — Anthropic `ChatProvider` implementation
- **Create:** `src/providers/anthropic.test.ts` — Anthropic provider unit tests
- **Create:** `src/providers/openai.ts` — OpenAI `ChatProvider` implementation
- **Create:** `src/providers/openai.test.ts` — OpenAI provider unit tests
- **Create:** `src/providers/registry.ts` — Provider factory from config
- **Create:** `src/providers/registry.test.ts` — Registry unit tests
- **Create:** `src/orchestrator.ts` — Tool call loop, confirmation logic, action schema conversion
- **Create:** `src/orchestrator.test.ts` — Orchestrator unit tests

### Frontend Plugin (`plugins/chat/`)
- **Create:** `package.json` — `@larafonse/plugin-chat`, role: `frontend-plugin`
- **Create:** `tsconfig.json` — Extends `@backstage/cli/config/tsconfig.json`
- **Create:** `src/index.ts` — Re-exports plugin
- **Create:** `src/plugin.ts` — `createFrontendPlugin` with component extension
- **Create:** `src/api/ChatApi.ts` — API ref + interface
- **Create:** `src/api/ChatClient.ts` — SSE fetch client implementing the API
- **Create:** `src/hooks/useChat.ts` — State management hook (messages, streaming, confirmation)
- **Create:** `src/hooks/useChat.test.ts` — Hook tests
- **Create:** `src/components/ChatWidget.tsx` — Floating button + panel toggle
- **Create:** `src/components/ChatPanel.tsx` — Panel container with header + message list + input
- **Create:** `src/components/MessageList.tsx` — Scrollable message list
- **Create:** `src/components/MessageBubble.tsx` — Individual message rendering with markdown
- **Create:** `src/components/ToolCallCard.tsx` — Action execution indicator
- **Create:** `src/components/ConfirmCard.tsx` — Mutation confirmation UI
- **Create:** `src/components/ChatInput.tsx` — Text input with send button

---

## Task 1: Monorepo Scaffolding

**Files:**
- Create: `package.json`
- Create: `tsconfig.json`
- Create: `.gitignore`
- Create: `plugins/chat-backend/package.json`
- Create: `plugins/chat-backend/tsconfig.json`
- Create: `plugins/chat/package.json`
- Create: `plugins/chat/tsconfig.json`

- [ ] **Step 1: Create root `package.json`**

```json
{
  "name": "backstage-plugins",
  "version": "1.0.0",
  "private": true,
  "engines": {
    "node": "22 || 24"
  },
  "scripts": {
    "tsc": "tsc",
    "build": "backstage-cli repo build --all",
    "test": "backstage-cli repo test",
    "lint": "backstage-cli repo lint",
    "clean": "backstage-cli repo clean"
  },
  "workspaces": [
    "plugins/*"
  ],
  "devDependencies": {
    "@backstage/cli": "^0.36.2",
    "typescript": "~5.8.0"
  },
  "packageManager": "yarn@4.4.1"
}
```

- [ ] **Step 2: Create root `tsconfig.json`**

```json
{
  "extends": "@backstage/cli/config/tsconfig.json",
  "include": [
    "plugins/*/src",
    "plugins/*/config.d.ts"
  ],
  "exclude": ["node_modules"],
  "compilerOptions": {
    "outDir": "dist-types",
    "rootDir": ".",
    "jsx": "react-jsx"
  }
}
```

- [ ] **Step 3: Create `.gitignore`**

```
node_modules/
dist/
dist-types/
.yarn/cache
.yarn/install-state.gz
coverage/
*.tsbuildinfo
```

- [ ] **Step 4: Create `plugins/chat-backend/package.json`**

```json
{
  "name": "@larafonse/plugin-chat-backend",
  "version": "0.1.0",
  "main": "src/index.ts",
  "types": "src/index.ts",
  "license": "Apache-2.0",
  "backstage": {
    "role": "backend-plugin",
    "pluginId": "chat"
  },
  "scripts": {
    "start": "backstage-cli package start",
    "build": "backstage-cli package build",
    "test": "backstage-cli package test",
    "lint": "backstage-cli package lint",
    "clean": "backstage-cli package clean"
  },
  "dependencies": {
    "@backstage/backend-plugin-api": "^1.9.0",
    "@backstage/config": "^1.3.0",
    "@backstage/errors": "^1.3.0",
    "@anthropic-ai/sdk": "^0.39.0",
    "openai": "^4.80.0",
    "express": "^4.22.0",
    "express-promise-router": "^4.1.0"
  },
  "devDependencies": {
    "@backstage/backend-test-utils": "^1.3.0",
    "@backstage/cli": "^0.36.2",
    "@types/express": "^4.17.6"
  }
}
```

- [ ] **Step 5: Create `plugins/chat-backend/tsconfig.json`**

```json
{
  "extends": "@backstage/cli/config/tsconfig.json",
  "include": ["src"],
  "exclude": ["node_modules"],
  "compilerOptions": {
    "outDir": "dist",
    "rootDir": "src"
  }
}
```

- [ ] **Step 6: Create `plugins/chat/package.json`**

```json
{
  "name": "@larafonse/plugin-chat",
  "version": "0.1.0",
  "main": "src/index.ts",
  "types": "src/index.ts",
  "license": "Apache-2.0",
  "backstage": {
    "role": "frontend-plugin",
    "pluginId": "chat"
  },
  "scripts": {
    "start": "backstage-cli package start",
    "build": "backstage-cli package build",
    "test": "backstage-cli package test",
    "lint": "backstage-cli package lint",
    "clean": "backstage-cli package clean"
  },
  "dependencies": {
    "@backstage/core-components": "^0.18.0",
    "@backstage/core-plugin-api": "^1.12.0",
    "@backstage/frontend-plugin-api": "^0.17.0",
    "@backstage/theme": "^0.7.0",
    "@material-ui/core": "^4.12.2",
    "@material-ui/icons": "^4.9.1",
    "react-markdown": "^9.0.0",
    "react": "^18.0.2",
    "react-dom": "^18.0.2"
  },
  "devDependencies": {
    "@backstage/cli": "^0.36.2",
    "@backstage/frontend-test-utils": "^0.6.0",
    "@testing-library/react": "^14.0.0",
    "@testing-library/jest-dom": "^6.0.0",
    "@testing-library/user-event": "^14.0.0",
    "@types/react": "^18",
    "@types/react-dom": "^18"
  },
  "peerDependencies": {
    "react": "^18.0.2",
    "react-dom": "^18.0.2"
  }
}
```

- [ ] **Step 7: Install dependencies**

Run: `cd /Users/nicolaslara/dev/backstage-plugins && yarn install`

- [ ] **Step 8: Verify TypeScript compiles**

Run: `cd /Users/nicolaslara/dev/backstage-plugins && yarn tsc`
Expected: No errors (no source files yet, just verifying config)

- [ ] **Step 9: Commit**

```bash
git add package.json tsconfig.json .gitignore plugins/chat-backend/package.json plugins/chat-backend/tsconfig.json plugins/chat/package.json plugins/chat/tsconfig.json yarn.lock .yarnrc.yml .yarn/
git commit -m "chore: scaffold monorepo with chat and chat-backend plugin packages"
```

---

## Task 2: Backend — Shared Types (`providers/types.ts`)

**Files:**
- Create: `plugins/chat-backend/src/providers/types.ts`

- [ ] **Step 1: Write the types file**

This file defines the core interfaces shared across the backend: the LLM provider contract, message types, and streaming event types.

```typescript
// plugins/chat-backend/src/providers/types.ts
import { JsonObject, JsonValue } from '@backstage/types';

export type ChatRole = 'user' | 'assistant' | 'tool';

export type ChatMessage =
  | { role: 'user'; content: string }
  | { role: 'assistant'; content: string; toolCalls?: ToolCall[] }
  | { role: 'tool'; toolCallId: string; content: string };

export interface ToolCall {
  id: string;
  name: string;
  input: JsonObject;
}

export interface ToolDefinition {
  name: string;
  description: string;
  inputSchema: JsonObject;
}

export type ChatStreamEvent =
  | { type: 'text'; content: string }
  | { type: 'tool_call'; id: string; name: string; input: JsonObject }
  | { type: 'done' };

export interface ChatProvider {
  chat(options: {
    messages: ChatMessage[];
    tools: ToolDefinition[];
    systemPrompt?: string;
  }): AsyncIterable<ChatStreamEvent>;
}
```

- [ ] **Step 2: Commit**

```bash
git add plugins/chat-backend/src/providers/types.ts
git commit -m "feat(chat-backend): add shared types for ChatProvider, messages, and streaming events"
```

---

## Task 3: Backend — Provider Registry

**Files:**
- Create: `plugins/chat-backend/src/providers/registry.ts`
- Create: `plugins/chat-backend/src/providers/registry.test.ts`

- [ ] **Step 1: Write the failing test**

```typescript
// plugins/chat-backend/src/providers/registry.test.ts
import { createProviderFromConfig } from './registry';
import { ChatProvider } from './types';
import { ConfigReader } from '@backstage/config';

describe('createProviderFromConfig', () => {
  it('throws if provider name is not recognized', () => {
    const config = new ConfigReader({
      chat: {
        provider: {
          name: 'unknown-provider',
          config: {},
        },
      },
    });

    expect(() => createProviderFromConfig(config)).toThrow(
      "Unknown chat provider: 'unknown-provider'. Supported providers: anthropic, openai",
    );
  });

  it('creates an anthropic provider when configured', () => {
    const config = new ConfigReader({
      chat: {
        provider: {
          name: 'anthropic',
          config: {
            apiKey: 'test-key',
            model: 'claude-sonnet-4-20250514',
          },
        },
      },
    });

    const provider = createProviderFromConfig(config);
    expect(provider).toBeDefined();
  });

  it('creates an openai provider when configured', () => {
    const config = new ConfigReader({
      chat: {
        provider: {
          name: 'openai',
          config: {
            apiKey: 'test-key',
            model: 'gpt-4o',
          },
        },
      },
    });

    const provider = createProviderFromConfig(config);
    expect(provider).toBeDefined();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd /Users/nicolaslara/dev/backstage-plugins && yarn backstage-cli repo test --testPathPattern=registry.test`
Expected: FAIL — module `./registry` not found

- [ ] **Step 3: Write the registry implementation**

```typescript
// plugins/chat-backend/src/providers/registry.ts
import { Config } from '@backstage/config';
import { ChatProvider } from './types';
import { AnthropicProvider } from './anthropic';
import { OpenAIProvider } from './openai';

const SUPPORTED_PROVIDERS = ['anthropic', 'openai'] as const;

export function createProviderFromConfig(config: Config): ChatProvider {
  const providerName = config.getString('chat.provider.name');
  const providerConfig = config.getConfig('chat.provider.config');

  switch (providerName) {
    case 'anthropic':
      return new AnthropicProvider({
        apiKey: providerConfig.getString('apiKey'),
        model: providerConfig.getOptionalString('model') ?? 'claude-sonnet-4-20250514',
      });
    case 'openai':
      return new OpenAIProvider({
        apiKey: providerConfig.getString('apiKey'),
        model: providerConfig.getOptionalString('model') ?? 'gpt-4o',
      });
    default:
      throw new Error(
        `Unknown chat provider: '${providerName}'. Supported providers: ${SUPPORTED_PROVIDERS.join(', ')}`,
      );
  }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd /Users/nicolaslara/dev/backstage-plugins && yarn backstage-cli repo test --testPathPattern=registry.test`
Expected: PASS (after anthropic.ts and openai.ts exist — these are stubs for now, created in the next tasks)

> **Note:** This test will fail until Tasks 4 and 5 create the `AnthropicProvider` and `OpenAIProvider` classes. That's expected. Come back and run this test after Task 5.

- [ ] **Step 5: Commit**

```bash
git add plugins/chat-backend/src/providers/registry.ts plugins/chat-backend/src/providers/registry.test.ts
git commit -m "feat(chat-backend): add provider registry with config-based creation"
```

---

## Task 4: Backend — Anthropic Provider

**Files:**
- Create: `plugins/chat-backend/src/providers/anthropic.ts`
- Create: `plugins/chat-backend/src/providers/anthropic.test.ts`

- [ ] **Step 1: Write the failing test**

```typescript
// plugins/chat-backend/src/providers/anthropic.test.ts
import { AnthropicProvider } from './anthropic';
import { ChatMessage, ToolDefinition, ChatStreamEvent } from './types';

// We test the provider against a mock Anthropic client
// to verify message/tool conversion logic without hitting the API.

describe('AnthropicProvider', () => {
  it('streams text events from the LLM response', async () => {
    const provider = new AnthropicProvider({
      apiKey: 'test-key',
      model: 'claude-sonnet-4-20250514',
    });

    // Mock the internal client
    const mockStream = createMockAnthropicStream([
      { type: 'content_block_delta', index: 0, delta: { type: 'text_delta', text: 'Hello' } },
      { type: 'content_block_delta', index: 0, delta: { type: 'text_delta', text: ' world' } },
      { type: 'message_stop' },
    ]);
    jest.spyOn(provider as any, 'createStream').mockReturnValue(mockStream);

    const messages: ChatMessage[] = [{ role: 'user', content: 'Hi' }];
    const events: ChatStreamEvent[] = [];

    for await (const event of provider.chat({ messages, tools: [] })) {
      events.push(event);
    }

    expect(events).toEqual([
      { type: 'text', content: 'Hello' },
      { type: 'text', content: ' world' },
      { type: 'done' },
    ]);
  });

  it('emits tool_call events when LLM requests a tool', async () => {
    const provider = new AnthropicProvider({
      apiKey: 'test-key',
      model: 'claude-sonnet-4-20250514',
    });

    const mockStream = createMockAnthropicStream([
      {
        type: 'content_block_start',
        index: 0,
        content_block: {
          type: 'tool_use',
          id: 'tool_1',
          name: 'catalog.query-catalog-entities',
          input: {},
        },
      },
      {
        type: 'content_block_delta',
        index: 0,
        delta: {
          type: 'input_json_delta',
          partial_json: '{"filter":{"kind":"Component"}}',
        },
      },
      {
        type: 'content_block_stop',
        index: 0,
      },
      { type: 'message_stop' },
    ]);
    jest.spyOn(provider as any, 'createStream').mockReturnValue(mockStream);

    const messages: ChatMessage[] = [
      { role: 'user', content: 'List all components' },
    ];
    const tools: ToolDefinition[] = [
      {
        name: 'catalog.query-catalog-entities',
        description: 'Query catalog entities',
        inputSchema: { type: 'object', properties: { filter: { type: 'object' } } },
      },
    ];

    const events: ChatStreamEvent[] = [];
    for await (const event of provider.chat({ messages, tools })) {
      events.push(event);
    }

    expect(events).toEqual([
      {
        type: 'tool_call',
        id: 'tool_1',
        name: 'catalog.query-catalog-entities',
        input: { filter: { kind: 'Component' } },
      },
      { type: 'done' },
    ]);
  });
});

// Helper to create an async iterable that mimics the Anthropic streaming API
async function* createMockAnthropicStream(events: any[]) {
  for (const event of events) {
    yield event;
  }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd /Users/nicolaslara/dev/backstage-plugins && yarn backstage-cli repo test --testPathPattern=anthropic.test`
Expected: FAIL — module `./anthropic` not found

- [ ] **Step 3: Write the Anthropic provider**

```typescript
// plugins/chat-backend/src/providers/anthropic.ts
import Anthropic from '@anthropic-ai/sdk';
import {
  ChatProvider,
  ChatMessage,
  ChatStreamEvent,
  ToolDefinition,
  ToolCall,
} from './types';

export class AnthropicProvider implements ChatProvider {
  private readonly client: Anthropic;
  private readonly model: string;

  constructor(options: { apiKey: string; model: string }) {
    this.client = new Anthropic({ apiKey: options.apiKey });
    this.model = options.model;
  }

  async *chat(options: {
    messages: ChatMessage[];
    tools: ToolDefinition[];
    systemPrompt?: string;
  }): AsyncIterable<ChatStreamEvent> {
    const anthropicMessages = options.messages.map(msg =>
      this.toAnthropicMessage(msg),
    );

    const anthropicTools = options.tools.map(tool => ({
      name: tool.name,
      description: tool.description,
      input_schema: tool.inputSchema as Anthropic.Tool['input_schema'],
    }));

    const stream = this.createStream({
      model: this.model,
      max_tokens: 4096,
      system: options.systemPrompt,
      messages: anthropicMessages,
      tools: anthropicTools.length > 0 ? anthropicTools : undefined,
    });

    // Track in-progress tool use blocks
    const toolBlocks = new Map<
      number,
      { id: string; name: string; jsonChunks: string[] }
    >();

    for await (const event of stream) {
      if (
        event.type === 'content_block_delta' &&
        event.delta.type === 'text_delta'
      ) {
        yield { type: 'text', content: event.delta.text };
      }

      if (
        event.type === 'content_block_start' &&
        event.content_block.type === 'tool_use'
      ) {
        toolBlocks.set(event.index, {
          id: event.content_block.id,
          name: event.content_block.name,
          jsonChunks: [],
        });
      }

      if (
        event.type === 'content_block_delta' &&
        event.delta.type === 'input_json_delta'
      ) {
        const block = toolBlocks.get(event.index);
        if (block) {
          block.jsonChunks.push(event.delta.partial_json);
        }
      }

      if (event.type === 'content_block_stop') {
        const block = toolBlocks.get(event.index);
        if (block) {
          const inputJson = block.jsonChunks.join('');
          const input = inputJson ? JSON.parse(inputJson) : {};
          yield {
            type: 'tool_call',
            id: block.id,
            name: block.name,
            input,
          };
          toolBlocks.delete(event.index);
        }
      }

      if (event.type === 'message_stop') {
        yield { type: 'done' };
      }
    }
  }

  protected createStream(params: {
    model: string;
    max_tokens: number;
    system?: string;
    messages: Anthropic.MessageParam[];
    tools?: Anthropic.Tool[];
  }): AsyncIterable<any> {
    return this.client.messages.stream({
      ...params,
      stream: true,
    });
  }

  private toAnthropicMessage(msg: ChatMessage): Anthropic.MessageParam {
    if (msg.role === 'user') {
      return { role: 'user', content: msg.content };
    }
    if (msg.role === 'tool') {
      return {
        role: 'user',
        content: [
          {
            type: 'tool_result',
            tool_use_id: msg.toolCallId,
            content: msg.content,
          },
        ],
      };
    }
    // assistant
    const content: Anthropic.ContentBlockParam[] = [];
    if (msg.content) {
      content.push({ type: 'text', text: msg.content });
    }
    if (msg.toolCalls) {
      for (const tc of msg.toolCalls) {
        content.push({
          type: 'tool_use',
          id: tc.id,
          name: tc.name,
          input: tc.input,
        });
      }
    }
    return { role: 'assistant', content };
  }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd /Users/nicolaslara/dev/backstage-plugins && yarn backstage-cli repo test --testPathPattern=anthropic.test`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add plugins/chat-backend/src/providers/anthropic.ts plugins/chat-backend/src/providers/anthropic.test.ts
git commit -m "feat(chat-backend): add Anthropic ChatProvider with streaming and tool-use"
```

---

## Task 5: Backend — OpenAI Provider

**Files:**
- Create: `plugins/chat-backend/src/providers/openai.ts`
- Create: `plugins/chat-backend/src/providers/openai.test.ts`

- [ ] **Step 1: Write the failing test**

```typescript
// plugins/chat-backend/src/providers/openai.test.ts
import { OpenAIProvider } from './openai';
import { ChatMessage, ToolDefinition, ChatStreamEvent } from './types';

describe('OpenAIProvider', () => {
  it('streams text events from the LLM response', async () => {
    const provider = new OpenAIProvider({
      apiKey: 'test-key',
      model: 'gpt-4o',
    });

    const mockStream = createMockOpenAIStream([
      { choices: [{ delta: { content: 'Hello' }, index: 0 }] },
      { choices: [{ delta: { content: ' world' }, index: 0 }] },
    ]);
    jest.spyOn(provider as any, 'createStream').mockReturnValue(mockStream);

    const messages: ChatMessage[] = [{ role: 'user', content: 'Hi' }];
    const events: ChatStreamEvent[] = [];

    for await (const event of provider.chat({ messages, tools: [] })) {
      events.push(event);
    }

    expect(events).toEqual([
      { type: 'text', content: 'Hello' },
      { type: 'text', content: ' world' },
      { type: 'done' },
    ]);
  });

  it('emits tool_call events when LLM requests a tool', async () => {
    const provider = new OpenAIProvider({
      apiKey: 'test-key',
      model: 'gpt-4o',
    });

    const mockStream = createMockOpenAIStream([
      {
        choices: [
          {
            delta: {
              tool_calls: [
                {
                  index: 0,
                  id: 'call_1',
                  type: 'function',
                  function: {
                    name: 'catalog.query-catalog-entities',
                    arguments: '',
                  },
                },
              ],
            },
            index: 0,
          },
        ],
      },
      {
        choices: [
          {
            delta: {
              tool_calls: [
                {
                  index: 0,
                  function: { arguments: '{"filter":{"kind":"Component"}}' },
                },
              ],
            },
            index: 0,
          },
        ],
      },
    ]);
    jest.spyOn(provider as any, 'createStream').mockReturnValue(mockStream);

    const messages: ChatMessage[] = [
      { role: 'user', content: 'List all components' },
    ];
    const tools: ToolDefinition[] = [
      {
        name: 'catalog.query-catalog-entities',
        description: 'Query catalog entities',
        inputSchema: { type: 'object', properties: { filter: { type: 'object' } } },
      },
    ];

    const events: ChatStreamEvent[] = [];
    for await (const event of provider.chat({ messages, tools })) {
      events.push(event);
    }

    expect(events).toEqual([
      {
        type: 'tool_call',
        id: 'call_1',
        name: 'catalog.query-catalog-entities',
        input: { filter: { kind: 'Component' } },
      },
      { type: 'done' },
    ]);
  });
});

async function* createMockOpenAIStream(chunks: any[]) {
  for (const chunk of chunks) {
    yield chunk;
  }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd /Users/nicolaslara/dev/backstage-plugins && yarn backstage-cli repo test --testPathPattern=openai.test`
Expected: FAIL — module `./openai` not found

- [ ] **Step 3: Write the OpenAI provider**

```typescript
// plugins/chat-backend/src/providers/openai.ts
import OpenAI from 'openai';
import {
  ChatProvider,
  ChatMessage,
  ChatStreamEvent,
  ToolDefinition,
} from './types';

export class OpenAIProvider implements ChatProvider {
  private readonly client: OpenAI;
  private readonly model: string;

  constructor(options: { apiKey: string; model: string }) {
    this.client = new OpenAI({ apiKey: options.apiKey });
    this.model = options.model;
  }

  async *chat(options: {
    messages: ChatMessage[];
    tools: ToolDefinition[];
    systemPrompt?: string;
  }): AsyncIterable<ChatStreamEvent> {
    const openaiMessages: OpenAI.ChatCompletionMessageParam[] = [];

    if (options.systemPrompt) {
      openaiMessages.push({ role: 'system', content: options.systemPrompt });
    }

    for (const msg of options.messages) {
      openaiMessages.push(this.toOpenAIMessage(msg));
    }

    const openaiTools: OpenAI.ChatCompletionTool[] = options.tools.map(
      tool => ({
        type: 'function' as const,
        function: {
          name: tool.name,
          description: tool.description,
          parameters: tool.inputSchema,
        },
      }),
    );

    const stream = this.createStream({
      model: this.model,
      messages: openaiMessages,
      tools: openaiTools.length > 0 ? openaiTools : undefined,
      stream: true,
    });

    // Track in-progress tool calls
    const toolCalls = new Map<
      number,
      { id: string; name: string; argumentChunks: string[] }
    >();

    for await (const chunk of stream) {
      const choice = chunk.choices[0];
      if (!choice) continue;

      const delta = choice.delta;

      if (delta.content) {
        yield { type: 'text', content: delta.content };
      }

      if (delta.tool_calls) {
        for (const tc of delta.tool_calls) {
          if (tc.id) {
            // New tool call starting
            toolCalls.set(tc.index, {
              id: tc.id,
              name: tc.function?.name ?? '',
              argumentChunks: [],
            });
          }
          if (tc.function?.arguments) {
            const existing = toolCalls.get(tc.index);
            if (existing) {
              existing.argumentChunks.push(tc.function.arguments);
            }
          }
        }
      }
    }

    // Emit accumulated tool calls
    for (const [, tc] of toolCalls) {
      const args = tc.argumentChunks.join('');
      yield {
        type: 'tool_call',
        id: tc.id,
        name: tc.name,
        input: args ? JSON.parse(args) : {},
      };
    }

    yield { type: 'done' };
  }

  protected createStream(
    params: OpenAI.ChatCompletionCreateParamsStreaming,
  ): AsyncIterable<OpenAI.ChatCompletionChunk> {
    return this.client.chat.completions.create(params) as any;
  }

  private toOpenAIMessage(
    msg: ChatMessage,
  ): OpenAI.ChatCompletionMessageParam {
    if (msg.role === 'user') {
      return { role: 'user', content: msg.content };
    }
    if (msg.role === 'tool') {
      return {
        role: 'tool',
        tool_call_id: msg.toolCallId,
        content: msg.content,
      };
    }
    // assistant
    const result: OpenAI.ChatCompletionAssistantMessageParam = {
      role: 'assistant',
      content: msg.content || null,
    };
    if (msg.toolCalls && msg.toolCalls.length > 0) {
      result.tool_calls = msg.toolCalls.map(tc => ({
        id: tc.id,
        type: 'function' as const,
        function: {
          name: tc.name,
          arguments: JSON.stringify(tc.input),
        },
      }));
    }
    return result;
  }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd /Users/nicolaslara/dev/backstage-plugins && yarn backstage-cli repo test --testPathPattern=openai.test`
Expected: PASS

- [ ] **Step 5: Run registry tests now that both providers exist**

Run: `cd /Users/nicolaslara/dev/backstage-plugins && yarn backstage-cli repo test --testPathPattern=registry.test`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add plugins/chat-backend/src/providers/openai.ts plugins/chat-backend/src/providers/openai.test.ts
git commit -m "feat(chat-backend): add OpenAI ChatProvider with streaming and tool-use"
```

---

## Task 6: Backend — Orchestrator

**Files:**
- Create: `plugins/chat-backend/src/orchestrator.ts`
- Create: `plugins/chat-backend/src/orchestrator.test.ts`

The orchestrator handles the core loop: converting MCP actions to LLM tools, calling the provider, executing read-only tool calls, and flagging mutating actions for confirmation.

- [ ] **Step 1: Write the failing test**

```typescript
// plugins/chat-backend/src/orchestrator.test.ts
import { Orchestrator, OrchestratorEvent } from './orchestrator';
import { ChatProvider, ChatStreamEvent, ToolDefinition } from './providers/types';

function createMockProvider(events: ChatStreamEvent[]): ChatProvider {
  return {
    async *chat() {
      for (const e of events) {
        yield e;
      }
    },
  };
}

const readOnlyAction = {
  id: 'catalog.query-catalog-entities',
  pluginId: 'catalog',
  name: 'query-catalog-entities',
  title: 'Query Catalog Entities',
  description: 'Query entities in the catalog',
  schema: {
    input: { type: 'object' as const, properties: { filter: { type: 'object' } } },
    output: { type: 'object' as const },
  },
  examples: [],
  attributes: { readOnly: true, destructive: false, idempotent: true },
};

const mutatingAction = {
  id: 'scaffolder.execute-template',
  pluginId: 'scaffolder',
  name: 'execute-template',
  title: 'Execute Template',
  description: 'Execute a scaffolder template',
  schema: {
    input: { type: 'object' as const, properties: { templateRef: { type: 'string' } } },
    output: { type: 'object' as const },
  },
  examples: [],
  attributes: { readOnly: false, destructive: false, idempotent: false },
};

describe('Orchestrator', () => {
  it('converts MCP actions to tool definitions', () => {
    const orchestrator = new Orchestrator({
      provider: createMockProvider([]),
      actionMode: 'auto',
    });

    const tools = orchestrator.actionsToTools([readOnlyAction]);
    expect(tools).toEqual([
      {
        name: 'catalog.query-catalog-entities',
        description: 'Query entities in the catalog',
        inputSchema: readOnlyAction.schema.input,
      },
    ]);
  });

  it('streams text through without modification', async () => {
    const provider = createMockProvider([
      { type: 'text', content: 'Hello' },
      { type: 'done' },
    ]);

    const orchestrator = new Orchestrator({ provider, actionMode: 'auto' });
    const events: OrchestratorEvent[] = [];

    for await (const e of orchestrator.run({
      messages: [{ role: 'user', content: 'Hi' }],
      actions: [],
      invokeAction: async () => ({ output: {} }),
    })) {
      events.push(e);
    }

    expect(events).toEqual([
      { type: 'text', content: 'Hello' },
      { type: 'done' },
    ]);
  });

  it('auto-executes read-only tool calls', async () => {
    const provider = createMockProvider([
      { type: 'tool_call', id: 'tc1', name: 'catalog.query-catalog-entities', input: { filter: { kind: 'Component' } } },
      { type: 'done' },
    ]);

    const orchestrator = new Orchestrator({ provider, actionMode: 'auto' });
    const invokeAction = jest.fn().mockResolvedValue({ output: { items: ['service-a'] } });

    const events: OrchestratorEvent[] = [];
    for await (const e of orchestrator.run({
      messages: [{ role: 'user', content: 'List components' }],
      actions: [readOnlyAction],
      invokeAction,
    })) {
      events.push(e);
    }

    expect(invokeAction).toHaveBeenCalledWith('catalog.query-catalog-entities', { filter: { kind: 'Component' } });
    expect(events).toContainEqual(
      expect.objectContaining({ type: 'tool_call_start', name: 'catalog.query-catalog-entities' }),
    );
    expect(events).toContainEqual(
      expect.objectContaining({ type: 'tool_call_result', name: 'catalog.query-catalog-entities' }),
    );
  });

  it('requires confirmation for mutating tool calls', async () => {
    const provider = createMockProvider([
      { type: 'tool_call', id: 'tc1', name: 'scaffolder.execute-template', input: { templateRef: 'template:default/node' } },
      { type: 'done' },
    ]);

    const orchestrator = new Orchestrator({ provider, actionMode: 'auto' });
    const invokeAction = jest.fn();

    const events: OrchestratorEvent[] = [];
    for await (const e of orchestrator.run({
      messages: [{ role: 'user', content: 'Create a node service' }],
      actions: [mutatingAction],
      invokeAction,
    })) {
      events.push(e);
    }

    expect(invokeAction).not.toHaveBeenCalled();
    expect(events).toContainEqual(
      expect.objectContaining({
        type: 'confirmation_required',
        actionId: 'scaffolder.execute-template',
        input: { templateRef: 'template:default/node' },
      }),
    );
  });

  it('respects allowlist filtering', () => {
    const orchestrator = new Orchestrator({
      provider: createMockProvider([]),
      actionMode: 'allowlist',
      allowlist: ['catalog.query-catalog-entities'],
    });

    const tools = orchestrator.actionsToTools([readOnlyAction, mutatingAction]);
    expect(tools).toHaveLength(1);
    expect(tools[0].name).toBe('catalog.query-catalog-entities');
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd /Users/nicolaslara/dev/backstage-plugins && yarn backstage-cli repo test --testPathPattern=orchestrator.test`
Expected: FAIL — module `./orchestrator` not found

- [ ] **Step 3: Write the orchestrator**

```typescript
// plugins/chat-backend/src/orchestrator.ts
import { JsonObject, JsonValue } from '@backstage/types';
import {
  ChatProvider,
  ChatMessage,
  ChatStreamEvent,
  ToolDefinition,
  ToolCall,
} from './providers/types';

interface ActionInfo {
  id: string;
  pluginId: string;
  name: string;
  title: string;
  description: string;
  schema: { input: JsonObject; output: JsonObject };
  attributes: { readOnly: boolean; destructive: boolean; idempotent: boolean };
}

export type OrchestratorEvent =
  | { type: 'text'; content: string }
  | { type: 'tool_call_start'; name: string; description: string }
  | { type: 'tool_call_result'; name: string; output: JsonValue }
  | {
      type: 'confirmation_required';
      toolCallId: string;
      actionId: string;
      actionTitle: string;
      actionDescription: string;
      input: JsonObject;
    }
  | { type: 'error'; message: string }
  | { type: 'done' };

export class Orchestrator {
  private readonly provider: ChatProvider;
  private readonly actionMode: 'auto' | 'allowlist';
  private readonly allowlist: string[];

  constructor(options: {
    provider: ChatProvider;
    actionMode: 'auto' | 'allowlist';
    allowlist?: string[];
  }) {
    this.provider = options.provider;
    this.actionMode = options.actionMode;
    this.allowlist = options.allowlist ?? [];
  }

  actionsToTools(actions: ActionInfo[]): ToolDefinition[] {
    const filtered =
      this.actionMode === 'allowlist'
        ? actions.filter(a => this.allowlist.includes(a.id))
        : actions;

    return filtered.map(action => ({
      name: action.id,
      description: action.description,
      inputSchema: action.schema.input,
    }));
  }

  async *run(options: {
    messages: ChatMessage[];
    actions: ActionInfo[];
    invokeAction: (id: string, input: JsonObject) => Promise<{ output: JsonValue }>;
    systemPrompt?: string;
  }): AsyncIterable<OrchestratorEvent> {
    const tools = this.actionsToTools(options.actions);
    const actionMap = new Map(options.actions.map(a => [a.id, a]));

    const pendingToolCalls: ToolCall[] = [];

    for await (const event of this.provider.chat({
      messages: options.messages,
      tools,
      systemPrompt: options.systemPrompt,
    })) {
      if (event.type === 'text') {
        yield { type: 'text', content: event.content };
      }

      if (event.type === 'tool_call') {
        pendingToolCalls.push({
          id: event.id,
          name: event.name,
          input: event.input,
        });
      }

      if (event.type === 'done') {
        // Process tool calls
        for (const tc of pendingToolCalls) {
          const action = actionMap.get(tc.name);
          if (!action) {
            yield { type: 'error', message: `Unknown action: ${tc.name}` };
            continue;
          }

          if (action.attributes.readOnly) {
            // Auto-execute read-only actions
            yield {
              type: 'tool_call_start',
              name: tc.name,
              description: action.description,
            };
            try {
              const result = await options.invokeAction(tc.name, tc.input);
              yield {
                type: 'tool_call_result',
                name: tc.name,
                output: result.output,
              };
            } catch (error) {
              yield {
                type: 'error',
                message: `Action ${tc.name} failed: ${error}`,
              };
            }
          } else {
            // Require confirmation for mutating actions
            yield {
              type: 'confirmation_required',
              toolCallId: tc.id,
              actionId: tc.name,
              actionTitle: action.title,
              actionDescription: action.description,
              input: tc.input,
            };
          }
        }

        yield { type: 'done' };
      }
    }
  }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd /Users/nicolaslara/dev/backstage-plugins && yarn backstage-cli repo test --testPathPattern=orchestrator.test`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add plugins/chat-backend/src/orchestrator.ts plugins/chat-backend/src/orchestrator.test.ts
git commit -m "feat(chat-backend): add orchestrator with tool-call execution and mutation confirmation"
```

---

## Task 7: Backend — Router + Plugin Registration

**Files:**
- Create: `plugins/chat-backend/src/router.ts`
- Create: `plugins/chat-backend/src/router.test.ts`
- Create: `plugins/chat-backend/src/plugin.ts`
- Create: `plugins/chat-backend/src/index.ts`

- [ ] **Step 1: Write the failing test**

```typescript
// plugins/chat-backend/src/router.test.ts
import { createRouter } from './router';
import express from 'express';
import request from 'supertest';
import { ChatProvider, ChatStreamEvent } from './providers/types';
import { JsonObject, JsonValue } from '@backstage/types';

function createMockProvider(events: ChatStreamEvent[]): ChatProvider {
  return {
    async *chat() {
      for (const e of events) {
        yield e;
      }
    },
  };
}

describe('createRouter', () => {
  it('returns SSE stream for a chat message', async () => {
    const provider = createMockProvider([
      { type: 'text', content: 'Hello!' },
      { type: 'done' },
    ]);

    const router = createRouter({
      provider,
      listActions: async () => [],
      invokeAction: async () => ({ output: {} }),
      systemPrompt: 'You are a test assistant',
      actionMode: 'auto',
    });

    const app = express();
    app.use(express.json());
    app.use(router);

    const response = await request(app)
      .post('/v1/messages')
      .send({
        messages: [{ role: 'user', content: 'Hi' }],
      })
      .expect(200)
      .expect('Content-Type', /text\/event-stream/);

    const lines = response.text.split('\n').filter(l => l.startsWith('data:'));
    const events = lines.map(l => JSON.parse(l.replace('data: ', '')));

    expect(events).toContainEqual({ type: 'text', content: 'Hello!' });
    expect(events).toContainEqual({ type: 'done' });
  });

  it('returns 400 if messages array is missing', async () => {
    const provider = createMockProvider([]);

    const router = createRouter({
      provider,
      listActions: async () => [],
      invokeAction: async () => ({ output: {} }),
      systemPrompt: undefined,
      actionMode: 'auto',
    });

    const app = express();
    app.use(express.json());
    app.use(router);

    await request(app)
      .post('/v1/messages')
      .send({})
      .expect(400);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd /Users/nicolaslara/dev/backstage-plugins && yarn backstage-cli repo test --testPathPattern=router.test`
Expected: FAIL — module `./router` not found

- [ ] **Step 3: Write the router**

```typescript
// plugins/chat-backend/src/router.ts
import { Router } from 'express';
import { ChatProvider, ChatMessage } from './providers/types';
import { Orchestrator, OrchestratorEvent } from './orchestrator';
import { JsonObject, JsonValue } from '@backstage/types';

interface ActionInfo {
  id: string;
  pluginId: string;
  name: string;
  title: string;
  description: string;
  schema: { input: JsonObject; output: JsonObject };
  attributes: { readOnly: boolean; destructive: boolean; idempotent: boolean };
}

export interface RouterOptions {
  provider: ChatProvider;
  listActions: () => Promise<ActionInfo[]>;
  invokeAction: (id: string, input: JsonObject) => Promise<{ output: JsonValue }>;
  systemPrompt?: string;
  actionMode: 'auto' | 'allowlist';
  allowlist?: string[];
}

export function createRouter(options: RouterOptions): Router {
  const router = Router();

  const orchestrator = new Orchestrator({
    provider: options.provider,
    actionMode: options.actionMode,
    allowlist: options.allowlist,
  });

  router.post('/v1/messages', async (req, res) => {
    const { messages, confirmedAction } = req.body as {
      messages?: ChatMessage[];
      confirmedAction?: { actionId: string; toolCallId: string; input: JsonObject } | null;
    };

    if (!messages || !Array.isArray(messages)) {
      res.status(400).json({ error: 'messages array is required' });
      return;
    }

    res.setHeader('Content-Type', 'text/event-stream');
    res.setHeader('Cache-Control', 'no-cache');
    res.setHeader('Connection', 'keep-alive');
    res.flushHeaders();

    try {
      // If a confirmed action is provided, execute it directly
      if (confirmedAction) {
        const sendEvent = (event: OrchestratorEvent) => {
          res.write(`data: ${JSON.stringify(event)}\n\n`);
        };

        sendEvent({
          type: 'tool_call_start',
          name: confirmedAction.actionId,
          description: 'Executing confirmed action...',
        });

        try {
          const result = await options.invokeAction(
            confirmedAction.actionId,
            confirmedAction.input,
          );
          sendEvent({
            type: 'tool_call_result',
            name: confirmedAction.actionId,
            output: result.output,
          });
        } catch (error) {
          sendEvent({
            type: 'error',
            message: `Action failed: ${error}`,
          });
        }

        sendEvent({ type: 'done' });
        res.end();
        return;
      }

      const actions = await options.listActions();

      for await (const event of orchestrator.run({
        messages,
        actions,
        invokeAction: options.invokeAction,
        systemPrompt: options.systemPrompt,
      })) {
        res.write(`data: ${JSON.stringify(event)}\n\n`);
      }
    } catch (error) {
      res.write(
        `data: ${JSON.stringify({ type: 'error', message: String(error) })}\n\n`,
      );
      res.write(`data: ${JSON.stringify({ type: 'done' })}\n\n`);
    }

    res.end();
  });

  return router;
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd /Users/nicolaslara/dev/backstage-plugins && yarn backstage-cli repo test --testPathPattern=router.test`
Expected: PASS

- [ ] **Step 5: Write the plugin registration**

```typescript
// plugins/chat-backend/src/plugin.ts
import {
  createBackendPlugin,
  coreServices,
} from '@backstage/backend-plugin-api';
import { actionsServiceRef } from '@backstage/backend-plugin-api/alpha';
import { createProviderFromConfig } from './providers/registry';
import { createRouter } from './router';

export const chatPlugin = createBackendPlugin({
  pluginId: 'chat',
  register(reg) {
    reg.registerInit({
      deps: {
        config: coreServices.rootConfig,
        logger: coreServices.logger,
        httpAuth: coreServices.httpAuth,
        httpRouter: coreServices.httpRouter,
        actions: actionsServiceRef,
      },
      async init({ config, logger, httpAuth, httpRouter, actions }) {
        const provider = createProviderFromConfig(config);

        const systemPrompt = config.getOptionalString('chat.systemPrompt');
        const actionMode =
          (config.getOptionalString('chat.actions.mode') as
            | 'auto'
            | 'allowlist') ?? 'auto';
        const allowlist =
          config.getOptionalStringArray('chat.actions.allowlist') ?? [];

        const router = createRouter({
          provider,
          systemPrompt,
          actionMode,
          allowlist,
          listActions: async () => {
            // Use service credentials to list actions — the full catalog
            // of available actions is not user-specific.
            const credentials = await httpAuth.credentials(
              // Dummy request — list() only needs valid credentials
              { headers: {} } as any,
              { allow: ['user', 'service'] },
            );
            const { actions: actionsList } = await actions.list({
              credentials,
            });
            return actionsList;
          },
          invokeAction: async (id, input) => {
            // This will be called from the router's request handler,
            // which will override this with per-request credentials.
            // This default is a fallback that shouldn't be reached.
            throw new Error('invokeAction must be called within a request context');
          },
        });

        // Wrap the router to inject per-request credentials into invokeAction
        const wrappedRouter = createRouter({
          provider,
          systemPrompt,
          actionMode,
          allowlist,
          listActions: async () => {
            const { actions: actionsList } = await actions.list({
              credentials: await httpAuth.credentials(
                { headers: {} } as any,
                { allow: ['user', 'service'] },
              ),
            });
            return actionsList;
          },
          invokeAction: async () => {
            throw new Error('Must use request-scoped invoke');
          },
        });

        // We need a custom middleware that injects per-request credentials
        httpRouter.use(async (req, res, next) => {
          if (req.method === 'POST' && req.path === '/v1/messages') {
            const credentials = await httpAuth.credentials(req);

            // Recreate router options with request-scoped invokeAction
            const requestRouter = createRouter({
              provider,
              systemPrompt,
              actionMode,
              allowlist,
              listActions: async () => {
                const { actions: actionsList } = await actions.list({
                  credentials,
                });
                return actionsList;
              },
              invokeAction: async (id, input) => {
                return actions.invoke({ id, input, credentials });
              },
            });

            requestRouter(req, res, next);
            return;
          }
          next();
        });

        logger.info(
          `Chat plugin initialized with provider: ${config.getString('chat.provider.name')}`,
        );
      },
    });
  },
});
```

- [ ] **Step 6: Write the index.ts**

```typescript
// plugins/chat-backend/src/index.ts
export { chatPlugin as default } from './plugin';
```

- [ ] **Step 7: Run all backend tests**

Run: `cd /Users/nicolaslara/dev/backstage-plugins && yarn backstage-cli repo test --testPathPattern=chat-backend`
Expected: ALL PASS

- [ ] **Step 8: Commit**

```bash
git add plugins/chat-backend/src/router.ts plugins/chat-backend/src/router.test.ts plugins/chat-backend/src/plugin.ts plugins/chat-backend/src/index.ts
git commit -m "feat(chat-backend): add Express router with SSE streaming and plugin registration"
```

---

## Task 8: Frontend — API Client

**Files:**
- Create: `plugins/chat/src/api/ChatApi.ts`
- Create: `plugins/chat/src/api/ChatClient.ts`

- [ ] **Step 1: Write the API ref and interface**

```typescript
// plugins/chat/src/api/ChatApi.ts
import { createApiRef } from '@backstage/frontend-plugin-api';
import { JsonObject, JsonValue } from '@backstage/types';

export type ChatRole = 'user' | 'assistant' | 'tool';

export interface ChatMessageData {
  role: ChatRole;
  content: string;
  toolCalls?: Array<{ id: string; name: string; input: JsonObject }>;
  toolCallId?: string;
}

export type ChatEvent =
  | { type: 'text'; content: string }
  | { type: 'tool_call_start'; name: string; description: string }
  | { type: 'tool_call_result'; name: string; output: JsonValue }
  | {
      type: 'confirmation_required';
      toolCallId: string;
      actionId: string;
      actionTitle: string;
      actionDescription: string;
      input: JsonObject;
    }
  | { type: 'error'; message: string }
  | { type: 'done' };

export interface ConfirmedAction {
  actionId: string;
  toolCallId: string;
  input: JsonObject;
}

export interface ChatApi {
  sendMessage(options: {
    messages: ChatMessageData[];
    confirmedAction?: ConfirmedAction | null;
    onEvent: (event: ChatEvent) => void;
    signal?: AbortSignal;
  }): Promise<void>;
}

export const chatApiRef = createApiRef<ChatApi>({
  id: 'plugin.chat',
});
```

- [ ] **Step 2: Write the SSE client implementation**

```typescript
// plugins/chat/src/api/ChatClient.ts
import { DiscoveryApi, FetchApi } from '@backstage/frontend-plugin-api';
import { ChatApi, ChatEvent, ChatMessageData, ConfirmedAction } from './ChatApi';

export class ChatClient implements ChatApi {
  private readonly discoveryApi: DiscoveryApi;
  private readonly fetchApi: FetchApi;

  constructor(options: { discoveryApi: DiscoveryApi; fetchApi: FetchApi }) {
    this.discoveryApi = options.discoveryApi;
    this.fetchApi = options.fetchApi;
  }

  async sendMessage(options: {
    messages: ChatMessageData[];
    confirmedAction?: ConfirmedAction | null;
    onEvent: (event: ChatEvent) => void;
    signal?: AbortSignal;
  }): Promise<void> {
    const baseUrl = await this.discoveryApi.getBaseUrl('chat');

    const response = await this.fetchApi.fetch(`${baseUrl}/v1/messages`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        messages: options.messages,
        confirmedAction: options.confirmedAction ?? null,
      }),
      signal: options.signal,
    });

    if (!response.ok) {
      throw new Error(`Chat request failed: ${response.statusText}`);
    }

    const reader = response.body?.getReader();
    if (!reader) {
      throw new Error('No response body');
    }

    const decoder = new TextDecoder();
    let buffer = '';

    while (true) {
      const { done, value } = await reader.read();
      if (done) break;

      buffer += decoder.decode(value, { stream: true });
      const lines = buffer.split('\n');
      buffer = lines.pop() ?? '';

      for (const line of lines) {
        if (line.startsWith('data: ')) {
          const data = line.slice(6);
          try {
            const event: ChatEvent = JSON.parse(data);
            options.onEvent(event);
          } catch {
            // Skip malformed events
          }
        }
      }
    }
  }
}
```

- [ ] **Step 3: Commit**

```bash
git add plugins/chat/src/api/ChatApi.ts plugins/chat/src/api/ChatClient.ts
git commit -m "feat(chat): add ChatApi ref and SSE ChatClient"
```

---

## Task 9: Frontend — useChat Hook

**Files:**
- Create: `plugins/chat/src/hooks/useChat.ts`
- Create: `plugins/chat/src/hooks/useChat.test.ts`

- [ ] **Step 1: Write the failing test**

```typescript
// plugins/chat/src/hooks/useChat.test.ts
import { renderHook, act } from '@testing-library/react';
import { useChat, DisplayMessage } from './useChat';
import { ChatApi, ChatEvent } from '../api/ChatApi';

function createMockChatApi(events: ChatEvent[]): ChatApi {
  return {
    sendMessage: async ({ onEvent }) => {
      for (const event of events) {
        onEvent(event);
      }
    },
  };
}

describe('useChat', () => {
  it('starts with empty messages', () => {
    const api = createMockChatApi([]);
    const { result } = renderHook(() => useChat(api));

    expect(result.current.messages).toEqual([]);
    expect(result.current.isLoading).toBe(false);
  });

  it('adds user message and streams assistant response', async () => {
    const api = createMockChatApi([
      { type: 'text', content: 'Hello!' },
      { type: 'done' },
    ]);
    const { result } = renderHook(() => useChat(api));

    await act(async () => {
      await result.current.sendMessage('Hi');
    });

    expect(result.current.messages).toHaveLength(2);
    expect(result.current.messages[0]).toMatchObject({
      role: 'user',
      content: 'Hi',
    });
    expect(result.current.messages[1]).toMatchObject({
      role: 'assistant',
      content: 'Hello!',
    });
  });

  it('tracks pending confirmation for mutating actions', async () => {
    const api = createMockChatApi([
      {
        type: 'confirmation_required',
        toolCallId: 'tc1',
        actionId: 'scaffolder.execute-template',
        actionTitle: 'Execute Template',
        actionDescription: 'Execute a scaffolder template',
        input: { templateRef: 'template:default/node' },
      },
      { type: 'done' },
    ]);
    const { result } = renderHook(() => useChat(api));

    await act(async () => {
      await result.current.sendMessage('Create a node service');
    });

    expect(result.current.pendingConfirmation).toMatchObject({
      actionId: 'scaffolder.execute-template',
      toolCallId: 'tc1',
    });
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd /Users/nicolaslara/dev/backstage-plugins && yarn backstage-cli repo test --testPathPattern=useChat.test`
Expected: FAIL — module `./useChat` not found

- [ ] **Step 3: Write the useChat hook**

```typescript
// plugins/chat/src/hooks/useChat.ts
import { useCallback, useRef, useState } from 'react';
import { ChatApi, ChatEvent, ChatMessageData, ConfirmedAction } from '../api/ChatApi';
import { JsonObject, JsonValue } from '@backstage/types';

export interface ToolCallDisplay {
  name: string;
  description: string;
  output?: JsonValue;
  status: 'running' | 'complete' | 'error';
}

export interface PendingConfirmation {
  toolCallId: string;
  actionId: string;
  actionTitle: string;
  actionDescription: string;
  input: JsonObject;
}

export interface DisplayMessage {
  id: string;
  role: 'user' | 'assistant';
  content: string;
  toolCalls?: ToolCallDisplay[];
  toolCallData?: Array<{ id: string; name: string; input: JsonObject }>;
}

export function useChat(api: ChatApi) {
  const [messages, setMessages] = useState<DisplayMessage[]>([]);
  const [isLoading, setIsLoading] = useState(false);
  const [pendingConfirmation, setPendingConfirmation] =
    useState<PendingConfirmation | null>(null);
  const idCounter = useRef(0);

  const nextId = () => String(++idCounter.current);

  const sendMessage = useCallback(
    async (content: string) => {
      const userMsg: DisplayMessage = {
        id: nextId(),
        role: 'user',
        content,
      };

      const assistantId = nextId();
      const assistantMsg: DisplayMessage = {
        id: assistantId,
        role: 'assistant',
        content: '',
        toolCalls: [],
      };

      setMessages(prev => [...prev, userMsg, assistantMsg]);
      setIsLoading(true);
      setPendingConfirmation(null);

      // Build the message history for the API
      const chatMessages: ChatMessageData[] = [
        ...messages.map(m => ({
          role: m.role as 'user' | 'assistant',
          content: m.content,
          toolCalls: m.toolCallData,
        })),
        { role: 'user' as const, content },
      ];

      await api.sendMessage({
        messages: chatMessages,
        onEvent: (event: ChatEvent) => {
          setMessages(prev =>
            prev.map(m => {
              if (m.id !== assistantId) return m;

              switch (event.type) {
                case 'text':
                  return { ...m, content: m.content + event.content };

                case 'tool_call_start':
                  return {
                    ...m,
                    toolCalls: [
                      ...(m.toolCalls ?? []),
                      {
                        name: event.name,
                        description: event.description,
                        status: 'running' as const,
                      },
                    ],
                  };

                case 'tool_call_result':
                  return {
                    ...m,
                    toolCalls: (m.toolCalls ?? []).map(tc =>
                      tc.name === event.name
                        ? { ...tc, output: event.output, status: 'complete' as const }
                        : tc,
                    ),
                  };

                case 'error':
                  return {
                    ...m,
                    content: m.content || `Error: ${event.message}`,
                  };

                default:
                  return m;
              }
            }),
          );

          if (event.type === 'confirmation_required') {
            setPendingConfirmation({
              toolCallId: event.toolCallId,
              actionId: event.actionId,
              actionTitle: event.actionTitle,
              actionDescription: event.actionDescription,
              input: event.input,
            });
          }
        },
      });

      setIsLoading(false);
    },
    [api, messages],
  );

  const confirmAction = useCallback(async () => {
    if (!pendingConfirmation) return;

    const confirmed: ConfirmedAction = {
      actionId: pendingConfirmation.actionId,
      toolCallId: pendingConfirmation.toolCallId,
      input: pendingConfirmation.input,
    };

    setPendingConfirmation(null);
    setIsLoading(true);

    const assistantId = nextId();
    const assistantMsg: DisplayMessage = {
      id: assistantId,
      role: 'assistant',
      content: '',
      toolCalls: [],
    };

    setMessages(prev => [...prev, assistantMsg]);

    const chatMessages: ChatMessageData[] = messages.map(m => ({
      role: m.role as 'user' | 'assistant',
      content: m.content,
      toolCalls: m.toolCallData,
    }));

    await api.sendMessage({
      messages: chatMessages,
      confirmedAction: confirmed,
      onEvent: (event: ChatEvent) => {
        setMessages(prev =>
          prev.map(m => {
            if (m.id !== assistantId) return m;

            switch (event.type) {
              case 'text':
                return { ...m, content: m.content + event.content };
              case 'tool_call_start':
                return {
                  ...m,
                  toolCalls: [
                    ...(m.toolCalls ?? []),
                    { name: event.name, description: event.description, status: 'running' as const },
                  ],
                };
              case 'tool_call_result':
                return {
                  ...m,
                  toolCalls: (m.toolCalls ?? []).map(tc =>
                    tc.name === event.name
                      ? { ...tc, output: event.output, status: 'complete' as const }
                      : tc,
                  ),
                };
              default:
                return m;
            }
          }),
        );
      },
    });

    setIsLoading(false);
  }, [api, messages, pendingConfirmation]);

  const cancelAction = useCallback(() => {
    setPendingConfirmation(null);
  }, []);

  const clearMessages = useCallback(() => {
    setMessages([]);
    setPendingConfirmation(null);
  }, []);

  return {
    messages,
    isLoading,
    pendingConfirmation,
    sendMessage,
    confirmAction,
    cancelAction,
    clearMessages,
  };
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd /Users/nicolaslara/dev/backstage-plugins && yarn backstage-cli repo test --testPathPattern=useChat.test`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add plugins/chat/src/hooks/useChat.ts plugins/chat/src/hooks/useChat.test.ts
git commit -m "feat(chat): add useChat hook with streaming, confirmation, and state management"
```

---

## Task 10: Frontend — Chat Components

**Files:**
- Create: `plugins/chat/src/components/ChatInput.tsx`
- Create: `plugins/chat/src/components/MessageBubble.tsx`
- Create: `plugins/chat/src/components/ToolCallCard.tsx`
- Create: `plugins/chat/src/components/ConfirmCard.tsx`
- Create: `plugins/chat/src/components/MessageList.tsx`
- Create: `plugins/chat/src/components/ChatPanel.tsx`
- Create: `plugins/chat/src/components/ChatWidget.tsx`

- [ ] **Step 1: Create ChatInput component**

```tsx
// plugins/chat/src/components/ChatInput.tsx
import React, { useState, useCallback } from 'react';
import { makeStyles, createStyles, Theme } from '@material-ui/core/styles';
import IconButton from '@material-ui/core/IconButton';
import InputBase from '@material-ui/core/InputBase';
import SendIcon from '@material-ui/icons/Send';

const useStyles = makeStyles((theme: Theme) =>
  createStyles({
    root: {
      display: 'flex',
      alignItems: 'center',
      padding: theme.spacing(1),
      borderTop: `1px solid ${theme.palette.divider}`,
    },
    input: {
      flex: 1,
      padding: theme.spacing(1),
      fontSize: '0.875rem',
    },
  }),
);

interface ChatInputProps {
  onSend: (message: string) => void;
  disabled?: boolean;
}

export function ChatInput({ onSend, disabled }: ChatInputProps) {
  const classes = useStyles();
  const [value, setValue] = useState('');

  const handleSend = useCallback(() => {
    const trimmed = value.trim();
    if (trimmed) {
      onSend(trimmed);
      setValue('');
    }
  }, [value, onSend]);

  const handleKeyDown = useCallback(
    (e: React.KeyboardEvent) => {
      if (e.key === 'Enter' && !e.shiftKey) {
        e.preventDefault();
        handleSend();
      }
    },
    [handleSend],
  );

  return (
    <div className={classes.root}>
      <InputBase
        className={classes.input}
        placeholder="Ask about your services..."
        value={value}
        onChange={e => setValue(e.target.value)}
        onKeyDown={handleKeyDown}
        disabled={disabled}
        multiline
        maxRows={3}
      />
      <IconButton
        size="small"
        onClick={handleSend}
        disabled={disabled || !value.trim()}
        color="primary"
      >
        <SendIcon fontSize="small" />
      </IconButton>
    </div>
  );
}
```

- [ ] **Step 2: Create MessageBubble component**

```tsx
// plugins/chat/src/components/MessageBubble.tsx
import React from 'react';
import { makeStyles, createStyles, Theme } from '@material-ui/core/styles';
import Typography from '@material-ui/core/Typography';
import ReactMarkdown from 'react-markdown';

const useStyles = makeStyles((theme: Theme) =>
  createStyles({
    root: {
      display: 'flex',
      marginBottom: theme.spacing(1),
    },
    userRoot: {
      justifyContent: 'flex-end',
    },
    assistantRoot: {
      justifyContent: 'flex-start',
    },
    bubble: {
      maxWidth: '85%',
      padding: theme.spacing(1, 1.5),
      borderRadius: 12,
      fontSize: '0.875rem',
      lineHeight: 1.5,
      '& p': { margin: 0 },
      '& p + p': { marginTop: theme.spacing(0.5) },
      '& code': {
        backgroundColor: theme.palette.action.hover,
        padding: '2px 4px',
        borderRadius: 4,
        fontSize: '0.8rem',
      },
      '& pre': {
        backgroundColor: theme.palette.action.hover,
        padding: theme.spacing(1),
        borderRadius: 4,
        overflow: 'auto',
        '& code': {
          backgroundColor: 'transparent',
          padding: 0,
        },
      },
    },
    userBubble: {
      backgroundColor: theme.palette.primary.main,
      color: theme.palette.primary.contrastText,
    },
    assistantBubble: {
      backgroundColor: theme.palette.background.default,
      border: `1px solid ${theme.palette.divider}`,
    },
  }),
);

interface MessageBubbleProps {
  role: 'user' | 'assistant';
  content: string;
}

export function MessageBubble({ role, content }: MessageBubbleProps) {
  const classes = useStyles();

  return (
    <div
      className={`${classes.root} ${
        role === 'user' ? classes.userRoot : classes.assistantRoot
      }`}
    >
      <div
        className={`${classes.bubble} ${
          role === 'user' ? classes.userBubble : classes.assistantBubble
        }`}
      >
        {role === 'assistant' ? (
          <ReactMarkdown>{content}</ReactMarkdown>
        ) : (
          <Typography variant="body2">{content}</Typography>
        )}
      </div>
    </div>
  );
}
```

- [ ] **Step 3: Create ToolCallCard component**

```tsx
// plugins/chat/src/components/ToolCallCard.tsx
import React from 'react';
import { makeStyles, createStyles, Theme } from '@material-ui/core/styles';
import CircularProgress from '@material-ui/core/CircularProgress';
import CheckCircleIcon from '@material-ui/icons/CheckCircle';
import ErrorIcon from '@material-ui/icons/Error';
import Typography from '@material-ui/core/Typography';
import { ToolCallDisplay } from '../hooks/useChat';

const useStyles = makeStyles((theme: Theme) =>
  createStyles({
    root: {
      display: 'flex',
      alignItems: 'center',
      gap: theme.spacing(1),
      padding: theme.spacing(0.5, 1),
      margin: theme.spacing(0.5, 0),
      borderRadius: 8,
      backgroundColor: theme.palette.action.hover,
      fontSize: '0.8rem',
    },
    icon: {
      fontSize: 16,
    },
    successIcon: {
      color: theme.palette.success.main,
      fontSize: 16,
    },
    errorIcon: {
      color: theme.palette.error.main,
      fontSize: 16,
    },
  }),
);

interface ToolCallCardProps {
  toolCall: ToolCallDisplay;
}

export function ToolCallCard({ toolCall }: ToolCallCardProps) {
  const classes = useStyles();

  const label = toolCall.name.replace(/\./g, ' > ');

  return (
    <div className={classes.root}>
      {toolCall.status === 'running' && (
        <CircularProgress size={14} thickness={4} />
      )}
      {toolCall.status === 'complete' && (
        <CheckCircleIcon className={classes.successIcon} />
      )}
      {toolCall.status === 'error' && (
        <ErrorIcon className={classes.errorIcon} />
      )}
      <Typography variant="caption">{label}</Typography>
    </div>
  );
}
```

- [ ] **Step 4: Create ConfirmCard component**

```tsx
// plugins/chat/src/components/ConfirmCard.tsx
import React from 'react';
import { makeStyles, createStyles, Theme } from '@material-ui/core/styles';
import Button from '@material-ui/core/Button';
import Typography from '@material-ui/core/Typography';
import { PendingConfirmation } from '../hooks/useChat';

const useStyles = makeStyles((theme: Theme) =>
  createStyles({
    root: {
      padding: theme.spacing(1.5),
      margin: theme.spacing(1, 0),
      borderRadius: 8,
      border: `1px solid ${theme.palette.warning.main}`,
      backgroundColor: theme.palette.background.paper,
    },
    title: {
      fontWeight: 600,
      marginBottom: theme.spacing(0.5),
    },
    input: {
      fontFamily: 'monospace',
      fontSize: '0.75rem',
      backgroundColor: theme.palette.action.hover,
      padding: theme.spacing(0.5, 1),
      borderRadius: 4,
      marginBottom: theme.spacing(1),
      overflow: 'auto',
      maxHeight: 80,
    },
    actions: {
      display: 'flex',
      gap: theme.spacing(1),
    },
  }),
);

interface ConfirmCardProps {
  confirmation: PendingConfirmation;
  onConfirm: () => void;
  onCancel: () => void;
}

export function ConfirmCard({
  confirmation,
  onConfirm,
  onCancel,
}: ConfirmCardProps) {
  const classes = useStyles();

  return (
    <div className={classes.root}>
      <Typography variant="body2" className={classes.title}>
        {confirmation.actionTitle}
      </Typography>
      <Typography variant="caption" color="textSecondary">
        {confirmation.actionDescription}
      </Typography>
      <pre className={classes.input}>
        {JSON.stringify(confirmation.input, null, 2)}
      </pre>
      <div className={classes.actions}>
        <Button
          size="small"
          variant="contained"
          color="primary"
          onClick={onConfirm}
        >
          Confirm
        </Button>
        <Button size="small" variant="outlined" onClick={onCancel}>
          Cancel
        </Button>
      </div>
    </div>
  );
}
```

- [ ] **Step 5: Create MessageList component**

```tsx
// plugins/chat/src/components/MessageList.tsx
import React, { useEffect, useRef } from 'react';
import { makeStyles, createStyles, Theme } from '@material-ui/core/styles';
import { DisplayMessage, PendingConfirmation } from '../hooks/useChat';
import { MessageBubble } from './MessageBubble';
import { ToolCallCard } from './ToolCallCard';
import { ConfirmCard } from './ConfirmCard';

const useStyles = makeStyles((theme: Theme) =>
  createStyles({
    root: {
      flex: 1,
      overflow: 'auto',
      padding: theme.spacing(1.5),
    },
    empty: {
      display: 'flex',
      alignItems: 'center',
      justifyContent: 'center',
      height: '100%',
      color: theme.palette.text.secondary,
      fontSize: '0.875rem',
    },
  }),
);

interface MessageListProps {
  messages: DisplayMessage[];
  pendingConfirmation: PendingConfirmation | null;
  onConfirm: () => void;
  onCancel: () => void;
}

export function MessageList({
  messages,
  pendingConfirmation,
  onConfirm,
  onCancel,
}: MessageListProps) {
  const classes = useStyles();
  const bottomRef = useRef<HTMLDivElement>(null);

  useEffect(() => {
    bottomRef.current?.scrollIntoView({ behavior: 'smooth' });
  }, [messages, pendingConfirmation]);

  if (messages.length === 0) {
    return (
      <div className={`${classes.root} ${classes.empty}`}>
        Ask me about your services, APIs, and teams.
      </div>
    );
  }

  return (
    <div className={classes.root}>
      {messages.map(msg => (
        <div key={msg.id}>
          <MessageBubble role={msg.role} content={msg.content} />
          {msg.toolCalls?.map((tc, i) => (
            <ToolCallCard key={`${msg.id}-tc-${i}`} toolCall={tc} />
          ))}
        </div>
      ))}
      {pendingConfirmation && (
        <ConfirmCard
          confirmation={pendingConfirmation}
          onConfirm={onConfirm}
          onCancel={onCancel}
        />
      )}
      <div ref={bottomRef} />
    </div>
  );
}
```

- [ ] **Step 6: Create ChatPanel component**

```tsx
// plugins/chat/src/components/ChatPanel.tsx
import React from 'react';
import { makeStyles, createStyles, Theme } from '@material-ui/core/styles';
import Typography from '@material-ui/core/Typography';
import IconButton from '@material-ui/core/IconButton';
import CloseIcon from '@material-ui/icons/Close';
import DeleteIcon from '@material-ui/icons/DeleteOutline';
import { useApi } from '@backstage/frontend-plugin-api';
import { chatApiRef } from '../api/ChatApi';
import { useChat } from '../hooks/useChat';
import { MessageList } from './MessageList';
import { ChatInput } from './ChatInput';

const useStyles = makeStyles((theme: Theme) =>
  createStyles({
    root: {
      display: 'flex',
      flexDirection: 'column',
      width: 400,
      height: 500,
      borderRadius: 12,
      boxShadow: theme.shadows[8],
      backgroundColor: theme.palette.background.paper,
      overflow: 'hidden',
    },
    header: {
      display: 'flex',
      alignItems: 'center',
      justifyContent: 'space-between',
      padding: theme.spacing(1, 1.5),
      borderBottom: `1px solid ${theme.palette.divider}`,
    },
    headerActions: {
      display: 'flex',
      gap: theme.spacing(0.5),
    },
  }),
);

interface ChatPanelProps {
  onClose: () => void;
}

export function ChatPanel({ onClose }: ChatPanelProps) {
  const classes = useStyles();
  const chatApi = useApi(chatApiRef);
  const {
    messages,
    isLoading,
    pendingConfirmation,
    sendMessage,
    confirmAction,
    cancelAction,
    clearMessages,
  } = useChat(chatApi);

  return (
    <div className={classes.root}>
      <div className={classes.header}>
        <Typography variant="subtitle2">Chat</Typography>
        <div className={classes.headerActions}>
          <IconButton size="small" onClick={clearMessages} title="Clear chat">
            <DeleteIcon fontSize="small" />
          </IconButton>
          <IconButton size="small" onClick={onClose} title="Close">
            <CloseIcon fontSize="small" />
          </IconButton>
        </div>
      </div>
      <MessageList
        messages={messages}
        pendingConfirmation={pendingConfirmation}
        onConfirm={confirmAction}
        onCancel={cancelAction}
      />
      <ChatInput onSend={sendMessage} disabled={isLoading} />
    </div>
  );
}
```

- [ ] **Step 7: Create ChatWidget component**

```tsx
// plugins/chat/src/components/ChatWidget.tsx
import React, { useState, useCallback } from 'react';
import { makeStyles, createStyles, Theme } from '@material-ui/core/styles';
import Fab from '@material-ui/core/Fab';
import ChatIcon from '@material-ui/icons/Chat';
import Slide from '@material-ui/core/Slide';
import { ChatPanel } from './ChatPanel';

const useStyles = makeStyles((theme: Theme) =>
  createStyles({
    root: {
      position: 'fixed',
      bottom: theme.spacing(3),
      right: theme.spacing(3),
      zIndex: theme.zIndex.modal,
    },
    panel: {
      position: 'absolute',
      bottom: 64,
      right: 0,
    },
  }),
);

export function ChatWidget() {
  const classes = useStyles();
  const [open, setOpen] = useState(false);

  const handleToggle = useCallback(() => setOpen(prev => !prev), []);
  const handleClose = useCallback(() => setOpen(false), []);

  return (
    <div className={classes.root}>
      <Slide direction="up" in={open} mountOnEnter unmountOnExit>
        <div className={classes.panel}>
          <ChatPanel onClose={handleClose} />
        </div>
      </Slide>
      <Fab color="primary" size="medium" onClick={handleToggle}>
        <ChatIcon />
      </Fab>
    </div>
  );
}
```

- [ ] **Step 8: Commit**

```bash
git add plugins/chat/src/components/
git commit -m "feat(chat): add ChatWidget, ChatPanel, MessageList, and supporting components"
```

---

## Task 11: Frontend — Plugin Registration

**Files:**
- Create: `plugins/chat/src/plugin.ts`
- Create: `plugins/chat/src/index.ts`

- [ ] **Step 1: Write the plugin definition**

The plugin uses `createFrontendPlugin` to register itself and provide the API factory + a component extension that renders the `ChatWidget` globally.

```typescript
// plugins/chat/src/plugin.ts
import {
  createFrontendPlugin,
  createApiFactory,
  discoveryApiRef,
  fetchApiRef,
} from '@backstage/frontend-plugin-api';
import { ExtensionBlueprint } from '@backstage/frontend-plugin-api';
import { chatApiRef } from './api/ChatApi';
import { ChatClient } from './api/ChatClient';

// Create a wrapper extension that renders the ChatWidget on every page
// by attaching to the app root.
const chatWidgetExtension = ExtensionBlueprint.make({
  name: 'chat-widget',
  attachTo: { id: 'app/layout', input: 'content' },
  output: {
    element: ExtensionBlueprint.dataRefs.reactElement,
  },
  *factory() {
    const { ChatWidget } = await import('./components/ChatWidget');
    yield { element: <ChatWidget /> };
  },
});

export const chatPlugin = createFrontendPlugin({
  pluginId: 'chat',
  extensions: [chatWidgetExtension],
  apis: [
    createApiFactory({
      api: chatApiRef,
      deps: { discoveryApi: discoveryApiRef, fetchApi: fetchApiRef },
      factory: ({ discoveryApi, fetchApi }) =>
        new ChatClient({ discoveryApi, fetchApi }),
    }),
  ],
});
```

> **Note:** The exact extension attachment point (`app/layout` + `content` input) may need adjustment based on the Backstage frontend system version. If `app/layout` doesn't support a `content` input, use `PluginWrapperBlueprint` instead — this is a known pattern for global components. The implementer should verify the available extension points in their Backstage version and adjust accordingly. An alternative approach is to export `ChatWidget` directly and have consumers add it manually to their `App.tsx`.

- [ ] **Step 2: Write the index.ts**

```typescript
// plugins/chat/src/index.ts
export { chatPlugin } from './plugin';
export { ChatWidget } from './components/ChatWidget';
export { chatApiRef } from './api/ChatApi';
export type { ChatApi, ChatEvent, ChatMessageData } from './api/ChatApi';
```

- [ ] **Step 3: Commit**

```bash
git add plugins/chat/src/plugin.ts plugins/chat/src/index.ts
git commit -m "feat(chat): add plugin registration with API factory and ChatWidget export"
```

---

## Task 12: Build Verification + All Tests

**Files:** None (verification only)

- [ ] **Step 1: Run all backend tests**

Run: `cd /Users/nicolaslara/dev/backstage-plugins && yarn backstage-cli repo test --testPathPattern=chat-backend`
Expected: ALL PASS

- [ ] **Step 2: Run all frontend tests**

Run: `cd /Users/nicolaslara/dev/backstage-plugins && yarn backstage-cli repo test --testPathPattern=chat`
Expected: ALL PASS

- [ ] **Step 3: Run TypeScript compilation**

Run: `cd /Users/nicolaslara/dev/backstage-plugins && yarn tsc`
Expected: No errors

- [ ] **Step 4: Run lint**

Run: `cd /Users/nicolaslara/dev/backstage-plugins && yarn lint`
Expected: No errors (or only warnings)

- [ ] **Step 5: Build both packages**

Run: `cd /Users/nicolaslara/dev/backstage-plugins && yarn build`
Expected: Both plugins build successfully

---

## Task 13: Integration with archx-forge

**Files:**
- Modify: `/Users/nicolaslara/dev/archxsuite/archx-forge/packages/backend/package.json` — Add `@larafonse/plugin-chat-backend` dependency
- Modify: `/Users/nicolaslara/dev/archxsuite/archx-forge/packages/backend/src/index.ts` — Register chat backend plugin
- Modify: `/Users/nicolaslara/dev/archxsuite/archx-forge/packages/app/package.json` — Add `@larafonse/plugin-chat` dependency
- Modify: `/Users/nicolaslara/dev/archxsuite/archx-forge/packages/app/src/App.tsx` — Add chat plugin to features
- Modify: `/Users/nicolaslara/dev/archxsuite/archx-forge/app-config.yaml` — Add chat config section

- [ ] **Step 1: Link the local plugins for development**

During development, use a local file reference instead of a published npm package.

Add to archx-forge's `packages/backend/package.json` dependencies:
```json
"@larafonse/plugin-chat-backend": "file:../../../backstage-plugins/plugins/chat-backend"
```

Add to archx-forge's `packages/app/package.json` dependencies:
```json
"@larafonse/plugin-chat": "file:../../../backstage-plugins/plugins/chat"
```

Run: `cd /Users/nicolaslara/dev/archxsuite/archx-forge && yarn install`

- [ ] **Step 2: Register the backend plugin**

Add to `/Users/nicolaslara/dev/archxsuite/archx-forge/packages/backend/src/index.ts`, after the MCP actions plugin line:

```typescript
// chat plugin
backend.add(import('@larafonse/plugin-chat-backend'));
```

- [ ] **Step 3: Add chat plugin to frontend features**

Update `/Users/nicolaslara/dev/archxsuite/archx-forge/packages/app/src/App.tsx`:

```typescript
import { createApp } from '@backstage/frontend-defaults';
import catalogPlugin from '@backstage/plugin-catalog/alpha';
import homePlugin from '@backstage/plugin-home/alpha';
import authPlugin from '@backstage/plugin-auth';
import { chatPlugin } from '@larafonse/plugin-chat';
import { navModule } from './modules/nav';
import { themeModule } from './modules/theme';
import { homeModule } from './modules/home';

export default createApp({
  features: [
    catalogPlugin,
    homePlugin,
    authPlugin,
    chatPlugin,
    navModule,
    themeModule,
    homeModule,
  ],
});
```

- [ ] **Step 4: Add chat configuration to app-config.yaml**

Add to the end of `/Users/nicolaslara/dev/archxsuite/archx-forge/app-config.yaml`:

```yaml
chat:
  provider:
    name: anthropic
    config:
      apiKey: ${ANTHROPIC_API_KEY}
      model: claude-sonnet-4-20250514
  systemPrompt: >
    You are a helpful developer portal assistant for Archx Forge.
    Help users find information about their services, APIs, and teams,
    and assist with creating new services from templates.
  actions:
    mode: auto
```

- [ ] **Step 5: Start the application and verify**

Run: `cd /Users/nicolaslara/dev/archxsuite/archx-forge && ANTHROPIC_API_KEY=your-key yarn start`

Verify:
1. Backstage loads without errors
2. A chat FAB button appears in the bottom-right corner
3. Clicking it opens the chat panel
4. Typing a message sends it to the backend and streams a response
5. Tool calls show progress indicators
6. Mutating actions show confirmation cards

- [ ] **Step 6: Commit the archx-forge integration**

```bash
cd /Users/nicolaslara/dev/archxsuite/archx-forge
git add packages/backend/package.json packages/backend/src/index.ts packages/app/package.json packages/app/src/App.tsx app-config.yaml
git commit -m "feat: integrate @larafonse/plugin-chat for AI-powered chat widget"
```
