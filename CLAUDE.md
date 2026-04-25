# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm run dev          # Start dev server (Next.js + Turbopack)
npm run dev:daemon   # Start dev server in background, logs to logs.txt
npm run build        # Production build
npm run lint         # ESLint
npm run test         # Run all Vitest tests
npm run setup        # Install deps + Prisma setup + DB migrations
npm run db:reset     # Reset SQLite database

# Run a single test file
npx vitest src/lib/__tests__/file-system.test.ts
# Run tests matching a name pattern
npx vitest --grep "renders chat interface"
```

Requires `ANTHROPIC_API_KEY` in `.env`; falls back to a mock model if absent.

## Architecture

UIGen is an AI-powered React component generator with live preview. The user describes components in chat; Claude generates code via tool calls; the result renders in an iframe.

### Core Data Flow

1. **Chat input** → `POST /api/chat` (`src/app/api/chat/route.ts`)
2. **AI generation** → Vercel AI SDK `streamText()` with Anthropic Claude; Claude calls two tools to modify files
3. **Virtual file system** updated by tool results; state lives in `FileSystemProvider`
4. **Preview** → Babel transforms JSX in-browser, renders in an isolated iframe (`PreviewFrame`)
5. **Persistence** → Logged-in users save messages + file system JSON to Prisma/SQLite; anonymous users use localStorage

### Key Modules

| Path | Role |
|------|------|
| `src/app/api/chat/route.ts` | AI streaming endpoint; wires tools to file system context |
| `src/lib/file-system.ts` | `VirtualFileSystem` class — in-memory `Map<path, content>`, no disk writes |
| `src/lib/contexts/chat-context.tsx` | `ChatProvider` — message history, streaming state |
| `src/lib/contexts/file-system-context.tsx` | `FileSystemProvider` — file CRUD, tool execution callbacks |
| `src/lib/tools/` | `str_replace_editor` and `file_manager` — tools Claude uses to create/edit files |
| `src/lib/transform/jsx-transformer.ts` | Babel Standalone transforms JSX → JS for iframe execution |
| `src/lib/prompts/generation.tsx` | System prompt for code generation |
| `src/lib/provider.ts` | `getLanguageModel()` — returns Anthropic model or `MockLanguageModel` |
| `src/lib/auth.ts` | JWT session management (jose, 7-day expiry) |
| `src/actions/` | Server actions for auth (sign-in/up/out) and project persistence |
| `src/components/preview/PreviewFrame.tsx` | Iframe renderer — receives transformed JS, runs it sandboxed |
| `src/components/editor/` | Monaco-based code editor + file tree |

### AI Tool Use

Claude has exactly two tools:
- **`str_replace_editor`** — view/create/edit files with surgical string replacement
- **`file_manager`** — rename, delete, list files

Tool calls from the streaming response are intercepted in the API route and applied to the server-side file system snapshot, then synced to `FileSystemProvider` on the client.

### Auth & Persistence

- JWT stored in an HTTP-only cookie (`session`)
- Prisma schema: `User` (email + bcrypt password) → `Project` (messages JSON, file system JSON)
- Anonymous sessions store project state in `localStorage` under a generated project ID
- `src/middleware.ts` redirects unauthenticated users away from protected routes

### Database

SQLite (`prisma/dev.db`). Schema at `prisma/schema.prisma`. Run `npx prisma studio` to inspect.

### Testing

Tests live in `src/**/__tests__/` as `.test.ts` (logic) and `.test.tsx` (components). Vitest with jsdom; contexts and heavy dependencies are mocked with `vi.mock()`.
