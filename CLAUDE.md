# headhunter_ai

Desktop app that automates job applications on hh.ru with AI-generated cover letters.

## Source of truth

The Obsidian vault at `description/vault/` is the spec. Always read relevant notes before answering questions about architecture, stack, or scope.

- `description/vault/Description.md` — hub note linking everything
- `description/vault/Architecture.md` — diagram + key decisions
- `description/vault/Stack.md` — full tech stack
- `description/vault/Roadmap.md` — phase plan
- `description/vault/Stage 1 - MVP.md` — current active stage

## Working mode (do not change without explicit user instruction)

- **Socratic mentoring.** When asked "how should I do X?", first respond with 2–3 leading questions that surface trade-offs. Let the user pick a direction before explaining why.
- **No code writing for features.** User writes all business logic, components, services, tests. Claude writes only scaffolding/boilerplate (project init, package configs, official-CLI scaffolding output, pre-commit, .gitignore, .editorconfig).
- **No unprompted code examples.** A 3–5 line snippet to illustrate is OK if user asks "show me an example". Otherwise, explain in prose and link to docs.
- **Reference vault notes by filename** in answers (e.g., "see `description/vault/Anti-bot.md`").

## Stack (summary; full table in `description/vault/Stack.md`)

- **Desktop shell:** Tauri 2 (Rust)
- **UI:** SvelteKit 2 + Svelte 5 (runes) + Tailwind CSS 4 + shadcn-svelte
- **UI data:** TanStack Query (server cache), Superforms + Zod v4 (forms), Paraglide JS (i18n, baseLocale `ru`)
- **Backend:** Python 3.12 + FastAPI + Playwright (patchright)
- **State machine:** `python-statemachine`
- **Storage:** SQLite (WAL) + SQLAlchemy 2 (async/aiosqlite) + Alembic
- **AI abstraction:** LiteLLM (`litellm.Router` with fallback chain)
- **Tooling:** pnpm workspaces (JS), uv (Python), pre-commit, ruff, biome, clippy
- **MCP:** deferred to Stage 2 — no `mcp` SDK in backend yet

## Layout

```
apps/desktop/       Tauri 2 + SvelteKit (UI)
services/backend/   Python (FastAPI + MCP + Playwright)
description/vault/  Obsidian vault — keep in sync with code; update when architecture/stack/scope shifts
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

Stage 0 (Setup) complete. Stage 1 (MVP) deep in flight. See `description/vault/Stage 1 - MVP.md`.

- **1.1** BrowserCore (persistent profile, stealth) — done
- **1.2** Auth-flow — done (`/auth` routes + `AuthWSEvent`)
- **1.3** Parser module — done (`browser/parser.py` streams `AsyncIterator[Vacancy]`)
- **1.4** FastAPI backend skeleton — done
- **1.5** SQLite + migrations — done
- **1.6** Orchestrator + queue — done (consumer with auth/rate-limit/captcha/fail handling, `recover_from_db` on startup, SearchService picker + scoped queue + cancel)
- **1.7** Backend API foundation — done (real endpoints over state machine; `cover_letters` + `settings` tables; settings get-or-create)
- **1.8** Writer module — done (`BrowserWriter.submit` + captcha-pause + rate-limit gate)
- **1.9** UI: SearchForm + VacancyList — done (queue page with picker + scoped list)
- **1.10** UI: LetterReview — done (Sheet-drawer in `lib/components/letter-review-sheet.svelte` + `letterReview` rune-store; preview/edit/regen/save/submit/skip + History tab with restore)
- **1.11** UI: Settings — done (search/user/limits + AI tab with resume/style/system-prompt + LLM deployments editor: shadcn Accordion + `@dnd-kit/svelte` reorder + per-deployment Eye/EyeOff key toggle)
- **1.12** AI Layer — done (`AILayer.generate_cover_letter` + `/api/v1/ai` routes + `AutoApplyService` for auto-submit path)

> [!note] AILayer rebuild on PUT /settings is wired in `api/routes/settings.py:24` — `ai_layer.rebuild(...)` fires after `update_settings`. Stage 1 (MVP) is functionally complete; remaining work is verification/demo.
