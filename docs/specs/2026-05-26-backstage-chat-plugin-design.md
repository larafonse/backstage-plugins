# Backstage Chat Plugin — Design Spec

**Date:** 2026-05-26
**Status:** Draft
**Repo:** github.com/larafonse/backstage-plugins

## Overview

An open-source Backstage plugin that adds an AI-powered chat widget to any Backstage instance. Users interact with a floating chat UI to ask questions about their software catalog, trigger scaffolder templates, and execute any actions exposed via MCP Actions — all through natural language.

## Goals

- Generic, open-source plugin usable by any Backstage instance
- LLM-provider agnostic (ships with Anthropic and OpenAI providers)
- Leverages Backstage's existing MCP Actions infrastructure — any action exposed via MCP is automatically available to the chatbot
- Confirmation step before any mutating action
- User's own Backstage permissions apply to all actions

## Non-Goals (v1)

- Conversation persistence across browser sessions
- Multi-user threads or shared conversations
- File uploads
- Custom slash commands
- Slack integration (future)

## Architecture

Two npm packages, developed in a standalone monorepo:

- **`@larafonse/plugin-chat`** — Frontend React plugin. Floating chat widget available on every page.
- **`@larafonse/plugin-chat-backend`** — Backend plugin. Orchestrates LLM calls and MCP action execution.

### Data Flow

```
User types message
  → Frontend POST /api/chat/v1/messages (with conversation history)
  → Backend loads available actions via actionsServiceRef.list()
  → Backend calls LLM with messages + tool definitions
  → LLM responds (possibly with tool calls)
  → If read-only action: Backend executes via actionsServiceRef.invoke()
  → If mutating action: Backend returns confirmation_required event to frontend
  → User confirms → Backend executes action
  → Backend streams final response to frontend
  → Frontend renders response + action results
```

The frontend never interacts with MCP actions or LLM providers directly. The backend is the sole orchestration layer.

## Frontend Plugin (`@backstage/plugin-chat`)

### Widget

- Floating action button in the bottom-right corner with a chat icon
- Clicking opens a slide-up chat panel (~400px wide, ~500px tall)
- Available on every page (rendered as a framework-level component, not page-routed)

### Chat UI (v1)

- Message bubbles for user and assistant
- Markdown rendering in assistant responses
- Streaming text display (tokens appear as they arrive)
- Tool-call indicators: "Looking up services...", "Creating service..." status messages while actions execute
- Confirmation cards for mutating actions: shows action details with [Confirm] and [Cancel] buttons
- Conversation persists in browser memory for the session (lost on refresh)

### Integration

Added to the app via a wrapper component:

```tsx
// packages/app/src/App.tsx
import { ChatWidget } from '@larafonse/plugin-chat';

// Rendered inside the app layout
<ChatWidget />
```

## Backend Plugin (`@backstage/plugin-chat-backend`)

### API

**`POST /api/chat/v1/messages`**

Request body:
```json
{
  "messages": [
    { "role": "user", "content": "What services does team-platform own?" }
  ],
  "conversationId": "optional-session-id",
  "confirmedAction": null
}
```

Response: Server-Sent Events stream with event types:
- `text` — Streamed text chunk from the LLM
- `tool_call_start` — Action is being executed (name, description)
- `tool_call_result` — Action result data
- `confirmation_required` — Mutating action needs user approval (action ID, input, description)
- `error` — Error details
- `done` — Stream complete

### Tool Orchestration Loop

1. On each request, call `actionsServiceRef.list()` to get all available MCP actions
2. Filter actions based on configured mode (`auto` or `allowlist`)
3. Convert action JSON schemas to LLM tool definitions
4. Send conversation messages + tool definitions to configured LLM provider
5. If LLM returns tool calls:
   - **Read-only actions** (`readOnly: true`): Execute immediately via `actionsServiceRef.invoke()`
   - **Mutating actions** (`readOnly: false`): Return `confirmation_required` event to frontend
6. If user confirms a mutating action: execute via `actionsServiceRef.invoke()`
7. Feed tool results back to LLM for natural language summary
8. Stream all events to frontend

### Authentication

- Uses Backstage's built-in `httpAuth` service — no custom auth
- Actions execute with the logged-in user's `BackstageCredentials` from `httpAuth.credentials(req)`
- Backstage's permission framework applies — if a user can't do something in Backstage, they can't do it via chat

### Credential Injection

```typescript
import { actionsServiceRef } from '@backstage/backend-plugin-api/alpha';

// In plugin init:
deps: {
  actions: actionsServiceRef,
  httpAuth: coreServices.httpAuth,
}

// In request handler:
const credentials = await httpAuth.credentials(req);
const result = await actions.invoke({
  id: 'catalog.query-catalog-entities',
  input: { filter: { kind: 'Component' } },
  credentials,
});
```

## LLM Provider Abstraction

### Interface

```typescript
interface ChatProvider {
  chat(options: {
    messages: ChatMessage[];
    tools: ToolDefinition[];
    systemPrompt?: string;
  }): AsyncIterable<ChatEvent>;
}

type ChatEvent =
  | { type: 'text'; content: string }
  | { type: 'tool_call'; id: string; name: string; input: JsonObject }
  | { type: 'done' };

type ChatMessage =
  | { role: 'user'; content: string }
  | { role: 'assistant'; content: string; toolCalls?: ToolCall[] }
  | { role: 'tool'; toolCallId: string; content: string };
```

### Built-in Providers (v1)

- **Anthropic** — Uses `@anthropic-ai/sdk`, Claude models with native tool-use
- **OpenAI** — Uses `openai` SDK, GPT models with function calling

### Provider Registration

Providers are registered by name. The active provider is selected via config:

```typescript
// Built-in
registerProvider('anthropic', AnthropicProvider);
registerProvider('openai', OpenAIProvider);

// Community providers can register additional ones
```

## Configuration

```yaml
chat:
  provider:
    name: anthropic
    config:
      apiKey: ${ANTHROPIC_API_KEY}
      model: claude-sonnet-4-20250514
  systemPrompt: >
    You are a helpful developer portal assistant.
    Help users find information about their services,
    APIs, and teams, and assist with creating new services.
  actions:
    mode: auto          # 'auto' = all MCP actions available
    # mode: allowlist   # restrict to specific actions
    # allowlist:
    #   - catalog.query-catalog-entities
    #   - catalog.get-catalog-entity
    #   - scaffolder.execute-template
```

## Repo Structure

```
backstage-plugins/
├── plugins/
│   ├── chat/                          # Frontend plugin
│   │   ├── src/
│   │   │   ├── index.ts
│   │   │   ├── plugin.ts             # Plugin definition
│   │   │   ├── components/
│   │   │   │   ├── ChatWidget.tsx     # Floating button + panel
│   │   │   │   ├── ChatPanel.tsx      # Chat panel container
│   │   │   │   ├── MessageList.tsx    # Message rendering
│   │   │   │   ├── MessageBubble.tsx  # Individual message
│   │   │   │   ├── ToolCallCard.tsx   # Action execution indicator
│   │   │   │   ├── ConfirmCard.tsx    # Mutation confirmation UI
│   │   │   │   └── ChatInput.tsx      # Message input
│   │   │   ├── api/
│   │   │   │   ├── ChatApi.ts         # API client interface
│   │   │   │   └── ChatClient.ts      # SSE client implementation
│   │   │   └── hooks/
│   │   │       └── useChat.ts         # Chat state management
│   │   └── package.json
│   │
│   └── chat-backend/                  # Backend plugin
│       ├── src/
│       │   ├── index.ts
│       │   ├── plugin.ts             # Plugin definition + init
│       │   ├── router.ts             # Express router
│       │   ├── providers/
│       │   │   ├── types.ts           # ChatProvider interface
│       │   │   ├── anthropic.ts       # Anthropic provider
│       │   │   ├── openai.ts          # OpenAI provider
│       │   │   └── registry.ts        # Provider registry
│       │   └── orchestrator.ts        # Tool call loop + confirmation logic
│       └── package.json
│
├── package.json                       # Monorepo root
├── tsconfig.json
└── README.md
```

## Security Considerations

- **API keys** are server-side only, configured via environment variables in `app-config.yaml`
- **User permissions** enforced by Backstage's permission framework — the chat cannot bypass existing access controls
- **Mutation confirmation** prevents accidental destructive actions
- **No data persistence** in v1 — conversations are ephemeral, no chat logs stored
- **Input sanitization** — user messages are passed to the LLM as-is but tool call inputs are validated against action schemas by `actionsServiceRef`
