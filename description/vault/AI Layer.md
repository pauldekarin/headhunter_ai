---
tags: [service]
status: planned
---

# AI Layer

Внутренняя абстракция над LLM-провайдерами. Основной канал AI-генерации сопроводительных писем в [[Stage 1 - MVP]] — backend-модуль с собственным router `/api/v1/ai`. [[MCP]]-канал — параллельный путь в [[Stage 2 - Расширение]].

## Зачем
- Self-contained приложение: пользователь приносит свой ключ к LLM-провайдеру, не зависит от внешнего Claude Desktop.
- Поддержка приватного режима — локальный LLM через Ollama.
- Resilience через multi-provider + fallback: если primary провайдер упал, автоматический переход к следующему.
- Удобство для энтузиастов — поэкспериментировать с разными моделями.

## Интерфейс

```python
class AILayer:
    async def generate(
        self,
        prompt: str,
        ctx: ChatContext,
        vacancy: Vacancy,
    ) -> CoverLetter: ...
```

Конкретный провайдер выбирается из настроек, переключаемо в рантайме.

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
