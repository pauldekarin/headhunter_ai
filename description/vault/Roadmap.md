---
tags: [roadmap]
status: in-progress
---

# Roadmap

Дорожная карта проекта [[Description|Headhunter AI]].

## Этапы

- [[Stage 0 - Setup]] — фундамент: монорепа, CI, дисклеймер — **done**
- [[Stage 1 - MVP]] — рабочий путь end-to-end через внутренний [[AI Layer]] (LiteLLM, multi-provider + fallback) — **done** (все задачи 1.1–1.12 закрыты; готов к MVP-демо)
- [[Stage 2 - Расширение]] — внешний AI-канал через [[MCP]], авто-режим, e2e-тесты — **planned**
- [[Stage 3 - Оптимизация]] — параллелизм, наблюдаемость, i18n, плагины — **planned**

> [!summary] **Где мы сейчас:** Stage 1 закрыт — все задачи 1.1–1.12 done. Следующий шаг — MVP-демо (E2E: Settings → Queue picker → AI генерация → LetterReview → Submit) и переход в Stage 2.

## Критический путь MVP

```
Stage 0 → 1.1 → 1.6 → 1.7 → 1.12 → 1.10 → 1.11 → MVP RELEASE
              ↑                ↑     ↑           ↑
        1.4, 1.5, 1.8, 1.9    │     │           │
        ──────────done────────┘     │           │
        + 1.10 LetterReview ────────┘           │
        + 1.11 Settings (с LLM editor) ─────────┘
```

**Минимум** для демо: backend + [[AI Layer]] (1.12) + UI (1.9–1.11). AI-генерация — основная фича MVP: 1.10 (LetterReview) показывает работающий вызов `POST /api/v1/ai/create_cover_letter/{vacancy_id}`, провайдеры настраиваются в 1.11 (Settings → AI tab).

## Сжатый MVP (фактическая последовательность)

1.1 (BrowserCore) ✓ → 1.4 (FastAPI) ✓ → 1.2 (auth) ✓ → 1.3 (parser) ✓ → 1.5 (SQLite) ✓ → 1.6 (orchestrator + SearchService) ✓ → 1.7 (Backend API foundation) ✓ → 1.8 (Writer) ✓ → 1.12 (AI Layer + AutoApplyService) ✓ → 1.9 (Queue UI + picker) ✓ → 1.10 (LetterReview) ✓ → 1.11 (Settings UI + AI tab + LLM editor) ✓ → **MVP-демо**

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
