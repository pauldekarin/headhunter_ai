---
tags: [service, infra]
status: live
---

# Chromium

Общее ядро браузерной автоматизации. **Один Chromium-инстанс на пользователя**, который шарят [[Parser service]] и [[Writer service]] (решает «расшаривание ядра» из исходного ТЗ).

## Реализация (snapshot)

- `services/backend/src/headhunter_backend/browser/core.py` — класс `BrowserCore`. Конструктор без параметров. Состояние: `profile_dir`, `headless=False`, `base_url="https://hh.ru"`, `_context`, `_auth_status`.
- Главные методы: `start()`, `stop()`, `new_page(url) -> BrowserPage`, `get_auth_status() -> AuthStatusAPISchema`, `wait_for_login(poll_interval)`.
- `browser/page.py` — `BrowserPage`: тонкая обёртка над Patchright Page (`goto`, `click`, `fill`, `text_content`, `wait_for_selector`, `bring_to_front`, ...).
- Профиль: `~/.headhunter_ai/chrome-profile/` (через `browser.launch_persistent_context()`).
- Шаринг: `BrowserCore` инстанцируется один раз в lifespan (`api/app.py:59`), Parser и Writer берут его как зависимость.

## Зона ответственности

1. Запуск Playwright Chromium с **отдельным** user-data-dir в `~/.headhunter_ai/chrome-profile/` (не основной пользовательский профиль — чтобы не повредить пользовательские куки/расширения).
2. Применение stealth-патчей (patchright).
3. Управление авторизацией:
    - проверка логина по селектору на hh.ru
    - если не залогинен — открытие видимого окна, ожидание ручного логина, сохранение сессии
4. Предоставление shared `BrowserContext` для модулей-потребителей.
5. Lifecycle: start / stop / restart на падение.

## Стек

| Задача | Библиотека | Почему | Альтернативы |
|---|---|---|---|
| Browser automation | **Playwright** (Python) | Async, auto-wait, лучшее API для SPA | Selenium |
| Stealth | **patchright** | Drop-in Playwright с патчами 2024–2025 | playwright-extra-stealth (Node) |
| Persistent context | `browser.launch_persistent_context()` | Профиль с куками + кеш | ручное сохранение storage_state |

## Структура

```
chrome-profile/        ← user-data-dir, выделенный для headhunter_ai
├── Default/
│   ├── Cookies         ← куки hh.ru
│   ├── Local Storage/
│   └── ...
└── ...
```

## Подводные камни

> [!warning] Бандл Playwright тянет за собой ~150MB Chromium. Решение по pre-bundled vs lazy download — открытый вопрос (см. [[Open Questions]] #1).

> [!warning] На macOS/Linux требуется системный sandbox; в Tauri-инсталляторе Python+Chromium должны быть подписаны.

## Связи
- [[Parser service]] — потребитель
- [[Writer service]] — потребитель
- [[Anti-bot]] — где живёт stealth-конфиг
