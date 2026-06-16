---
tags: [service]
status: live
---

# REST

REST + WebSocket API для собственного [[UI]]. Реализован на FastAPI как часть Python-бэка (того же процесса, что и orchestrator и [[AI Layer]]). [[MCP]] — отдельный процесс в `services/mcp/` (Stage 2), на текущий момент не существует.

> [!note] Все таблицы ниже отражают **фактически зарегистрированные** роутеры в `api/app.py` и эндпоинты в `api/routes/*.py`. CORS — `allow_origins=["*"]`. Регистрация ошибок — через `register_error_handlers`.

## Endpoints (live)

Все REST-роуты под префиксом `/api/v1`. WebSocket — корневой `/ws/...`.

### Auth (`/api/v1/auth`)

| Метод | Путь | Назначение |
|---|---|---|
| `GET` | `/api/v1/auth/status` | Текущий статус авторизации hh.ru: `authorized` / `unauthorized` / `authorizing` |
| `POST` | `/api/v1/auth` | Запустить auth-flow (открыть видимый Chromium на странице логина hh.ru, ждать ручного логина) |

### Search picker (`/api/v1/search/picker`) — двухшаговый запуск поиска

Пользователь сначала открывает hh.ru в Chromium, руками выставляет фильтры, потом подтверждает — backend читает URL из активной вкладки. См. [[Parser service]].

| Метод | Путь | Назначение |
|---|---|---|
| `POST` | `/api/v1/search/picker/new` | Открыть hh.ru в Chromium, вернуть `session_id` |
| `POST` | `/api/v1/search/picker/{session_id}/confirm` | Прочитать URL из открытой вкладки (валидирует домен hh.ru), закрыть вкладку, вернуть URL |
| `POST` | `/api/v1/search/picker/{session_id}/cancel` | Закрыть вкладку без подтверждения |

### Search lifecycle (`/api/v1/search/vacancies`)

| Метод | Путь | Назначение |
|---|---|---|
| `POST` | `/api/v1/search/vacancies/new` | Запустить парсинг по `VacanciesStartSearchRequestAPISchema` (URL + опциональные `max_pages`/`max_vacancies`; дефолты из `settings`) |
| `GET` | `/api/v1/search/vacancies/current` | Активный поисковый таск или `204 No Content` |
| `GET` | `/api/v1/search/vacancies/{search_id}` | Состояние таска по ID (in-memory; для завершённых — `searches` таблица, history-эндпоинт ещё не сделан) |
| `DELETE` | `/api/v1/search/vacancies/{search_id}` | Отменить активный поиск |

### Vacancies (`/api/v1/vacancies`)

| Метод | Путь | Назначение |
|---|---|---|
| `GET` | `/api/v1/vacancies/?search_id=latest\|all\|<uuid>` | Список вакансий с фильтром. `latest` (по умолчанию) — текущий или последний завершённый поиск; `all` — без фильтра; UUID — конкретный поиск из истории |
| `GET` | `/api/v1/vacancies/{vacancy_id}` | Детали вакансии |

### Application lifecycle (`/api/v1/vacancies/{vacancy_id}/*`)

State machine — см. [[Domain Model#State machine `Application`]].

| Метод | Путь | Назначение |
|---|---|---|
| `POST` | `/queue_for_letter` | Создать `application`, перевести в `letter_pending` |
| `POST` | `/review` | Перевести в `letter_reviewing` (manual mode) |
| `POST` | `/skip` | Перевести в `skipped` (терминальное) |
| `POST` | `/submit` | Поставить в очередь на отправку (`letter_sending`), Orchestrator забирает дальше |
| `POST` | `/retry` | После `error` → обратно в `letter_pending` |
| `GET` | `/status` | Текущий статус application'а |
| `GET` | `/cover_letter` | Получить последнюю версию письма для вакансии |
| `POST` | `/cover_letter` | Сохранить новую версию письма (`CoverLetterRequestAPISchema { text }`) |

### Applications (`/api/v1/applications`)

| Метод | Путь | Назначение |
|---|---|---|
| `GET` | `/api/v1/applications/` | Список всех applications (DESC по `updated_at`/`created_at`) |
| `GET` | `/api/v1/applications/{application_id}` | Заявка по ID |

### AI Layer (`/api/v1/ai`)

| Метод | Путь | Назначение |
|---|---|---|
| `POST` | `/api/v1/ai/create_cover_letter/{vacancy_id}` | Сгенерировать письмо через [[AI Layer]] (Router + fallback). Возвращает `AICoverLetterResponseAPISchema` (text, model_used, prompt/completion/total_tokens, was_fallback, cost_usd) |
| `GET` | `/api/v1/ai/health` | Статус AI-слоя: `healthy` / `unhealthy` / `no_deployments` |

### Settings (`/api/v1/settings`)

| Метод | Путь | Назначение |
|---|---|---|
| `GET` | `/api/v1/settings` | Получить настройки (`SettingsAPISchema`: search, user, llm, rate_limits — singleton row) |
| `PUT` | `/api/v1/settings` | Обновить настройки целиком |

> [!warning] **Open gap (см. [[Stage 1 - MVP#1.11]])**: PUT settings **не пересобирает** AILayer Router — `bootstrap_ai_layer` отрабатывает только на startup. Изменение `llm_deployments` через UI требует рестарта backend.

### Orchestrator (`/api/v1/orchestrator`)

| Метод | Путь | Назначение |
|---|---|---|
| `POST` | `/api/v1/orchestrator/resume` | Снять паузу с consumer-loop'а (после капчи / not-authorized / missing-cover-letter) |
| `GET` | `/api/v1/orchestrator/status` | `OrchestratorStatusAPISchema { paused, reason, queue_size, queue: [application_id, ...] }` |

### Rate limits (`/api/v1/rate-limits`)

| Метод | Путь | Назначение |
|---|---|---|
| `GET` | `/api/v1/rate-limits/budget` | Текущий бюджет (`RateLimitsBudgetAPISchema`): hourly + daily — каждый с `used / limit / resets_at` |

### WebSocket (вне `/api/v1`)

| Метод | Путь | Назначение |
|---|---|---|
| `WS` | `/ws/vacancies` | Снэпшот текущих вакансий из БД (legacy: отправка + close) |
| `WS` | `/ws/events` | Подписка на events. Типы: `application_event`, `search_event`, `captcha_event`, `vacancy_new`, `auth_changed` (см. `api/events.py`) |

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
