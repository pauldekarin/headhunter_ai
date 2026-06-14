---
tags: [infra]
status: planned
---

# Storage

Хранилища данных и файловая структура.

## SQLite

Расположение: `~/.headhunter_ai/db.sqlite`. Режим: **WAL** (Write-Ahead Logging) для конкурентного чтения.

### Таблицы

| Таблица | Назначение |
|---|---|
| `vacancies` | спарсенные вакансии ([[Domain Model]]); `search_id` FK → `searches.id` (nullable для legacy) |
| `searches` | история поисковых тасков: URL, лимиты, статус, прогресс, `started_at`/`finished_at`, error |
| `applications` | отклики (vacancy + letter + status) |
| `cover_letters` | сгенерированные тексты с версиями |
| `settings` | настройки приложения — синглтон (одна строка, фиксированный PK) |
| `prompts` | имена промпт-шаблонов |
| `prompt_versions` | версии шаблонов с метриками |
| `audit_log` | все действия (парсинг, генерация, отправка, ошибки) |
| `rate_limits` | состояние token-bucket (см. [[Anti-bot]]) |

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
