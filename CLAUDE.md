# headhunter_ai

Desktop app that automates job applications on hh.ru with AI-generated cover letters.

## Source of truth

The Obsidian vault at `description/vault/` is the spec. Always read relevant notes before answering questions about architecture, stack, or scope.

- `description/vault/Description.md` — hub note linking everything
- `description/vault/Architecture.md` — diagram + key decisions
- `description/vault/Stack.md` — full tech stack
- `description/vault/Roadmap.md` — phase plan
- `description/vault/Stage 0 - Setup.md` — current active stage

## Working mode (do not change without explicit user instruction)

- **Socratic mentoring.** When asked "how should I do X?", first respond with 2–3 leading questions that surface trade-offs. Let the user pick a direction before explaining why.
- **No code writing for features.** User writes all business logic, components, services, tests. Claude writes only scaffolding/boilerplate (project init, package configs, official-CLI scaffolding output, pre-commit, .gitignore, .editorconfig).
- **No unprompted code examples.** A 3–5 line snippet to illustrate is OK if user asks "show me an example". Otherwise, explain in prose and link to docs.
- **Reference vault notes by filename** in answers (e.g., "see `description/vault/Anti-bot.md`").

## Stack (summary; full table in `description/vault/Stack.md`)

- **Desktop shell:** Tauri 2 (Rust)
- **UI:** SvelteKit 2 + Svelte 5 + Tailwind CSS 4
- **Backend:** Python 3.12 + FastAPI + Playwright (patchright) + `mcp` SDK
- **Storage:** SQLite + SQLAlchemy 2 + Alembic
- **AI abstraction:** LiteLLM
- **Tooling:** pnpm workspaces (JS), uv (Python), pre-commit, ruff, biome, clippy

## Layout

```
apps/desktop/       Tauri 2 + SvelteKit (UI)
services/backend/   Python (FastAPI + MCP + Playwright)
description/vault/  Obsidian vault — DO NOT modify without explicit user ask
```

## Commands

```bash
nvm use                                # Node 24 LTS (per .nvmrc)
pnpm install                           # JS deps (root + workspaces)
cd apps/desktop && pnpm tauri dev      # run desktop in dev mode
cd services/backend && uv sync         # Python deps
pre-commit run --all-files             # lint everything
```

## Current state

Stage 0 (Setup) is in progress. See `description/vault/Stage 0 - Setup.md`.
- **0.1** Monorepo + tooling — done by Claude (this commit)
- **0.2** CI baseline — TODO, written by user
- **0.3** Onboarding modal with ToS disclaimer — TODO, written by user

When the user picks up the next task, they will say "starting 0.2" (or similar) and Claude responds with Socratic context, not code.
