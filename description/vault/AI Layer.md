---
tags: [service]
status: live
---

# AI Layer

Внутренняя абстракция над LLM-провайдерами. Основной канал AI-генерации сопроводительных писем в [[Stage 1 - MVP]] — backend-модуль с собственным router `/api/v1/ai`. [[MCP]]-канал — параллельный путь в [[Stage 2 - Расширение]].

## Зачем
- Self-contained приложение: пользователь приносит свой ключ к LLM-провайдеру, не зависит от внешнего Claude Desktop.
- Поддержка приватного режима — локальный LLM через Ollama (OpenAI-совместимый `api_base`).
- Resilience через multi-provider + fallback: если primary провайдер упал, автоматический переход к следующему.
- Удобство для энтузиастов — поэкспериментировать с разными моделями.

## Реализация (фактическая)

Модуль `services/backend/src/headhunter_backend/ai/`:

- **`layer.py`** — класс `AILayer`:
  - `generate_cover_letter(vacancy_model, resume, style, system_prompt) -> AICoverLetterResult` — собирает messages через `PromptBuilder`, вызывает `litellm.Router.acompletion`, парсит usage/cost/`was_fallback`.
  - `get_health_status() -> AILayerHealthStatus` — `HEALTHY` / `UNHEALTHY` / `NO_DEPLOYMENTS`. Под капотом — `PromptBuilder.build_ping()` через primary.
  - `rebuild(deployments)` — пересборка Router'а. **Сейчас вызывается только из `bootstrap_ai_layer` на startup** (`api/app.py:42-52`); invalidate-hook на PUT settings — open gap (см. [[Stage 1 - MVP#1.11]]).
- **`deployment.py`** — Pydantic `LLMDeployment { model, api_key?, api_base? }` с `id()` = SHA256(...)[:16].
- **`prompts.py`** — `PromptBuilder.build_cover_letter_prompt(...)` и `.build_ping()`. Default system-промпт: личный кириллический cover letter под ≈250 слов, без выдумок.
- **`result.py`** — `AICoverLetterResult { text, model_used, prompt/completion/total_tokens, was_fallback, cost_usd }`.
- **`health.py`** — `AILayerHealthStatus` enum (`HEALTHY`/`UNHEALTHY`/`NO_DEPLOYMENTS`), `is_ready()`.
- **`exceptions.py`** — `GenerationCoverLetterException`.

## Стек

| Задача | Библиотека | Почему | Альтернативы | Подводные камни |
|---|---|---|---|---|
| LLM proxy | **LiteLLM** (Python) | 100+ провайдеров, OpenAI-совместимый API, retry/fallback/caching из коробки | langchain (тяжёлый, нестабильный API); прямые SDK провайдеров (много дублирования) | Иногда отстаёт от свежих фич конкретного провайдера |
| Локальный LLM | **Ollama** | Простейший локальный хостинг моделей, OpenAI-совместимое API | llama.cpp (low-level), LM Studio (GUI, не headless) | Качество русского зависит от модели |
| Хранение ключей | **`keyring`** | Системный keychain | env-файлы (небезопасно) | На Linux нужен GNOME Keyring или KWallet |
| Промпт-шаблоны | YAML в `~/.headhunter_ai/prompts/` | Версионирование, A/B-тесты | — | — |

## Поддерживаемые провайдеры (MVP)

- **Anthropic Claude** (через API, опционально)
- **OpenAI** (GPT-4o / o1)
- **Ollama** (локально)

> [!note] Расширение списка — ноль кода. LiteLLM знает Mistral, Cohere, Groq, Together, Bedrock, Vertex и многих других.

## Связи
- [[MCP]] — параллельный канал в [[Stage 2 - Расширение]]
- [[Domain Model]] — `LLMProvider`, `PromptTemplate`
- [[Stage 1 - MVP]] — задача 1.12 (внедрение LiteLLM с multi-provider + fallback)
- [[Stage 2 - Расширение]] — задача 2.2 (промпт-шаблоны и версионирование)
