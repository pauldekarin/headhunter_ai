---
tags: [service]
status: planned
---

# REST

REST + WebSocket API для собственного [[UI]]. Реализован на FastAPI как часть Python-бэка (того же процесса, что и orchestrator и [[AI Layer]]). [[MCP]] — отдельный процесс в `services/mcp/`, ходит в backend через тот же REST.

## Endpoints (MVP)

Все REST-роуты под префиксом `/api/v1`. WebSocket — корневой `/ws/...`.

### Search picker (двухшаговый запуск поиска)

Пользователь сначала открывает hh.ru в Chromium, руками выставляет фильтры, потом подтверждает — backend читает URL из активной вкладки. См. [[Parser service]].

| Метод | Путь | Назначение |
|---|---|---|
| `POST` | `/api/v1/search/picker/new` | Открыть hh.ru в Chromium, вернуть `session_id` |
| `POST` | `/api/v1/search/picker/{session_id}/confirm` | Прочитать URL из открытой вкладки (валидирует домен hh.ru), закрыть вкладку, вернуть URL |
| `POST` | `/api/v1/search/picker/{session_id}/cancel` | Закрыть вкладку без подтверждения |

### Search lifecycle

| Метод | Путь | Назначение |
|---|---|---|
| `POST` | `/api/v1/search/vacancies/new` | Запустить парсинг по `VacanciesStartSearchRequestAPISchema` (URL + опциональные `max_pages`/`max_vacancies`; дефолты из `settings`) |
| `GET` | `/api/v1/search/vacancies/current` | Активный поисковый таск или `204 No Content` |
| `GET` | `/api/v1/search/vacancies/{search_id}` | Состояние таска по ID (in-memory; для завершённых смотри `searches` через будущий history-эндпоинт) |
| `DELETE` | `/api/v1/search/vacancies/{search_id}` | Отменить активный поиск |

### Vacancies

| Метод | Путь | Назначение |
|---|---|---|
| `GET` | `/api/v1/vacancies/?search_id=latest\|all\|<uuid>` | Список вакансий с фильтром. `latest` (по умолчанию) — текущий или последний завершённый поиск; `all` — без фильтра; UUID — конкретный поиск из истории |
| `GET` | `/api/v1/vacancies/{vacancy_id}` | Детали вакансии |

### Application lifecycle (state machine)

| Метод | Путь | Назначение |
|---|---|---|
| `POST` | `/api/v1/vacancies/{vacancy_id}/queue_for_letter` | Создать `application`, перевести в `letter_pending` |
| `POST` | `/api/v1/vacancies/{vacancy_id}/cover_letter` | Сохранить отредактированное письмо, перевести в `letter_ready` |
| `POST` | `/api/v1/vacancies/{vacancy_id}/submit` | Поставить в очередь на отправку (`letter_sending` → `letter_sent`) |
| `POST` | `/api/v1/vacancies/{vacancy_id}/retry` | Повторить отправку после `error` |
| `GET` | `/api/v1/vacancies/{vacancy_id}/status` | Текущий статус application'а |

### AI Layer

| Метод | Путь | Назначение |
|---|---|---|
| `POST` | `/api/v1/ai/create_cover_letter/{vacancy_id}` | Сгенерировать письмо через [[AI Layer]] (Router + fallback). Возвращает text, model_used, tokens, was_fallback, cost_usd |
| `GET` | `/api/v1/ai/health` | Статус AI-слоя: `healthy` / `unhealthy` / `no_deployments` |

### Settings / Orchestrator

| Метод | Путь | Назначение |
|---|---|---|
| `GET` | `/api/v1/settings` | Получить настройки (singleton row) |
| `PUT` | `/api/v1/settings` | Обновить настройки; AI Layer пересобирает Router |
| `POST` | `/api/v1/orchestrator/resume` | Снять паузу с consumer-loop'а (после капчи/rate-limit) |

### WebSocket

| Метод | Путь | Назначение |
|---|---|---|
| `WS` | `/ws/vacancies` | Снэпшот текущих вакансий из БД (отправка и close) |
| `WS` | `/ws/events` | Подписка на orchestrator-события (submission_event, captcha_event, search_event) |

## Стек

| Задача | Библиотека | Почему | Альтернативы |
|---|---|---|---|
| Web framework | **FastAPI** | Async, OpenAPI auto, WebSocket native, типизация Pydantic | Litestar (быстрее, моложе), Starlette (low-level) |
| ASGI server | **uvicorn** | Стандарт для FastAPI | hypercorn (HTTP/2, медленнее) |
| Валидация | **Pydantic v2** | Идёт в комплекте с FastAPI | — |
| Документация | **встроенный Swagger UI** на `/docs` | Бесплатно с FastAPI | redoc |

## WebSocket

> [!note] В исходном ТЗ было сомнение: «решение поднимать вебсокет сервер скорее всего избыточно». Это устаревшее мнение — FastAPI поддерживает WebSocket нативно через Starlette/uvicorn с минимумом кода. Используем без оговорок.

Канал `/ws/vacancies` подписан на orchestrator events; новые вакансии от [[Parser service]] стримятся в [[UI]] по мере появления.

## Связи
- [[UI]] — единственный (или один из) клиентов
- [[MCP]] — параллельный интерфейс (отдельный процесс в Stage 2, ходит через REST)
- [[AI Layer]] — собственный router `/api/v1/ai` (тот же процесс)
- [[Storage]] — CRUD сущностей
- [[Architecture]] — общая картина
