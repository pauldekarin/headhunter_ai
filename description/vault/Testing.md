---
tags: [infra]
status: partial
---

# Testing

Тестовая стратегия проекта.

> [!warning] **Текущее состояние:** инфраструктура подключена (pytest + pytest-asyncio в `pyproject.toml`, vitest в `package.json`), но реального покрытия пока почти нет. Это сознательный долг — закроется отдельной задачей до выхода на MVP-демо (или в Stage 2.7).

## Тестовая пирамида

| Уровень | Инструмент | Что покрывает |
|---|---|---|
| Unit (backend) | **pytest + pytest-asyncio** | Orchestrator state machine, [[AI Layer]], парсеры HTML-фикстур |
| Unit (UI) | **Vitest** | Компоненты Svelte, stores, утилиты |
| Component | **Playwright Component Testing** | Интеграция Svelte-компонентов с моками REST |
| Integration | **pytest + httpx** (in-process FastAPI) | REST endpoints ([[REST]]), WebSocket flow, MCP tools ([[MCP]]) |
| E2E | **Playwright Test** (TS) | Полный флоу против fixture-сервера hh.ru, запускается в Tauri-окне |
| Smoke | **Daily GitHub Action** | Проверка селекторов hh.ru на реальной выдаче (read-only, без откликов) |

## Стек

| Задача | Библиотека | Почему | Альтернативы |
|---|---|---|---|
| Test runner Python | **pytest** | Стандарт | unittest (verbose) |
| Async-тесты | **pytest-asyncio** | Native asyncio | anyio |
| HTTP-клиент | **httpx** | Async, поддерживает ASGI in-process | requests (sync) |
| HTML-фикстуры | хранить как `.html` файлы в `tests/fixtures/` | Воспроизводимо, можно diff | record-replay |
| Mock LLM | **respx** или мок-провайдер LiteLLM | Подменяет HTTP | unittest.mock |
| E2E TS | **@playwright/test** | Видеозапись, отчёты, retry | Cypress (хуже мульти-табы) |
| Mock hh.ru | **mitmproxy + HAR** или fixture-сервер | Воспроизводимо | wiremock |

## Покрытие (целевое)

| Слой | Покрытие |
|---|---|
| Orchestrator | > 80% |
| [[AI Layer]] | > 70% |
| [[REST]] endpoints | > 90% |
| [[MCP]] tools | > 90% |
| UI компоненты | > 60% |

## Запрещено

> [!danger] **E2E против реального hh.ru запрещены ToS.** Все автоматические тесты — против локального fixture-сервера или mitmproxy с записанными HAR.

Daily smoke-test (read-only выдача, без откликов) — допустим как «канарейка», но без логина и без массового скрапинга.

## CI

GitHub Actions matrix `[ubuntu, macos, windows]`:
- Lint (ruff + mypy + biome + clippy)
- Unit-тесты
- Integration-тесты
- E2E против fixture-сервера

## См. также
- [[Stage 2 - Расширение]] задача 2.7
- [[Observability]]
