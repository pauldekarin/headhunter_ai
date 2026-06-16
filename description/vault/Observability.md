---
tags: [infra]
status: partial
---

# Observability

Логи, метрики, трейсы, crash reporting.

> [!summary] **Что подключено live:** `structlog` + `rich` для логов (`log.py:configure_logging`, `get_logger`). **Метрики (Prometheus), tracing (OpenTelemetry), crash reporting (Sentry) — не подключены**, это план на [[Stage 3 - Оптимизация]].

## Логи

- **structlog** → JSON в `~/.headhunter_ai/logs/` ([[Storage]])
- Ротация по дням, 14 дней TTL — **planned** (сейчас вывод в stdout без ротации)
- Log-viewer вкладка в [[UI]] (опционально) — planned
- `trace_id` для корреляции — planned (требует opentelemetry-python)

| Задача | Библиотека | Статус |
|---|---|---|
| Структурные логи | **structlog** | live (`configure_logging` в lifespan) |
| Dev-вывод | **rich** | live (через structlog рендерер) |

## Метрики (planned)

- **prometheus-client** Python, экспортер на `localhost:9090` — не подключено
- Опциональный Grafana dashboard в репе (template JSON) — не подключено

| Метрика | Тип | Описание |
|---|---|---|
| `vacancies_parsed_total` | Counter | Всего распарсено |
| `applications_sent_total` | Counter | Всего отправлено |
| `applications_errors_total` | Counter | Ошибки отправки |
| `parser_duration_seconds` | Histogram | Время парсинга одной вакансии |
| `llm_tokens_total{provider}` | Counter | Токены по провайдерам |
| `llm_latency_seconds{provider}` | Histogram | Латентность LLM-вызовов |
| `rate_limit_remaining{period}` | Gauge | Оставшийся бюджет |

## Tracing (planned)

- **opentelemetry-python**, по умолчанию в файл, опционально в OTLP-эндпоинт (Jaeger / Honeycomb / Tempo) — не подключено
- Span'ы от [[REST]] endpoint до Playwright action в [[Chromium]] — не подключено

## Crash reporting (planned)

- Опциональный **Sentry** (с opt-in диалогом при первом крэше) — не подключено

## См. также
- [[Stage 3 - Оптимизация]] задачи 3.2, 3.3
- [[Testing]]
