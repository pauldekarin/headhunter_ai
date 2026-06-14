---
tags: [stack]
status: planned
---

# Stack

Технологический стек проекта [[Description|Headhunter AI]] целиком. Per-task детализация и альтернативы — в заметках этапов ([[Stage 1 - MVP]], [[Stage 2 - Расширение]], [[Stage 3 - Оптимизация]]).

## Общий обзор

| Слой | Выбор | Почему |
|---|---|---|
| Desktop shell | **Tauri 2** (Rust) | ~10MB бандл vs ~150MB Electron; меньше RAM; native menus; auto-updater из коробки |
| UI framework | **SvelteKit 2 + Svelte 5** (TS) | Меньше boilerplate, runes-реактивность, отличный DX в Tauri webview |
| Стили | **Tailwind CSS 4** | Утилитарный, без рантайма |
| UI-компоненты | **shadcn-svelte** | Copy-paste, без vendor lock-in |
| i18n | **Paraglide JS** (inlang) | Codegen JSON→типизированные tree-shakeable функции, ICU plurals, нативная интеграция со SvelteKit через Vite-плагин |
| Формы | **Superforms + Zod** | Лучшая форма-библиотека для SvelteKit |
| Backend язык | **Python 3.12+** | Прямо указано в ТЗ; лучшая экосистема для Playwright и MCP SDK |
| Web-фреймворк | **FastAPI** | Async, OpenAPI auto, нативные WebSocket |
| Browser automation | **Playwright** (Python) | Auto-wait, лучше Selenium для SPA |
| Stealth | **patchright** | Drop-in замена Playwright с актуальными патчами |
| MCP SDK | **`mcp`** (Anthropic official) + **`fastmcp`** | Декларативные tools через декораторы |
| LLM-абстракция | **LiteLLM** | 100+ провайдеров, OpenAI-совместимый API |
| ORM | **SQLAlchemy 2.0 + Alembic** | Async-режим, миграции |
| Хранилище | **SQLite** + WAL | Single-user приложение |
| Драйвер SQLite | **aiosqlite** | Async wrapper |
| Очередь | **asyncio.Queue** + персистенция в SQLite | Без внешних зависимостей |
| Логи | **structlog + rich** | Структурные JSON + красивый dev-вывод |
| Конфиг | **pydantic-settings + YAML** | Типизация, валидация, env-override |
| Секреты | **`keyring`** | Системный keychain (macOS / Windows / GNOME / KWallet) |
| Tooling JS | **pnpm** workspaces | Быстро, экономит диск |
| Tooling Python | **uv** | В 10-100× быстрее pip/poetry |
| Линтеры | **ruff + mypy** (Python), **biome** (TS), **clippy** (Rust) | Быстро, единообразно |
| Тесты backend | **pytest + pytest-asyncio + httpx** | Стандарт |
| Тесты UI | **Vitest** | Vite-нативный, быстрый |
| E2E | **Playwright Test** (TS) | Тот же тулинг что для парсинга |
| CI/CD | **GitHub Actions + tauri-action** | Билд под три платформы одним workflow |
| Подпись релизов | **Tauri updater + ed25519** | Безопасные авто-обновления |
| Метрики | **prometheus-client** | Экспортер на localhost:9090 |
| Tracing | **opentelemetry-python** | Vendor-neutral |
| Локальный LLM | **Ollama** | Простейший локальный хостинг |

## Детализация по компонентам
- [[UI]] — фронт
- [[REST]] — API-слой
- [[MCP]] — MCP-сервер
- [[AI Layer]] — LLM-абстракция
- [[Chromium]] — браузерное ядро
- [[Parser service]] / [[Writer service]] — модули поверх Chromium
- [[Storage]] — хранилища
- [[Testing]] / [[Observability]]
