# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm run setup          # Install deps, generate Prisma client, run migrations
npm run dev            # Start dev server with Turbopack (http://localhost:3000)
npm run dev:daemon     # Dev server in background, logs to logs.txt
npm run build          # Production build
npm run lint           # ESLint
npm test               # Run all Vitest tests
npx vitest run src/components/chat/__tests__/ChatInterface.test.tsx  # Run a single test file
npm run db:reset       # Reset and re-run all migrations (destroys data)
```

The dev server requires `NODE_OPTIONS='--require ./node-compat.cjs'` — this is already baked into the npm scripts, so always use them rather than running `next dev` directly.

## Architecture

UIGen is a Next.js 15 App Router app where users describe React components in a chat, and Claude generates them with a live preview.

### Request flow

1. User sends a chat message → `POST /api/chat` (`src/app/api/chat/route.ts`)
2. The route reconstructs a `VirtualFileSystem` from the serialized file state sent in the request body, then calls `streamText` with two tools: `str_replace_editor` and `file_manager`
3. The AI uses those tools to create/edit files inside the virtual FS
4. On the client, `onToolCall` in `ChatContext` (`src/lib/contexts/chat-context.tsx`) intercepts each tool call and mutates the client-side `FileSystemContext`
5. `PreviewFrame` (`src/components/preview/PreviewFrame.tsx`) watches `refreshTrigger` and re-renders the preview by: transpiling all VFS files with Babel standalone → building a browser import map with blob URLs → injecting an `<importmap>` script into a sandboxed `<iframe>`

### Virtual File System

`VirtualFileSystem` (`src/lib/file-system.ts`) is an in-memory tree of `FileNode`s. It lives in two places simultaneously:
- **Client**: held in `FileSystemContext` state; mutations trigger `refreshTrigger` increments
- **Server**: reconstructed from the serialized `Record<string, FileNode>` sent with each chat request; the server-side instance is mutated by AI tool calls and then serialized back to the DB on finish

The AI always operates on a fresh server-side copy. The client copy is the authoritative UI state; it is updated immediately via `handleToolCall` callbacks, not by waiting for the server response.

### AI provider

`src/lib/provider.ts` exports `getLanguageModel()`. If `ANTHROPIC_API_KEY` is missing, it returns a `MockLanguageModel` that produces static Counter/Card/ContactForm demos. With a real key, it uses `claude-haiku-4-5`.

### Auth

JWT-based, cookie-stored (`auth-token`). Implemented with `jose` in `src/lib/auth.ts`. Sessions expire in 7 days. Anonymous users can generate components — their work is stored in `sessionStorage` via `src/lib/anon-work-tracker.ts` and offered for save after sign-up.

### Persistence

Prisma/SQLite (`prisma/dev.db`). Two models:
- `User` — email + bcrypt password
- `Project` — stores `messages` (JSON array) and `data` (serialized VFS `Record<string, FileNode>`) as plain strings. `userId` is nullable for anonymous-created projects.

Project data is only saved when a `projectId` is provided **and** the user is authenticated (checked at the end of the `streamText` `onFinish` callback).

### Preview rendering

`src/lib/transform/jsx-transformer.ts` handles the client-side compilation pipeline:
- `transformJSX` — Babel-transpiles a single file (strips CSS imports, handles TS/TSX)
- `createImportMap` — processes all VFS files, creates blob URLs, maps `@/` aliases, resolves third-party packages to `esm.sh`, and generates placeholder modules for missing imports
- `createPreviewHTML` — builds the full `srcdoc` HTML with Tailwind CDN, the import map, an `ErrorBoundary`, and a `loadApp()` that dynamically imports the entry point

The AI is instructed (via `src/lib/prompts/generation.tsx`) to always create `/App.jsx` as the entry point and use `@/` for all local imports.

### Key conventions

- All local imports in generated code use the `@/` alias (maps to VFS root `/`)
- Every generated project must export a default React component from `/App.jsx`
- Styling is Tailwind only — no hardcoded styles in generated components
- Server-only code (auth, Prisma) uses the `server-only` package to prevent accidental client bundling
- Tests use Vitest + jsdom + Testing Library; config in `vitest.config.mts`
