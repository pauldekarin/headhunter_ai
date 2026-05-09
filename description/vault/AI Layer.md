---
tags: [service]
status: planned
---

# AI Layer

Внутренняя абстракция над LLM-провайдерами. Альтернативный путь к AI, когда пользователь не использует Claude Desktop через [[MCP]].

## Зачем
- Поддержка пользователей без подписки Claude Pro/Max (платный API напрямую)
- Поддержка приватного режима (локальный LLM через Ollama)
- Удобство для энтузиастов — поэкспериментировать с разными моделями

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
- [[MCP]] — альтернативный путь к AI
- [[Domain Model]] — `LLMProvider`, `PromptTemplate`
- [[Stage 2 - Расширение]] — задача 2.1 (внедрение LiteLLM)
