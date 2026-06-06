---
tags: [requirements]
status: planned
---

# Non-Functional Requirements

| Категория | Требование | Подробности |
|---|---|---|
| Платформа | Linux, macOS, Windows (desktop) | Tauri-бандлы под все три |
| Приватность | Резюме и куки hh.ru не покидают машину пользователя | в режиме «локальный AI» через Ollama |
| Стоимость AI | Поддержать бесплатный путь | локальный Ollama в [[AI Layer]] (MVP); подключение Claude Desktop по подписке через [[MCP]] (Stage 2) |
| Антибот | Не триггерить детекцию hh.ru на личном аккаунте | см. [[Anti-bot]] |
| Перформанс | Парсинг ≥ 50 вакансий/мин | без упирания в CPU (~1 vCPU достаточно) |
| Надёжность | Частичный сбой одной вакансии не валит всю сессию | error-isolation на уровне task в orchestrator |
| Наблюдаемость | Локальные логи, экспортируемые для отладки | structlog + ротация (см. [[Observability]]) |
| Юридика | Явное предупреждение о ToS hh.ru при первом запуске | модальное окно onboarding |
| Обновления | Auto-update с подписью | Tauri updater + ed25519 |
| Безопасность | API-ключи LLM в системном keychain | не в файлах конфигурации |

## См. также
- [[Functional Requirements]]
- [[Anti-bot]]
- [[Observability]]
