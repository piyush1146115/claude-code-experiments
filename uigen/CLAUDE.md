# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This App Does

**UIGen** is an AI-powered React component generator. Users describe components in a chat interface; Claude generates code using tool calls that write to an in-memory virtual file system. The result renders live in an iframe preview. Projects are persisted to SQLite for authenticated users.

## Commands

```bash
npm run setup        # First-time: install deps + prisma generate + prisma migrate dev
npm run dev          # Dev server with Turbopack at localhost:3000
npm run build        # Production build
npm run test         # Run all tests (vitest)
npm run test -- src/lib/__tests__/file-system.test.ts  # Run a single test file
npx prisma studio    # Browse database
npx prisma migrate dev --name <name>  # Create a new migration
```

`ANTHROPIC_API_KEY` in `.env` is optional — without it, a mock provider returns static example components.

## Architecture

### Request Flow

```
User message → POST /api/chat
  → streamText() with Claude + tools
  → AI calls str_replace_editor / file_manager tool calls
  → FileSystemContext.handleToolCall() updates VirtualFileSystem
  → PreviewFrame detects refreshTrigger change
  → Babel transforms JSX in-browser, builds import map, renders in iframe
  → onFinish: saves messages + serialized FS to Prisma (if authenticated)
```

### Key Abstractions

**`VirtualFileSystem`** (`src/lib/file-system.ts`) — All file state lives in a `Map<string, FileNode>` in memory. Never writes to disk. Can serialize/deserialize to plain JSON for Prisma storage. The AI tools (`str_replace_editor`, `file_manager`) operate on this object.

**`FileSystemContext`** (`src/lib/contexts/file-system-context.tsx`) — Wraps `VirtualFileSystem` in React state. Exposes `handleToolCall()` which routes incoming AI tool calls to the right FS methods and increments `refreshTrigger` to signal `PreviewFrame`.

**`ChatContext`** (`src/lib/contexts/chat-context.tsx`) — Wraps `@ai-sdk/react`'s `useChat`. Manages `input` state manually (the v3 SDK removed `input`/`handleInputChange`/`handleSubmit` from the hook return — these are now local state + `sendMessage({ text })`). Passes `body: { files, projectId }` to every request.

**`PreviewFrame`** (`src/components/preview/PreviewFrame.tsx`) — Subscribes to `refreshTrigger`. On change: reads all virtual files, runs Babel standalone on JSX, creates blob URLs, builds an ES module import map (maps bare specifiers to `esm.sh`), injects into `iframe.srcdoc`.

### Auth

JWT sessions via `jose` stored in an `httpOnly` cookie. `src/lib/auth.ts` handles create/verify/delete. Server actions in `src/actions/` use `getSession()`. The middleware (`src/middleware.ts`) guards `/api/projects` and `/api/filesystem`.

### AI Tools

Two tools are registered in `/api/chat/route.ts`:
- `str_replace_editor` — create, view, or str-replace content in virtual files
- `file_manager` — rename or delete virtual files

Tool definitions live in `src/lib/tools/`. The system prompt is in `src/lib/prompts/generation.tsx`.

### Data Model

The database schema is defined in `prisma/schema.prisma` — reference it whenever you need to understand data structure.

`Project.messages` and `Project.data` are JSON strings — chat history and the serialized virtual FS respectively. The Prisma client is generated into `src/generated/prisma/` (not `node_modules`).

## Code Style

Only comment complex or non-obvious code paths. Self-explanatory code should not have comments.

## Important Patterns

- **`@ai-sdk/react` v3 API**: `useChat` returns `{ messages, sendMessage, status, ... }` — no `input`, `handleInputChange`, or `handleSubmit`. Input is managed as local state; submit calls `sendMessage({ text: input })`.
- **`UIMessage` type**: `ai` v6 replaced `Message` with `UIMessage`. Messages have a `parts` array (no `content` string). Tool parts use `type: 'tool-${toolName}'` pattern; use `isToolUIPart()` and `getToolName()` helpers from `ai`.
- **Mock provider**: When `ANTHROPIC_API_KEY` is absent, `src/lib/provider.ts` returns a `MockLanguageModel` that simulates tool calls without hitting the API.
