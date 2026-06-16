---
tags: [infra]
status: live
---

# Storage

Хранилища данных и файловая структура.

## SQLite

Расположение: `~/.headhunter_ai/db.sqlite`. Режим: **WAL** + `foreign_keys=ON` — прагмы применяются в lifespan через `apply_sqlite_pragmas(engine)`.

### Таблицы

| Таблица | Статус | Назначение / ключевые колонки |
|---|---|---|
| `vacancies` | live | Спарсенные вакансии. PK `id`, `apply_link` unique+indexed, JSON-колонки `work_formats`/`employment_types` |
| `searches` | live | История поисковых тасков. PK `id` (UUID-str), `url`, `max_pages`, `max_vacancies`, `status` (SearchStatusAPISchema enum), `parsed_pages`, `parsed_vacancies`, `started_at`, `finished_at`, `error` |
| `search_vacancies` | live | M2M-ассоциация (`search_id`, `vacancy_id`); вакансия может принадлежать нескольким поискам. Парсер делает `INSERT OR IGNORE` через `link_vacancy_to_search` |
| `applications` | live | Отклики. PK `id`, **`vacancy_id` unique+indexed (1:1 с `vacancies`)**, `status` (ProcessingState enum, indexed), `retry_count`, `error_message`, `created_at`, `updated_at` (onupdate) |
| `cover_letters` | live | Версионированные тексты. PK `id`, FK `application_id` (indexed), `version` (default=1), `text`, `created_at` |
| `settings` | live | Синглтон. PK `id=1` (`autoincrement=False`) + **`CHECK (id = 1)`**. Колонки: `letter_style`, `resume_text`, `max_pages`, `max_vacancies`, `daily_limit`, `hourly_limit`, `min_delay_ms`, `delay_jitter_ms`, `auto_submit`, `llm_deployments` (JSON `list[LLMDeployment]`), `llm_system_prompt` |
| `rate_limits` | live | Sliding-window event log (`RateLimitEventORM`). PK `id`, `occurred_at`. Используется в `orchestrator/rate_limiter.py` (`ensure_within_limits`, `get_used_hourly_limits`, `get_used_daily_limits`) |
| `prompts` | planned (Stage 2) | Имена промпт-шаблонов |
| `prompt_versions` | planned (Stage 2) | Версии шаблонов с метриками |
| `audit_log` | planned | Все действия (парсинг, генерация, отправка, ошибки) — отдельная задача |

### Связи

```
Vacancy ──1:1── Application ──1:M── CoverLetter
   ╰── M2M (search_vacancies) ── SearchHistory
```

### Стек

| Задача | Библиотека | Почему | Альтернативы | Подводные камни |
|---|---|---|---|---|
| ORM | **SQLAlchemy 2.0** | Async, типизированный mapper, де-факто стандарт | SQLModel (надстройка для FastAPI), Tortoise ORM | Async требует aiosqlite |
| Драйвер | **aiosqlite** | Async wrapper над sqlite3 | sqlite-utils (sync) | Single-writer SQLite — ок для single-user |
| Миграции | **Alembic** | Стандарт для SQLAlchemy | yoyo-migrations (проще, менее powerful) | Async-revisions требуют отдельной настройки |

## Файловая система

```
~/.headhunter_ai/
├── db.sqlite              ← основная БД
├── chrome-profile/        ← Chromium user-data-dir (см. [[Chromium]])
├── prompts/               ← YAML-шаблоны промптов
│   ├── default.yaml
│   └── ...
├── consent.json           ← согласие с ToS-дисклеймером
├── logs/                  ← структурные JSON-логи (ротация по дням)
└── backups/               ← еженедельные бэкапы DB (ротация 4 шт.)
```

## Бэкапы

Авто-бэкап раз в неделю через `sqlite3 .backup` (атомарный, не блокирует writers с WAL). Ротация — последние 4 копии. См. [[Stage 3 - Оптимизация]] задачу 3.6.

## Шифрование (опционально)

> [!question] Шифровать ли локальную БД с резюме через SQLCipher? См. [[Open Questions]] #3.

## Связи
- [[Domain Model]] — какие сущности хранятся
- [[REST]] — CRUD-потребитель
- [[Anti-bot]] — `rate_limits`
