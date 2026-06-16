---
tags: [service]
status: live
---

# Writer service

Модуль отправки откликов на hh.ru. Часть Python-бэка ([[REST]]), работает поверх общего [[Chromium|BrowserCore]] (та же сессия, что и у [[Parser service]]).

## Реализация (snapshot)

- `services/backend/src/headhunter_backend/browser/writer.py` — класс `BrowserWriter`. Главный метод: `async def submit(vacancy_url, letter_text, selectors) -> SubmitResult` — возвращает `SubmitResult { type: SubmitResultType (CAPTCHA/SUBMITTED/FAILED), reason }`.
- Параметры паузы — `min_delay_ms` / `jitter_delay_ms`, прокидываются из `settings` через конструктор в lifespan (`api/app.py:62-64`).
- Consumer-loop — `Orchestrator.consume(...)` в `orchestrator/queue.py`. Поток: pop из `asyncio.Queue` → auth-check (`browser.get_auth_status`) → rate-limit (`ensure_within_limits`) → fetch latest cover letter из БД → `writer.submit(...)` → транзишн state machine + publish event + `log_submission`.
- Captcha-handling: при `SubmitResultType.CAPTCHA` orchestrator делает `pause(reason="captcha")` + `enqueue(app_id)` обратно + publish `CaptchaWSEvent`. Resume — через `POST /api/v1/orchestrator/resume`.

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

| Задача | Реализация | Примечание |
|---|---|---|
| Rate-limit | свой sliding-window (`orchestrator/rate_limiter.py`) поверх таблицы `rate_limits` (`RateLimitEventORM.occurred_at`) | Без внешней библиотеки `limits`. Гейт `ensure_within_limits(session)` в consumer-loop |
| Min delay | свой sleep с jitter в `BrowserWriter` | Параметры из `settings.min_delay_ms` / `settings.delay_jitter_ms` |
| Captcha-detect | DOM-маркер из `Selectors.Captcha.marker` | Проверяется в `submit` перед заполнением формы |
| Retry с backoff | `rate_limit_backoff_sec` в `Orchestrator.consume` + повторный `enqueue` после rate-limit | Без `tenacity` — обычный `asyncio.sleep` |

> [!warning] Никаких captcha-solving сервисов (2captcha и т.п.). Это активный обход защиты, риск ToS-нарушения. См. [[Anti-bot]].

## Связи
- [[Chromium]] — общий BrowserCore
- [[Parser service]] — шарит сессию
- [[Storage]] — обновляет `Application.status`
- [[Anti-bot]] — берёт `RateLimitBudget`
- [[MCP]] — tool `submit_cover_letter` дёргает Writer через orchestrator
