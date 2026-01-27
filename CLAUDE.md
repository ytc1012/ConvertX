# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

ConvertX is a self-hosted online file conversion service supporting 1,000+ formats across 20+ external converters (FFmpeg, ImageMagick, LibreOffice, Pandoc, Calibre, etc.). Built with Bun, Elysia, TypeScript, and server-side rendered JSX.

## Development Commands

```bash
# Start development server (watch mode)
bun run dev

# Start with hot module replacement
bun run hot

# Build for production (generates CSS + compiles TypeScript)
bun run build

# Run all tests
bun test

# Type checking
bun run lint:tsc

# Find dead code
bun run lint:knip

# Format code
bun run format

# Lint everything
bun run lint
```

## Architecture

### Converter Plugin System

The core architecture is a **converter registry** in [src/converters/main.ts](src/converters/main.ts). Each converter module exports:
- `properties`: Object with `from` (input formats), `to` (output formats), and `options`
- `convert`: Async function `(filePath, fileType, convertTo, targetPath, options) => Promise<string>`

Converters are manually registered in the properties object. The `mainConverter()` function dispatches to the appropriate converter based on input/output format matching.

**Adding a new converter** requires:
1. Create [src/converters/[name].ts](src/converters/) with exports
2. Register in [src/converters/main.ts](src/converters/main.ts)
3. Create [tests/converters/[name].test.ts](tests/converters/) with mocked `execFile` tests
4. Add binary to [Dockerfile](Dockerfile)
5. Add version check to [src/helpers/printVersions.ts](src/helpers/printVersions.ts)

### Elysia Server Pattern

Authentication is implemented as an **Elysia service plugin** in [src/pages/user.tsx](src/pages/user.tsx):
- `userService` plugin with JWT setup
- `.macro("auth")` decorator validates tokens pre-route
- Protected routes use `{ auth: true }` option

### Server-Side JSX (Not React)

Uses `@kitajs/html` for type-safe JSX that compiles to HTML strings at runtime. No virtual DOM, no client framework. Components in [src/components/](src/components/) are functions returning JSX. Client-side interactivity is vanilla JS in [public/script.js](public/script.js).

### File Processing Pipeline

1. **Upload** ([src/pages/upload.tsx](src/pages/upload.tsx)): Files saved to `data/uploads/{user_id}/{job_id}/`
2. **Convert** ([src/pages/convert.tsx](src/pages/convert.tsx)): Creates job, spawns background `handleConvert()`, processes in chunks (controlled by `MAX_CONVERT_PROCESS`), saves to `data/output/{user_id}/{job_id}/`
3. **Results**: Client polls completion or views history

### Database Schema

SQLite with WAL mode. Three tables with foreign key relationships:
- `users` (id, email, password)
- `jobs` (id, user_id, date_created, status, num_files)
- `file_names` (id, job_id, file_name, output_file_name, status)

Auto-cleanup runs based on `AUTO_DELETE_EVERY_N_HOURS`. Schema in [src/db/db.ts](src/db/db.ts).

## Key Directories

- [src/converters/](src/converters/) - File converter implementations
- [src/db/](src/db/) - SQLite schema and models
- [src/pages/](src/pages/) - Elysia route handlers (SSR pages)
- [src/components/](src/components/) - Reusable JSX components
- [src/helpers/](src/helpers/) - Utility functions (env, filetype normalization, Tailwind generation)
- [tests/converters/](tests/converters/) - Unit tests for converters (mock `execFile` pattern)

## Important Patterns

### Filetype Normalization

Critical for format matching ([src/helpers/normalizeFiletype.ts](src/helpers/normalizeFiletype.ts)):
- Input: `jpg` → `jpeg`, `md` → `markdown`
- Output: `jpeg` → `jpg`, `markdown` → `md`

Allows flexible user input while maintaining converter compatibility.

### Environment Configuration

All environment variables centralized in [src/helpers/env.ts](src/helpers/env.ts) with proper defaults and type coercion. Important: `JWT_SECRET` falls back to `randomUUID()` if unset, but should be set in production.

### TailwindCSS v4

Uses `@tailwindcss/cli` for CSS generation, not PostCSS. Dev mode generates in-memory via `/generated.css`; production writes to [public/generated.css](public/generated.css). Custom `@theme` block in [src/main.css](src/main.css) defines CSS variable integration.

## Code Style

- **Conventional commits** required (enforced by CI)
- **Prettier**: 2 spaces, 100 char width, double quotes, semicolons
- **TypeScript strict mode** enabled with `noUncheckedIndexedAccess`
- **Import type**: Type-only imports must use `import type`
- Tests use **mocking pattern** - override `execFile` to avoid external dependencies

## Security Notes

- HTTP-only auth cookies (rejected over HTTP unless `HTTP_ALLOWED=true`)
- All filenames sanitized with `sanitize-filename`
- First user created becomes admin (check `FIRST_RUN` flag)
- No streaming for large files - entire files loaded into memory

## Deployment

Docker-first with multi-stage build. Requires 20+ system packages (FFmpeg, LibreOffice, etc.). Data persistence via `/app/data` volume. Port always 3000 internally.
