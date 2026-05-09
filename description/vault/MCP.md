---
tags: [service]
status: planned
---

# MCP

MCP-сервер бэкенда. Позволяет внешним MCP-клиентам (**Claude Desktop**, **Claude Code**, любой совместимый) подключаться и работать с вакансиями. Это даёт «бесплатный» AI — расход покрывается подпиской пользователя, не per-token API.

## Транспорты

| Транспорт | Назначение | Когда |
|---|---|---|
| **stdio** | Подключение Claude Desktop через `claude_desktop_config.json` | Основной режим использования |
| **streamable HTTP** на `localhost:3845` | Внутренние UI-вызовы, отладка, тулзы для других AI-агентов в IDE | Опционально |

## Tools (MVP)

| Tool | Описание | Возвращает |
|---|---|---|
| `list_pending_vacancies()` | Вакансии в статусе `letter_pending` | `Vacancy[]` |
| `get_vacancy(id)` | Полные детали вакансии | `Vacancy` |
| `get_user_context()` | Резюме + контекст соискателя | `string` |
| `submit_cover_letter(vacancy_id, text)` | Сохранить письмо, перевести статус → `letter_ready` | `Application` |
| `get_application_status(vacancy_id)` | Текущий статус | `Status` |

## Resources (опционально)

- `resume://current` — текущее резюме как ресурс
- `prompt-template://{name}` — шаблоны промптов

## Стек

| Задача | Библиотека | Почему | Альтернативы |
|---|---|---|---|
| MCP SDK | **`mcp`** (Anthropic official, Python) | Официальный, поддерживает все транспорты, активная разработка | — |
| Декларативные tools | **`fastmcp`** | Удобная обёртка над `mcp` с декораторами `@tool` | ручной handler-стиль через `mcp` |

## Подключение Claude Desktop

Snippet для `claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "headhunter-ai": {
      "command": "/Applications/Headhunter AI.app/Contents/Resources/backend/headhunter-mcp",
      "args": ["--stdio"]
    }
  }
}
```

> [!warning] MCP-протокол молод и быстро меняется. Фиксируем версию `mcp` SDK, выделяем адаптер для абстракции на случай breaking changes.

## Связи
- [[Description]] — почему MCP, чтобы было «бесплатно»
- [[REST]] — параллельный интерфейс к тем же сущностям
- [[Domain Model]] — какие сущности экспонируются
- [[AI Layer]] — альтернатива MCP-режиму (прямой API-вызов)
