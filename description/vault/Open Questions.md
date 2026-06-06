---
tags: [open-question]
status: planned
---

# Open Questions

Вопросы, на которые нужен ответ до или во время реализации. Отсортированы по важности.

## High

### 1. Bundled Chromium vs lazy download
Pre-bundled — +150MB к инсталлятору, но zero-conf для пользователя; download на первом запуске — медленнее first-run, но компактнее.

> Влияет на: [[Chromium]], размер релизов из [[Stage 2 - Расширение]] 2.8.

### 2. Минимальный набор LLM-провайдеров для MVP
Anthropic + OpenAI через API + локальный Ollama — все три как опции в `llm_deployments`? Или ужать до одного-двух на момент выпуска?

> Влияет на: [[AI Layer]], [[Stage 1 - MVP]] 1.12.

## Medium

### 3. Шифровать ли локальную SQLite
Через SQLCipher, ключ из системного keychain. Плюс — приватность резюме; минус — overhead и сложность бэкапа.

> Влияет на: [[Storage]].

### 4. Публичный backlog или приватный
GitHub Issues или приватный таск-трекер?

### 5. Стратегия монетизации
Open source / paid / freemium? Влияет на лицензию и наличие telemetry ([[Observability]]).

## Low

### 6. Какие города/страны hh.ru в приоритете
Может повлиять на специфику селекторов (региональные различия минимальны, но есть).

> Влияет на: [[Parser service]].

### 7. Несколько резюме на одного пользователя
Для разных ролей. Если да — расширение [[Domain Model]].

### 8. Apple Developer ID и Windows Code Signing
Есть / нет / будем покупать? Влияет на [[Stage 2 - Расширение]] 2.8.

## См. также
- [[Gaps and Risks]]
