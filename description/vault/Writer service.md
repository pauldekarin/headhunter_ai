---
tags: [service]
status: planned
---

# Writer service

Модуль отправки откликов на hh.ru. Часть Python-бэка ([[REST]]), работает поверх общего [[Chromium|BrowserCore]] (та же сессия, что и у [[Parser service]]).

## Зона ответственности

1. По `vacancy_id` и тексту письма (`CoverLetter` из [[Domain Model]]):
    - открыть страницу вакансии
    - нажать «Откликнуться»
    - вставить письмо в форму
    - submit
    - проверить успех по DOM/URL
2. Обновить `Application.status` в [[Storage]] → `sent` или `error`.
3. Применить лимиты из [[Anti-bot]] перед отправкой.
4. При капче — пауза очереди, нотификация в [[UI]], ожидание ручного решения в открытом Chromium, resume.

## Что НЕ делает
- Не парсит вакансии — [[Parser service]]
- Не генерирует письмо — [[AI Layer]] / внешний AI-клиент через [[MCP]]
- Не управляет логином — общий [[Chromium]]

## Стек

Тот же что и у [[Parser service]] — **Playwright + patchright**. Дополнительно:

| Задача | Библиотека | Почему |
|---|---|---|
| Token-bucket лимиты | **`limits`** (Python) | Готовая реализация с разными бэкендами |
| Captcha-detect | свой детектор по DOM | hh.ru использует свой капча-виджет, специфичные селекторы |
| Retry с backoff | **tenacity** | Декларативный, экспоненциальный backoff |

> [!warning] Никаких captcha-solving сервисов (2captcha и т.п.). Это активный обход защиты, риск ToS-нарушения. См. [[Anti-bot]].

## Связи
- [[Chromium]] — общий BrowserCore
- [[Parser service]] — шарит сессию
- [[Storage]] — обновляет `Application.status`
- [[Anti-bot]] — берёт `RateLimitBudget`
- [[MCP]] — tool `submit_cover_letter` дёргает Writer через orchestrator
