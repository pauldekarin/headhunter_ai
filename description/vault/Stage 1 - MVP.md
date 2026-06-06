---
tags: [stage/1]
status: planned
---

# Stage 1 — MVP

**Цель**: пользователь может запустить приложение, авторизоваться в hh.ru, получить список вакансий по фильтру, сгенерировать письма через внутренний **[[AI Layer]]** (LiteLLM, multi-provider + fallback), и отправить отклики вручную с подтверждением.

> [!note] AI в MVP — внутренний [[AI Layer]] поверх LiteLLM с мульти-провайдером и fallback-цепочкой. MCP-канал для внешних AI-клиентов (Claude Desktop, Claude Code) перенесён в [[Stage 2 - Расширение]].

## Задачи

### 1.1 BrowserCore с персистентным профилем — `M`

Запуск Playwright Chromium с user-data-dir в `~/.headhunter_ai/chrome-profile/`. Stealth-патчи. См. [[Chromium]].

**AC**: Логин в hh.ru сохраняется между запусками; `navigator.webdriver === undefined`.

**Зависимости**: [[Stage 0 - Setup]] 0.1.

---

### 1.2 Auth-flow — `M`

Проверить логин по cookie `hhrole`, если не залогинен — открыть видимый Chromium на странице логина hh.ru и ждать, пока пользователь залогинится. См. [[Chromium]].

**AC**: [[UI]] отображает иконку авторизации в шапке (3 состояния: anonymous, waiting, authenticated). Клик по иконке в состоянии anonymous запускает логин. Backend открывает hh.ru-страницу логина в видимом Chromium с автоматическим bring-to-front. После успешного логина иконка обновляется реактивно через WS-событие.

**Note**: Авторизация **опциональна** для парсинга и генерации писем — обязательна только для отправки откликов (см. 1.8 Writer, 1.10 LetterReview UI).

**Зависимости**: 1.1, 1.4.

---

### 1.3 Parser module — `L`

По SearchFilter перейти на страницу выдачи, извлечь все 10 полей вакансии для каждой карточки + детали со страницы вакансии. См. [[Parser service]].

**AC**: Возвращает 50 вакансий из тестового запроса; селекторы вынесены в отдельный модуль `browser/selectors.py`.

**Зависимости**: 1.1.

---

### 1.4 FastAPI backend skeleton — `M`

REST endpoints под префиксом `/api/v1`: `POST /vacancies/search`, `GET /vacancies/`, `POST /vacancies/{vacancy_id}/cover_letter`, `POST /vacancies/{vacancy_id}/submit`. WebSocket `/ws/vacancies` для стрима. См. [[REST]].

**AC**: OpenAPI spec доступен на `/docs`; WebSocket стримит mock-данные.

**Зависимости**: [[Stage 0 - Setup]] 0.1.

---

### 1.5 SQLite + миграции — `S`

Таблицы: `vacancies`, `applications`, `prompts`, `audit_log`. Alembic init. См. [[Storage]].

**AC**: `alembic upgrade head` создаёт схему; CRUD для `vacancies` работает.

**Зависимости**: 1.4.

---

### 1.6 Orchestrator + очередь — `M`

State machine для каждой вакансии (см. [[Domain Model]]): `parsed → letter_pending → letter_ready → sent / error`. asyncio.Queue с персистенцией статусов в [[Storage|SQLite]]. История поисков восстанавливается через `GET /api/v1/vacancies/searches`, активный поиск — через `GET /api/v1/vacancies/search/{search_id}` и `DELETE /api/v1/vacancies/search/{search_id}` (отмена).

**AC**: При перезапуске бэка очередь восстанавливается из SQLite; история поисков и активные таски доступны через REST.

**Зависимости**: 1.4, 1.5.

---

### 1.7 Backend API foundation — `M`

Реальные эндпоинты поверх каркаса 1.4: убрать моки, подключить SQLite и стейт-машину 1.6, добавить недостающие таблицы и API настроек.

- Таблицы `cover_letters` (версии писем по `application_id`) и `settings` (синглтон с `CHECK(id=1)`).
- Реальные `GET /api/v1/vacancies`, `GET /api/v1/vacancies/{id}`, WebSocket `/ws/vacancies` — из БД, не моки.
- Lifecycle-эндпоинты на стейт-машине: `POST /api/v1/vacancies/{id}/queue_for_letter`, `/cover_letter`, `/submit`, `/retry`, `GET /{id}/status`.
- `GET`/`PUT /api/v1/settings` поверх таблицы `settings` (get-or-create с дефолтами из pydantic).

См. [[REST]], [[Storage]], [[Domain Model]].

**AC**: pytest зелёный по эндпоинтам; UI может ходить в реальные `vacancies` и `settings` без моков.

**Зависимости**: 1.4, 1.5, 1.6.

---

### 1.8 Writer module — `L`

По vacancy_id и тексту письма: открыть страницу вакансии, нажать «Откликнуться», вставить письмо, submit, проверить успех (поиск success-фразы в DOM с NFKC-нормализацией + латин-кириллица confusables). См. [[Writer service]].

Consumer-loop в `Orchestrator` пуллит заявки из очереди, проверяет auth + rate-limit ([[Anti-bot]]), дёргает Writer и пишет результат в БД + WS (`submission_event`, `captcha_event`). Детект капчи → `pause`; `POST /api/v1/orchestrator/resume` снимает паузу. Rate-limit использует таблицу `rate_limits` (token-bucket по sliding window из БД).

**AC**: pytest зелёный (consumer-loop по всем веткам, rate-limiter, pause/resume, resume-endpoint); E2E на тестовой вакансии — отклик отправлен, статус `letter_sent`, строка в `rate_limits`.

**Зависимости**: 1.1, 1.5, 1.6, 1.7.

---

### 1.9 UI: SearchForm + VacancyList — `M`

Форма фильтра (text, salary, region), список вакансий с realtime-обновлением через WebSocket. См. [[UI]].

**AC**: Поиск запускается, вакансии появляются по мере парсинга.

**Зависимости**: 1.4.

---

### 1.10 UI: LetterReview — `M`

Карточка вакансии с превью письма, кнопки «Редактировать», «Сгенерировать заново» (вызов `POST /api/v1/ai/create_cover_letter/{vacancy_id}`), «Отправить», «Пропустить».

**AC**: Письмо генерируется, редактируется и отправляется; статус обновляется в списке.

**Зависимости**: 1.6, 1.8, 1.9, 1.12.

---

### 1.11 UI: Settings — `S`

Контекст соискателя (резюме textarea/upload), стиль письма, лимиты ([[Anti-bot]]), путь к Chromium-профилю, **LLM deployments** (список модель + ключ + опциональный api_base, primary первый, остальные — fallback chain), system-промпт.

**AC**: Настройки сохраняются через `PUT /api/v1/settings`, применяются без рестарта. AI Layer пересобирает Router при обновлении `llm_deployments`.

**Зависимости**: 1.4, 1.12.

---

### 1.12 AI Layer — multi-provider + fallback — `L`

Backend-модуль под `services/backend/src/headhunter_backend/ai/` с собственным router `/api/v1/ai`. Унифицированный фасад над LLM-провайдерами через LiteLLM: мульти-провайдер (Anthropic / OpenAI / Ollama), fallback-цепочка через `litellm.Router`, retry с exponential backoff. См. [[AI Layer]].

**Эндпоинты**:
- `POST /api/v1/ai/create_cover_letter/{vacancy_id}` — синхронно генерирует письмо по vacancy_id из path. Берёт резюме, стиль и `system_prompt` из `settings`, дёргает `AILayer.generate_cover_letter`, возвращает `AICoverLetterResponseAPISchema` (text, model_used, prompt/completion/total tokens, was_fallback, cost_usd). 409 если AI слой не готов (нет deployments) или для вакансии нет открытой `application`. 404 если vacancy не найдена.
- `GET /api/v1/ai/health` — отдаёт статус AI-слоя (`AIHealthStatusAPISchema`): `healthy` / `unhealthy` / `no_deployments`. Под капотом — `ping`-промпт через primary deployment.

**Расширение `settings`** (отдаётся через существующий `PUT /api/v1/settings`):
- `llm_deployments: list[{model, api_key, api_base?}]` — упорядоченный список deployments, primary первый, остальные — fallback chain.
- `llm_system_prompt: str` — system-промпт, перекрывает / дополняет `letter_style`.

**Реализация**:
- `AILayer.generate(prompt, ctx) -> str` поверх `litellm.Router`.
- Router пересобирается при изменении settings (через invalidate-hook на PUT settings).
- Retry policy: 2 попытки на deployment, exponential backoff (100ms → 400ms).
- Fallback автоматически по цепочке при `litellm.exceptions.{RateLimitError, APIError, Timeout}`.
- Hardcoded MVP-промпт (версионирование — Stage 2 [[Stage 2 - Расширение#2.2|2.2]]).

**AC**:
- pytest зелёный: `AILayer` (с моком `litellm.Router`), endpoints, fallback-логика при имитации падения primary.
- E2E на тестовой вакансии: настроены два deployments (Anthropic + OpenAI как fallback). При отзыве ключа primary письмо всё равно генерируется через fallback. Статус вакансии — `letter_ready`, запись в `cover_letters`.
- Settings UI (1.11) показывает список deployments с возможностью добавления / удаления / переупорядочения.

**Зависимости**: 1.6, 1.7.

**Стек**:

| Задача | Библиотека | Почему | Альтернативы |
|---|---|---|---|
| LLM proxy | **LiteLLM** | 100+ провайдеров, унифицированный API, Router из коробки | langchain (тяжёлый); прямые SDK (дублирование) |
| Router/fallback | `litellm.Router` | Встроено, без сторонних зависимостей | ручной retry поверх `litellm.completion` |
| Локальный LLM | **Ollama** | Опциональный provider, OpenAI-совместимый API | llama.cpp (low-level) |

---

## Стек этапа (детально)

См. [[Stack]] и каждую заметку компонента ([[Parser service]], [[REST]], [[MCP]], [[UI]], [[Storage]]).

## См. также
- [[Roadmap]]
- предыдущий: [[Stage 0 - Setup]]
- следующий: [[Stage 2 - Расширение]]
- [[Verification]] — как проверить MVP
