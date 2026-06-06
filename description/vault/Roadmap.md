---
tags: [roadmap]
status: planned
---

# Roadmap

Дорожная карта проекта [[Description|Headhunter AI]].

## Этапы

- [[Stage 0 - Setup]] — фундамент: монорепа, CI, дисклеймер
- [[Stage 1 - MVP]] — рабочий путь end-to-end через внутренний [[AI Layer]] (LiteLLM, multi-provider + fallback)
- [[Stage 2 - Расширение]] — внешний AI-канал через [[MCP]], авто-режим, e2e-тесты
- [[Stage 3 - Оптимизация]] — параллелизм, наблюдаемость, i18n, плагины

## Критический путь MVP

```
Stage 0 → 1.1 → 1.6 → 1.7 → 1.12 → 1.10 → MVP RELEASE
              ↑                ↑
        1.4, 1.5, 1.8, 1.9, 1.11 — параллельно
```

**Минимум** для демо: backend + [[AI Layer]] (1.12) + UI (1.9–1.11). AI-генерация — основная фича MVP, идёт перед UI: 1.10 (LetterReview) показывает уже работающий вызов `/api/v1/ai/cover_letter`.

## Сжатый MVP

1.1 (BrowserCore) → 1.2 (auth) → 1.3 (parser) → 1.4 (FastAPI) → 1.5 (SQLite) → 1.6 (orchestrator) → 1.7 (Backend API foundation) → 1.8 (Writer) → 1.12 (AI Layer) → 1.9 (SearchForm) → 1.10 (LetterReview) → 1.11 (Settings)

Демо: backend генерирует письма через AI Layer с настроенным провайдером (Anthropic / OpenAI / Ollama), UI показывает их в LetterReview, пользователь подтверждает отправку. Валидация — через REST + UI; внешний Claude Desktop не нужен.

## Дальше

```
MVP ──→ 2.1 (MCP) ──→ 2.10 (docs) ──→ 2.2 (промпты) ──→ 2.3 (auto-submit)
    └─→ 2.4 (dedup) [parallel]
    └─→ 2.5 (captcha) [parallel]
    └─→ 2.6 (audit) [parallel]
    └─→ 2.9 (selectors) [parallel]
                            ──→ 2.7 (e2e) ──→ 2.8 (auto-update) ──→ v1.0

v1.0 ──→ Stage 3 (3.1-3.7 в основном параллельно)
```

## Принципы планирования

- **MVP включает [[AI Layer]]** — собственный путь к LLM (LiteLLM + multi-provider + fallback) даёт self-contained приложение; пользователь приносит ключ.
- **MCP — только во 2-м этапе** — параллельный канал для подписчиков Claude Pro/Max, не критичен для базового пути.
- **Auto-submit — только во 2-м этапе** — в MVP только ручное подтверждение, чтобы минимизировать риск бана.
- **E2E-тесты — после MVP** — сначала ручная валидация, потом автоматизация.
- **Параллелизм парсинга — последний этап** — сначала корректность, потом скорость.

## См. также
- [[Gaps and Risks]] — что может пойти не так
- [[Open Questions]] — что нужно решить до MVP
- [[Verification]] — как проверим успех
