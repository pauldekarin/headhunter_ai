---
tags: [infra]
status: planned
---

# Observability

Логи, метрики, трейсы, crash reporting.

## Логи

- **structlog** → JSON в `~/.headhunter_ai/logs/` ([[Storage]])
- Ротация по дням, 14 дней TTL
- Log-viewer вкладка в [[UI]] (опционально)
- Все логи содержат `trace_id` для корреляции с tracing

| Задача | Библиотека | Почему |
|---|---|---|
| Структурные логи | **structlog** | Контекстные binders, JSON-вывод |
| Dev-вывод | **rich** | Красивая подсветка |

## Метрики

- **prometheus-client** Python, экспортер на `localhost:9090`
- Опциональный Grafana dashboard в репе (template JSON)

| Метрика | Тип | Описание |
|---|---|---|
| `vacancies_parsed_total` | Counter | Всего распарсено |
| `applications_sent_total` | Counter | Всего отправлено |
| `applications_errors_total` | Counter | Ошибки отправки |
| `parser_duration_seconds` | Histogram | Время парсинга одной вакансии |
| `llm_tokens_total{provider}` | Counter | Токены по провайдерам |
| `llm_latency_seconds{provider}` | Histogram | Латентность LLM-вызовов |
| `rate_limit_remaining{period}` | Gauge | Оставшийся бюджет |

## Tracing

- **opentelemetry-python**, по умолчанию в файл, опционально в OTLP-эндпоинт (Jaeger / Honeycomb / Tempo)
- Span'ы от [[REST]] endpoint до Playwright action в [[Chromium]]

## Crash reporting

- Опциональный **Sentry** (с opt-in диалогом при первом крэше)

## См. также
- [[Stage 3 - Оптимизация]] задачи 3.2, 3.3
- [[Testing]]
