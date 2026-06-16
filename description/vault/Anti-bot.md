---
tags: [risk, infra]
status: planned
---

# Anti-bot

Стратегия против детекции автоматизации hh.ru. **Решение: гибрид** — личный Chromium-профиль + лёгкий stealth + жёсткие лимиты.

## Угрозы

| Угроза | Митигация |
|---|---|
| Детекция WebDriver-маркеров | patchright скрывает `navigator.webdriver`, fonts, plugins, etc. |
| Поведенческая детекция (слишком быстрые/равномерные клики) | Random human-like delays между действиями |
| Превышение лимитов hh.ru → бан | Token-bucket: дефолт 30 откликов/день, 5 в час |
| Капча | Pause-and-prompt (см. ниже), без автоматического обхода |
| Fingerprinting | Используем личный профиль пользователя — fingerprint совпадает с обычным |
| Account takeover-detection (новое устройство) | Логин происходит руками в видимом окне один раз, дальше та же сессия |

## Стек (фактический)

| Задача | Реализация | Подводные камни |
|---|---|---|
| Stealth | **patchright** (drop-in Playwright) | Гонка вооружений |
| Rate-limit | свой sliding-window over `rate_limits` таблицы (`RateLimitEventORM`, `orchestrator/rate_limiter.py`) | Без внешней `limits` библиотеки; consumer-loop вызывает `ensure_within_limits(session)` перед каждой отправкой |
| Random delays | свой sleep с `min_delay_ms` ± `delay_jitter_ms` из settings | Параметры конфигурируются в Settings UI (1.11) |
| Captcha-detect | DOM-маркер из `Selectors.Captcha.marker` | hh.ru использует свой капча-виджет, специфичные селекторы |

## Captcha-handling (фактический поток)

> [!warning] **Никаких 2captcha / anti-captcha.** Это активный обход, нарушает дух ToS и существенно повышает риск перманентного бана.

1. `BrowserWriter.submit` обнаружил DOM-маркер → возвращает `SubmitResult { type: CAPTCHA }`
2. `Orchestrator._process_one` вызывает `pause(reason="captcha")` + `enqueue(app_id)` обратно
3. Публикует `CaptchaWSEvent` в `EventBroadcaster` → `/ws/events`
4. UI показывает уведомление, открытое окно [[Chromium]] остаётся видимым
5. Пользователь решает капчу руками
6. Пользователь дёргает `POST /api/v1/orchestrator/resume` (через кнопку в UI) → `resume_event.set()` → consumer-loop возобновляется

## Лимиты по умолчанию

| Лимит | Значение | Обоснование |
|---|---|---|
| Откликов в час | 5 | Соответствует темпу человека «активного поиска» |
| Откликов в день | 30 | Не вызывает подозрений у hh.ru |
| Параллельных вкладок парсинга | 1 (MVP), до 3 ([[Stage 3 - Оптимизация]]) | Чтобы не было всплесков |
| Min delay между действиями | 800ms ± 400ms | Сравнимо с human reaction time |

Все лимиты конфигурируемы пользователем (с warning при увеличении).

## Юридические риски

> [!danger] hh.ru ToS формально запрещает автоматизацию откликов. Использование на свой риск.

При первом запуске показываем модал:
- Краткое описание риска бана
- Рекомендация использовать на не-основном аккаунте
- Согласие сохраняется в `~/.headhunter_ai/consent.json`

## Связи
- [[Chromium]] — где живёт stealth
- [[Writer service]] — применяет лимиты перед отправкой
- [[Storage]] — `rate_limits` таблица
- [[Gaps and Risks]] — общий регистр рисков
