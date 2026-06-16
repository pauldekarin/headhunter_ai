---
tags: [architecture]
status: live
---

# Architecture

Общая архитектура [[Description|Headhunter AI]].

> [!summary] **Live на текущий момент:** все блоки диаграммы (UI ↔ Backend ↔ Browser ↔ Storage ↔ LLMs) — собраны и связаны в `api/app.py` lifespan. **MCP блок и Claude Desktop путь — отложены до Stage 2** (`mcp` SDK не в `pyproject.toml`). См. [[Roadmap]] и [[Stage 1 - MVP]].

## Диаграмма

```mermaid
graph TB
    subgraph "Desktop App (Tauri)"
        UI["Svelte UI<br/>(webview)"]
        TauriCore["Tauri Core (Rust)<br/>process supervisor + IPC"]
    end

    subgraph "Backend (Python sidecar)"
        REST["FastAPI<br/>REST + WebSocket"]
        MCP["MCP Server<br/>(stdio + HTTP)"]
        Orchestrator["Orchestrator<br/>queue + state machine"]
        AILayer["LLM Abstraction<br/>(LiteLLM)"]
    end

    subgraph "Browser Layer (Python)"
        BrowserCore["BrowserCore<br/>(Playwright + stealth)"]
        Parser["Parser Module"]
        Writer["Writer Module"]
    end

    subgraph "Storage"
        SQLite[("SQLite<br/>~/.headhunter_ai/db.sqlite")]
        Profile[("Chromium Profile")]
        Config[("YAML configs")]
    end

    subgraph "External"
        ClaudeDesktop["Claude Desktop<br/>(MCP client)"]
        LLMs["OpenAI / Anthropic /<br/>Ollama / etc"]
        HH["hh.ru"]
    end

    UI -->|REST/WS| REST
    TauriCore -.spawn.-> REST
    REST --> Orchestrator
    Orchestrator --> Parser
    Orchestrator --> Writer
    Orchestrator --> AILayer
    Parser --> BrowserCore
    Writer --> BrowserCore
    BrowserCore --> Profile
    BrowserCore --> HH
    AILayer --> LLMs
    MCP --> Orchestrator
    ClaudeDesktop -->|MCP stdio| MCP
    Orchestrator --> SQLite
    REST --> SQLite
```

## Ключевые архитектурные решения

### 1. Tauri shell + Python sidecar
[[UI]] работает в Rust-обёртке, бэкенд запускается как дочерний процесс через Tauri sidecar API. Tauri следит за жизненным циклом, рестартит при падении.

### 2. Один Chromium-инстанс на пользователя
[[Parser service]] и [[Writer service]] делят [[Chromium|BrowserCore]] и одну сессию hh.ru. Сильно экономит RAM и решает куки-проблемы.

> [!note] Это разрешает «расшаривание ядра» — упомянуто в исходном ТЗ как открытый вопрос.

### 3. LLM через LiteLLM (основной AI-канал в MVP)
Единый интерфейс к 100+ провайдерам, multi-provider + fallback через `litellm.Router`. Backend-модуль `services/backend/src/headhunter_backend/ai/` с собственным router `/api/v1/ai`. См. [[AI Layer]] и [[Stage 1 - MVP#1.12 AI Layer — multi-provider + fallback — L|задачу 1.12]].

### 4. MCP — параллельный канал (Stage 2)
Появляется в [[Stage 2 - Расширение#2.1 MCP server — L|2.1]] как альтернатива AI Layer для подписчиков Claude Pro/Max. Два транспорта:
- **stdio** — для Claude Desktop (стандартный способ конфигурации в `claude_desktop_config.json`)
- **streamable HTTP** на `localhost:3845` — для внутренних UI-вызовов и отладки

См. [[MCP]].

### 5. Очередь внутри процесса
Для single-user не нужен Redis/RabbitMQ, достаточно `asyncio.Queue` с персистенцией состояния в [[Storage|SQLite]]. Восстановление при рестарте — `Orchestrator.recover_from_db(session)` в lifespan: заново enqueue'ит активные applications. Поиски в активном статусе на момент рестарта помечаются `INTERRUPTED`.

### 6. Single backend, multi-channel events
`EventBroadcaster` (`api/broadcaster.py`) — внутренний event-bus. Подписчики: WebSocket `/ws/events` (UI realtime), `AutoApplyService` (auto-submit при `auto_submit=true`). Event-типы: `application_event`, `search_event`, `captcha_event`, `vacancy_new`, `auth_changed`.

## Граничные интерфейсы

| Граница                  | Протокол                             | Описание                         |
| ------------------------ | ------------------------------------ | -------------------------------- |
| UI ↔ Backend             | REST + WebSocket                     | См. [[REST]]                     |
| Claude Desktop ↔ Backend | MCP stdio                            | См. [[MCP]]                      |
| Backend ↔ hh.ru          | Playwright (HTTP/WS внутри Chromium) | См. [[Chromium]]                 |
| Backend ↔ LLM            | LiteLLM (HTTP)                       | См. [[AI Layer]]                 |
| Tauri ↔ Python           | stdin/stdout + signals               | spawn через `tauri-plugin-shell` |

## См. также
- [[Stack]] — конкретные библиотеки
- [[Domain Model]] — сущности, которые гоняются между компонентами
