---
tags: [service]
status: planned
---

# REST

REST + WebSocket API для собственного [[UI]]. Реализован на FastAPI как часть Python-бэка (того же процесса, что и [[MCP]] и orchestrator).

## Endpoints (MVP)

| Метод | Путь | Назначение |
|---|---|---|
| `POST` | `/api/search` | Запустить поиск с `SearchFilter` |
| `GET` | `/api/vacancies` | Получить список вакансий с пагинацией и фильтрами по статусу |
| `GET` | `/api/vacancies/{id}` | Детали вакансии |
| `POST` | `/api/write/{vacancy_id}` | Сохранить сгенерированное письмо |
| `POST` | `/api/submit/{vacancy_id}` | Отправить отклик через [[Writer service]] |
| `GET` | `/api/settings` | Получить настройки |
| `PUT` | `/api/settings` | Обновить настройки |
| `WS` | `/ws/vacancies` | Стрим вакансий в реальном времени |
| `WS` | `/ws/events` | События orchestrator (status changes, errors, captcha) |

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
- [[MCP]] — параллельный интерфейс к orchestrator (тот же процесс)
- [[Storage]] — CRUD сущностей
- [[Architecture]] — общая картина
