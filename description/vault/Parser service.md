---
tags: [service]
status: planned
---

# Parser service

Модуль парсинга вакансий с hh.ru. Часть Python-бэка ([[REST]]), запускается как in-process модуль и работает поверх общего [[Chromium|BrowserCore]].

## Зона ответственности

1. По `SearchFilter` ([[Domain Model]]) перейти на страницу выдачи hh.ru.
2. Извлечь карточки вакансий и собрать поля:
   - description, title, salary, work_format, employer, employment_type, location, company_stars, published_at, apply_link
3. Опубликовать каждую вакансию в orchestrator → стримом в [[UI]] через WebSocket ([[REST]]).
4. Дедупликация по `apply_link` (см. [[Storage]]).

## Что НЕ делает
- Не отправляет отклики — это зона [[Writer service]]
- Не вызывает AI — это [[AI Layer]] / внешний MCP-клиент
- Не управляет логином — общий [[Chromium]] держит сессию

## Стек

| Задача | Библиотека | Почему | Альтернативы |
|---|---|---|---|
| Browser automation | **Playwright** (Python) | Auto-wait, async API, лучшее API для SPA | Selenium (медленнее, flaky); puppeteer (только Node) |
| Stealth | **patchright** | Drop-in Playwright с патчами 2024–2025 | playwright-extra-stealth (стагнирует), undetected-chromedriver (Selenium-only) |
| HTML-парсинг | **selectolax** | На порядок быстрее BeautifulSoup для мелких выдержек | BeautifulSoup (привычнее), parsel |
| Валидация полей | **Pydantic v2** | Типизация, integration с FastAPI | attrs |

## Селекторы

> [!warning] Селекторы — самая хрупкая часть. Централизуем в одном модуле `browser/selectors.py` и покрываем daily smoke-тестом (см. [[Stage 2 - Расширение]] задачу 2.9).

## Параллелизм
В MVP — одна вкладка. На [[Stage 3 - Оптимизация]] (3.1) — до 3 вкладок параллельно через пул контекстов Playwright.

## Связи
- [[Chromium]] — общий BrowserCore
- [[Writer service]] — родственный модуль, шарит [[Chromium|BrowserCore]]
- [[REST]] — публикует вакансии через orchestrator → WebSocket
- [[Anti-bot]] — лимиты и stealth применяются здесь
