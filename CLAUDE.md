# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**UIGen** — an AI-powered React component generator with live preview. Users describe components in a chat interface; Claude generates React + Tailwind code that renders instantly in a preview pane. All file operations are virtual (in-memory); nothing is written to disk.

The main application lives in the `uigen/` subdirectory.

## Commands

All commands run from `uigen/`:

```bash
npm run setup        # First-time setup: install deps, generate Prisma client, run migrations
npm run dev          # Dev server with Turbopack at http://localhost:3000
npm run build        # Production build
npm run lint         # ESLint
npm run test         # Vitest (unit + integration)
npm run db:reset     # Reset SQLite database
```

To run a single test file:
```bash
npx vitest run src/lib/__tests__/file-system.test.ts
```

## Architecture

### Data Flow
1. User sends a message → `POST /api/chat` (`src/app/api/chat/route.ts`)
2. Route streams a response from Claude (Anthropic SDK via `src/lib/provider.ts`) with two tools: `str_replace_editor` and `file_manager`
3. Tool calls are intercepted by `FileSystemContext` (`src/lib/contexts/file-system-context.tsx`), which mutates the in-memory `VirtualFileSystem`
4. `PreviewFrame` (`src/components/preview/PreviewFrame.tsx`) re-renders the iframe whenever the virtual FS changes — JSX is compiled client-side via Babel standalone in `src/lib/transform/jsx-transformer.ts`

### Key Abstractions

- **VirtualFileSystem** (`src/lib/file-system.ts`): In-memory tree of files. Serialized as JSON and stored in the `Project.data` DB column. No disk I/O.
- **Provider** (`src/lib/provider.ts`): Returns either the real Anthropic Claude model or a `MockLanguageModel` (used when `ANTHROPIC_API_KEY` is absent).
- **System prompt** (`src/lib/prompts/generation.tsx`): Instructs Claude to build React components with Tailwind CSS and write them to the virtual FS using the provided tools.
- **ChatContext** (`src/lib/contexts/chat-context.tsx`): Wraps Vercel AI SDK's `useChat`; owns message history and streaming state.
- **FileSystemContext** (`src/lib/contexts/file-system-context.tsx`): Owns VirtualFileSystem state and handles tool-call results from the AI stream.

### Auth & Persistence
- JWT sessions stored in httpOnly cookies (`src/lib/auth.ts` using `jose`)
- Passwords hashed with `bcrypt`
- Database: SQLite via Prisma (`prisma/schema.prisma`); client generated to `src/generated/prisma/`
- Anonymous users' project associations are tracked in `localStorage` (`src/lib/anon-work-tracker.ts`)
- Server actions in `src/actions/` handle CRUD for projects and auth

### Routing
- `/` — home page; redirects authenticated users to their latest project or creates one
- `/[projectId]` — loads a project and renders the full editor UI (`src/app/main-content.tsx`)
- `/api/chat` — streaming chat endpoint (protected by `src/middleware.ts`)

### UI Layout
`main-content.tsx` uses `resizable` panels (shadcn/ui) arranged as: **Chat | Preview / Code editor + File tree**. UI primitives are shadcn/ui components in `src/components/ui/`.

### Node Compatibility
`node-compat.cjs` (loaded via `next.config.ts`) removes global `localStorage`/`sessionStorage` on the server to prevent SSR crashes on Node.js 25+.
