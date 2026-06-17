---
tags: [stack]
status: live
---

# Stack

Технологический стек проекта [[Description|Headhunter AI]] целиком. Per-task детализация и альтернативы — в заметках этапов ([[Stage 1 - MVP]], [[Stage 2 - Расширение]], [[Stage 3 - Оптимизация]]).

> [!note] Метка «(live)» — реально подключено и используется в коде. «(planned)» — в плане, но ещё не в `pyproject.toml`/`package.json`.

## Общий обзор

| Слой | Выбор | Статус | Почему |
|---|---|---|---|
| Desktop shell | **Tauri 2** (Rust) | live | ~10MB бандл vs ~150MB Electron; меньше RAM; native menus; auto-updater из коробки |
| UI framework | **SvelteKit 2 + Svelte 5** (TS) | live | Меньше boilerplate, runes-реактивность, отличный DX в Tauri webview |
| Стили | **Tailwind CSS 4** | live | Утилитарный, без рантайма |
| UI-компоненты | **shadcn-svelte** (через `bits-ui` + `tailwind-variants`) | live | Copy-paste в `src/lib/components/ui/`, без vendor lock-in |
| Иконки | **`@lucide/svelte`** | live | Reactив-friendly Svelte 5 биндинги. Везде используются per-icon импорты (`@lucide/svelte/icons/x`) — barrel `from "@lucide/svelte"` дико тормозит Vite dev из-за on-demand resolve тысяч модулей |
| Drag-and-drop | **`@dnd-kit/svelte`** + `/sortable` | live | Sortable list в Settings AI tab (`createSortable` per item + `<DragDropProvider>` + ручной `arrayMove`). Адаптер отключает `OptimisticSortingPlugin` — reorder идёт через реактивный `{#each}` |
| Toast/Sonner | **`svelte-sonner`** | live | Используется через `<Toaster />` в layout + `toast.success/error` |
| Light/Dark | **`mode-watcher`** | live | Системная тема + переключатель |
| i18n | **Paraglide JS** (`@inlang/paraglide-js` + `@inlang/cli`) | live | Codegen в `apps/desktop/src/lib/paraglide/`; baseLocale `ru` |
| Server cache | **`@tanstack/svelte-query`** | live | Источник истины для `settings`, `vacancies`, `search`. `setQueryData(saved)` после PUT вместо invalidate |
| Формы | **`sveltekit-superforms` + `formsnap`** + **`zod` v4** (через `zod4` адаптер) | live | SPA-режим, `dataType: "json"`, `resetForm: false`. Адаптер `zod4` — потому что superforms ещё не дефолтит на Zod 4 |
| Adapter | **`@sveltejs/adapter-static`** | live | SSR не нужен, webview грузит из FS |
| Backend язык | **Python 3.12+** | live | Прямо указано в ТЗ; лучшая экосистема для Playwright и MCP SDK |
| Web-фреймворк | **FastAPI** (`fastapi[standard]`) | live | Async, OpenAPI auto, нативные WebSocket |
| ASGI | **uvicorn** | live | Запуск через `headhunter_backend.api.app:run` (`uvicorn.run(...)`, порт 8001) |
| Browser automation | **Playwright** (Python) — через `patchright` | live | Auto-wait, лучше Selenium для SPA. `patchright` — fork с актуальными stealth-патчами; единая точка входа `browser/core.py` |
| HTML-парсинг | **`selectolax`** | live | На порядок быстрее BeautifulSoup |
| MCP SDK | **`mcp`** (Anthropic official) + **`fastmcp`** | planned (Stage 2) | На текущий момент в `pyproject.toml` отсутствуют |
| LLM-абстракция | **LiteLLM** (`litellm.Router`) | live | 100+ провайдеров, OpenAI-совместимый API, встроенный fallback |
| ORM | **SQLAlchemy 2.0 + Alembic** | live | Async-режим, миграции в `services/backend/alembic/` |
| Хранилище | **SQLite** + WAL + `foreign_keys=ON` | live | `~/.headhunter_ai/db.sqlite`, прагмы применяются в lifespan |
| Драйвер SQLite | **aiosqlite** | live | Async wrapper |
| Очередь | **asyncio.Queue** + персистенция в SQLite (`Orchestrator.recover_from_db`) | live | Без внешних зависимостей |
| State machine | **`python-statemachine`** | live | `ProcessingStateMachine` ([[Domain Model#State machine `Application`]]) |
| Логи | **structlog + rich** | live | Структурные JSON + красивый dev-вывод |
| Rate-limit | свой sliding-window поверх таблицы `rate_limits` (`RateLimitEventORM`) | live | Без библиотеки `limits` — простая хранимая модель event'ов с `occurred_at` |
| Конфиг | **pydantic-settings + YAML** | planned | Сейчас все настройки в БД (`settings` singleton). YAML/env-override — отдельная задача |
| Секреты | **`keyring`** | planned | Сейчас `api_key` лежит в `settings.llm_deployments` (JSON в SQLite) — keychain интеграция отложена |
| Tooling JS | **pnpm** workspaces | live | Быстро, экономит диск |
| Tooling Python | **uv** | live | В 10-100× быстрее pip/poetry |
| Линтеры | **ruff + mypy** (Python), **biome** (TS), **clippy** (Rust) | partial | mypy `strict=true` в `pyproject.toml`; ruff/biome/clippy через pre-commit |
| Тесты backend | **pytest + pytest-asyncio + httpx** | live deps, sparse coverage | `pytest-asyncio>=0.24`, `asyncio_mode = "auto"`. Тестов пока мало |
| Тесты UI | **Vitest** | live deps | `vitest>=2.1.0`, тестов пока нет |
| E2E | **Playwright Test** (TS) | planned | Stage 2 (2.7) |
| CI/CD | **GitHub Actions + tauri-action** | planned | Stage 0 закрыт каркас, релизы — Stage 2/3 |
| Подпись релизов | **Tauri updater + ed25519** | planned | Stage 3 |
| Метрики | **prometheus-client** | planned | Stage 3 (3.x — Observability) |
| Tracing | **opentelemetry-python** | planned | Stage 3 |
| Локальный LLM | **Ollama** | supported by LiteLLM | Через `litellm.Router` с OpenAI-совместимым `api_base`; в коде нет специальных хелперов |

## Детализация по компонентам
- [[UI]] — фронт
- [[REST]] — API-слой
- [[MCP]] — MCP-сервер
- [[AI Layer]] — LLM-абстракция
- [[Chromium]] — браузерное ядро
- [[Parser service]] / [[Writer service]] — модули поверх Chromium
- [[Storage]] — хранилища
- [[Testing]] / [[Observability]]
